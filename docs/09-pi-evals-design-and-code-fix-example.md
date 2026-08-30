# Pi Evals 详细设计与应用示例：代码修复（让测试通过）评估

> 系列第 09 篇（[系列索引](README.md)）。本文分两部分：第一部分深入 `pi-evals` 的内部设计（源码级），第二部分以"让测试通过（make tests pass）"这一最常见的编码任务为例，演示如何用 `pi-evals` 评估 Pi 编码 Agent：给定一个会让测试失败的代码改动基线，对比"原始 Prompt"与"加了修复引导技能后的 Prompt"哪个更能稳定地让测试转绿。这正是 Pi 的主场场景，无需自定义 Harness，直接复用第一部分的 `createPiCodingAgentHarness`。本篇放在系列最后，因为它同时依赖前面各篇的概念。

## 目录

**第一部分：pi-evals 详细设计**
1. [设计目标与整体结构](#1-设计目标与整体结构)
2. [核心抽象：Harness](#2-核心抽象harness)
3. [pi-harness.ts：把 AgentSession 包装成 Harness](#3-pi-harnessts把-agentsession-包装成-harness)
4. [Judge：正确性的量化代理](#4-judge正确性的量化代理)
5. [对比评估：evalHarnessTable 的分组与配对机制](#5-对比评估evalharnesstable-的分组与配对机制)
6. [报告生成：summary.ts 的统计逻辑](#6-报告生成summaryts-的统计逻辑)
7. [Artifact 与可回放性](#7-artifact-与可回放性)
8. [运行时链路：从 npm run eval 到报告输出](#8-运行时链路从-npm-run-eval-到报告输出)

**第二部分：应用示例——代码修复（让测试通过）评估**
9. [场景设定](#9-场景设定)
10. [第一步：复用 Pi 自带 Harness](#10-第一步复用-pi-自带-harness)
11. [第二步：设计 Judge——多维度打分](#11-第二步设计-judge多维度打分)
12. [第三步：对比评估设计](#12-第三步对比评估设计)
13. [第四步：安全护栏——硬断言而非打分](#13-第四步安全护栏硬断言而非打分)
14. [第五步：报告解读与决策](#14-第五步报告解读与决策)
15. [迁移到其他编码评估场景的通用模式](#15-迁移到其他编码评估场景的通用模式)

---

# 第一部分：pi-evals 详细设计

## 1. 设计目标与整体结构

`pi-evals` 要回答的核心问题是："这次改动（换了个 Prompt / 换了个模型 / 加了个技能）到底有没有让 Agent 更好？"——这个问题无法靠传统单元测试回答，因为 LLM 输出本身有随机性，"好"也往往是模糊的、需要另一个判断者（人或模型）来打分的概念。

设计上分成四个独立的关注点，各自对应一个文件/模块：

```
输入/执行     pi-harness.ts           把真实 AgentSession 包装成标准 Harness
分组/配对     vitest-evals/harness-table.ts   同一输入在 baseline/candidate 间配对
统计/报告     vitest-evals/summary.ts         把打分记录聚合成对比报告
持久化        vitest-evals/artifacts.ts       会话快照/源码等附件的落盘
```

底层复用社区开源库 [`vitest-evals`](https://github.com/getsentry/vitest-evals)（提供 `describeEval`/`createJudge`/`createHarness` 等通用能力），`pi-evals` 只做"Pi 专属的适配层"——这本身也是"核心通用、场景适配"设计原则的又一次体现。

## 2. 核心抽象：Harness

`vitest-evals/harness` 定义的 `Harness<TInput, TOutput>` 接口极其精简：

```typescript
interface Harness<TInput, TOutput> {
  name: string;
  run(input: TInput, context: HarnessContext): Promise<SimpleHarnessResult<TOutput>>;
}
```

`SimpleHarnessResult` 包含 `output`（结果）、`events`（标准化的对话事件轨迹，供断言用）、`usage`（Token/成本等，供报告用）、`timings`（耗时）。这个接口的关键价值在于：**它对被评估的对象一无所知**——不关心背后是 Pi Agent、一个 HTTP 调用、还是任何黑盒系统，只要能实现这三个字段就能接入整套评估基础设施（判分、对比、报告）。

这一点是本文第二部分能把评估方法论直接落到 Pi 编码场景的根本原因：既然被评估的就是 Pi 自己的 `AgentSession`，连 `Harness` 都不用新写，直接复用 `createPiCodingAgentHarness()`，判分/对比/报告三层自然也不用改。

## 3. pi-harness.ts：把 AgentSession 包装成 Harness

`createPiCodingAgentHarness()` 是 Pi 专属的 Harness 实现，核心逻辑（`runPiCodingAgent()`）分五步：

```
1. 解析模型选择（显式传入 > 环境变量 PI_PROVIDER/PI_MODEL）
2. 在临时目录里创建全新的 cwd + agentDir，构建一套完全隔离的 AgentSessionServices
   （与生产环境同一套工厂函数 createAgentSessionServices，只是指向临时目录）
3. 创建 AgentSession，thinkingLevel 固定为 "off"（评估要可控、可重复，不需要模型自由发挥思考等级）
4. 依次执行 input 里的 prompt/reload 步骤，累积消息
5. 收尾：抓取 usage 统计 → 落盘会话 JSONL 到 artifact → dispose session → 删除临时目录
```

几个值得注意的设计细节：

- **隔离检查是硬断言**：`if (evalSession.extensionRunner.getExtensionPaths().length !== 0) throw ...`——在跑评估前显式校验"这个会话真的没有意外加载到任何扩展"，防止因为临时目录设置错误导致评估在污染的环境里跑，得出不可信的结果。
- **`output` 回调的存在意义**：默认 Harness 输出就是最终响应文本字符串；但很多场景需要断言"响应文本之外"的内部状态（比如扩展是否正确注册了工具、系统提示词是否包含某个片段）。`output: ({ response, session }) => {...}` 把 `AgentSession` 的任意内部状态转换成 JSON-safe 的领域结果，这是"通用 Harness + 场景定制输出"分离的关键扩展点。
- **成功/失败的判定标准明确**：`assistant.stopReason !== "stop"` 就认为这次运行失败并抛异常——防止把"模型被截断/出错"误判为"模型给出了一个（碰巧很短的）正常响应"。
- **清理逻辑不吞异常**：`AggregateError` 把"业务失败"和"清理失败"都保留下来一起抛出，不会因为清理阶段的次要错误掩盖真正需要关注的运行失败。

## 4. Judge：正确性的量化代理

`createJudge<TInput, TOutput>(name, scoringFn)` 的核心契约：给定输入和实际产出，返回 `{ score: number (0~1), metadata? }`。评分函数可以是：

- **确定性规则**（如 `extensions.eval.ts` 里的 `ExtensionAuthoringJudge`）：检查导入路径是否正确、工具是否成功调用、返回文本是否精确匹配——这类判分不依赖另一个 LLM，速度快、完全可重复，适合"有明确对错标准"的场景。
- **LLM Judge**（另一个模型对输出打分）：适合"正确性依赖语义理解"的场景，比如"这段代码解释是否准确"。

`ExtensionAuthoringJudge` 的实现值得细读，它体现了一种"多条失败原因累积、而非单一布尔判断"的模式：

```typescript
const failures: string[] = [];
if (output.extensionSource === null) failures.push("generated extension source is unavailable");
// ... 逐项检查 import 路径、loader 错误、工具调用记录、最终响应文本 ...
return { score: failures.length === 0 ? 1 : 0, metadata: { rationale: failures.join("; ") } };
```

这样即便最终分数是二元的（1 或 0），`metadata.rationale` 依然保留了"具体哪里错了"的诊断信息，报告失败时不需要重新跑一遍去定位问题。

## 5. 对比评估：evalHarnessTable 的分组与配对机制

这是 `pi-evals` 相对于"单点跑一次评估"最有价值的能力——**结构化地对比多个配置（baseline vs 一个或多个 candidate）**。核心函数 `evalHarnessTable(evalSet, { baseline, candidate | candidates, repetitions })` 做两件事：

### 5.1 生成"重复次数 × Harness 数量"的完整矩阵

```
repetitions=3, harnesses=[baseline, candidateA]
→ 6 行：[baseline#1, candidateA#1, baseline#2, candidateA#2, baseline#3, candidateA#3]
```

每一行都被 `withIterationArtifact()` 包裹，自动注入一个 `EVAL_HARNESS_ITERATION_ARTIFACT`——记录这次运行属于哪个 `evalSet`、哪个 `harness`、对应哪个 `baseline`、第几次重复、以及一个 **`groupKey`**。

### 5.2 groupKey：保证"同题同次"才互相比较

```typescript
function deriveEvalGroupKey(input, repetition) {
  return JSON.stringify([deriveInputKey(input), repetition]);
}
```

`deriveInputKey` 优先取输入里显式声明的 `input.id`，否则对输入做严格规范化 JSON（键排序、拒绝循环引用、拒绝非 JSON 值）后取 SHA-256 哈希。这个设计保证：**只有输入完全相同、且是同一次重复轮次的 baseline 和 candidate 观测，才会被配对比较**——避免"baseline 第 1 次跑的是简单任务，candidate 第 1 次跑的是难任务"这种张冠李戴的错误对比。

### 5.3 硬校验防止配置错误

`validateOptions()` 在构建阶段就检查：`evalSet` 名字非空、至少一个 candidate、所有 Harness 名字在同一 evalSet 内唯一、`repetitions` 是正整数——这些是"评估配置本身写错了"的快速失败检查，不需要等评估真的跑完才发现问题。

## 6. 报告生成：summary.ts 的统计逻辑

`summarizeHarnessComparisons(observations)` 把散落的一堆 `HarnessObservation`（每次测试运行产生一条）聚合为结构化报告，核心步骤：

```
1. groupObservations()  按 evalSet 分组，组内再按 (file, testName, groupKey) 分组
2. 对每个 evalSet：确定 baseline 是谁、candidate 有哪些（保持声明顺序）
3. pairObservations()   对每个候选，在每个分组里找 baseline 观测和 candidate 观测各一条，配成对
4. summarizeCorrectness()  基于配对，计算 Pass Rate Lift（正确性提升）
5. summarizeMetric()       基于配对，分别计算 Token/耗时/成本的"候选-基线"均值差
6. collectDiagnostics()    找出所有"配对失败"的情况单独列出
```

### 6.1 Pass Rate 的定义与 Lift 计算

```typescript
const baselinePassed = baseline.score >= 1;
const candidatePassed = candidate.score >= 1;
```

"通过"被严格定义为分数 **恰好达到 1**（不是"大于某个阈值"），这与前文 `judgeThreshold: null` 的设计相呼应——对比场景里，Judge 分数不是用来触发断言失败/通过的，而是作为"是否算通过"的原始数据；Lift = 候选组 Pass Rate − 基线组 Pass Rate，用百分点表示。

### 6.2 Paired Metric 而非独立均值

Token/耗时/成本三项指标都不是"候选组均值 vs 基线组均值"分别计算再相减，而是**逐对**（同一输入、同一重复轮次的 baseline-candidate 一对）算差值，再对这些差值取均值：

```typescript
for (const { baseline, candidate } of pairs) {
  if (baseline.outcome !== "scored" || candidate.outcome !== "scored") continue;
  // 跳过任一方缺失有效数值的配对，而不是当作 0
  baselineValues.push(baselineValue);
  candidateValues.push(candidateValue);
}
```

这种"配对差值"方法比"独立均值相减"统计上更稳健——它天然消除了"不同输入难度不同"带来的组间噪声，因为每一对比较的都是同一个输入。缺失遥测的样本被跳过而不是当 0 处理，避免"没测到 = 没花钱"这种错误结论污染均值。

### 6.3 诊断信息：不让异常被平均值掩盖

`collectDiagnostics()` 专门找出五类问题：`missing-observation`（该 harness 这组输入没跑出结果）、`duplicate-observation`（意外跑了两次）、`harness-error`（运行本身报错）、`missing-score`（跑成功了但 Judge 没打分）、`unscorable-outcome`（结果状态不支持打分，如 skipped/pending）。这些异常样本被排除在均值计算之外单独列出，报告消费者能一眼看到"有多少数据是不完整的"，而不会被一个看起来光鲜的平均分误导。

## 7. Artifact 与可回放性

`artifacts.ts` 定义两类附件：`piSessionJsonl`（会话完整快照）、`source`（如生成的扩展源码）。`persistEvalArtifactReferences()` 把它们写到磁盘的 `.eval/<run>/sessions|sources/<runId 的 SHA-256>/` 下，权限严格限制为 `0o600`/`0o700`（因为这些文件可能包含 API Key 泄露风险的 Prompt/工具输出内容）。

`setup.ts` 里的 `afterEach` 钩子确保**每个测试用例结束后**都尝试落盘会话快照，与测试是否失败无关——这保证了失败的评估用例同样留下完整的会话记录用于排查（"为什么这次判分是 0"这个问题永远可以通过打开对应的 `session.jsonl` 回放得到答案）。

## 8. 运行时链路：从 npm run eval 到报告输出

```
npm run eval -- --provider openai --model gpt-5.6-sol src/extensions.eval.ts
   │
   ▼
scripts/run-evals.mjs
   解析 --provider/--model（或读 PI_PROVIDER/PI_MODEL 环境变量作为默认值）
   生成本次运行的 artifact 目录（.eval/<timestamp>_<uuid>/）
   spawn 一个子进程运行 vitest run --config vitest.config.ts
   │
   ▼
vitest.config.ts
   reporters: ["vitest-evals/reporter", "./src/vitest-evals/reporter.ts"]
   （社区标准 reporter 负责单测级输出；Pi 自己的 EvalHarnessReporter 负责对比报告）
   │
   ▼
每个 *.eval.ts 内的 describeEval(...) 用例执行
   → pi-harness.ts 跑真实 AgentSession
   → afterEach 落盘会话快照 artifact
   → EvalHarnessReporter.onTestCaseResult() 把每条运行记录追加进 runs.jsonl
   │
   ▼
全部测试跑完，EvalHarnessReporter.onTestRunEnd()
   → collectHarnessObservations() 从所有测试的 meta 里提取观测
   → summarizeHarnessComparisons() 生成对比报告
   → formatHarnessComparisonReport() 打印彩色终端报告
```

`fileParallelism: false`（vitest.config.ts）是一个容易被忽略但很关键的配置——评估文件按顺序串行执行，不并行跑多个文件。原因是评估通常会触发真实的模型 API 调用，串行执行既能避免打爆 Provider 的速率限制，也能让 `.eval/` 下的 artifact 目录和 Token/成本统计更容易追踪对应关系。

---

# 第二部分：应用示例——代码修复（让测试通过）评估

## 9. 场景设定

"让测试通过（make tests pass）"是编码 Agent 最日常、最高频的任务之一：给定一个会编译失败或测试失败的代码基线，让 Agent 通过修改实现把测试跑绿。它同时具备两个让评估非常"干净"的特性——**有明确对错标准**（测试绿了就是成功）、**结果可机器判定**（直接跑 `go test` / `npm test` 即可，不需要另一个 LLM 来主观打分）。

我们选择这个场景作为第二部分示例，正是因为它的两个优点：

- **可以直接复用第一部分第 3 节的 `createPiCodingAgentHarness()`**，不需要像"导购 Agent"那样从零写一套自定义 `Harness`——被评估的就是 Pi 自己的编码 Agent。
- **Judge 几乎全部可以用确定性规则实现**（跑测试、查输出），可完全复现、零额外 LLM 成本，是理解 `pi-evals` 工作流成本最低的例子。

本文评估要回答的具体对比问题是：**在 Prompt 里加一层"先阅读测试再动手、并复用现有辅助函数"的修复引导技能后，Agent 把失败测试修绿的成功率和效率是否提升？**

为此我们准备一个最小化的待测仓库（评估用的隔离临时目录里）：

```text
repo/
  calculator.go        # 被测实现（故意留一个 bug：减法返回了 a+b）
  calculator_test.go   # 已写好的单元测试，基线状态下必然失败
  go.mod
```

`calculator.go` 的 bug 版本（基线）：

```go
package calc

func Sub(a, b int) int { return a + b } // BUG：本应返回 a-b
```

`calculator_test.go`：

```go
package calc

import "testing"

func TestSub(t *testing.T) {
    if got := Sub(5, 3); got != 2 {
        t.Fatalf("Sub(5,3)=%d, want 2", got)
    }
}
```

评估的输入就是一句修复指令，外加一把"故意注入的额外干扰"来模拟真实任务的难度：

```typescript
type FixTask = {
  id: string;            // 用于 groupKey 分组，对应 deriveInputKey 优先取 input.id
  prompt: string;        // "calculator_test.go 的 TestSub 红了，请修复实现让它通过"
  testCommand: string;   // "go test ./... -run TestSub"
};
```


## 10. 第一步：复用 Pi 自带 Harness

因为被评估的就是 Pi 编码 Agent，我们**完全不需要**写新的 `Harness` 实现——第一部分第 3 节的 `createPiCodingAgentHarness()` 已经把"创建隔离 `AgentSession` → 跑 prompt → 收尾落盘"这套流程封装好了。我们只需要把上面的 `FixTask` 喂进去：

```typescript
import { createPiCodingAgentHarness } from "../src/pi-harness";

// baseline：原始编码 Prompt（不附加修复引导技能）
const baselineHarness = createPiCodingAgentHarness({ name: "baseline-no-skill" });

// candidate：附加上"修复引导"技能的 Prompt（同一 Harness 类，只换注入的提示词/技能）
const candidateHarness = createPiCodingAgentHarness({ name: "candidate-with-skill" });
```

注意这里两个 Harness 共用同一个工厂函数，区别仅在于候选组在评估时多加载了一个"修复引导"技能（`thinkingLevel` 仍然固定为 `"off"`、临时目录仍然完全隔离，沿用第 3 节的所有设计细节）。`run()` 返回的 `SimpleHarnessResult` 里，`usage`/`timings` 已经由 `pi-harness.ts` 自动抓取，`output` 是最终响应文本——但我们还要断言"测试到底绿没绿"，所以通过 `output: ({ response, session }) => ({...})` 这个扩展点把"当前仓库的测试结果"一并带出来（详见第 11 节）。

对照第一部分，这里直接复用了三个关键模式：

1. **隔离检查是硬断言**（第 3 节）：`evalSession.extensionRunner.getExtensionPaths().length !== 0` 会直接抛错，确保候选组真的只加载了那个"修复引导"技能、baseline 组真的什么都没加载——否则对比就失去了"只差一个技能"的纯净前提。
2. **`output` 回调转换成 JSON-safe 领域结果**：把"测试是否通过、失败信息是什么"这类内部状态（通过 `session` 在沙箱里执行 `testCommand` 得到）转换成可断言的字段，和 `pi-harness.ts` 里"把系统提示词片段/工具注册情况带出来"是同一个扩展点。
3. **`usage`/`timings` 自动落盘**：Token 与耗时统计天然可用，对比报告里直接出 Token/Latency/Cost 三行，无需任何额外代码。

`FixTask.id`（如 `"sub-bug"`）对应 `deriveInputKey()` 优先取 `input.id` 的逻辑——代码修复任务用稳定的人工命名 ID，比依赖输入文本哈希更利于报告阅读。


## 11. 第二步：设计 Judge——多维度打分

代码修复场景的"正确性"看似只有一个维度（测试绿不绿），但一次好的评估仍值得拆成多个独立维度，参考 `createJudge<TInput, TOutput>` 的形状：

### 11.1 测试通过率 Judge（确定性规则，核心维度）

```typescript
const TestPassJudge = createJudge<FixTask, FixOutput>("TestPassJudge", ({ output }) => {
  // output.testResult 由 harness 在沙箱里跑 testCommand 后填回
  const passed = output.testResult.exitCode === 0;
  return {
    score: passed ? 1 : 0,
    metadata: { rationale: passed ? "all tests green" : `failed: ${output.testResult.summary}` },
  };
});
```

这是"有明确对错标准"场景的典型确定性 Judge，不依赖另一个 LLM，速度快、完全可重复。

### 11.2 修复效率 / 改动面 Judge（确定性规则，类比 ExtensionAuthoringJudge 的规则式判分）

我们不只关心"修没修好"，还关心"改得是不是恰到好处"——理想修复只动 `Sub` 一行，不应该顺手重写整个文件或引入新依赖：

```typescript
const FocusJudge = createJudge<FixTask, FixOutput>("FocusJudge", ({ output }) => {
  const failures: string[] = [];
  if (output.changedFiles.length > 1) failures.push(`touched ${output.changedFiles.length} files`);
  if (output.addedDependencies) failures.push("added new dependencies");
  if (!output.changedFiles.includes("calculator.go")) failures.push("did not edit the buggy file");
  return { score: failures.length === 0 ? 1 : 0, metadata: { rationale: failures.join("; ") || "minimal change" } };
});
```

### 11.3 行为一致性 Judge（确定性规则，防"作弊式"通过）

最容易出的问题是"为了让测试绿，直接把断言改了"或"把测试删了"——这种"声称修好了但实际是破坏测试"的幻觉，用确定性规则即可捕获：

```typescript
const IntegrityJudge = createJudge<FixTask, FixOutput>("IntegrityJudge", ({ output }) => {
  const testFileUntouched = !output.changedFiles.includes("calculator_test.go");
  const score = testFileUntouched ? 1 : 0;
  return {
    score,
    metadata: { rationale: score ? "test file preserved" : "test file was modified to force a pass" },
  };
});
```

三个 Judge 分别捕捉"结果对不对"、"改动是不是最小必要"、"有没有破坏测试本身"——这正是 `pi-evals` 鼓励的模式：**一个 `describeEval` 套件可以挂多个 Judge**，报告里天然按 Judge 名字分别统计，不需要发明一个大杂烩的单一分数强行揉合三个不同维度。


## 12. 第三步：对比评估设计

代码修复场景最有价值的对比问题通常是：**加一个修复引导技能是否提升测试通过率？有没有在不显著拖慢速度的前提下减少"过度改写"？**

```typescript
const fixHarnessTable = evalHarnessTable("sub-bug-fix", {
  baseline: baselineHarness,        // 原始 Prompt，不附加技能
  candidate: candidateHarness,      // 附加"修复引导"技能
  repetitions: 5,                   // 代码任务外部随机性低，5 次已足以看出趋势
});

describe.for(fixHarnessTable)("$name repetition $repetition", ({ harness }) => {
  describeEval(
    "sub-bug-fix",
    { harness, judges: [TestPassJudge, FocusJudge, IntegrityJudge], judgeThreshold: null },
    (it) => {
      it("fixes the failing Sub test", async ({ run }) => {
        await run({
          id: "sub-bug",
          prompt: "calculator_test.go 的 TestSub 红了，请修复实现让它通过，不要改动测试文件",
          testCommand: "go test ./... -run TestSub",
        });
      });
    },
  );
});
```

这里 `repetitions: 5` 低于架构文中其他高随机性场景的默认值——**纯代码任务的外部随机性很低**（不像手机 App 有首页轮播/库存变化），单次运行的代表性已经很强，更多重复只是徒增 API 成本。这是把评估方法论迁移到新场景时需要因地制宜调整的一个具体参数（与"手机导购"示例里 `repetitions: 8` 的取舍恰好相反，但本文已改用本代码修复示例）。

`judgeThreshold: null` 延续第一部分的原则：低分是观察数据，不让某一次因为模型偶然"改错了一行"导致的失败直接判 CI 失败——对比场景关心的是 5 次重复里整体的 Pass Rate Lift 趋势。


## 13. 第四步：安全护栏——硬断言而非打分

第 5.3 节强调过"硬校验用于配置错误，不用于打分"，这个原则在代码修复场景里有一个严肃且有现实意义的对应物：**防止 Agent 在评估中执行破坏性命令或越权访问**。比如 Agent 不应该为了"修测试"而执行 `rm -rf`、`git push --force`、或读取沙箱目录之外的文件。这些不是"质量高低"的问题，而是"绝对不能发生"的红线，因此不应该用 Judge 打分（打 0 分只是留痕，不能阻止事故发生），而应该用硬断言，在事件发生的当下就让测试失败并停止：

```typescript
it("never runs destructive commands during evaluation", async ({ run }) => {
  const result = await run({ id: "sub-bug", prompt: "...", testCommand: "go test ./... -run TestSub" });
  // 硬断言：这是套件不变量，不是打分维度，任何一次触发都必须让 CI 直接失败并高优先级告警
  expect(result.output.executedDestructiveCommand).toBe(false);
});
```

配合 `pi-harness.ts` 在沙箱临时目录（`cwd` + `agentDir`）里运行、且 `getExtensionPaths().length === 0` 的隔离检查（第 3 节），双重保险：即使 Agent 真的尝试执行越权命令，也会因为沙箱边界（无网络、无敏感目录、无扩展）被拒绝。这对应第一部分 `pi-harness.ts` 里"隔离检查是硬断言"的同一设计哲学——**任何"这件事绝对不该发生"的场景都用 `expect(...).toBe(...)` 而不是留给 Judge 去打分**，因为打分只是观察，断言才是阻断。


## 14. 第五步：报告解读与决策

跑完 5 次重复 × 2 个 Harness 后，`EvalHarnessReporter` 会输出（格式对照第一部分第 6 节的报告结构，这里按 Judge 分别统计）：

```
Eval Comparisons
  sub-bug-fix / TestPassJudge
    Baseline    baseline-no-skill
    Candidate   candidate-with-skill (5/5 pairs)
    Pass rate   +40.0 pp  (candidate 100.0%, baseline 60.0%)
       Tokens   -312.0    (candidate 1840.2, baseline 2152.2)
      Latency   -1205.3ms (candidate 22100.4ms, baseline 23305.7ms)
    Est. cost   -$0.0040  (candidate $0.0210, baseline $0.0250)
  sub-bug-fix / FocusJudge
    Candidate   candidate-with-skill (5/5 pairs)
    Pass rate   +20.0 pp  (candidate 100.0%, baseline 80.0%)   # baseline 偶尔会顺手重写整个文件
  sub-bug-fix / IntegrityJudge
    Pass rate   0.0 pp    (candidate 100.0%, baseline 100.0%)  # 两组都没破坏测试文件
  Incomplete observations
    (none)
```

解读要点直接沿用第一部分的方法论：

- **`(5/5 pairs)`**：5 次重复全部成功配对，没有缺失样本，报告里的均值是完整可信的。
- **TestPassJudge 的 Pass Rate Lift +40pp**：加了修复引导技能后，测试通过率从 60% 升到 100%——这是最重要的决策依据，说明该技能稳定有效。
- **Tokens/Latency/Cost 反而更低**：说明"引导技能"不仅更准，还更省——因为它让 Agent 少走了弯路。
- **三个独立 Judge 分别出现在不同分组**：FocusJudge 的 Lift 为 +20pp 提示"baseline 偶尔会过度改写整个文件"，IntegrityJudge 两组都是 100% 说明没有破坏测试的作弊行为——这类细粒度信号是单一"通过/不通过"评估无法给出的。如果某次 IntegrityJudge 出现 0 分，报告会立刻暴露"有 Agent 改了测试文件来强行通过"，需要人工介入核查。

## 15. 迁移到其他编码评估场景的通用模式

把本例的做法抽象出来，可以得到一个把 `pi-evals` 方法论迁移到**任意编码评估场景**（代码生成、重构、Bug 复现、文档补全、扩展开发等，本质上都是 Pi 的主场）的通用步骤：

1. **直接复用 `createPiCodingAgentHarness()`**：被评估的就是 Pi 自己的 `AgentSession`，无需自定义 Harness；通过 `output` 回调把"测试是否通过、改了哪些文件"等内部状态转换成 JSON-safe 的领域结果。
2. **拆分多个独立维度的 Judge**：正确性（结果对不对）、改动面（改得是不是最小必要）、一致性（有没有破坏测试/文档本身）分开打分，不要强行合并成一个分数。能用确定性规则（跑测试、查 diff）就不要上 LLM Judge，成本更低、完全可复现。
3. **用 `evalHarnessTable` 做结构化 A/B**：根据场景的外部随机性调整 `repetitions`（纯代码任务 3~5 次即可，涉及外部环境/随机输入时再加大）；用 `input.id` 给任务命名，让报告可读。
4. **`judgeThreshold: null` 用于对比、硬断言用于不可逆的安全红线**——两者不要混用：质量判断交给 Judge 观察趋势，"绝不能发生的事"（执行破坏性命令、越权访问）交给 `expect(...)` 直接拦截。
5. **落盘可回放证据**（会话 JSONL 快照、源码快照），保证任何一次失败的评估都能事后完整复盘，而不必依赖最终文字总结去猜测过程。

这五条本质上就是第一部分里从 `pi-evals` 源码提炼出的设计原则——`Harness` 接口的领域无关性，让这套方法论既能直接服务于 Pi 的编码主场，也能被复用到"手机端 computer-use 导购 Agent"这种 Pi 官方代码库完全没有涉及的场景。
