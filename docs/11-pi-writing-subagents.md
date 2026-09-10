# Pi Subagent 编写指南：从编码型 Agent 到架构设计型 Agent

> 系列第 11 篇（[系列索引](README.md)）。第 03 篇讲的是 `subagent` 扩展**怎么跑起来**——进程隔离、Single/Parallel/Chain 三种模式、三条窄信息通道；本文讲的是**怎么写出好用的 subagent**：Agent 定义文件的每一段该写什么、工具与模型怎么选、任务文本怎么写、工作流模板怎么编排、怎么单独调试一个子 Agent。
>
> 文章分三部分：第一部分是编写模型（对所有 subagent 通用）；第二部分是**编码类 subagent**（侦察/规划/实现/评审，以及一个可复用的自定义实现型 Agent）；第三部分是**架构设计类 subagent**——这类 Agent 的产物是"决策 + 理由 + 权衡"而不是 diff，无法用测试判定对错，因此提示词结构、证据约束和编排方式都和编码类不同。
>
> 本文所有行为描述都基于 `packages/coding-agent/examples/extensions/subagent/`（`index.ts` / `agents.ts` / `agents/*.md` / `prompts/*.md`）与 `packages/coding-agent/src/core/`（`prompt-templates.ts`、`resource-loader.ts`、`package-manager.ts`）的源码核实。

## 目录

**第一部分：编写模型**
1. [写一个 subagent 到底在写什么](#1-写一个-subagent-到底在写什么)
2. [Agent 定义文件与 frontmatter 四个字段](#2-agent-定义文件与-frontmatter-四个字段)
3. [系统提示词正文的六段式结构](#3-系统提示词正文的六段式结构)
4. [工具白名单的语义与选择矩阵](#4-工具白名单的语义与选择矩阵)
5. [模型与思考等级的继承规则](#5-模型与思考等级的继承规则)

**第二部分：编码类 subagent**
6. [官方四个编码 Agent 的结构拆解](#6-官方四个编码-agent-的结构拆解)
7. [从零写一个编码 Agent](#7-从零写一个编码-agent)
8. [任务文本、工作流模板与编排](#8-任务文本工作流模板与编排)
9. [调试：单独跑一次子 Agent](#9-调试单独跑一次子-agent)
10. [编码类 Agent 的常见失败模式](#10-编码类-agent-的常见失败模式)

**第三部分：架构设计类 subagent**
11. [架构设计类任务与编码类任务的差异](#11-架构设计类任务与编码类任务的差异)
12. [architect Agent 的证据约束与输出契约](#12-architect-agent-的证据约束与输出契约)
13. [编排：并行出候选，收敛成一份 ADR](#13-编排并行出候选收敛成一份-adr)
14. [完整示例：architect 与 adr-critic](#14-完整示例architect-与-adr-critic)
15. [验收清单与抽查方法](#15-验收清单与抽查方法)
16. [什么时候不该用 subagent 做架构设计](#16-什么时候不该用-subagent-做架构设计)

**收尾**
17. [分发与团队共享](#17-分发与团队共享)
18. [小结](#18-小结)

---

# 第一部分：编写模型

## 1. 写一个 subagent 到底在写什么

先破除一个误解：`subagent` 不是一个 API，也不是框架里的类。**写 subagent 就是写 Markdown 文件**——一个带 YAML frontmatter 的提示词文件，扩展在运行时发现它、把 frontmatter 翻译成子进程的命令行参数、把正文当作追加系统提示词。

一次委派实际发生的事（`index.ts` 的 `runSingleAgent()`）：

```
发现     discoverAgents(cwd, scope) 扫 ~/.pi/agent/agents/*.md（+ 可选的 .pi/agents/*.md）
翻译     frontmatter.tools  → --tools
        frontmatter.model   → --model
        body                → 写入临时文件 → --append-system-prompt <file>
拉起     spawn(pi, ["--mode","json","-p","--no-session", ...args, "Task: <task>"])
回收     解析 stdout 的 JSON 事件流，取最后一条 assistant 文本作为工具结果
```

所以"编写 subagent"的产出物只有三类：

| 产出物 | 位置 | 作用 |
|---|---|---|
| **Agent 定义** | `~/.pi/agent/agents/*.md`（用户级）<br>`.pi/agents/*.md`（项目级） | 一个专职角色：角色边界、工具、模型、输出契约 |
| **工作流模板** | `~/.pi/agent/prompts/*.md`（用户级）<br>`.pi/prompts/*.md`（项目级） | 把多个 Agent 编排成斜杠命令，如 `/implement` |
| **任务文本** | 运行时由主 Agent 填写的 `task` 参数 | 每次委派的唯一输入 |

第三类最容易被忽略，但它是决定成败的关键：子 Agent 能拿到的输入**只有** `Task: <task>` 这一段文本（外加它自己的系统提示词）。主 Agent 必须在发起委派前把子 Agent 需要的一切写进去。这是刻意的设计约束——迫使调用方想清楚"这个子任务真正需要哪些信息"，而不是靠隐式上下文继承。

## 2. Agent 定义文件与 frontmatter 四个字段

一个最小可用的 Agent 定义：

```markdown
---
name: my-agent
description: What this agent does, and when to use it
tools: read, grep, find, ls
model: claude-sonnet-4-5
---

System prompt body goes here.
```

`agents.ts` 的 `loadAgentsFromDir()` 只读取四个字段，其余一律忽略：

| 字段 | 必需 | 语义 | 解析细节 |
|---|---|---|---|
| `name` | **是** | 委派时用的标识（`agent: "scout"`） | 必须是字符串。同名时项目级覆盖用户级（仅在 `agentScope: "both"` 下） |
| `description` | **是** | 给主 Agent 看的用途说明 | 会随 Agent 列表一起注入工具描述，决定主 Agent 会不会选它 |
| `tools` | 否 | 工具白名单 | 字符串 `"read, grep, find, ls"` 或 YAML 数组 `[read, grep]` 都接受；解析不出有效列表时**等价于不限制**，不报错 |
| `model` | 否 | 指定模型 | 省略 = 继承主会话当前模型（见第 5 节） |

三个容易踩的点：

1. **缺 `name` 或 `description` 的文件会被静默跳过**，不会报错也不会出现在 Agent 列表里。写完发现"调用时说 unknown agent"，第一件事是检查这两个字段的拼写和类型（YAML 里 `name: 123` 是数字，同样不合法）。
2. **多余字段不是错误，只是无效**。常见误解是以为能写 `thinking: high`、`temperature: 0` 之类——frontmatter 没有这些键，写了会被忽略，模型行为不会有任何变化。
3. **Agent 是每次调用时重新发现的**（`discoverAgents()` 在 `execute()` 内部调用），改完 `.md` 文件下次调用就生效，不需要重启 Pi 或 `/reload`。

作用域与安全：`agentScope` 默认 `"user"`，只加载 `~/.pi/agent/agents`。项目级 `.pi/agents` 需要显式传 `agentScope: "project"` 或 `"both"`，且沿目录树向上找最近的一个 `.pi/agents` 目录。项目级 Agent 本质是"仓库作者写的、会被自动执行的系统提示词"，所以在未信任的项目里交互式运行会弹确认（`confirmProjectAgents`，默认 true）；把它写进工作流模板时要想清楚——模板是用户自己写的，等于预先同意。

## 3. 系统提示词正文的六段式结构

正文通过 `--append-system-prompt` **追加**到 Pi 的默认系统提示词之后——不是替换。这意味着子 Agent 仍然知道自己是 Pi、仍然知道工具怎么用、仍然会看到项目的 `AGENTS.md`（除非传 `--no-context-files`）。你的正文是"在上层叠加角色约束"，所以**不要重复基础约定，也不要和它冲突**。

把 `agents/*.md` 拆开看，四个官方 Agent 的正文都由六段组成：

| 段 | 作用 | 缺失后果 |
|---|---|---|
| **1. 角色与边界** | 一句话说清"你是谁、不做什么" | 子 Agent 越界（planner 直接改代码） |
| **2. 输入契约** | 你会收到什么、来自谁 | 收到上游文本后不知道怎么解析 |
| **3. 工作策略** | 有序步骤 + 停止条件 | 漫无目的探索，烧 token 后给个敷衍结论 |
| **4. 输出契约** | 固定的 Markdown 标题与字段 | 输出格式漂移，下游 Agent 无法稳定消费 |
| **5. 交接约定** | 下游是谁、必须带什么信息 | chain 里后一步拿不到关键事实 |
| **6. 硬约束** | 禁用的命令/文件/行为 | 只读 Agent 顺手改了代码 |

其中 **4 是硬性要求，不是建议**。因为 `getFinalOutput()` 只返回最后一条 assistant 消息的文本，输出格式就是 Agent 与外部世界唯一的接口。看 `scout.md` 的处理：

```markdown
Output format:

## Files Retrieved
List with exact line ranges:
1. `path/to/file.ts` (lines 10-50) - Description of what's here
```

它不仅规定了标题，还给了**字面示例**（`lines 10-50`）。这比"请列出文件路径"有效得多——模型会照着示例的格式填充。写输出契约时务必给示例。

`planner.md` 里还有一句容易被忽略但很关键的收尾：

```markdown
Keep the plan concrete. The worker agent will execute it verbatim.
```

它把输出契约和下游消费方（`worker`）显式绑定：计划的粒度直接决定下游能不能执行。这是第 5 段"交接约定"的极简写法。

## 4. 工具白名单的语义与选择矩阵

`--tools` 是**白名单**，而且作用于全部工具——内置工具、扩展工具、自定义工具都受它约束（见 `--tools` 的帮助文本："Applies to built-in, extension, and custom tools"）。

这带来两个必须知道的推论：

- **不写 `tools` ≠ 拥有全部 8 个工具**。默认激活的内置工具是 `read, bash, edit, write`（`defaultActiveToolNames`，可被 settings 的 `defaultTools` 覆盖）。官方 `worker` 没写 `tools`，它拿到的是这四个 + 全部扩展工具，而**不是** `grep/find/ls`。要用 `grep` 就必须显式声明。
- **写 `tools` 会顺手禁掉扩展工具**，包括 `subagent` 自己。想让子 Agent 具备嵌套委派能力，必须把 `subagent` 写进白名单；想让只读 Agent 保持纯粹，不写就是了。

选择矩阵：

| Agent 类型 | 推荐白名单 | 理由 |
|---|---|---|
| 侦察型（scout） | `read, grep, find, ls, bash` | 需要 bash 跑 `rg`/`git log` 等只读命令；不给 `edit`/`write` |
| 规划型（planner） | `read, grep, find, ls` | 完全不给 bash，从机制上杜绝"顺手执行" |
| 评审型（reviewer） | `read, grep, find, ls, bash` | 需要 `git diff`/`git log`/`git show` |
| 实现型（worker/fixer） | `read, bash, edit, write, grep, find, ls` | 实现 + 需要跑测试验证 |
| 架构设计型（architect） | `read, grep, find, ls, bash` | 只读；bash 限定为 `git log`/`git show`/`rg` |
| 落盘型（adr-writer） | `read, write, ls` | 只允许产出新文件，不允许改既有代码 |

工具白名单是**机制层面的强制**，比提示词里的"不要修改文件"可靠得多——这也正是 `reviewer.md` 里那句略显矛盾的话的由来：

```markdown
Bash is for read-only commands only: `git diff`, `git log`, `git show`. Do NOT modify files or run builds.
Assume tool permissions are not perfectly enforceable; keep all bash usage strictly read-only.
```

给了 bash 就等于给了写文件的能力（提示词拦不住 `bash` 里的 `rm`）。所以原则很简单：**能不给 bash 就不给；给了 bash 就在提示词里枚举允许的命令**，双保险。

## 5. 模型与思考等级的继承规则

`runSingleAgent()` 里这两行决定了继承行为：

```typescript
const inheritsDispatchConfig = !agent.model;
const model = agent.model ?? dispatchDefaults.model;
if (model) args.push("--model", model);
if (inheritsDispatchConfig && dispatchDefaults.thinkingLevel) {
  args.push("--thinking", dispatchDefaults.thinkingLevel);
}
```

规则用一句话说：**`model` 省略时，子 Agent 继承主会话的模型和思考等级；`model` 一旦写了，思考等级就不再继承**（子进程会用该模型的默认思考等级）。

推论：

- 想让一组 Agent 全部跟随用户在主会话里选的模型 → 都不写 `model`。这是"行为符合直觉"的默认路径：主会话切到 Opus，子 Agent 也跟着上 Opus。
- 想做成本优化（侦察用便宜快模型、实现用强模型）→ 显式写 `model`，但要接受"这个 Agent 的思考等级不再跟随主会话"。
- **frontmatter 不支持 `thinking` 字段**。需要高思考等级时只有两条路：省略 `model` 让它继承主会话等级，或在正文里要求"先列证据再给结论"，用推理步骤的显式化来弥补（第三部分的 architect 就靠这一招）。

---

# 第二部分：编码类 subagent

## 6. 官方四个编码 Agent 的结构拆解

把四个官方 Agent 按六段式对齐，规律非常清晰：

| Agent | 角色与边界 | 工作策略 | 输出契约 | 模型/工具 |
|---|---|---|---|---|
| `scout` | 快速侦察，返回**压缩后的**上下文 | grep/find 定位 → 读关键片段（不是整个文件）→ 识别类型与依赖 | `## Files Retrieved`（带行号）/ `## Key Code`（贴真实代码）/ `## Architecture` / `## Start Here` | Haiku + `read,grep,find,ls,bash` |
| `planner` | 只规划，禁止改动 | 消费 scout 的上下文 → 拆成可执行小步骤 | `## Goal` / `## Plan`（编号步骤，指向具体文件函数）/ `## Files to Modify` / `## New Files` / `## Risks` | Sonnet + 只读四件套 |
| `worker` | 通用实现，全能力 | 自主完成，不设固定策略（任务差异太大） | `## Completed` / `## Files Changed` / `## Notes` + 交接时补"改动路径 + 触及的关键函数" | Sonnet + 默认工具 |
| `reviewer` | 只读评审，bash 仅限 `git diff/log/show` | `git diff` → 读改动文件 → 查 bug/安全/坏味道 | `## Files Reviewed` / `## Critical` / `## Warnings` / `## Suggestions` / `## Summary`（都要求带 file:line） | Sonnet + `read,grep,find,ls,bash` |

三点值得抄的设计：

1. **`scout` 的职责被明确定义成"为他人压缩上下文"**，而不是"回答问题"。它的提示词第一句就声明"你的输出会交给一个没看过这些文件的 Agent"——这让模型明白：贴真实代码片段比描述代码更有价值，因为下游不会重新读文件。这是 chain 模式能work的前提。
2. **`scout` 有明确的彻底性档位**（Quick / Medium / Thorough，默认 medium），并说明"从任务推断"。这避免了两个极端：简单任务过度探索烧 token，复杂任务浅尝辄止。
3. **`reviewer` 的分级输出**（Critical / Warnings / Suggestions）让下游 `worker` 知道先修什么。`/implement-and-review` 的第三步就是让 worker 按这个分级去改。

## 7. 从零写一个编码 Agent

下面这个 `test-fixer` 是一个可直接使用的实现型 Agent，覆盖了六段式，并加了编码类 Agent 最需要的"验证闭环"和"失败上报"：

```markdown
---
name: test-fixer
description: Fixes failing tests with minimal, verified diffs; reports instead of guessing when blocked
tools: read, bash, edit, write, grep, find, ls
model: claude-sonnet-4-5
---

You are a test-fixing agent. You run in an isolated context and the caller only
receives your final message, so that message must be complete and self-contained.

## Scope
Fix the failing tests you are given. Do NOT refactor, rename, or touch unrelated
files. If a correct fix requires an API change or a product decision, stop and
report it instead of inventing one.

## Input you will receive
- The command that reproduces the failure (run it exactly as given)
- Optionally: failing test names, error output, and the files involved

## Strategy
1. Reproduce: run the given command. Read the actual error before changing anything.
2. Localize: read only the code on the failing path. Follow imports at most 2 hops.
3. Fix the root cause, not the symptom.
4. Re-run the command. If it still fails, retry at most twice, then report failure.
5. Run the surrounding test file (not the whole suite) to catch regressions.

## Hard constraints
- Never weaken or delete assertions to make a test pass, unless the task says so.
- bash is for running tests and read-only inspection. No `git commit`, no `rm`.
- Prefer the smallest diff that fixes the root cause.

## Output format

## Root Cause
One or two sentences: what was broken and why.

## Fix
What changed, and why this is the smallest correct change.

## Files Changed
- `path/to/file.ts:42` - what changed

## Verification
The exact command you ran and its result.

## Not Fixed (if any)
What is still broken, why, and what you need from the caller.
```

和 `worker` 相比它多了三样东西，这正是"专用 Agent 比通用 worker 更好用"的原因：

- **复现步骤写进策略第 1 步**：没有复现就改代码是编码 Agent 最常见的失败模式。
- **重试上限**（最多两次）：防止子 Agent 在错误方向上无限循环烧 token——子 Agent 的上下文是隔离的，它"卡住"时主 Agent 完全看不见，只能等它跑完。
- **`## Not Fixed` 段**：给失败一个结构化的出口。没有这个出口，模型倾向于"编造一个看起来成功的结论"来终止循环。

再补一条编码类的通用经验：**写操作不要并行**。Parallel 模式下的任务互相完全隔离（`scout` A 不知道 `scout` B 在读什么），这对只读侦察是优点，对改代码是灾难——两个 `test-fixer` 并行改同一批文件会互相覆盖。需要改多个独立模块时，要么在任务文本里划清"你只负责 `packages/a/`"，要么改用 chain 串行。

## 8. 任务文本、工作流模板与编排

### 8.1 任务文本的写法

因为 task 是唯一输入通道，主 Agent（也就是你写的模板/提示词）应该按五要素填：

```
目标        Fix the failing tests in packages/ai/test/streaming.test.ts
已知事实    Failing: "aborts mid-stream"; command: npm test -- streaming
范围边界    Only touch packages/ai/src/stream.ts; do not edit the test file
验收标准    npm test -- streaming passes
输出要求    Report root cause, files changed, and the verification command output
```

缺"范围边界"和"验收标准"是委派质量差的主因：前者导致子 Agent 扩大改动面，后者导致它无法自我判断完成。

### 8.2 工作流模板

模板是 `prompts/` 下的 Markdown，文件名（去掉 `.md`）就是斜杠命令名：

```markdown
---
description: Worker implements, reviewer reviews, worker applies feedback
---
Use the subagent tool with the chain parameter to execute this workflow:

1. First, use the "worker" agent to implement: $@
2. Then, use the "reviewer" agent to review the implementation from the previous step (use {previous} placeholder)
3. Finally, use the "worker" agent to apply the feedback from the review (use {previous} placeholder)

Execute this as a chain, passing output between steps via {previous}.
```

frontmatter 支持 `description`（命令列表里显示）和 `argument-hint`（补全提示）。变量的替换由 `substituteArgs()` 完成（bash 风格）：

| 写法 | 含义 |
|---|---|
| `$@` / `$ARGUMENTS` | 全部参数拼成一个字符串 |
| `$1` `$2` | 位置参数 |
| `${@:-默认值}` | 全部参数为空时的默认值 |
| `${@:2}` / `${@:2:3}` | 从第 N 个参数开始取（可指定长度） |

注意 `{previous}` 不是模板变量——它由 subagent 工具的 chain 模式在运行时替换（`step.task.replace(/\{previous\}/g, previousOutput)`），模板展开阶段不碰它。

### 8.3 三种模式怎么选

| 场景 | 模式 | 要点 |
|---|---|---|
| 单一明确子任务 | Single `{ agent, task }` | 简单直接 |
| 多个互不依赖的**只读**探索 | Parallel `{ tasks: [...] }` | 最多 8 个任务、4 并发；每任务输出 50KB 截断，汇总成一条工具结果 |
| 有依赖的流水线 | Chain `{ chain: [...] }` | 步间只传一段文本；任一步失败即停 |
| 改代码的多步流程 | Chain（不要 Parallel） | 见第 7 节末尾 |

Parallel 的两个实用提醒：单任务失败不影响其他任务（汇总里标 `failed`），所以"3 个侦察任务有 1 个失败"仍然能拿到另外 2 个的结果；而 Chain 是"失败即停"，设计链条时要保证每一步都能独立失败而不留下半成品状态。

## 9. 调试：单独跑一次子 Agent

subagent 的黑盒感来自"子 Agent 的过程看不见"。调试手段就是把子 Agent 单独跑一遍，看它完整的 JSON 事件流：

```bash
pi --mode json -p --no-session \
   --tools read,grep,find,ls \
   --append-system-prompt ~/.pi/agent/agents/scout.md \
   "Task: 找出所有与 OAuth 回调处理相关的代码"
```

几点说明：

- 这个命令就是扩展实际构造的命令行（`index.ts` 的 `runSingleAgent()`），只是把 `--append-system-prompt <file>` 指向了原始 `.md` 文件。**唯一差异**：扩展传的是 `parseFrontmatter()` 剥掉 frontmatter 后的正文，手工跑传的是含 frontmatter 的整个文件——少量噪声，不影响调试。
- 关注两类事件：`message_end`（每轮 assistant 消息，带 `usage`/`stopReason`）和 `tool_result_end`（工具结果）。这就是父进程收集的全部信息，也是 TUI 展开视图显示的内容。
- 看 `usage` 里的 `turns` 和 `contextTokens`：turns 异常高说明策略段没给它停止条件；contextTokens 逼近窗口说明任务粒度太大。
- 想确认子 Agent 到底看到了什么系统提示词，加 `--system-prompt` 打印或临时把正文写到临时文件用 `--append-system-prompt` 逐个替换来对比。

调试清单（按排查成本从低到高）：

1. 单独跑命令，看它是否按输出契约产出 → 修正文
2. 看 turns / contextTokens → 修策略或拆任务
3. 看它调用了哪些工具 → 修 `tools` 白名单
4. 看它是不是复读了同一批文件 → 修任务文本（补"已知事实"）
5. 确认 Agent 被发现了：调用时若返回 `Unknown agent: "x". Available agents: ...`，说明 `name` 字段或文件位置不对，或项目级 Agent 没开 `agentScope`

## 10. 编码类 Agent 的常见失败模式

| 症状 | 根因 | 修法 |
|---|---|---|
| `Unknown agent: "x"` | `name` 缺失/类型不对；文件不在扫描目录；项目级未开 `agentScope` | 检查 frontmatter 两个必需字段；确认路径 |
| 子 Agent 反复读同一批文件 | 任务文本没给起点 | 任务里写"从 `path/file.ts` 开始，重点看 A/B/C" |
| 输出格式每次都不一样 | 输出契约太抽象 | 写死标题 + 给字面示例；用"必须包含以下小节" |
| 只读 Agent 改了文件 | 白名单里给了 `write`/`edit`，或给了 `bash` | 只读 Agent 只给 `read,grep,find,ls`；必须给 bash 时在正文枚举允许命令 |
| 子 Agent 卡在循环里烧 token | 没有重试上限和失败出口 | 加"最多重试 N 次"和 `## Not Fixed` 段 |
| 委派后主 Agent 还是不知道细节 | 只有最后一条文本会返回（details 只用于 TUI） | 让 Agent 在最终输出里复述关键事实 |
| 并行任务互相覆盖改动 | 并行用于写操作 | 改操作串行；或按目录严格分区 |
| 单个委派成本过高 | 子 Agent 自己触发了压缩 | 拆任务；收紧 tools 减少无效探索；给 `Start Here` |
| 换了模型后思考深度不对 | 写了 `model` 就不再继承 thinking | 需要跟随主会话就省略 `model` |
| 长输出被截断（50KB） | Parallel 每任务硬截断 | 让 Agent 输出更精炼的摘要；完整内容在 TUI 展开视图里 |

---

# 第三部分：架构设计类 subagent

## 11. 架构设计类任务与编码类任务的差异

"写一个做架构设计的 subagent"听起来只是换个角色描述，实际上两类任务的失败模式完全不同：

| 维度 | 编码类 | 架构设计类 |
|---|---|---|
| 产物 | diff / 文件改动 | 决策 + 理由 + 被否方案 + 验证计划 |
| 正确性判定 | 机器可判（测试/构建/lint） | 无自动判据，只能靠证据核查与人评审 |
| 主要风险 | 改坏代码、改动面失控 | **幻觉**：编造不存在的模块、接口、约束 |
| 上下文需求 | 局部（相关文件） | 全局（入口、依赖、既有约定、迁移成本） |
| 自我验证能力 | 强（跑测试就知道） | 弱（没有"跑一下就知道"的闭环） |
| 工具 | 常需要 `edit`/`write` | 只读为主，最多落一个 `.md` |
| 编排偏好 | 链式（侦察→规划→实现→评审） | 扇出/扇入（多方案并行→挑错→收敛） |

最关键的一条是**幻觉风险**。编码 Agent 幻觉了会写出编译不过的代码，测试会拦住它；架构 Agent 幻觉了会写出一份读起来非常专业、引用了 `EventBus.publish()` 这样的方法、但仓库里根本没有这个方法的方案——没有任何自动检查能拦住它，而读者会当真。

所以架构设计类 Agent 的提示词设计要围绕一件事：**把"断言"变成"带引用的断言"，把"引用不了的断言"显式标注出来**。同时，因为子 Agent 的上下文是隔离的，它无法像人一样"边读边在脑子里积累全局图"，所以证据收集必须前置成独立步骤。

## 12. architect Agent 的证据约束与输出契约

三条硬规则（对应上面两个风险）：

**规则一：每条事实性论断必须带 `path:line` 引用。** 提示词里要写成不可协商的形式，并要求引用能被复查。

**规则二：无法验证的内容必须显式标注 `UNVERIFIED:`。** 这条比规则一更重要——模型在"必须给引用"的压力下会编造引用。给出合法的"我不知道"出口，编造率会显著下降。

**规则三：事实与推断分离输出。** 让 `## Evidence` 和 `## Inference` 成为不同的小节，读者（和下游 critic）能一眼区分"仓库里确实有的"和"Agent 认为的"。

输出契约用 ADR（Architecture Decision Record）结构。选它的理由不是形式主义，而是它天然包含架构决策最容易被跳过的三件事：**被否方案**、**后果的负面部分**、**验证计划**。普通"设计方案"文档最常见的缺陷就是只写"我们决定用 X"，不写"为什么不用 Y"和"怎么知道 X 是对的"。

另外一条容易被忽略的约束：**要求至少两个本质不同的方案**。模型有强烈的锚定倾向——第一个想到的方案会被不断合理化。强制写两个，能逼出真正的权衡。

最后，因为 frontmatter 不能设思考等级，架构 Agent 要用**推理步骤显式化**来弥补：在输出契约里要求"先列 `## Evidence`，再给 `## Decision`"。模型必须先写证据再写结论，结论质量明显高于直接给结论。

## 13. 编排：并行出候选，收敛成一份 ADR

架构设计用 Parallel + Chain 组合，而不是纯 Chain：

```
第 1 步  Parallel：3 个 architect，同一个问题，三个不同的强制视角
         ├─ 最小改动派：复用现有模块
         ├─ 干净重做派：先不考虑迁移成本
         └─ 运维 boring 派：最少的新增活动部件
         （三个进程上下文完全隔离 → 互不锚定，这是并行在这里的真正价值）

第 2 步  Single：adr-critic 拿第 1 步的汇总做对抗性评审
         （复查引用是否真实、攻击每个方案的未考虑失败模式、检查验证计划可执行性）

第 3 步  Single：architect 拿第 2 步的评审，收敛成一份 ADR
         （保留存活部分，记录被否方案与理由）
```

这里有两个必须理解的信息流约束：

- **Parallel 的输出是一份拼接后的汇总文本**（每任务截断到 50KB），不是三条独立消息。所以第 2 步的 critic 收到的是"三个方案 + 三个方案的评审请求"拼在一起的文本——这正是我们要的：critic 能横向比较。
- **Chain 的每一步只拿到上一步的一段文本**。所以 critic 的输出必须**保留每个方案的要点和缺陷**，否则第 3 步的 architect 看不到任何方案内容，只能凭评审意见重新编方案。这是 chain 编排最容易犯的错：中间步骤忘了当"信息搬运工"。

收敛这一步也可以不让 Agent 做：让 critic 的输出直接回到主 Agent，由主 Agent（也就是和你对话的那个会话）做裁决。好处是你能看到三个方案和挑错过程，自己拍板；坏处是三份全文进主上下文。方案简单时用主 Agent 裁决，方案复杂时用 chain 收敛。

**留痕问题**：子 Agent 用 `--no-session` 跑，过程不落盘、不进会话树，跑完即销毁。对编码任务无所谓（产物是代码改动），对架构任务是个问题——决策过程本身是资产。解法是加一步落盘 Agent（`adr-writer`，只给 `read, write, ls`），把收敛后的 ADR 写成 `docs/adr/NNNN-xxx.md`。这也是"工具白名单"的典型用法：允许它产出新文件，但不允许它改既有代码。

## 14. 完整示例：architect 与 adr-critic

### `~/.pi/agent/agents/architect.md`

```markdown
---
name: architect
description: Produces evidence-grounded ADR drafts with rejected alternatives; analysis only, never modifies code
tools: read, grep, find, ls, bash
model: claude-sonnet-4-5
---

You are a software architect. You produce decision records, not code changes.
You run in an isolated context and the caller only receives your final message,
so that message must be complete and self-contained.

## Evidence rules (non-negotiable)
- Every factual claim about this codebase MUST cite `path/to/file.ts:123`.
- A claim you could not verify MUST be written as `UNVERIFIED: <claim>` and
  repeated under the `## UNVERIFIED` section. Do not invent module names,
  exported symbols, config keys, or performance numbers.
- Keep `## Evidence` (what you actually read) separate from `## Inference`
  (what you concluded from it).
- If you have not read the entry point yet, do not propose an interface.

## Method
1. Locate the entry point and the boundary of the change.
2. Map constraints: existing conventions (AGENTS.md, docs/adr/*), dependency
   direction, data flow, and anything that makes a migration expensive.
3. Write at least two materially different options. Do not present your first
   idea as the answer.
4. Choose one, and state what evidence would make you change the decision.

## Hard constraints
- Read-only. bash is limited to `git log`, `git show`, `rg`, `wc`, `ls`.
  No edits, no artifact-producing builds, no `git commit`.
- No code beyond short interface sketches (under 15 lines).
- If the request is underspecified, list blocking questions instead of
  inventing requirements.

## Output format

# ADR-NNNN: <decision title>
- Status: proposed
- Scope: <what is in, what is out>

## Context
Forces at play, each with a citation.

## Options Considered
### Option A: <name>
- Shape: ...
- Pros: ... / Cons: ...
- Migration cost: ...
### Option B: <name>
(same shape, must be materially different from A)

## Decision
Option X, in 3-5 sentences, leading with the single most important reason.

## Consequences
- Positive: ...
- Accepted costs: ...

## Migration & Rollback
Ordered steps. If step K fails, how to revert to step K-1.

## Verification Plan
How to prove this decision was right within one iteration:
concrete commands, metrics, or checks.

## Evidence Index
- `path/file.ts:10-40` - what it shows

## UNVERIFIED
- Claims you could not ground, each with how to check them.

## Open Questions
What you need from a human before this can be adopted.
```

### `~/.pi/agent/agents/adr-critic.md`

```markdown
---
name: adr-critic
description: Adversarial reviewer for architecture proposals; verifies citations and attacks feasibility
tools: read, grep, find, ls, bash
model: claude-sonnet-4-5
---

You are an adversarial architecture reviewer. Your job is to break proposals,
not to improve them politely. You receive one or more candidate ADRs.

## Method
1. Re-verify at least 5 citations by reading the cited lines. Any citation that
   does not support its claim is reported as WRONG CITATION with the real content.
2. For each option, find the failure mode the author did not consider: scale,
   ordering, partial failure, migration lock-in, operational burden, team cost.
3. Check the proposal against existing conventions: AGENTS.md, docs/adr/*.md.
4. Check whether the Verification Plan is actually executable in this repo.

## Hard constraints
- Read-only. Never edit files.
- Do not rewrite the proposal. Produce findings only.
- Every finding needs a `file:line` citation or a concrete named scenario.
- You are the information carrier for the next step: your output MUST preserve
  each option's name, shape, and your verdict on it, so a later agent can
  converge on it without seeing the originals.

## Output format

## Option Verdicts
- [Option A] keep / reject - one-line reason, with `file:line` if evidence-backed

## Fatal Flaws
- [Option X] <flaw> - why it blocks adoption

## Citation Errors
- "<quoted claim>" - `path:line` does not contain it (actual content: ...)

## Missed Constraints
- <constraint found in the codebase> - `path:line`

## Verification Gaps
- The plan says X, but no command or metric in this repo measures X

## Weakest Assumption
The single assumption most likely to be wrong, and the cheapest way to test it.
```

注意 `adr-critic` 里那条"你是下一步的信息载体"——这是第 13 节讲的中间步骤信息保真要求，直接写进提示词。

### `~/.pi/agent/agents/adr-writer.md`（可选，落盘用）

```markdown
---
name: adr-writer
description: Writes a converged ADR to docs/adr/ as one new file; never modifies existing code
tools: read, write, ls
model: claude-sonnet-4-5
---

You persist an architecture decision record. The subagent that produced the
content ran with an ephemeral session, so this file is the only durable trace.

## Hard constraints
- Write exactly one new file: `docs/adr/NNNN-<slug>.md`, where NNNN is
  `max(existing number) + 1` (check with `ls docs/adr`). If the directory does
  not exist, create it.
- Never modify existing files. If the target path already exists, stop and report.
- Preserve the ADR content verbatim. Do not improve, summarize, or reformat it.

## Output format

## File Written
`docs/adr/NNNN-<slug>.md`

## Next Number Rationale
Existing ADRs found: ...

## Not Written (if any)
Why, and what to do about it.
```

### `~/.pi/agent/prompts/design.md`

```markdown
---
description: Architecture design workflow - parallel candidate ADRs, adversarial review, converged decision
argument-hint: <the design question or change you need a decision on>
---
Produce an architecture decision for: $@

Do NOT implement anything. Use the subagent tool:

1. First, run a PARALLEL batch of three "architect" agents with the same goal but
   three different mandated lenses, all grounded in this repository:
   - "Prefer the smallest change that satisfies: $@. Reuse existing modules."
   - "Prefer a clean-slate design for: $@. Ignore migration cost for now."
   - "Prefer the operationally boring option for: $@. Minimize new moving parts."
2. Then, run ONE "adr-critic" agent on the {previous} output: verify its citations
   against the real files and attack every option.
3. Then, run ONE "architect" agent on the {previous} output: produce a single
   converged ADR that keeps what survived review and records why the rest was
   rejected.

Return the converged ADR to me, and ask before writing any files.
```

最后一句"ask before writing any files"是刻意的：把落盘决定权留给你。`adr-writer` 是可选的第四步，需要时再说一句 "now persist it with adr-writer"。

## 15. 验收清单与抽查方法

架构设计类 Agent 的输出不能靠"读起来专业"来判断。最小验收清单：

| 检查项 | 不通过的样子 |
|---|---|
| **引用抽查**：随机挑 3 条 `path:line`，实际打开看 | 引用的文件不存在 / 行号处不是所说的内容 |
| **被否方案存在且具体** | 只有一个方案；或"方案 B"是明显稻草人 |
| **事实与推断分离** | 全篇都是断言，没有 `## UNVERIFIED` 段（等于说它全知） |
| **验证计划可执行** | "上线后观察稳定性"这类无法证伪的表述 |
| **考虑了回滚** | 只有正向迁移步骤，没有失败路径 |
| **与既有约定一致** | 和 `AGENTS.md` 或已有 ADR 冲突却没提到冲突 |
| **规模诚实** | 声称"小改动"但迁移步骤有 12 步 |

抽查引用这一步，正好可以让 `adr-critic` 或另一个 subagent 代劳——`adr-critic` 的"复查至少 5 条引用"就是为此设计的。人工评审时重点看它的 `## Citation Errors` 段是否为空。

如果想把这套做法固化成可重复的质量门禁，可以接第 09 篇的 evals 体系：把"给定 N 个设计问题，检查产出 ADR 的引用准确率"作为一个评估项，用 Judge 给"引用可核查性"打分，用硬断言（引用文件必须存在）做红线。

## 16. 什么时候不该用 subagent 做架构设计

subagent 的隔离性在架构任务上有一处明显的代价：**子 Agent 看不到主会话里已经积累的讨论**。你和主 Agent 聊了十轮需求细节、澄清了三个约束，一旦委派，这些全部丢失——只有写进 `task` 的才算数。所以：

- **需求还在演化时不要委派**。先把讨论收敛，再让 architect 基于确定的上下文产出 ADR。委派一个模糊问题只会得到一份结构上完整、内容上空洞的 ADR。
- **决策依赖仓库外的知识**（组织结构、排期、历史事故、团队偏好）时，subagent 帮不上忙——它只有代码库。这类信息必须写进任务文本，或者由人来补。
- **改动很小**（一个文件、一个函数级的选型）时不值得走完整流程，直接让主 Agent 回答更快。
- **需要跨仓库视野**时，单个子 Agent 的 cwd 是固定的（可在任务项里覆盖 `cwd`，但一次只有一个）。这种情况用 Parallel 给每个仓库派一个 architect，再收敛。

判断标准很简单：**如果委派时你写不出"已知事实 + 范围边界 + 验收标准"这三段任务文本，说明这件事还不适合委派。**

---

# 收尾

## 17. 分发与团队共享

**Pi Packages 不分发 agents 目录。** package manifest（`pi` 字段）只声明 `extensions` / `skills` / `prompts` / `themes` 四类资源，`agents` 不在自动发现范围内（见 `package-manager.ts` 的 `RESOURCE_TYPES`）。可行的分发方式：

1. **项目级目录 + 版本控制**：把 Agent 定义放在 `.pi/agents/*.md` 提交进仓库，团队共享。代价是调用时需要 `agentScope: "both"`，且这些文件是"仓库作者写的会被自动执行的系统提示词"——属于需要信任的资源，未信任项目会弹确认。命名上加团队前缀（如 `acme-architect`）避免和用户级 Agent 撞名（同名时项目级覆盖用户级）。
2. **随扩展分发**：把 Agent 定义放进你自己扩展的目录，在扩展里替换/扩展发现逻辑（官方 `discoverAgents()` 只认 `~/.pi/agent/agents` 和 `.pi/agents` 两个位置，要支持别的位置就得改 `agents.ts`）。
3. **安装说明 + 软链**：官方 README 采用的方式，适合内部小范围共享。

配套的两点：
- 工作流模板（`prompts/*.md`）**是**被 Pi Packages 支持的资源，所以"把 `/design` 这类编排分发出去"很顺畅，只有 Agent 定义本身需要额外处理。
- 项目级 `prompts/` 同样需要项目信任（`TRUST_REQUIRING_PROJECT_CONFIG_RESOURCES` 包含 `prompts`）。

## 18. 小结

1. **写 subagent = 写 Markdown**，不是写插件。frontmatter 只有 `name`/`description`/`tools`/`model` 四个字段被读取，正文是追加到（不是替换）Pi 默认系统提示词之上的角色约束。
2. **输出契约是硬要求**：子 Agent 与主 Agent 之间只有"任务文本进、最终文本出"两条通道，输出格式就是它唯一的对外接口——写死标题、给字面示例。
3. **工具白名单是机制级约束**，比提示词里的"不要修改文件"可靠。给了 `bash` 就等于给了写能力，所以要么不给，要么在正文里枚举允许的命令。
4. **编码类 Agent 的核心是验证闭环**：复现 → 定位 → 修复 → 复跑 → 结构化失败上报；写操作串行，只读探索才并行。
5. **架构设计类 Agent 的核心是把断言变成带引用的断言**，并用合法的 `UNVERIFIED:` 出口换取较低的编造率；用 ADR 结构强制写出被否方案与验证计划；用"并行出候选 + critic 挑错 + 收敛"代替线性链式。
6. **中间步骤要当信息载体**：chain 的每一步只拿到上一步的一段文本，收敛型 Agent 看不到更早的方案原文，所以挑错步骤必须保留方案要点。
7. **委派前先自问三件事**：已知事实写了吗、范围边界划了吗、验收标准给了吗——缺任何一个，说明这个任务还不该委派。
