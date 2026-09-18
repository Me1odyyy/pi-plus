# @earendil-works/pi-ai

统一的 LLM（大语言模型）API，提供提供商集合、自动身份验证解析、令牌与成本跟踪，以及简单的上下文持久化和会话中途向其他模型的移交。

**注意**：此库仅包含支持工具调用（函数调用）的模型，因为这是智能体工作流不可或缺的能力。

## 目录

- [支持的提供商](#支持的提供商)
- [安装](#安装)
- [快速开始](#快速开始)
- [提供商与模型](#提供商与模型)
  - [提供商工厂](#提供商工厂)
  - [所有内置提供商](#所有内置提供商)
  - [查询模型](#查询模型)
  - [读取静态目录](#读取静态目录)
  - [动态提供商](#动态提供商)
- [身份验证](#身份验证)
  - [身份验证的解析方式](#身份验证的解析方式)
  - [转换请求头](#转换请求头)
  - [凭据存储](#凭据存储)
  - [环境变量](#环境变量)
- [工具](#工具)
  - [定义工具](#定义工具)
  - [处理工具调用](#处理工具调用)
  - [通过部分 JSON 流式传输工具调用](#通过部分-json-流式传输工具调用)
  - [验证工具参数](#验证工具参数)
  - [完整事件参考](#完整事件参考)
- [图像输入](#图像输入)
- [图像生成](#图像生成)
- [思考/推理](#思考推理)
  - [统一接口（streamSimple/completeSimple）](#统一接口streamsimplecompletesimple)
  - [提供商专用选项（stream/complete）](#提供商专用选项streamcomplete)
  - [流式传输思考内容](#流式传输思考内容)
- [停止原因](#停止原因)
- [错误处理](#错误处理)
  - [中止请求](#中止请求)
  - [中止后继续](#中止后继续)
  - [调试提供商载荷](#调试提供商载荷)
- [自定义提供商](#自定义提供商)
  - [createProvider()](#createprovider)
  - [直接调用 API 实现](#直接调用-api-实现)
  - [OpenAI 兼容性设置](#openai-兼容性设置)
- [用于测试的仿真提供商](#用于测试的仿真提供商)
- [跨提供商移交](#跨提供商移交)
- [上下文序列化](#上下文序列化)
- [浏览器用法](#浏览器用法)
- [打包与摇树优化](#打包与摇树优化)
- [OAuth 提供商](#oauth-提供商)
  - [Vertex AI](#vertex-ai)
  - [CLI 登录](#cli-登录)
  - [以编程方式使用 OAuth](#以编程方式使用-oauth)
- [从旧版全局 API 迁移](#从旧版全局-api-迁移)
- [开发](#开发)
- [许可证](#许可证)

## 支持的提供商

- **OpenAI**
- **Ant Ling**
- **Azure OpenAI (Responses)**
- **OpenAI Codex**（ChatGPT Plus/Pro 订阅，需要 OAuth，见下文）
- **DeepSeek**
- **NVIDIA NIM**
- **Anthropic**
- **Google**
- **Vertex AI**（通过 Vertex AI 使用 Gemini）
- **Mistral**
- **Groq**
- **Cerebras**
- **Cloudflare AI Gateway**
- **Cloudflare Workers AI**
- **xAI**
- **OpenRouter**
- **Vercel AI Gateway**
- **ZAI Coding Plan (Global)**（另有独立的中国区提供商）
- **MiniMax**（另有独立的中国区提供商）
- **Together AI**
- **Baseten**
- **Hugging Face**
- **Moonshot AI**（另有独立的中国区提供商）
- **GitHub Copilot**（需要 OAuth，见下文）
- **Amazon Bedrock**
- **OpenCode Zen**
- **OpenCode Go**
- **Fireworks**（使用与 OpenAI 和 Anthropic 兼容的 API）
- **Kimi For Coding**（Moonshot AI 订阅端点，使用与 Anthropic 兼容的 API）
- **Qwen Token Plan**（Individual 与现有目录分开，另有独立的中国区提供商）
- **Xiaomi MiMo**（默认使用 API 计费端点，并为 `cn`/`ams`/`sgp` 区域提供独立的 Token Plan 提供商）
- **任何与 OpenAI 兼容的 API**：Ollama、vLLM、LM Studio 等。

## 安装

```bash
npm install @earendil-works/pi-ai
```

此包会从 `@earendil-works/pi-ai` 重新导出 TypeBox 的导出项：`Type`、`Static` 和 `TSchema`。

## 快速开始

先构建一个由多个提供商组成的 `Models` 集合，再通过它进行流式传输。最快的入门方式是注册所有内置提供商；注重包体积的应用应改为注册单独的提供商（参见[提供商工厂](#提供商工厂)和[打包与摇树优化](#打包与摇树优化)）。

```typescript
import { Type, type Context, type Tool } from '@earendil-works/pi-ai';
import { builtinModels } from '@earendil-works/pi-ai/providers/all';

// A Models collection with every built-in provider registered
const models = builtinModels();

// Sync lookup against the collection
const model = models.getModel('openai', 'gpt-4o-mini')!;

// Define tools with TypeBox schemas for type safety and validation
const tools: Tool[] = [{
  name: 'get_time',
  description: 'Get the current time',
  parameters: Type.Object({
    timezone: Type.Optional(Type.String({ description: 'Optional timezone (e.g., America/New_York)' }))
  })
}];

// Build a conversation context (easily serializable and transferable between models)
const context: Context = {
  systemPrompt: 'You are a helpful assistant.',
  messages: [{ role: 'user', content: 'What time is it?', timestamp: Date.now() }],
  tools
};

// Option 1: Streaming with all event types.
// Auth resolves through the provider (OPENAI_API_KEY from the environment here).
const s = models.stream(model, context);

for await (const event of s) {
  switch (event.type) {
    case 'start':
      console.log(`Starting with ${event.partial.model}`);
      break;
    case 'text_start':
      console.log('\n[Text started]');
      break;
    case 'text_delta':
      process.stdout.write(event.delta);
      break;
    case 'text_end':
      console.log('\n[Text ended]');
      break;
    case 'thinking_start':
      console.log('[Model is thinking...]');
      break;
    case 'thinking_delta':
      process.stdout.write(event.delta);
      break;
    case 'thinking_end':
      console.log('[Thinking complete]');
      break;
    case 'toolcall_start':
      console.log(`\n[Tool call started: index ${event.contentIndex}]`);
      break;
    case 'toolcall_delta':
      // Partial tool arguments are being streamed
      const partialCall = event.partial.content[event.contentIndex];
      if (partialCall.type === 'toolCall') {
        console.log(`[Streaming args for ${partialCall.name}]`);
      }
      break;
    case 'toolcall_end':
      console.log(`\nTool called: ${event.toolCall.name}`);
      console.log(`Arguments: ${JSON.stringify(event.toolCall.arguments)}`);
      break;
    case 'done':
      console.log(`\nFinished: ${event.reason}`);
      break;
    case 'error':
      console.error(`Error: ${event.error.errorMessage}`);
      break;
  }
}

// Get the final message after streaming, add it to the context
const finalMessage = await s.result();
context.messages.push(finalMessage);

// Handle tool calls if any
const toolCalls = finalMessage.content.filter(b => b.type === 'toolCall');
for (const call of toolCalls) {
  const result = call.name === 'get_time'
    ? new Date().toLocaleString('en-US', {
        timeZone: call.arguments.timezone || 'UTC',
        dateStyle: 'full',
        timeStyle: 'long'
      })
    : 'Unknown tool';

  // Add tool result to context (supports text and images)
  context.messages.push({
    role: 'toolResult',
    toolCallId: call.id,
    toolName: call.name,
    content: [{ type: 'text', text: result }],
    isError: false,
    timestamp: Date.now()
  });
}

// Continue if there were tool calls
if (toolCalls.length > 0) {
  const continuation = await models.complete(model, context);
  context.messages.push(continuation);
  console.log('After tool execution:', continuation.content);
}

console.log(`Total tokens: ${finalMessage.usage.input} in, ${finalMessage.usage.output} out`);
console.log(`Cost: $${finalMessage.usage.cost.total.toFixed(4)}`);

// Option 2: Get complete response without streaming
const response = await models.complete(model, context);

for (const block of response.content) {
  if (block.type === 'text') {
    console.log(block.text);
  } else if (block.type === 'toolCall') {
    console.log(`Tool: ${block.name}(${JSON.stringify(block.arguments)})`);
  }
}
```

本 README 余下部分的代码片段假定已经按上述方式设置了 `models` 集合（并注册了相关提供商）。

## 提供商与模型

**提供商（provider）**是运行时单元：它拥有自己的模型目录、身份验证机制（API 密钥解析、OAuth 流程）和流式传输行为。`Models` 集合容纳提供商，并将每个请求路由到拥有相应模型的提供商。

提供商在内部共享 **API 实现**（线路协议）：Anthropic 模型使用 `anthropic-messages`，OpenAI 使用 `openai-responses`，而 xAI、Groq、Cerebras、OpenRouter 及大多数其他提供商共享 `openai-completions`。使用混合 API 的提供商（GitHub Copilot、OpenCode Zen）会根据模型进行分派。

### 提供商工厂

对于只需要特定提供商的应用，每个内置提供商都有一个工厂；每个工厂都通过一个子路径导入，并且只引入该提供商的目录：

```typescript
import { anthropicProvider } from '@earendil-works/pi-ai/providers/anthropic';
import { openaiProvider } from '@earendil-works/pi-ai/providers/openai';
import { openrouterProvider } from '@earendil-works/pi-ai/providers/openrouter';
import { amazonBedrockProvider } from '@earendil-works/pi-ai/providers/amazon-bedrock';
// ...one module per provider in the Supported Providers list

const models = createModels();
models.setProvider(anthropicProvider());
models.setProvider(openrouterProvider());
```

提供商工厂会导入其模型目录和一个惰性 API 包装器，而不会导入其他提供商。启用打包器代码拆分后，各 SDK 实现（`@anthropic-ai/sdk`、`openai`、`@google/genai` 等）会保留在惰性代码块中，直到第一次向采用相应 API 的模型发出请求时才加载。

### 所有内置提供商

对于需要全部提供商的应用（如“快速开始”所示）：

```typescript
import { builtinModels } from '@earendil-works/pi-ai/providers/all';

const models = builtinModels(); // a Models collection with every built-in provider registered
```

这会导入所有目录和每个内置提供商工厂。它是重量级的显式入口点。`builtinModels()` 接受与 `createModels()` 相同的选项（`credentials`、`authContext`）；如果想在自己的集合上注册提供商，`builtinProviders()` 会返回提供商数组。

### 查询模型

读取操作是同步的，并返回最近一次已知的列表：

```typescript
const providers = models.getProviders();           // registered Provider objects
const provider = models.getProvider('anthropic');  // one provider

const all = models.getModels();                    // every model across providers
const anthropicModels = models.getModels('anthropic');
const model = models.getModel('anthropic', 'claude-sonnet-4-5');

for (const m of anthropicModels) {
  console.log(`${m.id}: ${m.name}`);
  console.log(`  API: ${m.api}`);
  console.log(`  Context: ${m.contextWindow} tokens`);
  console.log(`  Vision: ${m.input.includes('image')}`);
  console.log(`  Reasoning: ${m.reasoning}`);
}
```

动态列出的模型类型为 `Model<Api>`。需要 API 专用的选项类型时，请使用 `hasApi()` 类型守卫缩窄类型：

```typescript
import { hasApi } from '@earendil-works/pi-ai';

const m = models.getModel('anthropic', 'claude-sonnet-4-5');
if (m && hasApi(m, 'anthropic-messages')) {
  // m: Model<'anthropic-messages'> — stream options fully typed
  models.stream(m, context, { thinkingEnabled: true, thinkingBudgetTokens: 2048 });
}
```

### 读取静态目录

如果工具需要独立于任何集合、且具有完整字面量类型（提供商和模型 ID 自动补全）的已生成内置目录，可以使用：

```typescript
import { getBuiltinModel, getBuiltinModels, getBuiltinProviders } from '@earendil-works/pi-ai/providers/all';

const model = getBuiltinModel('openai', 'gpt-4o-mini'); // typed Model<'openai-responses'>
const providers = getBuiltinProviders();
const anthropic = getBuiltinModels('anthropic');
```

### 动态提供商

提供商可以拥有动态模型列表（例如 llama.cpp 服务器或实时 OpenRouter 列表）。读取仍保持同步；获取数据则是显式的异步操作：

```typescript
// getModels() returns the last-known list (empty before the first refresh)
await models.refresh({ providers: ['llamacpp'] }); // refresh one provider
await models.refresh();                            // refresh all providers concurrently, best-effort
const fresh = models.getModel('llamacpp', 'qwen3-30b');
```

对于静态的内置提供商，`refresh()` 不执行任何操作。有关构建动态提供商的说明，请参阅 [createProvider()](#createprovider)。

## 身份验证

每个提供商都拥有自己的身份验证机制：如何解析 API 密钥（已存储凭据、环境变量、AWS 配置文件或 gcloud ADC 等环境凭据来源），以及在支持时如何执行 OAuth 登录/刷新流程。

### 身份验证的解析方式

调用 `models.stream()` 时，集合会通过相应提供商解析身份验证信息，并将其合并到请求中。每个请求显式提供的值始终具有最高优先级：

```typescript
// Resolved through the provider (env var, stored credential, OAuth token):
await models.complete(model, context);

// Explicit key wins over anything the provider would resolve:
await models.complete(model, context, { apiKey: 'sk-explicit' });
```

无需发起请求也可以检查解析结果。传入提供商 ID 可进行提供商范围的身份验证；传入模型还会包含其静态 `model.headers`：

```typescript
const providerAuth = await models.getAuth(model.provider);
const modelAuth = await models.getAuth(model);

if (modelAuth) {
  console.log(`configured via ${modelAuth.source}`); // e.g. "ANTHROPIC_API_KEY", "OAuth", "stored credential"
  console.log(modelAuth.auth.headers);              // Provider auth headers + model.headers
} else {
  console.log('not configured');
}
```

两个重载都会解析凭据、在必要时刷新过期的 OAuth，并可能返回身份验证衍生的 `apiKey`、`headers` 或 `baseUrl`。未配置的提供商会让 `getAuth()` 解析为 `undefined`；实际发生故障时，则会以 `ModelsError` 拒绝（`"oauth"`：令牌刷新失败，并保留凭据以便重新登录；`"auth"`：密钥解析或凭据存储失败）。请求路径会以流错误的形式呈现同样的故障。

`getAuth()`、`checkAuth()`、`getAvailable()`、登录和注销都通过各自已有的选项或交互对象接受可选的调用方取消信号；未提供信号时不会设置时限。提供商的 `login`、`ApiKeyAuth.check`、`ApiKeyAuth.resolve` 和 `OAuthAuth.refresh` 实现始终会收到具体信号，并且必须在阻塞操作中遵守该信号。

### 转换请求头

`Models.stream()`、`complete()`、`streamSimple()` 和 `completeSimple()` 接受一个仅供 Models 使用的 `transformHeaders` 选项。它会在提供商身份验证信息、`model.headers` 和显式的 `options.headers` 合并后、提供商分派前运行一次：

```typescript
const response = await models.completeSimple(model, context, {
  headers: { "X-Client": "my-app" },
  transformHeaders: async (headers) => ({
    ...headers,
    "X-Request-ID": crypto.randomUUID(),
  }),
});
```

顺序如下：

```text
provider auth headers -> model.headers -> explicit options.headers -> transformHeaders -> Provider.stream*()
```

请求头名称按不区分大小写的方式合并。显式请求头会覆盖身份验证/模型请求头，而转换函数拥有最终控制权；为某个请求头返回 `null` 会抑制支持删除操作的下层默认值。

`transformHeaders` 属于 `Models`，而不属于 `Provider`。`Models` 实现必须使用该选项，并在调用 `Provider.stream*()` 前将其移除。提供商实现仍只接收普通的 `ApiStreamOptions` 或 `SimpleStreamOptions`，绝不会自行处理该转换。应使用此选项，而不是先调用 `getAuth(model)` 再调用 `stream*()`，否则请求身份验证会被解析两次。

### 凭据存储

已存储的凭据（交互式输入的 API 密钥、OAuth 令牌）位于 `CredentialStore` 中——每个提供商有一个带类型标签的凭据。pi-ai 自带基于内存的默认实现；应用可以注入持久化存储：

```typescript
import { createModels, type CredentialStore } from '@earendil-works/pi-ai';

const models = createModels({ credentials: myFileBackedStore });
// builtinModels() takes the same options:
// const models = builtinModels({ credentials: myFileBackedStore });
```

该契约很精简：`read(providerId)`；`list()` 用于获取非敏感的 `{ providerId, type }` 元数据；作为唯一写入路径、执行串行化“读取-修改-写入”的 `modify(providerId, fn)`；以及 `delete(providerId)`。每个操作都接受可选的取消选项。枚举不得解析机密信息或执行已配置的密钥命令。OAuth 令牌刷新在 `modify` 内运行，因此并发请求和进程不会对轮换后的令牌重复刷新。已存储凭据会*接管*其提供商：仅在没有存储任何内容时才查询环境变量，而且刷新失败时绝不会悄悄回退到环境变量密钥。

API 密钥凭据使用与 pi 的 `auth.json` 相同的判别字段，并且可以携带提供商范围的环境/配置值：

```typescript
const credential = {
  type: 'api_key',
  key: '...',
  env: {
    CLOUDFLARE_ACCOUNT_ID: 'account-id',
    CLOUDFLARE_GATEWAY_ID: 'gateway-id'
  }
} as const;
```

### 环境变量

内置提供商会解析以下环境变量（Node.js；在浏览器中请显式传入 `apiKey`）：

| 提供商 | 环境变量 |
|----------|------------------------|
| OpenAI | `OPENAI_API_KEY` |
| Ant Ling | `ANT_LING_API_KEY` |
| Azure OpenAI | `AZURE_OPENAI_API_KEY` + `AZURE_OPENAI_BASE_URL`（例如 `https://{resource}.ai.azure.com`）或 `AZURE_OPENAI_RESOURCE_NAME`。支持 `*.openai.azure.com`、`*.cognitiveservices.azure.com` 和 `*.ai.azure.com`；根端点会自动规范化为 `/openai/v1`。可选：`AZURE_OPENAI_API_VERSION`（默认 `v1`）、`AZURE_OPENAI_DEPLOYMENT_NAME_MAP`。 |
| Anthropic | `ANTHROPIC_API_KEY` 或 `ANTHROPIC_OAUTH_TOKEN` |
| DeepSeek | `DEEPSEEK_API_KEY` |
| NVIDIA NIM | `NVIDIA_API_KEY` |
| Google | `GEMINI_API_KEY` |
| Vertex AI | `GOOGLE_CLOUD_API_KEY`，或 `GOOGLE_CLOUD_PROJECT`（也可用 `GCLOUD_PROJECT`）+ `GOOGLE_CLOUD_LOCATION` + ADC |
| Mistral | `MISTRAL_API_KEY` |
| Groq | `GROQ_API_KEY` |
| Cerebras | `CEREBRAS_API_KEY` |
| Cloudflare AI Gateway | `CLOUDFLARE_API_KEY` + `CLOUDFLARE_ACCOUNT_ID` + `CLOUDFLARE_GATEWAY_ID` |
| Cloudflare Workers AI | `CLOUDFLARE_API_KEY` + `CLOUDFLARE_ACCOUNT_ID` |
| xAI | `XAI_API_KEY` |
| Fireworks | `FIREWORKS_API_KEY` |
| Together AI | `TOGETHER_API_KEY` |
| Baseten | `BASETEN_API_KEY` |
| OpenRouter | `OPENROUTER_API_KEY` |
| Vercel AI Gateway | `AI_GATEWAY_API_KEY` |
| ZAI Coding Plan (Global) | `ZAI_API_KEY` |
| ZAI Coding Plan (China) | `ZAI_CODING_CN_API_KEY` |
| MiniMax (Global) | `MINIMAX_API_KEY` |
| MiniMax (China) | `MINIMAX_CN_API_KEY` |
| Moonshot AI / Moonshot AI (China) | `MOONSHOT_API_KEY` |
| Hugging Face | `HF_TOKEN` |
| OpenCode Zen / OpenCode Go | `OPENCODE_API_KEY` |
| Kimi For Coding | `KIMI_API_KEY` |
| Qwen Token Plan（现有目录） | `QWEN_TOKEN_PLAN_API_KEY` |
| Qwen Token Plan (Individual) | `QWEN_TOKEN_PLAN_API_KEY` |
| Qwen Token Plan (China) | `QWEN_TOKEN_PLAN_CN_API_KEY` |
| Xiaomi MiMo（API 计费） | `XIAOMI_API_KEY` |
| Xiaomi MiMo Token Plan (China) | `XIAOMI_TOKEN_PLAN_CN_API_KEY` |
| Xiaomi MiMo Token Plan (Amsterdam) | `XIAOMI_TOKEN_PLAN_AMS_API_KEY` |
| Xiaomi MiMo Token Plan (Singapore) | `XIAOMI_TOKEN_PLAN_SGP_API_KEY` |
| GitHub Copilot | `COPILOT_GITHUB_TOKEN` |

`qwen-token-plan-individual` 和 `qwen-token-plan` 共用国际端点及
`QWEN_TOKEN_PLAN_API_KEY`。Individual 提供商仅公开 Individual
订阅文档中列出的模型，而现有提供商为保持向后兼容而保留更广泛的目录。
已存储凭据仍按提供商划分，因此请将密钥保存到你所注册的提供商 ID 下。

Amazon Bedrock 会解析环境中的 AWS 凭据（`AWS_PROFILE`、访问密钥对、`AWS_BEARER_TOKEN_BEDROCK`、ECS 任务角色、Web 身份令牌）；其由提供商管理的登录流程支持持有者令牌、AWS 配置文件和现有凭据链。Vertex AI 会解析显式密钥，或 gcloud Application Default Credentials（应用默认凭据，ADC）以及项目/位置；它还为 API 密钥、ADC 和服务账号文件提供由提供商管理的登录流程。

## 工具

工具让 LLM 能够与外部系统交互。此库使用 TypeBox 架构定义类型安全的工具，并通过 TypeBox 内置的验证器和值转换实用程序执行自动验证。TypeBox 架构可以作为普通 JSON 进行序列化和反序列化，因此非常适合分布式系统。

### 定义工具

```typescript
import { Type, type Tool, StringEnum } from '@earendil-works/pi-ai';

// Define tool parameters with TypeBox
const weatherTool: Tool = {
  name: 'get_weather',
  description: 'Get current weather for a location',
  parameters: Type.Object({
    location: Type.String({ description: 'City name or coordinates' }),
    units: StringEnum(['celsius', 'fahrenheit'], { default: 'celsius' })
  })
};

// Note: For Google API compatibility, use StringEnum helper instead of Type.Enum
// Type.Enum generates anyOf/const patterns that Google doesn't support

const bookMeetingTool: Tool = {
  name: 'book_meeting',
  description: 'Schedule a meeting',
  parameters: Type.Object({
    title: Type.String({ minLength: 1 }),
    startTime: Type.String({ format: 'date-time' }),
    endTime: Type.String({ format: 'date-time' }),
    attendees: Type.Array(Type.String({ format: 'email' }), { minItems: 1 })
  })
};
```

### 工具的约束采样

工具可以选择使用提供商侧的约束采样。对于 JSON 架构工具，`strict: 'prefer'` 会在支持时使用提供商侧的严格架构强制执行，否则回退到普通工具调用。`strict: 'require'` 会在当前提供商/模型无法满足要求时让请求失败。设置 `constrainedSampling: false` 可显式退出，其行为与省略该字段相同。

```typescript
const strictTool: Tool = {
  name: 'edit_file',
  description: 'Edit a file',
  parameters: Type.Object({
    path: Type.String(),
    content: Type.String()
  }, { additionalProperties: false }),
  constrainedSampling: { type: 'json_schema', strict: 'prefer' }
};
```

OpenAI、Anthropic、受支持的 Amazon Bedrock Converse 模型、Mistral，以及通过 Google Generative AI 和 Vertex 适配器使用的 Gemini 3 工具调用，都支持严格 JSON 架构约束采样。Google 使用 `VALIDATED` 函数调用模式（显式请求时也可使用 `ANY`）；较早的 Gemini 版本会为 `strict: 'prefer'` 回退，并拒绝 `strict: 'require'`，因为它们不会强制执行必需参数。Bedrock 的严格工具能力根据模型的结构化输出元数据生成；自定义 Bedrock 模型可以覆盖 `compat.supportsStrictMode`。OpenAI Responses 和 Chat Completions 还可以通过 OpenAI Lark 或正则表达式语法变体输出受语法约束的自定义工具。如果提供多个 OpenAI 变体，Lark 的优先级高于正则表达式。当当前模型支持语法工具时会强制执行语法约束；否则工具会回退到普通函数/JSON 架构处理。语法工具能力属于模型元数据：生成的目录会为能够透传 OpenAI 自定义工具的端点上的 GPT-5+ 模型（OpenAI、OpenAI Codex、Azure OpenAI Responses、GitHub Copilot、opencode 和 Cloudflare AI Gateway）设置 `compat.supportsOpenAIGrammarTools`。OpenAI 会拒绝用于 GPT-5 之前模型的 `type: "custom"` 工具，而会规范化工具架构的网关（例如 OpenRouter）会破坏这类工具，因此该标志在其他地方保持关闭。自定义模型定义可以通过 `compat` 选择启用。支持语法的模型会拒绝没有任何非空受支持变体的语法配置。原生语法工具必须具有对象参数架构，其中恰好有一个必需的字符串属性：

```typescript
const patchTool: Tool = {
  name: 'apply_patch',
  description: 'Apply a patch',
  parameters: Type.Object({
    input: Type.String()
  }, { additionalProperties: false }),
  constrainedSampling: {
    type: 'grammar',
    variants: {
      openai_lark: 'start: /.+/s'
    }
  }
};
```

### 处理工具调用

工具结果使用内容块，并且可以同时包含文本和图像：

```typescript
import { readFileSync } from 'fs';

const context: Context = {
  messages: [{ role: 'user', content: 'What is the weather in London?', timestamp: Date.now() }],
  tools: [weatherTool]
};

const response = await models.complete(model, context);

// Check for tool calls in the response
for (const block of response.content) {
  if (block.type === 'toolCall') {
    // Execute your tool with the arguments
    // See "Validating Tool Arguments" section for validation
    const result = await executeWeatherApi(block.arguments);

    // Add tool result with text content
    context.messages.push({
      role: 'toolResult',
      toolCallId: block.id,
      toolName: block.name,
      content: [{ type: 'text', text: JSON.stringify(result) }],
      isError: false,
      timestamp: Date.now()
    });
  }
}

// Tool results can also include images (for vision-capable models)
const imageBuffer = readFileSync('chart.png');
context.messages.push({
  role: 'toolResult',
  toolCallId: 'tool_xyz',
  toolName: 'generate_chart',
  content: [
    { type: 'text', text: 'Generated chart showing temperature trends' },
    { type: 'image', data: imageBuffer.toString('base64'), mimeType: 'image/png' }
  ],
  isError: false,
  timestamp: Date.now()
});
```

### 通过部分 JSON 流式传输工具调用

在流式传输期间，工具调用参数会随着数据到达逐步解析。这样，在完整参数可用前就可以实时更新 UI：

```typescript
const s = models.stream(model, context);

for await (const event of s) {
  if (event.type === 'toolcall_delta') {
    const toolCall = event.partial.content[event.contentIndex];

    // toolCall.arguments contains partially parsed JSON during streaming
    // This allows for progressive UI updates
    if (toolCall.type === 'toolCall' && toolCall.arguments) {
      // BE DEFENSIVE: arguments may be incomplete
      // Example: Show file path being written even before content is complete
      if (toolCall.name === 'write_file' && toolCall.arguments.path) {
        console.log(`Writing to: ${toolCall.arguments.path}`);

        // Content might be partial or missing
        if (toolCall.arguments.content) {
          console.log(`Content preview: ${toolCall.arguments.content.substring(0, 100)}...`);
        }
      }
    }
  }

  if (event.type === 'toolcall_end') {
    // Here toolCall.arguments is complete (but not yet validated)
    const toolCall = event.toolCall;
    console.log(`Tool completed: ${toolCall.name}`, toolCall.arguments);
  }
}
```

**关于部分工具参数的重要说明：**

- 在 `toolcall_delta` 事件期间，`arguments` 包含对部分 JSON 尽力而为的解析结果
- 字段可能缺失或不完整——使用前始终检查其是否存在
- 字符串值可能在单词中途被截断
- 数组可能不完整
- 嵌套对象可能只填充了一部分
- `arguments` 至少会是一个空对象 `{}`，绝不会是 `undefined`
- Google 提供商不支持函数调用的流式传输。你会收到一个包含完整参数的 `toolcall_delta` 事件。

### 验证工具参数

实现自己的工具执行循环时，请使用 `validateToolCall` 验证参数，然后再将其传递给工具：

```typescript
import { validateToolCall, type Tool } from '@earendil-works/pi-ai';

const tools: Tool[] = [weatherTool, calculatorTool];
const s = models.stream(model, { messages, tools });

for await (const event of s) {
  if (event.type === 'toolcall_end') {
    const toolCall = event.toolCall;

    try {
      // Validate arguments against the tool's schema (throws on invalid args)
      const validatedArgs = validateToolCall(tools, toolCall);
      const result = await executeMyTool(toolCall.name, validatedArgs);
      // ... add tool result to context
    } catch (error) {
      // Validation failed - return error as tool result so model can retry
      context.messages.push({
        role: 'toolResult',
        toolCallId: toolCall.id,
        toolName: toolCall.name,
        content: [{ type: 'text', text: error.message }],
        isError: true,
        timestamp: Date.now()
      });
    }
  }
}
```

### 完整事件参考

生成助手消息期间会发出以下所有流式事件：

| 事件类型 | 说明 | 关键属性 |
|------------|-------------|----------------|
| `start` | 流开始 | `partial`：初始助手消息结构 |
| `text_start` | 文本块开始 | `contentIndex`：在内容数组中的位置 |
| `text_delta` | 收到文本片段 | `delta`：新文本；`contentIndex`：位置 |
| `text_end` | 文本块完成 | `content`：完整文本；`contentIndex`：位置 |
| `thinking_start` | 思考块开始 | `contentIndex`：在内容数组中的位置 |
| `thinking_delta` | 收到思考片段 | `delta`：新文本；`contentIndex`：位置 |
| `thinking_end` | 思考块完成 | `content`：完整思考内容；`contentIndex`：位置 |
| `toolcall_start` | 工具调用开始 | `contentIndex`：在内容数组中的位置 |
| `toolcall_delta` | 工具参数流式传输 | `delta`：JSON 片段；`partial.content[contentIndex].arguments`：部分解析的参数 |
| `toolcall_end` | 工具调用完成 | `toolCall`：包含 `id`、`name`、`arguments` 的完整已验证工具调用 |
| `done` | 流完成 | `reason`：停止原因（"stop"、"length"、"toolUse"）；`message`：最终助手消息 |
| `error` | 发生错误 | `reason`：错误类型（"error" 或 "aborted"）；`error`：包含部分内容的 AssistantMessage |

不同内容块的流式事件不保证连续。提供商可能在同一个上游数据块中发出文本、思考和工具调用的增量，pi 可能将对应事件交错呈现，例如 `text_start`、`text_delta`、`toolcall_start`、`text_delta`、`toolcall_delta`。使用方必须通过 `contentIndex` 将每个增量/结束事件与其内容块关联起来，不得假定某个内容块的 `*_start`/`*_delta`/`*_end` 序列不会被其他内容块的事件打断。

## 图像输入

具备视觉能力的模型可以处理图像。你可以通过 `input` 属性检查模型是否支持图像。如果向不具备视觉能力的模型传入图像，图像会被静默忽略。

```typescript
import { readFileSync } from 'fs';

const model = models.getModel('openai', 'gpt-4o-mini')!;

// Check if model supports images
if (model.input.includes('image')) {
  console.log('Model supports vision');
}

const imageBuffer = readFileSync('image.png');
const base64Image = imageBuffer.toString('base64');

const response = await models.complete(model, {
  messages: [{
    role: 'user',
    content: [
      { type: 'text', text: 'What is in this image?' },
      { type: 'image', data: base64Image, mimeType: 'image/png' }
    ],
    timestamp: Date.now()
  }]
});

// Access the response
for (const block of response.content) {
  if (block.type === 'text') {
    console.log(block.text);
  }
}
```

## 图像生成

图像生成使用与文本/聊天生成分离的 API 接口，其设计与聊天侧相呼应：`ImagesModels` 集合容纳多个 `ImagesProvider`；读取操作是同步的；身份验证通过拥有相应模型的提供商进行解析。图像生成是一次性 API：`generateImages()` 会等待提供商响应，并返回最终的 `AssistantImages` 结果——不要使用聊天/流式 API 来生成图像。

### 基本图像生成

```typescript
import { builtinImagesModels } from '@earendil-works/pi-ai/providers/all';

// Every built-in image-generation provider; accepts the same options as createModels()
const imagesModels = builtinImagesModels();

const model = imagesModels.getModel('openrouter', 'google/gemini-2.5-flash-image')!;

// Auth resolves through the provider (OPENROUTER_API_KEY here); explicit apiKey wins
const result = await imagesModels.generateImages(model, {
  input: [{ type: 'text', text: 'Generate a red circle on a plain white background.' }]
});

for (const block of result.output) {
  if (block.type === 'text') {
    console.log(block.text);
  } else if (block.type === 'image') {
    console.log(block.mimeType);
    console.log(block.data.substring(0, 32));
  }
}
```

与聊天侧一样，你可以用各个部分构建集合：`createImagesModels({ credentials?, authContext? })`；`openrouterImagesProvider()` 工厂（来自 `@earendil-works/pi-ai/providers/openrouter-images`）；以及用于自定义图像提供商的 `createImagesProvider({ id, auth, models, refreshModels?, api })`（动态列表可使用 `imagesModels.refresh(provider?)`）。失败绝不会拒绝 Promise，而是返回一个 `AssistantImages`，其中 `stopReason: "error"`。集合中提供商范围的 `getAuth(providerId)` 与聊天侧的工作方式完全相同。

旧版全局 API（`getImageModel()` / `getImageModels()` / `getImageProviders()` / `generateImages()`）仍可通过[兼容入口点](#从旧版全局-api-迁移)使用：

```typescript
import { getImageModel, generateImages } from '@earendil-works/pi-ai/compat';

const model = getImageModel('openrouter', 'google/gemini-2.5-flash-image');
const result = await generateImages(model, {
  input: [{ type: 'text', text: 'Generate a red circle on a plain white background.' }]
}, {
  apiKey: process.env.OPENROUTER_API_KEY
});
```

有些模型也支持图像输入：

```typescript
import { readFileSync } from 'fs';

const imageBuffer = readFileSync('input.png');
const result = await imagesModels.generateImages(model, {
  input: [
    { type: 'text', text: 'Create a variation of this image with a blue background.' },
    { type: 'image', data: imageBuffer.toString('base64'), mimeType: 'image/png' }
  ]
});
```

请通过模型元数据检查能力：

```typescript
console.log(model.input);   // ['text', 'image']
console.log(model.output);  // ['image'] or ['image', 'text']
```

### 说明与限制

- 图像模型位于 `ImagesModels` 集合中，聊天模型位于 `Models` 集合中；两者是独立的接口。
- 使用 `generateImages()`，不要使用聊天/流式 API。
- 图像生成模型不参与工具调用。
- 输出在 `AssistantImages.output` 中返回，可同时包含 base64 编码的 `ImageContent` 块和 `TextContent` 块。
- 有些模型只返回图像，有些会同时返回图像和文本。请检查 `model.output`。
- 有些模型接受图像输入，有些只支持文本生成图像。请检查 `model.input`。
- 与流式 API 一样，图像生成支持 `apiKey`、`signal`、`headers`、`onPayload` 和 `onResponse` 等选项，结果还可能包含 `stopReason`、`responseId` 和 `usage`。
- 如果希望模型在对话中分析图像或调用工具，请使用普通聊天 API，并选择支持图像输入的模型。
- 目前只有 OpenRouter 一个提供商支持图像生成。

## 思考/推理

许多模型支持思考/推理能力，可以展示其内部思考过程。你可以通过 `reasoning` 属性检查模型是否支持推理。如果向不支持推理的模型传入推理选项，这些选项会被静默忽略。

### 统一接口（streamSimple/completeSimple）

```typescript
// Many models across providers support thinking/reasoning
const model = models.getModel('anthropic', 'claude-sonnet-4-5')!;
// or models.getModel('openai', 'gpt-5-mini');
// or models.getModel('google', 'gemini-2.5-flash');
// or models.getModel('xai', 'grok-4.5');

// Check if model supports reasoning
if (model.reasoning) {
  console.log('Model supports reasoning/thinking');
}

// Use the simplified reasoning option
const response = await models.completeSimple(model, {
  messages: [{ role: 'user', content: 'Solve: 2x + 5 = 13', timestamp: Date.now() }]
}, {
  reasoning: 'medium'  // 'minimal' | 'low' | 'medium' | 'high' | 'xhigh' | 'max'
});

// Access thinking and text blocks
for (const block of response.content) {
  if (block.type === 'thinking') {
    console.log('Thinking:', block.thinking);
  } else if (block.type === 'text') {
    console.log('Response:', block.text);
  }
}
```

`xhigh` 和 `max` 是按模型选择启用的级别。请使用 `getSupportedThinkingLevels(model)` 判断具体模型是否公开其中任一级别；GPT-5.6 等模型可以同时公开两者。

### 提供商专用选项（stream/complete）

`models.stream()`/`complete()` 接受所属 API 的完整选项集。对于动态查找的模型，请使用 `hasApi()` 将其缩窄到对应 API，以获得完整的选项类型：

```typescript
import { hasApi } from '@earendil-works/pi-ai';

// OpenAI Reasoning (o1, o3, gpt-5)
const openaiModel = models.getModel('openai', 'gpt-5-mini')!;
if (hasApi(openaiModel, 'openai-responses')) {
  await models.complete(openaiModel, context, {
    reasoningEffort: 'medium',
    reasoningSummary: 'detailed'  // OpenAI Responses API only
  });
}

// Anthropic Thinking
const anthropicModel = models.getModel('anthropic', 'claude-sonnet-4-5')!;
if (hasApi(anthropicModel, 'anthropic-messages')) {
  await models.complete(anthropicModel, context, {
    thinkingEnabled: true,
    thinkingBudgetTokens: 8192  // Optional token limit
  });
}

// Google Gemini Thinking
const googleModel = models.getModel('google', 'gemini-2.5-flash')!;
if (hasApi(googleModel, 'google-generative-ai')) {
  await models.complete(googleModel, context, {
    thinking: {
      enabled: true,
      budgetTokens: 8192  // -1 for dynamic, 0 to disable
    }
  });
}
```

### 流式传输思考内容

流式传输时，思考内容通过特定事件传递：

```typescript
const s = models.streamSimple(model, context, { reasoning: 'high' });

for await (const event of s) {
  switch (event.type) {
    case 'thinking_start':
      console.log('[Model started thinking]');
      break;
    case 'thinking_delta':
      process.stdout.write(event.delta);  // Stream thinking content
      break;
    case 'thinking_end':
      console.log('\n[Thinking complete]');
      break;
  }
}
```

## 停止原因

每个 `AssistantMessage` 都包含一个 `stopReason` 字段，用于说明生成过程如何结束：

- `"pending"`——仅出现在尚不清楚停止原因的部分消息中
- `"stop"`——这是模型在本轮生成的最终消息
- `"length"`——输出达到最大令牌限制
- `"toolUse"`——模型正在调用工具，并等待工具结果
- `"error"`——生成期间发生错误
- `"aborted"`——请求已通过中止信号取消

如果底层 API 公开了响应或消息标识符，`AssistantMessage` 还可能包含提供商专用的上游 `responseId`。不要假定它在所有提供商中都存在。

## 错误处理

请求失败绝不会从流函数中抛出：当请求以错误结束时（包括中止和工具调用验证错误），流式 API 会发出错误事件，最终消息则会携带详细信息：

```typescript
// In streaming
for await (const event of s) {
  if (event.type === 'error') {
    // event.reason is either "error" or "aborted"
    // event.error is the AssistantMessage with partial content
    console.error(`Error (${event.reason}):`, event.error.errorMessage);
    console.log('Partial content:', event.error.content);
  }
}

// The final message will have the error details
const message = await s.result();
if (message.stopReason === 'error' || message.stopReason === 'aborted') {
  console.error('Request failed:', message.errorMessage);
  // message.content contains any partial content received before the error
  // message.usage contains partial token counts and costs
}
```

身份验证失败（未配置密钥、OAuth 刷新失败、未知提供商）也以相同方式呈现：一个 `stopReason: "error"` 的流错误。

### 中止请求

中止信号允许你取消正在进行的请求。已中止请求具有 `stopReason === 'aborted'`：

```typescript
const controller = new AbortController();

// Abort after 2 seconds
setTimeout(() => controller.abort(), 2000);

const s = models.stream(model, {
  messages: [{ role: 'user', content: 'Write a long story', timestamp: Date.now() }]
}, {
  signal: controller.signal
});

for await (const event of s) {
  if (event.type === 'text_delta') {
    process.stdout.write(event.delta);
  } else if (event.type === 'error') {
    // event.reason tells you if it was "error" or "aborted"
    console.log(`${event.reason === 'aborted' ? 'Aborted' : 'Error'}:`, event.error.errorMessage);
  }
}

// Get results (may be partial if aborted)
const response = await s.result();
if (response.stopReason === 'aborted') {
  console.log('Request was aborted:', response.errorMessage);
  console.log('Partial content received:', response.content);
  console.log('Tokens used:', response.usage);
}
```

### 中止后继续

已中止的消息可以加入对话上下文，并在后续请求中继续：

```typescript
const context = {
  messages: [
    { role: 'user', content: 'Explain quantum computing in detail', timestamp: Date.now() }
  ]
};

// First request gets aborted after 2 seconds
const controller1 = new AbortController();
setTimeout(() => controller1.abort(), 2000);

const partial = await models.complete(model, context, { signal: controller1.signal });

// Add the partial response to context
context.messages.push(partial);
context.messages.push({ role: 'user', content: 'Please continue', timestamp: Date.now() });

// Continue the conversation
const continuation = await models.complete(model, context);
```

### 调试提供商载荷

使用 `onPayload` 回调检查发送给提供商的请求载荷。它适合用来调试请求格式问题或提供商验证错误。

```typescript
const response = await models.complete(model, context, {
  onPayload: (payload) => {
    console.log('Provider payload:', JSON.stringify(payload, null, 2));
  }
});
```

`stream`、`complete`、`streamSimple` 和 `completeSimple` 均支持该回调。

## 自定义提供商

### createProvider()

`createProvider()` 使用身份标识、身份验证、模型列表和 API 实现等部分构建提供商。它适用于本地推理服务器、代理或任何兼容 OpenAI/Anthropic 的端点：

```typescript
import { createModels, createProvider, envApiKeyAuth, type Model } from '@earendil-works/pi-ai';
import { openAICompletionsApi } from '@earendil-works/pi-ai/api/openai-completions.lazy';

const ollamaModel: Model<'openai-completions'> = {
  id: 'llama-3.1-8b',
  name: 'Llama 3.1 8B (Ollama)',
  api: 'openai-completions',
  provider: 'ollama',
  baseUrl: 'http://localhost:11434/v1',
  reasoning: false,
  input: ['text'],
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
  contextWindow: 128000,
  maxTokens: 32000
};

const ollama = createProvider({
  id: 'ollama',
  name: 'Ollama',
  baseUrl: 'http://localhost:11434/v1',
  // Every provider declares auth; keyless local servers resolve as configured with no key.
  auth: { apiKey: { name: 'Ollama', resolve: async () => ({ auth: {} }) } },
  models: [ollamaModel],
  api: openAICompletionsApi(),
});

const models = createModels();
models.setProvider(ollama);

await models.complete(models.getModel('ollama', 'llama-3.1-8b')!, context);
```

对于确实使用密钥的提供商，`envApiKeyAuth(displayName, envVars)` 可提供标准行为（已存储凭据优先，其次是第一个已设置的环境变量）：

```typescript
const proxy = createProvider({
  id: 'my-proxy',
  auth: { apiKey: envApiKeyAuth('My proxy API key', ['MY_PROXY_API_KEY']) },
  models: [/* ... */],
  api: openAICompletionsApi(),
});
```

混合 API 提供商会传入一个以 `model.api` 为键的映射；每个模型都会分派到其 API 的实现：

```typescript
import { anthropicMessagesApi } from '@earendil-works/pi-ai/api/anthropic-messages.lazy';
import { openAIResponsesApi } from '@earendil-works/pi-ai/api/openai-responses.lazy';

const gateway = createProvider({
  id: 'my-gateway',
  auth: { apiKey: envApiKeyAuth('Gateway key', ['GATEWAY_API_KEY']) },
  models: [/* models with api: 'anthropic-messages' or 'openai-responses' */],
  api: {
    'anthropic-messages': anthropicMessagesApi(),
    'openai-responses': openAIResponsesApi(),
  },
});
```

提供商级的端点或请求转换应放在提供商的 API 实现中：包装 `ProviderStreams`，再将它作为 `api` 传入，使每个请求在分派前都经过转换。Cloudflare 提供商会这样做，以便根据已解析的提供商环境值填充账号/网关端点占位符：

```typescript
function tenantStreams(streams: ProviderStreams): ProviderStreams {
  const withTenant = (model: Model<Api>) => ({ ...model, baseUrl: model.baseUrl.replace('{tenant}', tenantId) });
  return {
    stream: (model, context, options) => streams.stream(withTenant(model), context, options),
    streamSimple: (model, context, options) => streams.streamSimple(withTenant(model), context, options),
  };
}

const tenantGateway = createProvider({
  id: 'tenant-gateway',
  auth: { apiKey: envApiKeyAuth('Gateway key', ['GATEWAY_API_KEY']) },
  models: [/* ... */],
  api: tenantStreams(openAICompletionsApi()),
});
```

动态模型列表使用 `fetchModels`。`Models.refresh()` 会刷新每个已配置的动态提供商，并传入其有效 API 密钥或已刷新的 OAuth 凭据。`ModelsStore` 用于持久化动态目录；两个存储都默认使用基于内存的实现。其 `read`、`write` 和 `delete` 操作接受可选的取消信号，`Models` 会将这些等待绑定到提供商的刷新信号。

```typescript
const models = createModels({ credentials, modelsStore });
const llamacpp = createProvider({
  id: 'llamacpp',
  auth: { apiKey: { name: 'llama.cpp', resolve: async () => ({ auth: {} }) } },
  models: [],
  fetchModels: async ({ signal }) => fetchModelsFromServer('http://localhost:8080', signal),
  api: openAICompletionsApi(),
});

models.setProvider(llamacpp);
const result = await models.refresh({ signal });
if (result.aborted) console.log('refresh cancelled');
for (const [provider, error] of result.errors) console.error(provider, error);
```

未传入可选信号时，`Models.refresh()` 不设时限。提供商始终会收到具体的 `RefreshModelsContext.signal`，并且必须在网络请求和其他阻塞操作中遵守该信号。调用方提供信号后，即使自定义提供商不配合，`Models.refresh()` 也会在取消后迅速返回 `aborted: true`；但提供商仍必须遵守信号，以停止其底层工作。

使用 `models.refresh({ providers: ['openrouter'] })` 将工作限制到指定提供商，使用 `models.refresh({ allowNetwork: false })` 在不访问网络的情况下恢复已持久化的目录，或使用 `models.refresh({ force: true })` 绕过提供商的新鲜度检查。模型读取保持同步，并返回最近恢复或刷新的列表。

`createProvider()` 会自动处理动态发布和持久化。手写的 `Provider.refreshModels()` 实现会收到只读的 `context.stored` 快照，并通过 `context.publish({ persist?, update? })` 发布。省略 `persist` 会保持存储不变；传入 `ModelsStoreEntry` 会写入；传入 `persist: null` 会删除。发布操作会检查代次；同步的内存目录更改应放在 `update` 中，不要在发布前改变状态。

自定义模型可以携带 `headers`（例如用于位于机器人检测机制之后的代理）和 `compat` 标志。`Models.getAuth(model)` 会包含这些模型请求头，而流方法会先合并它们，再合并显式请求头和 `transformHeaders`。参见 [OpenAI 兼容性设置](#openai-兼容性设置)。

某些与 OpenAI 兼容的服务器不理解具备推理能力的模型所使用的 `developer` 角色。对于这些提供商，请将 `compat.supportsDeveloperRole` 设置为 `false`，让系统提示词以 `system` 消息发送。如果服务器也不支持 `reasoning_effort`，还应将 `compat.supportsReasoningEffort` 设置为 `false`。这通常适用于 Ollama、vLLM、SGLang 和类似的 OpenAI 兼容服务器。

使用模型级 `thinkingLevelMap` 描述模型专用的思考控制。键是 pi 的思考级别（`off`、`minimal`、`low`、`medium`、`high`、`xhigh`、`max`）。缺少的 `high` 及以下标准级别会使用提供商默认值；`xhigh` 和 `max` 需要选择启用，并要求映射中存在非空项。字符串值会发送给提供商，`null` 表示不支持某个级别，映射可以跳过级别。

```typescript
const ollamaReasoningModel: Model<'openai-completions'> = {
  id: 'gpt-oss:20b',
  name: 'GPT-OSS 20B (Ollama)',
  api: 'openai-completions',
  provider: 'ollama',
  baseUrl: 'http://localhost:11434/v1',
  reasoning: true,
  input: ['text'],
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
  contextWindow: 131072,
  maxTokens: 32000,
  thinkingLevelMap: {
    minimal: null,
    low: null,
    medium: null,
    high: 'high',
    xhigh: null,
  },
  compat: {
    supportsDeveloperRole: false,
    supportsReasoningEffort: false,
  }
};
```

### 直接调用 API 实现

API 实现可以单独导入。每个模块仅导出 `stream` 和 `streamSimple`，并带有相应 API 的完整选项类型。直接调用会绕过提供商身份验证——请显式传入 `apiKey`：

```typescript
import { stream } from '@earendil-works/pi-ai/api/anthropic-messages';

const s = stream(claudeModel, context, {
  apiKey: process.env.ANTHROPIC_API_KEY,
  thinkingEnabled: true,
  thinkingBudgetTokens: 2048,
});
```

内置 API 实现位于 `./api/<api-id>` 下：

| API id | 选项类型 |
|--------|--------------|
| `anthropic-messages` | `AnthropicOptions` |
| `openai-completions` | `OpenAICompletionsOptions` |
| `openai-responses` | `OpenAIResponsesOptions` |
| `openai-codex-responses` | `OpenAICodexResponsesOptions` |
| `azure-openai-responses` | `AzureOpenAIResponsesOptions` |
| `google-generative-ai` | `GoogleOptions` |
| `google-vertex` | `GoogleVertexOptions` |
| `mistral-conversations` | `MistralOptions` |
| `bedrock-converse-stream` | `BedrockOptions` |

导入实现模块会加载其 SDK。`./api/<id>.lazy` 包装器（由提供商工厂使用）会在运行时或打包器支持动态导入代码块时，将加载推迟到第一次请求。旧版本中的传统原始 API 子路径（`./anthropic`、`./google`、`./mistral`、`./openai-completions` 等）已移除；请使用 `@earendil-works/pi-ai/api/<api-id>`。

### OpenAI 兼容性设置

许多提供商都实现了 `openai-completions` API，但存在细微差异。默认情况下，对于一小部分已知的 OpenAI 兼容提供商（Cerebras、xAI、Chutes、DeepSeek、NVIDIA NIM、Together AI、zAi、OpenCode、Cloudflare Workers AI 等），库会根据 `baseUrl` 自动检测兼容性设置。对于自定义代理或未知端点，可以通过 `compat` 字段覆盖这些设置。对于 `openai-responses` 模型，compat 字段支持 Responses 专用标志。

```typescript
interface OpenAICompletionsCompat {
  supportsStore?: boolean;           // Whether provider supports the `store` field (default: true)
  supportsDeveloperRole?: boolean;   // Whether provider supports `developer` role vs `system` (default: true)
  supportsReasoningEffort?: boolean; // Whether provider supports `reasoning_effort` (default: true)
  supportsUsageInStreaming?: boolean; // Whether provider supports `stream_options: { include_usage: true }` (default: true)
  supportsStrictMode?: boolean;      // Whether provider supports `strict` in tool definitions (default: true)
  supportsOpenAIGrammarTools?: boolean; // Whether to emit OpenAI custom Lark/regex grammar tools; false falls back to normal function tools (default: false; the generated catalog enables it for capable models)
  sendSessionAffinityHeaders?: boolean; // Send session-affinity data from `sessionId` (default: false)
  sessionAffinityFormat?: 'openai' | 'openai-nosession' | 'openrouter'; // Format for session affinity: 'openai' uses `prompt_cache_key`, `session_id`, `x-client-request-id`, and `x-session-affinity`; 'openai-nosession' uses `prompt_cache_key`, `x-client-request-id`, and `x-session-affinity`; 'openrouter' uses `x-session-id` (default: auto-detected)
  maxTokensField?: 'max_completion_tokens' | 'max_tokens';  // Which field name to use (default: max_completion_tokens)
  requiresToolResultName?: boolean;  // Whether tool results require the `name` field (default: false)
  requiresAssistantAfterToolResult?: boolean; // Whether tool results must be followed by an assistant message (default: false)
  requiresThinkingAsText?: boolean;  // Whether thinking blocks must be converted to text (default: false)
  requiresReasoningContentOnAssistantMessages?: boolean; // Whether all replayed assistant messages must include empty reasoning_content when reasoning is enabled (default: auto-detected for DeepSeek)
  thinkingFormat?: 'openai' | 'openrouter' | 'deepseek' | 'together' | 'baseten' | 'zai' | 'qwen' | 'chat-template' | 'qwen-chat-template' | 'string-thinking' | 'ant-ling'; // Format for reasoning param: 'openai' uses reasoning_effort, 'openrouter' uses reasoning: { effort }, 'deepseek' uses thinking: { type } plus reasoning_effort when supported, 'together' uses reasoning: { enabled } plus reasoning_effort when supported, 'baseten' uses configurable chat_template_args plus reasoning_effort when supported, 'zai' uses thinking: { type }, 'qwen' uses enable_thinking, 'chat-template' uses configurable chat_template_kwargs, 'qwen-chat-template' uses chat_template_kwargs.enable_thinking and preserve_thinking, 'string-thinking' uses top-level thinking, 'ant-ling' uses reasoning: { effort } only for mapped efforts (default: openai)
  chatTemplateKwargs?: Record<string, string | number | boolean | null | { '$var': 'thinking.enabled' | 'thinking.effort'; omitWhenOff?: boolean }>; // chat_template_kwargs values; use $var for pi-controlled thinking values
  chatTemplateArgs?: Record<string, string | number | boolean | null | { '$var': 'thinking.enabled' | 'thinking.effort'; omitWhenOff?: boolean }>; // chat_template_args values for thinkingFormat: 'baseten'; use $var for pi-controlled thinking values
  cacheControlFormat?: 'anthropic';  // Anthropic-style cache_control on system prompt, last tool, and last user/assistant text content
  openRouterRouting?: OpenRouterRouting; // OpenRouter routing preferences (default: {})
  vercelGatewayRouting?: VercelGatewayRouting; // Vercel AI Gateway routing preferences (default: {})
}

interface OpenAIResponsesCompat {
  supportsDeveloperRole?: boolean;   // Whether provider supports `developer` role vs `system` (default: true)
  sessionAffinityFormat?: 'openai' | 'openai-nosession' | 'openrouter'; // Session-affinity header format: 'openai' sends `session_id` and `x-client-request-id`; 'openai-nosession' sends `x-client-request-id`; 'openrouter' sends `x-session-id`. Does not affect the `prompt_cache_key` body param (default: auto-detected)
  supportsLongCacheRetention?: boolean; // Whether provider supports `prompt_cache_retention: "24h"` (default: true)
  supportsStrictMode?: boolean;      // Whether provider supports strict JSON-schema function tools (default: false; enabled in metadata for built-in OpenAI models)
  supportsOpenAIGrammarTools?: boolean; // Whether to emit OpenAI custom Lark/regex grammar tools; false falls back to normal function tools (default: false; the generated catalog enables it for capable models)
}
```

如果没有设置 `compat`，库会回退到基于 URL 的检测。如果只设置了部分 `compat`，未指定的字段会使用检测到的默认值。这适用于：

- **LiteLLM 代理**：可能不支持 `store` 字段
- **自定义推理服务器**：可能使用非标准字段名
- **自托管端点**：可能具有不同的功能支持

## 用于测试的仿真提供商

`fauxProvider()` 会构建一个内存中的提供商，为测试和演示返回预先编排的响应：

```typescript
import {
  createModels,
  fauxAssistantMessage,
  fauxProvider,
  fauxText,
  fauxThinking,
  fauxToolCall,
} from '@earendil-works/pi-ai';

const faux = fauxProvider({
  tokensPerSecond: 50 // optional
});

const models = createModels();
models.setProvider(faux.provider);

const model = faux.getModel();
const context = {
  messages: [{ role: 'user', content: 'Summarize package.json and then call echo', timestamp: Date.now() }]
};

faux.setResponses([
  fauxAssistantMessage([
    fauxThinking('Need to inspect package metadata first.'),
    fauxToolCall('echo', { text: 'package.json' })
  ], { stopReason: 'toolUse' })
]);

const first = await models.complete(model, context, {
  sessionId: 'session-1',
  cacheRetention: 'short'
});
context.messages.push(first);

context.messages.push({
  role: 'toolResult',
  toolCallId: first.content.find((block) => block.type === 'toolCall')!.id,
  toolName: 'echo',
  content: [{ type: 'text', text: 'package.json contents here' }],
  isError: false,
  timestamp: Date.now()
});

faux.setResponses([
  fauxAssistantMessage([
    fauxThinking('Now I can summarize the tool output.'),
    fauxText('Here is the summary.')
  ])
]);

const s = models.stream(model, context);
for await (const event of s) {
  console.log(event.type);
}

// Optional: multiple faux models for model-switching tests
const multiModel = fauxProvider({
  provider: 'faux-multi',
  models: [
    { id: 'faux-fast', reasoning: false },
    { id: 'faux-thinker', reasoning: true }
  ]
});
models.setProvider(multiModel.provider);
const thinker = multiModel.getModel('faux-thinker');

console.log(thinker?.reasoning);
console.log(faux.getPendingResponseCount());
console.log(faux.state.callCount);
```

说明：

- 响应按请求开始的顺序从队列中取用。
- 如果队列为空，仿真提供商会返回一条助手错误消息，其 `errorMessage: "No more faux responses queued"`。
- 使用 `faux.setResponses([...])` 替换队列中剩余的响应，使用 `faux.appendResponses([...])` 添加更多响应。
- `faux.models` 会公开所有仿真模型。`faux.getModel()` 返回第一个模型，`faux.getModel(id)` 返回指定模型。
- 使用 `fauxAssistantMessage(...)` 编排助手回复。使用 `fauxText(...)`、`fauxThinking(...)` 和 `fauxToolCall(...)` 构建内容块，无需手动填写底层字段。
- 用量估算为每 4 个字符约 1 个令牌。当存在 `sessionId` 且 `cacheRetention` 不是 `"none"` 时，会自动模拟提示缓存的读取和写入。
- 工具调用参数通过 `toolcall_delta` 片段逐步流式传输。
- 默认情况下，每个流式片段都在自己的微任务中发出。设置 `tokensPerSecond` 可按实时速度控制片段传递。
- 预期用法是每个句柄对应一个确定性的编排流程。如果需要相互独立的并发流程，请创建具有不同 `provider` ID 的独立仿真提供商。

## 跨提供商移交

此库支持在同一对话中不同 LLM 提供商之间无缝移交。你可以在对话中途切换模型，同时保留上下文，包括思考块、工具调用和工具结果。

将一个提供商的消息发送给另一个提供商时，库会自动转换消息以保持兼容：

- **用户消息和工具结果消息**保持不变
- **来自同一提供商/API 的助手消息**原样保留
- **来自不同提供商的助手消息**会将其思考块转换为带 `<thinking>` 标签的文本
- **工具调用和普通文本**保持不变

```typescript
import { createModels, type Context } from '@earendil-works/pi-ai';
import { anthropicProvider } from '@earendil-works/pi-ai/providers/anthropic';
import { openaiProvider } from '@earendil-works/pi-ai/providers/openai';
import { googleProvider } from '@earendil-works/pi-ai/providers/google';

const models = createModels();
models.setProvider(anthropicProvider());
models.setProvider(openaiProvider());
models.setProvider(googleProvider());

const context: Context = { messages: [] };

// Start with Claude
const claude = models.getModel('anthropic', 'claude-sonnet-4-5')!;
context.messages.push({ role: 'user', content: 'What is 25 * 18?', timestamp: Date.now() });
context.messages.push(await models.completeSimple(claude, context, { reasoning: 'medium' }));

// Switch to GPT-5 - it will see Claude's thinking as <thinking> tagged text
const gpt5 = models.getModel('openai', 'gpt-5-mini')!;
context.messages.push({ role: 'user', content: 'Is that calculation correct?', timestamp: Date.now() });
context.messages.push(await models.complete(gpt5, context));

// Switch to Gemini
const gemini = models.getModel('google', 'gemini-2.5-flash')!;
context.messages.push({ role: 'user', content: 'What was the original question?', timestamp: Date.now() });
const geminiResponse = await models.complete(gemini, context);
```

所有提供商都能处理来自其他提供商的消息——文本、工具调用及结果（包括图像）、思考块（转换为带标签的文本），以及包含部分内容的已中止消息。这支持灵活的工作流：先使用快速模型，遇到复杂推理时切换到能力更强的模型，或在提供商服务中断时保持对话连续性。

## 上下文序列化

`Context` 对象可以使用标准 JSON 方法轻松序列化和反序列化，从而方便地持久化对话、实现聊天记录，或在服务之间传输上下文：

```typescript
const context: Context = {
  systemPrompt: 'You are a helpful assistant.',
  messages: [
    { role: 'user', content: 'What is TypeScript?', timestamp: Date.now() }
  ]
};

const model = models.getModel('openai', 'gpt-4o-mini')!;
const response = await models.complete(model, context);
context.messages.push(response);

// Serialize the entire context
const serialized = JSON.stringify(context);

// Save to database, localStorage, file, etc.
localStorage.setItem('conversation', serialized);

// Later: deserialize and continue the conversation
const restored: Context = JSON.parse(localStorage.getItem('conversation')!);
restored.messages.push({ role: 'user', content: 'Tell me more about its type system', timestamp: Date.now() });

// Continue with any model
const newModel = models.getModel('anthropic', 'claude-3-5-haiku-20241022')!;
const continuation = await models.complete(newModel, restored);
```

模型本身也是普通的可序列化数据——没有附加函数或实现——因此持久化“此对话使用哪个模型”只需一次 `JSON.stringify`。

> **注意**：如果上下文包含图像（如“图像输入”一节所示，以 base64 编码），这些图像也会被序列化。

## 浏览器用法

此库支持浏览器环境。核心入口点和提供商工厂没有副作用，可以干净地打包。浏览器中无法使用环境变量，因此请显式传入 API 密钥；也可以注入 `CredentialStore`（例如基于 localStorage 的实现），让提供商身份验证从已存储凭据中解析：

```typescript
import { createModels } from '@earendil-works/pi-ai';
import { anthropicProvider } from '@earendil-works/pi-ai/providers/anthropic';

const models = createModels();
models.setProvider(anthropicProvider());

const model = models.getModel('anthropic', 'claude-3-5-haiku-20241022')!;
const response = await models.complete(model, {
  messages: [{ role: 'user', content: 'Hello!', timestamp: Date.now() }]
}, {
  apiKey: 'your-api-key'
});
```

> **安全警告**：在前端代码中公开 API 密钥很危险。任何人都能提取并滥用你的密钥。此方式仅适用于内部工具或演示。生产应用应使用后端代理来妥善保护 API 密钥。

浏览器兼容性说明：

- 浏览器环境不支持 Amazon Bedrock（`bedrock-converse-stream`）。它仍可能出现在模型列表中，但调用会在运行时失败。
- OAuth 登录流程仅支持 Node。它们通过打包器不透明的导入惰性加载，因此注册支持 OAuth 的提供商不会将仅限 Node 的代码引入浏览器包——只有实际登录时才会加载。
- 如果 Web 应用需要 Bedrock 或基于 OAuth 的身份验证，请使用服务端代理或后端服务。

## 打包与摇树优化

为了减小包体积，只导入所需的提供商：

```typescript
import { createModels } from '@earendil-works/pi-ai';
import { openaiProvider } from '@earendil-works/pi-ai/providers/openai';

const models = createModels();
models.setProvider(openaiProvider());
```

规则：

- `@earendil-works/pi-ai` 是核心入口点，不会导入内置目录、提供商工厂或 SDK 实现。
- `@earendil-works/pi-ai/providers/<provider>` 只导入该提供商的目录和惰性 API 包装器。
- `@earendil-works/pi-ai/providers/all` 会导入每个内置提供商工厂和所有目录。仅在需要完整内置集合时使用它。
- 使用代码拆分时，提供商 SDK 会留在惰性代码块中，并在第一次请求时加载。
- 不使用代码拆分时，打包器会把可达的惰性 API 实现合并到单个包中。单提供商包会包含该提供商的 SDK；`providers/all` 会包含所有静态可见的 SDK。Bedrock 是例外：其 AWS SDK 实现通过打包器不透明、仅限 Node 的导入加载。
- 直接导入 `@earendil-works/pi-ai/api/<api-id>` 会立即加载该 API 实现及其 SDK。

新建的打包应用应避免使用 `@earendil-works/pi-ai/compat`；它保留旧版全局 API，并导入完整的内置目录接口。

对于单文件 Node ESM 包，某些 SDK 依赖项在内部仍可能使用动态 CommonJS `require()`。如果看到 `Dynamic require of "child_process" is not supported` 等错误，请向包中添加 Node `require` 垫片。使用 esbuild 时：

```bash
esbuild app.js --bundle --platform=node --format=esm \
  --banner:js='import { createRequire } from "module";const require = createRequire(import.meta.url);' \
  --outfile=app.bundle.js
```

此做法仅用于 Node 包；它不是浏览器或 Cloudflare Workers 的解决办法。

Bedrock 仅支持 Node。添加方式与其他提供商相同：

```typescript
import { createModels } from '@earendil-works/pi-ai';
import { amazonBedrockProvider } from '@earendil-works/pi-ai/providers/amazon-bedrock';

const models = createModels();
models.setProvider(amazonBedrockProvider());
```

在普通 Node 包用法和代码拆分包中，Bedrock 会惰性加载其 AWS SDK 实现。如果独立单文件包必须包含 Bedrock 支持，请显式注册实现模块：

```typescript
import { setBedrockProviderModule } from '@earendil-works/pi-ai/api/bedrock-converse-stream.lazy';
import { bedrockProviderModule } from '@earendil-works/pi-ai/bedrock-provider';

setBedrockProviderModule(bedrockProviderModule);
```

该显式覆盖会打包 AWS SDK。如果没有它，Bedrock 的不透明运行时导入会要求该包的 Bedrock 实现文件在运行时可用。

### 提供商范围的环境变量覆盖

在流选项中传入 `env`，可将提供商配置限定到单个请求。`env` 中的值优先于进程环境变量，用于提供商身份验证，以及 Cloudflare 账号 ID、Azure OpenAI 设置、Vertex 项目/位置、Bedrock 设置、`PI_CACHE_RETENTION` 和 `HTTP_PROXY`/`HTTPS_PROXY` 等配置。

```typescript
const models = builtinModels();
const model = models.getModel('cloudflare-ai-gateway', 'workers-ai/@cf/moonshotai/kimi-k2.6')!;

const response = await models.complete(model, context, {
  env: {
    CLOUDFLARE_API_KEY: '...',
    CLOUDFLARE_ACCOUNT_ID: 'account-id',
    CLOUDFLARE_GATEWAY_ID: 'gateway-id'
  }
});
```

当同一进程需要为每个请求使用不同的提供商设置，或不希望环境变量泄漏到提供商调用中时，请使用此功能。

## OAuth 提供商

有几个提供商支持 OAuth 身份验证，而不是静态 API 密钥：

- **Anthropic**（Claude Pro/Max 订阅）
- **OpenAI Codex**（ChatGPT Plus/Pro 订阅，可访问 GPT-5.x Codex 模型）
- **GitHub Copilot**（Copilot 订阅）
- **OpenRouter**（OAuth PKCE，生成用户控制的 API 密钥）

这些提供商各自包含一个 `OAuthAuth`，位于 `provider.auth.oauth` 上。它有三个操作：`login(interaction)` 使用提供商中立的 `AuthInteraction.prompt()`/`notify()` 协议并返回凭据；`refresh(credential, signal)` 在适用时刷新即将过期的凭据；`toAuth(credential)` 派生请求身份验证信息（GitHub Copilot 的按账号 base URL 就来自这里）。提供商登录交互和刷新调用始终携带具体的中止信号。刷新是自动的：`models.getAuth(providerId)` 和请求路径会在凭据存储锁内刷新过期令牌，因此并发请求和进程不会重复刷新。OpenRouter 的 OAuth 流程会返回永久 API 密钥，所以其刷新操作不执行任何工作。

```typescript
import { createModels } from '@earendil-works/pi-ai';
import { anthropicProvider } from '@earendil-works/pi-ai/providers/anthropic';

const models = createModels({ credentials: myStore }); // persistent CredentialStore
models.setProvider(anthropicProvider());

// Login: Models drives the flow and persists the credential
await models.login('anthropic', 'oauth', {
  prompt: async (p) => {
    // p.type: 'text' | 'secret' | 'select' | 'manual_code'
    // manual_code prompts race a local callback server; p.signal aborts them when the server wins
    return await askUser(p.message);
  },
  notify: (event) => {
    // event.type: 'info' | 'auth_url' | 'device_code' | 'progress'
    if (event.type === 'info') {
      console.log(event.message);
      for (const link of event.links ?? []) console.log(`${link.label ?? 'More information'}: ${link.url}`);
    }
    if (event.type === 'auth_url') console.log(`Open: ${event.url}`);
    if (event.type === 'device_code') console.log(`Code: ${event.userCode} at ${event.verificationUri}`);
    if (event.type === 'progress') console.log(event.message);
  },
});

// From here on, requests resolve and refresh the token automatically
const model = models.getModel('anthropic', 'claude-sonnet-4-5')!;
await models.complete(model, context);

// Logout
await models.logout('anthropic');
```

### Vertex AI

Vertex AI 模型支持 Google Cloud API 密钥或 Application Default Credentials（应用默认凭据，ADC）。它由提供商管理的 API 密钥登录流程可以配置任一方式：

- **API 密钥**：设置 `GOOGLE_CLOUD_API_KEY`，或在调用选项中传入 `apiKey`。
- **本地开发（ADC）**：运行 `gcloud auth application-default login`
- **CI/生产环境（ADC）**：将 `GOOGLE_APPLICATION_CREDENTIALS` 设为服务账号 JSON 密钥文件的路径

使用 ADC 时，还要设置 `GOOGLE_CLOUD_PROJECT`（也可用 `GCLOUD_PROJECT`）和 `GOOGLE_CLOUD_LOCATION`。你也可以在调用选项中传入 `project`/`location`。使用 `GOOGLE_CLOUD_API_KEY` 时，不需要 `project` 和 `location`。

```bash
# Local (uses your user credentials)
gcloud auth application-default login
export GOOGLE_CLOUD_PROJECT="my-project"
export GOOGLE_CLOUD_LOCATION="us-central1"

# CI/Production (service account key file)
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"
```

官方文档：[Application Default Credentials](https://cloud.google.com/docs/authentication/application-default-credentials)

### CLI 登录

最快的身份验证方式：

```bash
npx @earendil-works/pi-ai login              # interactive provider selection
npx @earendil-works/pi-ai login anthropic    # login to specific provider
npx @earendil-works/pi-ai list               # list available providers
```

凭据会保存到当前目录的 `auth.json`。

### 以编程方式使用 OAuth

内置登录和刷新流程是私有的提供商实现。请使用提供商自己的 `OAuthAuth`；它可以与 `CredentialStore` 组合，并通过 `Models` 获得带锁的自动刷新。`@earendil-works/pi-ai/oauth` 入口点只保留编码智能体扩展的 OAuth 兼容性所需的类型声明。

提供商说明：

**OpenAI Codex**：需要 ChatGPT Plus 或 Pro 订阅。它提供对 GPT-5.x Codex 模型的访问，具备扩展的上下文窗口和推理能力。当流选项中提供 `sessionId` 且 `cacheRetention` 不是 `"none"` 时，库会自动处理基于会话的提示缓存。你可以在流选项中将 `transport` 设置为 `"sse"`、`"websocket"` 或 `"auto"`，以选择 Codex Responses 传输方式。使用带 `sessionId` 且启用缓存保留的 WebSocket 时，连接会按会话复用，并在空闲 5 分钟后过期。

**Azure OpenAI (Responses)**：仅使用 Responses API。设置 `AZURE_OPENAI_API_KEY`，以及 `AZURE_OPENAI_BASE_URL` 或 `AZURE_OPENAI_RESOURCE_NAME`。`AZURE_OPENAI_BASE_URL` 同时支持 `https://<resource>.openai.azure.com` 和 `https://<resource>.cognitiveservices.azure.com`；根端点会自动规范化为 `.../openai/v1`。如有需要，可以通过 `AZURE_OPENAI_API_VERSION`（默认为 `v1`）覆盖 API 版本。部署名称默认视为模型 ID；可以使用 `azureDeploymentName` 覆盖，或通过 `AZURE_OPENAI_DEPLOYMENT_NAME_MAP` 提供以逗号分隔的 `model-id=deployment` 对（例如 `gpt-4o-mini=my-deployment,gpt-4o=prod`）。有意不支持传统的基于部署的 URL。

**GitHub Copilot**：如果出现 "The requested model is not supported" 错误，请在 VS Code 中手动启用该模型：打开 Copilot Chat，点击模型选择器，选择相应模型（带警告图标），再点击“Enable”。

## 从旧版全局 API 迁移

旧版本公开了全局 API：`stream()`/`complete()` 通过全局注册表根据 `model.api` 分派；同步的 `getModel()`/`getModels()`/`getProviders()` 用于读取目录；此外还有 `registerApiProvider()`、`getEnvApiKey()` 以及各 API 的惰性流函数。这些接口在**兼容入口点**中保持不变：

```typescript
// Before
import { getModel, complete } from '@earendil-works/pi-ai';

// After (verbatim behavior, one import-path change)
import { getModel, complete } from '@earendil-works/pi-ai/compat';
```

Compat 是根入口点的严格超集，因此文件可以整体切换其导入路径。它将在未来版本中移除；请迁移到 `createModels()` + 提供商工厂：

| 旧版 | 新版 |
|-----|-----|
| `getModel('openai', 'gpt-4o-mini')` | `models.getModel('openai', 'gpt-4o-mini')`，或 `getBuiltinModel()`（来自 `providers/all`） |
| `getModels('anthropic')` / `getProviders()` | `models.getModels('anthropic')` / `models.getProviders()`，或 `getBuiltin*` |
| `stream(model, ctx, opts)`（注入环境变量密钥） | `models.stream(model, ctx, opts)`（提供商身份验证解析） |
| `registerApiProvider({ api, stream, streamSimple })` | `createProvider({ id, auth, models, api })` + `models.setProvider()` |
| `getEnvApiKey('openai')` | `await models.getAuth(model.provider)` |
| `streamAnthropic(model, ctx, opts)` | `stream`（来自 `@earendil-works/pi-ai/api/anthropic-messages`），或集合中的提供商 |
| `registerFauxProvider()` | `fauxProvider()` + `models.setProvider()` |

## 开发

### 添加新提供商

添加新的 LLM 提供商需要修改多个文件。分层布局如下：API 实现位于 `src/api/`；提供商工厂位于 `src/providers/`；稳定的已生成目录包装器位于 `src/providers/<id>.models.ts`；`src/models.generated.ts` 负责注册它们。下面的检查清单涵盖所有必要步骤：

#### 1. 核心类型（`src/types.ts`）

- 如果是新 API，将 API 标识符加入 `KnownApi`（例如 `"bedrock-converse-stream"`）
- 将提供商名称加入 `KnownProvider`（例如 `"amazon-bedrock"`）
- 将选项类型加入 `ApiOptionsMap`

#### 2. API 实现（`src/api/<api-id>.ts`，仅用于新 API）

创建新的 API 实现文件（例如 `bedrock-converse-stream.ts`），仅导出 `stream` 和 `streamSimple`，此外还应包括：

- 一个扩展 `StreamOptions` 的选项接口（例如 `BedrockOptions`）
- 将 `Context` 转换为提供商格式的消息转换函数
- 如果提供商支持工具，则加入工具转换
- 解析响应并发出标准化事件（`text`、`tool_call`、`thinking`、`usage`、`stop`）

添加惰性包装器 `src/api/<api-id>.lazy.ts`（`<name>Api()`，通过 `lazyApi()` 创建），让提供商可以引用实现而不导入其 SDK。在根级别添加相应的 `export type` 重新导出，位置为 `src/index.ts`，以便继续从 `@earendil-works/pi-ai` 使用。

#### 3. 模型生成（`scripts/generate-models.ts`、`scripts/generate-image-models.ts`）

- 添加从提供商来源（例如 models.dev API）获取并解析模型的逻辑
- 将支持聊天/工具的提供商模型数据映射到标准化的 `Model` 接口；此操作通过 `scripts/generate-models.ts` 完成。数据填充过程按 API 对被忽略的 `src/providers/data/<id>.json` 值分组，而稳定的 `src/providers/<id>.models.ts` 包装器直接从这些 JSON 键中派生精确的模型/API 类型
- 将图像生成提供商的模型数据映射到标准化的 `ImagesModel` 接口；此操作通过 `scripts/generate-image-models.ts` 完成
- 处理提供商专用细节（定价格式、能力标志、模型 ID 转换）

#### 4. 提供商工厂（`src/providers/<id>.ts`）

- 使用 `createProvider()` 连接目录、身份验证和惰性 API 包装器
- 身份验证：标准密钥提供商使用 `envApiKeyAuth`；环境身份验证（AWS 配置文件、ADC）使用自定义 `ApiKeyAuth`；存在 OAuth 流程时使用 `lazyOAuth`
- 在 `src/providers/all.ts` 中注册工厂
- 如果是新 API：在 `src/compat.ts` 的内置列表中注册它，并在 `package.json` 中添加包子路径导出

#### 5. 测试（`test/`）

创建或更新测试文件以覆盖新提供商：

- `stream.test.ts`——基本流式传输和工具使用
- `tokens.test.ts`——令牌用量报告
- `abort.test.ts`——请求取消
- `empty.test.ts`——空消息处理
- `context-overflow.test.ts`——上下文限制错误
- `image-limits.test.ts`——图像支持（如适用）
- `unicode-surrogate.test.ts`——Unicode 处理
- `tool-call-without-result.test.ts`——没有结果的孤立工具调用
- `image-tool-result.test.ts`——工具结果中的图像
- `total-tokens.test.ts`——令牌计数准确性
- `cross-provider-handoff.test.ts`——跨提供商上下文重放
- `providers.test.ts`——提供商列表和身份验证解析

对于 `cross-provider-handoff.test.ts`，至少添加一个提供商/模型对。如果提供商公开多个模型系列（例如 GPT 和 Claude），则每个系列至少添加一个模型对。

对于使用非标准身份验证的提供商（AWS、Google Vertex），请创建类似 `bedrock-utils.ts` 的实用程序，包含凭据检测辅助函数。

#### 6. 编码智能体集成（`../coding-agent/`）

更新 `src/core/model-resolver.ts`：

- 在 `DEFAULT_MODELS` 中添加该提供商的默认模型 ID

更新 `src/cli/args.ts`：

- 在帮助文本中添加环境变量文档

更新 `README.md`：

- 将提供商及其设置说明添加到提供商章节

#### 7. 文档

更新 `packages/ai/README.md`：

- 添加到“支持的提供商”表格
- 记录任何提供商专用选项或身份验证要求
- 将环境变量添加到“环境变量”章节

#### 8. 变更日志

在 `packages/ai/CHANGELOG.md` 的 `## [Unreleased]` 下添加一条记录：

```markdown
### Added
- Added support for [Provider Name] provider ([#PR](link) by [@author](link))
```

## 许可证

MIT
