# Pi 多 Agent 协作机制与上下文管理详解

> 系列第 03 篇（[系列索引](README.md)）。本文聚焦 Pi 生态中"多 Agent 协作"这一具体实现——官方 `subagent` 示例扩展（`packages/coding-agent/examples/extensions/subagent/`）。内容覆盖：协作的整体流程、内部实现机制（进程、协议、并发控制），以及最核心的问题——上下文是如何在主 Agent 和子 Agent 之间被管理和隔离的。

## 目录

1. [协作模型总览](#1-协作模型总览)
2. [Agent 定义：Markdown + Frontmatter](#2-agent-定义markdown--frontmatter)
3. [内部机制：如何委派一个任务](#3-内部机制如何委派一个任务)
4. [三种协作模式的具体实现](#4-三种协作模式的具体实现)
5. [上下文管理的核心：进程级隔离](#5-上下文管理的核心进程级隔离)
6. [上下文如何在 Agent 之间传递](#6-上下文如何在-agent-之间传递)
7. [上下文体量控制：多层截断与压缩](#7-上下文体量控制多层截断与压缩)
8. [权限与安全边界](#8-权限与安全边界)
9. [与单 Agent 内部上下文管理机制的对比](#9-与单-agent-内部上下文管理机制的对比)
10. [设计小结](#10-设计小结)

---

## 1. 协作模型总览

Pi 本身的核心引擎（`pi-agent-core`）**不提供**多 Agent 协作能力——`Agent`/`AgentSession` 都是单一会话的抽象。多 Agent 协作完全是通过**扩展系统 + 操作系统进程**这两层机制组合出来的：一个扩展注册一个叫 `subagent` 的工具，这个工具的 `execute()` 函数不是本地计算，而是**再拉起一个完整的 `pi` 子进程**去跑另一个独立的 Agent。

整体协作模型：

```
主 Agent（当前交互会话）
   │  调用工具 subagent({ agent: "scout", task: "..." })
   ▼
subagent 工具 execute()
   │  spawn 子进程：pi --mode json -p --no-session --model ... --tools ... "Task: ..."
   ▼
子 Agent（独立 pi 进程，独立 AgentSession，独立上下文窗口）
   │  子进程通过 stdout 逐行输出 JSON 事件（message_end / tool_result_end / ...）
   ▼
subagent 工具解析事件流，提取最终文本 + usage 统计
   │  作为工具结果返回
   ▼
主 Agent 只看到"这次委派任务的最终结论"，不看到子 Agent 内部的探索过程
```

这个模型的关键特征：**子 Agent 是完全独立的 `pi` 进程**，不是同进程内的另一个 `Agent` 实例。这个设计选择直接决定了上下文管理的方式——下文会看到，隔离是通过进程边界"免费"获得的，而不需要在 `pi-agent-core` 里实现任何特殊的多会话隔离逻辑。

---

## 2. Agent 定义：Markdown + Frontmatter

每个可被委派的"专职 Agent"是一个 Markdown 文件，YAML frontmatter 声明元数据，正文是系统提示词：

```markdown
---
name: scout
description: Fast codebase recon that returns compressed context for handoff to other agents
tools: read, grep, find, ls, bash
model: claude-haiku-4-5
---

You are a scout. Quickly investigate a codebase and return structured
findings that another agent can use without re-reading everything.
...
## Files Retrieved
List with exact line ranges: ...
## Key Code
## Architecture
## Start Here
```

官方内置四个示例 Agent，分工非常明确：

| Agent | 职责 | 模型 | 工具 |
|---|---|---|---|
| `scout` | 快速侦察，返回压缩后的结构化上下文 | Haiku（便宜快） | read/grep/find/ls/bash |
| `planner` | 基于侦察结果制定实施方案 | Sonnet | read/grep/find/ls（只读） |
| `reviewer` | 代码评审 | Sonnet | read/grep/find/ls/bash |
| `worker` | 通用实现，全部默认工具 | Sonnet | 默认全部工具 |

**发现规则**（`agents.ts` 的 `discoverAgents()`）：

- 默认 `agentScope: "user"`，只加载 `~/.pi/agent/agents/*.md`
- 传入 `agentScope: "project"` 或 `"both"` 才会额外加载 `.pi/agents/*.md`（沿目录树向上找最近的 `.pi/agents`）
- `"both"` 模式下同名 Agent，项目级覆盖用户级（`Map` 按 `name` 去重，后写入的项目 Agent 覆盖先写入的用户 Agent）
- 单个 Agent 文件解析失败（缺 `name`/`description`）会被静默跳过，不影响其他 Agent 被发现——一个坏文件不能拖垒整个发现流程

**模型继承规则**：frontmatter 里的 `model` 是可选的；省略时，子 Agent 继承**发起委派的那个会话**当前激活的模型和思考等级（`dispatchDefaults`），而不是用固定默认值——这样主会话切换模型后，子 Agent 也能跟着变化，行为符合直觉。

---

## 3. 内部机制：如何委派一个任务

`runSingleAgent()` 是最底层的委派执行单元，完整链路：

### 3.1 构造子进程命令行

```typescript
const args = ["--mode", "json", "-p", "--no-session"];
if (model) args.push("--model", model);
if (inheritsDispatchConfig && thinkingLevel) args.push("--thinking", thinkingLevel);
if (agent.tools) args.push("--tools", agent.tools.join(","));
if (agent.systemPrompt.trim()) args.push("--append-system-prompt", tmpPromptPath);
args.push(`Task: ${task}`);
```

四个关键标志决定了子 Agent 的运行形态：

- **`--mode json`**：让子进程以结构化 JSON 事件流打印到 stdout（而不是渗染 TUI），父进程逐行解析
- **`-p`**：一次性 print 模式，处理完 prompt 就退出，不等待交互输入
- **`--no-session`**：**不持久化会话**——子 Agent 的整个对话过程不会写入 JSONL、不会出现在 `/tree`/`/resume` 里，是纯粹的"用完即扔"
- **`--append-system-prompt <tmpfile>`**：Agent 的系统提示词通过一个临时文件传入（而非命令行参数字符串），避免超长提示词或特殊字符在命令行里被 shell 转义搞坏；执行完立即用 `finally` 块清理临时文件和临时目录

进程本身通过 `getPiInvocation()` 智能判断当前是 Node 脚本还是 Bun 编译的二进制（检测 `/$bunfs/root/` 虚拟路径），用同一个可执行文件重新拉起自己,保证子 Agent 用的是**同一版本的 pi**,不依赖 PATH 上可能存在的另一个版本。

### 3.2 事件流解析

子进程 stdout 是逐行 JSON,父进程按行缓冲解析（`buffer.split("\n")`，保留最后一段不完整行等下一次数据到达再拼接）：

```typescript
if (event.type === "message_end" && event.message) {
  currentResult.messages.push(event.message);
  if (msg.role === "assistant") {
    currentResult.usage.turns++;
    // 累加 usage.input/output/cacheRead/cacheWrite/cost
    currentResult.usage.contextTokens = usage.totalTokens;
  }
  emitUpdate();  // 实时推送给父 Agent 的 UI（onUpdate 回调）
}
if (event.type === "tool_result_end" && event.message) {
  currentResult.messages.push(event.message);
  emitUpdate();
}
```

只监听 `message_end` 和 `tool_result_end` 两类事件（对应架构文中介绍的 Agent Loop 事件模型的一个子集）,忽略流式的 `message_update` 增量——因为父进程只需要"这一步完整发生了什么"用于展示,不需要逐字流式转发。

### 3.3 中止（Abort）传播

传入的 `AbortSignal` 绑定到子进程的 kill 逻辑：先 `SIGTERM`，5 秒后如果进程仍未退出再 `SIGKILL`。这保证了用户在主会话按 Ctrl+C 时，中止信号能一路传播到最底层的子进程，不会留下孤儿进程。

---

## 4. 三种协作模式的具体实现

### 4.1 Single（单任务）

`{ agent: "scout", task: "..." }` → 直接调用一次 `runSingleAgent()`，等待结果，包装成工具结果返回。

### 4.2 Parallel（并行）

```typescript
{ tasks: [{ agent: "scout", task: "..." }, { agent: "scout", task: "..." }] }
```

用一个简单的"信号量式"并发控制器 `mapWithConcurrencyLimit()` 实现：

```typescript
const limit = Math.min(concurrency, items.length);
const workers = Array(limit).fill(null).map(async () => {
  while (true) {
    const current = nextIndex++;
    if (current >= items.length) return;
    results[current] = await fn(items[current], current);
  }
});
await Promise.all(workers);
```

限制是硬编码的 `MAX_PARALLEL_TASKS = 8`、`MAX_CONCURRENCY = 4`——最多 8 个任务排队，同时最多 4 个子进程在跑，超出部分排队等待空闲 worker。每个任务的中间进度通过独立的 `onUpdate` 回调实时汇总，UI 层能展示"2/3 done, 1 running"这样的聚合状态。

### 4.3 Chain（链式）

```typescript
{ chain: [
  { agent: "scout", task: "Find all auth code" },
  { agent: "planner", task: "Plan based on: {previous}" },
  { agent: "worker", task: "Implement: {previous}" },
] }
```

顺序执行，每一步把 `{previous}` 占位符替换为**前一步的最终文本输出**（`getFinalOutput()` 提取上一步最后一条 assistant 消息的文本内容），再作为下一步任务描述的一部分传给下一个子进程。任意一步失败（`isFailedResult`：退出码非 0，或 `stopReason` 是 `"error"`/`"aborted"`），链条立即停止，报告是哪一步失败。

配套的 Prompt Template（`prompts/implement.md`）把这个链式调用封装成一个斜杠命令：

```
/implement add Redis caching to the session store
```

展开后指示主 Agent："用 subagent 工具的 chain 参数，先 scout 找相关代码，再 planner 基于 {previous} 出方案，最后 worker 基于 {previous} 实现。"——即链式编排的顺序本身也是**提示词层面**声明的，不是工具内部硬编码的固定流程,工具只提供"链式执行 + 占位符替换"这个通用机制。

---

## 5. 上下文管理的核心：进程级隔离

这是本文最关键的部分。多 Agent 协作解决的根本问题是：**单一长对话的上下文窗口会被无关的中间探索过程污染、迅速膨胀**。Pi 的解法不是在 `pi-agent-core` 内部做什么精细的"子会话"抽象，而是最朴素也最彻底的隔离手段——**操作系统进程边界**。

### 5.1 为什么进程隔离是"免费"的强隔离

子 Agent 是一个全新的 `pi --mode json -p --no-session` 进程，这意味着：

- 它有自己独立的 `AgentSession` 实例，从零开始的空消息历史开始对话——**完全不共享**主 Agent 当前对话里已经积累的任何消息（不管主会话现在有多少轮对话、读过多少文件、上下文用了多少 Token，子 Agent 都感知不到）
- 它有自己独立的系统提示词（来自 Agent 定义的 frontmatter body，通过 `--append-system-prompt` 追加，而不是继承主会话的系统提示词）
- 它有自己独立的工具集（由 `tools:` frontmatter 限定，例如 `scout` 只有只读工具 + bash，没有 `write`/`edit`）
- 它有自己独立的模型（可以是完全不同厂商/规格的模型，如子任务用便宜的 Haiku，主会话用 Sonnet）
- 它的会话**不持久化**（`--no-session`），进程退出后这段对话完全消失，不会出现在任何会话树里

这五点合起来，子 Agent 的上下文窗口是一个真正意义上的"白板"——只包含它自己被分配的任务描述和自己产生的工具调用历史，不带任何主会话的"包袱"。

### 5.2 主 Agent 侧看到的是什么

反过来看主 Agent 这一侧：子进程内部产生的**所有**中间消息（工具调用、探索性的读文件、多轮思考）**都不会**进入主 Agent 的上下文。主 Agent 通过 `subagent` 工具调用最终只拿到一个**工具结果（ToolResult）**，其 `content` 字段是 `getFinalOutput()` 提取出的**最后一条 assistant 消息的纯文本**：

```typescript
function getFinalOutput(messages: Message[]): string {
  for (let i = messages.length - 1; i >= 0; i--) {
    const msg = messages[i];
    if (msg.role === "assistant") {
      for (const part of msg.content) {
        if (part.type === "text") return part.text;
      }
    }
  }
  return "";
}
```

也就是说，无论子 Agent 内部读了 20 个文件、跑了 15 次 grep，主 Agent 的上下文里只会新增**一条工具结果消息**，内容是子 Agent 精心组织的总结文本（比如 `scout` 的 Markdown 结构化输出）。这正是"上下文压缩前置到源头"的设计：不是等主会话膨胀了再去压缩，而是从一开始就不让无关细节进入主上下文。

`details` 字段（`SubagentDetails`，包含 `results: SingleResult[]` 完整消息历史）确实会附着在工具结果上，但这部分数据**只用于本地 TUI 渲染**（展开视图里能看到子 Agent 的每一步工具调用），**不会被序列化进发给 LLM 的消息**——这是 `pi-agent-core` 工具结果模型里"面向人类的展示细节"和"面向模型的内容"分离的直接应用。

### 5.3 三种模式下的上下文隔离粒度

| 模式 | 隔离粒度 |
|---|---|
| Single | 一次委派 = 一个独立上下文窗口，用完即弃 |
| Parallel | 每个并行任务各自一个独立进程/上下文窗口，互相之间也完全隔离——`scout` 任务 A 和 `scout` 任务 B 互不知道对方在做什么 |
| Chain | 每一步各自一个独立上下文窗口；步骤间**不共享消息历史**，只共享 `{previous}` 占位符替换进去的**一段文本** |

Chain 模式的这个细节值得强调：即便是"顺序执行、后一步依赖前一步"的场景，Pi 依然坚持不做"完整消息历史转移"，而是只做"提取最终文本 → 塞进下一步的任务描述里"这种最小化的信息传递。这保证了链条越长，每一步的上下文负担也不会累积膨胀——`planner` 步骤的上下文里只有"scout 的总结文本"，不会有"scout 读的所有原始文件内容"。

---

## 6. 上下文如何在 Agent 之间传递

隔离是"默认状态"，但协作总需要有信息流动，Pi 设计了三条明确、有限的信息传递通道：

### 6.1 任务描述（Task Text）——唯一的输入通道

子 Agent 能获得的**全部**输入是：`Task: {task}` 这一段文本（外加自己的系统提示词）。主 Agent 必须在发起委派前，把子 Agent 需要知道的一切都写进 `task` 参数里——这是刻意的设计约束：迫使调用方（LLM 自己）显式思考"这个子任务真正需要哪些信息"，而不是依赖某种隐式的上下文继承。

### 6.2 最终文本输出——唯一的返回通道

如 5.2 节所述，子 Agent 能带回给调用方的**全部**输出是 `getFinalOutput()` 提取的最后一条助手文本消息。这就是为什么 `scout` 这类 Agent 的系统提示词特意要求它按固定的 Markdown 结构（`## Files Retrieved` / `## Key Code` / `## Architecture` / `## Start Here`）输出——因为这是它与外部世界唯一的接口，必须自己把探索过程中的关键信息"打包"进这一段文本里，探索过程本身（读了哪些文件、试了哪些 grep）不会自动带出去。

### 6.3 `{previous}` 占位符——Chain 模式下的步间传递

`step.task.replace(/\{previous\}/g, previousOutput)`：纯字符串替换，把上一步的最终文本输出，拼进下一步任务描述的指定位置。这是 6.1 和 6.2 两条通道的组合应用——上一步的"返回通道"输出，变成下一步的"输入通道"内容,除此之外再无其他信息流动路径。

### 6.4 Parallel 模式下的结果汇总——多对一的传递

并行模式下，每个任务的输出会被 `truncateParallelOutput()` 截断到 50KB 以内（超出部分保留在 `details` 里供本地展开查看，但不进入模型可见内容），再拼成一份多任务汇总文本返回给主 Agent：

```
Parallel: 2/2 succeeded

### [scout] completed
{scout 任务 A 的输出}

---

### [scout] completed
{scout 任务 B 的输出}
```

主 Agent 拿到的是"结构化拼接后的汇总"，而不是分别两条工具结果——这符合"一次工具调用产生一次工具结果"的 Agent Loop 约定，同时把多个子任务的产出合并呈现，方便主模型统一消化。

---

## 7. 上下文体量控制：多层截断与压缩

即便有了进程隔离，"返回文本仍可能过长"这个问题依然存在（比如 `worker` 完成了一个大改动，总结也可能很长）。Pi 在这条路径上叠加了几层防护：

```
子 Agent 内部（每个独立进程自己的 AgentSession）
  └─ 完全遵循单 Agent 的自动压缩机制（见架构文第 8 节）
       └─ 子 Agent 自己的上下文超阈值时会自己触发压缩，与主 Agent 无关

子 Agent → 主 Agent 的返回路径
  └─ getFinalOutput() 只提取最后一条文本消息（天然的"摘要优先"设计）
  └─ Parallel 模式：truncateParallelOutput() 硬截断到 50KB/任务
  └─ details 中的完整消息历史只用于 TUI 展示，不进入模型上下文

主 Agent 侧
  └─ 收到的工具结果作为一条普通消息进入主会话
       └─ 如果主会话自身也超过阈值，走标准的会话级压缩流程（与是否用过 subagent 无关）
```

三层控制的分工非常清晰：**子 Agent 内部的膨胀由子 Agent 自己的压缩机制兜底；子到主的传递路径由摘要化+硬截断兜底；主 Agent 整体的膨胀由主会话自身的压缩机制兜底**——三层互不重叠、各管一段，任何一层出问题不会波及另外两层。

---

## 8. 权限与安全边界

上下文隔离的同时，Pi 也在权限维度做了对应的隔离/限制,这与上下文管理紧密相关（决定了子 Agent"能看到/能做什么"的边界）：

- **工具集限定**：每个 Agent 定义可以限制 `tools:`（如 `planner` 只有只读工具），子进程通过 `--tools` 参数强制生效，即使子 Agent 的系统提示词试图诱导它用别的工具，这些工具在这个子进程里根本不存在
- **项目级 Agent 的信任检查**：默认只加载用户级 Agent（`~/.pi/agent/agents`）；要启用项目级 Agent（`.pi/agents`，仓库可控）需要显式传 `agentScope: "both"/"project"`，且在不信任的项目里交互式运行时会弹出确认——因为项目级 Agent 定义本质上是"仓库作者写的、会被自动执行的系统提示词"，如果不加确认直接信任，等同于让任意仓库都能定义能读写文件/跑命令的自定义 Agent
- **cwd 可覆盖**：`tasks`/`chain` 里每项都可以指定独立的 `cwd`，子进程可以在不同目录下运行——配合权限限制，可以让"侦察型"子 Agent 只在只读挂载的目录里跑

这些权限边界与上下文隔离共同构成"子 Agent 是一个真正独立、能力受限的执行单元"，而不只是"上下文层面隔离但权限上还是主 Agent"的半隔离。

---

## 9. 与单 Agent 内部上下文管理机制的对比

理解多 Agent 协作的上下文模型，最好和《Pi Agent 架构》一文里介绍的单 Agent 上下文管理机制对照来看，两者解决的问题类似（防止上下文膨胀），但手段完全不同：

| 维度 | 单 Agent 内部（Compaction） | 多 Agent 协作（Subagent） |
|---|---|---|
| 隔离手段 | 同一进程、同一 `AgentSession`，靠算法性摘要压缩历史 | 独立操作系统进程，靠进程边界物理隔离 |
| 触发条件 | Token 超过 `contextWindow - reserveTokens` 阈值自动触发 | 每次工具调用天然触发（隔离是常态，不是应对膨胀的补救） |
| 保留内容 | LLM 生成的结构化摘要（Goal/Progress/Key Decisions/...）+ 保留最近若干条原始消息 | 只有子 Agent 自己产出的最终文本，中间过程完全不可见 |
| 可逆性 | 摘要仍在会话树里，`/tree` 理论上可以看到压缩前的完整历史 | 不可逆——`--no-session` 意味着子 Agent 的完整过程用完即销毁，无法追溯（除非查看本地 details 的临时渲染） |
| 适用场景 | 长时间持续对话，历史确实相关但太长 | 探索/子任务过程本身对主线程无用，只有结论有用 |

两种机制并非互斥，而是**分层应对不同粒度的问题**：压缩解决"同一件事聊太久"，Subagent 解决"这件事的调查过程本不该出现在主线程里"。一个复杂任务的完整生命周期里，两者会同时发生——主会话可能因为多轮对话触发过几次压缩，同时其中某一轮又委派了好几个 Subagent 去做探索，两套机制各自独立运作，互不干扰。

---

## 10. 设计小结

1. **多 Agent 协作 = 扩展系统 + 进程边界组合出的能力**，`pi-agent-core` 核心引擎对此毫无感知——从核心的角度看，`subagent` 只是一个普通工具，调用它和调用 `bash` 没有本质区别。这再次印证了"核心极简、能力靠组合"的架构原则。
2. **上下文隔离靠进程边界"免费"获得，而不是靠精细的应用层设计**——不共享内存、不共享会话文件、不共享系统提示词、不共享工具集，四个"不共享"叠加起来就是完整的隔离，不需要在 `AgentSession` 内部实现任何"多租户上下文"之类的复杂机制。
3. **信息传递被刻意约束为窄通道**（任务文本入、最终文本出、`{previous}` 占位符），逼迫每一层的 Agent 自己对信息做提炎——这是"上下文工程"在多 Agent 场景下的具体实践：不是等上下文膨胀了再压缩，而是从源头上不让无关信息有机会流入。
4. **权限限制与上下文隔离协同设计**：工具集限定、项目 Agent 信任确认，保证隔离不仅停留在"看不到"的层面，也落实到"做不了"的层面。
