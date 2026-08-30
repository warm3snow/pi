# Pi 主要应用场景与 pi-coding-agent 详细设计

> 系列第 02 篇（[系列索引](README.md)）。本文承接[第 01 篇](01-pi-agent-architecture.md)，聚焦两个问题：Pi 在实践中主要用在哪些场景，以及 `pi-coding-agent` 这个"外壳"包内部具体是怎么设计的。内容按"先看场景、再看内部结构"的顺序展开。

## 目录

1. [Pi 的主要应用场景](#1-pi-的主要应用场景)
2. [pi-coding-agent 定位回顾](#2-pi-coding-agent-定位回顾)
3. [源码目录结构总览](#3-源码目录结构总览)
4. [核心层：AgentSession 与 Runtime 分层](#4-核心层agentsession-与-runtime-分层)
5. [资源加载：ResourceLoader 与项目信任](#5-资源加载resourceloader-与项目信任)
6. [工具子系统的具体设计](#6-工具子系统的具体设计)
7. [扩展运行时：ExtensionRunner 的事件分发机制](#7-扩展运行时extensionrunner-的事件分发机制)
8. [四种运行模式的具体实现](#8-四种运行模式的具体实现)
9. [系统提示词的构建逻辑](#9-系统提示词的构建逻辑)
10. [CLI 层：入口与参数解析](#10-cli-层入口与参数解析)
11. [设计小结](#11-设计小结)

---

## 1. Pi 的主要应用场景

Pi 官方定位是"终端编码 Agent"，但通过 SDK/RPC/扩展体系，实际落地的场景比"终端里聊天写代码"要广得多。可以归纳为五大类：

### 1.1 交互式终端编码（核心场景）

最直接的用法：在项目目录下运行 `pi`，通过自然语言驱动 Agent 读文件、改代码、跑命令。

```bash
cd /path/to/project
pi
```

典型任务：

- 让 Agent 阅读仓库并总结架构、定位 Bug、实现新功能
- 通过 `@file` 引用文件、`!command` 直接跑 shell 命令并把输出喂给模型
- 通过 `AGENTS.md`/`CLAUDE.md` 声明项目规范（如"改动后跑 `npm run check`"），让 Agent 遵守团队约定
- 用 `/model`、Shift+Tab 在不同模型/思考等级间切换，按任务难度控制成本

这一场景直接对应 `docs/quickstart.md` 和 `docs/usage.md` 描述的默认工作流，是其余场景的基础。

### 1.2 CI/自动化流水线中的无人值守任务

通过 `-p`（print 模式）或 `--mode json` 一次性调用 Agent，不需要人工交互，适合脚本/CI 集成：

```bash
pi -p "Fix the failing test in test/foo.test.ts and run the test suite"
```

结合 `--mode rpc` 可以让外部脚本/流水线**结构化控制**每一步（发送 prompt、订阅事件、判断是否成功），而不是解析终端文本输出。典型用途：自动修复 CI 报错、批量代码迁移、自动生成 PR 描述等。这类场景通常搭配容器化方案（见 1.5）做隔离，避免无人值守时权限过大带来的风险。

### 1.3 嵌入到 IDE / 自定义应用（SDK 集成场景）

`pi-coding-agent` 本身是一个可编程的 npm 包，`createAgentSession()` 可以直接在 Node.js 应用里创建一个 Agent 会话，无需拉起子进程：

```typescript
import { createAgentSession, ModelRuntime, SessionManager } from "@earendil-works/pi-coding-agent";

const modelRuntime = await ModelRuntime.create();
const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
  modelRuntime,
});
session.subscribe((event) => { /* 渲染到自定义 UI */ });
await session.prompt("What files are in the current directory?");
```

适用于：给桌面/Web IDE 接入 Agent 面板、给内部工具加一个"用自然语言操作代码库"的入口、构建定制化的编码助手产品。SDK 与 RPC 的选择依据很明确：**同进程、要类型安全、要直接读写 Agent 状态**选 SDK；**跨语言、要进程隔离**选 RPC。

### 1.4 多 Agent 协作 / 任务委派（Subagent 场景）

官方 `subagent` 示例扩展展示了一种进阶模式：主 Agent 把子任务委派给独立的 `pi` 子进程（各自独立上下文窗口），支持单任务、并行任务（最多 8 个，4 并发）、链式任务三种模式，并通过 Markdown + YAML frontmatter 定义专职子 Agent（如 `scout` 快速侦察、`planner` 出方案、`reviewer` 代码评审、`worker` 通用执行）。

```text
Use a chain: first have scout find the read tool, then have planner suggest improvements
```

这类场景解决的问题是：单一长对话上下文会被无关探索污染、变得臃肿；拆分为"侦察-规划-实现-评审"流水线后，每个子 Agent 只带着任务必需的上下文工作，主 Agent 上下文保持精炼。适合复杂重构、大范围代码调研等任务。

### 1.5 隔离环境下的沙箱化运行

由于 Pi 本身不提供沙箱（见安全模型一节），"在隔离环境中运行 Pi"本身构成一类重要场景，尤其用于：不信任的仓库、需要无人值守运行的自动化任务、多租户 SaaS 化的 Agent 服务。官方给出三种模式：

| 模式 | 隔离范围 | 典型场景 |
|---|---|---|
| Gondolin 扩展 | 仅内置工具与 `!` 命令进入本地 micro-VM，鉴权仍在主机 | 本地开发时想要文件系统级隔离，又不想丢失主机凭证配置 |
| 纯 Docker | 整个 `pi` 进程跑在容器里 | 简单的本地隔离，CI Runner |
| OpenShell | 整个 `pi` 进程跑在策略化沙箱（本地网关或远程 K8s 网关）里 | 多租户远程沙箱，需要文件系统/进程/网络/凭证/推理的精细策略控制 |

配合 `pi-protocol`/`pi-server`/`pi-client` 三件套，这类场景可以进一步演化为"云端跑一批隔离的 Agent 沙箱，本地/Web 客户端远程连接会话"的架构，是构建多用户 Agent 云服务的基础。

### 1.6 场景与包的对应关系

把以上场景和文章一开头介绍的分层架构对应起来：

```
场景                          主要依赖的包/机制
──────────────────────────────────────────────────────
交互式终端编码                 pi-coding-agent（Interactive 模式 + TUI）
CI/自动化流水线                pi-coding-agent（Print / JSON / RPC 模式）
IDE/应用集成                   pi-coding-agent SDK（createAgentSession）
多 Agent 协作                  pi-coding-agent 扩展系统（subagent 扩展）+ 子进程 RPC
沙箱化/远程化运行               containerization 方案 + pi-protocol/client/server
```

可以看到，`pi-coding-agent` 是几乎所有场景的**唯一交汇点**——不管是交互终端、CI 脚本、IDE 插件还是子 Agent，最终都是在创建和驱动一个 `AgentSession`。这也是下文把 `pi-coding-agent` 单独拆开详细讲解的原因。

---

## 2. pi-coding-agent 定位回顾

`pi-coding-agent` = **`pi-agent-core`（引擎）+ 编码专用工具 + 资源系统（扩展/技能/模板）+ 四种运行模式（Interactive/Print/RPC/JSON）+ CLI**。它不是简单的"UI 层"，而是把通用 Agent 引擎"产品化"为一个可安装、可配置、可扩展的编码助手所需的一切粘合代码。

设计目标可以概括为一句话：**同一个 `AgentSession` 核心，套上不同的壳，服务不同的接入方式**。

---

## 3. 源码目录结构总览

```
packages/coding-agent/src/
├── main.ts                    CLI 入口：参数解析 → 组装 AgentSessionRuntime → 分发到对应模式
├── config.ts                  路径常量（APP_NAME、agentDir 等）
├── migrations.ts               版本升级迁移与弃用警告
├── package-manager-cli.ts      `pi install/remove/list/update` 命令实现
│
├── cli/                        CLI 专属逻辑（与核心解耦）
│   ├── args.ts                 参数解析（Mode、Args 类型定义）
│   ├── auth-check.ts / auth-command.ts   鉴权检测与 `pi auth` 子命令
│   ├── file-processor.ts       `@file` 引用处理
│   ├── initial-message.ts      首条消息拼装
│   ├── list-models.ts          `--list-models`
│   ├── project-trust.ts        CLI 层的信任决策封装
│   ├── session-picker.ts       `-c`/session 选择 UI
│   └── startup-ui.ts           首次运行引导、启动选择器
│
├── core/                        核心业务逻辑（可被 SDK 直接复用，不依赖具体 UI）
│   ├── agent-session.ts         AgentSession：跨模式共享的会话外观类
│   ├── agent-session-runtime.ts AgentSessionRuntime：可替换会话的运行时容器
│   ├── agent-session-services.ts cwd 绑定的服务集合（工具/资源/鉴权等的组装）
│   ├── resource-loader.ts       ResourceLoader 接口 + DefaultResourceLoader
│   ├── extensions/              扩展加载、运行时事件分发（本文第 7 节详解）
│   ├── tools/                   内置工具实现（read/write/edit/bash/grep/find/ls）
│   ├── compaction/               压缩算法实现
│   ├── model-runtime.ts / model-registry.ts / model-resolver.ts   模型解析与鉴权
│   ├── session-manager.ts        会话树持久化（JSONL）
│   ├── settings-manager.ts       全局/项目设置的加载与合并
│   ├── system-prompt.ts          系统提示词拼装
│   ├── slash-commands.ts         斜杠命令解析
│   ├── skills.ts / prompt-templates.ts   技能与模板加载
│   ├── project-trust.ts / trust-manager.ts   项目信任判定与持久化
│   ├── sdk.ts                    对外 SDK 类型汇总（CreateAgentSessionOptions 等）
│   └── export-html/              会话导出为 HTML
│
├── modes/                        四种运行模式的具体实现
│   ├── interactive/               TUI 交互模式（组件、主题、编辑器等）
│   ├── print-mode.ts               `-p` 一次性模式
│   ├── json-event.ts               `--mode json` 结构化事件流模式
│   └── rpc/                        `--mode rpc` JSONL 协议模式
│
└── extensions/                    随包内置的默认扩展（如 llama.cpp 集成）
```

这个划分体现了一条清晰的边界线：**`core/` 是与 UI 无关的业务核心，`modes/` 是套在 `core/` 外面的具体交互壳，`cli/` 是专属于命令行启动流程的粘合代码**。SDK 使用者只需要 `core/` 暴露的 API，完全不需要关心 `modes/`、`cli/` 的实现。

---

## 4. 核心层：AgentSession 与 Runtime 分层

`pi-coding-agent` 的核心抽象分两层，职责边界非常明确：

### 4.1 AgentSession：单个会话的外观类

`AgentSession`（`core/agent-session.ts`）包裹一个 `pi-agent-core` 的 `Agent` 实例，向所有运行模式暴露统一接口，职责包括：

- 转发/包装 `Agent` 的事件流，补充会话级事件（如 `agent_settled`、`compaction_start/end`、`auto_retry_*`）
- 事件订阅时自动做**会话持久化**（把消息写入 JSONL）
- 模型与思考等级管理（`setModel`、`cycleModel`、`setThinkingLevel`）
- 手动/自动压缩调度（`compact()`、内部的阈值判断 `shouldCompact`）
- Bash 直接执行支持（区别于 LLM 触发的 `bash` 工具调用，这是给 RPC `bash` 命令和交互模式的 `!command` 用的直接执行通道）
- 会话内树导航（`navigateTree()`，用于 `/tree` 分支切换）

一个会话生命周期内只有一个 `AgentSession` 实例，是"当前活跃对话"的具体化。

### 4.2 AgentSessionRuntime：可替换会话的容器

`AgentSessionRuntime`（`core/agent-session-runtime.ts`）解决的问题是：`/new`、`/resume`、`/fork`、`/clone`、导入 JSONL 等操作本质上是**替换掉当前的 `AgentSession`**，但 UI 层（TUI 组件、事件订阅）不应该在每次替换时重新搭建。

```typescript
const createRuntime: CreateAgentSessionRuntimeFactory = async ({ cwd, sessionManager, sessionStartEvent }) => {
  const services = await createAgentSessionServices({ cwd });   // 重建 cwd 绑定的服务
  return {
    ...(await createAgentSessionFromServices({ services, sessionManager, sessionStartEvent })),
    services,
    diagnostics: services.diagnostics,
  };
};
const runtime = await createAgentSessionRuntime(createRuntime, { cwd, agentDir, sessionManager });

// 之后任何这些调用都会重建内部的 AgentSession：
await runtime.newSession();
await runtime.switchSession(path);
await runtime.fork(entryId);
```

关键约束（源码注释明确标注）：`runtime.session` 在这些操作后会**变成一个新实例**，旧的事件订阅不会自动迁移——调用方必须重新 `subscribe()`，如果用了扩展系统还要重新 `bindExtensions()`。这是一个典型的"显式优于隐式"设计：把"谁负责重新订阅"这件事留给调用方决定，而不是让 Runtime 悄悄做胶水绑定。

### 4.3 AgentSessionServices：cwd 绑定的服务集合

`createAgentSessionServices()`（`core/agent-session-services.ts`）是"给定一个 cwd，组装出这个 cwd 下所有会话都要用的共享服务"的工厂函数，包括模型运行时、设置管理器、资源加载器、内置工具集等。`AgentSessionRuntime` 在切换 cwd（如通过 `/cd` 或多工作区场景）时会重新调用它，保证工具路径解析、项目级配置都锚定到正确的目录。

三层关系总结：

```
AgentSessionServices  (cwd 绑定的一组共享依赖：ModelRuntime/SettingsManager/ResourceLoader/Tools)
        ↓ 被用来构造
AgentSession          (单个具体会话：状态 + 事件 + 持久化)
        ↓ 被包裹在
AgentSessionRuntime   (可替换容器：/new /resume /fork /clone 时重建 AgentSession)
```

---

## 5. 资源加载：ResourceLoader 与项目信任

### 5.1 ResourceLoader 接口

`ResourceLoader`（`core/resource-loader.ts`）统一了扩展、技能、提示词模板、主题、上下文文件（`AGENTS.md`）五类资源的发现与暴露：

```typescript
interface ResourceLoader {
  getExtensions(): LoadExtensionsResult;
  getSkills(): { skills: Skill[]; diagnostics: ResourceDiagnostic[] };
  getPrompts(): { prompts: PromptTemplate[]; diagnostics: ResourceDiagnostic[] };
  getThemes(): { themes: Theme[]; diagnostics: ResourceDiagnostic[] };
  getAgentsFiles(): { agentsFiles: Array<{ path: string; content: string }> };
  getSystemPrompt(): string | undefined;
  reload(options?: ResourceLoaderReloadOptions): Promise<void>;
}
```

默认实现 `DefaultResourceLoader` 按"全局目录（`~/.pi/agent/`）+ 项目目录（`.pi/`）"两级扫描；SDK 使用者可以通过 `xxxOverride` 回调（如 `skillsOverride`、`promptsOverride`、`systemPromptOverride`）注入自定义资源而不用完全重写加载器，也可以传入自定义类实现来完全接管发现逻辑。

`/reload` 命令直接调用 `reload()`，实现扩展/技能/模板的热重载，无需重启进程。

### 5.2 项目信任作为资源加载的前置闸门

`reload(options)` 接受一个 `resolveProjectTrust` 回调——这体现了"信任判断"和"资源加载"解耦的设计：`ResourceLoader` 只负责发现文件，是否加载取决于外部注入的信任决策函数。触发信任判断的条件很精确（详见第 10 节 CLI 层，`hasTrustRequiringProjectResources` 只在检测到 `.pi/settings.json`、`.pi/extensions|skills|prompts|themes`、`.pi/SYSTEM.md` 等特定文件时才认为"有需要信任的资源"，一个空的 `.pi` 目录不触发）。

`findShadowedContextFile()`（`resource-loader.ts` 内的一个精巧细节）专门处理 git worktree 场景：当在一个链接的子 worktree 里运行时，主仓库根目录下同名的 `AGENTS.md` 会被子 worktree 自己的版本"遮蔽"，避免同一份上下文被同时加载两次。这是一个典型的"边界条件被显式建模而不是被忽略"的例子。

---

## 6. 工具子系统的具体设计

### 6.1 分层：ToolDefinition vs AgentTool

`core/tools/` 下每个工具（`read.ts`/`write.ts`/`edit.ts`/`bash.ts`/`powershell.ts`/`grep.ts`/`find.ts`/`ls.ts`）都同时导出两种形态：

- `createXxxTool(cwd, options)` → 返回 `pi-agent-core` 认识的 `AgentTool`（真正参与 Agent Loop 的执行体）
- `createXxxToolDefinition(cwd, options)` → 返回 coding-agent 自己的 `ToolDefinition`（携带渲染/展示相关的元信息，如 TUI 里怎么显示这次工具调用）

`core/tools/index.ts` 提供了统一的工厂入口：

```typescript
createTool(toolName, cwd, options)              // 单个工具（AgentTool 形态）
createToolDefinition(toolName, cwd, options)    // 单个工具（ToolDefinition 形态）
createCodingTools(cwd)      / createCodingToolDefinitions(cwd)      // read+bash+edit+write
createReadOnlyTools(cwd)    / createReadOnlyToolDefinitions(cwd)    // read+grep+find+ls
createAllTools(cwd)         / createAllToolDefinitions(cwd)         // 全部 8 个工具的 Record
```

`ToolName` 是一个受限的字面量联合类型（`"read"|"bash"|"powershell"|"edit"|"write"|"grep"|"find"|"ls"`），保证工具名在编译期就被约束，不会拼错。

### 6.2 横切关注点：文件互斥队列

`file-mutation-queue.ts` 的 `withFileMutationQueue()` 是一个跨工具的横切能力——当 `write`/`edit` 并发执行模式打开时（见架构文中"工具执行模型"一节的 parallel 模式），多个工具调用可能同时想改同一个文件；这个队列确保对同一文件路径的写操作被序列化，避免并发写入互相覆盖导致的数据损坏，同时不影响写不同文件的并行度。

### 6.3 bash/powershell 的可插拔执行后端

`BashOperations`/`PowerShellOperations` 是抽象接口，`createLocalBashOperations()` 是默认的本机 `child_process` 实现。这个抽象层正是 Gondolin/OpenShell 容器化扩展（第 1.5 节场景）的接入点——扩展可以提供把命令路由进 micro-VM 的自定义 `BashOperations` 实现，替换掉 `read`/`write`/`edit`/`bash`/`grep`/`find`/`ls` 背后的真实执行逻辑，而不需要改动 Agent Loop 或工具的 Schema 定义。`BashSpawnHook` 允许扩展在真正 spawn 子进程前拦截、改写命令。

### 6.4 输出治理：截断与累积

`truncate.ts`（`truncateHead`/`truncateTail`/`truncateLine`，及 `DEFAULT_MAX_BYTES`/`DEFAULT_MAX_LINES`）和 `output-accumulator.ts` 共同解决"工具输出可能远超模型上下文预算"的问题——大文件读取、长时间运行命令的输出会被截断并保留指向完整输出的路径（如 RPC `bash` 命令响应里的 `fullOutputPath`），同时流式累积逻辑保证 TUI/RPC 的 `tool_execution_update` 事件里的 `partialResult` 始终是"迄今为止的完整累积内容"而不是增量 diff，简化了客户端渲染逻辑。

---

## 7. 扩展运行时：ExtensionRunner 的事件分发机制

`core/extensions/` 是整个"核心极简、扩展负责一切"哲学的落地引擎，包含四个文件：

```
loader.ts    发现 + 加载扩展文件（基于 jiti，免编译执行 TypeScript）
types.ts     ExtensionAPI / ExtensionContext / 各类事件类型定义
runner.ts    ExtensionRunner：事件订阅与分发的运行时核心
wrapper.ts   包装用户注册的工具，注入 ExtensionContext
```

### 7.1 加载：jiti 免编译执行

`loadExtensionsCached()`/`loadExtensionFromFactory()` 使用 [jiti](https://github.com/unjs/jiti) 直接执行 `.ts` 扩展文件，用户不需要预先编译。`clearExtensionCache()` 支撑 `/reload` 的热重载语义——重新加载前清空模块缓存，保证拿到文件的最新内容。

### 7.2 分发：事件总线 + 钩子链

`ExtensionRunner` 承担两类不同性质的分发：

- **通知型事件**（`pi.on("session_start", handler)`）：多个扩展的 handler 按注册顺序依次触发，互不阻塞对方（除非显式约定返回值有特殊语义，比如 `project_trust` 事件"第一个返回决策的扩展生效"）。
- **可拦截型钩子**（`beforeToolCall`/`afterToolCall`/`tool_call` 事件）：任意一个扩展返回 `{ block: true }` 就能阻止工具执行；多个扩展的钩子按顺序链式调用，前一个的输出可以影响后一个看到的输入（如 `afterToolCall` 改写结果后，下一个扩展的 `afterToolCall` 看到的是已经被改写的结果）。

这种区分保证了"想通知我一下"和"我要能拦下来"两种截然不同的扩展意图都能被清晰表达，而不需要每个扩展自己判断"要不要 return"。

### 7.3 ExtensionContext：给扩展的能力面

`ExtensionContext`（含 `ctx.ui`）是扩展在事件处理函数里拿到的唯一"操作面"，划分出清晰的能力边界：

- `ctx.ui.confirm/select/input/editor`：阻塞式对话，等待用户响应（RPC 模式下映射为 `extension_ui_request`/`extension_ui_response` 子协议）
- `ctx.ui.notify/setStatus/setWidget/setTitle`：即发即弃的展示更新
- `ctx.ui.custom()`：仅在 TUI 模式下可用的完整自定义组件渲染（RPC 模式下返回 `undefined`，因为没有真实终端）
- `pi.registerTool/registerCommand/registerShortcut/registerFlag`：扩展点注册（在 `session_start` 之前的初始化阶段调用）

RPC 文档中明确列出了哪些 `ExtensionContext` 方法在 RPC 模式下是"降级"或"空操作"的（如 `getEditorText()` 返回 `""`），这体现了"同一套扩展 API，在不同壳层下能力有真实差异，且差异被显式文档化"而不是隐藏起来让扩展作者自己踩坑。

### 7.4 wrapper.ts：把用户工具接入 Agent Loop

`wrapRegisteredTools()` 把扩展通过 `pi.registerTool()` 注册的用户工具，转换/包装成 `pi-agent-core` 认识的 `AgentTool`，同时注入 `ExtensionContext` 作为工具 `execute()` 的额外参数——这是"扩展注册的工具"和"内置工具"能够在 Agent Loop 里被同等对待、统一调度的关键粘合层。

---

## 8. 四种运行模式的具体实现

`modes/` 下每种模式都是在同一个 `AgentSessionRuntime` 之上加一层不同的 I/O 驱动逻辑：

### 8.1 Interactive（`modes/interactive/`）

体量最大的模式（40+ 文件），核心是 `interactive-mode.ts` 的 `InteractiveMode` 类，配合：

- `components/`：TUI 组件（编辑器、消息渲染、Footer、Header、模型选择器等），基于 `pi-tui` 的差量渲染
- `theme/`：主题加载与文件热监听（`stopThemeWatcher`）
- `external-editor.ts`：调用系统 `$EDITOR` 编辑长文本
- `model-search.ts`/`model-catalog-refresh.ts`：模型选择器背后的搜索与刷新逻辑
- `session-share.ts`：会话分享功能

`InteractiveMode` 消费 `AgentSessionRuntime` 而不是裸的 `AgentSession`，这样才能在用户执行 `/new`、`/fork` 等命令时正确地重新绑定订阅。

### 8.2 Print（`modes/print-mode.ts`）

`runPrintMode(runtime, options)`：发送初始消息（及可选的一串 follow-up 消息），等待 Agent 完成，把结果打印到标准输出后退出。是 `-p` 参数和 CI 场景（第 1.2 节）的底层实现。

### 8.3 JSON Event Stream（`modes/json-event.ts`）

导出 `JsonAgentSessionEvent` 类型——把 `AgentSessionEvent` 结构化打印到 stdout，每行一个 JSON 对象，但不接受交互输入（单向流），适合只需要观察 Agent 执行过程、不需要在过程中插话的自动化监控场景。

### 8.4 RPC（`modes/rpc/`）

三个核心文件构成完整的双向协议实现：

- `rpc-mode.ts`：`runRpcMode(runtime)`，服务端主循环——解析 stdin 的 JSONL 命令、执行、把响应/事件写到 stdout
- `rpc-types.ts`：所有 `RpcCommand`/`RpcResponse`/`RpcExtensionUIRequest/Response` 的类型定义（对应第 8 章 rpc.md 里列出的 `prompt`/`steer`/`compact`/`get_state`/`get_entries` 等命令）
- `rpc-client.ts`：**可选**的类型化 TypeScript 客户端封装（`RpcClient`），给需要从 Node.js 侧以子进程方式驱动 `pi --mode rpc` 的场景用；跨语言客户端可以完全不用它，只需按 rpc.md 协议自己实现 JSONL 读写。

RPC 协议的一个关键设计约束（在 rpc.md 中反复强调）：**严格 JSONL**，只用 `\n` 分隔记录，明确指出 Node `readline` 因为也会在 Unicode 分隔符（`U+2028`/`U+2029`）处断行而"协议不兼容"——这是一个容易被忽略但会导致生产环境偶发解析错误的细节，文档特意把它点出来。

### 8.5 四种模式的能力对比

| 维度 | Interactive | Print | JSON Event | RPC |
|---|---|---|---|---|
| 交互方向 | 双向（键盘输入） | 单向（一次性 prompt） | 单向（只读事件流） | 双向（stdin 命令 + stdout 事件） |
| 典型消费者 | 人类用户 | Shell 脚本 | 监控/日志系统 | IDE、其他进程、跨语言客户端 |
| UI 能力（ctx.ui.custom 等） | 完整 | 无 | 无 | 部分降级 |
| 会话持久化 | 默认开启 | 可选 | 可选 | 可选（`--no-session`） |

---

## 9. 系统提示词的构建逻辑

`core/system-prompt.ts` 的 `buildSystemPrompt()` 是所有场景下 Agent "认知边界"的最终来源，输入包括：

```typescript
interface BuildSystemPromptOptions {
  customPrompt?: string;           // 完全替换默认系统提示词
  selectedTools?: string[];        // 决定"可用工具"章节列出哪些
  toolSnippets?: Record<string, string>;  // 每个工具的一行说明
  promptGuidelines?: string[];     // 追加的准则条目
  appendSystemPrompt?: string;     // 追加文本（对应 .pi/APPEND_SYSTEM.md）
  cwd: string;
  contextFiles?: Array<{ path: string; content: string }>;  // AGENTS.md 等
  skills?: Skill[];                // 技能渐进式披露（只放名称+描述）
}
```

两条路径分支值得注意：

1. **`customPrompt` 存在时**：仍然会追加项目上下文文件（`<project_context>` 块）和技能列表——即便完全自定义了系统提示词，项目规范和技能发现能力也不会丢失,除非选择的工具集里不含 `read`（因为技能需要靠 `read` 工具去加载完整内容，没有 `read` 就没必要告诉模型有哪些技能）。
2. **默认路径**：从内置模板出发，动态拼装"可用工具"清单——工具清单只列出**调用方提供了 `toolSnippets` 里对应说明**的工具，而不是简单地把 `selectedTools` 全部列出来，这样保证系统提示词与真正可调用的工具描述保持同步,不会出现"提示词里提到了一个工具但其实没有一行说明"的信息缺口。

这个函数是连接"资源系统"（技能、上下文文件）和"Agent 认知"的最后一道拼装工序,几乎所有前面章节讨论的资源类型最终都会汇入这里。

---

## 10. CLI 层：入口与参数解析

`main.ts` 是可执行文件的真正入口,职责是"把命令行参数翻译成 `createAgentSessionRuntime()`/`createAgentSession()` 的选项对象",本身不包含业务逻辑（注释里也明确写了"The SDK does the heavy lifting"）。关键处理链：

```
解析 argv (cli/args.ts)
  → 处理 auth 子命令 (cli/auth-command.ts) 或 package 子命令 (package-manager-cli.ts)
  → 读取管道 stdin（非 TTY 场景）
  → 处理 @file 引用 (cli/file-processor.ts)
  → 解析目标模型 (core/model-resolver.ts resolveCliModel)
  → 判定项目信任 (core/project-trust.ts resolveProjectTrusted)
  → 组装 AgentSessionRuntime (core/agent-session-runtime.ts)
  → 按 --mode 分发到 InteractiveMode / runPrintMode / runRpcMode / json-event
```

值得注意的实现细节：

- `readPipedStdin()`：仅当 `process.stdin.isTTY` 为假时才读取管道输入,保证交互模式启动时不会误读一个空的 TTY 输入流。
- `EXTENSION_LOAD_FAILURE_HINT`：扩展加载失败时的固定提示，引导用户用 `pi -ne`（no-extensions）跳过问题扩展启动，是一个很实用的"逃生舱"设计——避免一个坏扩展彻底堵死 Pi 的可用性。
- `cli/` 下的逻辑全部是**无状态的纯翻译层**：真正的会话状态、工具、扩展全部在 `core/` 里；这保证了 SDK 使用者绕过 `main.ts` 直接调用 `core/` API 时,行为和 CLI 启动完全一致,不会有"CLI 专属的隐藏逻辑"导致行为分叉。

---

## 11. 设计小结

把本文的两个主题串起来看:

1. **场景驱动分层**：Pi 的五大应用场景（交互终端、CI 自动化、SDK 集成、多 Agent 协作、沙箱化运行）之所以能够共存且互不冲突,根本原因是 `pi-coding-agent` 把"会话核心"（`AgentSession`/`AgentSessionRuntime`）和"交互壳"（`modes/`）彻底分离——同一套核心逻辑，套上 4 种不同的壳，就能覆盖几乎所有已知需求，无需为每个场景单独实现一套 Agent 逻辑。
2. **能力通过组合而非继承获得**：工具（`core/tools/`）、扩展（`core/extensions/`）、资源加载（`ResourceLoader`）三者相互独立、通过明确定义的接口协作，使得沙箱化（替换 `BashOperations`）、多 Agent 协作（扩展注册 `subagent` 工具）、自定义 UI（`ctx.ui.custom()`）都能在不修改核心代码的前提下实现。
3. **每个抽象边界都对应一个真实的可变维度**：`AgentSession` vs `AgentSessionRuntime` 对应"单次会话状态" vs "会话可被替换"；`ToolDefinition` vs `AgentTool` 对应"如何展示" vs "如何执行"；通知型事件 vs 拦截型钩子对应"想知道" vs "想控制"。这种"边界随可变维度而设的分层方式，是本文两处深入源码后能提炼出的最核心的工程经验。
