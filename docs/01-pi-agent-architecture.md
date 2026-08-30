# Pi Agent 架构与关键功能全解析

> 系列第 01 篇（[系列索引](README.md)）。本文基于 `pi-mono` 仓库源码整理，系统介绍 Pi Agent 的整体架构、核心运行机制、关键功能模块，以及配套的 Agent 评估体系。内容按照"从整体到局部、从核心到扩展"的顺序展开，便于循序渐进地建立完整认知。作为全系列的地图，建议先读本篇。

## 目录

1. [项目概览](#1-项目概览)
2. [整体架构：分层的 Monorepo](#2-整体架构分层的-monorepo)
3. [核心运行引擎：Agent Loop](#3-核心运行引擎agent-loop)
4. [持久化执行模型：Agent Harness](#4-持久化执行模型agent-harness)
5. [多模型抽象层：pi-ai](#5-多模型抽象层pi-ai)
6. [工具系统](#6-工具系统)
7. [可扩展性：Extensions / Skills / Prompt Templates](#7-可扩展性extensions--skills--prompt-templates)
8. [会话管理与上下文压缩](#8-会话管理与上下文压缩)
9. [多种运行模式与集成方式](#9-多种运行模式与集成方式)
10. [安全与信任模型](#10-安全与信任模型)
11. [可观测性：遥测体系](#11-可观测性遥测体系)
12. [Agent 评估体系（Evals）](#12-agent-评估体系evals)
13. [架构设计原则总结](#13-架构设计原则总结)

---

## 1. 项目概览

Pi 是一个**极简的终端编码 Agent 框架（Coding Agent Harness）**，核心设计目标是"内核尽量小，扩展尽量强"：核心只负责 Agent 循环、工具调用、状态管理，其余能力（编辑器 UI、技能、扩展、多模型支持等）都通过分层的独立包组合而成。

官方定位的三大核心包：

| 包 | 定位 |
|---|---|
| `@earendil-works/pi-ai` | 统一多 Provider 的 LLM API（OpenAI、Anthropic、Google 等 30+ 家） |
| `@earendil-works/pi-agent-core` | Agent 运行时：工具调用循环 + 状态管理 |
| `@earendil-works/pi-coding-agent` | 面向编码场景的交互式 Agent CLI（工具、扩展、会话、UI 的落地实现） |

在这三者之上，还有面向远程化、可观测性、评估的配套包，共同构成完整体系。

---

## 2. 整体架构：分层的 Monorepo

Pi 采用 monorepo 组织，各包职责边界清晰，依赖关系是**严格单向**的（下层不依赖上层）：

```
┌─────────────────────────────────────────────────────────────────┐
│  packages/coding-agent   交互式编码 Agent（CLI / TUI / SDK / RPC）│
│  ├─ core/        AgentSession、工具、压缩、扩展运行时              │
│  ├─ modes/       interactive / print / rpc / json-event 四种模式  │
│  └─ examples/    扩展、SDK、技能等示例                            │
├─────────────────────────────────────────────────────────────────┤
│  packages/agent         Agent 运行时核心（引擎层）                 │
│  ├─ agent.ts / agent-loop.ts   事件驱动的对话+工具循环             │
│  └─ harness/     持久化、可恢复的 Agent 执行状态机                 │
├─────────────────────────────────────────────────────────────────┤
│  packages/ai            统一多 Provider LLM API（stream/complete）│
│  packages/protocol       实验性二进制协议（CBOR + 帧）             │
│  packages/client/server   远程会话的客户端/服务端（基于 protocol） │
│  packages/session-backends 会话持久化后端（如 SQLite）             │
├─────────────────────────────────────────────────────────────────┤
│  packages/telemetry      厂商无关的遥测契约（Span/Attribute/Event）│
│  packages/tui             终端 UI 组件库（差量渲染）                │
│  packages/evals           行为级 Agent 评估框架                    │
└─────────────────────────────────────────────────────────────────┘
```

**依赖流向**：`pi-tui`、`pi-telemetry`、`pi-ai`、`pi-protocol` 是最底层的基础设施包，互不依赖；`pi-agent-core` 依赖 `pi-ai` 和 `pi-telemetry`；`pi-coding-agent` 依赖 `pi-agent-core`、`pi-ai`、`pi-tui`；`pi-client`/`pi-server` 依赖 `pi-protocol`；`pi-evals` 依赖 `pi-coding-agent`。

这种分层带来两个直接好处：

- **可独立使用**：可以只用 `pi-ai` 做多 Provider LLM 调用，不需要 Agent 循环；也可以只用 `pi-agent-core` 构建自己的 Agent UI，不需要编码专用工具。
- **可替换实现**：会话持久化（`session-backends`）、遥测后端（`telemetry` 适配器）、传输层（`client`/`server` 的 `ByteTransport`）都是可插拔接口。

---

## 3. 核心运行引擎：Agent Loop

`pi-agent-core` 是整个体系的心脏，核心抽象是 **事件驱动的对话循环**。

### 3.1 消息模型：AgentMessage vs LLM Message

Agent 内部使用 `AgentMessage`（可通过声明合并扩展自定义消息类型，例如 UI 通知），而 LLM 只理解 `user` / `assistant` / `toolResult` 三种角色。两者通过一条转换管线桥接：

```
AgentMessage[] → transformContext() → AgentMessage[] → convertToLlm() → Message[] → LLM
                   (可选：裁剪/压缩)                    (必需：过滤/转换)
```

- `transformContext`：在发起 LLM 调用前对历史消息做裁剪、注入外部上下文（常用于压缩）。
- `convertToLlm`：过滤掉纯 UI 消息，把自定义消息类型转换为标准 LLM 消息。

### 3.2 事件驱动的循环

调用 `agent.prompt("...")` 会触发一串结构化事件，这是理解 Agent 行为的关键心智模型：

**无工具调用的简单场景：**

```
prompt("Hello")
├─ agent_start
├─ turn_start
├─ message_start / message_end   { userMessage }
├─ message_start                 { assistantMessage 开始响应 }
├─ message_update ...            { 流式增量 }
├─ message_end                   { assistantMessage 完整 }
├─ turn_end        { message, toolResults: [] }
└─ agent_end        { messages: [...] }
```

**包含工具调用的场景（循环会持续到没有更多工具调用）：**

```
prompt("Read config.json")
├─ agent_start / turn_start
├─ message_start/end  { userMessage }
├─ message_start      { assistantMessage 携带 toolCall }
├─ message_end        { assistantMessage }
├─ tool_execution_start / update / end   { toolCallId, toolName, result }
├─ message_start/end  { toolResultMessage }
├─ turn_end           { toolResults: [toolResult] }
│
├─ turn_start          （新一轮）
├─ message_start/end   { LLM 基于工具结果的响应 }
├─ turn_end
└─ agent_end
```

完整事件类型：

| 事件 | 说明 |
|---|---|
| `agent_start` / `agent_end` | 一次 run 的起止 |
| `turn_start` / `turn_end` | 一轮"LLM 调用 + 工具执行"的起止 |
| `message_start` / `message_update` / `message_end` | 消息（user/assistant/toolResult）的生命周期，`message_update` 仅 assistant 消息触发，携带流式 delta |
| `tool_execution_start` / `update` / `end` | 单个工具调用的生命周期 |

### 3.3 工具执行模型

工具执行支持两种模式：

- **parallel（默认）**：所有工具调用的前置校验（preflight）串行进行，随后允许并发工具并发执行；`tool_execution_end` 按完成顺序触发，但持久化的 `toolResult` 消息仍按 assistant 消息中声明的原始顺序落盘。
- **sequential**：逐个执行，行为等价于早期版本。

执行模式可以全局配置，也可以在单个工具上通过 `executionMode` 覆盖；只要一批工具调用中有任意一个被标记为 `sequential`，整批都会退化为顺序执行。

两个关键钩子贯穿工具调用生命周期：

```
tool_execution_start → beforeToolCall（可阻断执行）→ 执行 → afterToolCall（可改写结果）→ tool_execution_end
```

`beforeToolCall`、`afterToolCall`、以及工具的 `execute()` 本身都可以返回 `terminate: true`，作为"建议 Agent 在本批工具结束后停止自动追问 LLM"的信号——但只有当**同一批**中所有已完成的工具结果都设置了该标记时，循环才会真正提前结束。

### 3.4 消息队列：Steering 与 Follow-up

用户可以在 Agent 运行期间插话：

- **Steering（插话/打断）**：在当前工具批次执行完后立即注入，下一轮 LLM 会看到它。
- **Follow-up（排队追加）**：只有当没有更多工具调用、也没有 Steering 消息时才会被注入。

两者都支持 `one-at-a-time`（默认，每次只消费一条）或 `all`（一次性消费全部排队消息）模式。

### 3.5 低层 API

除了面向对象的 `Agent` 类，还提供函数式的底层 API `agentLoop()` / `agentLoopContinue()`，直接返回异步事件流，适合需要完全自定义控制流的高级场景。但底层流是"观察性"的——不会等待事件订阅者的异步处理完成再继续，如果需要"消息处理必须作为屏障"的语义，应使用 `Agent` 类。

---

## 4. 持久化执行模型：Agent Harness

在基础 `Agent`/`agentLoop` 之上，`pi-agent-core/harness` 提供了一层更强的抽象——**AgentHarness**，用于支撑生产级、可持久化、可从故障中恢复的 Agent 执行。

关键概念：

- **Lane（执行道）**：一个 Harness 可以有多条独立的执行 Lane，每条 Lane 维护自己的会话叶子节点（leaf）、操作状态和消息队列。
- **Durable Operation（持久化操作）**：`run`（对话轮次）、`compaction`（压缩）、`navigation`（分支导航）都被建模为可挂起、可恢复的操作，操作状态和结果写入 `Session`（会话树），进程重启后可以 `resume()` 继续。
- **SessionTree**：会话本身是一棵树（entry 之间通过 `id`/`parentId` 关联），支持分支、跳转、摘要合并。
- **Result 型错误处理**：Harness 的公开方法几乎全部返回 `ResultValue<Outcome, RejectedReason>`（而非抛异常），把"业务性拒绝"（如 `LaneBusy`、`NothingToCompact`）与真正的运行时故障（`HarnessFault`）区分开。

Harness 对外暴露的核心操作面（`AgentLane` 接口）大致对应：

```typescript
lane.prompt(...)          // 发起一轮对话
lane.compact(...)         // 触发压缩
lane.navigateTree(...)    // 分支导航（可选摘要）
lane.resume()             // 从崩溃/挂起点恢复
lane.abort()              // 中止当前操作
lane.steer(...) / lane.followUp(...) / lane.nextRun(...)  // 消息队列
lane.watch()              // 订阅 LaneSnapshot 的实时快照
```

这一层设计的核心价值：把"对话循环"升级为"可审计、可恢复、可并发隔离（多 Lane）的状态机"，是构建多会话服务端（`pi-server`）、长时间运行的自动化任务的基础。

---

## 5. 多模型抽象层：pi-ai

`pi-ai` 是 LLM 调用的统一入口，设计上把 **Provider（运行时单元）** 和 **API 实现（协议实现）** 分离：

- **Provider**：拥有模型目录、鉴权（API Key / OAuth）、流式行为，例如 `anthropicProvider()`、`openaiProvider()`。
- **API 实现**：真正的协议编解码，多个 Provider 可以共享同一套 API 实现（例如 xAI、Groq、Cerebras 都用 `openai-completions`）。

```
Models 集合
 ├─ Provider: anthropic  → api: anthropic-messages
 ├─ Provider: openai     → api: openai-responses
 ├─ Provider: xai        → api: openai-completions（与多家共享）
 └─ Provider: llamacpp   → api: openai-completions（动态模型列表）
```

关键能力：

- **统一事件流**：无论底层 Provider 是谁，`stream()` 都会产出统一的 `text_delta` / `thinking_delta` / `toolcall_delta` / `done` / `error` 事件序列。
- **鉴权自动解析**：`models.getAuth()` 按"显式传入 → Provider 已存储凭证 → 环境变量/OAuth"的顺序解析，OAuth 过期自动刷新且加锁防止并发重复刷新。
- **跨 Provider 无缝切换（Handoff）**：同一段对话可以从 Claude 切到 GPT 再切到 Gemini，thinking 块会自动转换为带 `<thinking>` 标签的文本，工具调用/结果原样保留。
- **思考/推理（Thinking）统一接口**：`reasoning: 'minimal' | 'low' | 'medium' | 'high' | 'xhigh' | 'max'`，屏蔽了各家 Provider 的字段差异（OpenAI 的 `reasoning_effort`、Anthropic 的 `thinkingBudgetTokens`、Google 的 `thinking.budgetTokens` 等）。
- **约束采样（Constrained Sampling）**：工具可声明 `strict` JSON Schema 或语法约束（Lark/Regex grammar），在支持的模型上启用严格的结构化输出。
- **上下文可序列化**：`Context` 是纯 JSON，可直接 `JSON.stringify` 持久化、跨进程传递。
- **Faux Provider**：内置的脚本化假 Provider，用于测试和演示，无需真实 API Key 即可模拟完整的流式/工具调用行为。

支持的 Provider 数量超过 30 家（OpenAI、Anthropic、Google/Vertex、Bedrock、Mistral、Groq、Cerebras、OpenRouter、各类 OpenAI 兼容端点等），且按"按需引入"设计——只导入用到的 Provider 子路径即可获得最小打包体积，SDK 依赖通过懒加载分块。

---

## 6. 工具系统

`pi-coding-agent/core/tools` 提供了编码场景的内置工具集：

| 工具 | 作用 |
|---|---|
| `read` | 读取文件内容 |
| `write` | 写入/创建文件 |
| `edit` | 精确字符串替换编辑 |
| `bash` / `powershell` | 执行 shell 命令 |
| `grep` | 内容搜索（基于 ripgrep 语义） |
| `find` | 按文件名模式查找 |
| `ls` | 目录列举 |

组合方式分层：

```
createTool(name, cwd, options)          // 单个工具
createCodingTools(cwd)                  // read+bash+edit+write（可写场景）
createReadOnlyTools(cwd)                // read+grep+find+ls（只读场景）
createAllTools(cwd)                     // 全部工具的 Record
```

在 `pi-agent-core` 层，工具通过统一的 `AgentTool` 接口定义（`name` / `parameters`（TypeBox Schema）/ `execute()`），错误处理约定是**抛异常**而非返回错误内容——抛出的错误会被 Agent 捕获并转换为 `isError: true` 的工具结果反馈给 LLM，保证"成功返回内容，失败抛异常"的一致契约。

工具还支持流式进度反馈（`onUpdate` 回调）、执行模式覆盖（`executionMode: "sequential" | "parallel"`），以及通过 `beforeToolCall`/`afterToolCall` 钩子实现权限门（如确认 `rm -rf`）、路径保护（阻止写 `.env`）等安全策略——这也是扩展系统最典型的应用场景之一。

---

## 7. 可扩展性：Extensions / Skills / Prompt Templates

Pi 坚持"核心极简，扩展负责一切"的哲学，提供三层不同粒度的扩展机制：

### 7.1 Extensions（扩展）

TypeScript 模块，通过 `pi.on(eventName, handler)` 订阅生命周期事件（资源加载、会话、Agent、模型、工具事件），并可以：

- `pi.registerTool()` 注册自定义工具
- `pi.registerCommand()` 注册 `/mycommand` 斜杠命令
- `pi.registerShortcut()` / `pi.registerFlag()` 注册快捷键/CLI 参数
- `ctx.ui.custom()` 渲染完整的自定义 TUI 组件

扩展从 `~/.pi/agent/extensions/`（全局）或 `.pi/extensions/`（项目级，需项目信任后才加载）自动发现，支持 `/reload` 热重载。典型应用：权限确认、Git checkpoint、自定义压缩策略、外部集成（webhook/CI）。

### 7.2 Skills（技能）

遵循 [Agent Skills 标准](https://agentskills.io/specification) 的"按需加载"能力包：一个目录 + `SKILL.md`（frontmatter + 指令），可附带脚本、参考文档、模板资源。启动时只把技能的**名称+描述**放入系统提示词（渐进式披露），Agent 判断任务匹配后再用 `read` 加载完整内容，避免上下文膨胀。

### 7.3 Prompt Templates（提示词模板）与 Pi Packages

Prompt Templates 是可复用的、通过斜杠命令展开的提示词片段。Pi Packages 则是打包分发以上三种资源（扩展/技能/模板/主题）的机制，可通过 npm、git、本地路径三种来源安装，支持项目/全局级过滤与去重。

三者共同构成"核心不变、能力靠组合"的插件体系。

---

## 8. 会话管理与上下文压缩

### 8.1 会话即树

每个会话是一棵以 JSONL（或 SQLite）持久化的**树结构**：每个 entry 有 `id`/`parentId`，当前工作位置是活跃叶子节点。这使得 `/tree` 分支导航、`/fork`（从历史消息派生新会话）、`/clone`（复制当前分支）成为一等能力，而不需要额外的版本控制系统。

### 8.2 自动压缩（Compaction）

当上下文 Token 超过 `contextWindow - reserveTokens` 阈值时自动触发（也可用 `/compact` 手动触发）：

```
压缩前：
  entry:  0    1    2    3     4    5    6     7     8    9
        ┌────┬────┬────┬─────┬────┬────┬─────┬─────┬────┬────┐
        │hdr │usr │ass │tool │usr │ass │tool │tool │ass │tool│
        └────┴────┴────┴─────┴────┴────┴─────┴─────┴────┴────┘
              └─────┬──────┘ └───────────┬────────────┘
           待摘要消息(0-3)         保留消息(4-9)
                                     ↑
                          firstKeptEntryId = 4

压缩后（追加一条 compaction entry）：
  LLM 实际看到的上下文 = system + 摘要 + 保留消息(from firstKeptEntryId)
```

步骤：定位裁剪点（从最新消息倒序累计 Token，直到达到 `keepRecentTokens`）→ 抽取待摘要区间 → 调用 LLM 生成结构化摘要（Goal/Progress/Key Decisions/Next Steps/读写文件列表）→ 追加 `CompactionEntry`。多次压缩时，摘要区间会从上一次压缩的 `firstKeptEntryId` 开始，保证信息不丢失地滚动累积。

对于单轮就超预算的"超大轮次"（Split Turn），会分别生成"历史摘要"和"轮次前缀摘要"再合并。

### 8.3 分支摘要（Branch Summarization）

当用户用 `/tree` 切换到不同分支时，Pi 会提议把即将放弃的分支内容摘要后注入新分支，避免上下文丢失，机制与压缩共享同一套结构化摘要格式与文件追踪逻辑。

### 8.4 扩展点

`session_before_compact`、`session_compact_failed`、`session_before_tree` 三个事件允许扩展完全接管摘要生成逻辑（例如换用更便宜的模型做摘要），或阻止压缩/导航发生。

---

## 9. 多种运行模式与集成方式

同一个 `AgentSession` 核心可以运行在多种壳层之下，这是"内核与界面分离"的直接体现：

| 模式 | 场景 |
|---|---|
| **Interactive（TUI）** | 默认交互模式，基于 `pi-tui` 的差量渲染终端界面 |
| **Print（`-p`）** | 一次性非交互执行，适合脚本调用 |
| **RPC（`--mode rpc`）** | JSONL over stdin/stdout 的进程内协议，适合嵌入 IDE/其他应用 |
| **JSON Event Stream（`--mode json`）** | 结构化事件流打印模式 |
| **SDK** | 直接在 Node.js 中 `import { createAgentSession } from "@earendil-works/pi-coding-agent"` 编程式调用 |

面向**远程化**场景，还有独立的三件套：

- `pi-protocol`：运行时无关的二进制协议（4 字节长度前缀 + CBOR 编码消息），定义 `SessionMetadata`/`SessionSnapshot` 等权威状态结构。
- `pi-server`：`PiServer` 组合多个 `PiServerListener`（如 Unix Socket），每个 Listener 负责自己的传输层鉴权。
- `pi-client`：`PiClient` 通过可插拔 `ByteTransport` 连接远程会话，支持独占/共享两种会话租约（Lease）模式。

这套协议栈让"本地 CLI Agent"和"多用户远程 Agent 服务"共享同一套会话语义。

---

## 10. 安全与信任模型

Pi **不内置沙箱**——工具以启动 Pi 进程的用户权限直接读写文件、执行 shell 命令，扩展代码同样拥有完整系统权限。这是有意为之的设计选择：一个"伪安全"的进程内沙箱容易被误认为真实边界，而实际隔离必须来自操作系统或虚拟化层。

**项目信任（Project Trust）** 是唯一的内建防线，但它只是"资源加载守卫"而非沙箱：

- 只有当目录下存在 `.pi/settings.json`、`.pi/extensions|skills|prompts|themes`、`.pi/SYSTEM.md` 等"需要信任的资源"时才触发信任判断。
- 默认策略 `defaultProjectTrust: "ask"`，交互式询问；决策按目录持久化在 `~/.pi/agent/trust.json`。
- 信任只决定"是否加载项目级配置/扩展"，**不会**限制模型在信任后能让工具做什么，也无法防御来自仓库内容的提示注入（prompt injection）。

对于不信任的仓库、需要无人值守运行的自动化任务，官方建议使用容器/微虚拟机（Docker、Gondolin、OpenShell 等）做真正的系统级隔离，仅挂载任务所需的最小文件与凭证。

---

## 11. 可观测性：遥测体系

`pi-telemetry` 提供一套**厂商无关**的遥测契约，核心抽象是 Span（一次带时长的操作记录）及其属性/事件/状态，`pi-agent-core` 在此之上定义了 pi 专属的类型化 Schema：

```typescript
import { AGENT_TELEMETRY_SCHEMAS, startAiSpan, startHarnessSpan } from "@earendil-works/pi-agent-core";
```

两条核心 Span：

- **`pi.ai.request`**：一次到 Provider 的逻辑请求，记录 `provider`/`model`/`api`，完成时补充 `usage.*`（输入/输出/缓存/推理 token）、`cost`、首字节延迟等。
- **`pi.harness.run`**：一次被接纳的 Harness 运行，记录 `session.id`/`lane.name`/`operation.id`，用于追踪持久化操作的恢复链路。

设计上区分 `startAttributes`（发起时已知，强制要求 required）和 `endAttributes`（完成后补充，永远可选），并提供 `NOOP_TELEMETRY_CONTEXT`（默认关闭遥测时的占位实现）与 `InMemoryTelemetryContext`（测试/诊断用参考实现）。真正接入 OpenTelemetry/Sentry 等后端需要应用自行实现 Adapter，并可用官方提供的 **Adapter 一致性测试套件** 验证语义正确性（原子性、状态合并、嵌套父子关系等）。

这套遥测体系天然衔接下一节的评估体系——评估报告中的 Token/延迟/成本对比数据，正是从这些 Span 中提取的。

---

## 12. Agent 评估体系（Evals）

`pi-evals` 是 Pi 用来做**行为级、模型驱动**的 Agent 质量评估的独立框架，构建在 [`vitest-evals`](https://github.com/getsentry/vitest-evals) 之上，核心思想是：**用真实的 `AgentSession` 跑真实任务，再用断言或 LLM Judge 给结果打分**，而不是脱离真实 Agent 循环的单元测试。

### 12.1 基本运行方式

```bash
# 指定默认 Provider/Model 运行全部评估
npm run eval -- --provider openai --model gpt-5.6-sol

# 只跑某个文件，或按名称过滤
npm run eval -- src/extensions.eval.ts
npm run eval -- -t "creates, reloads, and uses"
```

鉴权复用 Pi 正常的 `ModelRuntime`（订阅凭证或环境变量 API Key），无需为评估单独配置密钥。每次运行会在忽略于版本控制的 `.eval/` 目录下生成 `runs.jsonl` 索引，及每次运行对应的 **原生 Pi 会话 JSONL** 附件（`sessions/` 下）——这意味着每条评估结果都可以完整回放当时 Agent 的每一步工具调用与响应。

### 12.2 Harness：把 AgentSession 接入评估框架

`createPiCodingAgentHarness()` 是核心适配器，把一个真实的编码 Agent 会话包装成 `vitest-evals` 认识的 `Harness`：

```typescript
const harness = createPiCodingAgentHarness({ noTools: "all" });

describeEval("Pi smoke", { harness }, (it) => {
  it("answers a factual question", async ({ run }) => {
    const result = await run("What is the capital of France? Reply with only the city name.");
    expect(result.output).toBe("Paris");
  });
});
```

Harness 支持的关键配置：

| 配置项 | 作用 |
|---|---|
| `name` | 稳定的 Harness 身份标识，用于对比报告 |
| `model` | 显式指定 `{ provider, id }`，覆盖运行器默认模型（模型对比场景必须显式指定） |
| `noTools` | 控制工具禁用范围 |
| `transformSystemPrompt` | 在评估开始前改写完整默认系统提示词 |
| `output` | 把最终响应 + `AgentSession` 转换为 JSON 安全的领域结果，供断言使用 |

`run()` 支持单条 prompt，也支持"prompt + reload"的步骤序列——用于验证"创建资源后重新加载再使用"这类跨会话生命周期的行为：

```typescript
await run([
  { type: "prompt", content: "Create a Pi extension." },
  { type: "reload" },
  { type: "prompt", content: "Use the extension." },
]);
```

### 12.3 对比评估：Baseline vs Candidate

除了单点断言，`pi-evals` 的核心能力是**结构化的 A/B 对比**——比较不同 Prompt、工具集、技能、模型或任意 Harness 配置的效果差异：

```typescript
const harnessTable = evalHarnessTable("target skill effectiveness", {
  baseline: withoutTargetSkillHarness,
  candidate: withTargetSkillHarness,
  repetitions: 6,
});

describe.for(harnessTable)("$name repetition $repetition", ({ harness }) => {
  describeEval("target skill effectiveness", { harness, judges: [TargetTaskJudge], judgeThreshold: null }, (it) => {
    it("completes the target task", async ({ run }) => {
      await run("Complete the target task.");
    });
  });
});
```

设计要点：

- **重复次数（`repetitions`）**：由于 LLM 输出存在随机性，对比评估需要多次重复以获得统计意义上可信的结果。
- **`judgeThreshold: null`**：让低分成为"观察数据"而不是让 Vitest 用例直接失败——对比场景关心的是**趋势**而非单次通过/失败。硬断言只用于套件不变量（infrastructure invariants），不作为打分手段。
- **LLM Judge**：`createJudge<Input, Output>(name, scoringFn)` 用另一个模型或规则函数对输出打分（0~1），作为正确性的量化代理。
- **分组匹配**：同一批评估按"重复次数 + 输入标识（`input.id` 或输入内容的 SHA-256 哈希）"分组，确保 baseline 与每个 candidate 在同一输入、同一次重复下才互相比较。

### 12.4 报告指标

对比报告（`HarnessComparisonReport`）为每一对 `(baseline, candidate)` 计算：

- **正确性提升（Correctness Lift）**：候选组通过率 − 基线组通过率（百分点），通过率定义为"某次运行的平均 Judge 分数 ≥ 1"。
- **配对指标差值（Paired Metric Summary）**：Token 用量、耗时、估算成本三项，均以"候选 − 基线"的配对差值形式呈现（缺失遥测数据的样本标记为不可用，而非按 0 处理）。
- **诊断信息（Diagnostics）**：缺失观测、重复观测、Harness 报错、分数缺失等异常情况会被单独列出，避免被平均值掩盖。

### 12.5 与会话/遥测体系的关系

评估体系并非孤立存在，而是复用了整套基础设施：

```
pi-evals
  └─ createPiCodingAgentHarness()
       └─ AgentSession（与生产环境同一套核心）
            ├─ 产出真实的会话 JSONL（可回放、可人工审查）
            └─ 产出真实的 pi.ai.request / pi.harness.run 遥测 Span
                 └─ 聚合为评估报告中的 Token / 延迟 / 成本对比
```

这种"用生产级 Agent 内核跑评估"的设计避免了"评估环境与生产环境行为不一致"的经典陷阱：评估中观测到的工具调用序列、压缩行为、提示词渲染结果，与真实用户会话完全一致，因为它们本就是同一套代码路径。

---

## 13. 架构设计原则总结

纵览 Pi 的整体设计，可以提炼出几条贯穿始终的原则：

1. **分层解耦，单向依赖**：LLM 抽象（`pi-ai`）、Agent 引擎（`pi-agent-core`）、编码场景实现（`pi-coding-agent`）、UI（`pi-tui`）、协议（`pi-protocol`/`client`/`server`）各自独立，任何一层都可以被单独替换或复用。
2. **事件驱动而非过程式**：Agent 循环、工具执行、会话变更全部通过结构化事件（`agent_start`/`tool_execution_end`/...）暴露，UI、扩展、遥测都是事件的订阅者，彼此不感知。
3. **状态可持久化、可恢复**：会话是树、操作是可挂起的状态机（Harness），保证"进程重启不丢状态"是一等设计目标，而不是事后补丁。
4. **核心极简，能力靠组合**：内核只提供循环与工具契约，扩展/技能/提示词模板三层机制承载几乎所有业务定制，保证核心长期稳定。
5. **安全边界诚实**：不假装提供沙箱，把"信任判断"与"系统级隔离"清晰分开，避免用户对安全边界产生误解。
6. **质量由评估驱动**：`pi-evals` 用生产级 Agent 内核做行为级、可重复、带统计显著性的对比评估，把"这个改动到底有没有让 Agent 更好"变成可衡量的问题，而不是主观判断。

理解以上几点，就能把 Pi 仓库中数百个文件归位到一张清晰的架构图上：**引擎（agent-core）驱动模型（ai）与工具，外壳（coding-agent）承载体验，协议栈（protocol/client/server）负责分布，评估（evals）与遥测（telemetry）保证可衡量的持续改进。**
