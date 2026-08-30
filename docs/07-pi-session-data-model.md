# Pi 会话数据模型与持久化格式

> 系列第 07 篇。前面几篇多次提到"会话是一棵树"、"压缩会追加一个 `CompactionEntry`"、"扩展可以写入 `CustomEntry`"，但没有系统讲过这棵树到底长什么样。这一篇把会话的数据模型完整摊开——它既是理解 `/tree`、压缩、扩展持久化的基础，也是编程解析会话文件（做统计、审计、迁移工具）的必读参考。

## 目录

1. [为什么是树而不是线性列表](#1-为什么是树而不是线性列表)
2. [文件布局与命名](#2-文件布局与命名)
3. [Entry 基类与树的构成](#3-entry-基类与树的构成)
4. [消息类型全谱：AgentMessage](#4-消息类型全谱agentmessage)
5. [十一种 Entry 类型](#5-十一种-entry-类型)
6. [哪些 Entry 进入 LLM 上下文](#6-哪些-entry-进入-llm-上下文)
7. [retainedTail：压缩格式的演进](#7-retainedtail压缩格式的演进)
8. [版本迁移：v1 → v2 → v3](#8-版本迁移v1--v2--v3)
9. [编程解析会话文件](#9-编程解析会话文件)
10. [设计小结](#10-设计小结)

---

## 1. 为什么是树而不是线性列表

多数 Chat 应用把对话存成线性消息数组。Pi 存成树，因为编码工作有一个特征：**经常需要"回到某一步换个思路重试"**。

线性存储下要支持这件事，只有两个选择：要么截断（丢失被放弃的分支）、要么复制成新文件（分支之间失去关联，无法对比）。树结构直接解决了这个问题——每个 entry 有 `id`/`parentId`，当前位置是"活跃叶子节点"，从任意历史节点长出新分支只是"新 entry 的 `parentId` 指向那个节点"。

```text
├─ user: "Hello, can you help..."
│  └─ assistant: "Of course! I can..."
│     ├─ user: "Let's try approach A..."
│     │  └─ assistant: "For approach A..."
│     │     └─ user: "That worked..."  ← active
│     └─ user: "Actually, approach B..."
│        └─ assistant: "For approach B..."
```

官方文档的表述是 "enabling in-place branching without creating new files"——**原地分支，不产生新文件**。这一条决定了 `/tree`（在同一文件内探索多个方案）和 `/fork`/`/clone`（真的产生新文件）是三个语义不同的操作：

| 特性 | `/tree` | `/fork` | `/clone` |
|---|---|---|---|
| 输出 | 同一会话文件 | 新会话文件 | 新会话文件 |
| 视图 | 完整树 | 用户消息选择器 | 当前活跃分支 |
| 典型用途 | 原地探索备选方案 | 从早期 prompt 开一个新会话 | 在继续前复制当前工作 |
| 摘要 | 可选分支摘要 | 无 | 无 |

---

## 2. 文件布局与命名

```
~/.pi/agent/sessions/--<path>--/<timestamp>_<uuid>.jsonl
```

`<path>` 是工作目录，`/` 替换为 `-`。按 cwd 分目录的效果是 `pi -c`（继续最近会话）、`pi -r`（浏览历史会话）天然只看当前项目的会话，不需要额外的索引或过滤。

文件格式是 JSONL——每行一个独立 JSON 对象。这个选择带来三个具体好处：

- **追加即写入**：新 entry 直接 append 一行，不需要读改写整个文件。对于一个可能有几千条 entry 的长会话，这是性能上的必要条件。
- **崩溃不损坏历史**：进程在写入中途崩溃最多损坏最后一行，前面的内容完好可解析。这也是第 04 篇提到的"会话历史消息级可恢复"的物理基础。
- **可流式解析**：做统计工具时不需要把整个文件读进内存。

删除会话就是删 `.jsonl` 文件；`/resume` 里 `Ctrl+D` 交互删除时，Pi **优先调用 `trash` CLI 而不是永久删除**——一个小细节，但符合"用户数据默认可恢复"的取向。

---

## 3. Entry 基类与树的构成

除文件首行的 `SessionHeader` 外，所有 entry 继承同一个基类：

```typescript
interface SessionEntryBase {
  type: string;
  id: string;                // 8-char hex ID
  parentId: string | null;   // Parent entry ID (null for first entry)
  timestamp: string;         // ISO timestamp
}
```

`id` 是 8 字符 hex（不是 UUID）——足够短便于在 `/tree` UI 和命令行里直接引用（`--session <partial-id>` 支持部分 ID 匹配），冲突概率在单个会话文件的规模下可忽略。

首行 `SessionHeader` **不属于树**（没有 `id`/`parentId`），它是纯元数据：

```json
{"type":"session","version":3,"id":"uuid","timestamp":"2024-12-03T14:00:00.000Z","cwd":"/path/to/project"}
```

通过 `/fork`、`/clone` 或 `newSession({ parentSession })` 创建的会话多一个 `parentSession` 字段指向原文件路径：

```json
{"type":"session","version":3,"id":"uuid","cwd":"/path","parentSession":"/path/to/original/session.jsonl"}
```

这条链接让"这个会话是从哪来的"可追溯——`/fork` 虽然产生了新文件，但血缘关系没有丢失。

---

## 4. 消息类型全谱：AgentMessage

`SessionMessageEntry.message` 字段的类型是 `AgentMessage`，它是一个七元联合类型，来自两层：

```typescript
type AgentMessage =
  // 来自 pi-ai（LLM 原生理解的三种）
  | UserMessage
  | AssistantMessage
  | ToolResultMessage
  // 来自 pi-coding-agent（Pi 专属扩展的四种）
  | BashExecutionMessage
  | CustomMessage
  | BranchSummaryMessage
  | CompactionSummaryMessage;
```

这个分层正好对应第 01 篇讲的 `AgentMessage` → `convertToLlm()` → `Message` 转换管线：**前三种直接对应 LLM 角色，后四种是 Pi 自己的概念，需要转换或过滤**。

### 4.1 基础三种（pi-ai）

```typescript
interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: string;
  provider: string;
  model: string;
  usage: Usage;
  stopReason: "stop" | "length" | "toolUse" | "error" | "aborted";
  errorMessage?: string;
  timestamp: number;
}
```

`AssistantMessage` 逐条记录 `provider`/`model`/`api`/`usage`——**每条消息都知道自己是哪个模型产生的、花了多少钱**。这是第 05 篇 `cache-stats.ts` 能"按消息自身实际单价反推浪费成本"、以及跨模型会话能正确统计总成本的数据基础。如果 usage 只记在会话级别，模型切换后的成本归因就无法进行。

`stopReason` 是第 04 篇长任务机制的核心判据：`"error"` 触发 Auto-Retry 判定，`"length"` 参与压缩触发判断，`"stop"` 表示正常结束。

文档特意指出一个边界：`StopReason` 类型还包含 `"pending"`，但那只用于流式事件中的部分消息，**持久化的 assistant 消息里永远不会出现 `"pending"`**——解析器不需要处理这个值。

```typescript
interface ToolResultMessage {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];
  details?: any;      // Tool-specific metadata
  usage?: Usage;      // Nested LLM work performed by the tool
  isError: boolean;
  timestamp: number;
}
```

两个字段值得注意：

- **`details`**：工具专属元数据，**不发给 LLM**，只用于 UI 渲染。第 03 篇提到的 `subagent` 工具把子 Agent 完整消息历史放在这里供 TUI 展开查看，就是用的这个字段。
- **`usage?`**：工具内部如果自己调用了 LLM（如 `subagent`、或某个扩展工具做了摘要），这部分开销记在这里并计入会话总成本。**嵌套的 LLM 开销不会从成本统计里漏掉**。

### 4.2 Pi 专属四种

```typescript
interface BashExecutionMessage {
  role: "bashExecution";
  command: string;
  output: string;
  exitCode: number | undefined;
  cancelled: boolean;
  truncated: boolean;
  fullOutputPath?: string;
  excludeFromContext?: boolean;  // true for !! prefix commands
  timestamp: number;
}
```

这是交互模式下用户直接敲 `!command` 的记录（区别于 LLM 调用 `bash` 工具产生的 `ToolResultMessage`）。三个字段体现了第 04/05 篇讨论过的机制：`truncated` + `fullOutputPath` 是输出治理（截断 + 完整内容落盘），`cancelled` 是中止传播的记录。

`excludeFromContext` 对应 `!!` 前缀——用户想跑个命令看看结果但**不想让它进入模型上下文**（比如查一个跟当前任务无关的东西，或者输出里有敏感信息）。这是一个把"上下文成本控制"直接交给用户的细粒度开关。

```typescript
interface CustomMessage {
  role: "custom";
  customType: string;            // Extension identifier
  content: string | (TextContent | ImageContent)[];
  display: boolean;              // Show in TUI
  details?: any;                 // Extension-specific metadata
  timestamp: number;
}
```

扩展注入的消息。`customType` 标识来源扩展，`display` 控制是否在 TUI 显示（可以有"发给模型但用户看不到"或反之的组合）。v3 版本把这个 role 从 `hookMessage` 改名为 `custom`，是扩展系统统一化的一部分。

```typescript
interface BranchSummaryMessage { role: "branchSummary"; summary: string; fromId: string; timestamp: number; }
interface CompactionSummaryMessage { role: "compactionSummary"; summary: string; tokensBefore: number; timestamp: number; }
```

分支摘要与压缩摘要——第 01 §8 和第 04 §5 讨论的压缩机制在数据层的落点。`tokensBefore` 记录压缩前的 token 数，用于向用户展示压缩效果。

### 4.3 内容块

```typescript
interface TextContent { type: "text"; text: string; }
interface ImageContent { type: "image"; data: string; mimeType: string; }   // base64
interface ThinkingContent { type: "thinking"; thinking: string; }
interface ToolCall { type: "toolCall"; id: string; name: string; arguments: Record<string, any>; }
```

`ThinkingContent` 独立成一种块类型（而非塞进 text）是跨 Provider Handoff 的前提：第 01 §5 提到"切换 Provider 时 thinking 块自动转换为带 `<thinking>` 标签的文本"，能这样做正是因为思考内容在数据模型里是可识别的独立块。

---

## 5. 十一种 Entry 类型

| Entry 类型 | 作用 | 进入 LLM 上下文 |
|---|---|---|
| `session` | 文件首行元数据（不在树中） | — |
| `message` | 对话消息，携带 `AgentMessage` | 是（取决于消息类型） |
| `model_change` | 用户中途切换模型 | 否 |
| `thinking_level_change` | 用户切换思考等级 | 否 |
| `compaction` | 压缩摘要 + 保留边界 | 是（摘要作为上下文） |
| `branch_summary` | 分支摘要 | 是 |
| `custom` | 扩展状态持久化 | **否** |
| `custom_message` | 扩展注入的消息 | **是** |
| `label` | 用户在某个 entry 上打的书签 | 否 |
| `session_info` | 会话显示名等元数据 | 否 |

几个关键的设计点：

### 5.1 `custom` vs `custom_message`：扩展的两种持久化需求

这一对是整个数据模型里最容易混淆但设计最清晰的地方：

```json
// custom：扩展自己的状态，不参与 LLM 上下文
{"type":"custom","id":"h8i9j0k1","parentId":"g7h8i9j0","customType":"my-extension","data":{"count":42}}

// custom_message：扩展注入的上下文，参与 LLM 上下文
{"type":"custom_message","id":"i9j0k1l2","parentId":"h8i9j0k1","customType":"my-extension","content":"Injected context...","display":true}
```

扩展有两种截然不同的持久化需求：**"我需要记住一个计数器/配置，重载后恢复"**（`custom`，`data` 字段任意 JSON），和**"我要往对话里插一段内容让模型看到"**（`custom_message`，`content` 字段是消息内容）。

把它们分成两种 entry 类型，而不是用一个类型加 flag 区分，好处是解析器和上下文构建逻辑都不会搞错——"要构建 LLM 上下文"时可以直接按 type 过滤，不需要检查某个布尔字段。

`custom` entry 还可以通过 `pi.registerEntryRenderer(customType, renderer)` 在交互模式下自定义渲染——**能显示不等于进入上下文**，这两件事被明确解耦。

### 5.2 `model_change` / `thinking_level_change`：配置变更也是历史

这两种 entry 记录用户在会话中途的配置变更。它们不参与 LLM 上下文，但它们的存在让会话文件成为**完整的操作审计记录**：回看一个会话时能知道"这一段是用 Haiku 跑的、从这里开始换成了 Sonnet 且思考等级提到了 high"。

这也是第 05 篇 `cache-stats.ts` 能识别"Cache miss after model switch"的依据之一——切模型这个事件在数据层是可查的。

### 5.3 `label`：用户侧的锚点

```json
{"type":"label","id":"j0k1l2m3","parentId":"i9j0k1l2","targetId":"a1b2c3d4","label":"checkpoint-1"}
```

注意 `label` entry 自己也在树里（有 `id`/`parentId`），但它通过 `targetId` 指向被标记的 entry。这种"标注作为独立 entry"的设计使得 append-only 文件也能支持"给历史 entry 加标注"——不需要修改已写入的行。设 `label` 为 `undefined` 即清除标注（同样是追加一条新 entry）。

`/tree` 的 `labeled-only` 过滤模式配合它使用，在很长的会话里快速跳到自己标记的关键节点。

---

## 6. 哪些 Entry 进入 LLM 上下文

这是解析会话文件时最需要搞清楚的问题。按第 5 节表格，规则可以概括为：

```
进入上下文：
  message（role 为 user/assistant/toolResult/bashExecution(除非 excludeFromContext)/
           custom/branchSummary/compactionSummary）
  custom_message
  compaction（摘要文本 + retainedTail 之后的内容）
  branch_summary

不进入上下文：
  session（头部元数据）
  model_change / thinking_level_change（配置变更记录）
  custom（扩展状态）
  label（用户标注）
  session_info（会话名）
```

再叠加压缩的影响：一次压缩之后，**LLM 实际看到的上下文 = 系统提示词 + 压缩摘要 + `firstKeptEntryId`（或 `retainedTail`）之后的内容**，压缩点之前的原始 entry 虽然仍在文件里（用于 `/tree` 回看），但不再发给模型。

这解释了一个常见困惑：会话文件很大 ≠ 上下文很大。文件是完整历史的档案，上下文是当前发给模型的切片。

---

## 7. retainedTail：压缩格式的演进

`CompactionEntry` 有两种格式，反映了一次实际的设计改进：

**旧格式（用指针）：**

```json
{"type":"compaction","id":"f6g7h8i9","parentId":"e5f6g7h8","summary":"User discussed X, Y, Z...","firstKeptEntryId":"c3d4e5f6","tokensBefore":50000}
```

**新格式（内嵌内容）：**

```json
{"type":"compaction","id":"f6g7h8i9","parentId":"e5f6g7h8","summary":"User discussed X, Y, Z...","tokensBefore":50000,"retainedTail":[{"role":"user","content":"latest request"},{"role":"assistant","content":[...],"stopReason":"stop"}]}
```

官方对 `retainedTail` 的说明点出了问题所在：

> Materialized `AgentMessage[]` kept after compaction. This is optional only for backward compatibility with older sessions. Newer harness-generated compactions include it so we can rebuild context from this checkpoint **without walking older entries before the compaction entry**.

`firstKeptEntryId` 是一个**指针**——要重建压缩后的上下文，必须回头遍历压缩点之前的 entry 找到那个 id 并从它开始收集消息。`retainedTail` 直接把保留的消息**内容**物化进 compaction entry 本身，重建上下文只需要读这一条 entry。

差异在两个场景下变得重要：

1. **多次压缩的会话**：旧格式下，第 3 次压缩要重建上下文需要往前追溯到第 2 次压缩的保留点，再往前……变成一条指针链。新格式下每个 compaction entry 都是自包含的检查点。
2. **只读取尾部的场景**（如快速恢复、外部工具做增量分析）：新格式下不需要解析整个文件。

这是一个典型的"用空间换确定性"取舍——`retainedTail` 会重复存储一部分消息内容（它们在前面的 message entry 里也有），换来"每个压缩点都是一个自包含检查点"这个更强的性质。第 01 §4 提到的 `AgentHarness` 之所以能把压缩建模为可持久化恢复的操作，依赖的正是这个自包含性。

`CompactionEntry` 的其他可选字段也值得一提：

- `usage`：生成摘要这次 LLM 调用的开销，**计入会话 token 与成本总计**——对应第 05 §5.6 讲的"压缩自身成本明确显示"。
- `details`：默认是 `{ readFiles: string[], modifiedFiles: string[] }`（压缩时追踪的文件读写记录），扩展可以放自定义数据。
- `fromHook`：`true` 表示这次压缩由扩展生成而非 Pi 内置逻辑（字段名是历史遗留，语义是"来自扩展"）。

---

## 8. 版本迁移：v1 → v2 → v3

```
Version 1: 线性 entry 序列（legacy，加载时自动迁移）
Version 2: 树结构，通过 id/parentId 关联
Version 3: 把 hookMessage role 改名为 custom（扩展系统统一化）
```

"Existing sessions are automatically migrated to the current version (v3) when loaded"——**加载时迁移，不需要用户执行任何迁移命令**。

三个版本的变更性质不同，也说明了什么样的改动需要版本号：

- **v1→v2 是结构性变更**：从线性数组变成树，`parentId` 字段从无到有。旧会话的线性序列可以机械地转换为一条单链树（每个 entry 的 `parentId` 指向前一个）。
- **v2→v3 是命名变更**：`hookMessage` → `custom`。虽然只是改名，但因为字段值会被解析器和扩展代码读取，仍然需要版本号来触发迁移。

Pi 的整体规则（见项目 AGENTS.md）是"不保留向后兼容性除非用户要求"，但**会话数据是例外**——用户的历史会话是不可再生的资产，破坏它们的读取能力代价太高。所以这里采用了"格式演进 + 自动迁移"而非"直接改格式"。

`CompactionEntry` 的 `firstKeptEntryId` 也被保留为"for compatibility with old entry format"，同样是这个原则的体现：新代码写新格式，但永远能读旧格式。

---

## 9. 编程解析会话文件

做统计、审计、迁移工具时的实用要点：

### 9.1 类型定义在哪

| 内容 | 位置 |
|---|---|
| Entry 类型 + `SessionManager` API | `packages/coding-agent/src/core/session-manager.ts` |
| Pi 扩展消息类型（`BashExecutionMessage` 等） | `packages/coding-agent/src/core/messages.ts` |
| 基础消息类型（`UserMessage`/`AssistantMessage`/`ToolResultMessage`） | `packages/ai/src/types.ts` |
| `AgentMessage` 联合类型 | `packages/agent/src/types.ts` |

在自己项目里用时，可以直接看 `node_modules/@earendil-works/pi-coding-agent/dist/` 和 `node_modules/@earendil-works/pi-ai/dist/` 的 `.d.ts`。

### 9.2 解析的基本流程

```
1. 读第一行 → SessionHeader，检查 version（如果不是 3，注意可能是旧格式）
2. 逐行解析剩余行 → 按 type 分派
3. 用 id/parentId 构建树（或者只关心活跃分支：从最后一个 entry 沿 parentId 回溯到根）
4. 要重建 LLM 上下文：找到最后一个 compaction entry，取其 summary + retainedTail
   （或 firstKeptEntryId 之后的消息），再追加之后的消息
5. 要统计成本：累加所有 AssistantMessage.usage、ToolResultMessage.usage、
   CompactionEntry.usage、BranchSummaryEntry.usage
```

第 5 步的四个 usage 来源容易漏——只统计 `AssistantMessage.usage` 会低估总成本，因为工具内部的嵌套 LLM 调用（`subagent`）和压缩/摘要的开销都在别的地方。

### 9.3 会话信息的现成入口

不写代码的话，`/session` 命令直接显示当前会话文件路径、session ID、消息数、token 数和成本；`/export [file]` 导出为 HTML；`/share` 上传为 private GitHub gist 并给出可分享的 HTML 链接。

---

## 10. 设计小结

1. **树结构是为"回退重试"这个真实工作模式服务的**，不是为了炫技。它直接换来了 `/tree` 原地分支、`/fork` 血缘追溯、分支摘要三项能力，而这些在线性存储下要么做不到、要么要引入额外的版本管理机制。

2. **JSONL + append-only 是可靠性的基础**：崩溃最多损坏最后一行、写入无需读改写、可流式解析。第 04 篇讲的"会话历史消息级可恢复"不是靠什么复杂机制，就是靠这个格式选择。

3. **"进入上下文"与"持久化"是两个正交维度**，且在类型层面被明确区分（`custom` vs `custom_message`、`details` 字段不发给 LLM、`excludeFromContext` 标记）。这种正交性让"我要存东西"和"我要影响模型"两种需求不会互相干扰。

4. **每条消息自带完整的模型与成本溯源**（`provider`/`model`/`api`/`usage`），这是跨模型会话能正确统计成本、`cache-stats.ts` 能精确归因浪费的前提。数据模型层面的这个选择，支撑了第 05 篇整套成本可观测性。

5. **`retainedTail` 展示了"自包含检查点优于指针链"的取舍**：接受一定的存储冗余，换来每个压缩点都能独立重建上下文。这种取舍在需要恢复能力的系统里通常是对的。

6. **会话数据是向后兼容规则的例外**。项目整体不追求向后兼容，但历史会话不可再生，所以采用"版本号 + 加载时自动迁移 + 永远能读旧格式"。什么该兼容、什么不该兼容，取决于数据能否重建,而不是一刀切的原则。
