# Pi Agent Telemetry 设计详解

> 系列第 10 篇（[系列索引](README.md)）。本文基于 `packages/telemetry/`、`packages/agent/src/harness/telemetry.ts`、`packages/ai/src/types.ts` 的源码整理，系统解释 Pi 的遥测（Telemetry）体系是**如何在不依赖任何具体后端**（OpenTelemetry / Sentry / Datadog / 自建日志）的前提下，保证跨包一致地观测 Agent 每一次 LLM 调用、每一次工具执行、每一次会话写入的。第 01 篇给了遥测的一段式概览，本篇是它的完整放大版：契约层怎么设计、Schema 层怎么切分、Pi 各包如何分工、生产环境如何接入后端、以及与评估体系怎么衔接。

## 目录

1. [为什么 Pi 要自己写一套遥测契约](#1-为什么-pi-要自己写一套遥测契约)
2. [核心概念：Span/Attribute/Event/Status/Context](#2-核心概念spanattributeeventstatuscontext)
3. [契约层：`TelemetryContext` 的显式回调设计](#3-契约层telemetrycontext-的显式回调设计)
4. [参考实现：`NOOP_TELEMETRY_CONTEXT` 与 `InMemoryTelemetryContext`](#4-参考实现noop_telemetry_context-与-inmemorytelemetrycontext)
5. [Schema 层：类型化的领域词汇](#5-schema-层类型化的领域词汇)
6. [Pi 专属 Schema：AI 请求 + Harness 操作 + 会话写入](#6-pi-专属-schemaai-请求--harness-操作--会话写入)
7. [Adapter 一致性测试套件：确保后端接入正确](#7-adapter-一致性测试套件确保后端接入正确)
8. [跨包分工与显式传递](#8-跨包分工与显式传递)
9. [三条工程规范：数据边界、Portability、稳定性](#9-三条工程规范数据边界portability稳定性)
10. [与评估体系的衔接](#10-与评估体系的衔接)
11. [接入实践：从零对接一个后端](#11-接入实践从零对接一个后端)
12. [设计原则总结](#12-设计原则总结)

---

## 1. 为什么 Pi 要自己写一套遥测契约

Pi 是**通用可嵌入的 Agent 内核**：可能被塞进 CLI、CI、SDK、远程会话服务，甚至运行在浏览器/Worker/Bun 里。这样的运行环境决定了它**不能**在核心包里直接依赖任何具体的遥测 SDK：

- 依赖 OpenTelemetry SDK：浏览器/Worker 场景装不下、Node 场景版本冲突严重。
- 依赖 Sentry SDK：绑定了商用供应商，且不同应用会有各自的 DSN 与采样策略。
- 什么都不做：无法量化"这次改动到底让 Agent 变快了还是变慢了"，也无法在生产环境定位问题。

于是 Pi 选择了第四条路：**自己定义一个只有几十行的厂商无关契约**，让核心包只与契约打交道；具体后端（OTel / Sentry / 自定义日志）由应用侧适配。这套契约就是 `@earendil-works/pi-telemetry`。

契约层只做四件事：

1. 定义 `TelemetryContext` / `TelemetrySpan` 这套**回调式**的 API 形状；
2. 提供一个 `NOOP_TELEMETRY_CONTEXT`——当应用不接入任何后端时的零成本占位；
3. 提供一个 `InMemoryTelemetryContext`——测试/评估/离线诊断用的参考实现；
4. 提供**类型化 Schema 工具**（`defineTelemetrySchema` + `startAiSpan` / `startHarnessSpan` 之类的辅助函数），让"往 span 上写什么属性"这件事变成 TypeScript 编译期约束，而不是运行期字符串拼装。

这四件事都不涉及任何后端，也不引入运行时依赖。

## 2. 核心概念：Span/Attribute/Event/Status/Context

Pi 的遥测语义完全对齐 OpenTelemetry 的经典模型，但 API 形态是自定义的（回调式，而非命令式），核心概念有 5 个：

| 概念 | 含义 | Pi 里的例子 |
|---|---|---|
| **Span** | 一次带时长的操作记录（开始 + 结束） | 一次到 Provider 的请求；一次 Harness `run()` |
| **Parent/Child** | 操作嵌套形成树 | `pi.harness.run` → `pi.harness.turn` → `pi.ai.request` |
| **Attribute** | 附加在 span 上的键值对，描述这次操作是什么/结果如何 | `pi.ai.provider="openai"`, `pi.ai.usage.input_tokens=1240` |
| **Event** | span 生命周期内的一个瞬时事件（无时长） | `retry.scheduled`, `cache.lookup` |
| **Status** | 最终结果：`ok` 或 `error` | 请求超时 → `{ status: "error", error: { name, message } }` |
| **Context** | 一个"父位置"的句柄，用于把新 span 挂到正确的父节点下 | 显式传递 `telemetryContext`，不用全局变量 |

它们组合起来，一次典型的 Agent 请求会形成这样一棵 Span 树：

```text
pi.harness.run              (root)  session.id=... operation.id=...
├─ pi.harness.turn                  turn.id=turn-1
│  ├─ pi.harness.step               step.attempt=1  step.kind=assistant
│  │  └─ pi.ai.request              provider=openai model=gpt-5.6 streaming=true
│  │     ├─ event retry.scheduled   (无时长)
│  │     └─ attrs usage.input_tokens=1240 usage.output_tokens=380 cost=0.0032
│  ├─ pi.harness.tool               tool.name=Bash call_id=...
│  └─ pi.harness.tool               tool.name=Read
└─ pi.harness.checkpoint            checkpoint.kind=normal
```

看得出来遥测树天然反映 Agent 的运行结构——这在后端里聚合"一次 turn 用了多少 token / 花了多少钱 / 慢在哪一步"变得非常直观。

## 3. 契约层：`TelemetryContext` 的显式回调设计

`packages/telemetry/src/index.ts` 的核心契约就 5 个类型：

```typescript
export type AttributeValue =
  | string | number | boolean
  | readonly string[] | readonly number[] | readonly boolean[];

export interface SpanAttributes { [name: string]: AttributeValue | undefined; }
export interface SpanOptions { name: string; attributes?: SpanAttributes; }
export type SpanStatus =
  | { status: "ok" }
  | { status: "error"; error?: { name: string; message: string } };

export interface TelemetryContext {
  startSpan<T>(options: SpanOptions, callback: (span: TelemetrySpan) => T | Promise<T>): Promise<T>;
}

export interface TelemetrySpan extends TelemetryContext {
  addEvent(name: string, attributes?: SpanAttributes): void;
  setAttributes(attributes: SpanAttributes): void;
  setStatus(status: SpanStatus): void;
}
```

这份契约里有三个关键设计决策值得单独拎出来讲：

### 3.1 回调式而非命令式

OTel SDK 常见的用法是 `const span = tracer.startSpan(); ...; span.end();`——命令式，容易忘记调 `end()` 造成"永远开着的 span"。Pi 反其道而行：**`startSpan(options, callback)` 是一个接受回调的方法**，callback 结束（Resolve / Reject / Throw）就等于 span 结束，`try/finally` 由 Adapter 内部保证。使用侧的形态永远是：

```typescript
await ctx.startSpan({ name: "pi.ai.request", attributes }, async (span) => {
  // ... 干活 ...
  span.setAttributes({ "pi.ai.usage.total_tokens": 1620 });
  return result;
});
```

这样"span 忘记 end"从代码结构上就不可能发生，同时错误状态（抛异常 / Promise reject）由 Adapter 自动映射到 `SpanStatus.error`。

### 3.2 `TelemetrySpan extends TelemetryContext`

Span 本身也是 Context——从 span 上再 `startSpan()` 得到的就是它的子 span。这套设计让"父子关系"完全靠**引用传递**表达，不需要全局的"当前活动 Span"变量。这一步是关键，直接决定了 Pi 可以做到下一节讲的 Portability。

### 3.3 显式传递，不用 `AsyncLocalStorage`

OTel 常用 `AsyncLocalStorage` 来隐式传播当前 span，但这有代价：

- 浏览器/Worker 没有 `AsyncLocalStorage`；
- 跨异步边界（如 `setTimeout` / 消息队列 / Worker Thread）会丢上下文；
- 单元测试里"当前活动 span"是共享状态，容易互相污染。

Pi 的规则是**永远显式把 `telemetryContext` 作为参数传递**：`pi-ai` 的 `ProviderRequestOptions` 里有 `telemetryContext?: TelemetryContext`，`pi-agent-core` 的每一层调用也都把这个上下文明确沿着调用链传下去。代价是签名里多一个参数，收益是——**在任何 JS 运行时都能跑，且测试永远可控**。

## 4. 参考实现：`NOOP_TELEMETRY_CONTEXT` 与 `InMemoryTelemetryContext`

契约本身不做任何"记录"工作。为了让 Pi 在没有接入后端时也能正常运行、并且让测试能验证遥测是否被正确发出，`pi-telemetry` 内置了两个参考实现。

### 4.1 `NOOP_TELEMETRY_CONTEXT`——零成本占位

```typescript
// 用途：应用侧不需要遥测时，把它作为 telemetryContext 传下去
import { NOOP_TELEMETRY_CONTEXT } from "@earendil-works/pi-telemetry";
```

它对所有 `startSpan()` / `addEvent()` / `setAttributes()` / `setStatus()` 都是空实现，callback 该跑还是跑，返回值/异常正常透传。这样核心包完全不需要"如果没配 telemetry 就跳过"这种分支，代码永远走同一条路径。

### 4.2 `InMemoryTelemetryContext`——测试/评估/诊断参考

`packages/telemetry/src/memory.ts` 是官方的参考适配器，它把每一个 span 完整记录进进程内的数组里，可以在事后通过 `getSpans()` 拿到只读快照：

```typescript
const ctx = new InMemoryTelemetryContext();
await runAgent({ telemetryContext: ctx });
for (const span of ctx.getSpans()) {
  console.log(span.name, span.attributes, span.status);
}
```

它同时是"Adapter 应该长成什么样"的活教材——外部实现（OTel Adapter / Sentry Adapter）应该在语义上与它一致。几个关键实现细节值得注意：

- **原子性**：`setAttributes` / `addEvent` 内部 `try/catch`，遇到 unreadable proxy 或抛错的对象**整个调用被丢弃**，绝不会记录半个属性——避免污染下游。
- **settle 后惰性**：span callback 结束后再调 `setAttributes` / `addEvent` 会被静默忽略（不会抛错），保护调用侧不用担心异步竞争。
- **status 合并**：如果 callback 抛异常且用户没显式 `setStatus`，自动设为 `error`；如果用户显式设过 status，就尊重用户的最后一次显式设置。
- **顺序快照**：`getSpans()` 返回**按开始顺序**的深拷贝，父子关系用 `parentId` 表达，`endSequence` 表示结束顺序（可用于验证父子嵌套时序）。

这些细节看似小，但正因为它们被固定在参考实现里、又通过后面第 7 节的一致性测试套件强制推向所有 Adapter，Pi 才能保证"换后端不换语义"。

## 5. Schema 层：类型化的领域词汇

如果只有 `startSpan(name, attributes)` 这一层，用起来仍然会是"字符串魔法"——某个包写 `provider=openai`，另一个包写 `providerId=openai`，聚合报表就废了。为此 Pi 引入了 **Schema 层**：

```typescript
export interface TelemetrySchemaDefinition {
  version: number;
  spans: Record<string, TelemetrySpanDefinition>;
}

export function defineTelemetrySchema<const T extends TelemetrySchemaDefinition>(schema: T): T {
  return schema;
}
```

`defineTelemetrySchema()` **只是一个类型化的 identity 函数**——它不做运行时校验，只做 TypeScript 编译期约束。每一个 `TelemetrySpanDefinition` 包含：

- `description`：这个 span 是干什么的（写给读代码的人看）；
- `parents`：允许的父 span——`any` / `root_or_external`（根或外部包持有的 span）/ `spans: [...]`（只能挂在指定 span 下）；
- `startAttributes`：开始时**必须**提供的属性（`required: true` 会成为 TypeScript 强制字段）；
- `endAttributes`：结束前可选补充的属性（永远是可选的）；
- `events`：允许发出的事件名 + 每个事件的属性 schema；
- `status`：默认状态与出错判定描述。

配合 `startAiSpan` / `startHarnessSpan` 之类的类型化 helper（下节详解），最终效果是：**你想在 `pi.ai.request` 上写一个未声明的属性名，TypeScript 直接报错；漏掉一个 `required: true` 的属性，也会报错。**

这一层是 Pi 让"跨包遥测保持一致词汇"的机制：Schema 由 owner 包声明并导出，所有消费方复用同一份类型定义。

## 6. Pi 专属 Schema：AI 请求 + Harness 操作 + 会话写入

`packages/agent/src/harness/telemetry.ts` 里定义了 Pi 自己的 Schema，最终导出两个常量：

```typescript
export const AI_TELEMETRY_SCHEMA = { version: 1, spans: { "pi.ai.request": {...} } } as const;
export const HARNESS_TELEMETRY_SCHEMA = { version: 1, spans: {
  "pi.harness.run": {...},
  "pi.harness.compaction": {...},
  "pi.harness.navigation": {...},
  "pi.harness.checkpoint": {...},
  "pi.harness.turn": {...},
  "pi.harness.step": {...},
  "pi.harness.tool": {...},
  "pi.harness.hook": {...},
  "pi.harness.sleep": {...},
  "pi.harness.event_handler": {...},
  "pi.session.write": {...},
} } as const;

export const AGENT_TELEMETRY_SCHEMAS = [AI_TELEMETRY_SCHEMA, HARNESS_TELEMETRY_SCHEMA] as const;
```

### 6.1 `pi.ai.request`——LLM 调用的原子

一次到 Provider 的**逻辑请求**（无论底层是流式还是一次性），描述"我们发起了什么、拿到了什么、烧了多少 token / 花了多少钱"：

- **Start attributes（发起时已知，必填）**：`pi.ai.operation`（`stream` / `fetch_deferred` / `cancel_deferred` / `generate_images`）、`pi.ai.provider`、`pi.ai.model`、`pi.ai.api`、`pi.ai.streaming`；`pi.ai.deferred` 可选。
- **End attributes（结束时补充，全部可选）**：`pi.ai.response.model` / `pi.ai.response.stop_reason`、`pi.ai.http.status_code`、四个 `pi.ai.usage.*_tokens` + `pi.ai.usage.cost`、`pi.ai.stream.chunk_count` / `time_to_first_chunk_ms`、`pi.ai.error.type`。
- **父级**：`any`——因为不是每次 AI 请求都发生在 harness 里（比如评估 Judge 里用另一个模型打分，此时它可能挂在评估自己的 span 下）。

这一个 span 就撑起了架构文里说的"评估报告的 Token/延迟/成本对比数据从遥测中提取"——把所有 `pi.ai.request` 的 `usage.total_tokens` / `cost` 加和，就是这一整次运行的账单。

### 6.2 `pi.harness.*`——持久化操作树

Harness 的每种操作各一个 span，用 `pi.operation.kind` 区分，parents 用 `root_or_external` 表明这是"顶层被外部接纳的一次调用"：

- `pi.harness.run`：一次被接纳的 `AgentSession.run()` 调用；
- `pi.harness.compaction`：一次手动压缩；
- `pi.harness.navigation`：一次会话导航；
- 三者共享 `operationStartAttributes`：`pi.session.id` / `pi.lane.name` / `pi.operation.id` / `pi.operation.recovery`——最后这个 boolean 明确记录"这是不是一次进程重启后的恢复执行"，对于诊断"为什么这次跑得比平时慢"至关重要。
- 结束时补充 `pi.operation.outcome`（`completed` / `aborted` / `failed` / `suspended`）+ 可选的 `pi.error.code` / `pi.error.type`。

在这三个操作 span 下，还嵌套着更细粒度的 span：

- `pi.harness.checkpoint`（父：`run`）：一次检查点，`checkpoint.kind` 区分正常/失败落盘/中断和解；
- `pi.harness.turn`（父：`run`）：一次助手响应 + 它的工具批次；
- `pi.harness.step`（父：`turn` / `checkpoint` / `compaction` / `navigation`）：**一次持久化重试尝试**——带有 `pi.step.attempt`（1-based）和 `pi.step.outcome`（`succeeded` / `retry` / `failed` / `aborted` / `deferred` / `overflow`），对应第 04 篇讲的自动重试机制；
- `pi.harness.tool`（父：`turn` / `run`）：一次原始 phase-2 工具执行，带 `pi.tool.name` / `pi.tool.call_id` / `pi.tool.replay`（`never` / `safe`——决定了这次工具能否被安全重放）；
- `pi.harness.hook`（父：`any`）：一次已注册 hook 的调用，`pi.hook.name` 从 11 个固定枚举里取；
- `pi.harness.sleep`（父：`step` / `run`）：一次重试延时，方便看"我们究竟花了多少时间在退避上"；
- `pi.harness.event_handler`（父：`any`）：一次被动事件监听器调用，`pi.event.type` 从 29 个固定枚举里取；
- `pi.session.write`（父：`any`）：一次已提交的会话变更，区分 `entry` / `record` / `lane` / `fact`，可暴露 `pi.session.seq`。

**这套 span 树几乎 1:1 对应 Harness 自己的状态机**——它不是"额外贴上去的观测层"，而是"运行时结构本身的可视化"。

### 6.3 类型化 helper：`startAiSpan` / `startHarnessSpan`

给 `pi.ai.request` 写一次调用长这样：

```typescript
await startAiSpan(
  telemetryContext,
  "pi.ai.request",
  {
    "pi.ai.operation": "stream",
    "pi.ai.provider": "openai",
    "pi.ai.model": "gpt-5.6-sol",
    "pi.ai.api": "chat",
    "pi.ai.streaming": true,
  },
  async (span) => {
    const result = await callProvider();
    span.setAttributes({
      "pi.ai.usage.input_tokens": result.usage.inputTokens,
      "pi.ai.usage.output_tokens": result.usage.outputTokens,
      "pi.ai.usage.total_tokens": result.usage.totalTokens,
      "pi.ai.usage.cost": result.usage.cost,
      "pi.ai.response.stop_reason": result.stopReason,
    });
    return result;
  },
);
```

关键约束都由 TypeScript 保证：漏了 `"pi.ai.streaming"` 编译不过；写了 `"pi.ai.usage.extra"` 编译不过（`ExactTelemetryAttributes` 会把未声明的键推成 `never`）；`stop_reason` 传了 `"weird_value"` 也编译不过（closed values 集合被枚举成字面量联合）。

## 7. Adapter 一致性测试套件：确保后端接入正确

Schema 定义了"名字和属性"，但一个 Adapter 是否正确实现契约，涉及一堆语义细节：错误状态怎么合并？settle 后再调用要不要静默？属性设置失败要不要原子回滚？父子关系的时序对不对？

`packages/telemetry/src/testing/conformance.ts` 就是官方的**跨 Adapter 一致性测试套件**，它是一组**测试运行器无关**的 case 工厂，任何实现了 `TelemetryContext` 的 Adapter 都可以用它验证自己：

```typescript
import { createTelemetryAdapterConformance } from "@earendil-works/pi-telemetry/testing";

for (const testCase of createTelemetryAdapterConformance(myAdapterFixtureFactory)) {
  it(`${testCase.group} / ${testCase.name}`, () => testCase.run());
}
```

套件覆盖了 5 个关键语义 group：

- **callback lifecycle**：`startSpan` 必须同步接纳一次并只接纳一次；同步/异步/`undefined`/unreadable 的 rejection 值必须**原样**透传（错误对象不能被吞或替换）。
- **status**：多次显式 `setStatus` 以最后一次为准；显式设为 `ok` 后再抛异常，最终 status 仍是 `ok`（尊重用户显式意图）；返回 `{ ok: false }` 但用户显式设了 error status，最终就是 error。
- **recording**：属性合并（后设覆盖先设）、事件按调用顺序追加；`undefined` 值被忽略；属性对象读取时抛错要**原子丢弃整次调用**（不能写入一半）。
- **parentage**：并发起两个 child span、先完成的 child 的 `endSequence` 必须先于后完成的；父 span 的 `endSequence` 必须晚于所有 child——这是"看时序图是否画得对"的强约束。
- **passivity**：unreadable 的 `SpanOptions` 或 attribute 传进来时，Adapter 必须自己吞掉错误而不是把它抛给业务代码。

**任何后端 Adapter 只要跑通这套测试，就能保证与 `InMemoryTelemetryContext` 语义一致**——Pi 换后端时业务代码不用改一行。这是"厂商无关"这句话真正的兑现方式，而不是空口承诺。

## 8. 跨包分工与显式传递

Pi 的仓库里有 6 个包会碰到遥测，职责边界严格切开：

| 包 | 拥有什么 | 只传递什么 |
|---|---|---|
| `pi-telemetry` | 契约类型、`NOOP`/`InMemory` 参考实现、Schema 工具、一致性测试套件 | —— |
| `pi-ai` | 不定义任何 span schema | 通过 `ProviderRequestOptions.telemetryContext` 接收上下文并传给 Provider |
| `pi-agent-core` | 拥有并导出 `AI_TELEMETRY_SCHEMA` / `HARNESS_TELEMETRY_SCHEMA` / `AGENT_TELEMETRY_SCHEMAS` 以及 `startAiSpan` / `startHarnessSpan` | 在运行时把 telemetry 传给 pi-ai 和 tools |
| `pi-coding-agent` | 不定义 span schema（除了应用侧独立的 `enableInstallTelemetry` 版本 ping） | 创建 `TelemetryContext` 传给 `AgentSession` |
| `pi-server` / `pi-client` | 协议层不承载遥测语义 | Listener 层可以自行给到 server 端会话 |
| 应用（如 evals / CI） | 组装 Adapter（真实后端 or InMemory）并注入 | —— |

`pi-ai/src/types.ts` 里的 `ProviderRequestOptions` 就是这条传递链上的第一个接触点：

```typescript
export interface ProviderRequestOptions {
  // ...
  telemetryContext?: TelemetryContext;
}
```

`pi-ai` 自己**不定义** `pi.ai.request` schema，只把 `telemetryContext` 从入参"透传"下去。真正调 `startAiSpan("pi.ai.request", ...)` 的是 `pi-agent-core`——因为 schema 属于 agent 领域词汇。这个分工的价值是：`pi-ai` 完全独立可用（可以作为纯 LLM 库使用），不背上 Agent 概念；`pi-agent-core` 想换 span 名/属性也不影响 `pi-ai`。

这就是 README 里那句话——"包所有权是**故意**切开的"——的具体含义。

## 9. 三条工程规范：数据边界、Portability、稳定性

`packages/telemetry/README.md` 里明确写下了三条给使用者的规范，值得单独强调，因为它们决定了遥测能否"长期可靠"。

### 9.1 数据边界：遥测是过程诊断，不是应用状态

> Telemetry is process-local diagnostics, not durable application state.

不能把 `TelemetryContext` / `TelemetrySpan` / 后端原生 trace 对象序列化后**存进会话记录、消息、快照、延时句柄里**——它们是进程本地的诊断句柄，跨进程无意义、跨重启会失效。Pi 的会话 JSONL 里保存的是 `AgentMessage` 结构，不含任何 span 引用；持久化的 usage 统计从 span 属性中读出成为普通数字后落盘，而不是保存 span 本身。

### 9.2 属性数据类型受限，避免"敏感数据一泄了之"

`AttributeValue` 只允许 `string / number / boolean` 及其数组——**没有对象、没有 unknown、没有 JSON**。这不是懒，而是刻意的边界：

- 后端一致性简单（每个后端都支持这些基本类型）；
- 强迫写者做取舍——想把整个模型响应塞进 span 里？做不到，只能记 `response_length` 或 `stop_reason`；
- 减少敏感数据泄漏：`prompt` / `completion` / `tool_arguments` / `file_contents` / `credentials` **默认都不进遥测**，除非某个 span schema 明确声明允许（并配套敏感度标记 `sensitive: true`）。

这与"日志无脑打 request/response body"的习惯形成鲜明对比——后者一旦上生产就成为无解的 PII 泄漏源。

### 9.3 Portability：不用 `AsyncLocalStorage`，也不用运行时特化 API

契约本身完全用同步/异步回调实现，没有绑定 Node.js 特有的 API。这直接决定了：

- 浏览器可用（把 Adapter 换成一个 fetch 到自建 collector 的实现即可）；
- Bun / Deno 可用（无需 polyfill）；
- Worker Thread / Service Worker 可用（每个 Worker 各自一套 Context，天然不互扰）。

后端 Adapter 内部当然可以用 `AsyncLocalStorage` 或平台特定 API 来做后端侧的关联——但那属于 Adapter 的实现自由，不通过契约向上传染。

## 10. 与评估体系的衔接

这套遥测不是孤立存在的，它是评估体系（第 09 篇）**唯一**的数据来源。`pi-evals` 的报告里"Token / Latency / Cost 对比"三行，来源就是每一次运行结束后，把该次运行内所有 `pi.ai.request` span 的 `pi.ai.usage.total_tokens` / `pi.ai.usage.cost` 加和：

```text
pi-evals 运行流程
  └─ createPiCodingAgentHarness()
       └─ 每次 run() 都传入一个 InMemoryTelemetryContext
       └─ AgentSession 内部：
            ├─ pi.harness.run                     ← 报告里 "Latency"
            │  └─ pi.harness.step
            │     └─ pi.ai.request                ← 报告里 "Tokens" / "Cost"
            └─ pi.session.write                   ← 落盘会话 JSONL
       └─ 收尾：ctx.getSpans() → 提取 usage/timings 上报到 EvalHarnessReporter
```

这带来两个直接好处：

1. **评估中观测到的成本 == 生产中观测到的成本**——因为它们是同一套 span 定义，评估和生产环境唯一的差别就是 Adapter（一个用 InMemory，一个用 OTel/Sentry），语义完全一致。
2. **评估报告可以细分到 span 级别**——比如"候选组比基线组多出的耗时全在 `pi.harness.sleep` 里"就能直接告诉你"是被重试退避拖慢的，不是模型本身慢"。

这也是第 01 篇末尾"遥测天然衔接评估"的具体机制。

## 11. 接入实践：从零对接一个后端

假设我们要把 Pi 接入一个用 OpenTelemetry SDK 的自建 collector，只需要写一个几十行的 Adapter：

```typescript
import type { TelemetryContext, TelemetrySpan, SpanOptions } from "@earendil-works/pi-telemetry";
import type { Tracer, Span as OTelSpan, SpanStatusCode } from "@opentelemetry/api";

export function createOTelTelemetryContext(tracer: Tracer, parent?: OTelSpan): TelemetryContext {
  return {
    async startSpan(options, callback) {
      return await tracer.startActiveSpan(options.name, { attributes: options.attributes }, parent as any, async (otelSpan) => {
        const piSpan: TelemetrySpan = {
          startSpan: (childOpts, childCb) =>
            createOTelTelemetryContext(tracer, otelSpan).startSpan(childOpts, childCb),
          addEvent(name, attrs) { otelSpan.addEvent(name, attrs as any); },
          setAttributes(attrs) { otelSpan.setAttributes(attrs as any); },
          setStatus(status) {
            otelSpan.setStatus({
              code: status.status === "ok" ? 1 : 2 satisfies SpanStatusCode,
              message: status.status === "error" ? status.error?.message : undefined,
            });
          },
        };
        try {
          const result = await callback(piSpan);
          otelSpan.end();
          return result;
        } catch (err) {
          otelSpan.recordException(err as Error);
          otelSpan.setStatus({ code: 2, message: (err as Error).message });
          otelSpan.end();
          throw err;
        }
      });
    },
  };
}
```

然后在应用启动处把它注入 Agent：

```typescript
const tracer = provider.getTracer("my-app");
const telemetryContext = createOTelTelemetryContext(tracer);

const session = await createAgentSession({ /* ... */, telemetryContext });
```

**验证 Adapter 正确性只需要一步**——用第 7 节的一致性测试套件跑一遍：

```typescript
import { createTelemetryAdapterConformance } from "@earendil-works/pi-telemetry/testing";

for (const testCase of createTelemetryAdapterConformance(makeFixtureFactory)) {
  it(`${testCase.group}/${testCase.name}`, () => testCase.run());
}
```

跑通就说明这套 OTel Adapter 与 `InMemoryTelemetryContext` 语义完全一致，可以放心上生产。

对于**只需要本地诊断**（不接后端）的场景，一行代码就够了：

```typescript
import { InMemoryTelemetryContext } from "@earendil-works/pi-telemetry";
const telemetryContext = new InMemoryTelemetryContext();
// ... 跑完之后
for (const span of telemetryContext.getSpans()) console.log(span);
```

## 12. 设计原则总结

纵览 Pi 的整套遥测设计，可以提炼出五条贯穿始终的原则：

1. **契约优先，后端可插拔**：核心包只依赖几十行的 `TelemetryContext` 契约，OTel / Sentry / 自建日志都是应用侧 Adapter 的实现选择，核心永不改动。
2. **回调式 + 显式传递**：`startSpan(options, callback)` 从代码结构上保证 span 一定会 end；`telemetryContext` 沿调用链显式传，不依赖 `AsyncLocalStorage`，跨运行时/跨异步边界都可靠。
3. **Schema 与 span 名字属于领域 owner**：`pi-agent-core` 拥有 `pi.ai.*` / `pi.harness.*` / `pi.session.*` 词汇，别的包只传递不定义；Schema 是 TypeScript 编译期约束，不需要运行时校验就能防止属性走样。
4. **参考实现 + 一致性测试**：`InMemoryTelemetryContext` 是活的语义文档，`createTelemetryAdapterConformance` 是所有 Adapter 必须通过的语义合同——"厂商无关"由此从口号变成可执行的验收标准。
5. **过程诊断而非应用状态**：Span/Context 不进快照、不进消息、不跨进程；属性值类型受限（只允许标量与其数组），从数据结构层面杜绝敏感 payload 泄漏。

理解以上五点，就能明白为什么第 09 篇的评估报告能拿到"和生产一致"的成本数据、为什么第 04 篇讲的重试和 sleep 能被完整还原成时序图、为什么 Pi 在浏览器/Worker 里嵌入也不会因为遥测 SDK 装不下而卡壳——**这套体系的所有能力，都是这五条原则的自然推论。**
