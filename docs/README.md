# Pi Agent 深度解析系列

基于 `pi-mono` 仓库源码整理的系列文章。每篇独立成文，但按下面的顺序阅读能获得从整体到细节、从核心到外围的完整认知路径。

## 阅读路径

### 第一部分：建立整体认知

**[01 · Pi Agent 架构与关键功能全解析](01-pi-agent-architecture.md)**
分层 monorepo 结构、Agent Loop 事件模型、AgentHarness 持久化执行模型、pi-ai 多模型抽象、工具系统、扩展体系、会话与压缩、四种运行模式、安全模型、遥测、评估体系概览。
→ 全系列的地图，先读这篇。

**[02 · Pi 主要应用场景与 pi-coding-agent 详细设计](02-pi-use-cases-and-coding-agent-design.md)**
五大落地场景（交互终端 / CI 自动化 / SDK 集成 / 多 Agent 协作 / 沙箱化运行），以及 `pi-coding-agent` 的源码级设计：`core`/`modes`/`cli` 三层边界、AgentSession 与 Runtime 分层、ResourceLoader、工具子系统、ExtensionRunner、四种模式实现、系统提示词构建。
→ 理解"Pi 拿来干什么"和"外壳包内部怎么组织"。

### 第二部分：核心机制专题

**[03 · Pi 多 Agent 协作机制与上下文管理详解](03-pi-multi-agent-collaboration.md)**
`subagent` 扩展如何用"扩展系统 + 操作系统进程"组合出多 Agent 能力；Single/Parallel/Chain 三种模式；上下文靠进程边界物理隔离而非应用层算法；三条窄信息通道（任务文本 / 最终文本 / `{previous}`）。

**[04 · Pi Agent 长任务处理：从工具超时到会话恢复的六层机制](04-pi-long-running-tasks.md)**
`_runAgentPrompt` 的 continue 驱动循环；Auto-Retry（两层重试的分工、指数回退、可重试判定边界）；Auto-Compaction 三种触发场景与 Split Turn；工具超时与流式反馈；Abort/Steering/Follow-up；会话恢复的现实边界。

**[05 · Pi Agent 的 Token 成本控制：为什么核心是 Prompt Cache 前缀稳定性](05-pi-token-efficiency-and-cost.md)**
四个计费桶的成本模型；Anthropic 三断点放置与 OpenAI cache key；**Deferred Tools**（把中途新增的工具放进 transcript 而非前缀，避免击穿缓存）；会话亲和性；`cache-stats.ts` 的缓存浪费量化；压缩的成本双刃剑；思考等级与异构模型路由。

### 第三部分：支撑体系

**[06 · Pi 模型与 Provider 治理层](06-pi-model-and-provider-governance.md)**
`ModelRuntime` 如何把 30+ Provider 统一成一个模型目录；四层模型来源的叠加顺序；鉴权解析链与 OAuth 刷新；`models.json` 的值解析（`!command` / `$ENV`）；**compat 兼容层**——如何在不改代码的前提下适配任意 OpenAI 兼容端点；成本分层定价与 `thinkingLevelMap`。

**[07 · Pi 会话数据模型与持久化格式](07-pi-session-data-model.md)**
会话为什么是树而不是线性列表；11 种 entry 类型全谱；`AgentMessage` 联合类型的四个 Pi 专属扩展；`retainedTail` 解决的具体问题；v1→v2→v3 版本迁移；哪些 entry 进入 LLM 上下文、哪些不进；如何编程解析会话文件。

**[08 · Pi 的交付层：TUI 渲染、包分发与配置治理](08-pi-delivery-layer.md)**
`pi-tui` 的差量渲染与两种渲染器、Focusable/IME 支持、同步输出；Pi Packages 的三种源与依赖规则（`peerDependencies` 为何必须用 `"*"`）；设置的全局/项目分层与 `pi config`；协议层三件套（`pi-protocol`/`server`/`client`）的远程化设计。

**[09 · Pi Evals 详细设计与应用示例：代码修复（让测试通过）评估](09-pi-evals-design-and-code-fix-example.md)**
`Harness` 接口的领域无关性；`pi-harness.ts` 的隔离与硬校验；`evalHarnessTable` 的 groupKey 配对机制；`summary.ts` 的配对差值统计；以及把这套方法论迁移到"手机导购 Agent"的完整示例（含"质量打分 vs 安全红线"的区分）。
→ 依赖前面各篇的概念，建议读完 01-08 再读。

**[10 · Pi Agent Telemetry 设计详解](10-pi-telemetry-design.md)**
厂商无关的 `TelemetryContext` 回调式契约；`NOOP` / `InMemoryTelemetryContext` 参考实现；类型化 Schema（`AI_TELEMETRY_SCHEMA` / `HARNESS_TELEMETRY_SCHEMA`）与 `startAiSpan` / `startHarnessSpan`；Adapter 一致性测试套件；跨包（`pi-telemetry` / `pi-ai` / `pi-agent-core` / 应用）的所有权切分；与评估体系里 Token/Latency/Cost 报告的衔接机制。

### 第四部分：实践指南

**[11 · Pi Subagent 编写指南：从编码型 Agent 到架构设计型 Agent](11-pi-writing-subagents.md)**
Agent 定义文件的 frontmatter 四字段与解析边界；系统提示词正文的六段式结构（角色边界/输入契约/工作策略/输出契约/交接约定/硬约束）；工具白名单的真实语义与选择矩阵；模型与思考等级的继承规则；官方四个编码 Agent 拆解 + 一个可直接使用的 `test-fixer`；任务文本五要素、工作流模板（`$@`/`$1`/`{previous}`）与三种模式的选择；单独跑一次子 Agent 的调试方法；以及**架构设计类 subagent**——为什么它的核心是"把断言变成带引用的断言"、ADR 输出契约、并行出候选 + 对抗性评审 + 收敛的编排、验收抽查清单。
→ 依赖 02 §6/§7（工具与扩展）和 03（subagent 机制），是它们的实践落地篇。

**[12 · Pi Coding Agent 提示词体系：主干、注入链路与六个改写入口](12-pi-coding-agent-prompts.md)**
最终 system prompt 的五个来源与拼装顺序（`buildSystemPrompt` 逐段拆解）；工具提示词贡献点（`promptSnippet` / `promptGuidelines`）；`AGENTS.md` 的逐级发现与 worktree 去重；skills 为什么只进索引；六个改写入口（AGENTS.md / APPEND_SYSTEM.md / SYSTEM.md / CLI / 模板 / 扩展）的作用边界与代价；斜杠命令模板的变量与加载规则；`before_agent_start` 的链式覆盖；压缩提示词这一套独立体系；以及怎么 dump 出最终 prompt。
→ 想把"让 Pi 按我的方式干活"落到具体文件时读这篇。

## 按问题查找

| 我想知道… | 看哪篇 |
|---|---|
| Pi 整体是怎么分层的 | 01 |
| Agent Loop 会发出哪些事件 | 01 §3 |
| Pi 能用在什么场景 | 02 §1 |
| `AgentSession` 和 `AgentSessionRuntime` 有什么区别 | 02 §4 |
| 扩展怎么写、能拦截什么 | 02 §7 |
| 多个 Agent 之间怎么传信息 | 03 §6 |
| 上下文为什么不会被子 Agent 污染 | 03 §5 |
| LLM 调用失败了会怎样 | 04 §4 |
| 上下文满了会怎样 | 04 §5 / 01 §8 |
| 长时间跑的命令会不会被 kill | 04 §7 |
| 怎么降低 token 成本 | 05（全篇） |
| 为什么加一个工具会让费用飙升 | 05 §3 |
| 缓存到底命中了没有 | 05 §5 |
| 怎么接入自建/本地模型 | 06 §4 |
| 某个 OpenAI 兼容端点报错怎么办 | 06 §5 |
| 会话文件怎么解析 | 07 |
| `/tree` 分支是怎么存的 | 07 §2 |
| 怎么分发我写的扩展 | 08 §3 |
| 怎么远程连接一个 Agent 会话 | 08 §5 |
| 怎么衡量我的改动有没有让 Agent 变好 | 09 |
| 怎么写一个自己的 subagent | 11（全篇） |
| subagent 的 frontmatter 能写哪些字段 | 11 §2 |
| 子 Agent 为什么改了不该改的文件 | 11 §4 |
| 编码类 subagent 怎么写 | 11 §6 / §7 |
| 委派时任务文本该写什么 | 11 §8.1 |
| 子 Agent 行为不对怎么调试 | 11 §9 |
| 架构设计类 subagent 怎么写 | 11 §12-§14 |
| 怎么把 subagent 分发给团队 | 11 §17 |
| 模型每轮到底看到了什么 | 12 §1 / §6 |
| 该把项目规矩写到哪个文件 | 12 §7 / §14 |
| `SYSTEM.md` 和 `APPEND_SYSTEM.md` 有什么区别 | 12 §8 |
| 自定义工具为什么从不被调用 | 12 §3 / §14 |
| `AGENTS.md` 改了没生效怎么排查 | 12 §4 / §14 |
| 斜杠命令模板的变量怎么写 | 12 §9 |
| 怎么看到最终的 system prompt | 12 §12 |
| 压缩摘要的提示词在哪 | 12 §11 |
| 怎么把 Pi 接入 OpenTelemetry/Sentry | 10 §11 |
| 一次 LLM 调用能观测到哪些字段 | 10 §6.1 |
| Harness 的重试/压缩/工具执行怎么在遥测里表达 | 10 §6.2 |
| 遥测为什么不能塞进会话记录 | 10 §9.1 |

## 说明

- 所有内容基于仓库源码与 `packages/*/docs/` 官方文档核实。
- 涉及"设计意图已存在但实现尚未落地"的部分（如 `AgentHarness` 的操作级恢复），文中会明确标注边界，不将设计骨架描述为已交付能力。
- 代码引用格式为 `包名/路径`，可直接在仓库中定位。
