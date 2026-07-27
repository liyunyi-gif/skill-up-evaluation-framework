# 08 · 逐行讲解 ① — 单 case 生命周期 `executeCaseOnce`

> 「逐行讲解」系列第 1 篇。项目很大，按点逐个讲透，每篇一个点。
> 本篇对应 [07-核心流程讲解](./07-核心流程讲解.md) 里「7 步考试流程」的**真实代码**版本。

## 深度讲解路线图（8 个点）

1. **单 case 生命周期 `executeCaseOnce`**（7 步真实代码）← 本篇
2. Judge 三策略内部（rule_based / script / agent_judge）
3. Agent 多引擎适配（工厂 + 模板方法 + BaseAgent/CLIAgent）
4. 并发 + 重试 + 超时（EvaluateAll + executeCase）
5. 配置加载与分层校验
6. Runtime 沙箱三后端（none/opensandbox/docker）
7. Report 生成 + Anthropic 兼容
8. 安全：Custom Engine 不可信输入

---

## 全貌（先记骨架）

整个项目的**脊椎**。一个 case 从「建沙箱」到「出分」全在这个函数（[internal/evaluator/evaluator.go:333](../internal/evaluator/evaluator.go#L333)）：

```go
func (e *defaultEvaluator) executeCaseOnce(ctx, caseCfg, configName, ...) EvalResult {
    ① rt, _ := e.prepareRuntimeForCase(...)        // 建隔离沙箱
       defer rt.Close()
    ② e.setupCaseEnvironment(ctx, rt, ...)         // 装环境+Agent+MCP+Skill+夹具
    ③ sessionResult, _ := runAgent.Run(...)        // 发题，跑被测 Skill
    ④ e.collectGlobArtifacts(...)                  // 抓附加产物 + workspace diff
    ⑤ expect 预检查 → 失败就 return FAIL (短路)
    ⑥ grading, _ := j.Evaluate(ctx, judgeInput)    // 打分
    ⑦ 合并 expect+judge 断言 → EvalResult
}
```

---

## ① 建隔离考场（沙箱）

```go
// evaluator.go:375-388
var rt runtime.Runtime
rt, err = e.prepareRuntimeForCase(ctx, caseCfg, configName, runAgent)
if err != nil {
    result.Status = judge.StatusError
    return result
}
defer func() { _ = rt.Close() }()   // ★ 用完即关，即使中途出错也清理
```

`prepareRuntimeForCase` 内部（[968](../internal/evaluator/evaluator.go#L968)）：`runtime.NewRuntime(cfg)` 按 `type` 选 none/opensandbox/docker → `rt.Create(ctx)` 建工作区。

> **要点**：每个 case 一个独立沙箱，互不污染；`defer Close` 保证异常路径也清理。

---

## ② 布置考场（装环境 + Agent + MCP + Skill + 夹具）

`setupCaseEnvironment`（[992](../internal/evaluator/evaluator.go#L992)），按顺序 5 件事：

```go
// 1. 跑自定义初始化命令（eval.yaml 的 setup_steps，如 "npm install"）
for i, step := range e.evalCfg.Environment.SetupSteps {
    result, err := rt.Exec(ctx, step.Run, ...)
}

// 2. 装 Agent CLI（none runtime 跳过，本机可能已装）
if e.evalCfg.Environment.Type != "none" {
    ag.Install(ctx, rt)
}

// 3. 装 MCP server
ag.InstallMCP(ctx, rt, mcpCfg)

// 4. ★ 装"你的 Skill"——without_skill 配置跳过！
if configName != "without_skill" {
    for _, skillRef := range e.evalCfg.Skills {
        ag.InstallSkill(ctx, rt, skillCfg)
    }
}

// 5. 上传测试夹具（repo_fixture / files / git init）
e.fixtures.UploadAll(ctx, rt, caseCfg, ...)
```

> **重点看第 4 步**：`configName != "without_skill"` 时才装 Skill。这就是 **A/B 基线对比的核心机制** —— 同一个 case 跑两遍，一遍装 Skill、一遍不装，差异就是 Skill 的增量价值。
> **幂等性**：`Install` 内部先 `command -v` 检查，已装就 short-circuit，不重复装。

---

## ③ 发题收卷（跑被测 Agent）

```go
// evaluator.go:437-446
agentExecOpts := agent.ExecOptions{
    ArtifactDir:   agentArtifactDir,           // 产物落盘目录
    TimeoutSec:    caseTimeoutSeconds(...),     // case 级超时
    AgentMetadata: &runtime.AgentMetadata{CaseID, Variant: configName, MaxTurns},
}
sessionResult, execErr := runAgent.Run(agentCtx, rt, agentExecOpts, messages)
```

`messages` 来自 `buildCaseMessages(caseCfg)`（[270](../internal/evaluator/evaluator.go#L270)）—— 把 `input.prompt` 或 `input.turns[]` 转成统一 `[]transcript.Message`。

`runAgent.Run` 内部（点 3 细讲）：渲染命令 → 启动 cc/codex 二进制（子进程）→ 把 prompt 喂进去 → 解析它的 JSONL/NDJSON 输出 → 返回 `SessionResult{FinalMessage, Transcript, ExitCode, InputTokens, OutputTokens, Artifacts}`。

> **要点**：skill-up 这里只是「发题 + 收卷」，真正执行的是被调起的 Agent 引擎。

---

## ④ 抓附加卷面（产物 + workspace diff）

```go
// evaluator.go:456
e.collectGlobArtifacts(ctx, rt, configName, caseCfg)
```

外加 `prepareWorkspaceArtifacts`（[1134](../internal/evaluator/evaluator.go#L1134)）：用 `agent_judge` 时，会在 ③ 之前 `git commit` 打 baseline，③ 之后 `git diff baseline` —— 抓出 **Agent 改了哪些文件**，给裁判当证据。

> **要点**：不只看 Agent「说了什么」（finalMessage），还看它「做了什么」（文件改动）。对评测「写代码/改文件」类 Skill 很关键。

---

## ⑤ 快速预判（expect 短路）

`evaluateCaseSession` → `runExpectPreCheck`（[540](../internal/evaluator/evaluator.go#L540)）：

```go
expectResult := judge.CheckExpect(expectCfg, judgeInput)   // 纯函数，零成本
result.ExpectResult = expectResult
if !expectResult.Passed {
    result.Grading = judge.NewResult(assertions, ...)
    result.Status = judge.StatusFail
    return true   // ★★★ 短路：直接 FAIL，跳过昂贵的 judge
}
```

> **省钱的关键设计**：能用关键字/文件存在/退出码确定性判断的，绝不花一次 LLM 调用去判。失败直接出局。

---

## ⑥ 正式判卷（Judge 打分）

`runJudgePhase`（[567](../internal/evaluator/evaluator.go#L567)）核心两行：

```go
// evaluator.go:643
grading, err := j.Evaluate(ctx, judgeInput)   // judgeInput 是统一数据边界
result.Grading = grading
result.Status = grading.Status
```

`judgeInput`（[503-516](../internal/evaluator/evaluator.go#L503)）就是那个**统一数据边界**：

```go
judgeInput := judge.Input{
    FinalMessage:   result.FinalMessage,
    WorkspacePath:  rt.Workspace(),
    WorkspaceDiff:  ...,        // ④ 抓的文件改动
    GeneratedFiles: ...,
    Transcript:     ...,        // 完整对话
    TurnResults:    ...,        // 多轮结果
}
```

`j` 是工厂按 `judge.type` 创建的（rule_based/script/agent_judge），三种 judge 消费**同一个 Input**。

> **架构精髓**：这一行就是「执行层 ↔ 评测层」的边界。换 Judge 策略不动执行层，换执行引擎不动 Judge —— 靠的就是这个 Input struct。

---

## ⑦ 出小分（合并断言）

```go
// evaluator.go:527-535
// expect 通过的断言，前置拼到 judge 断言里一起算分
finalResult.Grading.AssertionResults = append(expectAssertions, finalResult.Grading.AssertionResults...)
finalResult.Grading.Summary.Passed += len(expectAssertions)
finalResult.Grading.Summary.Total   += len(expectAssertions)
finalResult.Grading.Summary.PassRate = float64(Passed) / float64(Total)
```

> **要点**：expect 通过的规则也生成断言（evidence="all checks passed"），保证报告能看到「expect 检查了什么」，而不只是「什么挂了」。

---

## 一句话记住这 7 步

> **建沙箱 → 装(Skill/MCP/夹具) → Agent.Run 收卷 → 抓 diff → expect 短路 → Judge 打分 → 合并出分。**
>
> 三个面试加分点：
> - ② 的 `without_skill` 跳过装 Skill = **A/B 机制**
> - ⑤ 的短路 = **省钱**
> - ⑥ 的 `judge.Input` = **解耦边界**

---

**下一篇**：[点 2 — Judge 三策略内部](./)（待写）

**回到**：[README](./README.md) ｜ [07 核心流程讲解（概念版）](./07-核心流程讲解.md) ｜ [03 子系统详解](./03-子系统详解.md)
