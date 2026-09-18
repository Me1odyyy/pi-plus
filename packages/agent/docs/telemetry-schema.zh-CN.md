# Pi Agent Telemetry Schema

<!-- 由 generate-telemetry-docs.ts 生成。请勿手动编辑英文原文。 -->

## AI 请求 Schema

Schema 版本：1

### `pi.ai.request`

向 AI Provider 发起的一次逻辑请求

- Parent：root 或任意调用方 span
- 默认状态：`ok`
- 错误条件：操作抛出异常或返回错误结果

#### 开始属性

| 名称 | 类型 | 必需 | 值 | 备注 | 说明 |
|---|---|---:|---|---|---|
| `pi.ai.operation` | `string` | yes | stream, fetch_deferred, cancel_deferred, generate_images |  | 逻辑 Provider 操作 |
| `pi.ai.provider` | `string` | yes |  |  | 选定的 Provider id |
| `pi.ai.model` | `string` | yes |  |  | 请求的 model id |
| `pi.ai.api` | `string` | yes |  |  | Provider API id |
| `pi.ai.streaming` | `boolean` | yes |  |  | 此操作是否返回 stream |
| `pi.ai.deferred` | `boolean` | no |  |  | 此操作是否请求或参与 deferred execution |

#### 结束属性

所有结束属性都是可选的完成信息补充。

| 名称 | 类型 | 值 | 备注 | 说明 |
|---|---|---|---|---|
| `pi.ai.response.model` | `string` |  |  | 实际响应 model |
| `pi.ai.response.id` | `string` |  | 高基数 | Provider 响应 id |
| `pi.ai.response.stop_reason` | `string` | stop, length, tool_use, error, aborted, deferred |  | 规范化的终止响应原因 |
| `pi.ai.http.status_code` | `number` |  |  | 最终 HTTP 状态 |
| `pi.ai.usage.input_tokens` | `number` |  |  | 报告的 input token 数 |
| `pi.ai.usage.output_tokens` | `number` |  |  | 报告的 output token 数 |
| `pi.ai.usage.cache_read_tokens` | `number` |  |  | 报告的 cache-read token 数 |
| `pi.ai.usage.cache_write_tokens` | `number` |  |  | 报告的 cache-write token 数 |
| `pi.ai.usage.reasoning_tokens` | `number` |  |  | 报告的 reasoning token 数 |
| `pi.ai.usage.total_tokens` | `number` |  |  | 报告的 token 总数 |
| `pi.ai.usage.cost` | `number` |  |  | 报告的总成本 |
| `pi.ai.stream.chunk_count` | `number` |  |  | 流式更新 chunk 数 |
| `pi.ai.stream.time_to_first_chunk_ms` | `number` |  |  | 首个更新 chunk 前经过的毫秒数 |
| `pi.ai.error.type` | `string` |  | 低基数 | Provider 或传输错误类别 |

#### 事件

未声明 span 事件。
## Harness Schema

Schema 版本：1

### `pi.harness.run`

一次已准入的进程内 run 调用

- Parent：root 或调用方拥有的外部 span
- 默认状态：`ok`
- 错误条件：run 失败或抛出异常

#### 开始属性

| 名称 | 类型 | 必需 | 值 | 备注 | 说明 |
|---|---|---:|---|---|---|
| `pi.session.id` | `string` | yes |  | 高基数 | Session id |
| `pi.lane.name` | `string` | yes |  | 高基数 | Lane 名称 |
| `pi.operation.id` | `string` | yes |  | 高基数 | 持久化 operation id |
| `pi.operation.recovery` | `boolean` | yes |  |  | 此次调用是否恢复持久化工作 |
| `pi.operation.kind` | `string` | yes | run |  | Run operation 类型 |

#### 结束属性

所有结束属性都是可选的完成信息补充。

| 名称 | 类型 | 值 | 备注 | 说明 |
|---|---|---|---|---|
| `pi.operation.outcome` | `string` | completed, aborted, failed, suspended |  | Run 调用结果 |
| `pi.error.code` | `string` |  | 低基数 | 稳定的 operation 错误码 |
| `pi.error.type` | `string` |  | 低基数 | 低基数 operation 错误类别 |

#### 事件

未声明 span 事件。

### `pi.harness.compaction`

一次已准入的进程内手动 compaction 调用

- Parent：root 或调用方拥有的外部 span
- 默认状态：`ok`
- 错误条件：compaction 失败或抛出异常

#### 开始属性

| 名称 | 类型 | 必需 | 值 | 备注 | 说明 |
|---|---|---:|---|---|---|
| `pi.session.id` | `string` | yes |  | 高基数 | Session id |
| `pi.lane.name` | `string` | yes |  | 高基数 | Lane 名称 |
| `pi.operation.id` | `string` | yes |  | 高基数 | 持久化 operation id |
| `pi.operation.recovery` | `boolean` | yes |  |  | 此次调用是否恢复持久化工作 |
| `pi.operation.kind` | `string` | yes | compaction |  | Compaction operation 类型 |

#### 结束属性

所有结束属性都是可选的完成信息补充。

| 名称 | 类型 | 值 | 备注 | 说明 |
|---|---|---|---|---|
| `pi.operation.outcome` | `string` | completed, declined, aborted, failed |  | Compaction 调用结果 |
| `pi.error.code` | `string` |  | 低基数 | 稳定的 operation 错误码 |
| `pi.error.type` | `string` |  | 低基数 | 低基数 operation 错误类别 |

#### 事件

未声明 span 事件。

### `pi.harness.navigation`

一次已准入的进程内 navigation 调用

- Parent：root 或调用方拥有的外部 span
- 默认状态：`ok`
- 错误条件：navigation 失败或抛出异常

#### 开始属性

| 名称 | 类型 | 必需 | 值 | 备注 | 说明 |
|---|---|---:|---|---|---|
| `pi.session.id` | `string` | yes |  | 高基数 | Session id |
| `pi.lane.name` | `string` | yes |  | 高基数 | Lane 名称 |
| `pi.operation.id` | `string` | yes |  | 高基数 | 持久化 operation id |
| `pi.operation.recovery` | `boolean` | yes |  |  | 此次调用是否恢复持久化工作 |
| `pi.operation.kind` | `string` | yes | navigation |  | Navigation operation 类型 |

#### 结束属性

所有结束属性都是可选的完成信息补充。

| 名称 | 类型 | 值 | 备注 | 说明 |
|---|---|---|---|---|
| `pi.operation.outcome` | `string` | completed, declined, aborted, failed |  | Navigation 调用结果 |
| `pi.error.code` | `string` |  | 低基数 | 稳定的 operation 错误码 |
| `pi.error.type` | `string` |  | 低基数 | 低基数 operation 错误类别 |

#### 事件

未声明 span 事件。

### `pi.harness.checkpoint`

一个 run checkpoint

- Parent：`pi.harness.run`
- 默认状态：`ok`
- 错误条件：Checkpoint 工作抛出异常

#### 开始属性

| 名称 | 类型 | 必需 | 值 | 备注 | 说明 |
|---|---|---:|---|---|---|
| `pi.lane.name` | `string` | yes |  | 高基数 | Lane 名称 |
| `pi.operation.id` | `string` | yes |  | 高基数 | 持久化 operation id |
| `pi.checkpoint.kind` | `string` | yes | normal, failure_drain, abort_reconcile |  | Checkpoint 用途 |

#### 结束属性

所有结束属性都是可选的完成信息补充。

| 名称 | 类型 | 值 | 备注 | 说明 |
|---|---|---|---|---|
| _无_ | | | | |

#### 事件

未声明 span 事件。

### `pi.harness.turn`

一次 assistant 响应及其 tool batch

- Parent：`pi.harness.run`
- 默认状态：`ok`
- 错误条件：Turn 工作抛出异常

#### 开始属性

| 名称 | 类型 | 必需 | 值 | 备注 | 说明 |
|---|---|---:|---|---|---|
| `pi.lane.name` | `string` | yes |  | 高基数 | Lane 名称 |
| `pi.operation.id` | `string` | yes |  | 高基数 | 持久化 operation id |
| `pi.turn.id` | `string` | yes |  | 高基数 | 调用本地 turn id |

#### 结束属性

所有结束属性都是可选的完成信息补充。

| 名称 | 类型 | 值 | 备注 | 说明 |
|---|---|---|---|---|
| _无_ | | | | |

#### 事件

未声明 span 事件。

### `pi.harness.step`

一次持久化重试 attempt

- Parent：`pi.harness.turn`、`pi.harness.checkpoint`、`pi.harness.compaction`、`pi.harness.navigation`
- 默认状态：`ok`
- 错误条件：attempt 重试、失败或抛出异常

#### 开始属性

| 名称 | 类型 | 必需 | 值 | 备注 | 说明 |
|---|---|---:|---|---|---|
| `pi.lane.name` | `string` | yes |  | 高基数 | Lane 名称 |
| `pi.operation.id` | `string` | yes |  | 高基数 | 持久化 operation id |
| `pi.step.kind` | `string` | yes | assistant, compaction, branch_summary |  | 可重试 step 类型 |
| `pi.step.attempt` | `number` | yes |  |  | 从 1 开始的持久化 attempt 编号 |
| `pi.compaction.reason` | `string` | no | manual, threshold, overflow |  | Compaction 触发原因 |

#### 结束属性

所有结束属性都是可选的完成信息补充。

| 名称 | 类型 | 值 | 备注 | 说明 |
|---|---|---|---|---|
| `pi.step.outcome` | `string` | succeeded, retry, failed, aborted, deferred, overflow |  | Attempt 结果 |

#### 事件

未声明 span 事件。

### `pi.harness.tool`

一次原始 phase-2 tool 执行

- Parent：`pi.harness.turn`、`pi.harness.run`
- 默认状态：`ok`
- 错误条件：原始 phase-2 执行返回错误

#### 开始属性

| 名称 | 类型 | 必需 | 值 | 备注 | 说明 |
|---|---|---:|---|---|---|
| `pi.lane.name` | `string` | yes |  | 高基数 | Lane 名称 |
| `pi.operation.id` | `string` | yes |  | 高基数 | 持久化 operation id |
| `pi.turn.id` | `string` | no |  | 高基数 | 调用本地的实时 turn id |
| `pi.tool.name` | `string` | yes |  |  | Tool 名称 |
| `pi.tool.call_id` | `string` | yes |  | 高基数 | Tool call id |
| `pi.tool.replay` | `string` | yes | never, safe |  | 声明的 replay 策略 |
| `pi.tool.recovery` | `boolean` | yes |  |  | 此次是否为 recovery 执行 |

#### 结束属性

所有结束属性都是可选的完成信息补充。

| 名称 | 类型 | 值 | 备注 | 说明 |
|---|---|---|---|---|
| `pi.tool.is_error` | `boolean` |  |  | 原始 phase-2 执行是否返回错误 |

#### 事件

未声明 span 事件。

### `pi.harness.hook`

一次已注册 hook handler 调用

- Parent：root 或任意调用方 span
- 默认状态：`ok`
- 错误条件：handler 抛出异常

#### 开始属性

| 名称 | 类型 | 必需 | 值 | 备注 | 说明 |
|---|---|---:|---|---|---|
| `pi.lane.name` | `string` | yes |  | 高基数 | Lane 名称 |
| `pi.operation.id` | `string` | no |  | 高基数 | 持久化 operation id when accepted |
| `pi.hook.name` | `string` | yes | before_run, before_resume, before_run_end, transform_context, before_request, before_payload, after_response, before_tool, after_tool, before_compaction, before_navigation |  | Hook 名称 |
| `pi.hook.registration_id` | `string` | no |  |  | 稳定的 hook 注册 id |

#### 结束属性

所有结束属性都是可选的完成信息补充。

| 名称 | 类型 | 值 | 备注 | 说明 |
|---|---|---|---|---|
| `pi.hook.outcome` | `string` | completed, skipped, blocked, failed |  | Handler 结果 |

#### 事件

未声明 span 事件。

### `pi.harness.sleep`

一次重试延迟

- Parent：`pi.harness.step`、`pi.harness.run`
- 默认状态：`ok`
- 错误条件：Sleep 工作抛出异常

#### 开始属性

| 名称 | 类型 | 必需 | 值 | 备注 | 说明 |
|---|---|---:|---|---|---|
| `pi.operation.id` | `string` | yes |  | 高基数 | 持久化 operation id |
| `pi.sleep.delay_ms` | `number` | yes |  |  | 请求的延迟毫秒数 |

#### 结束属性

所有结束属性都是可选的完成信息补充。

| 名称 | 类型 | 值 | 备注 | 说明 |
|---|---|---|---|---|
| `pi.sleep.outcome` | `string` | elapsed, aborted |  | 延迟结果 |

#### 事件

未声明 span 事件。

### `pi.harness.event_handler`

一次被动 event listener 调用

- Parent：root 或任意调用方 span
- 默认状态：`ok`
- 错误条件：listener 抛出异常

#### 开始属性

| 名称 | 类型 | 必需 | 值 | 备注 | 说明 |
|---|---|---:|---|---|---|
| `pi.event.type` | `string` | yes | run_start, run_resume, run_suspend, run_abort, run_end, fault, handler_error, turn_start, turn_end, retry_scheduled, retry_start, retry_end, message_start, message_update, message_end, tool_start, tool_update, tool_end, entry_added, write_pending, queue_update, fact_update, config_update, compaction_start, compaction_end, navigation_start, navigation_end, lane_created, usage | 低基数 | 已投递的 Harness event 类型 |
| `pi.lane.name` | `string` | no |  | 高基数 | Lane 名称 for lane-scoped events |

#### 结束属性

所有结束属性都是可选的完成信息补充。

| 名称 | 类型 | 值 | 备注 | 说明 |
|---|---|---|---|---|
| _无_ | | | | |

#### 事件

未声明 span 事件。

### `pi.session.write`

一次已提交的 Session 变更

- Parent：root 或任意调用方 span
- 默认状态：`ok`
- 错误条件：Storage 拒绝该变更

#### 开始属性

| 名称 | 类型 | 必需 | 值 | 备注 | 说明 |
|---|---|---:|---|---|---|
| `pi.lane.name` | `string` | yes |  | 高基数 | Lane 名称 |
| `pi.operation.id` | `string` | no |  | 高基数 | 持久化 operation id when accepted |
| `pi.session.mutation` | `string` | yes | entry, record, lane, fact |  | Session 变更类型 |
| `pi.session.item_type` | `string` | no |  |  | Entry、record、lane 或 fact 子类型 |

#### 结束属性

所有结束属性都是可选的完成信息补充。

| 名称 | 类型 | 值 | 备注 | 说明 |
|---|---|---|---|---|
| `pi.session.seq` | `number` |  |  | 公开时的已提交 Session sequence |

#### 事件

未声明 span 事件。
