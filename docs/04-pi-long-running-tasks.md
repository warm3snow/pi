# Pi Agent 长任务处理：从工具超时到会话恢复的六层机制

> 系列第 04 篇（[系列索引](README.md)）。"长任务"在 Pi 里不是一个单一功能，而是多层机制叠加的结果：底层 Agent Loop 的多轮工具调用循环、`AgentSession` 的后处理驱动循环（自动重试/自动压缩）、工具层面的超时与流式反馈、用户插话打断、以及会话的持久化与恢复。本文按"从一次工具调用，到一整轮对话，到跨进程的会话生命周期"的粒度递进，逐层拆解这些机制的具体实现。

## 目录

1. ["长任务"的四种含义](#1长任务的四种含义)
2. [基础引擎：Agent Loop 的多轮工具调用](#2-基础引擎agent-loop-的多轮工具调用)
3. [会话层驱动循环：_runAgentPrompt 的 continue 循环](#3-会话层驱动循环_runagentprompt-的-continue-循环)
4. [机制一：自动重试（Auto-Retry）](#4-机制一自动重试auto-retry)
5. [机制二：自动压缩（Auto-Compaction）与超大单轮的 Split Turn](#5-机制二自动压缩auto-compaction与超大单轮的-split-turn)
6. [机制三：扩展驱动的追加轮次](#6-机制三扩展驱动的追加轮次)
7. [单个工具调用的长时间运行：超时、流式反馈、输出治理](#7-单个工具调用的长时间运行超时流式反馈输出治理)
8. [打断长任务：Abort、Steering、Follow-up](#8-打断长任务abortsteeringfollow-up)
9. [跨进程的长任务：会话持久化与当前的恢复边界](#9-跨进程的长任务会话持久化与当前的恢复边界)
10. [超长任务的分解：多 Agent 协作](#10-超长任务的分解多-agent-协作)
11. [六层机制总览](#11-六层机制总览)

---

## 1. "长任务"的四种含义

在深入机制之前先明确"长任务"在 Pi 里实际对应哪些具体问题，因为每一种都由不同层次的机制负责：

| 含义 | 具体表现 | 负责机制 |
|---|---|---|
| 一轮对话里工具调用很多次 | "读 20 个文件、跑 10 次 grep 才能回答" | Agent Loop 的多轮循环（第 2 节） |
| 单次 LLM 调用/工具调用瞬时失败 | 触发限流、服务端过载、网络抖动 | Auto-Retry（第 4 节） |
| 对话轮次很多、历史很长 | 连续开发一个功能聊了几十轮 | Auto-Compaction（第 5 节） |
| 单个工具本身执行很久 | 跑一个几分钟的测试套件、大文件读取 | 工具超时与流式反馈（第 7 节） |
| 任务跨越进程重启 | 中途崩溃、用户主动退出后想继续 | 会话持久化（第 9 节） |
| 任务本身逻辑上可拆分 | 大重构、跨仓库调研 | 多 Agent 协作（第 10 节） |

这六种问题在源码里对应完全不同的代码路径，理解"长任务不是一个整体功能"是读懂 Pi 长任务处理设计的第一步。

---

## 2. 基础引擎：Agent Loop 的多轮工具调用

最基础的一层是 `pi-agent-core` 的事件驱动循环（详见架构篇第 3 节）：一次 `agent.prompt()` 内部会持续"LLM 调用 → 执行工具 → 把工具结果喂回 LLM → 再调用"，直到某一轮 LLM 响应不再包含工具调用为止。这一层循环本身没有"最大轮数"之类的硬限制——只要模型持续产出工具调用，循环就持续进行,这是应对"一轮任务需要很多步骤才能完成"的基础能力。

但这层循环有一个边界：它假设"这轮对话本身能顺利进行到底"。真实场景中，一次完整任务往往会遇到：调用瞬时失败、上下文用满了需要腾空间、或者扩展想在结束后再追加一轮——这些都不是"多调几次工具"能解决的,需要一层更高的驱动逻辑。

---

## 3. 会话层驱动循环：`_runAgentPrompt` 的 continue 循环

`pi-coding-agent` 的 `AgentSession._runAgentPrompt()` 是这层更高驱动逻辑的落地位置,结构非常直白：

```typescript
private async _runAgentPrompt(messages: AgentMessage | AgentMessage[]): Promise<void> {
  this._isAgentRunActive = true;
  try {
    await this.agent.prompt(messages);
    while (await this._handlePostAgentRun()) {
      await this.agent.continue();
    }
  } finally {
    this._systemPromptOverride = undefined;
    this._flushPendingBashMessages();
    await this._emitAgentSettled();
  }
}
```

`_handlePostAgentRun()` 是这个循环的判断中枢，每次 `agent.prompt()`/`agent.continue()` 跑完一整轮（即底层 Agent Loop 判断"没有更多工具调用"而自然停下）之后，依次检查三件事，**任意一件命中就返回 `true`，驱动再来一轮 `agent.continue()`**：

```typescript
private async _handlePostAgentRun(): Promise<boolean> {
  const msg = this._lastAssistantMessage;
  if (!msg) return false;

  // 1) 这一轮是不是因为可重试的瞬时错误而结束的？
  if (this._isRetryableError(msg) && (await this._prepareRetry(msg))) {
    return true;
  }

  // 2) 是不是需要触发压缩（上下文超限，或响应被截断需要腾空间重试）？
  if (await this._checkCompaction(msg)) {
    return true;
  }

  // 3) 扩展在 agent_end 阶段有没有排队新消息？
  // ...（agent_end 之后 extension handler 可能调用 steer()/followUp() 入队）
}
```

三个检查项对应三种完全不同性质的"为什么这轮该继续"：**瞬时故障需要重试**、**容量不足需要腾空间**、**外部逻辑主动决定还要继续**。下面三节分别展开。

---

## 4. 机制一：自动重试（Auto-Retry）

### 4.1 两层重试:SDK 层 vs 会话层

Pi 的重试实际分两层，容易混淆但职责完全不同：

- **Provider/SDK 层重试**（`ProviderRetrySettings`：`timeoutMs`/`maxRetries`/`maxRetryDelayMs`，默认 `maxRetryDelayMs: 60000`）：在 `pi-ai` 发起单次 HTTP 请求时生效，处理连接级的瞬时故障（超时、连接被拒），对 Agent 逻辑完全透明——重试成功与否，Agent 层根本感知不到。
- **会话层自动重试**（`RetrySettings`：默认 `enabled: true`, `maxRetries: 3`, `baseDelayMs: 2000`）：在**一整轮 LLM 响应已经以 `stopReason: "error"` 结束**之后触发，处理"请求发出去了，但服务端返回了业务性错误"（过载、限流、5xx）的场景。

### 4.2 判定"可重试"的边界

```typescript
private _isRetryableError(message: AssistantMessage): boolean {
  // 上下文溢出走压缩逻辑，不走重试——重试无助于让内容变小
  if (isContextOverflow(message, this.model?.contextWindow ?? 0)) return false;
  return isRetryableAssistantError(message);
}
```

这条判断体现了一个重要的设计取舍：**"错误"不是一个单一类别，必须先分类再决定用什么机制应对**。过载/限流类错误的正确应对是"等一会再试"；上下文溢出类错误的正确应对是"先腾空间"——如果不做这个区分，对一个上下文溢出错误无脑重试只会得到同样的失败结果。

### 4.3 指数回退与状态管理

```typescript
private async _prepareRetry(message: AssistantMessage): Promise<boolean> {
  const settings = this.settingsManager.getRetrySettings();
  if (!settings.enabled) return false;

  this._retryAttempt++;
  if (this._retryAttempt > settings.maxRetries) {
    this._retryAttempt--;   // 保留计数，供最终失败事件展示"已尝试几次"
    return false;
  }

  const delayMs = settings.baseDelayMs * 2 ** (this._retryAttempt - 1);
  // 默认序列：2s → 4s → 8s
  this._emit({ type: "auto_retry_start", attempt: this._retryAttempt, maxAttempts: settings.maxRetries, delayMs, ... });

  // 把出错的响应从 agent 状态里移除（但仍保留在会话持久化历史中），再触发 continue()
  const messages = this.agent.state.messages;
  if (messages.length > 0 && messages[messages.length - 1].role === "assistant") {
    this.agent.state.messages = messages.slice(0, -1);
  }
  ...
}
```

几个细节值得注意：

- **计数在成功时立即清零**：一旦某次响应 `stopReason !== "error"`，`_retryAttempt` 立刻归零并发出 `auto_retry_end(success: true)`——这保证重试计数是"针对当前这一次故障的连续失败次数"，不会在一次长对话里跨多个独立故障累积，避免"用户开了很久的会话，早期偶发的一次错误耗尽了后面本该有的重试预算"。
- **移除出错消息但不销毁历史**：出错的 assistant 消息从**内存里的 agent 状态**（用于下一次 LLM 调用的上下文）中移除，但会话持久化文件里这条记录依然保留——保证 `/tree` 里能看到"这里出过一次错，重试后成功了"的完整轨迹，同时不让这条错误消息污染重试时发给模型的上下文。
- **失败预算耗尽后的收尾**：`msg.stopReason === "error" && this._retryAttempt > 0` 触发 `auto_retry_end(success: false)`，把最终失败原因暴露给 UI，而不是静默放弃。

### 4.4 摘要生成复用同一套重试预算

压缩/分支摘要调用 LLM 生成摘要文本时（见第 5 节），也复用同一份 `RetrySettings` 做重试（`_summarizationRetryCallbacks`）——这个设计避免了"正常对话轮次有重试保护，但压缩这一步本身的 LLM 调用一失败就直接判整个压缩失败"的不一致体验：一次瞬时的流断开不应该让整个长任务在压缩这一步彻底卡死。

---

## 5. 机制二：自动压缩（Auto-Compaction）与超大单轮的 Split Turn

`_checkCompaction()` 检查三种需要触发压缩的场景，对应架构篇已介绍的压缩机制在"长任务"场景下的具体触发路径：

### 5.1 场景一：上下文溢出但响应已完整（不重试，只腾空间）

```typescript
if (contextOverflow || recoverableLength) {
  const willRetry = assistantMessage.stopReason !== "stop";
  if (!willRetry) {
    return await this._runAutoCompaction("overflow", false);   // 压缩，但不 continue
  }
  ...
}
```

如果模型的响应虽然触发了"上下文即将溢出"的信号，但本身已经正常说完了（`stopReason: "stop"`），压缩只是为下一轮对话腾出空间，不需要重跑这一轮。

### 5.2 场景二：上下文溢出且响应被截断（压缩后重试一次）

```typescript
if (this._overflowRecoveryAttempted) {
  // 已经压缩重试过一次还是溢出，放弃，报错让用户知道
  return false;
}
this._overflowRecoveryAttempted = true;
// 从 agent 状态移除被截断的响应
const messages = this.agent.state.messages;
if (messages.length > 0 && messages[messages.length - 1].role === "assistant") {
  this.agent.state.messages = messages.slice(0, -1);
}
return await this._runAutoCompaction("overflow", willRetry);
```

`_overflowRecoveryAttempted` 是一个"只允许尝试一次"的开关——压缩本身也要占用上下文，如果压缩后依然溢出，说明问题不是"历史太长"，无限重试没有意义，此时会直接报错并给出明确建议（"减少上下文或换更大上下文窗口的模型"），而不是陷入死循环。

### 5.3 场景三：阈值触发的常规压缩（长对话最常见的情况）

```typescript
if (shouldCompact(contextTokens, contextWindow, settings)) {
  return await this._runAutoCompaction("threshold", false);
}
```

这是长任务里最常发生的一种——多轮持续对话，Token 用量逐步逼近 `contextWindow - reserveTokens` 阈值，自动触发压缩（架构篇第 8.2 节已详述压缩算法本身：定位裁剪点 → 生成结构化摘要 → 追加 `CompactionEntry`）。压缩完成后返回 `false`（不需要 `continue()`），因为常规压缩不是在"恢复一次失败"，只是在"清理空间"，当前这轮对话已经正常结束。

### 5.4 Split Turn：单轮本身就超预算的极端情况

一个"长任务"里某一单轮可能自己就非常大（比如一次工具调用返回了海量输出）。压缩算法（`packages/agent/src/harness/compaction/compaction.ts` 的 `findCutPoint`）在定位裁剪点时，如果发现理想的裁剪点恰好落在一个"正在进行的轮次"内部（`isSplitTurn: true`），会把这一轮拆成两段分别摘要：

```typescript
if (isSplitTurn && turnPrefixMessages.length > 0) {
  const historyText = ... // 历史部分正常摘要
  const turnPrefixResult = await generateTurnPrefixSummary(turnPrefixMessages, ...);
  summary = `${historyText}\n\n---\n\n**Turn Context (split turn):**\n\n${turnPrefixResult.value.text}`;
}
```

这保证了即便某一轮本身极其庞大，压缩依然能找到一个合理的裁剪点，而不是"要么保留整个巨型轮次、要么完全丢弃它"这种粗糙的二选一。

---

## 6. 机制三：扩展驱动的追加轮次

`_handlePostAgentRun()` 的第三个检查——"Agent Loop 会在发出 `agent_end` 之前排空 Steering/Follow-up 两个队列，但 `agent_end` 事件的扩展处理函数本身也可能在这个时机调用 `steer()`/`followUp()` 往队列里加新消息"。这类队列项需要触发额外一次 `continue()` 才能被消费。这是一个相对边缘但重要的设计闭环——保证"扩展在任务收尾阶段决定还要再做点什么"这种模式（比如一个 Git checkpoint 扩展在任务完成后自动追问"要不要提交这次改动"）能够正确驱动新一轮对话，而不需要用户手动再发一条消息。

---

## 7. 单个工具调用的长时间运行：超时、流式反馈、输出治理

对应第 1 节表格里"单个工具本身执行很久"这一类问题，机制集中在工具实现本身：

### 7.1 无默认超时，超时行为可配置

```typescript
const bashSchema = Type.Object({
  command: Type.String(...),
  timeout: Type.Optional(Type.Number({ description: "Timeout in seconds (optional, no default timeout)" })),
});
```

`bash`/`powershell` 工具**默认不设超时**——这是有意为之：很多合理的长任务（跑完整测试套件、编译大型项目）本身就需要几分钟甚至更久，强加一个默认超时反而会打断正常任务。超时是否需要、需要多久，交给调用方（LLM 自己根据任务性质判断）显式指定。指定后由 `setTimeout` + `killProcessTree` 落地：

```typescript
if (timeoutMs !== undefined) {
  timeoutHandle = setTimeout(() => {
    timedOut = true;
    if (child.pid) killProcessTree(child.pid);
  }, timeoutMs);
}
```

`killProcessTree` 而不是简单 `kill`——保证超时或中止时，命令可能派生出的整棵子进程树都会被清理，不留下孤儿进程（这一点与多 Agent 协作里子进程的 SIGTERM→SIGKILL 逐级升级策略是同一设计考量的复用）。

### 7.2 执行期间的流式进度反馈

长时间运行的命令不会让用户"盯着一个转圈等几分钟"，工具通过 `onUpdate` 回调节流地推送中间输出：

```typescript
const clearUpdateTimer = ...;
updateTimer ??= setTimeout(() => {
  updateTimer = undefined;
  emitOutputUpdate();
}, delay);
```

节流（而非每个字节都推送一次更新）的设计目的是控制 UI 重渲染频率——一个每秒输出几百行日志的命令,如果逐字节推送会淹没 TUI 的渲染循环；节流后的更新频率保证界面响应流畅，同时用户始终能看到"命令还在正常跑，这是目前的输出"。

### 7.3 输出治理：避免长任务的输出本身压垮上下文

如架构篇第 6.4 节提到的，`truncate.ts`/`output-accumulator.ts` 保证一个长时间运行命令产生的海量输出不会直接把当前轮次撑爆——超出 `DEFAULT_MAX_BYTES`/`DEFAULT_MAX_LINES` 的部分被截断，完整内容落盘到临时文件，工具结果里只保留"最后 N 行 + 完整输出的路径"。这是"长任务"在**空间维度**（不只是时间维度）上的防护——运行时间长的命令往往也伴随输出量大，两者需要分别应对。

---

## 8. 打断长任务：Abort、Steering、Follow-up

长任务不应该是"发起后就锁死等结果"，Pi 在 `AgentSession` 层暴露了三种介入方式：

### 8.1 Abort：彻底终止

```typescript
async abort(): Promise<void> {
  this.abortRetry();      // 取消正在等待的自动重试倒计时
  this.agent.abort();     // 中止底层 Agent Loop（包括正在执行的工具，通过 AbortSignal 传播）
  await this.waitForIdle();
}
```

`abortRetry()` 单独存在的原因：如果用户恰好在"等待重试倒计时"期间按下中止（比如已经等了 6 秒准备第 3 次重试），需要专门取消这个计时器，否则单纯 `agent.abort()` 不会影响一个尚未真正发起的、还在 `setTimeout` 里等待的重试。

### 8.2 Steering：插话但不打断当前工具批次

`steer()` 把消息排入队列，在**当前工具调用批次执行完、下一轮 LLM 调用发起前**被注入——用户可以在 Agent 跑到一半时补充信息（"记得也检查一下测试文件"），而不需要先中止再重新开始整个任务。

### 8.3 Follow-up：任务做完后追加

`followUp()` 排队的消息只有在"没有更多工具调用、也没有排队的 Steering 消息"时才会被处理——用于"任务做完之后再顺手做一件事"，不会打乱正在进行中的当前任务。

这三种机制配合本文第 3 节的驱动循环共同工作：`abort()` 会让 `_handlePostAgentRun()` 里已经在跑的自动重试/自动压缩全部中止,`waitForIdle()` 保证调用方拿到的是"Agent 真正停下来了"这个确定状态,而不是"发了中止信号但内部循环可能还没反应过来"的不确定状态。

---

## 9. 跨进程的长任务：会话持久化与当前的恢复边界

### 9.1 会话即 Append-Only 的持久化树

如架构篇第 8.1 节所述,`AgentSession` 的每条消息（包括本文提到的重试事件、压缩摘要）都会经 `SessionManager` 追加写入 JSONL 会话文件。这意味着：进程崩溃或用户主动退出，已经完成的对话历史**不会丢失**——重新用 `pi -c`/`--resume` 打开同一个会话文件，能看到崩溃前的完整轨迹,并从那个点继续对话。这是"长任务能跨越进程重启"的持久化基础。

### 9.2 当前生产路径的诚实边界

需要指出一个重要的现状：本文第 3-6 节描述的重试/压缩驱动循环，是在 `pi-coding-agent` 的 `AgentSession` 里**直接调用 `pi-agent-core` 的 `Agent` 类**（`agent.prompt()`/`agent.continue()`）实现的,这套逻辑本身完全是**内存态**的——一次未完成的重试等待、一次正在进行的压缩,如果进程在这个中间状态崩溃,这次"驱动循环"是不可恢复的,只能恢复到"最后一条已经持久化写入的消息"这个粒度,而不是"精确恢复到重试的第几次尝试"这种细粒度状态。

架构篇第 4 节提到的 `AgentHarness`（`packages/agent/src/harness/`）设计目标正是解决这个问题——把"运行中的操作"本身建模为可挂起、可持久化、可精确恢复的状态机（`SuspendedOperation`、`resume()`、`RunOutcome` 里的 `"suspended"` 分支）。但基于当前源码，这一层的具体落地（`AgentHarness` 类里 `prompt`/`compact`/`navigateTree`/`resume` 等核心方法）大多仍抛出 `HarnessNotImplemented`,是面向未来的骨架而非已经在 `pi-coding-agent` 生产路径里生效的实现。也就是说：**"会话历史不丢失"已经是现实,但"精确恢复到长任务中断的那个内部状态"仍是在建能力**,当前恢复粒度停留在"消息级"而非"操作级"。这个区分对于依赖 Pi 构建生产系统的场景很重要,不应被架构文档的设计意图误读为已经交付的能力。

---

## 10. 超长任务的分解：多 Agent 协作

对于"任务本身逻辑上可拆分、拆分后每部分都不需要感知整体历史"的场景（如"审查整个仓库的安全隐患"这种任务，本质是许多独立的子调查拼起来），最有效的"长任务"应对策略往往不是让单个 Agent 硬扛到底，而是拆解——参见《Pi 多 Agent 协作机制》一文：`subagent` 扩展把子任务分发给独立的 `pi` 子进程，各自拥有全新的、不受主任务历史拖累的上下文窗口，完成后只把提炼后的结论带回主线程。这本质上是"用空间换时间"的另一种表达：与其让一个 Agent 的上下文随着长任务不断膨胀最终依赖压缩硬撑,不如从任务分解阶段就避免膨胀发生。

---

## 11. 六层机制总览

把本文覆盖的机制按处理粒度从细到粗排列：

```
工具调用级   bash/powershell 超时 + killProcessTree、onUpdate 节流反馈、输出截断
   ↓
LLM 请求级   pi-ai 的 Provider/SDK 层重试（连接级瞬时故障，Agent 层无感知）
   ↓
对话轮级     AgentSession 会话层 Auto-Retry（业务性错误，指数回退 2s/4s/8s，最多 3 次）
   ↓
上下文级     Auto-Compaction（阈值触发 + 溢出触发 + Split Turn 应对超大单轮）
   ↓
用户交互级   Abort（彻底终止）/ Steering（插话不中断当前批次）/ Follow-up（排队追加）
   ↓
进程/会话级  会话持久化（JSONL 树，消息级可恢复）+ 多 Agent 协作（任务分解避免膨胀）
```

这六层各自独立、边界清晰，且**故意不做成一个大一统的"长任务管理器"**——重试解决瞬时性问题、压缩解决容量性问题、超时解决单点阻塞问题、多 Agent 协作解决可分解性问题，每种问题都有专门对应的、可以独立理解和调试的机制,这也是本系列文章反复出现的"核心极简、能力靠组合"设计原则在"长任务处理"这一具体问题域上的又一次体现。
