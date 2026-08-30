# Pi Agent 的 Token 成本控制：为什么核心是 Prompt Cache 前缀稳定性

> 系列第 05 篇（[系列索引](README.md)）。Coding Agent 的成本结构与普通 LLM 应用有本质差异：每一轮对话都要重新发送整个对话历史（系统提示词 + 工具定义 + 全部消息），一个跑了 30 轮的会话，累计发送的 prompt token 是单轮的几十倍。因此"降低成本"的核心不是"少说话"，而是**让重复发送的部分尽可能命中 Prompt Cache**。本文从成本模型出发，逐层拆解 Pi 在这个问题上的具体设计。

## 目录

1. [先搞清楚钱花在哪：Coding Agent 的成本模型](#1-先搞清楚钱花在哪coding-agent-的成本模型)
2. [第一杠杆：Prompt Cache 的写入点设计](#2-第一杠杆prompt-cache-的写入点设计)
3. [核心洞察：前缀稳定性与 Deferred Tools](#3-核心洞察前缀稳定性与-deferred-tools)
4. [会话亲和性：让请求落到持有缓存的那台机器](#4-会话亲和性让请求落到持有缓存的那台机器)
5. [缓存浪费的量化与暴露：cache-stats.ts](#5-缓存浪费的量化与暴露cache-statsts)
6. [上下文压缩的成本权衡](#6-上下文压缩的成本权衡)
7. [渐进式披露：技能与工具清单](#7-渐进式披露技能与工具清单)
8. [工具输出治理：截断而非全量入上下文](#8-工具输出治理截断而非全量入上下文)
9. [思考等级：推理 Token 的显式预算控制](#9-思考等级推理-token-的显式预算控制)
10. [异构模型路由：便宜模型做便宜的事](#10-异构模型路由便宜模型做便宜的事)
11. [机制总览与优先级](#11-机制总览与优先级)

---

## 1. 先搞清楚钱花在哪：Coding Agent 的成本模型

`pi-ai` 的 `Usage` 类型把每次请求的 Token 分成四个计费桶，这四个桶的单价差异极大（以典型 Anthropic 定价为例，量级关系是 `cacheRead ≈ input/10`，`cacheWrite ≈ input × 1.25`）：

| 桶             | 含义                                                  | 相对单价     |
| -------------- | ----------------------------------------------------- | ------------ |
| `input`      | 未命中缓存、按全价计费的 prompt token                 | 基准 1×     |
| `cacheWrite` | 写入缓存的 prompt token（有写入溢价）                 | ~1.25×      |
| `cacheRead`  | 命中缓存的 prompt token                               | ~0.1×       |
| `output`     | 模型生成的 token（**包含推理/thinking token**） | 数倍于 input |

关键推论：**一个长会话里绝大部分 Token 是重复发送的 prompt**。如果这些 token 全部落在 `input` 桶，成本是落在 `cacheRead` 桶的约 10 倍。因此 Pi 的成本优化设计里，**Prompt Cache 命中率是压倒性的第一优先级**，其他手段（压缩、截断、模型路由）都是次要的。

这也解释了为什么 Pi 的 TUI 页脚要专门显示这四个桶的实时数据：

```typescript
if (usageTotals.input) statsParts.push(`↑${formatTokens(usageTotals.input)}`);
if (usageTotals.output) statsParts.push(`↓${formatTokens(usageTotals.output)}`);
if (usageTotals.cacheRead) statsParts.push(`R${formatTokens(usageTotals.cacheRead)}`);
if (usageTotals.cacheWrite) statsParts.push(`W${formatTokens(usageTotals.cacheWrite)}`);
if ((usageTotals.cacheRead > 0 || usageTotals.cacheWrite > 0) && latestCacheHitRate !== undefined) {
  statsParts.push(`CH${latestCacheHitRate.toFixed(1)}%`);
}
```

`CH` 就是缓存命中率——它被放在页脚这个"始终可见"的位置，而不是藏在某个 `/stats` 命令里，是一个明确的设计表态：**缓存命中率是用户需要持续关注的一等指标**。

---

## 2. 第一杠杆：Prompt Cache 的写入点设计

### 2.1 三档缓存保留策略

`CacheRetention` 是一个三值枚举，默认 `"short"`：

```typescript
function resolveCacheRetention(cacheRetention?: CacheRetention, env?: ProviderEnv): CacheRetention {
  if (cacheRetention) return cacheRetention;
  if (getProviderEnvValue("PI_CACHE_RETENTION", env) === "long") return "long";
  return "short";
}
```

- `"short"`（默认）：使用 Provider 的标准缓存 TTL（Anthropic 默认 5 分钟）
- `"long"`：Anthropic 用 `cache_control.ttl: "1h"`，OpenAI 用 `prompt_cache_retention: "24h"`——适合"边想边做、经常离开几十分钟再回来"的开发节奏，用更高的缓存写入费换取跨长时间间隔的命中
- `"none"`：完全关闭缓存——用于一次性调用（见第 10 节）

注意 `"long"` 还要 Provider 侧确认支持：`retention === "long" && getAnthropicCompat(model).supportsLongCacheRetention` 才真的加 `ttl`。这是因为很多 Anthropic 兼容的第三方代理不接受 `ttl` 字段，盲目发送会导致整个请求被拒——这类"能力探测而非假设"的 compat 字段在 Pi 里非常普遍。

### 2.2 Anthropic：三个缓存断点的精确放置

Anthropic 的 Prompt Cache 需要显式标记断点（`cache_control`），Pi 在三个位置放置断点，覆盖"每轮都重复发送"的三大块内容：

**断点一：系统提示词**

```typescript
} else if (context.systemPrompt) {
  params.system = [{
    type: "text",
    text: sanitizeSurrogates(context.systemPrompt),
    ...(cacheControl ? { cache_control: cacheControl } : {}),
  }];
}
```

OAuth 场景下还会多一个前置 block（Claude Code 身份声明），两个 block 都加缓存标记——保证不管走 API Key 还是订阅 OAuth，系统提示词都被缓存。

**断点二：最后一个工具定义**

```typescript
...(cacheControl && index === tools.length - 1 ? { cache_control: cacheControl } : {}),
```

只在**最后一个**工具上打标记，而不是每个工具都打——Anthropic 的缓存断点语义是"缓存到这个位置为止的全部前缀"，所以标记最后一个工具就等于缓存了"系统提示词 + 全部工具定义"这整段前缀。断点数量本身是有配额限制的（Anthropic 最多 4 个），这种"只在段落末尾打一个"的放置方式把配额用在了最有价值的地方。

**断点三：最后一条 user 消息的最后一个 block**

```typescript
// Add cache_control to the last user message to cache conversation history
if (cacheControl && params.length > 0) {
  const lastMessage = params[params.length - 1];
  if (lastMessage.role === "user") {
    const lastBlock = lastMessage.content[lastMessage.content.length - 1];
    if (lastBlock && (lastBlock.type === "text" || lastBlock.type === "image" || lastBlock.type === "tool_result")) {
      (lastBlock as any).cache_control = cacheControl;
    }
  }
}
```

这个断点是"滚动"的——每一轮对话，缓存边界都往后推到最新的用户消息/工具结果。效果是：第 N 轮把前 N-1 轮的完整历史写入缓存，第 N+1 轮就能把这段历史整体作为 `cacheRead` 读回来。**这是长会话成本控制的主力机制**：会话越长，这个断点带来的节省越大。

注意它显式检查了 `tool_result` 类型——因为在 Agent 场景里，最后一条 user 角色消息经常是一批工具结果（而非用户手打的文字），如果只处理 `text` 类型，工具调用密集的轮次就会全部漏掉缓存。这是一个"通用 LLM SDK 容易忽略、但 Agent 场景必须处理"的细节。

### 2.3 OpenAI：cache key + 显式模式

OpenAI 的 Prompt Cache 是自动的（不需要断点标记），但需要一个 cache key 来提高路由命中率：

```typescript
const disableImplicitPromptCache = cacheRetention === "none" && compat.supportsExplicitPromptCacheMode;
const params = {
  model: model.id,
  input: messages,
  stream: true,
  prompt_cache_key: cacheRetention === "none" ? undefined : clampOpenAIPromptCacheKey(options?.sessionId),
  prompt_cache_retention: getPromptCacheRetention(compat, cacheRetention),
  prompt_cache_options: disableImplicitPromptCache ? { mode: "explicit" } : undefined,
  store: false,
};
```

三个细节：

- **`prompt_cache_key` 用 sessionId**：同一个 Pi 会话的所有请求共享一个 cache key，让 OpenAI 侧能把它们识别为"同一个不断增长的前缀"。
- **`clampOpenAIPromptCacheKey` 截断到 64 字符**（`OPENAI_PROMPT_CACHE_KEY_MAX_LENGTH = 64`）：这是一个防御性处理——超长 key 会被 API 拒绝，与其让请求整体失败，不如截断（截断后仍然是同一会话内稳定的 key，不影响命中）。截断用 `Array.from(key)` 按码点而非字节切分，避免把一个 emoji 或中文字符切成半个非法 UTF-8 序列。
- **`store: false`**：不让 Provider 侧持久化请求内容。这不是成本优化，而是隐私默认值——但值得一起提，因为它说明 Pi 在"用 Provider 的缓存"和"让 Provider 存我的数据"之间做了明确区分：**用缓存不等于同意存档**。

`cacheRetention: "none"` 时不仅不发 cache key，还会在支持的 Provider 上主动发 `prompt_cache_options: { mode: "explicit" }` 把隐式缓存也关掉——"关闭"是真的彻底关闭，不是"我不主动用但你随便"。

---

## 3. 核心洞察：前缀稳定性与 Deferred Tools

这是 Pi 在 Token 成本上最精妙的一处设计，也是最容易被忽略的。

### 3.1 问题：中途新增一个工具，会击穿整个缓存前缀

Prompt Cache 的匹配是**前缀匹配**：请求的前 N 个 token 与缓存内容完全一致才能命中。而工具定义位于请求的最前面（系统提示词之后、消息之前）。

考虑这个场景：对话进行到第 10 轮，某个技能被加载、或某个扩展动态注册了一个新工具。如果把这个新工具追加进工具定义列表，请求的前缀就变了——**前 10 轮辛苦累积的整个缓存前缀（系统提示词 + 工具定义 + 10 轮历史）全部作废**，第 11 轮要按全价重新发送并重新写入缓存。对一个已经有 10 万 token 历史的会话，这一次"新增工具"的代价是 10 万 token 的全价重新计费。

### 3.2 解法：把中途新增的工具放进 transcript，而不是前缀

`splitDeferredTools()`（`packages/ai/src/utils/deferred-tools.ts`，仅 39 行但信息密度极高）就是这个问题的答案：

```typescript
export function splitDeferredTools(
  context: Context,
  enabled: boolean,
  normalizeName: ToolNameNormalizer = identityToolName,
): { immediate: Tool[]; deferred: Map<string, Tool> } {
  const uniqueTools = new Map<string, Tool>();
  for (const tool of context.tools ?? []) uniqueTools.set(normalizeName(tool.name), tool);
  if (!enabled) return { immediate: [...uniqueTools.values()], deferred: new Map() };

  const deferredNames = new Set<string>();
  const usedNames = new Set<string>();
  for (const message of context.messages) {
    if (message.role === "assistant") {
      for (const block of message.content) {
        if (block.type === "toolCall") usedNames.add(normalizeName(block.name));
      }
    } else if (message.role === "toolResult") {
      for (const name of message.addedToolNames ?? []) {
        const normalizedName = normalizeName(name);
        if (!usedNames.has(normalizedName)) deferredNames.add(normalizedName);
      }
    }
  }

  const immediate: Tool[] = [];
  const deferred = new Map<string, Tool>();
  for (const [name, tool] of uniqueTools) {
    if (deferredNames.has(name)) deferred.set(name, tool);
    else immediate.push(tool);
  }
  return { immediate, deferred };
}
```

逻辑拆解：

1. **toolResult.addedToolNames `** 是关键数据来源——当某次工具调用（比如加载一个技能）导致新工具被注册时，这个工具结果消息会记录"我新增了哪些工具"。
2. 这些"中途新增"的工具被归入 `deferred`，**不进入工具定义前缀**，而是序列化到 transcript 里（Anthropic 用 `tool_reference` block，OpenAI 用 `tool_search_output` item），并带上 `defer_loading: true`。
3. transcript 是**追加式**的——在消息历史末尾追加内容不会破坏前面的缓存前缀。于是"新增一个工具"的代价从"击穿 10 万 token 前缀"降到"在 transcript 末尾多几百 token"。
4. **`usedNames` 的作用**：如果一个工具已经被实际调用过（`toolCall` 出现在 assistant 消息里），说明它的定义已经稳定存在于某个位置，就不再需要 defer 处理了——避免对已稳定的工具做无意义的搬移。

### 3.3 兜底与能力探测

```typescript
let immediateTools = toolPlacement.immediate;
let deferredTools = [...toolPlacement.deferred.values()];
if (immediateTools.length === 0 && deferredTools.length > 0) {
  immediateTools = deferredTools;
  deferredTools = [];
}
```

如果所有工具都被判为 deferred（极端情况：这个会话的全部工具都是中途加载的），会把它们全部提回 immediate——因为"工具定义列表为空但 transcript 里有工具引用"对多数 Provider 是非法请求。这是"优化不能以破坏正确性为代价"的具体落地。

能力探测也很保守（`defaultSupportsToolReferences`）：

```typescript
function defaultSupportsToolReferences(model: Model<"anthropic-messages">): boolean {
  if (model.provider !== "anthropic" || model.id.includes("haiku")) return false;
  const version = model.id.match(/^claude-(?:opus|sonnet|fable)-(\d+)(?:-(\d+))?(?:-|$)/);
  if (!version) return false;
  const major = Number(version[1]);
  const minor = version[2] && version[2].length < 8 ? Number(version[2]) : 0;
  return major > 4 || (major === 4 && minor >= 5);
}
```

默认只对"第一方 Anthropic、非 Haiku、Claude 4.5 及以上"启用——Haiku 会拒绝客户端发来的 `tool_reference` block，4.5 以前的模型不支持 tool search。**默认关闭、白名单开启**：一个能省钱但可能让请求失败的优化，宁可少省一点也不能让用户的请求报错。

---

## 4. 会话亲和性：让请求落到持有缓存的那台机器

Prompt Cache 是存在 Provider 后端某个具体节点上的。如果第 11 轮请求被负载均衡路由到另一个节点，即便前缀完全一致也会 miss。Pi 因此在启用缓存时发送会话亲和性 header：

```typescript
if (sessionId) {
  if (compat.sessionAffinityFormat === "openrouter") {
    headers["x-session-id"] = sessionId;
  } else {
    if (compat.sessionAffinityFormat === "openai") {
      headers.session_id = sessionId;
    }
    headers["x-client-request-id"] = sessionId;
  }
}
```

三种 header 格式（`openai` / `openai-nosession` / `openrouter`）通过 `sessionAffinityFormat` 区分，默认自动探测：

```typescript
function detectSessionAffinityFormat(model) {
  return model.provider === "openrouter" || model.baseUrl.includes("openrouter.ai") ? "openrouter" : "openai";
}
```

`"openai-nosession"` 这个变体的存在很说明问题——某些代理/网关不接受带下划线的 `session_id` header（会当成非法 header 拒绝），所以需要一个"发 `x-client-request-id` 但不发 `session_id`"的模式。同样地，`cacheRetention === "none"` 时 `cacheSessionId` 会被置为 `undefined`（`const cacheSessionId = cacheRetention === "none" ? undefined : options?.sessionId;`）——不用缓存就不发亲和性 header，保持行为一致。

---

## 5. 缓存浪费的量化与暴露：cache-stats.ts

有了缓存机制还不够——用户需要知道"我的缓存到底命中了没有、没命中损失了多少钱"。`packages/coding-agent/src/core/cache-stats.ts` 专门做这件事，是 Pi 里成本可观测性做得最细的一个模块。

### 5.1 miss 的定义与计算

```typescript
const missedTokens = Math.min(prev.promptTokens, promptTokens) - usage.cacheRead;
if (missedTokens <= NOISE_FLOOR_TOKENS) return undefined;   // NOISE_FLOOR_TOKENS = 1024
```

定义："上一轮请求的 prompt 里已经有的内容（理论上这轮应该整段从缓存读），实际却没从缓存读到的部分"。取 `min(prev.promptTokens, promptTokens)` 是因为这轮 prompt 可能比上轮短（压缩后），只能按两者较小值来算"本应命中的上限"。

`NOISE_FLOOR_TOKENS = 1024` 的噪声地板很重要：缓存断点有粒度（block 级别），每轮总会有几百 token 落在断点之外，这是机制固有的，不算浪费。只有超过 1024 token 的 miss 才计入——**避免每一轮都报一个无意义的"你浪费了 200 token"警告，让告警本身失去信号价值**。

### 5.2 成本计算：用这条消息自己的实际单价

```typescript
const paidTokens = usage.input + usage.cacheWrite;
const paidPerToken = paidTokens > 0 ? (usage.cost.input + usage.cost.cacheWrite) / paidTokens : 0;
const readPerToken =
  usage.cacheRead > 0
    ? usage.cost.cacheRead / usage.cacheRead
    : (models.getModel(message.provider, message.model)?.cost.cacheRead ?? 0) / 1_000_000;

return {
  missedTokens,
  missedCost: missedTokens * Math.max(0, paidPerToken - readPerToken),
  ...
};
```

不用静态定价表算，而是**从这条消息自己的 `usage.cost` 反推实际单价**——因为 miss 的 token 只可能落在 `input` 或 `cacheWrite` 桶里，这条消息的成本明细里就直接包含了它们的真实计费单价（含 cacheWrite 溢价）。这样即便 Provider 有促销折扣、阶梯定价或者定价表过期，算出的"多花了多少钱"依然准确。`Math.max(0, ...)` 保证异常定价数据不会产生负成本。

### 5.3 什么情况不算浪费：压缩豁免、模型切换不豁免

```typescript
if (entry.type === "compaction" || entry.type === "branch_summary") {
  // The context legitimately changed; the next turn's prompt is new content,
  // not re-billed content. Model switches are NOT exempt: they re-bill the
  // full prompt and should be counted.
  prev = undefined;
  continue;
}
```

这段注释里的判断标准非常清晰：

- **压缩/分支摘要之后不算 miss**：上下文被合法地重写了，下一轮的 prompt 是**新内容**而不是"本该命中缓存的旧内容"，把它算成浪费是错误归因。
- **模型切换算 miss**：切模型确实会导致整个 prompt 按全价重新计费，这是真实发生的成本，用户应该知道。切模型可能是完全合理的决定（难题换强模型），但**代价必须可见**——这是"呈现事实、不替用户做价值判断"的设计态度。

### 5.4 `reportedCache` 粘性标记：区分"没命中"和"这家不支持缓存"

```typescript
if (!prev || promptTokens <= 0 || (usage.cacheRead + usage.cacheWrite === 0 && !prev.reportedCache)) {
  return undefined;
}
```

有些 Provider（OpenAI 风格）只报告 `cacheRead`、不报告 `cacheWrite`；有些 Provider 完全不报告缓存数据。如果不做区分，对"完全不支持缓存的 Provider"每一轮都会报"100% 缓存未命中，你浪费了全部 token"——这是纯噪声。

`reportedCache` 是一个**粘性**标记：只要这段扫描区间内**曾经**有任何一轮报告过缓存活动，就认为"这个 Provider 是支持缓存的"，此后出现的零缓存轮次才被当作真实 miss。逻辑上很微妙但效果关键：在支持缓存的 Provider 上，一次彻底的 miss 能被准确捕获；在不支持缓存的 Provider 上，完全不产生噪声告警。

### 5.5 UI 暴露：只在损失显著时提示，且只陈述可观测事实

```typescript
private addCacheMissNotice(miss: CacheMiss): void {
  if (miss.missedTokens < 20_000 && miss.missedCost < 0.1) return;

  const cost = miss.missedCost >= 0.01 ? ` (~$${miss.missedCost.toFixed(2)})` : "";
  const reBilled = `${formatTokens(miss.missedTokens)} tokens re-billed${cost}`;
  let label = "Cache miss";
  if (miss.modelChanged) {
    label = "Cache miss after model switch";
  } else if (miss.idleMs >= CACHE_TTL_MS) {
    label = `Cache miss after ${Math.round(miss.idleMs / 60_000)}m idle`;
  }
  ...
}
```

三层过滤保证提示的信噪比：

1. **双阈值**：`missedTokens >= 20000` **或** `missedCost >= $0.1` 才提示——用"或"而不是"且"，兼顾"便宜模型上量大"和"贵模型上量小"两种都值得关注的情况。
2. **归因只陈述可观测事实**：源码注释明确写了 "Only states observable facts: the miss itself, a model switch, or an idle gap past the cache TTL"——切换了模型、或者空闲超过缓存 TTL（`CACHE_TTL_MS = 5 * 60 * 1000`，Anthropic 默认值），这两件事是**确定发生的**；不会去猜"可能是因为你改了系统提示词"这类无法证实的原因。**宁可少解释，不可错误归因**。
3. **可关闭**：`getShowCacheMissNotices()` 是一个设置项，不想看的人可以关掉；关掉后 `rebuildChatFromMessages()` 会重建整个聊天视图移除已显示的提示。

另外，缓存提示**不持久化**为会话条目，而是在 resume/压缩后重建视图时用 `collectCacheMisses()` 从完整 entry 列表**重新推导**——保证这些提示始终与实际数据一致，不会出现"会话文件里存了一条过时的提示"这种状态不一致。

### 5.6 压缩本身的成本也要显示

```typescript
private addCompactionCostNotice(notice: CompactionCostNotice): void {
  const { usage } = notice;
  const tokens = usage.input + usage.output + usage.cacheRead + usage.cacheWrite;
  const cost = usage.cost.total >= 0.01 ? ` (~$${usage.cost.total.toFixed(2)})` : "";
  const label = notice.kind === "compaction" ? "Compaction" : "Branch summary";
  ...`${label}: ${formatTokens(tokens)} tokens billed${cost}`...
}
```

压缩是一次额外的 LLM 调用，本身要花钱。Pi 把这笔钱明确显示出来，而不是当作"系统开销"隐藏——这直接引出下一节的权衡。

---

## 6. 上下文压缩的成本权衡

压缩在成本上是一把双刃剑，理解这个权衡才能理解 Pi 的压缩参数为什么是这个值。

**压缩省钱**：把 10 万 token 的历史换成 2 千 token 的摘要，此后每一轮都少发 9.8 万 token。

**压缩花钱**：(1) 生成摘要本身是一次 LLM 调用，要把待摘要的全部历史发过去；(2) 更隐蔽的代价是——**压缩改变了 prompt 前缀，之前累积的缓存全部作废**，压缩后的第一轮必须全价重新写入缓存。

所以压缩绝不能"急于触发"。默认配置体现了这种保守：

```typescript
export const DEFAULT_COMPACTION_SETTINGS: CompactionSettings = {
  enabled: true,
  reserveTokens: 16384,
  keepRecentTokens: 20000,
};

export function shouldCompact(contextTokens: number, contextWindow: number, settings: CompactionSettings): boolean {
  if (!settings.enabled) return false;
  return contextTokens > contextWindow - settings.reserveTokens;
}
```

- **只在真的快满了才压缩**：`contextTokens > contextWindow - 16384`。不做"上下文用到 50% 就主动瘦身"这种看似节省实则亏本的优化——在 20 万上下文窗口的模型上，从 10 万 token 压到 2 千 token 看起来省了很多，但如果这 10 万 token 本来每轮都以 `cacheRead` 单价（1/10）计费，压缩的收益远小于"作废缓存 + 摘要调用"的代价。**压缩的首要目的是"不撞上下文墙"，省钱只是副作用。**
- **保留最近 2 万 token 原文**：`keepRecentTokens: 20000` 保证压缩后最近的工作上下文仍是原始消息而非摘要——摘要有信息损失，如果压得太狠，模型会因为丢失细节而多做几轮无效试探，那些额外轮次的成本可能超过压缩省下的钱。
- **摘要输出有硬上限**：`maxTokens = Math.min(Math.floor(0.8 * reserveTokens), model.maxTokens)`——摘要最多用掉 `reserveTokens` 的 80%，防止"为了压缩历史反而生成一份超长摘要"。

多次压缩时摘要是**滚动更新**的（`previousSummary` 存在时用 `UPDATE_SUMMARIZATION_PROMPT` 合并，而非重新摘要全部历史）——这避免了"每次压缩都要把从会话开始的所有内容重新发给模型"的二次方级成本增长。

---

## 7. 渐进式披露：技能与工具清单

系统提示词是**每一轮都会发送**的内容（虽然有缓存，但缓存写入也要钱，且首轮全价）。因此系统提示词里的每个 token 都要衡量价值。

### 7.1 技能：只放名称与描述

Skills 遵循"渐进式披露"（progressive disclosure）：启动时只把每个技能的**名称 + 一行描述**注入系统提示词，完整的 `SKILL.md`（可能几千 token，还带脚本和参考文档）只在模型判断任务匹配后，主动用 `read` 工具加载。

一个装了 20 个技能的用户，如果全量注入，系统提示词可能膨胀到几万 token 且每轮都在；渐进式披露下常态开销是 20 行描述（几百 token），只有真的用到某个技能时才付它的完整代价。

`buildSystemPrompt()` 里还有一个配套的条件：如果所选工具集里不含 `read`，技能列表就不注入——因为没有 `read` 工具，模型没法加载技能完整内容，此时告诉它"有这些技能"纯属浪费 token。

### 7.2 工具清单：只列出有说明的工具

系统提示词的"可用工具"章节只列出**调用方在 `toolSnippets` 里提供了对应说明**的工具，而不是把 `selectedTools` 全部列出来。这既保证了提示词与真实可用工具的一致性，也避免了"列了一堆工具名但没有说明"的低价值 token。

---

## 8. 工具输出治理：截断而非全量入上下文

工具输出是 Agent 场景下上下文膨胀最快的来源——一次 `cat` 大文件、一次跑测试套件的输出，轻易就是几万 token，而且它会**永久留在对话历史里**，此后每一轮都要重新发送（哪怕命中缓存也要付 `cacheRead` 的钱）。

```typescript
export const DEFAULT_MAX_LINES = 2000;
export const DEFAULT_MAX_BYTES = 50 * 1024;      // 50KB
export const GREP_MAX_LINE_LENGTH = 500;          // Max chars per grep match line
```

三个限制各有针对性：

- **`DEFAULT_MAX_LINES = 2000` / `DEFAULT_MAX_BYTES = 50KB` 双限制**：行数限制应对"很多短行"（日志），字节限制应对"少数超长行"（压缩过的 JS、minified 数据）。只有一种限制都会被另一种情况绕过。
- **`GREP_MAX_LINE_LENGTH = 500`**：grep 匹配到一个 minified 文件时，单行可能几十万字符。截断到 500 字符仍然能让模型看到匹配上下文，但不会一行就吃掉整个预算。
- **完整输出落盘，路径回传**：截断后完整内容写到临时文件，工具结果里带上路径（如 RPC `bash` 响应的 `fullOutputPath`）。模型如果真的需要完整内容，可以针对性地用 `read` 读取特定区间——**按需付费，而不是预先全量付费**。

`truncateHead`/`truncateTail` 的区分也有讲究：读文件通常想看开头（`truncateHead`），跑命令通常想看结尾的结果和报错（`truncateTail`）。截断策略匹配语义，能在同样的 token 预算下保留更高价值的内容。

---

## 9. 思考等级：推理 Token 的显式预算控制

推理/thinking token 按 **output 单价** 计费（数倍于 input），而且一个高推理等级的复杂任务可能产生上万 thinking token。这是 output 侧最大的成本变量。

```typescript
export const DEFAULT_THINKING_LEVEL: ThinkingLevel = "medium";
export const THINKING_LEVEL_OPTIONS: readonly ThinkingLevel[] = [
  "off", "minimal", "low", "medium", "high", "xhigh", "max",
];
```

设计要点：

- **默认 `"medium"` 而非 `"high"`**：默认值是"日常任务的合理平衡点"，不是"效果最好"——把成本控制内置在默认值里，而不是要求用户主动去调低。
- **交互模式下 Shift+Tab 一键切换**：把思考等级放在最容易触达的快捷键上（而不是埋在 `/settings` 里），是因为它是一个**需要按任务频繁调整**的参数。简单任务降到 `low`/`off`、卡住的难题临时升到 `high`——降低调整摩擦，才会真的有人去调。
- **`pi-ai` 屏蔽 Provider 差异**：`ThinkingLevel` 统一映射到各家的实际参数（OpenAI `reasoning_effort`、Anthropic `thinkingBudgetTokens`、Google `thinking.budgetTokens`），换模型时思考等级设置继续有效，用户不需要重新学一套参数。
- **`thinkingTokenBudgetField`** 为支持的本地/兼容端点提供硬性 token 上限，并且"clamped so at least 1024 tokens remain for the answer"——限制推理预算时保证答案还有空间，避免"钱花在思考上但答案被截断"的最坏结果。

另一个侧面印证：`pi-evals` 的 Harness 把 `thinkingLevel` 固定为 `"off"`（`runPiCodingAgent()`）。评估要跑成百上千次，思考 token 的成本和随机性都是负担，关掉是正确选择——这也说明"思考等级应该按场景显式决策"是贯穿设计的理念。

---

## 10. 异构模型路由：便宜模型做便宜的事

前面的机制都在优化"单个会话内的 token 效率"。最后一层是**任务与模型的匹配**——不是所有工作都需要最强的模型。

### 10.1 子 Agent 用便宜模型

`subagent` 扩展的 Agent 定义里可以指定 `model:`，官方示例 `scout`（快速代码库侦察）指定的是 Haiku：

```markdown
---
name: scout
description: Fast codebase recon that returns compressed context for handoff to other agents
tools: read, grep, find, ls, bash
model: claude-haiku-4-5
---
```

"把 20 个文件读一遍、grep 几次、总结出关键位置"这类任务对模型智能要求不高但 token 消耗巨大，用便宜快的模型跑最划算；然后把它压缩后的产出交给强模型做真正需要判断力的工作。这是"用模型异构性换成本"最直接的应用。

更重要的是第二重节省：子 Agent 的探索过程（读的 20 个文件的完整内容）**完全不进入主会话上下文**——主 Agent 只收到一份结构化总结。这避免了"探索过程永久留在主会话历史里、此后每轮都要重新发送"的持续成本。这一点在《Pi 多 Agent 协作机制》一文有详细展开。

### 10.2 一次性调用不污染主会话缓存

`summarize.ts` 示例扩展展示了一个容易被忽略的细节——一次性的辅助调用应该显式关掉缓存并用独立的 session id：

```typescript
const response = await ctx.modelRegistry.complete(
  model,
  { messages: summaryMessages },
  {
    reasoningEffort: "high",
    cacheRetention: "none",
    sessionId: uuidv7(),
  },
);
```

两个决策各有原因：

- **`cacheRetention: "none"`**：这次调用的 prompt 是一次性的（当前会话文本的一次摘要），此后永远不会有第二次同前缀的请求。写缓存要付 `cacheWrite` 溢价，而这份缓存**永远不会被读取**——纯亏损。
- **`sessionId: uuidv7()`（全新 id 而非复用当前会话 id）**：如果复用主会话的 sessionId，这次结构完全不同的请求会与主会话共享 cache key / 亲和性路由，可能干扰主会话的缓存状态。用一个全新的 id 把它彻底隔离开。

这是一个很好的通用准则：**一次性调用应该显式关闭缓存并使用独立 session id**。写扩展时如果无脑复用主会话的默认选项，就会在无声中付出双份浪费（无用的 cacheWrite + 干扰主缓存）。

### 10.3 跨 Provider Handoff

`pi-ai` 支持同一段对话在不同 Provider 间切换（thinking 块自动转为带标签的文本，工具调用/结果原样保留）。这让"简单阶段用便宜模型、难点切到强模型"成为一次 `/model` 就能完成的操作。代价是切模型会作废缓存——而这个代价会被第 5.3 节的 "Cache miss after model switch" 提示明确告知用户，让这个权衡是知情的而非隐形的。

---

## 11. 机制总览与优先级

按对成本影响的量级从大到小排列：

```
【量级最大】Prompt Cache 命中率
  ├─ Anthropic 三断点放置（系统提示词 / 最后一个工具 / 最后一条 user 消息滚动断点）
  ├─ OpenAI prompt_cache_key（= sessionId，64 字符按码点截断）+ prompt_cache_retention
  ├─ 会话亲和性 header（三种格式自动探测，让请求落回持有缓存的节点）
  ├─ cacheRetention 三档（short 默认 / long 换取跨长间隔命中 / none 彻底关闭）
  └─ ★ Deferred Tools：中途新增的工具放进 transcript 而非前缀，避免击穿整个缓存前缀

【量级次之】上下文体量控制
  ├─ 自动压缩（阈值触发 contextWindow−16384，保留最近 20000 token 原文，摘要滚动更新）
  ├─ 工具输出截断（2000 行 / 50KB / grep 单行 500 字符，完整输出落盘按需读取）
  └─ 渐进式披露（技能只放名称+描述；无 read 工具则完全不注入技能列表）

【量级次之】Output 侧控制
  └─ 思考等级（默认 medium 而非 high，Shift+Tab 低摩擦切换，token 预算硬上限保留答案空间）

【结构性优化】任务与模型匹配
  ├─ 子 Agent 用便宜模型 + 独立上下文窗口（探索过程不进主会话）
  ├─ 一次性调用显式 cacheRetention: "none" + 独立 sessionId
  └─ 跨 Provider Handoff（按阶段难度换模型）

【使能层】成本可观测性
  ├─ 页脚常显 ↑input ↓output R(cacheRead) W(cacheWrite) CH(命中率)
  ├─ cache-stats.ts 量化缓存浪费（1024 噪声地板、按实际单价反推成本、压缩豁免/切模型不豁免、
  │   reportedCache 粘性标记区分"没命中"与"不支持缓存"）
  └─ 压缩/分支摘要自身成本明确显示，不作为隐藏开销
```

从这套设计里可以提炼三条贯穿性的原则：

1. **成本优化的重心在"前缀稳定性"，不在"少发内容"**。Deferred Tools 这个机制不减少任何一个 token 的发送量，它做的只是"把新增内容放到不破坏缓存前缀的位置"，收益却可能是几万 token 的全价重新计费。理解"prompt cache 是前缀匹配"这一个事实，比任何压缩技巧都更能降低成本。
2. **优化永不以正确性为代价**。Deferred Tools 在能力探测上采用默认关闭+白名单（Haiku 会拒绝 `tool_reference`、旧模型不支持 tool search），在 immediate 为空时无条件回退，`supportsLongCacheRetention` 在第三方代理上可关闭。每一个省钱机制都配了一条"这个 Provider 不支持时怎么办"的退路。
3. **成本必须可见，但不能变成噪声**。这是 `cache-stats.ts` 整个模块的设计张力：既要让用户知道钱花在哪（20k token / $0.1 双阈值触发提示、压缩自身成本明示、切模型代价明示），又要坚决避免误报（1024 噪声地板、压缩豁免、`reportedCache` 粘性标记、只陈述可观测归因）。一个天天喊"你浪费了 200 token"的提示，最终会被用户关掉，从此真正的重大浪费也一起看不见了——**告警的价值取决于它的信噪比，不取决于它的完备性**。
