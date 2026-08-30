# Pi 模型与 Provider 治理层

> 系列第 06 篇。前五篇讲的是"Agent 怎么跑"，这一篇讲"跑在什么模型上"。Pi 支持 30+ Provider、订阅/API Key/OAuth 三种鉴权、内置目录/远程目录/用户自定义/扩展注册四层模型来源——把这些异构性收敛成"一个模型列表 + 一次 `/model` 选择"的，是 `ModelRuntime` 这一层。这是 Pi 里最不起眼但最影响可用性的设计。

## 目录

1. [问题：模型生态的四种异构性](#1-问题模型生态的四种异构性)
2. [ModelRuntime：统一的模型目录](#2-modelruntime统一的模型目录)
3. [鉴权体系：订阅 / API Key / OAuth 的统一抽象](#3-鉴权体系订阅--api-key--oauth-的统一抽象)
4. [models.json：声明式接入任意端点](#4-modelsjson声明式接入任意端点)
5. [compat 兼容层：不改代码适配残缺端点](#5-compat-兼容层不改代码适配残缺端点)
6. [模型能力描述：成本分层与思考等级映射](#6-模型能力描述成本分层与思考等级映射)
7. [扩展注册 Provider：代码级接入](#7-扩展注册-provider代码级接入)
8. [设计小结](#8-设计小结)

---

## 1. 问题：模型生态的四种异构性

一个要支持"任意模型"的 Agent 必须同时处理四个维度的差异：

| 维度 | 具体差异 |
|---|---|
| **协议** | Anthropic Messages / OpenAI Completions / OpenAI Responses / Google Generative AI，四套完全不同的请求响应结构 |
| **鉴权** | 环境变量 API Key / `auth.json` 存储 / OAuth 订阅（会过期需刷新）/ 云厂商签名 / 无鉴权本地服务 |
| **能力** | 是否支持 reasoning、是否支持 image 输入、上下文窗口多大、支持哪些 thinking 等级、是否支持 prompt cache、是否支持 tool reference |
| **兼容度** | 号称"OpenAI 兼容"的端点实际支持程度千差万别——有的不认 `developer` 角色，有的不认 `reasoning_effort`，有的拒绝 `ttl` 字段 |

如果不做治理，这些差异会直接泄漏到 Agent 逻辑里，变成散落各处的 `if (provider === "xxx")`。Pi 的做法是在 `pi-ai` 和 `pi-coding-agent` 之间建立一个治理层，把四种异构性分别收敛到四个明确的机制上。

---

## 2. ModelRuntime：统一的模型目录

`packages/coding-agent/src/core/model-runtime.ts` 的 `ModelRuntime` 类实现 `pi-ai` 的 `Models` 接口，是所有模型访问的唯一入口。它的内部状态说明了它在做什么：

```typescript
export class ModelRuntime implements Models {
  private readonly models: MutableModels;
  private readonly credentials: RuntimeCredentials;
  private readonly defaultBuiltins: ReadonlyMap<string, Provider>;
  private readonly builtins = new Map<string, Provider>();
  private readonly nativeExtensionProviders = new Map<string, Provider>();
  private readonly extensionProviders = new Map<string, ProviderConfigInput>();
  private readonly compositionErrors = new Map<string, string>();
  private config: ModelConfig;
  ...
}
```

### 2.1 四层模型来源的叠加顺序

四个不同的 Map 对应四层来源，叠加顺序是（后者覆盖前者）：

```
1. defaultBuiltins        内置 Provider 目录（随包发布，离线可用）
2. nativeExtensionProviders  扩展通过 pi.registerProvider(完整 Provider) 注册
3. extensionProviders     扩展通过 pi.registerProvider(id, config) 注册（legacy 形式）
4. config (models.json)   用户自定义，优先级最高
```

官方文档明确了这个顺序："Pi composes `models.json` overrides above registered native providers"——**用户手写的配置永远赢**。这是一个重要的产品决策：扩展作者不能通过注册 Provider 来劫持用户已经明确配置好的端点。

### 2.2 快照机制：把"当前有哪些可用模型"物化下来

```typescript
interface ModelRuntimeSnapshot {
  all: readonly Model<Api>[];
  available: readonly Model<Api>[];
  configuredProviders: ReadonlySet<string>;
  storedProviders: ReadonlySet<string>;
  auth: ReadonlyMap<string, AuthCheck | undefined>;
}
```

`all` 与 `available` 的区分是关键：**模型加载成功 ≠ 模型可用**。一个在 `models.json` 里声明了但没有配置任何鉴权的模型会出现在 `all` 里、不出现在 `available` 里，`/model` 选择器和 `--list-models` 只展示 `available`。

这样设计的好处是错误定位清晰：用户看不到自己刚加的模型时，能区分两种情况——"配置文件没被解析"（`all` 里也没有）vs"配置解析了但缺鉴权"（`all` 有、`available` 没有）。`compositionErrors` 这个 Map 则专门保留"这个 Provider 组装失败的原因"，让第一种情况有具体错误信息而不是静默消失。

### 2.3 网络访问是显式选项，不是默认行为

```typescript
export interface CreateModelRuntimeOptions {
  /** Allow create() to refresh model catalogs over the network. Defaults to false. */
  allowModelNetwork?: boolean;
  modelRefreshTimeoutMs?: number;
  /** Skip initial catalog and availability refresh. Static models remain available. */
  refreshOnCreate?: boolean;
  signal?: AbortSignal;
  ...
}
```

`allowModelNetwork` **默认 false**——`ModelRuntime.create()` 默认不联网。内置目录随包发布，刷新到的远程目录缓存在 `~/.pi/agent/models-store.json` 供离线使用。这对应 `--offline`/`PI_OFFLINE=1` 的整体离线模式，也保证了"网络不通时 Pi 依然能启动并用已知模型工作"。

`signal?: AbortSignal` 出现在这里也值得注意——连"初始化模型目录"这种启动期操作都可中断，用户不会因为一个卡住的目录刷新请求而无法 Ctrl+C 退出。

### 2.4 Header 合并的大小写处理

```typescript
function mergeHeaders(base, override) {
  const merged = { ...base };
  for (const [name, value] of Object.entries(override ?? {})) {
    const lowerName = name.toLowerCase();
    for (const existingName of Object.keys(merged)) {
      if (existingName.toLowerCase() === lowerName) delete merged[existingName];
    }
    merged[name] = value;
  }
  return merged;
}
```

HTTP header 名大小写不敏感，但 JS 对象的 key 是敏感的。如果不做这个处理，`{ "Authorization": "..." }` 和 `{ "authorization": "..." }` 会同时存在于一个请求里——多数服务端会因此报错或行为不确定。这类"协议语义与语言语义不匹配"的地方是最容易埋 bug 的位置，Pi 显式处理了它。

---

## 3. 鉴权体系：订阅 / API Key / OAuth 的统一抽象

### 3.1 三条鉴权路径

| 路径 | 存储位置 | 特点 |
|---|---|---|
| 环境变量 | `ANTHROPIC_API_KEY` 等 | 最适合 CI；映射表见 `packages/ai/src/env-api-keys.ts` |
| `auth.json` | `~/.pi/agent/auth.json` | `/login` 写入，`{ "anthropic": { "type": "api_key", "key": "..." } }` |
| OAuth 订阅 | `~/.pi/agent/auth.json` | ChatGPT Plus/Pro (Codex)、Claude Pro/Max、GitHub Copilot、xAI、OpenRouter、Radius；token 过期自动刷新 |

解析顺序是"显式传入 → Provider 已存储凭证 → 环境变量/OAuth"，`minOAuthValidityMs` 默认要求剩余有效期至少 5 分钟：

```typescript
export interface ModelRuntimeAuthOverrides extends AuthOperationOptions {
  apiKey?: string;
  env?: Record<string, string>;
  /** Require this much remaining OAuth-token validity; defaults to five minutes. */
  minOAuthValidityMs?: number;
}
```

"提前 5 分钟就认为该刷新了"避免了一类典型竞态：token 检查时还有 10 秒有效期，请求发出时已经过期。

### 3.2 凭证写入与本地状态同步的解耦

```typescript
export type CredentialSynchronizationOperation = "login" | "logout" | "setRuntimeApiKey" | "removeRuntimeApiKey";

/** Credentials changed successfully, but the local model/auth snapshot could not be synchronized. */
export class CredentialSynchronizationError extends Error {
  readonly providerId: string;
  readonly operation: CredentialSynchronizationOperation;
  readonly credential: Credential | undefined;
  constructor(providerId, operation, credential, options) {
    super(`Credential ${operation} committed for ${providerId}, but local synchronization failed`, options);
    ...
  }
}
```

这个专门的错误类型解决一个具体的两阶段问题："凭证已经成功写入磁盘"和"本地模型可用性快照已经更新"是两件事，前者成功后者失败时，**不能简单报"登录失败"**——那会导致用户重复登录，而凭证其实已经存好了。

错误对象携带 `credential` 字段，让调用方能拿到已提交的凭证做补救（比如提示"登录已完成，请重启 Pi"而不是"登录失败，请重试"）。这是"分布式事务的部分成功"这一经典问题在本地文件层面的体现，Pi 没有把它藏起来。

---

## 4. models.json：声明式接入任意端点

`~/.pi/agent/models.json` 是不写代码接入模型的方式。最小配置只需要 4 个字段：

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [{ "id": "llama3.1:8b" }, { "id": "qwen2.5-coder:7b" }]
    }
  }
}
```

四种支持的 API 类型：`openai-completions`（最通用）、`openai-responses`、`anthropic-messages`、`google-generative-ai`，可在 provider 级设默认、model 级覆盖。

### 4.1 `apiKey: "ollama"` 这个占位符为什么必需

官方文档解释得很直接：Ollama 忽略 API Key，但**Pi 仍然把"有无鉴权配置"作为模型是否出现在 `/model` 里的判据**（第 2.2 节的 `available` 逻辑）。所以无鉴权的本地服务需要留一个假值、或用 `/login` 存一个 key、或 `--api-key` 传一个。

这是一个有意的取舍：统一的可用性判据（有鉴权才算可用）带来了"本地服务需要填假 key"的小别扭，但避免了"某些 Provider 不需要鉴权"这个特例渗透进可用性判断逻辑的所有分支。

### 4.2 值解析：`!command` / `$ENV` / 转义

`apiKey` 和 `headers` 的值支持三种形式：

```json
"apiKey": "!security find-generic-password -ws 'anthropic'"   // 执行命令取 stdout
"apiKey": "!op read 'op://vault/item/credential'"             // 1Password CLI
"apiKey": "$MY_API_KEY"                                        // 环境变量
"apiKey": "${KEY_PREFIX}_${KEY_SUFFIX}"                        // 插值进更大的字面量
"apiKey": "$$literal-dollar-prefix"                            // 转义：字面量 $
"apiKey": "$!literal-bang-prefix"                              // 转义：字面量 !
"apiKey": "sk-..."                                             // 字面量
```

`!command` 支持让"从系统钥匙串/密码管理器动态取密钥"成为一行配置，密钥不需要落在明文文件里。

**关键设计决策**——文档明确说明 Pi 故意不为这些命令提供 TTL / 陈旧值复用 / 失败恢复：

> pi intentionally does not apply built-in TTL, stale reuse, or recovery logic for arbitrary commands. Different commands need different caching and failure strategies, and pi cannot infer the right one.

理由很硬：`security find-generic-password` 是本地即时操作、`op read` 可能触发生物识别、某个自建脚本可能调远程 API 有速率限制——三者需要的缓存策略完全不同，Pi 无法推断。所以把责任明确交还用户："如果你的命令慢/贵/有限流，自己包一层脚本实现缓存"。

**这类"明确拒绝提供某个功能并说明为什么"的设计，比提供一个猜测性的默认行为更有价值**——后者会在用户不知情时产生难以排查的行为（比如密钥被缓存了 5 分钟，轮转后一直用旧值）。

配套的一个细节：`/model` 的可用性检查**不执行** shell 命令，只看"是否配置了鉴权"。否则打开一次模型选择器就会触发几十次钥匙串访问/生物识别弹窗。

### 4.3 热重载

`models.json` 每次打开 `/model` 都会重新读取——会话中途编辑配置文件、加一个新模型，不需要重启 Pi。这与扩展/技能的 `/reload` 是同一套"配置即数据、随时可换"的理念。

---

## 5. compat 兼容层：不改代码适配残缺端点

这是整个治理层里最实用的机制。"OpenAI 兼容"是一个宽泛的说法，实际实现程度差异巨大。Pi 用一个 `compat` 对象把这些差异全部变成配置：

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [{ "id": "gpt-oss:20b", "reasoning": true }]
    }
  }
}
```

### 5.1 已在前面文章中出现过的 compat 字段

回顾前面几篇文章，`compat` 已经反复出现，这里可以看到它们其实是同一个机制的不同字段：

| 字段 | 作用 | 出现在 |
|---|---|---|
| `supportsDeveloperRole` | 不支持时把系统提示词作为 `system` 消息发送 | 本篇 |
| `supportsReasoningEffort` | 不支持时不发 `reasoning_effort` | 本篇 |
| `supportsLongCacheRetention` | 不支持时不发 `ttl` 字段 | 05 §2.1 |
| `supportsExplicitPromptCacheMode` | 支持时可用 `mode: "explicit"` 彻底关缓存 | 05 §2.3 |
| `sessionAffinityFormat` | 三种会话亲和性 header 格式 | 05 §4 |
| `thinkingTokenBudgetField` | 指定思考 token 预算字段名 | 05 §9 |

**`compat` 是"能力探测而非能力假设"这一原则的统一载体**。Pi 的每一个 Provider 侧优化（缓存、思考预算、会话亲和）都配了对应的 compat 开关，保证"这个优化在某个端点上不支持"永远是一个配置问题而不是代码问题。

### 5.2 provider 级与 model 级的合并

`compat` 可以设在 provider 级（对该 provider 全部模型生效）或 model 级（覆盖单个模型），两者都设时会**合并**而非替换。这对"同一个端点上跑多个能力不同的模型"很必要——比如一个 vLLM 服务同时部署了 reasoning 和非 reasoning 模型。

---

## 6. 模型能力描述：成本分层与思考等级映射

### 6.1 分层定价（Cost Tiers）

```json
{
  "cost": {
    "input": 5, "output": 30, "cacheRead": 0.5, "cacheWrite": 6.25,
    "tiers": [
      { "inputTokensAbove": 272000, "input": 10, "output": 45, "cacheRead": 1, "cacheWrite": 12.5 }
    ]
  }
}
```

规则："a cost tier supplies a complete alternate rate set and applies to the full request when total input usage (`input + cacheRead + cacheWrite`) exceeds `inputTokensAbove`. When multiple tiers match, the highest threshold wins."

三个要点：

- **整套替换而非部分覆盖**：tier 提供完整的一套费率，不是"只改 input 单价"。避免了"改了 input 但忘了改 cacheRead"导致的成本计算错误。
- **判据是 `input + cacheRead + cacheWrite` 总量**：这与第 05 篇的成本模型一致——分层判定看的是总输入规模，不管这些 token 落在哪个桶。
- **应用于整个请求**：不是"超出部分按高价"，而是"整个请求都按高价"。这是 Gemini 等模型的真实计费方式，Pi 如实建模而不是简化成阶梯累加。

这个机制直接支撑了第 05 篇讨论的成本可观测性——`cache-stats.ts` 之所以能"按消息自身 `usage.cost` 反推实际单价"算出准确的浪费金额，前提是定价模型本身足够准确地反映了分层。

### 6.2 thinkingLevelMap：三态映射

Pi 有 7 个统一的思考等级（`off`/`minimal`/`low`/`medium`/`high`/`xhigh`/`max`），但没有模型全部支持。`thinkingLevelMap` 用三态描述：

| 值 | 含义 |
|---|---|
| 省略 | `high` 及以下用 provider 默认映射；`xhigh`/`max` 视为不支持 |
| 字符串 | 该等级支持，发送这个值给 provider |
| `null` | 该等级不支持，UI 中隐藏 / 切换时跳过 / 数值被 clamp 掉 |

`null` 与"省略"的区分是必要的：省略表示"用默认推断"，`null` 表示"明确知道不支持"。有了 `null`，Shift+Tab 循环切换思考等级时会自动跳过不支持的档位，而不是让用户切到一个会导致请求失败的等级。

### 6.3 samplingParams：逃生舱

```json
{
  "id": "deepseek-v4-flash",
  "samplingParams": { "temperature": 1.0, "top_p": 0.95, "top_k": 0, "min_p": 0.0 }
}
```

"merged verbatim into every request body for the model, after the fields pi sets itself, so its keys win"——原样合并、且**覆盖 Pi 自己设的字段**。这是为"Pi 没有建模的采样参数"（llama.cpp 的 `min_p`、vLLM 的 `top_k`）留的逃生舱。

只有 OpenAI 兼容的三种 API 会应用它，其他 API 忽略。文档还特意提醒：可以在这里塞一个固定的思考 token 上限，但那样**不会跟随 `thinkingBudgets`、也不会为答案留空间**，所以应优先用 `compat.thinkingTokenBudgetField`——又一次"提供逃生舱但明确说明它的代价"。

---

## 7. 扩展注册 Provider：代码级接入

`models.json` 解决不了的场景（需要自定义 OAuth 流程、非标准流式协议、动态模型发现），由扩展的 `pi.registerProvider()` 承担。两种形式：

**完整 Provider（推荐，能力最全）：**

```typescript
import { createProvider, openAICompletionsApi } from "@earendil-works/pi-ai";

pi.registerProvider(createProvider({
  id: "native-local",
  name: "Native Local",
  baseUrl: "http://localhost:8080/v1",
  auth: {
    apiKey: {
      name: "Local server API key",
      async login(interaction) {
        return { type: "api_key", key: await interaction.prompt({ type: "secret", message: "API key" }) };
      },
      async resolve({ credential }) {
        return credential?.key ? { auth: { apiKey: credential.key }, source: "stored API key" } : undefined;
      },
    },
  },
  models: [],
  api: openAICompletionsApi(),
}));
```

**legacy config 形式（简单场景）：**

```typescript
pi.registerProvider("anthropic", { baseUrl: "https://proxy.example.com" });   // 只改端点，走公司代理
```

一个重要的时序细节：

> The extension factory can also be `async`. For dynamic model discovery, fetch and register models in the factory instead of `session_start`. pi waits for the factory before startup continues, so the provider is available during interactive startup and to `pi --list-models`.

**动态模型发现要放在扩展工厂函数里，不能放在 `session_start` 事件里**——因为 `--list-models` 和交互启动时的模型选择都发生在 `session_start` 之前。Pi 会等待工厂函数完成才继续启动，这是唯一能保证"注册的 Provider 在所有入口都可见"的时机。这类"扩展点的执行时机决定了它能做什么"的约束，是扩展系统设计里最容易出错的部分，官方文档把它明确写出来了。

`validateExtensionProvider()`（`model-runtime.ts` 的导入之一）负责校验扩展注册的 Provider 结构合法性，失败原因进入前面提到的 `compositionErrors`。

---

## 8. 设计小结

1. **四种异构性 → 四个专门机制**：协议差异由 `pi-ai` 的四种 API 实现吸收；鉴权差异由 `RuntimeCredentials` + 三条解析路径吸收；能力差异由模型描述字段（`cost.tiers` / `thinkingLevelMap` / `input` / `contextWindow`）吸收；兼容度差异由 `compat` 吸收。每一类差异都有唯一的归口，不会散落成条件分支。

2. **"加载成功"与"可用"严格区分**（`all` vs `available` vs `compositionErrors`），让配置错误可定位。这个看起来很小的设计，直接决定了用户接入自建端点时是"5 分钟搞定"还是"折腾一小时不知道哪错了"。

3. **明确拒绝猜测性默认行为**。`!command` 不提供内置 TTL/重试，因为不同命令需要不同策略且无法推断；这种"少做但说清楚"比"多做但可能做错"更负责。同样地，`samplingParams` 提供了逃生舱，但文档同时说明了用它设置思考预算的代价。

4. **优先级顺序是产品决策而非实现细节**：`models.json` 覆盖扩展注册的 Provider，保证扩展不能劫持用户的显式配置；网络访问默认关闭，保证离线可用。这两条都不是技术上的必然选择，而是明确的价值取向。
