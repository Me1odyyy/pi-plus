# Session 搜索

Pi 搜索是在已提交 Session 条目之上提供的小型查询接口。共享契约只返回稳定的命中标识；具体实现可以使用后端特有的展示数据扩展命中结果。

## 核心 API

```ts
export interface SessionSearchHit {
  /** Logical identifier of the session that owns the entry. */
  readonly sessionId: string;

  /** Logical identifier of the entry within that session. */
  readonly entryId: string;
}

export interface SessionSearchOptions {
  /** Restrict results to specific canonical entry types. */
  readonly entryTypes?: readonly Entry["type"][];

  /** Maximum number of hits to return. Backends may return fewer, not more. */
  readonly limit?: number;

  /** Abort signal for cancellation, e.g. search-as-you-type. */
  readonly signal?: AbortSignal;
}

export interface SessionSearch<T extends SessionSearchHit = SessionSearchHit> {
  search(text: string, options?: SessionSearchOptions): AsyncIterable<T>;
}
```

基础命中结果有意保持最小化：`(sessionId, entryId)` 是跨 JSONL、Memory、SQLite FTS 和远程索引的可移植标识。摘要、时间戳、分数、元数据、偏移量和排序语义由具体实现定义。

## 为什么使用 async iterable

`AsyncIterable` 允许使用方尽早渲染结果、在获得足够结果后停止迭代，并通过 `AbortSignal` 取消进行中的工作。防抖仍由 UI 或调用方负责；API 只提供取消原语。

```ts
let currentAbortController: AbortController | undefined;

async function updateResults(query: string) {
  currentAbortController?.abort();
  const controller = new AbortController();
  currentAbortController = controller;

  try {
    for await (const hit of search.search(query, { limit: 10, signal: controller.signal })) {
      render(hit);
    }
  } catch (error) {
    if (!(error instanceof Error) || error.name !== "AbortError") throw error;
  }
}
```

## 默认实现

### 扫描搜索

可复用扫描器把类似 Session 的可读对象（`getMetadata`、`findEntries` 和 `getLabel`）适配为投影条目：

```ts
export interface SessionSearchCandidate {
  readonly entryId: string;
  readonly seq: number;
  readonly type: Entry["type"];
  readonly timestamp: number;
  readonly text: string;
  readonly fields?: Record<string, unknown>;
}

export interface ScanningSessionSearchHit extends SessionSearchHit {
  readonly timestamp: number;
  readonly snippet: string;
}
```

`SessionSearchCandidate` 是匹配前的扫描器输入：它包含可搜索文本、类型、序列号和可选投影字段。扫描器把匹配的候选项转换为公开命中结果。

已经打开的 Session 或 Storage 可以直接扫描：

```ts
const search = createScanningSessionSearch(sessions);

for await (const hit of search.search("authentication", { limit: 10 })) {
  const session = sessionsById.get(hit.sessionId)!;
  const entry = await session.getEntry(hit.entryId);
  console.log(entry);
}
```

JSONL 不需要单独的公开搜索适配器。基于 JSONL 的代码可以在本地完成发现和加载，然后把已加载的 Storage 传给同一个扫描器：

```ts
async function* jsonlReadables(jsonl: JsonlSessionRepoOptions, query: JsonlSessionListOptions = {}) {
  for (const metadata of await listJsonlSessionMetadata(jsonl, query)) {
    yield loadJsonlSessionStorage(jsonl, metadata);
  }
}

const search = createScanningSessionSearch((query) => jsonlReadables(jsonl, query));
```

如果 `SessionRepo.open()` 可能获取写入者租约，扫描源不得对 Harness 拥有的 Session 调用它。JSONL 应使用只读加载辅助函数；已经打开的 Session 或 Storage 可以直接扫描。

### SQLite FTS

SQLite 搜索公开扩展命中结果：

```ts
export interface SqliteSessionSearchHit extends SessionSearchHit {
  readonly metadata: SqliteSessionMetadata;
  readonly timestamp: number;
  readonly score: number;
}
```

```ts
const search = createSqliteSessionSearch({ env, sqlite, databasePath });

for await (const hit of search.search("auth", {
  entryTypes: ["message", "compaction"],
  limit: 20,
})) {
  console.log(hit.sessionId, hit.entryId, hit.score);
}
```

FTS 表和触发器会在第一次非空搜索时延迟创建。首次创建 FTS 时，SQLite 会根据规范 `entries` 执行一次性重建；之后，SQLite 触发器会让 FTS 与规范条目的插入、删除和 payload 更新保持同步。因此，SQLite 搜索会在提交后立即保持最新，但也意味着当该数据库启用搜索时，FTS 触发器失败可能回滚规范 SQLite 写入。

## 索引后端

搜索索引是由后端负责的派生状态。共享包只导出查询 API；当应用或后端包需要显式维护索引时，可以定义自己的写入或 feed 契约。

### 使用 Elasticsearch 的 JSONL Session

这是由应用负责的粘合代码。Core 提供查询契约和 JSONL Session 发现；Elastic 写入契约仅属于此适配器。

```ts
import { Client } from "@elastic/elasticsearch";
import {
  scanningEntries,
  type JsonlSessionMetadata,
  type JsonlSessionRepoOptions,
  type SessionSearch,
  type SessionSearchHit,
  type SessionSearchOptions,
} from "@earendil-works/pi-agent-core";

// JSONL-backed code can provide this locally from existing JSONL list/load helpers.
async function* jsonlReadables(jsonl: JsonlSessionRepoOptions, options: { cwd?: string } = {}) {
  for (const metadata of await listJsonlSessionMetadata(jsonl, options)) {
    yield loadJsonlSessionStorage(jsonl, metadata);
  }
}

interface SearchIndexWriter<TItem> {
  apply(items: TItem[]): Promise<void>;
  flush?(): Promise<void>;
}

interface IndexedSessionSearch<T extends SessionSearchHit, TItem>
  extends SessionSearch<T>, SearchIndexWriter<TItem> {}

type ElasticSessionFeedItem =
  | { type: "upsert"; id: string; body: ElasticSessionDoc }
  | { type: "delete"; id: string };

interface ElasticSessionDoc {
  sessionId: string;
  entryId: string;
  seq: number;
  timestamp: number;
  cwd: string;
  text: string;
  metadata: JsonlSessionMetadata;
  fields?: Record<string, unknown>;
}

interface ElasticSessionSearchHit extends SessionSearchHit {
  readonly timestamp: number;
  readonly snippet: string;
  readonly score?: number;
}

class ElasticSessionSearch
  implements IndexedSessionSearch<ElasticSessionSearchHit, ElasticSessionFeedItem>
{
  constructor(
    private readonly client: Client,
    private readonly index: string,
  ) {}

  async apply(items: ElasticSessionFeedItem[]): Promise<void> {
    const operations = items.flatMap((item) => {
      if (item.type === "delete") {
        return [{ delete: { _index: this.index, _id: item.id } }];
      }
      return [{ index: { _index: this.index, _id: item.id } }, item.body];
    });

    if (operations.length > 0) await this.client.bulk({ operations });
  }

  async flush(): Promise<void> {
    await this.client.indices.refresh({ index: this.index });
  }

  async *search(
    text: string,
    options: SessionSearchOptions = {},
  ): AsyncIterable<ElasticSessionSearchHit> {
    const result = await this.client.search<ElasticSessionDoc>({
      index: this.index,
      size: options.limit ?? 20,
      query: {
        bool: {
          must: [{ match: { text } }],
        },
      },
    });

    for (const hit of result.hits.hits) {
      if (!hit._source) continue;
      if (options.signal?.aborted) throw options.signal.reason;
      yield {
        sessionId: hit._source.sessionId,
        entryId: hit._source.entryId,
        timestamp: hit._source.timestamp,
        snippet: hit._source.text,
        score: hit._score ?? undefined,
      };
    }
  }
}
```

追赶或重建任务可以在不获取写入者租约的情况下，把 JSONL 投影写入 Elasticsearch：

```ts
async function indexJsonlSessionsIntoElastic(
  jsonl: JsonlSessionRepoOptions,
  elastic: ElasticSessionSearch,
  options: { cwd?: string } = {},
): Promise<void> {
  for await (const session of jsonlReadables(jsonl, { cwd: options.cwd })) {
    const metadata = await session.getMetadata();
    for await (const candidate of scanningEntries(session)) {
      await elastic.apply([{
        type: "upsert",
        id: `${metadata.id}:${candidate.entryId}`,
        body: {
          sessionId: metadata.id,
          entryId: candidate.entryId,
          seq: candidate.seq,
          timestamp: candidate.timestamp,
          cwd: metadata.cwd,
          text: candidate.text,
          metadata,
          fields: candidate.fields,
        },
      }]);
    }
  }

  await elastic.flush();
}
```

## 正确性与失败边界

对于共享 API，搜索索引属于派生状态：应用可以重试、重建，或把搜索标记为过期。不同后端可以作出不同取舍；SQLite FTS 使用同库触发器，因此搜索初始化触发器后，FTS 失败可能回滚规范 SQLite 写入。

如果扫描源产生重复的 `sessionId`，应立即失败，因为基础命中标识是 `(sessionId, entryId)`。索引后端通常在其 Storage 或索引层强制唯一性。

选择启用搜索仍需要同步或索引层。后续应添加默认无操作的搜索索引 sink（例如 `NOOP_SEARCH_INDEX_SINK`），使规范写入点可以无条件发出索引事件，类似于禁用 Telemetry 时使用无操作实现。
