# Pi Coding Agent 提示词体系：主干、注入链路与六个改写入口

> 系列第 12 篇（[系列索引](README.md)）。前面的文章讲架构（01）、外壳包内部组织（02）、多 Agent 协作（03）、长任务（04）、成本（05）……本文回到最具体的东西：**模型每一轮真正看到的那段 system prompt 是怎么拼出来的**，以及你能在哪些位置改它、每种改法会连带丢掉什么。
>
> 这个问题比看上去重要。Pi 的内置提示词很短（默认分支只有几十行），大部分"人格"和"规矩"来自你塞进去的项目上下文与追加段。改错位置的效果差异很大：有的改法只加一段话，有的改法会把工具说明和对 Pi 自身的文档指引一起抹掉。
>
> 本文所有行为描述都基于 `packages/coding-agent/src/core/` 的源码核实：`system-prompt.ts`、`resource-loader.ts`、`agent-session.ts`、`prompt-templates.ts`、`skills.ts`、`extensions/runner.ts`、`tools/*.ts`，以及官方 `packages/coding-agent/docs/prompt-templates.md`。

## 目录

1. [一段 prompt，五个来源](#1-一段-prompt五个来源)
2. [内置主干：默认分支逐段拆解](#2-内置主干默认分支逐段拆解)
3. [工具提示词贡献点](#3-工具提示词贡献点)
4. [项目上下文：AGENTS.md 的发现与注入](#4-项目上下文agentsmd-的发现与注入)
5. [Skills 段：只放索引，正文按需读](#5-skills-段只放索引正文按需读)
6. [拼装顺序与注入链路](#6-拼装顺序与注入链路)
7. [六个改写入口（速查表）](#7-六个改写入口速查表)
8. [整体替换 vs 追加：SYSTEM.md 与 APPEND_SYSTEM.md](#8-整体替换-vs-追加systemmd-与-append_systemmd)
9. [斜杠命令模板：格式、变量与加载规则](#9-斜杠命令模板格式变量与加载规则)
10. [运行期改写的最后一环：before_agent_start](#10-运行期改写的最后一环before_agent_start)
11. [另一套提示词：压缩与分支摘要](#11-另一套提示词压缩与分支摘要)
12. [怎么看到最终 prompt](#12-怎么看到最终-prompt)
13. [本仓库的现成例子](#13-本仓库的现成例子)
14. [常见坑与选择建议](#14-常见坑与选择建议)
15. [小结](#15-小结)

---

## 1. 一段 prompt，五个来源

最终喂给模型 system 角色的字符串，由五类来源按固定顺序拼成：

| # | 来源 | 位置 | 谁决定 |
|---|---|---|---|
| 1 | **内置主干**（角色 + 工具列表 + Guidelines + Pi 文档指引） | `core/system-prompt.ts:128` | 代码写死 |
| 2 | **工具提示词贡献点**（`snippet` / `guidelines`） | `core/tools/*.ts` | 当前启用的工具集 |
| 3 | **追加段**（`APPEND_SYSTEM.md` / `--append-system-prompt`） | 用户/项目/CLI | 用户 |
| 4 | **项目上下文**（`AGENTS.md` 等） | 目录树逐级向上 | 仓库作者 |
| 5 | **Skills 索引** | `core/skills.ts:355` | 发现的技能 |

第 1 项是二选一：如果提供了 `customPrompt`（`SYSTEM.md` 或 `--system-prompt`），**主干整个被替换**，第 2 项也随之消失（工具列表和 Guidelines 都在主干里）。这是最容易踩的一条，第 8 节展开。

## 2. 内置主干：默认分支逐段拆解

`buildSystemPrompt()`（`core/system-prompt.ts:28`）是唯一拼装入口。默认分支产出的文本：

```
You are an expert coding assistant operating inside pi, a coding agent harness. You help
users by reading files, executing commands, editing code, and writing new files.

Available tools:
${toolsList}

In addition to the tools above, you may have access to other custom tools depending on the project.

Guidelines:
${guidelines}

Pi documentation (read only when the user asks about pi itself, its SDK, extensions, themes, skills, or TUI):
- Main documentation: ${readmePath}
- Additional docs: ${docsPath}
- Examples: ${examplesPath} (extensions, custom tools, SDK)
- When reading pi docs or examples, resolve docs/... under Additional docs and examples/... under Examples, not the current working directory
- When asked about: extensions (docs/extensions.md, examples/extensions/), themes (docs/themes.md), skills (docs/skills.md), prompt templates (docs/prompt-templates.md), ...
- When working on pi topics, read the docs and examples, and follow .md cross-references before implementing
- Always read pi .md files completely and follow links to related docs (e.g., tui.md for TUI API details)
```

四个细节值得单独说：

**1）工具列表只列"有 snippet 的工具"。** `visibleTools = tools.filter((name) => !!toolSnippets?.[name])`（第 82 行）。工具注册了但没提供 `promptSnippet`，就不会出现在 `Available tools` 里——它仍然可调用，只是模型不知道有它。自定义工具忘了写 `promptSnippet` 是"我的工具从来不被调用"的头号原因。

**2）Guidelines 按工具可用性动态生成，且去重。** `addGuideline()` 用 Set 去重（第 89-95 行）。唯一的动态分支是第 105-113 行：当只有 shell 工具而没有 `grep`/`find`/`ls` 时，追加一条 `"Use bash for file operations like ls, rg, find"`（PowerShell-only 时措辞换成 PowerShell）。末尾固定追加两条：

```
- Be concise in your responses
- Show file paths clearly when working with files
```

**3）Pi 文档段是"懒加载指引"而不是文档本身。** 它只给三个绝对路径（`getReadmePath()` / `getDocsPath()` / `getExamplesPath()`）加一份"哪个话题看哪个文件"的索引，并明确要求"只在用户问 Pi 自身时才读"。这样设计是为了不把文档常量塞进每一轮的前缀——和 05 篇讲的 prompt cache 前缀稳定性是同一个动机。

**4）Working directory 在最后。** 拼装结束前追加 `Current working directory: <cwd>`，路径分隔符统一成 `/`。

`customPrompt` 分支（第 46-72 行）则只做四件事：拼追加段 → 拼 `<project_context>` → 拼 skills 段 → 追加 cwd。**没有工具列表、没有 Guidelines、没有 Pi 文档段。**

## 3. 工具提示词贡献点

每个内置工具导出一个 `{ snippet, guidelines }` 常量，注册时挂到工具定义的 `promptSnippet` / `promptGuidelines` 上，最终被 `AgentSession` 收集进 `toolSnippets` / `promptGuidelines`：

| 工具 | 定义位置 | snippet | guidelines |
|---|---|---|---|
| `read` | `tools/read.ts:27` | Read file contents | 用 `read` 看文件，不要用 `cat`/`sed` |
| `bash` | `tools/bash.ts:47` | Execute bash commands (ls, grep, find, etc.) | 可查看 `PI_*` 环境变量获取当前模型与会话信息 |
| `powershell` | `tools/powershell.ts:18` | Execute PowerShell commands | 同上 |
| `edit` | `tools/edit.ts:56` | Make precise file edits with exact text replacement, including multiple disjoint edits in one call | `oldText` 必须精确匹配；同一文件多处改动用一次调用的 `edits[]`；`oldText` 尽量小但要唯一；不要重叠/嵌套 |
| `write` | `tools/write.ts:20` | Create or overwrite files | 仅用于新建文件或整体重写 |
| `grep` | `tools/grep.ts:38` | Search file contents for patterns (respects .gitignore) | — |
| `find` | `tools/find.ts:37` | Find files by glob pattern (respects .gitignore) | — |
| `ls` | `tools/ls.ts:19` | List directory contents | — |

`edit` 的 guidelines 最长，因为它承载了工具最难说清的语义（第 58-63 行）。这也给自定义工具一个参照：**把"模型容易用错的地方"写进 guidelines，把"这是什么"写进 snippet。**

自定义工具走同一套：工具定义上的 `promptSnippet` 进 `Available tools`，`promptGuidelines` 进 `Guidelines`。

## 4. 项目上下文：AGENTS.md 的发现与注入

`loadProjectContextFiles()`（`core/resource-loader.ts:119`）负责收集，规则如下。

**候选文件名按优先级取第一个**：

```
AGENTS.override.md > AGENTS.md > AGENTS.MD > CLAUDE.md > CLAUDE.MD
```

（`loadContextFileFromDir()`，第 71-90 行。`AGENTS.override.md` 的存在意义是"我承认有 AGENTS.md，但我这台机器要覆盖它"——不用改名躲加载。）

**收集顺序**：先全局 `~/.pi/agent/`（agentDir），再从 cwd 沿目录树向上直到根，每级取一个。向上过程中用 `unshift`（第 145 行），所以**越靠近仓库根的文件排在越前**，`seenPaths` 去重。

**git worktree 去重**：`findShadowedContextFile()`（第 101 行）处理一种具体情形——主仓库的上下文文件与 linked worktree 自己的副本占用同一逻辑仓库范围，两份都加载等于把同一份规矩应用两次。判定逻辑很窄：只有当 worktree 根是主仓库根的子目录、且 `mainRepoRoot/.git` 规范化后等于 common git dir 时才遮蔽；sibling worktree（`git worktree add ../feat`）和 bare 布局不触发，退回正常的祖先继承。

**注入形式**：每个文件包成一段 XML，整体再包一层：

```
<project_context>

Project-specific instructions and guidelines:

<project_instructions path="/abs/path/AGENTS.md">
...文件内容原样...
</project_instructions>

</project_context>
```

路径是绝对路径，模型报错时能直接定位到文件。关闭用 `--no-context-files` / `-nc`。

## 5. Skills 段：只放索引，正文按需读

`formatSkillsForPrompt()`（`core/skills.ts:355`）生成的是 XML 索引，不是技能正文：

```
The following skills provide specialized instructions for specific tasks.
Use the read tool to load a skill's file when the task matches its description.
When a skill file references a relative path, resolve it against the skill directory (parent of SKILL.md / dirname of the path) and use that absolute path in tool commands.

<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>/abs/path/SKILL.md</location>
  </skill>
</available_skills>
```

三个约束：

- **只在有 `read` 工具时注入**（默认分支第 162 行判 `hasRead`；`customPrompt` 分支第 64 行判 `!selectedTools || selectedTools.includes("read")`）。没有读文件能力的 Agent 收到技能索引只会产生幻觉调用。
- **`disable-model-invocation: true` 的技能不进 prompt**（第 356 行过滤），只能通过 `/skill:name` 显式调用。适合那种"一旦被自动触发就很啰嗦"的技能。
- **正文靠 `read` 工具懒加载**，所以技能数量对前缀长度的影响是线性的、可控的；代价是模型必须主动去读，description 写得不清楚就等于没有这个技能。

## 6. 拼装顺序与注入链路

完整顺序（`buildSystemPrompt()`）：

```
1. customPrompt（有则用它，跳过 2-4；没有则用内置主干）
2. Available tools（toolSnippets 过滤后的 selectedTools）
3. Guidelines（动态工具指引 + promptGuidelines + 固定两条，去重）
4. Pi 文档指引（readme / docs / examples 三个绝对路径）
5. appendSystemPrompt（追加段，多个用 "\n\n" 连接）
6. <project_context> → <project_instructions path=...>
7. <available_skills>（仅当有 read 工具）
8. Current working directory: <cwd>
```

注入链路：

```
DefaultResourceLoader（发现 SYSTEM.md / APPEND_SYSTEM.md / AGENTS.md / skills / prompts）
        │
        ▼
AgentSession._rebuildSystemPrompt(toolNames)      core/agent-session.ts:1034
        │  收集 _toolPromptSnippets / _toolPromptGuidelines
        │  组装 BuildSystemPromptOptions
        ▼
buildSystemPrompt(options)                        core/system-prompt.ts:28
        ▼
agent.state.systemPrompt
        ▼
ExtensionRunner.emitBeforeAgentStart(...)         core/extensions/runner.ts:1081
        │  扩展可返回 systemPrompt 链式覆盖
        ▼
最终发给 provider
```

要点：

- `_rebuildSystemPrompt()` 只在工具集变化时调用（`setActiveTools()`），不是每轮重建。这意味着**中途开关工具会重写前缀**，直接击穿 prompt cache（见 05 篇）。
- 追加段在 `agent-session.ts:1052-1053` 合并：loader 返回数组，`join("\n\n")` 成一个字符串；空数组则传 `undefined`。
- SDK 侧还有一条并行路径：`server/create-harness.ts:56` 的 `buildCodingAgentHarnessSystemPrompt()`，最终同样落到 `buildSystemPrompt()`。
- CLI 参数在 `main.ts` 传给 `resourceLoaderOptions`：`--system-prompt`、`--append-system-prompt`（可重复）、`--prompt-template`（可重复）、`--no-context-files` 等（解析见 `cli/args.ts:110-194`）。

## 7. 六个改写入口（速查表）

| 想改什么 | 放哪 | 作用范围 |
|---|---|---|
| 加项目规矩 | 各级目录 `AGENTS.md`（或 `AGENTS.override.md`） | 该目录及以下，随目录树继承 |
| 追加全局偏好 | `~/.pi/agent/APPEND_SYSTEM.md` | 所有会话 |
| 追加项目偏好 | `<cwd>/.pi/APPEND_SYSTEM.md`（需项目受信任） | 该项目 |
| 整体替换 | `~/.pi/agent/SYSTEM.md` 或 `<cwd>/.pi/SYSTEM.md` | **替换掉整个主干** |
| 一次性/临时 | `--system-prompt <text>`、`--append-system-prompt <text\|path>`（可重复） | 单次运行 |
| 编程式改写 | 扩展的 `before_agent_start` 返回 `systemPrompt`；SDK 用 `DefaultResourceLoaderOptions` 的 `systemPromptOverride` / `appendSystemPromptOverride` / `agentsFilesOverride` / `skillsOverride` / `promptsOverride` | 运行期 |

还有两个"不是 system prompt，但形状一样"的入口：斜杠命令模板（第 9 节，作用于用户输入）和压缩提示词（第 11 节，作用于另一路 LLM 调用）。

## 8. 整体替换 vs 追加：SYSTEM.md 与 APPEND_SYSTEM.md

两个文件都是"项目级优先，回退全局"，且项目级都要求仓库受信任（`discoverSystemPromptFile()` 第 1023 行、`discoverAppendSystemPromptFile()` 第 1037 行）：

```ts
private discoverSystemPromptFile(): string | undefined {
	const projectPath = join(this.cwd, CONFIG_DIR_NAME, "SYSTEM.md");
	if (this.settingsManager.isProjectTrusted() && existsSync(projectPath)) return projectPath;
	const globalPath = join(this.agentDir, "SYSTEM.md");
	if (existsSync(globalPath)) return globalPath;
	return undefined;
}
```

`resolvePromptInput()`（第 54 行）让这两个值**既可以是路径也可以是字面文本**：`existsSync(input)` 为真就读文件，否则原样当作提示词内容。所以 `--system-prompt "You are ..."` 和 `--system-prompt ./my-prompt.md` 都成立。

**选哪个的判断标准**：

- 想加规矩、加风格、加禁止事项 → `APPEND_SYSTEM.md` 或 `AGENTS.md`。主干保留，工具说明和 Pi 文档指引都还在。
- 想让 Pi 完全变成另一个东西（非编码场景、严格结构化输出、接外部规范）→ `SYSTEM.md`。但这时你必须自己把"有什么工具、怎么用"写回去，否则模型会用得很笨。

一个折中做法：`SYSTEM.md` 里先抄一遍主干的关键段（工具列表、Guidelines），再叠加自己的内容。代价是你承担了主干升级时的同步成本。

## 9. 斜杠命令模板：格式、变量与加载规则

模板作用于**用户输入**，不在 system prompt 里。加载器 `core/prompt-templates.ts` 的 `loadPromptTemplates()` 扫五处：

- 全局 `~/.pi/agent/prompts/*.md`
- 项目 `<cwd>/.pi/prompts/*.md`（需项目受信任）
- 包：`prompts/` 目录或 `package.json` 的 `pi.prompts` 条目
- 设置：`prompts` 数组（文件或目录）
- CLI：`--prompt-template <path>`（可重复）

发现可整体关闭：`--no-prompt-templates` / `-np`。

**格式**：YAML frontmatter + 正文，**文件名即命令名**（`pr.md` → `/pr`）。

```markdown
---
description: Review PRs from URLs with structured issue and code analysis
argument-hint: "<PR-URL>"
---

Review the PR at $1. Focus on $2.
```

- `description` 可选，缺失时用第一个非空行；用于自动补全下拉展示。
- `argument-hint` 可选，展示在补全列表里；`<>` 表示必填，`[]` 表示选填。

**变量替换**（`substituteArgs()`，第 70 行）：

| 写法 | 含义 |
|---|---|
| `$1`、`$2`… | 位置参数 |
| `$@` / `$ARGUMENTS` | 全部参数拼接 |
| `${1:-default}` | 位置参数缺省/为空时用默认值 |
| `${@:-default}` / `${ARGUMENTS:-default}` | 全部参数为空时用默认值 |
| `${@:N}` | 从第 N 个开始（1-indexed） |
| `${@:N:L}` | 从第 N 个起取 L 个 |

参数切分用 `parseCommandArgs()`（第 24 行），支持 bash 风格引号：`/component Button "click handler"` 是两个参数。替换**只做一遍**，参数值或默认值里再出现 `$1` 不会被递归展开。

**两个加载规则**：`prompts/` 目录**不递归**发现；想放子目录就得在设置或包 manifest 里显式列出。

模板是"把重复的高质量提示词固化下来"的主要手段，也是 subagent 工作流的编排层（第 03、11 篇的 `/implement`、`/scout-and-plan` 就是这类文件）。

## 10. 运行期改写的最后一环：before_agent_start

`emitBeforeAgentStart()`（`core/extensions/runner.ts:1081`）在每轮 agent 循环前把已拼好的 prompt 交给扩展：

```ts
const event: BeforeAgentStartEvent = {
	type: "before_agent_start",
	prompt, images,
	systemPrompt: currentSystemPrompt,
	systemPromptOptions,          // BuildSystemPromptOptions
};
const handlerResult = await handler(event, ctx);
if (result.systemPrompt !== undefined) {
	currentSystemPrompt = result.systemPrompt;   // 链式覆盖
	systemPromptModified = true;
}
```

三个用法：

1. **读**：`ctx.getSystemPrompt()` 返回当前生效值（示例 `examples/extensions/system-prompt-header.ts` 用它显示 prompt 长度）。
2. **按上下文改**：`systemPromptOptions` 暴露了 `selectedTools`、`skills`、`cwd` 等，可以"有 bash 就加 shell 规范、有某 skill 就加对应流程"，而不是无脑追加（`examples/extensions/prompt-customizer.ts` 就是这个模式）。
3. **整体改**：返回 `systemPrompt` 直接替换。多个扩展按顺序执行，**后面的覆盖前面的**，所以写这类扩展时要意识到自己在跟别人抢最后一棒。

## 11. 另一套提示词：压缩与分支摘要

压缩走的是独立的一次 LLM 调用，不属于 system prompt，但同属提示词体系，改的时候容易混：

| 常量 | 位置 | 用途 |
|---|---|---|
| `SUMMARIZATION_SYSTEM_PROMPT` | `core/compaction/utils.ts:156` | 压缩调用的 system 角色：要求"只输出结构化摘要，不要继续对话、不要回答对话里的问题" |
| `SUMMARIZATION_PROMPT` | `core/compaction/compaction.ts:467` | 首次摘要的用户侧模板 |
| `UPDATE_SUMMARIZATION_PROMPT` | `compaction.ts:537` | 已有摘要时的增量更新模板（第 679 行按 `previousSummary` 二选一） |
| `TURN_PREFIX_SUMMARIZATION_PROMPT` | `compaction.ts:836` | 按轮次前缀切分时的模板 |
| `BRANCH_SUMMARY_PROMPT` | `core/compaction/branch-summarization.ts` | 分支摘要 |

自定义说明通过 `customInstructions` 传入，语义是**追加**（`${basePrompt}\n\nAdditional focus: ${customInstructions}`），只有 `branch-summarization.ts` 额外提供 `replaceInstructions` 走整体替换（第 328-333 行）。

改这里的收益和改主干不一样：它决定"上下文被压掉之后还剩下什么"，直接影响长任务后段的连贯性，见第 04 篇。

## 12. 怎么看到最终 prompt

Pi 没有 `dump system prompt` 这样的内置命令。三种实用办法：

1. **一次性扩展**：在 `before_agent_start` 里把 `event.systemPrompt` 写到 `/tmp`，跑一轮再看。这是唯一能看到"所有来源叠加后"结果的方式。
2. **长度监控**：`examples/extensions/system-prompt-header.ts` 用 `ctx.getSystemPrompt()` 把长度显示在状态栏，适合验证"我加的这一份到底进没进去"。
3. **SDK 侧**：`examples/sdk/03-custom-prompt.ts` 直接传 `systemPromptOverride`，所见即所得。

调试顺序建议：先看长度（确认注入了）→ 再 dump 全文（确认顺序和重复）→ 最后再看模型行为。

## 13. 本仓库的现成例子

| 路径 | 说明 |
|---|---|
| `.pi/prompts/cl.md` | 发布前审计各包 CHANGELOG |
| `.pi/prompts/is.md` | issue 分析 |
| `.pi/prompts/pr.md` | PR 审查（带 `argument-hint: "<PR-URL>"`） |
| `.pi/prompts/sa.md` | security advisory 更新，含 YAML 输出契约 |
| `.pi/prompts/wr.md` | 收尾流程：changelog + commit + push + 关闭 issue |
| `.pi/skills/add-llm-provider.md` | 项目级技能 |
| `.pi/extensions/` | 项目级扩展 |
| `examples/extensions/subagent/agents/*.md` | `planner` / `reviewer` / `scout` / `worker`，frontmatter 四字段 + 六段式正文 |
| `examples/extensions/subagent/prompts/*.md` | 工作流编排模板（`$@`、`{previous}`） |
| `examples/extensions/prompt-customizer.ts` | 按 `systemPromptOptions` 条件追加 |
| `examples/extensions/system-prompt-header.ts` | `ctx.getSystemPrompt()` 用法 |

写新的模板时先抄一份结构最接近的改，比从空白开始快得多。

## 14. 常见坑与选择建议

**坑 1：用 `--system-prompt` / `SYSTEM.md` 之后模型不会用工具了。** 因为主干（含 `Available tools` 和 `Guidelines`）被整体替换。九成情况下你要的是 `APPEND_SYSTEM.md` 或 `AGENTS.md`。

**坑 2：自定义工具存在但模型从不调用。** 检查工具定义有没有 `promptSnippet`——没有就不进 `Available tools`。

**坑 3：`AGENTS.md` 写了但没生效。** 依次排查：是否被更近目录的同名文件顶掉（只取路径上每级一个）、是否在未受信任的项目里（`.pi/` 下的资源需要信任）、是否用了 `--no-context-files`、git worktree 场景下是否被遮蔽逻辑去重。

**坑 4：中途切换工具集导致成本飙升。** `setActiveTools()` 会触发 `_rebuildSystemPrompt()`，前缀变了，缓存失效。

**坑 5：追加段和项目上下文的先后顺序。** 追加段（5）排在 `<project_context>`（6）**之前**。两份内容冲突时，位置靠后的项目上下文离生成更近，实际约束力通常更强。想让你的规矩压过仓库的 `AGENTS.md`，别放在 `APPEND_SYSTEM.md`，用 `before_agent_start` 覆盖或放更靠后的位置。

**选择建议**（从弱到强）：

```
AGENTS.md          加项目规矩，随仓库走，团队共享
APPEND_SYSTEM.md   加个人或项目偏好，不影响主干结构
斜杠命令模板       固化可复用的任务提示词
before_agent_start 按运行期上下文条件改写
SYSTEM.md          彻底换人设（自己背工具说明的成本）
```

## 15. 小结

Pi 的提示词设计可以概括成三句话：

1. **主干极简，人格靠外挂。** 内置提示词只负责"你是编码助手 + 有什么工具 + 怎么读 Pi 文档"，项目规矩全部由 `AGENTS.md` 树和追加段提供。
2. **每个来源有明确的边界和代价。** 替换主干最自由也最贵（丢工具说明），追加最安全，扩展最灵活但要注意覆盖顺序。
3. **拼装顺序是稳定的，稳定性本身就是设计目标。** 固定顺序 + 工具集变化才重建，是为了保住 prompt cache 前缀——这也是为什么不建议在每轮都动态改 system prompt。

理解这套结构之后，"让 Pi 按我的方式干活"就从调玄学提示词，变成了一个有明确落点的工程问题：想清楚这条规矩属于哪一层，然后放到对应的文件里。
