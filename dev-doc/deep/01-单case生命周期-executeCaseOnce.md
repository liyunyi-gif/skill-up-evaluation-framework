# 单 case 生命周期：executeCaseOnce（7 步真实代码）

> 面向：Java 后端/全栈工程师，看懂 skill-up 评测主流程，准备简历与面试。
> 配套源码：`internal/evaluator/evaluator.go`、`internal/evaluator/multiturn.go`、`internal/evaluator/artifacts_collect.go`、`internal/evaluator/fixtures.go`。

---

## 0. 一句话定位

**`executeCaseOnce` 是 skill-up 评测框架里「跑一个 case 一次」的编排核心**：它把
「建沙箱 → 装环境 → 跑 Agent → 收产物 → expect 预检 → judge 评分 → 合并断言」
七个动作串成一条线性流水线，是 `Evaluator` 接口里**最值得拿来面试官追问的一个方法**。

### 在整体架构中的位置

skill-up 是一个声明式评测框架（输入 `evals/eval.yaml` + `cases/*.yaml`，输出 JSON / JUnit / HTML / Anthropic `grading.json`）。整体可以分四层，从外到内：

```
┌──────────────────────────────────────────────────────────────────┐
│  CLI 层    cmd/skill-up/main.go → internal/cli/run.go            │
│            （Cobra 子命令：run / validate / list-cases ...）     │
└──────────────────────────────────────────────────────────────────┘
                              │ 调用
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  Runner 层  internal/runner （端到端编排：加载配置 → 跑评测 → 报告）│
└──────────────────────────────────────────────────────────────────┘
                              │ 调用 Evaluator.EvaluateAll
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  Evaluator 层  internal/evaluator/evaluator.go                   │
│   ┌────────────────────────────────────────────────────────┐    │
│   │ EvaluateAll   并发调度（sem + goroutine，concurrency） │    │
│   │   └─ executeCase          ← 重试包装层（指数退避）     │    │
│   │        └─ executeCaseOnce ← 【本文主角】7 步流水线     │    │
│   │             ├─ prepareRuntimeForCase                   │    │
│   │             ├─ setupCaseEnvironment                    │    │
│   │             ├─ Agent.Run / executeMultiTurn            │    │
│   │             ├─ collectGlobArtifacts / workspace diff   │    │
│   │             ├─ runExpectPreCheck                       │    │
│   │             ├─ runJudgePhase                          │    │
│   │             └─ 合并 expect + judge 断言                │    │
│   └────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
                              │ 依赖
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  基础设施层  runtime（沙箱） / agent（引擎适配） / judge（裁判） │
│              / mcp / credential / config / observability         │
└──────────────────────────────────────────────────────────────────┘
```

`executeCaseOnce` 处于「Evaluator 层」的最内圈，是**单 case、单次执行**的真正实现。
它外面的 `executeCase` 负责「重试 + 超时」，再外面的 `EvaluateAll` 负责「并发调度」。
本文聚焦最内圈。

---

## 1. 外壳：`executeCase`（重试包装层）

在讲 7 步之前，先看它外层的 retry 包装。`executeCaseOnce` 不会自己重试，重试是父函数 `executeCase` 做的（`evaluator.go:289-331`）。

### 1.1 真实代码

```go
// evaluator.go:289
func (e *defaultEvaluator) executeCase(ctx context.Context, caseCfg *config.CaseConfig, configName string, overrideRT runtime.Runtime, overrideAgent agent.Agent) EvalResult {
	maxAttempts := max(e.evalCfg.Cases.RetryPolicy.MaxRetries+1, 1)  // :290

	var result EvalResult
	for attempt := 1; attempt <= maxAttempts; attempt++ {             // :293
		attemptCtx, cancel, timeoutSource, timeoutSec := withCaseTimeout(ctx, e.evalCfg.Cases.Defaults.TimeoutSeconds, caseCfg.Constraints.TimeoutSeconds) // :294
		result = e.executeCaseOnce(attemptCtx, caseCfg, configName, overrideRT, overrideAgent) // :295
		// 关键：先标注超时，再 cancel()，避免 cancel() 把 Err 改写成 Canceled 抹掉信号
		if errors.Is(attemptCtx.Err(), context.DeadlineExceeded) && ctx.Err() == nil { // :305
			annotateCaseTimeoutError(&result, timeoutSource, timeoutSec)                // :306
		}
		cancel()                                                       // :308

		retryReason, ok := retryReasonForResult(result)                // :310
		if !ok || !retryAllowed(e.evalCfg.Cases.RetryPolicy, retryReason) || attempt == maxAttempts { // :311
			return result
		}

		logging.WarnContextf(ctx, "Runner: case %s (%s) attempt %d/%d hit %s, retrying after %s", ...) // :315
		if err := sleepWithContext(ctx, retryBackoffDelay(attempt)); err != nil { // :325
			return result
		}
	}

	return result
}
```

### 1.2 逐段讲

**(a) `maxAttempts`（`evaluator.go:290`）**

```go
maxAttempts := max(e.evalCfg.Cases.RetryPolicy.MaxRetries+1, 1)
```

- YAML 里 `cases.retry_policy.max_retries: 2` 表示「最多重试 2 次」，所以**总尝试次数 = `max_retries + 1`**（首次 + 重试 N 次）。
- 用 `max(..., 1)` 兜底：即使配置写成 0 或负数，也至少跑一次。
- **Java 类比**：Spring Retry 的 `@Retryable(maxAttempts = 3)` 里 `maxAttempts` 是**包含首次**的总次数；skill-up 这里把 YAML 的语义设计成「重试次数」，所以代码里要 +1，相当于 `Spring 的 maxAttempts = retryPolicy.maxRetries + 1`。

**(b) `withCaseTimeout`（`evaluator.go:294` → 定义在 `873-883`）**

```go
func withCaseTimeout(ctx context.Context, defaultTimeoutSec, caseTimeoutSec int) (context.Context, context.CancelFunc, string, int) {
	if caseTimeoutSec > 0 {
		c, cancel := context.WithTimeout(ctx, time.Duration(caseTimeoutSec)*time.Second)
		return c, cancel, "case.constraints.timeout_seconds", caseTimeoutSec
	}
	if defaultTimeoutSec > 0 {
		c, cancel := context.WithTimeout(ctx, time.Duration(defaultTimeoutSec)*time.Second)
		return c, cancel, "cases.defaults.timeout_seconds", defaultTimeoutSec
	}
	return ctx, func() {}, "", 0
}
```

- 优先用 case 级超时（`case.constraints.timeout_seconds`），其次用全局默认（`cases.defaults.timeout_seconds`），都没配就不限时。
- 关键设计：**返回超时值的同时，返回它来自哪个配置键**（`timeoutSource`），这样超时报错时能精确告诉用户「请调 `case.constraints.timeout_seconds`」而不是甩一句「超时了」。
- **Java 类比**：相当于把 `CompletableFuture.orTimeout(long, unit)` 的超时值与对应的 `@Timeout` 注解位置一起带出来，方便排错。

**(c) 超时标注的顺序极其讲究（`evaluator.go:305-308`）**

```go
// 先标注
if errors.Is(attemptCtx.Err(), context.DeadlineExceeded) && ctx.Err() == nil {
	annotateCaseTimeoutError(&result, timeoutSource, timeoutSec)
}
cancel() // 再 cancel
```

这里有两层微妙：

1. **必须先 `annotate` 再 `cancel()`**：`cancel()` 调用后 `attemptCtx.Err()` 会变成 `context.Canceled`，可能把原来的 `DeadlineExceeded` 信号覆盖掉（错误链上多挂一层 Canceled）。先标注、再 cancel，信号才不被抹掉。
2. **`ctx.Err() == nil` 的二次判断**：`attemptCtx` 是 `ctx` 的子 context。如果父 `ctx` 自己也到点了（比如调用方用 `context.WithTimeout` 包了 `EvaluateAll`），那 `attemptCtx.DeadlineExceeded` 其实是父级到点传导下来的，**不是 case 级超时**。这种情况就不该标注「case 超时」，否则会误导用户去调错 YAML 旋钮。
   - 注释原话：`a parent-supplied deadline ... firing through attemptCtx isn't relabelled as a case timeout — that would point users at the wrong YAML knob`。

`annotateCaseTimeoutError` 本体（`evaluator.go:859-867`）只是把超时秒数和来源 key wrap 进 error：

```go
func annotateCaseTimeoutError(result *EvalResult, source string, seconds int) {
	if result == nil || result.Error == nil || source == "" || seconds <= 0 {
		return
	}
	if !errors.Is(result.Error, context.DeadlineExceeded) {
		return
	}
	result.Error = fmt.Errorf("%w (case timeout %ds via %s)", result.Error, seconds, source)
}
```

注意 `%w` 是**包裹（wrap）不是替换**，原始的 `DeadlineExceeded` 仍在错误链里，下游的 `errors.Is(err, context.DeadlineExceeded)` 还能匹配上。

**(d) `retryReasonForResult`（`evaluator.go:897-905`）——只有 ERROR 才重试**

```go
func retryReasonForResult(result EvalResult) (string, bool) {
	if result.Status != judge.StatusError || result.Error == nil {
		return "", false
	}
	if isTimeoutError(result.Error) {
		return "timeout", true
	}
	return "error", true
}
```

- **FAIL 不重试**：判 FAIL 说明 case 跑完了、断言不通过，这是「评测结论」不是「故障」，重跑没意义（除非配置允许，但默认 `RetryOn` 不含 `fail`）。
- 只有 `Status == ERROR`（执行出错）才考虑重试，并细分为 `timeout` 和 `error` 两类原因。

**(e) `retryAllowed`（`evaluator.go:885-895`）——白名单匹配**

```go
func retryAllowed(policy config.RetryPolicy, reason string) bool {
	if reason == "" || policy.MaxRetries <= 0 {
		return false
	}
	for _, allowed := range policy.RetryOn {
		if strings.EqualFold(allowed, reason) {
			return true
		}
	}
	return false
}
```

- 用 `RetryOn: [timeout, error]` 这种白名单来决定「这种原因是否允许重试」。
- **Java 类比**：Spring Retry 的 `@Retryable(include = {TimeoutException.class, IOException.class})` —— 用异常类型当白名单；Go 没有检查异常，所以这里用字符串分类。

**(f) 指数退避 `retryBackoffDelay`（`evaluator.go:939-944`）**

```go
func retryBackoffDelay(attempt int) time.Duration {
	if attempt < 1 {
		attempt = 1
	}
	return time.Duration(1<<attempt) * time.Second  // 2s, 4s, 8s, 16s ...
}
```

- `1<<attempt`：`attempt=1` → 2s，`attempt=2` → 4s，`attempt=3` → 8s。
- 注意：**第 1 次重试前已经等了 2s**（attempt=1），不是 1s。这是 `2^n` 而不是 `2^(n-1)`。
- `sleepWithContext`（`evaluator.go:946-956`）用 `select` 同时监听 `ctx.Done()` 和 timer，**父 context 取消时立刻退出**，不会傻等。
- **Java 类比**：Spring Retry 的 `ExponentialBackOffPolicy` 默认 `initial=100ms, multiplier=2.0`；这里是写死的 `2^n` 秒，没有 jitter，也没暴露 multiplier。

---

## 2. 主角：`executeCaseOnce` 的 7 步流水线

`executeCaseOnce`（`evaluator.go:333-481`）是本文主角。注释里明确写了：
`orchestrates the full case lifecycle (runtime prep → agent run → judge → finalize); splitting further would obscure the linear flow`。
作者故意不拆这个函数，因为拆了反而看不见「线性」这条主线。

下面把它拆成 7 步逐段讲。

---

### 步骤 0（预备）：埋点、defer、初始化 `EvalResult`

```go
// evaluator.go:334-368
ctx, span := observability.Tracer().Start(ctx, "evaluator.case")
defer span.End()
span.SetAttributes(
	attribute.String("skill_up.case.id", caseCfg.ID),
	attribute.String("skill_up.case.configuration", configName),
)
// ... 把 case id / configName 塞进 context，给后续 agent 层做 trace 关联

startTime := time.Now()

prompt, turnsTotal := casePromptAndTurnsTotal(caseCfg) // :347
messages := buildCaseMessages(caseCfg)                  // :348

result := EvalResult{
	CaseID:        caseCfg.ID,
	CaseName:      caseCfg.Title,
	Prompt:        prompt,
	SessionResult: &agent.SessionResult{},
	TurnsTotal:    turnsTotal,
}
defer func() { /* 记录 durationMs、status、span 属性 */ }()
```

- 用 OpenTelemetry 开一个 span（`evaluator.case`），把 case id 和 `configName`（`with_skill` 或 `without_skill`）作为属性打上。
- `casePromptAndTurnsTotal`（`evaluator.go:239-249`）：单轮用 `input.prompt`，多轮用 `input.turns[0].content` 当 prompt；`turnsTotal` 在多轮场景下是 `len(input.turns)`。
- `buildCaseMessages`（`evaluator.go:270-287`）：把 YAML 里的 `input.turns` 转成 `transcript.Message` 列表，每条带 `Turn` 序号。
- **defer 里的兜底计时**（`evaluator.go:357-368`）：如果走到最后 `result.DurationMs == 0`（比如提前 return 的错误路径没设置），用 `time.Since(startTime)` 兜底，保证每条结果都有耗时。

---

### 步骤 ① `prepareRuntimeForCase`：建沙箱（`evaluator.go:375-388`）

```go
// evaluator.go:375-388
var rt runtime.Runtime
if overrideRT != nil {
	rt = overrideRT
} else {
	var err error
	rt, err = e.prepareRuntimeForCase(ctx, caseCfg, configName, runAgent)
	if err != nil {
		result.Status = judge.StatusError
		result.Error = err
		result.Configuration = configName
		return result
	}
	defer func() { _ = rt.Close() }()
}
```

`prepareRuntimeForCase`（`evaluator.go:968-990`）做了三件事：

```go
func (e *defaultEvaluator) prepareRuntimeForCase(ctx context.Context, caseCfg *config.CaseConfig, configName string, ag agent.Agent) (runtime.Runtime, error) {
	rtCfg := e.evalCfg.Environment.ToRuntimeConfig()
	rtCfg.Delete = e.deleteWorkspace
	mcpCfg, mcpEnv, err := e.provisionMCPConfigForCase(caseCfg)  // :971
	if err != nil {
		return nil, err
	}
	rtCfg.Env = mergeEnvMaps(rtCfg.Env, mcpEnv)

	rt, err := runtime.NewRuntime(rtCfg)  // :977 工厂方法
	if err != nil {
		return nil, fmt.Errorf("failed to create runtime: %w", err)
	}
	if err := rt.Create(ctx); err != nil {  // :981 真正创建沙箱（起容器/建目录）
		_ = rt.Close()
		return nil, fmt.Errorf("failed to create runtime workspace: %w", err)
	}
	if err := e.setupCaseEnvironment(ctx, rt, caseCfg, configName, ag, mcpCfg); err != nil { // :985
		_ = rt.Close()
		return nil, fmt.Errorf("failed to setup case environment: %w", err)
	}
	return rt, nil
}
```

- **`runtime.Runtime` 是个接口**（`internal/runtime/runtime.go:118`），实现有 `none`（本机直接跑）/ `docker` / `opensandbox`（远程容器）三种。`NewRuntime` 是工厂。
- `rt.Create(ctx)` 真正起沙箱（建临时目录或拉容器），失败时**先 `rt.Close()` 再返回错误**，避免资源泄漏。
- **defer 里的 `_ = rt.Close()`**（`:387`）保证每次 case 结束都关掉沙箱——每个 case 都用**独立 workspace**，互不污染。
- `overrideRT` 参数：测试或外层调用可以注入自定义 runtime，跳过工厂。
- **Java 类比**：相当于 `try (Sandbox sandbox = SandboxFactory.create(config)) { ... }`（带 try-with-resources），`Runtime.Close()` 就是 `AutoCloseable.close()`。

---

### 步骤 ② `setupCaseEnvironment`：装环境、装 Agent、装 MCP、装 Skill、上传夹具

这是 A/B 测试机制的核心。完整代码在 `evaluator.go:992-1031`：

```go
func (e *defaultEvaluator) setupCaseEnvironment(ctx context.Context, rt runtime.Runtime, caseCfg *config.CaseConfig, configName string, ag agent.Agent, mcpCfg runtime.MCPConfig) error {
	// (a) 全局 setup_steps：跑 shell 命令
	for i, step := range e.evalCfg.Environment.SetupSteps {                 // :993
		result, err := rt.Exec(ctx, step.Run, runtime.ExecOptions{})
		if err != nil {
			return fmt.Errorf("setup step %d (%q) failed: %w", i+1, step.Run, err)
		}
		if result.ExitCode != 0 {
			return fmt.Errorf("setup step %d (%q) exited with code %d: %s", i+1, step.Run, result.ExitCode, result.Stderr)
		}
	}

	// (b) 安装 Agent（CLI 引擎本身，比如 claude-code / codex CLI）
	if e.evalCfg.Environment.Type != "none" {                                // :1003
		if err := ag.Install(ctx, rt); err != nil {
			return fmt.Errorf("failed to install agent %s: %w", ag.Name(), err)
		}
	}

	// (c) 安装 MCP servers（外部工具协议，比如 mock 服务）
	if err := ag.InstallMCP(ctx, rt, mcpCfg); err != nil {                   // :1009
		return fmt.Errorf("failed to install MCP servers: %w", err)
	}

	// (d) 【A/B 关键】with_skill 才装 Skill，without_skill 跳过！
	if configName != "without_skill" && e.loader != nil {                    // :1013
		for _, skillRef := range e.evalCfg.Skills {
			skillCfg := resolveSkillConfig(e.loader.SkillDir(), skillRef)
			if err := ag.InstallSkill(ctx, rt, skillCfg); err != nil {
				return fmt.Errorf("failed to install skill %s: %w", skillRef.Path, err)
			}
			logging.DebugContextf(ctx, "Evaluator: skill installed: %s", filepath.Base(skillCfg.Source))
		}
	}

	// (e) 上传夹具（repo / context.files / git init / apply_diff）
	if e.fixtures != nil && e.loader != nil && e.skillDir != "" {            // :1023
		fixtureBaseDir := e.loader.SkillDir()
		if err := e.fixtures.UploadAll(ctx, rt, caseCfg, e.skillDir, fixtureBaseDir); err != nil {
			return fmt.Errorf("failed to upload fixtures: %w", err)
		}
	}

	return nil
}
```

逐条解释：

- **(a) `setup_steps`**：YAML 里 `environment.setup_steps` 是一段 shell 脚本（比如 `npm install`），在沙箱里直接 `rt.Exec` 跑。失败要带 `i+1`（步骤序号）和原始命令，方便定位。
- **(b) `ag.Install`**：装 Agent 引擎本身（如 claude-code CLI、codex CLI）。`Environment.Type == "none"` 时跳过，意味着「直接用宿主机已经装好的 agent」。
- **(c) `ag.InstallMCP`**：把 MCP servers（比如一个 mock 的 weather server）注册进 agent。
- **(d) `InstallSkill` —— A/B 测试的灵魂**：

  ```go
  if configName != "without_skill" && e.loader != nil {
  ```

  这一行是**整个 skill-up 框架的核心**。`EvaluateAll` 会对每个 case 生成两个任务（`evaluator.go:200-206`）：

  ```go
  tasks = append(tasks, task{caseCfg: c, configName: "with_skill"})
  if e.withBaseline {
      tasks = append(tasks, task{caseCfg: c, configName: "without_skill"})
  }
  ```

  - `with_skill`：装上 SKILL.md，看 agent「有技能」时表现。
  - `without_skill`：**完全跳过 `InstallSkill`**，看 agent 「裸跑」时表现。

  两次执行用**完全相同**的 prompt / 夹具 / setup_steps / MCP，**唯一的变量就是有没有 Skill**。这就是典型的 **A/B 对照实验**——技能带来的提升 = `with_skill - without_skill`。
  - **Java 类比**：相当于 JMH 基准测试里的 `@Benchmark` 和对照组，控制变量；或者像 A/B test 平台里「实验组 / 对照组」的分桶。

- **(e) 上传夹具**：`fixtureRegistry.UploadAll`（`fixtures.go:285-292`）按固定顺序跑一系列 uploader：

  ```go
  // fixtures.go:275-282
  uploaders: []FixtureUploader{
      &repoFixtureUploader{},    // 1. 拷贝整份仓库（repo_fixture）
      &gitInitUploader{},        // 2. git init + 配 user
      &gitCheckoutUploader{},    // 3. git switch 到指定分支
      &contextFilesUploader{},   // 4. 写入 inline context.files
      &applyDiffUploader{},      // 5. git apply 一份 patch
  }
  ```

  顺序非常讲究（注释原话 `fixtures.go:271-274`）：
  > *Order matters: the git repo must exist and be on the right branch before inline context.files are written, so those files land on the target branch as case overrides instead of being clobbered by (or aborting) the branch switch. apply_diff runs last, on top of both.*

  即：先有仓库 → 切到目标分支 → 再写 inline 文件（这样文件落到正确分支上，不会被 `git switch` 覆盖）→ 最后打 patch。

---

### 步骤 ③ `Agent.Run`（单轮） / `executeMultiTurn`（多轮分支）

```go
// evaluator.go:392-417
judgeCfg := judge.MergeJudgeConfig(e.evalCfg.Judge, caseCfg.Judge)

// 多轮分支：case 定义了 input.turns 且 agent 支持 SessionResumer
if len(caseCfg.Input.Turns) > 0 {                                            // :398
	if _, ok := runAgent.(agent.SessionResumer); ok {                        // :399
		agentExecOpts := agent.ExecOptions{
			ArtifactDir: e.prepareOutputDir(ctx, configName, caseCfg.ID, "agent/run"),
			TimeoutSec:  caseTimeoutSeconds(e.evalCfg, caseCfg),
			AgentMetadata: &runtime.AgentMetadata{
				CaseID:   caseCfg.ID,
				Variant:  configName,
				MaxTurns: caseMaxTurns(e.evalCfg, caseCfg),
			},
		}
		return e.executeMultiTurnCase(ctx, rt, caseCfg, configName, runAgent, agentExecOpts, startTime, judgeCfg, &result) // :409
	}
	// agent 不支持多轮 → 退化成「把所有 turn 拼成一个 prompt 一次跑」
	logging.WarnContextf(ctx,
		"Evaluator: case %s defines input.turns but agent %s does not support session resumption; falling back to a single batch prompt",
		caseCfg.ID,
		runAgent.Name(),
	)
}
```

#### ③-a 多轮机制简介（细节在下一篇文档讲）

`executeMultiTurnCase`（`multiturn.go:649-712`）委托给 `executeMultiTurn`（`multiturn.go:72-153`），核心循环：

```go
for i, turn := range turns {
	turnNum := i + 1
	// 1. 模板替换 {{var}} → 上一轮 captured 值
	content, err := substituteTemplate(turn.Content, state.capturedVars)
	// 2. 调 agent 的 RunTurn（带 sessionID 续接）
	sessionResult, execErr := executeSingleTurn(ctx, resumer, rt, agentExecOpts, content, turnNum, turn.TimeoutSeconds, state.sessionID)
	// 3. 累加 token / duration（区分「本轮增量」和「累计转增量」两种语义）
	accumulatePerTurnMetrics(...) 或 accumulateCumulativeMetrics(...)
	// 4. 评估 post_condition
	if failed, earlyReturn := e.handlePostCondition(...); failed || earlyReturn { return ... }
	// 5. 捕获 capture 变量（regex 或 jsonpath）
	captured, captureErr := captureVariables(turn.Capture, tr.Response)
	maps.Copy(state.capturedVars, captured)
}
```

`SessionResumer` 是个**可选接口**（`agent.go:72-74`）：

```go
type SessionResumer interface {
	RunTurn(ctx context.Context, rt Runtime, opts ExecOptions, message transcript.Message, sessionID string) (*SessionResult, error)
}
```

只有 `ClaudeCodeAgent` / `QoderCLIAgent` / `CodexAgent` 实现了它（`agent.go:151-155` 的编译期检查）。`SessionID` 在 turn 之间传递，agent CLI 内部维护会话状态。

#### ③-b 单轮 `Agent.Run`（`evaluator.go:419-478`）

如果是单轮，或者多轮 agent 不支持续接（退化情况），就走 `Agent.Run`：

```go
// evaluator.go:419-448
var cleanupArtifacts func()
finalizeArtifacts := func(*agent.SessionResult) {}
if judgeNeedsWorkspaceDiff(judgeCfg) {                                  // :421
	cleanupArtifacts, finalizeArtifacts = e.prepareWorkspaceArtifacts(ctx, rt, caseCfg)
	defer cleanupArtifacts()
}

agentArtifactDir := e.prepareOutputDir(ctx, configName, caseCfg.ID, "agent/run")
agentCtx := observability.ContextWithConfiguredAgentSpanAttributes(ctx, nil)
agentCtx, agentSpan := startAgentRunSpan(agentCtx)
// ... 给 span 打属性

agentExecOpts := agent.ExecOptions{
	ArtifactDir: agentArtifactDir,
	TimeoutSec:  caseTimeoutSeconds(e.evalCfg, caseCfg),
	AgentMetadata: &runtime.AgentMetadata{
		CaseID:   caseCfg.ID,
		Variant:  configName,
		MaxTurns: caseMaxTurns(e.evalCfg, caseCfg),
	},
}
sessionResult, execErr := runAgent.Run(agentCtx, rt, agentExecOpts, messages) // :446
agentSpan.End()
finalizeArtifacts(sessionResult)  // :448 收集 workspace diff
result.SessionResult = normalizeSessionResult(sessionResult)
```

- `Agent.Run` 是 agent 接口的核心方法（`agent.go:143`）：`Run(ctx, rt, opts, messages) (*SessionResult, error)`。每个 agent 适配器（claude-code、codex、qoder_cli、custom）各自实现，本质是**起子进程跑 agent CLI**。
- **第 ④ 步会紧接着做产物收集**，所以这里先记一下：`prepareWorkspaceArtifacts` 在 Run 之前已经把 git baseline 准备好（详见 ④）。

---

### 步骤 ④ 产物收集：`collectGlobArtifacts` + workspace diff

case 跑完之后，要把沙箱里的产物（生成的文件、改动的代码）拉出来给 judge 看。这里有两套机制：

#### ④-a workspace diff（git baseline → diff）

`prepareWorkspaceArtifacts`（`evaluator.go:1134-1153`）返回**两个闭包**：

```go
func (e *defaultEvaluator) prepareWorkspaceArtifacts(ctx context.Context, rt runtime.Runtime, caseCfg *config.CaseConfig) (func(), func(*agent.SessionResult)) {
	state, err := prepareWorkspaceDiffState(ctx, rt, caseCfg.Context.Git) // 跑 agent 之前先存 baseline
	if err != nil {
		logging.WarnContextf(ctx, "Judge: failed to snapshot workspace before run for case %s: %v", caseCfg.ID, err)
	}

	finalize := func(sessionResult *agent.SessionResult) {
		if sessionResult == nil || !state.enabled {
			return
		}
		workspaceDiff, diffErr := collectWorkspaceDiff(ctx, rt, state, sessionGeneratedFiles(sessionResult)) // 跑完之后 diff baseline
		if diffErr != nil {
			logging.WarnContextf(ctx, "Judge: failed to collect workspace diff for case %s: %v", caseCfg.ID, diffErr)
			return
		}
		ensureArtifacts(sessionResult).WorkspaceDiff = workspaceDiff
	}

	return func() {}, finalize
}
```

- **Run 之前**调 `prepareWorkspaceDiffState`（`evaluator.go:1202-1247`）：在沙箱里 `git init` + `git add --all` + `git commit` 打一个 baseline，记下 `baselineRev`。

  ```bash
  # evaluator.go:1219-1233 的脚本
  set -eu
  if git rev-parse --git-dir >/dev/null 2>&1; then :; else git init -q ...; fi
  git config user.name skill-up
  git config user.email skill-up@example.invalid
  git add --all
  git commit --allow-empty -qm skill-up-baseline
  git rev-parse HEAD
  ```

- **Run 之后**调 `collectWorkspaceDiff`（`evaluator.go:1249-1270`）：`git diff --cached <baseline>` 算出 agent 改了哪些代码，**排除 agent 自己生成的产物文件**（避免把一大坨 JSON 输出当 diff 喂给 judge）。

  ```go
  // evaluator.go:1254-1260
  cmd.WriteString("set -eu\n")
  cmd.WriteString(`git add --all` + "\n")
  cmd.WriteString(`git diff --cached --no-ext-diff ` + shellQuote(state.baselineRev) + ` -- .`)
  for _, pathspec := range gitDiffExcludePathspecs(rt.Workspace(), generatedFiles) {
      cmd.WriteString(" " + shellQuote(pathspec))
  }
  ```

- 只有 `judge.type == agent_judge` 时才需要 diff（`judgeNeedsWorkspaceDiff`，`evaluator.go:1123-1125`）。因为 rule_based / script judge 通常只看 stdout，agent_judge 才需要看代码改动。

#### ④-b glob artifacts（`collectGlobArtifacts`）

```go
// evaluator.go:456
e.collectGlobArtifacts(ctx, rt, configName, caseCfg)
```

这个调用在 `evaluator.go:456`，**位置极其关键**——它紧跟在 `Agent.Run` 之后，**在 `handleExecutionResult` 的 early-return 之前**，也**在 `defer rt.Close()` 之前**。意思是：

> **即使 agent 失败、超时，也要把沙箱里的产物拉出来**——因为失败 case 的现场最有价值，丢了就调试不了。

`collectGlobArtifacts`（`artifacts_collect.go:62-105`）逻辑：

```go
func (e *defaultEvaluator) collectGlobArtifacts(ctx context.Context, rt runtime.Runtime, configName string, caseCfg *config.CaseConfig) {
	patterns := resolveCollectArtifacts(e.evalCfg, caseCfg) // 合并默认 + case 级 glob
	if len(patterns) == 0 { return }
	// ...

	// case context 可能已经因为超时被 cancel，这里 detach 出来重设 60s 超时
	dctx := context.WithoutCancel(ctx)                                    // :73
	dctx, cancel := context.WithTimeout(dctx, collectArtifactsTimeout)   // 60s
	defer cancel()

	files, err := listWorkspaceFiles(dctx, rt) // find . -type f -not -path './.git/*'
	// ...

	targetRoot := filepath.Join(e.outputDir, caseCfg.ID, configName, "outputs", "workspace")
	os.RemoveAll(targetRoot) // 清掉旧产物，避免 retry 残留
	for _, rel := range files {
		if !matchesAnyGlob(patterns, rel) { continue } // doublestar 匹配
		target := filepath.Join(targetRoot, filepath.FromSlash(rel))
		rt.DownloadFile(dctx, rel, target)
	}
}
```

三个值得记的设计：

1. **`context.WithoutCancel`**（Go 1.21 新 API）：父 context 已经 cancel，但产物收集必须跑完。所以**脱离父 cancel**，自己挂个 60s 新超时。
2. **`find . -type f -not -path './.git/*'`**（`artifacts_collect.go:117`）：跨 runtime 通用（none / docker / opensandbox 都有 find），并显式排除 `.git/`——因为 agent_judge 会 commit baseline，否则 `**` 会把整个 VCS 对象库当产物拖出来。
3. **任何错误只 `Warn` 不 fail**：产物收集永远不能让 case 失败，最多丢失调试信息（注释原话 `artifacts_collect.go:60-61`：`never fails the case: every error is logged as a warning`）。

---

### 步骤 ⑤ `runExpectPreCheck`：短路省钱

到这步 agent 已经跑完，产物已经收集。接下来是 **expect 预检**——一个零成本、确定性的硬门控（`evaluator.go:518`）。

```go
// evaluator.go:492-520
func (e *defaultEvaluator) evaluateCaseSession(...) EvalResult {
	judgeInput := judge.Input{ ... } // 见步骤 ⑥

	if failed := e.runExpectPreCheck(ctx, caseCfg, configName, judgeInput, turnsTotal, result); failed { // :518
		return *result  // expect 失败 → 直接 FAIL，跳过 judge
	}

	var expectAssertions []judge.AssertionResults
	if result.ExpectResult != nil {
		expectAssertions = result.ExpectResult.ToAssertionResults()
	}

	finalResult := e.runJudgePhase(...) // expect 通过才走到这
	// ... 合并断言
}
```

`runExpectPreCheck` 本体（`evaluator.go:540-565`）：

```go
func (e *defaultEvaluator) runExpectPreCheck(...) bool {
	expectCfg := resolveExpectConfig(&caseCfg.Expect)
	if expectCfg == nil {
		return false  // 没配 expect，跳过
	}

	expectResult := judge.CheckExpect(expectCfg, judgeInput) // 纯函数，不调 LLM
	result.ExpectResult = expectResult
	if expectResult.Passed {
		return false  // 全通过，继续往下走 judge
	}

	// expect 失败 → 立刻 FAIL
	assertions := expectResult.ToAssertionResults()
	result.Grading = judge.NewResult(assertions, result.Turns, turnsTotal)
	result.Status = judge.StatusFail
	result.Configuration = configName
	logging.DebugContextf(ctx, "Judge: case %s expect pre-check FAILED", caseCfg.ID)
	return true
}
```

`judge.CheckExpect`（`judge/expect.go:48-90`）支持的规则全是**纯字符串/文件检查**：

| 规则 | 含义 |
|---|---|
| `must_contain` | final_message 必须包含某些子串 |
| `must_not_contain` | 必须不包含 |
| `exit_code` | 进程退出码必须等于指定值 |
| `files_exist` / `files_not_exist` | workspace 里某些文件必须存在/不存在 |
| `golden_file` | 输出和 golden 文件比对 |
| `file_contains` | 指定文件里必须包含某些串 |

**关键设计：expect 失败立刻 FAIL，跳过 judge**。为什么？

- judge（尤其 `agent_judge`）要**再调一次 LLM**，又贵又慢（一次评测动辄几美元、几十秒）。
- expect 是免费的纯函数（字符串 `Contains` + `os.Stat`），跑一次几毫秒。
- 如果 expect 已经判定 FAIL（比如 agent 输出根本没提那个 bug），再花 judge 的钱纯属浪费。

> 用 `expect.go:5-7` 的原注释：
> *Expect is a zero-cost, deterministic gate that runs BEFORE any judge. If any expect check fails, the case is immediately marked FAIL and the (potentially expensive) judge execution is skipped entirely.*

- **Java 类比**：Servlet Filter 链里的短路；或者 JUnit 的 `Assume.assumeTrue(...)`（不过 assume 是 skip，这里是 fail）。更像 Spring Security Filter 里「认证不通过直接 401，不进 Controller」。

---

### 步骤 ⑥ `runJudgePhase`：裁判评分 + `judge.Input` 统一边界

expect 通过（或没配 expect）才走到这。`runJudgePhase`（`evaluator.go:567-588`）开个 trace span 然后委托给 `runJudgePhaseWithSpan`：

```go
func (e *defaultEvaluator) runJudgePhase(...) EvalResult {
	if observability.LinkedTraceTopologyEnabled() && judgeNeedsWorkspaceDiff(judgeCfg) {
		ctx = observability.ContextWithConfiguredAgentSpanAttributes(ctx, nil)
		ctx, span := observability.StartLinkedRootSpan(ctx, "evaluator.judge")
		defer span.End()
		return e.runJudgePhaseWithSpan(ctx, span, ...)
	}
	ctx, span := observability.Tracer().Start(ctx, "evaluator.judge")
	defer span.End()
	return e.runJudgePhaseWithSpan(ctx, span, ...)
}
```

注意：`agent_judge` 时会 `StartLinkedRootSpan`——开一个**独立的根 trace**，让 judge agent 的 LLM 调用和主 agent 的 LLM 调用在 trace 树里分离（否则一次 case 的 trace 会缠成一团）。

#### ⑥-a `judge.Input` —— 执行层与评测层的统一边界

在 `evaluateCaseSession`（`evaluator.go:503-516`）里组装：

```go
judgeInput := judge.Input{
	CaseID:         caseCfg.ID,
	FinalMessage:   result.FinalMessage,
	ExitCode:       result.ExitCode,
	WorkspacePath:  rt.Workspace(),
	SkillDir:       e.skillDir,
	TurnsExecuted:  result.Turns,
	TurnsTotal:     turnsTotal,
	Transcript:     sessionTranscript(sessionResult),
	WorkspaceDiff:  sessionWorkspaceDiff(sessionResult),
	GeneratedFiles: sessionGeneratedFiles(sessionResult),
	SessionResult:  sessionResult,
	TurnResults:    toJudgeTurnResults(result.TurnResults),
}
```

`judge.Input`（`judge/judge.go:55-98`）是**所有 judge 实现共用的入参**。三种 judge 拿到的是同一个结构：

- `rule_based`：纯字段判断（`MustContain` 等），等价于 expect 的进阶版。
- `script`：起一个外部脚本，把 Input 序列化成环境变量 / JSON 喂进去（`evaluator.go:631-641` 把 transcript 序列化到临时文件，路径通过 `EVAL_TRANSCRIPT_PATH` 环境变量传给脚本）。
- `agent_judge`：起另一个 agent（裁判 agent），把 Input 拼成 prompt 喂进去，让 LLM 打分。

**设计意义**：执行层（agent 跑出来的乱七八糟结果）和评测层（judge 怎么打分）**只通过 `judge.Input` 这一个 struct 解耦**。换 judge 实现不影响执行层，加新的 agent 也不需要改 judge。

- **Java 类比**：策略模式（Strategy Pattern）。`Judge` 是 `Strategy` 接口（`judge.go:31-33` 的 `Evaluate(ctx, in Input) (*Result, error)`），`Input` 是上下文对象，`rule_based` / `script` / `agent_judge` 是三个 `ConcreteStrategy`。

#### ⑥-b `runJudgePhaseWithSpan`（`evaluator.go:590-668`）

```go
j, err := e.newJudgeForCase(ctx, rt, configName, judgeCfg, runAgent) // :613 建裁判
if err != nil {
	result.Status = judge.StatusError
	result.Error = err
	result.Configuration = configName
	return *result
}
if j == nil {  // 没配 judge → 默认 PASS
	result.Status = judge.StatusPass
	result.Grading = judge.NewResult(nil, result.Turns, turnsTotal)
	result.Configuration = configName
	return *result
}
if judgeCfg.Type == judgeTypeAgentJudge {
	judgeInput.ArtifactDir = e.prepareOutputDir(ctx, configName, caseCfg.ID, "judge/run") // :628
}

// ScriptJudge 特殊处理：把 transcript 落临时文件
if sj, ok := j.(*judge.ScriptJudge); ok && len(judgeInput.Transcript) > 0 {
	transcriptPath, cleanupFn, serErr := serializeTranscript(judgeInput.Transcript)
	if serErr != nil { /* warn */ } else {
		defer cleanupFn()
		sj.TranscriptPath = transcriptPath
	}
}

grading, err := j.Evaluate(ctx, judgeInput) // :643 真正评分
if err != nil {
	// judge 自己跑挂了 → ERROR（不是 FAIL）
	if judgeSession := judge.SessionResultFromError(err); judgeSession != nil { /* 保住 judge 产物 */ }
	result.Status = judge.StatusError
	result.Error = fmt.Errorf("judge evaluation failed: %w", err)
	result.Configuration = configName
	return *result
}

result.Grading = grading
result.Status = grading.Status
result.Configuration = configName
```

**「没配 judge 默认 PASS」**（`evaluator.go:620-626`）是条非常宽松的兜底：YAML 里完全没写 `judge:` 块，case 跑完就算 PASS。这意味着 skill-up 允许「只看 agent 能不能跑起来，不评分」。

#### ⑥-c `newJudgeForCase`：裁判中立（移除 run skill）

`newJudgeForCase`（`evaluator.go:670-696`）建裁判时做了一件**非常重要的事**：

```go
func (e *defaultEvaluator) newJudgeForCase(...) (judge.Judge, error) {
	judgeCfg = resolveJudgeScriptPath(e.judgeScriptBaseDir(), judgeCfg)

	judgeAgent, err := e.resolveJudgeAgent(ctx, judgeCfg, runAgent) // :679
	if err != nil { return nil, err }

	// 关键：裁判开始前，移除 run 阶段装的 skill！
	if err := e.removeDefaultRunSkillsBeforeJudge(ctx, rt, configName, judgeCfg, runAgent); err != nil { // :683
		return nil, err
	}
	if err := e.installJudgeSkills(ctx, rt, judgeCfg, judgeAgent); err != nil { // :686 装裁判自己的 skills
		return nil, err
	}

	j, err := judge.NewJudge(judgeCfg, judgeAgent, rt)
	// ...
}
```

`removeDefaultRunSkillsBeforeJudge`（`evaluator.go:719-754`）干的事：

- 只在 `configName == "with_skill"` 且 `judge.type == agent_judge` 时触发。
- 把 run 阶段装的 default skill（被测的 SKILL.md）**从 skill 目录里 `rm -rf` 掉**。
- 目的：**让裁判 agent 看不到被测 skill**，避免裁判「作弊」——否则裁判可能直接用被测 skill 给出高分，评测就失去意义了。
- 只删 default install target，不动 user 显式指定的 `target`（因为那是用户自己控制的，可能有跨 setup 共享的需要）。

**裁判中立原则**：裁判不能既当运动员又当裁判。这是评测公平性的底线。

---

### 步骤 ⑦ 合并 expect + judge 断言

最后一步在 `evaluateCaseSession` 末尾（`evaluator.go:527-535`）：

```go
finalResult := e.runJudgePhase(ctx, rt, caseCfg, configName, judgeCfg, turnsTotal, runAgent, judgeInput, result)
if len(expectAssertions) > 0 && finalResult.Grading != nil {
	finalResult.Grading.AssertionResults = append(expectAssertions, finalResult.Grading.AssertionResults...)
	finalResult.Grading.Summary.Passed += len(expectAssertions)
	finalResult.Grading.Summary.Total += len(expectAssertions)
	if finalResult.Grading.Summary.Total > 0 {
		finalResult.Grading.Summary.PassRate = float64(finalResult.Grading.Summary.Passed) / float64(finalResult.Grading.Summary.Total)
	}
}
return finalResult
```

- expect 通过后，把它的断言** prepend 到 judge 断言列表前面**，统一进 `grading.json`。
- 重新算 `Passed / Total / PassRate`。
- 为什么 prepend 而不是 append？expect 是「确定性的硬规则」，放在前面更醒目；judge 的 LLM 判分放在后面。报告里一眼就能看到「expect 全过了，judge 给了多少分」。

到这一步，单次 case 的完整生命周期结束，`EvalResult` 被返回给 `executeCase`，由它决定是否重试。

---

## 3. 关键设计点（5 条，带 Java 类比）

### 设计点 1：A/B 对照机制（with_skill vs without_skill）

- **机制**：`EvaluateAll` 对每个 case 生成两个 task（`evaluator.go:200-206`），`setupCaseEnvironment` 里 `if configName != "without_skill"` 决定**装不装 skill**（`evaluator.go:1013`）。其他条件（prompt、setup_steps、MCP、fixtures）完全一致。
- **意义**：技能净收益 = `with_skill - without_skill`。这是评测一个 Skill 是否真有用的**唯一可靠方法**——直接看 with_skill 的绝对分会受到「agent 本身能力」干扰（强 agent 不带 skill 也行）。
- **Java 类比**：A/B 测试平台分桶、JMH 的 baseline 对照、Spring Benchmark 的 control group。本质都是「控制变量法」。

### 设计点 2：expect 短路省钱

- **机制**：`runExpectPreCheck`（`evaluator.go:540-565`）在 judge 之前跑免费的字符串 / 文件检查，任一失败立刻 FAIL 并跳过 judge。
- **意义**：judge（尤其 agent_judge）要调 LLM，又贵又慢。expect 把「明显错得离谱」的 case 先挡掉，省 90% 不必要的 judge 调用。
- **Java 类比**：Servlet Filter 链短路、Spring Security Filter 的 `AccessDeniedHandler`、或者数据库查询里 `WHERE` 短路（先过滤再 join）。

### 设计点 3：`judge.Input` 解耦执行层与评测层

- **机制**：`judge.Input`（`judge.go:55-98`）是所有 judge 实现共用的入参 struct，由 `evaluateCaseSession`（`evaluator.go:503-516`）一次性组装。
- **意义**：执行层（agent 适配器）和评测层（judge 实现）只通过这一个 struct 通信。加新 agent（比如 cursor）不动 judge；加新 judge（比如 vector_similarity）不动 agent。
- **Java 类比**：策略模式（Strategy）的 Context 对象；或者 Spring MVC 的 `ModelAndView`——controller 把数据塞进 ModelAndView，view 解析器（不同的 judge）拿同一份数据自己渲染。

### 设计点 4：超时是 agent + judge 的总预算

- **机制**：`withCaseTimeout`（`evaluator.go:294, 873-883`）在 `executeCase` 包了一层 `context.WithTimeout`，传给 `executeCaseOnce`。这个 ctx 是 agent 和 judge 共享的。agent 跑超时，judge 也跑不了。
- **意义**：避免「agent 卡 5 分钟超时了，judge 又调 LLM 卡 5 分钟」的总耗时失控。把整个 case 的 wall-clock 钉死在一个预算里。
- **Java 类比**：Spring 的 `@Transactional(timeout=30)` 是整个事务的超时，不是单个 SQL 的；这里 `withCaseTimeout` 是整个 case 的，不是单个 agent 或 judge 的。
- **代码佐证**：`handleExecutionResult`（`evaluator.go:907-930`）的注释：
  > *the case timeout is treated as a single budget for agent + judge, so we do not try to salvage partial output by running the judge against a truncated agent transcript.*

### 设计点 5：错误不丢现场

- **机制**：
  - `collectGlobArtifacts`（`evaluator.go:456`）放在 `handleExecutionResult` 的 early-return **之前**，agent 失败也拉产物。
  - 收集时用 `context.WithoutCancel`（`artifacts_collect.go:73`）脱离父 cancel，保证 case 超时后还能跑 60s 收尾。
  - 收集的所有错误都只 `Warn` 不向上抛（`artifacts_collect.go:60-61`：`never fails the case`）。
- **意义**：失败 case 的现场最珍贵——没有产物就没法调试为什么 fail。这一条把「调试友好性」放到了「执行简洁性」前面。
- **Java 类比**：`try-catch-finally` 的 finally 块里强制 flush 日志和 dump 线程；或者 Chaos Engineering 工具失败时自动 collect core dump。

---

## 4. 边界与异常

### 4.1 退出码语义：非 0 ≠ FAIL

很多人会以为「agent 进程退出码非 0 = case 失败」，**这是错的**。看 `evaluator.go:465-478`：

```go
if sessionResult != nil && sessionResult.ExitCode != 0 {
	hasExitCodeCheck := caseCfg.Expect.ExitCode != nil
	hasJudge := judgeCfg.Type != ""
	if !hasExitCodeCheck && !hasJudge {
		// 既没配 expect.exit_code 也没配 judge → 直接 FAIL
		result.Status = judge.StatusFail
		result.Configuration = configName
		return result
	}
	// 否则继续走 expect / judge
	logging.DebugContextf(ctx, "Evaluator: case %s agent exited with code %d, proceeding to evaluation", caseCfg.ID, sessionResult.ExitCode)
}
```

- **退出码非 0 是「信号」，不是「结论」**。
- 如果用户显式配了 `expect.exit_code: [0, 2]`（比如 lint 工具 exit 2 表示有 warning 但不算错），那 exit != 0 完全可能是 PASS。
- 如果配了 judge，judge 会综合 exit_code + final_message + workspace_diff 给出最终结论。
- 只有「啥都没配」的退化情况才把 exit != 0 当 FAIL（兜底）。

### 4.2 多轮 post_condition 门控

多轮场景里，每一轮都可以挂 `post_condition`（`multiturn.go:491-530`）：

```go
reason := evaluatePostCondition(turn.PostCondition, tr.Response)
if reason == "" {
	return false, false  // 通过，继续下一轮
}

onFail := turn.PostCondition.OnFail
if onFail == "" { onFail = "fail" }
tr.Status = TurnFailed

if onFail == "skip_remaining" {
	// 后续轮次全标 TurnSkipped，但仍走 judge 评分
	for j := turnIdx + 1; j < len(turns); j++ {
		state.turnResults = append(state.turnResults, TurnResult{Status: TurnSkipped, ...})
	}
	return true, true
}
// on_fail == "fail" → 直接 FAIL，跳过 judge
return true, false
```

两种 `on_fail` 策略：

| `on_fail` | 行为 | 是否进 judge |
|---|---|---|
| `fail`（默认） | 整个 case 立刻 FAIL | 否 |
| `skip_remaining` | 跳过后续轮次，但已有数据进 judge | 是 |

`multiTurnStatus`（`multiturn.go:602-644`）还要细分：

- `TurnFailed` 后面跟着 `TurnSkipped` → 是 `skip_remaining`，**继续走 judge**。
- `TurnFailed` 后面没跟 `TurnSkipped` → 是 `fail`，**整 case FAIL**。

这个区分保证了 `skip_remaining` 的 case 不会被一刀切地 FAIL，已经跑出来的数据还能被 judge 利用。

### 4.3 judge 前移除 run skill：保证裁判中立

详见步骤 ⑥-c（`evaluator.go:719-754`）。要点：

- 只在 `with_skill` + `agent_judge` 时触发。
- 只删 default install target（用户显式 `target` 不动）。
- 用 `rm -rf` 删整个 skill 目录（`removeRuntimePath`，`evaluator.go:778-799`），且只在 POSIX shell 下跑（Windows 的 PowerShell 不动）。
- 删完才装 judge 自己的 skills（`installJudgeSkills`，`evaluator.go:698-717`）。

### 4.4 retry 只重试 ERROR，不重试 FAIL

`retryReasonForResult`（`evaluator.go:897-905`）只认 `Status == ERROR`。FAIL 是「评测结论」，重跑也没用（agent 是确定性的，再跑大概率还是 FAIL）；ERROR 是「执行故障」（超时、panic、网络抖动），值得重试。

### 4.5 多轮失败时 token 累计仍有意义

`accumulateCumulativeMetrics`（`multiturn.go:545-554`）处理某些 agent 返回「累计 token」而非「本轮增量」的情况——用 `max(result - base, 0)` 转成增量，避免负数。这是 agent CLI 实现差异的兼容层。

---

## 5. 面试 Q&A

### Q1：为什么 skill-up 要搞 with_skill / without_skill 两次跑？直接看 with_skill 分不行吗？

**A**：直接看绝对分会受到「agent 本身能力」干扰。比如 GPT-5 不带 skill 也能解 70% 的题，Claude-3.5 不带 skill 只能解 30%——如果你只看 with_skill 分，根本分不清是 skill 厉害还是 agent 厉害。A/B 对照把「有没有 skill」作为唯一变量，**净收益 = with_skill - without_skill**，才能客观评价 skill 的价值。这是控制变量法在评测里的标准做法，对应代码是 `evaluator.go:200-206`（生成两个 task）+ `evaluator.go:1013`（`if configName != "without_skill"` 跳过装 skill）。

### Q2：`executeCaseOnce` 里超时是怎么传递的？为什么 agent 超时了 judge 还能跑吗？

**A**：超时由父函数 `executeCase`（`evaluator.go:294`）通过 `context.WithTimeout` 设置，**这个 ctx 一路传给 `executeCaseOnce` → `Agent.Run` → `j.Evaluate`**。也就是说，**agent 和 judge 共享同一个超时预算**。如果 agent 跑超时了，ctx 已经 `DeadlineExceeded`，judge 调用时第一个 ctx 检查就会失败，judge 跑不动。这是有意设计的——避免「agent 卡 5 分钟超时，judge 又调 LLM 卡 5 分钟」的总耗时失控。代码注释在 `evaluator.go:907-911` 写得很清楚：`the case timeout is treated as a single budget for agent + judge`。

### Q3：expect 和 judge 有什么区别？为什么不直接用 judge？

**A**：
- **expect 是「零成本确定性硬门控」**：纯字符串/文件检查（`must_contain` / `files_exist` / `exit_code` 等），跑一次几毫秒，不调 LLM。
- **judge 是「贵但灵活的评分」**：尤其 `agent_judge` 要再调一次 LLM，又贵又慢。

如果 expect 已经判定 FAIL（比如 agent 输出根本没提到那个 bug），再花 judge 的钱就是浪费。所以 expect 失败立刻短路返回（`runExpectPreCheck`，`evaluator.go:540-565`）。但 expect 通过不等于 PASS——还要让 judge 细评。Java 类比：Servlet Filter 链短路 + Controller，Filter 是 expect，Controller 是 judge。

### Q4：`judge.Input` 这个 struct 为什么重要？

**A**：它是**执行层和评测层的唯一边界**。所有 judge 实现（rule_based / script / agent_judge）拿到的是同一个 Input（`evaluator.go:503-516` 组装）。好处：
1. **解耦**：加新 agent（比如 cursor 适配器）不动 judge；加新 judge（比如 vector_similarity）不动 agent。
2. **可测试**：构造一个 `judge.Input` 就能单测任何 judge，不需要真跑 agent。
3. **可观测**：Input 是天然的「评测上下文快照」，可以落盘供回放。

对应设计模式是策略模式（Strategy）：`Judge` 是 Strategy 接口（`judge.go:31-33`），Input 是 Context 对象。

### Q5：为什么失败 case 也要拉产物？怎么实现的？

**A**：失败 case 的现场最珍贵——没有产物就没法调试。代码上：
1. `collectGlobArtifacts`（`evaluator.go:456`）放在 `handleExecutionResult` early-return **之前**，保证 agent 失败也走到。
2. 在 `defer rt.Close()` **之前**，保证沙箱关掉前产物已拉出。
3. 用 `context.WithoutCancel(ctx)`（`artifacts_collect.go:73`）脱离父 cancel——case 超时后 ctx 已 done，但产物收集必须跑完，所以脱离父 cancel 自己挂 60s 新超时。
4. 所有错误只 `Warn` 不向上抛（注释原话 `never fails the case`），保证收集失败不会让 case 从 PASS 变 ERROR。

---

## 6. 一句话总结

> `executeCaseOnce` 是一条 **「prepareRuntime → setupEnvironment(A/B) → Agent.Run → collect artifacts → expect 短路 → judge 评分 → 合并断言」** 的线性流水线，外层用 `executeCase` 套了重试和总超时；它通过 `judge.Input` 这一个 struct 把执行层和评测层彻底解耦，靠 `without_skill` 跳过 `InstallSkill` 实现 A/B 对照，靠 expect 短路省下昂贵的 judge 调用，靠 `context.WithoutCancel` 保证失败 case 也能拉到调试现场——**这七个步骤串起来，就是 skill-up 整个评测框架最值得讲的一条主链路**。
