# Judge 评分系统三策略深度讲解（rule_based / script / agent_judge）

> 面向：Java 后端/全栈工程师，备简历 + 面试用。
> 目标：把 `internal/judge/` 这套评分子系统从契约、工厂、预检查、三种策略、跨平台派发、上下文物化，一路讲到设计模式与面试问答。
> 阅读姿势：建议同时打开 `internal/judge/*.go` 对照看，下文每个关键论点都用 `文件名:行号` 锚定真实代码。

---

## 0. 一句话定位 + Judge 在评测流水线中的位置

**Judge 是 skill-up 评测框架的"评分层"：给定一个 case 跑完后的所有产物（transcript / 最终输出 / 退出码 / workspace diff / 生成的文件），它要回答两个问题 ——（1）这个 case 是否通过？（2）每条断言的过/不过证据是什么？**

在流水线里它**不是**第一步。完整的执行 → 评分链路是这样的（参见 `internal/evaluator/evaluator.go`）：

```
[加载 case] → [构建 runtime + agent] → [Agent.Run 跑用例]
        ↓
[组装 judge.Input（执行层产物 → 评测层契约）]
        ↓
[expect 预检查]   ← 零成本、确定性，跑在 Judge 之前
   ├─ 失败 → 直接 FAIL，跳过 Judge（省钱！）
   └─ 通过 → 继续
        ↓
[Judge.Evaluate（本文件主题）]
   ├─ rule_based  ─→ 纯 Go 内存计算
   ├─ script      ─→ 子进程，退出码即结论
   └─ agent_judge ─→ 起一个独立 agent session 当裁判
        ↓
[judge.Result → grading.json]
```

关键代码位置（`evaluator.go`）：
- `judgeCfg := judge.MergeJudgeConfig(e.evalCfg.Judge, caseCfg.Judge)`（`evaluator.go:392`）—— 先把"评测级配置"与"用例级配置"合并。
- `expectResult := judge.CheckExpect(expectCfg, judgeInput)`（`evaluator.go:553`）—— expect 先跑。
- `if expectResult.Passed { return false }`（`evaluator.go:555-557`）—— 通过才继续，否则短路。
- `grading, err := j.Evaluate(ctx, judgeInput)`（`evaluator.go:643`）—— 真正的 Judge 调用。

> **Java 类比**：Judge 这一层相当于 JUnit 的 `Assert` + 自定义 `TestRule`，但比 JUnit 多了"用 LLM 当裁判"和"跑外部脚本"两种扩展形态。`expect` 相当于 `assumeXXX()`，不通过就跳过后续昂贵判定。

---

## 1. 数据边界：`judge.Input` / `Result` 逐字段讲

### 1.1 为什么说它是"执行层 ↔ 评测层统一契约"

在 `judge.go:30-33` 定义了所有策略的共同抽象：

```go
// judge.go:30-33
type Judge interface {
    Evaluate(ctx context.Context, in Input) (*Result, error)
}
```

**这个接口只有一个方法，而且只接受 `Input`、只返回 `Result`。** 这是典型的"策略模式"接口 —— 三种实现（`RuleBasedJudge` / `ScriptJudge` / `AgentJudge`）面对的是同一个数据契约。这意味着：

- 执行层（evaluator）只要按 `Input` 字段把产物组装好，不需要关心下游是哪种 Judge。
- 任何一种 Judge 都不能"额外要字段"。如果某条信息没在 `Input` 里，要么扩展 `Input`，要么不用它。
- 这等价于 Java 里 `Strategy` 模式：`interface Strategy { Result evaluate(Input in) }`，三个 `Impl` 共享同一个上下文对象。

### 1.2 `Input`：执行层往评测层送的"全量证据包"

`judge.go:55-98`：

| 字段 | 类型 | 含义 | 谁会用 |
|---|---|---|---|
| `CaseID` | `string` | 用例唯一 id | 日志、报告 |
| `Transcript` | `transcript.Transcript` | Agent 完整交互记录（含 tool_call） | rule_based 的 `tool_called`、agent_judge 的评审材料 |
| `FinalMessage` | `string` | 最后一轮 assistant 文本 | expect 的 `must_contain`、rule_based 的 `output_contains`、script 通过 env 传 |
| `ExitCode` | `int` | engine 进程退出码 | expect `exit_code`、rule_based `exit_code`、script env `EVAL_EXIT_CODE` |
| `WorkspacePath` | `string` | case 工作区绝对路径 | `files_exist` / `files_not_exist` / `file_contains`、script 的 `cwd` |
| `SkillDir` | `string` | SKILL.md 所在目录 | `golden_file`、`attachment` 的相对路径基准 |
| `WorkspaceDiff` | `string` | 跑完后的 git diff | agent_judge 的评审材料 |
| `GeneratedFiles` | `[]string` | Agent 新建的文件路径列表 | agent_judge 的评审材料 |
| `ArtifactDir` | `string` | 裁判 agent 写产物的目录 | 仅 agent_judge 用 |
| `SessionResult` | `*agent.SessionResult` | 完整 engine 输出 | "高级 Judge"备用 |
| `TurnResults` | `[]InputTurnResult` | 每轮结果（多轮用） | rule_based 的 `turn_response_contains` 等轮级断言 |
| `TurnsExecuted` | `int` | 实际执行了几轮 | 写入 `Result` |
| `TurnsTotal` | `int` | 用例定义了几轮 | 写入 `Result` |

注意 `InputTurnResult`（`judge.go:36-49`）里同时携带了 `Response`、`Transcript`、`Status`、`Reason` —— **多轮场景下，rule_based 可以针对"第 3 轮响应里是否包含 X"做断言**，这是单轮评测系统做不到的。

> **设计巧思**：`Transcript` 是包级类型 `pkg/transcript.Transcript`，跨"执行层 ↔ 评测层"复用，避免重复定义。Java 里相当于把 `Transcript` 抽到公共 API 模块，两边共用。

### 1.3 `Result`：直接对应 `grading.json` 的"评分报告"

`judge.go:112-142`，字段几乎和 `grading.json` 一一对应：

```go
type Result struct {
    Status           Status            `json:"status"`            // PASS/FAIL/SKIP/ERROR
    SkipReason       *string           `json:"skip_reason,omitempty"`
    ErrorReason      *string           `json:"error_reason,omitempty"`
    TurnsExecuted    int               `json:"turns_executed"`
    TurnsTotal       int               `json:"turns_total"`
    AssertionResults []AssertionResult `json:"assertion_results"`
    Summary          ResultSummary     `json:"summary"`
    JudgeSession     *agent.SessionResult `json:"-"`              // 不入 grading.json
    JudgeContext     *ContextMetadata  `json:"judge_context,omitempty"`
}
```

四类 Status（`judge.go:19-28`）的语义要分清：
- `PASS`：所有断言通过。
- `FAIL`：有断言没过（业务失败）。
- `SKIP`：主动跳过（如多轮 post_condition 触发 `skip_remaining`）。
- `ERROR`：执行异常（agent 调用炸了、脚本起不来等）。

**区分 FAIL vs ERROR 是关键设计**：FAIL 是"被评测对象做错了"，ERROR 是"评测系统自己出了问题"。Java 里相当于 `AssertionError` vs `RuntimeException`。

### 1.4 三个构造器：`NewResult` / `NewSkipResult` / `NewErrorResult`

这是 Result 的"工厂方法"，避免每个策略自己手算 Summary。

`NewResult`（`judge.go:213-252`）的核心逻辑：

```go
passed, failed := 0, 0
for _, a := range assertions {
    if a.Passed { passed++ } else { failed++ }
}
total := passed + failed
var passRate float64
if total > 0 {
    passRate = float64(passed) / float64(total)
} else {
    passRate = 1.0 // no assertions = vacuous pass
}
status := StatusPass
if failed > 0 {
    status = StatusFail
}
```

**两个细节**：
1. **零断言 = 空真（vacuous pass）**：`passRate=1.0`。语义上，"没要求"等于"无义务"，所以默认 PASS。这对 evaluator.go 里"没配置 Judge 就默认 PASS"的兜底（`evaluator.go:620-625`）是基础。
2. **`nil` 切片转 `[]AssertionResult{}`**：保证 `grading.json` 里永远是数组而不是 `null`，前端展示更稳定。

`NewSkipResult` / `NewErrorResult`（`judge.go:255-277`）把对应 reason 包成指针塞进去 —— 用指针是为了让 `omitempty` 在「非 SKIP/ERROR」场景能省略字段。

### 1.5 `SessionResultError`：错误链里抢救 Session

`judge.go:154-182`：

```go
type SessionResultError struct {
    Err     error
    Session *agent.SessionResult
}
```

这个设计很巧妙。`agent_judge` 跑到一半 LLM 调用失败时，**虽然返回了 error，但 session 已经产生了 transcript / artifacts**。如果直接把 error 往上抛，这些产物就丢了 —— 调试时无从下手。

`SessionResultError` 把 session 用 error 链携带上去，调用方可以用 `errors.As` 提取（`judge.go:176-182`）：

```go
func SessionResultFromError(err error) *agent.SessionResult {
    var withSession *SessionResultError
    if errors.As(err, &withSession) {
        return withSession.Session
    }
    return nil
}
```

evaluator 用它把 Judge 的 artifacts 也下载下来（`evaluator.go:645-649`）。

> **Java 类比**：相当于把一个附加上下文塞进 `Throwable.addSuppressed()` 或自定义异常的 `attachment` 字段。Go 没有异常链的语法糖，靠 `Unwrap()`（`judge.go:168-173`）配合 `errors.As` 实现。

### 1.6 `safePath`：路径遍历防御

`judge.go:302-313`：

```go
func safePath(workspace, rel string) (string, error) {
    abs := filepath.Join(workspace, rel)
    clean := filepath.Clean(abs)
    workspaceClean := filepath.Clean(workspace)
    if clean == workspaceClean {
        return clean, nil
    }
    if !strings.HasPrefix(clean, workspaceClean+string(filepath.Separator)) {
        return "", fmt.Errorf("path %q escapes workspace", rel)
    }
    return clean, nil
}
```

凡是 case 配置里写的相对路径（`files_exist: ["../../etc/passwd"]`、`golden_file: "../../../secret"`）都会被这层挡住。**评测框架尤其需要这种防御**：用例文件来自用户/三方仓库，必须假设它们是不可信输入。

注意它是**前缀比对 + `filepath.Separator`**，而不是简单的 `strings.HasPrefix(clean, workspaceClean)` —— 否则 `/tmp/work` 会被 `/tmp/work-evil` 匹配上。这是个经典坑，skill-up 写得很规范。

`fileExistsInWorkspace`（`judge.go:284-297`）就是 `safePath` + `os.Stat` 的组合。

---

## 2. 工厂 `NewJudge` + `MergeJudgeConfig`

### 2.1 `NewJudge`：type 字段 switch 派发

`factory.go:22-50`，标准的简单工厂：

```go
func NewJudge(cfg config.JudgeConfig, ag agent.Agent, rt runtime.Runtime) (Judge, error) {
    switch cfg.Type {
    case "rule_based":
        return NewRuleBasedJudge(cfg), nil
    case "script":
        if cfg.ScriptPath == "" {
            return nil, errors.New("script judge requires script_path")
        }
        return &ScriptJudge{
            ScriptPath:     cfg.ScriptPath,
            TimeoutSeconds: derefInt(cfg.TimeoutSeconds),
            Runtime:        rt,
        }, nil
    case "agent_judge":
        if ag == nil {
            return nil, errors.New("agent_judge requires an Agent")
        }
        return NewAgentJudgeWithContextAndSkills(ag, rt, cfg.Model, cfg.Criteria,
            cfg.PassThreshold, cfg.Context, derefInt(cfg.TimeoutSeconds),
            SkillInfosFromRefs(cfg.Skills)), nil
    case "":
        return nil, nil  // 没配置 → 调用方默认 PASS
    default:
        return nil, fmt.Errorf("unknown judge type: %q", cfg.Type)
    }
}
```

要点：
1. **依赖注入**：`ag`（agent）和 `rt`（runtime）由外部传入，不在这里 new。便于测试 mock。
2. **必备参数校验**：`script` 要 `ScriptPath`、`agent_judge` 要 `Agent`，缺失即报错。
3. **空 type 返回 `nil, nil`**：约定"无 Judge = 默认 PASS"，调用方在 `evaluator.go:620-625` 处理。
4. **`derefInt`**（`factory.go:53-58`）：把 `*int` 拍平，nil → 0。`TimeoutSeconds` 在 schema 里是 `*int`（"未配置"语义需要它），到具体 Judge 里则是 `int`（0 表示用默认）。

> **Java 类比**：相当于 `JudgeFactory.create(cfg, agent, runtime)`。Go 没有构造器重载，靠"函数 + switch"实现。

### 2.2 `MergeJudgeConfig`：case 级覆盖 eval 级

`factory.go:68-87`，这段语义要仔细品：

```go
func MergeJudgeConfig(global, caseLevel config.JudgeConfig) config.JudgeConfig {
    // If case has a type, use it entirely
    if caseLevel.Type != "" {
        merged := caseLevel
        // Fill in defaults from global if case doesn't specify
        if merged.Model == "" {
            merged.Model = global.Model
        }
        if merged.PassThreshold == nil {
            merged.PassThreshold = global.PassThreshold
        }
        if merged.TimeoutSeconds == nil {
            merged.TimeoutSeconds = global.TimeoutSeconds
        }
        merged.Context = MergeJudgeContextConfig(global.Context, caseLevel.Context)
        return merged
    }
    // No case-level judge, use global
    return global
}
```

**核心设计选择 ——「case 配了 Type 就是全量覆盖」**：

- 一旦 case 写了 `judge.type: script`，它就**不会**继承全局的 `rule_based` 规则，连 `Criteria` / `Success` / `Failure` / `ScriptPath` 也都只看 case 自己的。
- **只有三个字段会回退**：`Model`、`PassThreshold`、`TimeoutSeconds`。

为什么这么设计？看注释（`factory.go:62-67`）：

> When case-level specifies a Type, it is treated as a full override ... Other fields (Criteria, Success, Failure, ScriptPath) come from the case-level config exclusively — this is intentional to allow cases to define completely independent judge strategies.

也就是：**避免"全局 rule_based + case script"这种混搭把字段串到一起出怪 bug**。全量覆盖语义最清晰。

> **Java 类比**：Spring 的 `@PropertySource` 多级覆盖是"逐字段 merge"；这里是"分类型整体替换 + 少量共享默认值"，更接近"模板方法里 hook 方法被重写"的语义。

### 2.3 `MergeJudgeContextConfig`：context 字段的逐项 merge

`factory.go:91-123`：和上面相反，context 是**逐字段 merge**而非整体替换。原因：context 调节的是"评审材料怎么送"（profile / mode / limits / attachments），这些字段是"细节旋钮"，逐项 merge 更灵活。

- 任何一边为 nil 就克隆另一边。
- case 非 `""` 的字段覆盖 global。
- `Limits` 走 `mergeJudgeContextLimits`（`factory.go:125-145`）：只在 case 显式设了 `> 0` 的值时才覆盖，避免 case 的"未配置"清掉 global 的合理值。
- `Attachments` **整体替换**（`factory.go:119-121`）：因为 attachments 是个 list，逐项 merge 语义不清。**注意克隆**：`append([]config.JudgeContextAttachment(nil), caseLevel.Attachments...)` —— 避免共享底层数组导致后续误改。

`cloneJudgeContextConfig`（`factory.go:147-158`）的"深拷贝"细节：
- 顶层 struct 直接解引用拷贝。
- `Limits` 指针单独再拷一次。
- `Attachments` 切片单独再 append 一份。

> **Java 类比**：相当于 `Cloneable` + 手动深拷贝 List。Go 没有内置深拷贝约定，得手写。

---

## 3. expect 预检查：零成本短路省钱

### 3.1 设计目标

`expect.go:1-8` 的注释把意图说得明明白白：

> Expect is a zero-cost, deterministic gate that runs BEFORE any judge. If any expect check fails, the case is immediately marked FAIL and the (potentially expensive) judge execution is skipped entirely.

**"省钱"是关键**：agent_judge 要再起一个 LLM 调用，可能花几毛到几块钱；script 要起子进程；哪怕 rule_based 也要算多轮。**如果用例连"必须包含 hello"都没满足，那还浪费钱跑 Judge 干嘛？**

### 3.2 `CheckExpect` 主流程

`expect.go:48-90`：

```go
func CheckExpect(expect *config.Expect, in Input) *ExpectResult {
    if expect == nil {
        return &ExpectResult{Passed: true}
    }
    var failures []ExpectFailure
    var configuredRules []string

    if len(expect.MustContain) > 0 {
        configuredRules = append(configuredRules, "must_contain")
        failures = append(failures, checkMustContain(expect.MustContain, in.FinalMessage)...)
    }
    // ... 7 类规则依次 append
    return &ExpectResult{
        Passed:          len(failures) == 0,
        Failures:        failures,
        ConfiguredRules: configuredRules,
    }
}
```

特点：
1. **纯函数**（除了 file 相关的 `os.Stat` / `os.ReadFile`）。
2. **顺序固定**：must_contain → must_not_contain → exit_code → files_exist → files_not_exist → golden_file → file_contains。`ConfiguredRules` 按这个顺序记录，给后面 `ToAssertionResults` 提供排序依据。
3. **不短路**：和 rule_based 不同，expect 会跑完所有规则，收集全部 failures —— 因为 expect 很便宜，而且报告完整更好。

### 3.3 七类 check 实现

逐一看一下，每个都很简短：

- **`checkMustContain`**（`expect.go:128-139`）：`strings.Contains` 检查每个 keyword，缺哪个记哪个。
- **`checkMustNotContain`**（`expect.go:142-153`）：反向。
- **`checkExitCode`**（`expect.go:157-168`）：`*int` 为 nil 视为"未配置"。注意类型 —— schema 里 `Expect.ExitCode` 是 `*int`，nil 表示用例没写。
- **`checkFilesExist`**（`expect.go:171-190`）：调 `fileExistsInWorkspace`（即 `safePath` + `os.Stat`）。路径不合法也算 failure（不是 error）。
- **`checkFilesNotExist`**（`expect.go:193-212`）：反向。
- **`checkGoldenFile`**（`expect.go:216-249`）：把 final_message 跟"黄金文件"做完全字符串比对（`TrimSpace` 后相等）。基准目录优先用 `SkillDir`，否则用 `WorkspacePath`。
- **`checkFileContains`**（`expect.go:252-279`）：把指定文件读出来，检查 `strings.Contains`。

> **路径安全的下沉**：`checkFilesExist` / `checkGoldenFile` / `checkFileContains` 全部经过 `safePath`，所以 expect 这一层不会被 `../../` 类输入攻破。

### 3.4 `ToAssertionResults`：把规则结果转成 `AssertionResult`

`expect.go:96-121`：把 `ConfiguredRules` 顺序里每条规则汇总成**一条** `AssertionResult`（多条失败合并 evidence）。这样报告里 expect 段落永远是"每类一条"，不会因为 `must_contain` 配了 5 个 keyword 就出 5 行。

```go
failDetails := make(map[string][]string, len(r.Failures))
for _, f := range r.Failures {
    failDetails[f.Rule] = append(failDetails[f.Rule], f.Detail)
}
// Emit one AssertionResult per configured rule, in evaluation order.
for _, rule := range r.ConfiguredRules {
    if details, failed := failDetails[rule]; failed {
        results = append(results, AssertionResult{
            Text:     "expect." + rule,
            Passed:   false,
            Evidence: strings.Join(details, "; "),
        })
    } else {
        results = append(results, AssertionResult{
            Text:     "expect." + rule,
            Passed:   true,
            Evidence: "all checks passed",
        })
    }
}
```

**注意 text 前缀 `expect.`**：报告里一眼能区分"这条是 expect 预检查产生的"vs"这条是 Judge 真正产生的"。

### 3.5 `diffHint`：golden_file 不一致时给个差异提示

`expect.go:286-300`：实现是个朴素的双指针扫字符，找到第一个不同字符位置，然后截前后各 20 字符做提示：

```go
maxLen := min(len(expected), len(actual))
for i := range maxLen {
    if expected[i] != actual[i] {
        start := max(i-20, 0)
        end := min(i+20, maxLen)
        return fmt.Sprintf("first diff at offset %d: expected ...%q... got ...%q...",
            i, expected[start:end], actual[start:end])
    }
}
if len(expected) != len(actual) {
    return fmt.Sprintf("length mismatch: expected %d chars, got %d chars", ...)
}
```

为什么不直接上 `diff` 库？因为 expect 是"零成本"层，引入外部依赖和复杂度不划算，一段提示足够人定位问题。这是个"够用就好"的工程判断。

---

## 4. 策略一：`rule_based` —— 纯 Go 声明式断言

### 4.1 数据模型

`RuleBasedJudge`（`rule_based.go:17-22`）只有两个字段：

```go
type RuleBasedJudge struct {
    Success []config.Rule  // 全部通过 = PASS
    Failure []config.Rule  // 任一命中 = FAIL
}
```

`config.Rule`（`schema.go:190-200`）是个"tagged union"（标签联合体），9 种断言类型各占一个指针字段：

```go
type Rule struct {
    OutputContains          *OutputContainsRule
    ExitCode                *int
    ToolCalled              *ToolCalledRule
    FilesExist              []string
    FilesNotExist           []string
    TurnResponseContains    *TurnResponseContainsRule
    TurnResponseNotContains *TurnResponseNotContainsRule
    ToolCalledInTurn        *ToolCalledInTurnRule
    ToolNotCalledInTurn     *ToolNotCalledInTurnRule
}
```

YAML 反序列化后，只有用户写的那一项非空。`evaluateAssertion`（`rule_based.go:75-102`）就是一长串 `case` 把非空的那一项派发到对应的 `evalXxx` 函数。

> **Java 类比**：tagged union 在 Java 里通常用 sealed interface + records 表达（JDK 17+），或者用抽象基类 + 多个 subclass。Go 没有 sealed class，靠"多个字段只有一个非 nil"约定。

### 4.2 核心算法：**先 failure 短路，再 success 全检**

这是 rule_based 最值得讲的设计（`rule_based.go:33-67`）：

```go
func (j *RuleBasedJudge) Evaluate(_ context.Context, in Input) (*Result, error) {
    var allAssertions []AssertionResult

    // 1. Check failure rules first — any match is immediate FAIL.
    for _, rule := range j.Failure {
        ar := evaluateAssertion(rule, in)
        if ar.Passed {
            // A failure rule "passed" means the bad pattern WAS found → FAIL.
            allAssertions = append(allAssertions, AssertionResult{
                Text:     "failure: " + ar.Text,
                Passed:   false,
                Evidence: "failure rule matched: " + ar.Evidence,
            })
        }
        // If the failure rule did not match, we don't record it (it's good).
    }

    // Short-circuit: if any failure rule matched, return immediately.
    if len(allAssertions) > 0 {
        return NewResult(allAssertions, in.TurnsExecuted, in.TurnsTotal), nil
    }

    // 2. Check success rules — all must pass.
    for _, rule := range j.Success {
        ar := evaluateAssertion(rule, in)
        allAssertions = append(allAssertions, ar)
    }
    ...
}
```

#### 4.2.1 failure 语义反转（重点！）

注意上面那段代码：**failure 规则"命中"时，`ar.Passed` 是 `true`**（因为底层 `evalOutputContains` 不知道自己被用于 failure 还是 success），但 RuleBasedJudge 在外层**把它的 Passed 反转成 `false`**！

代码里的注释（`rule_based.go:40-44`）说得很直白：

> A failure rule "passed" means the bad pattern WAS found → FAIL.

举例：
- 配置 `failure: [{output_contains: {any: ["panic", "fatal error"]}}]`
- 实际输出里出现了 "panic"
- `evalOutputContains` 返回 `Passed: true`（"any 命中了"）
- RuleBasedJudge 反转 → `Passed: false`，记录 `text: "failure: output_contains.any: ..."`

这个反转设计让 `evaluateAssertion` 保持"语义纯粹"（passed = 条件成立），由调用方决定"条件成立是好事还是坏事"。**这是策略模式里"复用底层算子"的经典做法。**

#### 4.2.2 短路省钱

如果第一条 failure 命中，后面的 failure 和 success 都不跑了 —— `return NewResult(...)`。这避免了：
- 多条 failure 同时命中时报告里 noise 过多。
- success 规则里有昂贵检查（如读大文件）时白跑。

> **Java 类比**：相当于 `if (anyMatch(failures)) return FAIL; else if (allMatch(successes)) return PASS; else return FAIL;` 的短路流。

### 4.3 支持的 9 种断言

按 `evaluateAssertion` 的 switch 顺序列出：

| 类型 | 适用场景 | 实现位置 |
|---|---|---|
| `output_contains` (all/any/not) | 终态输出含/不含关键词 | `rule_based.go:109-175` |
| `exit_code` | 进程退出码精确匹配 | `rule_based.go:178-191` |
| `tool_called` | 全局检查某 tool 是否被调用过 | `rule_based.go:194-234` |
| `files_exist` | 工作区下指定文件存在 | `rule_based.go:237-260` |
| `files_not_exist` | 反向 | `rule_based.go:263-286` |
| `turn_response_contains` (contains_all/any) | 第 N 轮响应含关键词 | `rule_based.go:341-395` |
| `turn_response_not_contains` | 第 N 轮响应不含关键词 | `rule_based.go:398-419` |
| `tool_called_in_turn` | 第 N 轮调用了某 tool | `rule_based.go:422-463` |
| `tool_not_called_in_turn` | 第 N 轮没调用某 tool | `rule_based.go:466-491` |

`evalOutputContains`（`rule_based.go:109-175`）特别值得注意：它**同时**支持 `all` / `any` / `not` 三种语义，而且**all 和 not 是硬失败（任一不满足就 fail），any 是"至少一个"**。一条规则可以混合三种语义：

```yaml
output_contains:
  all: ["null pointer", "fix"]
  any: ["fixed", "patched"]
  not: ["TODO"]
```

### 4.4 `lookupTurn`：多轮场景的"按轮查找 + 状态校验"

`rule_based.go:315-338`：

```go
func lookupTurn(turnNum int, turns []InputTurnResult, ruleName string) (*InputTurnResult, *AssertionResult) {
    if turnNum >= 1 {
        for i := range turns {
            if turns[i].TurnNumber == turnNum {
                tr := &turns[i]
                if tr.Status != "completed" {
                    ar := AssertionResult{
                        Text:     fmt.Sprintf("%s[turn=%d]", ruleName, turnNum),
                        Passed:   false,
                        Evidence: fmt.Sprintf("turn %d has status %q (reason: %s); assertion requires completed turn", turnNum, tr.Status, tr.Reason),
                    }
                    return nil, &ar
                }
                return tr, nil
            }
        }
    }
    ar := AssertionResult{
        Text:     fmt.Sprintf("%s[turn=%d]", ruleName, turnNum),
        Passed:   false,
        Evidence: fmt.Sprintf("turn %d does not exist (executed %d turns)", turnNum, len(turns)),
    }
    return nil, &ar
}
```

设计要点：
- **第二个返回值是 `*AssertionResult`**（注意是指针）：nil 表示"找到了且状态 OK"，非 nil 表示"已经为你准备好失败断言了，直接用"。
- **状态校验**：即使轮次存在，如果 `Status != "completed"`（如 `skipped` / `failed` / `error`），也直接生成失败断言。这避免了"用第 3 轮的失败响应当成功响应检查"的逻辑漏洞。
- **1-based index**：`turnNum >= 1` 才查，配置里写 `turn: 0` 或负数直接失败。

调用方代码（`rule_based.go:341-345`）很优雅：

```go
tr, failAR := lookupTurn(rule.Turn, turns, "turn_response_contains")
if failAR != nil {
    return *failAR
}
// 用 tr 继续做 contains 检查
```

> **Java 类比**：这种"返回 `Optional<Failure>`"的写法相当于 `Either<AssertionResult, InputTurnResult>`，Go 没有 Either，用双返回值模拟。

### 4.5 `argsMatch`：tool 参数部分匹配

`rule_based.go:294-306`：

```go
func argsMatch(expected, actual map[string]any) bool {
    for k, ev := range expected {
        av, ok := actual[k]
        if !ok {
            return false
        }
        // Compare as strings for flexibility (YAML values may deserialize differently).
        if fmt.Sprintf("%v", ev) != fmt.Sprintf("%v", av) {
            return false
        }
    }
    return true
}
```

两点设计：
1. **只校验 expected 里出现的字段**：actual 多出来的字段忽略。这是"部分匹配"，符合"我只关心这个参数对不对"的语义。
2. **字符串化比较**：YAML 里 `1.0` 可能解析成 `float64(1)`，JSON 里可能是 `int(1)`，直接 `==` 不等。`fmt.Sprintf("%v", ...)` 把两者都拍成字符串 `"1"` / `"1"`，绕开类型不一致问题。

> **代价**：`fmt.Sprintf` 比 `==` 慢，但断言匹配不在热路径，可接受。这种 trade-off 在 Java 里也常见 —— `Objects.toString(a).equals(Objects.toString(b))`。

---

## 5. 策略二：`script` —— 用户自写脚本，退出码即结论

### 5.1 契约（极简且跨语言）

`script.go:17-23` 的注释写明：

> Script contract (design doc):
> - Working directory: case workspace root
> - Environment variables: EVAL_TRANSCRIPT_PATH, EVAL_FINAL_MESSAGE, EVAL_EXIT_CODE
> - Exit code 0 → PASS, non-0 → FAIL
> - stdout → evaluation evidence, stderr → debug info

这个契约最大的好处是**语言中立**：用户可以用 Python / Bash / PowerShell / Node 写评分脚本，只要遵守"退出码 = 结论"即可。这相当于把"评分逻辑"外置成"用户插件"。

### 5.2 `ScriptJudge` 结构

`script.go:24-38`：

```go
type ScriptJudge struct {
    ScriptPath     string          // 用户脚本的本地路径
    TimeoutSeconds int             // <=0 用 DefaultScriptTimeout (30s)
    TranscriptPath string          // 序列化好的 transcript.json 本地路径
    Runtime        evalruntime.Runtime
}
```

**`Runtime` 字段是关键**：脚本不是直接在 host 上跑，而是被**上传到 runtime**（可能是 docker、可能是远程、可能是本地 none）里执行。这和"被执行的 Agent 跑在同一个 runtime 里"保持一致 —— 评分脚本看到的工作区、文件系统、shell 跟被评测对象一模一样，**避免环境差异带来的判定偏差**。

### 5.3 `Evaluate` 流程

`script.go:41-60`：

```go
func (j *ScriptJudge) Evaluate(ctx context.Context, in Input) (*Result, error) {
    if j.TranscriptPath == "" {
        log.Println("warning: ScriptJudge.TranscriptPath is empty; EVAL_TRANSCRIPT_PATH will be unset")
    }
    timeout := DefaultScriptTimeout
    if j.TimeoutSeconds > 0 {
        timeout = time.Duration(j.TimeoutSeconds) * time.Second
    }
    ctx, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()
    rt, cleanup, err := j.runtime(ctx)
    if err != nil {
        return nil, err
    }
    defer cleanup()
    return j.evaluateInRuntime(ctx, rt, in, timeout)
}
```

要点：
1. **Timeout 默认 30s**（`script.go:15`）：脚本卡死不会拖垮整个评测。
2. **`context.WithTimeout`**：用 context 而不是 sleep + kill，标准 Go 做法。
3. **runtime 注入**：`j.runtime(ctx)`（`script.go:62-74`）如果 `j.Runtime` 为 nil，就现场创建一个 `none` runtime（host 本地执行）。

### 5.4 `evaluateInRuntime`：上传脚本、跑、收结果

`script.go:76-146` 的步骤：

1. **`planScript(j.ScriptPath, shell)`**（行 79）：决定"用什么命令跑这个脚本"（见下一节 interpreter.go）。
2. **创建临时目录**（行 84）：`judgeTempDir(targetGOOS)` —— Windows 用 `os.TempDir()`，POSIX 用 `/tmp`。
3. **上传脚本**（行 86-88）：`rt.UploadFile(ctx, j.ScriptPath, remoteScript)`。
4. **defer cleanup**（行 89-91）：跑完删临时目录。
5. **上传 transcript**（行 93-99）：如果有 transcript，序列化成 `transcript.json` 也传过去。
6. **构造命令 + cwd + env**（行 101-123）：

```go
result, err := rt.Exec(ctx, command, evalruntime.ExecOptions{
    Cwd:        cwd,
    TimeoutSec: int(timeout.Seconds()),
    Env: map[string]string{
        "EVAL_TRANSCRIPT_PATH": transcriptEnv,
        "EVAL_FINAL_MESSAGE":   in.FinalMessage,
        "EVAL_EXIT_CODE":       strconv.Itoa(in.ExitCode),
    },
})
```

7. **处理结果**（行 124-145）：
   - `DeadlineExceeded` → 当 FAIL，evidence 写"超时"。
   - 退出码 != 0 → FAIL，evidence 用 stdout（没有就用"exit code N"）。
   - 退出码 == 0 → PASS，evidence 用 stdout（没有就用默认文案）。

### 5.5 `buildResult`：把 stdout/stderr 拼成 evidence

`script.go:149-161`：

```go
func (j *ScriptJudge) buildResult(passed bool, evidence, debugInfo string, in Input) *Result {
    if debugInfo != "" {
        evidence = evidence + " [stderr: " + debugInfo + "]"
    }
    assertion := AssertionResult{
        Text:     "script: " + j.ScriptPath,
        Passed:   passed,
        Evidence: evidence,
    }
    return NewResult([]AssertionResult{assertion}, in.TurnsExecuted, in.TurnsTotal)
}
```

一个脚本 = 一条断言。stderr 会被附在 evidence 末尾（`[stderr: ...]`），方便排查但不会盖过 stdout 的结论。

### 5.6 `EVAL_TRANSCRIPT_PATH` 的跨平台路径转换

`script.go:106-114` 有个细节：

```go
// Translate the transcript path into the script interpreter's preferred
// form (e.g. forward slashes for `.sh` running under Git Bash so POSIX
// tools can `cat "$EVAL_TRANSCRIPT_PATH"`).
transcriptEnv := remoteTranscript
if remoteTranscript != "" {
    transcriptEnv = plan.envPath(remoteTranscript)
}
```

`plan.envPath` 是个函数指针：
- POSIX 计划：`identityEnvPath`（原样返回）。
- Windows `.sh` 计划：`filepath.ToSlash`（反斜杠 → 正斜杠）。

为什么？因为 Git Bash 里的 POSIX 工具（`cat`、`grep`）不认 Windows 反斜杠。如果不转，脚本里 `cat "$EVAL_TRANSCRIPT_PATH"` 会失败。**这种细节就是工程经验和"业余评测框架"的分水岭。**

---

## 6. 跨平台派发：`interpreter.go`（脚本怎么跑由它决定）

### 6.1 `planScript` 的二分叉

`interpreter.go:47-65`：

```go
func planScript(scriptPath string, shell platform.Shell) (scriptPlan, error) {
    if err := shell.Validate(); err != nil {
        return scriptPlan{}, fmt.Errorf("invalid runtime shell: %w", err)
    }
    if shell.GOOS != platform.GOOSWindows {
        return scriptPlan{
            uploadName: "script",
            command: func(remoteScript string) string {
                q := shellquote.QuotePOSIX(remoteScript)
                return "chmod 700 " + q + " && " + q
            },
            cleanupCommand: func(dir string) string {
                return "rm -rf " + shellquote.QuotePOSIX(dir)
            },
            envPath: identityEnvPath,
        }, nil
    }
    return planWindowsScript(scriptPath, shell)
}
```

**POSIX 极简**：上传脚本原样不改名，`chmod 700` + 执行。**所有解释器选择交给内核的 shebang 机制**（`#!/usr/bin/env python3` 内核直接派发）。Go 代码不掺和。

**Windows 复杂**：内核不认 shebang（除非配置过），所以要在 Go 代码里根据**扩展名 + shebang**决定怎么调解释器。这就是 `planWindowsScript` 要做的事。

`scriptPlan` 是个"行为包"（`interpreter.go:18-35`），用三个闭包打包：
- `command(remoteScript)`：怎么跑脚本。
- `cleanupCommand(dir)`：怎么删临时目录（**和 command 用同一个 shell 的 quoter，保证引号语义一致**）。
- `envPath(p)`：路径转换。

> **Java 类比**：这种"用闭包打包一组相关行为"等价于返回一个实现 `ScriptPlan` 接口的匿名类。Go 没有匿名类，但函数是一等公民，写起来更轻。

### 6.2 `planWindowsScript`：四种扩展名分支

`interpreter.go:67-185`：

1. 先 `parseShebang(readShebang(scriptPath))`（行 77）拿到 `shebangInterp` 和 `shebangOpts`。
2. 取扩展名（行 79-82）：没有扩展名就从 shebang 推导一个合成扩展名（`extensionForShebangInterpreter`，行 230-242）。
3. `winCleanup`（行 90-92）统一用 `cmd /d /s /c rd /s /q <dir>` 删目录 —— **`.ps1`/`.cmd`/`.bat` 三种都跑过 cmd.exe，cleanup 也走 cmd，引号剥离规则一致**。
4. switch 扩展名：

#### `.ps1` 分支（`interpreter.go:96-132`）

- 默认用 `powershell.exe`（Windows PowerShell 5.x）。
- shebang 显式声明 `pwsh`（PowerShell Core 7+）才用 `pwsh`。
- **转发 shebang 里的 flag**（行 110-118）：`#!/usr/bin/env -S pwsh -NoLogo` 里的 `-NoLogo` 不会丢。
- 始终追加 `-NoProfile -ExecutionPolicy Bypass -File`（行 119）：
  - `-NoProfile` 避免用户 profile 污染执行环境。
  - `-ExecutionPolicy Bypass` 绕开 Windows 默认的 Restricted 策略（否则未签名脚本会被拒）。

注意行 126 的注释强调了**防御性拷贝**：

```go
args := append([]string{}, psArgs...)
```

因为闭包捕获的 `psArgs` 可能在多次调用间被共享，如果直接 `append(psArgs, x)` 会改到底层数组。这是个 Go 切片坑。

#### `.cmd` / `.bat` 分支（`interpreter.go:133-141`）

最简单：`cmd /d /s /c <script>`。
- `/d`：禁用 AutoRun 注册表键。
- `/s`：改 `"` 的剥离规则（成对剥离）。
- `/c`：执行完退出。

#### `.sh` 分支（`interpreter.go:142-178`）

- 要求 `shell.IsBash()`（行 143-145）：Windows 上跑 `.sh` 必须有 Git Bash。
- **`envPath: filepath.ToSlash`**（行 177）：路径转成正斜杠，给 bash 里的 POSIX 工具用。
- **cleanup 也走 bash**（行 174-176）：`rm -rf` 配合正斜杠路径，避免 `bash → cmd` 的额外跳转。

#### 默认分支（行 179-184）

报错 "cannot determine interpreter"，引导用户加扩展名或 shebang。

### 6.3 shebang 解析：`parseShebang` + `parseEnvShebang`

`interpreter.go:284-293`：

```go
func parseShebang(body string) (string, []string) {
    fields := strings.Fields(body)
    if len(fields) == 0 {
        return "", nil
    }
    if filepath.Base(fields[0]) == "env" {
        return parseEnvShebang(fields[1:])
    }
    return filepath.Base(fields[0]), append([]string{}, fields[1:]...)
}
```

`body` 是 `#!` 后面的内容（去掉了 `#!`）。三种形态：
1. `#!/bin/bash -eu` → 直接路径 + opts，返回 `("bash", ["-eu"])`。
2. `#!/usr/bin/env python3` → `env` + 单个解释器名，返回 `("python3", [])`。
3. `#!/usr/bin/env -S pwsh -NoLogo` 或 `#!/usr/bin/env -Sbash -c "echo hi"` → `env -S` 形态，由 `splitStringInterpreter` 处理。

### 6.4 `parseEnvShebang`：GNU `env` flag 解析

`interpreter.go:343-395` 是个状态机，处理 GNU coreutils `env` 的各种 flag 形态：

| 形态 | 处理 |
|---|---|
| `-S body` / `--split-string body` | 行 347-351：把后续 token 重新 join，交给 `splitStringInterpreter` |
| `--split-string=body` | 行 352-363：取等号后部分 + 后续 token |
| `-Sbody`（紧凑） | 行 364-376：剥 `-S` 前缀 |
| `-u VAR` / `-C DIR` / `-a ARG0` / `--unset FOO` 等 | 行 377-381：跳过 flag + value（不把它当成解释器） |
| 其他 `-xxx`（如 `-i`、`--ignore-environment`） | 行 382-385：只跳一个 token |
| 第一个非 flag token | 行 386-391：解释器！调用 `interpreterFromArgs`（行 400-409）会先跳过 `NAME=VALUE` 形式的环境变量赋值 |

特别注意 `envValueTakingLongFlags` 的注释（行 304-317）解释了**为什么 `--ignore-signal` 不在表里**：

> GNU env documents those as OPTIONAL-argument flags (`[=sig]`) -- without the `=sig` suffix they take no value, so consuming the next token would swallow the interpreter (e.g. `env --ignore-signal bash` would otherwise be parsed as flag `--ignore-signal` with value `bash` and no interpreter remaining).

这是精读 GNU 文档才发现的坑。

### 6.5 `tokenizeShebangSplitString`：完整状态机

`interpreter.go:455-483` + 三个 step 函数（`tokenizeStepSingle/Double/Unquoted`，行 485-555）。

状态机有四种状态：
- 默认（unquoted）
- `inSingle`（在 `'...'` 里）
- `inDouble`（在 `"..."` 里）
- 加上 `skipNext`（下一个字符被反斜杠吃了）和 `done`（遇到 `\c` 提前终止）

`tokenizeStepUnquoted`（行 520-555）是最复杂的：
- 空白字符 → `flush()`（落 token）。
- `'` / `"` → 进入对应状态。
- `\` 后跟特殊字符：
  - `\c` → GNU env -S 的"忽略剩余"，设 `done = true`。
  - `\_` / `\t` / `\n` / `\r` / `\v` / `\f` → 这些转义解码为空白，**当 token 分隔符**（行 511-518 的 `envSWhitespaceEscapes` map）。
  - 其他 `\x` → 写入 `x`（去反斜杠）。

为什么要支持 `\_`？因为内核 shebang 是"一个参数"模式（`-S` 后的整个字符串会被 kernel 拆成多个 argv），`env -S` 的文档规定 `\_` 表示空格，是"在没引号的情况下塞空格"的合法手段（如 `bash\_-eu`）。

> **Java 类比**：这种手写 tokenizer 相当于写一个 mini `StreamTokenizer`，但目标窄得多 —— 只为 env -S 服务。**为什么不用正则？** 注释（行 356-363）说得很清楚：嵌套大括号 + 转义引号让正则脆弱。状态机才是健壮方案。

---

## 7. 策略三：`agent_judge` —— LLM 当裁判

这是最复杂、也最有"AI 工程味"的策略。整个流程：**物化上下文 → 构建 prompt → Agent.Run → 解析 JSON → 校验防作弊 → 阈值判定**。

### 7.1 `AgentJudge` 结构

`agent_judge.go:55-81`：

```go
type AgentJudge struct {
    Agent          agent.Agent
    Runtime        runtime.Runtime
    Model          string
    Criteria       []string         // 评分标准（用户写的若干条文字描述）
    PassThreshold  float64          // 默认 0.7（DefaultPassThreshold）
    TimeoutSeconds int              // 单次 Evaluate 超时
    JudgeSkills    []SkillInfo      // 强制裁判加载的 Skill
    Context        *config.JudgeContextConfig
}
```

`DefaultPassThreshold = 0.7`（`agent_judge.go:19`）—— LLM 裁判不可能 100% 准确，所以默认"10 条过了 7 条就算 PASS"，比 rule_based 的"全过"宽容。

### 7.2 `Evaluate` 主流程

`agent_judge.go:116-176`，分步讲：

#### Step 1：空 criteria 直接返回 PASS

```go
if len(j.Criteria) == 0 {
    return NewResult(nil, in.TurnsExecuted, in.TurnsTotal), nil
}
```

无标准 = 无义务 = vacuous pass（和 `NewResult` 一致）。

#### Step 2：context 超时

```go
parentCtx := ctx
if j.TimeoutSeconds > 0 {
    var cancel context.CancelFunc
    ctx, cancel = context.WithTimeout(ctx, time.Duration(j.TimeoutSeconds)*time.Second)
    defer cancel()
}
```

**注意 `parentCtx := ctx`** —— 在套 timeout 之前**先快照父 ctx**。后面要靠它判断"是 judge 自己超时，还是 case 上游超时"（见 7.7）。

#### Step 3：物化评审材料

```go
materialized, err := MaterializeJudgeContext(ctx, j.Runtime, j.Context, in, in.ArtifactDir)
```

把 final_message、transcript、workspace_diff、generated_files、attachments 写盘 + 上传到 runtime（详见第 8 节）。

#### Step 4：构建 prompt

```go
prompt := buildJudgePrompt(ctx, j.Criteria, materialized, j.JudgeSkills)
messages := []transcript.Message{{Role: transcript.RoleUser, Content: prompt, Turn: 1}}
```

prompt 是一段精心设计的文本（见 7.5）。

#### Step 5：调 Agent.Run

```go
sessionResult, err := j.Agent.Run(ctx, j.Runtime, agent.ExecOptions{ArtifactDir: in.ArtifactDir}, messages)
parentExpired := parentCtx.Err() != nil
```

**`parentExpired` 立刻快照** —— 注释（`agent_judge.go:137-142`）专门解释为什么要在 Run 返回的瞬间读：

> Snapshot parentCtx.Err() the instant Run returns, before the parent's timer has any chance to fire on its own; this is how we distinguish a judge-level deadline from a parent (case-level) one and decide whether to annotate. Reading parentCtx.Err() later would race against the parent timer in the caseTimeout ≈ judgeTimeout boundary.

这是个典型的"并发竞争窗口"问题：如果 case timeout 和 judge timeout 设置得很接近，Run 返回后再延迟读 `parentCtx.Err()`，父 timer 可能恰好触发，误判成"父超时"。

错误处理（行 145-157）：

```go
if err != nil {
    annotated := err
    if !parentExpired {
        annotated = j.annotateTimeoutError(ctx, err)
    }
    if !canRecoverAgentJudgeResult(err, sessionResult) {
        return nil, &SessionResultError{
            Err:     fmt.Errorf("agent_judge agent call failed: %w", annotated),
            Session: sessionResult,
        }
    }
    logging.WarnContextf(ctx, "agent_judge recovering judge output despite agent error: %v ...", err, ...)
}
```

也就是：能恢复（有 final message 且非 Cancel）就继续；否则把 session 包进 `SessionResultError` 抛上去。

#### Step 6：解析 JSON + 校验

```go
var resp judgeResponse
if err := extractJSON(sessionResult.FinalMessage, &resp); err != nil {
    return nil, &SessionResultError{...}
}
criterionResults := resp.Results
if err := validateAgentJudgeResponse(j.Criteria, criterionResults); err != nil {
    return nil, &SessionResultError{...}
}
```

四层兜底解析（7.6）+ 防作弊校验（7.8）。

#### Step 7：阈值判定 buildResult

```go
return j.buildResult(in, sessionResult, materialized, prompt, criterionResults), nil
```

### 7.3 `buildResult`：阈值制 vs rule_based 全过制

`agent_judge.go:178-197`：

```go
func (j *AgentJudge) buildResult(in Input, sessionResult *agent.SessionResult, materialized *MaterializedContext, prompt string, criterionResults []CriterionResult) *Result {
    assertions := make([]AssertionResult, 0, len(criterionResults))
    for _, cr := range criterionResults {
        assertions = append(assertions, AssertionResult{
            Text:     cr.Criterion,
            Passed:   cr.Passed,
            Evidence: cr.Evidence,
        })
    }

    result := NewResult(assertions, in.TurnsExecuted, in.TurnsTotal)
    result.JudgeSession = sessionResult
    result.JudgeContext = buildContextMetadata(sessionResult, materialized, prompt)
    if len(assertions) > 0 && result.Summary.PassRate >= j.PassThreshold {
        result.Status = StatusPass
    } else if len(assertions) > 0 {
        result.Status = StatusFail
    }
    return result
}
```

关键区别：
- `NewResult` 默认"有 failed 就 FAIL"（全过制）。
- AgentJudge **覆盖 Status**：只要 `PassRate >= PassThreshold`（默认 0.7），即使有 failed 也算 PASS。
- 这是个**软判定**模型，承认 LLM 裁判的噪声。

> **Java 类比**：相当于把 JUnit 的 `assertTrue` 改成了"10 个 assert 过 7 个就算测试通过"。这种"软判定"在模糊评测（代码风格、文档质量）场景是合理的。

### 7.4 `buildJudgePrompt`：提示工程

`agent_judge.go:442-468` 拼接出的 prompt 大致结构：

1. **角色设定**："You are an expert evaluator for an AI agent skill evaluation."
2. **任务说明**：要求对每条 criterion 给出 `passed` + `evidence`。
3. **强制规则**：`You MUST NOT pass a criterion without specific evidence.` —— 反作弊的第一道闸门，写进 prompt。
4. **JSON 格式约束**（行 455-456）：要求转义双引号。
5. **可选 Judge Skill 说明**（`appendJudgeSkillInstructions`，行 470-488）：如果配置了 `judge.skills`，强制要求裁判先调用 `Skill` 工具加载每个 Skill 的 rubric。
6. **Criteria 列表**：编号列出每条标准。
7. **Review Materials 表格**（`appendReviewMaterials`，行 490-517）：用 Markdown 表格列出每个材料（key、mode、path、bytes、是否截断），inline 的材料再单独贴出来。
8. **Required Response Format**（`appendRequiredResponseFormat`，行 519-534）：贴一段模板 JSON，每条标准的 `criterion` 字段已经预先填好。

**模板里把 `criterion` 字段填好是个反作弊细节** —— 强制 LLM 用我们提供的原文，避免它改写后影响 `validateAgentJudgeResponse` 的数量匹配。

### 7.5 `extractJSON` 四层兜底（重点！）

`agent_judge.go:229-259`，LLM 输出从来不可靠，所以这是个层层降级的解析链：

```go
func extractJSON(output string, v any) error {
    // Layer 1: 直接当 JSON 解
    candidates := []string{output}
    // Layer 2: 抽 ```json ... ``` 围栏里的内容
    if fenced := extractFencedJSON(output); fenced != "" {
        candidates = append(candidates, fenced)
    }

    // 对每个候选先试一次标准 Unmarshal
    for _, candidate := range candidates {
        if err := json.Unmarshal([]byte(candidate), v); err == nil {
            return nil
        }
    }

    // Layer 3: 扫所有 "看起来像 JSON 对象" 的子串
    for _, candidate := range findJSONObjectCandidates(output) {
        if err := json.Unmarshal([]byte(candidate), v); err == nil {
            return nil
        }
    }

    // Layer 4: 引号修复后再扫一遍
    repaired := repairJSONQuotes(output)
    if repaired != output {
        for _, candidate := range findJSONObjectCandidates(repaired) {
            if err := json.Unmarshal([]byte(candidate), v); err == nil {
                return nil
            }
        }
    }

    return fmt.Errorf("no valid JSON found in agent output (length=%d)", len(output))
}
```

#### Layer 1 + 2：直接 + 围栏

第一层最便宜。第二层 `extractFencedJSON`（`agent_judge.go:318-331`）找 ` ```json ` 和 ` ``` ` 之间的内容。LLM 经常这样包装 JSON 输出，必须支持。

#### Layer 3：`findJSONObjectCandidates` + `findJSONObjectEnd`（大括号配对状态机）

`agent_judge.go:333-351` 扫描所有 `{`，对每个起点调 `findJSONObjectEnd` 找配对的 `}`。

`findJSONObjectEnd`（`agent_judge.go:364-401`）是个**字符级状态机**：

```go
depth := 0
inDoubleQuotedString := false
inSingleQuotedString := false

for i := start; i < len(output); i++ {
    ch := output[i]
    if inDoubleQuotedString || inSingleQuotedString {
        if ch == '\\' { i++; continue }
        if inDoubleQuotedString && ch == '"' { inDoubleQuotedString = false }
        if inSingleQuotedString && ch == '\'' { inSingleQuotedString = false }
        continue
    }
    switch ch {
    case '"': inDoubleQuotedString = true
    case '\'': inSingleQuotedString = true
    case '{': depth++
    case '}':
        depth--
        if depth == 0 { return i, true }
    }
}
```

四个状态维度：双引号字符串、单引号字符串、转义、嵌套深度。注释（`agent_judge.go:353-363`）解释了为什么不用正则：

> nested braces and escaped quotes make regexp matching brittle, and the standard JSON decoder cannot help until we first isolate a candidate object.

**为什么单引号也跳过？** 注释（行 360-363）解释：是为了跳过 Python 风格的"伪 JSON 片段"（单引号字符串），继续找后面真正的 JSON 对象。但单引号内容**不会被当成合法 JSON 接受** —— 最终解码还是用 `encoding/json`，它严格要求双引号。

#### Layer 4：`repairJSONQuotes`（修复未转义引号）

`agent_judge.go:261-316`，处理 LLM 最常犯的错 —— 在 JSON 字符串里写未转义的双引号：

```
"evidence": "output contains "hello" which is wrong"
```

正确应该写 `\"hello\"`，但 LLM 经常忘。这个函数的状态机逻辑：

- 扫描整个字符串。
- 不在字符串里时，正常写字节，遇到 `"` 进入字符串。
- 在字符串里：
  - `\\` → 已转义，原样透传（连下一个字符一起）。
  - `"` → 看**后面**（跳空白后）是不是 JSON 结构字符（`:` / `,` / `}` / `]`）：
    - 是 → 这是字符串结尾，闭合。
    - 否 → 这是字符串内部的未转义引号，加 `\` 转义。

代码（`agent_judge.go:297-309`）：

```go
if ch == '"' {
    rest := strings.TrimLeft(raw[i+1:], " \t\r\n")
    if len(rest) == 0 || rest[0] == ':' || rest[0] == ',' || rest[0] == '}' || rest[0] == ']' {
        // Looks like a structural boundary — close the string.
        buf.WriteByte(ch)
        inString = false
    } else {
        // Inner quote — escape it.
        buf.WriteString(`\"`)
    }
    continue
}
```

这个启发式不可能 100% 正确（理论上 `"a,b":` 这种会被误判），但在真实 LLM 输出上命中率极高，且**只作为最后一层 fallback** —— 前面三层失败才用。

> **Java 类比**：相当于写了一个容错的 `JsonParser`，但目标是"从脏文本里抢救出 JSON"。生产里类似 Jackson 的 `DeserializationFeature.FAIL_ON_TRAILING_TOKENS=false` 但更激进。

### 7.6 `validateAgentJudgeResponse`：防作弊校验

`agent_judge.go:213-225`：

```go
func validateAgentJudgeResponse(criteria []string, results []CriterionResult) error {
    if len(results) != len(criteria) {
        return fmt.Errorf("agent_judge: expected %d criterion results, got %d", len(criteria), len(results))
    }
    for i, cr := range results {
        if strings.TrimSpace(cr.Evidence) == "" {
            return fmt.Errorf("agent_judge: criterion %d %q has empty evidence", i+1, cr.Criterion)
        }
    }
    return nil
}
```

两条防作弊规则，注释把"为什么"说得很清楚（行 214-216）：

> A short response inflates pass_rate because NewResult uses the returned count as the denominator (e.g. 1-of-1 returned = 100% even if 3 were sent).

**Rule 1：数量必须匹配**。LLM 偷懒只返回 1 条 → 1/1 = 100% → 假 PASS。必须强制 `len(results) == len(criteria)`。

**Rule 2：每条必须有 evidence**。LLM 直接 `{"passed": true, "evidence": ""}` 蒙混 → 拒绝。

这两条规则配合 prompt 里的 `You MUST NOT pass a criterion without specific evidence.` 形成两道防线 —— prompt 引导 + 代码强制。

### 7.7 `annotateTimeoutError` + `canRecoverAgentJudgeResult`

`annotateTimeoutError`（`agent_judge.go:410-421`）的作用：当 judge 自己的 timeout 触发时，给 error 加一段注释 `(judge timeout 30s via judge.timeout_seconds)`，方便排查。注意它的三层守卫：
- `err == nil || TimeoutSeconds <= 0` → 直接返回（没设 timeout 就别瞎标注）。
- `!errors.Is(err, context.DeadlineExceeded)` → 不是超时错误别标。
- `judgeCtx.Err() == nil` → **judge ctx 没触发就别标**（可能是上游 HTTP 层的更短 deadline）。

`canRecoverAgentJudgeResult`（`agent_judge.go:423-435`）决定"出错但还能继续"：
- err 为 nil 或 sessionResult 为 nil → 不能恢复。
- final message 为空 → 不能恢复。
- `errors.Is(err, context.Canceled)`（用户主动取消）→ **永不恢复**。
- 其他情况（含 DeadlineExceeded）→ **可以恢复**，只要有 final message。

这是个 trade-off：严格起见应该一遇错就抛，但 LLM API 经常在已生成完整输出后才报"连接重置"，丢弃输出等于浪费一次成功调用。**容错策略让评测更稳定。**

### 7.8 `buildContextMetadata`：报告用元数据

`agent_judge.go:199-211`：把物化结果（profile、目录、manifest）和 prompt 投递方式（`sessionResult.PromptDelivery`）打包成 `ContextMetadata`，塞进 `Result.JudgeContext`。这个字段在 `grading.json` 里以 `judge_context` 形式出现，让报告读者知道"裁判看到了哪些材料、prompt 多大、用什么模式投递的"。

---

## 8. `context_materializer`：评审材料的"投递策略"

### 8.1 为什么要"物化"

`context_materializer.go:85-87` 注释：

> MaterializeJudgeContext writes agent_judge materials to disk and uploads readable copies into the runtime workspace.

LLM 裁判需要看到 final_message / transcript / diff / 生成的文件才能评分。但这些产物有四个特点：
1. **体积大**：transcript 可能几十 KB 到几 MB。
2. **来源分散**：在 host 内存里、在工作区里、在 artifact dir 里。
3. **要送到 runtime 里**：裁判 agent 在 runtime（可能 docker）里跑，得让 agent 能读到。
4. **prompt 体积有限**：全 inline 进 prompt 会爆 token。

物化的本质就是：**把这些产物按"投递策略"写成文件 + 上传到 runtime + （可选）inline 进 prompt**。

### 8.2 `MaterializeJudgeContext` 主流程

`context_materializer.go:87-149`：

1. **解析配置**（行 88）：`resolveJudgeContext(cfg)` → `effectiveJudgeContext`，把 profile + 用户覆写合并成最终模式。
2. **准备目录**（行 89-98）：
   - host dir：`<artifactDir 父目录>/context/` 或临时目录。
   - **先 `os.RemoveAll` 再 `MkdirAll`**：保证目录干净（重跑场景必备）。
3. **runtime dir**（行 100-103）：`.skill-up/judge/context`，挂在 runtime workspace 下。
4. **物化 final_message**（行 113-115）。
5. **物化 transcript**（行 116-122）：先 `marshalTranscript`（带 max turns 截断）再写。
6. **物化 workspace_diff**（行 123-126）：先 `limitLines`（按行截断）。
7. **物化 generated_files**（行 127-129）：按列表写。
8. **物化 attachments**（行 130-132）：用户自定义附件。
9. **写 manifest.json**（行 134-145）：记录每个材料的 mode / bytes / truncated 等元信息，上传到 runtime。

**核心是 `materializeText` 函数**（`context_materializer.go:206-253`）。

### 8.3 profile × mode 双维度

`resolveJudgeContext`（`context_materializer.go:151-197`）实现了"双维度配置"：

#### 维度一：profile（预设套餐）

两种 profile：

```go
switch profile {
case judgeContextProfileMinimal:    // "minimal"
    effective.finalMessageMode = judgeContextModeTruncate
    effective.transcriptMode = judgeContextModeOmit
    effective.workspaceDiffMode = judgeContextModeOmit
    effective.generatedFilesMode = judgeContextGeneratedOmit
default:                             // "standard"（默认）
    effective.finalMessageMode = judgeContextModeInclude
    effective.transcriptMode = judgeContextModeFileRef
    effective.workspaceDiffMode = judgeContextModeFileRef
}
```

- **minimal**：只给 final_message 截断版，其他全关。用于"省钱模式"（裁判只看输出文本）。
- **standard**：final_message 全文 inline，transcript / diff 用 file_ref（写在文件里、prompt 里只放路径）。

#### 维度二：mode（每材料的精细控制）

五种 mode（`context_materializer.go:24-32`）：

| mode | 含义 |
|---|---|
| `include` | 文件 + inline 进 prompt |
| `omit` | 不投递 |
| `truncate` | 截断到 max_bytes 后 inline |
| `file_ref` | 文件上传到 runtime，prompt 里只放路径 |
| `index` | 仅 generated_files 用，只列文件名 |

case 级配置（`cfg.FinalMessage` 等）非 `""` 就覆盖 profile 设的默认。

### 8.4 自动降级：`include` 超 max_bytes 退化为 `file_ref`

`materializeText`（`context_materializer.go:229-232`）：

```go
if mode == judgeContextModeInclude && len([]byte(content)) > maxBytes {
    mode = judgeContextModeFileRef
    material.Mode = mode
}
```

**默认 `maxBytes = 64KB`**（`context_materializer.go:33`）。一旦 inline 内容超 64KB，自动改成 file_ref —— **避免 prompt 撑爆 token 上限**。

这是个非常工程化的设计：用户写 `mode: include` 表示"我希望 inline"，但系统在边界情况下自动降级，不报错、不打断评测。

### 8.5 mode 各分支的实现

`context_materializer.go:239-252`：

```go
switch mode {
case judgeContextModeInclude:
    material.InlineContent = content
case judgeContextModeTruncate:
    material.InlineContent = truncateBytes(content, maxBytes)
    material.Bytes = len([]byte(material.InlineContent))
    material.Truncated = material.Bytes < material.OriginalBytes
case judgeContextModeFileRef:
default:
    return fmt.Errorf("unsupported judge context mode %q for %s", mode, key)
}
```

- `include`：`InlineContent = content`（全部 inline）。
- `truncate`：调 `truncateBytes`（行 481-487，直接按字节切）。
- `file_ref`：什么都不写进 `InlineContent`（只放在文件里）。

注意 `materializeText` 行 218-221 的边界处理：**空内容 + 非 omit 模式 → 自动改成 omit**。避免 prompt 里出现"### Inline Material: transcript\n```\n\n```\n"这种空段落。

### 8.6 manifest.json：材料清单

最后会写一份 `manifest.json` 到 host 和 runtime（行 134-145），结构（`context_materializer.go:47-63`）：

```go
type ContextManifest struct {
    Profile         string                    `json:"profile"`
    MaterializedDir string                    `json:"materialized_dir,omitempty"`
    RuntimeDir      string                    `json:"runtime_dir,omitempty"`
    Materials       []ContextMaterialManifest `json:"materials"`
}

type ContextMaterialManifest struct {
    Key           string `json:"key"`
    Label         string `json:"label,omitempty"`
    Mode          string `json:"mode"`
    Path          string `json:"path,omitempty"`
    Bytes         int    `json:"bytes"`
    OriginalBytes int    `json:"original_bytes,omitempty"`
    Truncated     bool   `json:"truncated,omitempty"`
}
```

这份 manifest 同时被 `Result.JudgeContext.Manifest` 引用（`agent_judge.go:201-205`），所以报告里能看到"裁判看到了什么、用什么模式、是否被截断" —— 完全透明可追溯。

### 8.7 attachment 的路径安全

`materializeAttachments`（`context_materializer.go:386-419`）+ `resolveAttachmentPath`（行 421-442）：

```go
func resolveAttachmentPath(raw string, in Input) (string, error) {
    path := raw
    if !filepath.IsAbs(path) {
        path = filepath.Join(in.SkillDir, path)
    }
    abs, err := filepath.Abs(path)
    ...
    for _, root := range []string{in.SkillDir, in.WorkspacePath} {
        if root == "" { continue }
        if isWithin(abs, root) {
            return abs, nil
        }
    }
    if in.SkillDir == "" && in.WorkspacePath == "" {
        return abs, nil
    }
    return "", fmt.Errorf("judge context attachment %q must stay within skill_dir or workspace", raw)
}
```

**强制附件路径必须在 `SkillDir` 或 `WorkspacePath` 之内**（用 `isWithin`，行 444-454，基于 `filepath.Rel` 判断）。这又是路径遍历防御 —— 用户配置里写 `attachment: [{path: "/etc/passwd"}]` 会被拒绝。

文件名也经过 `safeMaterialName`（行 489-505）—— 把 label 里的非字母数字转成 `-`，避免特殊字符（如 `..`、`/`）逃逸。最终附件被命名成 `01-<safe-label>.<ext>` 放进 `attachments/` 子目录。

写入 host 的路径也再过一道 `safeMaterialHostPath`（行 344-352）+ `safeMaterialRelPath`（行 333-342），双层防御。

### 8.8 `copyMaterialFile`：流式拷贝 + 三重 close

`context_materializer.go:278-323` 是个值得学习的文件拷贝函数：

- 用 `io.Copy` 流式拷贝（不一次性读进内存，大文件友好）。
- **三重 close 错误收集**：`dst.Close()` / `src.Close()` / copy 三个错误分别报告，不丢任何一个。
- 大小检查：`copied > int64(^uint(0)>>1)`（行 315-317）—— 防止 32 位平台溢出。

> **Java 类比**：相当于 try-with-resources 里手动收 `Throwable.addSuppressed()`。Go 没有 ARM，得手写 defer + 错误收集。

---

## 9. 关键设计点（5 条 + Java 类比）

### 9.1 策略模式 + 工厂：一个接口三种实现

`Judge` 接口（`judge.go:31-33`）+ `NewJudge` 工厂（`factory.go:22-50`）是教科书级策略模式：

```go
type Judge interface {
    Evaluate(ctx context.Context, in Input) (*Result, error)
}
```

- **RuleBasedJudge**：纯内存计算，O(规则数 × 文本长度)。
- **ScriptJudge**：子进程隔离，跨语言扩展。
- **AgentJudge**：LLM 当裁判，模糊评测。

**Java 类比**：`Strategy` 模式 + `StrategyFactory`，相当于 Spring 里 `List<ScoringStrategy>` + `@Qualifier` 选择实现。

### 9.2 LLM 输出的鲁棒解析（四层兜底）

`extractJSON`（`agent_judge.go:229-259`）的四层 fallback 是工业级 LLM 应用必备：

1. 直接解（最快）。
2. 围栏抽取（最常见格式）。
3. 状态机大括号配对（处理前后多余文本）。
4. 引号修复（处理 LLM 最常犯的错）。

**核心思想**：LLM 输出永远会有 5%-20% 的格式问题，要么惩罚用户重跑（贵），要么代码兜底（便宜）。skill-up 选了后者，并把兜底逻辑写得**完全无副作用**（纯字符串处理 + 最终用 encoding/json 校验合法性）。

**Java 类比**：Jackson 的 `lenient` 模式，但更激进。

### 9.3 prompt 体积控制（maxBytes 自动降级）

`materializeText` 的 `include → file_ref` 降级（`context_materializer.go:229-232`）解决了一个 LLM 应用通病：

- 用户配 `mode: include` 期望 inline。
- 但 transcript 10MB，inline 进 prompt 直接超 token。
- 系统自动改 file_ref：写文件、上传、prompt 里只放路径。

**结合 LLM 的工具调用能力**：裁判 agent 看到路径后会主动调 `read_file` 工具按需读取 —— 既能看到完整内容，又不撑爆 prompt。这是"AI Agent 系统设计"的经典模式：**让模型自己决定何时拉取上下文**。

**Java 类比**：相当于把一个 List 全量序列化改成懒加载 + 分页，避免 OOM。

### 9.4 防作弊（数量 + evidence 双校验）

`validateAgentJudgeResponse`（`agent_judge.go:213-225`）的两条规则解决了 LLM 评测的两个核心作弊模式：

1. **少返回**：LLM 偷懒只回 1 条 → `len(results) != len(criteria)` 拒绝。
2. **空 evidence**：LLM 全标 passed 但不举证 → `evidence` 为空拒绝。

配合 prompt 里的硬约束（`You MUST NOT pass a criterion without specific evidence.`、强制 JSON 转义说明），形成"prompt 引导 + 代码强制"的双层防线。

**这种"不信 LLM"的设计思维在 Java 后端工程师做 AI 应用时特别要学习** —— 不能把 LLM 输出当可信数据源，必须当不可信输入处理。

### 9.5 短路省钱（expect + failure 优先）

两层短路：

- **expect 预检查**（`expect.go`）：跑 Judge 前先用零成本规则挡掉明显失败。
- **rule_based failure 优先**（`rule_based.go:36-53`）：先扫 failure 规则，命中即返回，不跑 success。

**为什么省钱这么重要？** agent_judge 一次调用几分钱，跑 1000 个 case 就是几十块。expect 把"明显错误的 case"提前拦下，能省掉大量 LLM 调用。这是个**ROI 极高的工程优化**。

**Java 类比**：相当于 Servlet Filter 链式短路、Spring Security 的 `AccessDecisionManager` 早退。

---

## 10. 面试 Q&A

### Q1：为什么要区分 `FAIL` 和 `ERROR`？设计上怎么体现？

**答**：FAIL 是"被评测对象没做好"（业务失败），ERROR 是"评测系统自己炸了"（系统失败）。两者混在一起会让"系统不稳定"伪装成"agent 能力差"，污染数据。

设计上：
- `Status` 枚举（`judge.go:19-28`）显式区分 `FAIL` / `ERROR` / `SKIP` / `PASS`。
- `Result.SkipReason` / `ErrorReason` 是分开的字段。
- `SessionResultError`（`judge.go:154-182`）让 ERROR 场景下仍能抢救 session 产物，便于排查"为什么 ERROR"。
- agent_judge 的 `canRecoverAgentJudgeResult`（`agent_judge.go:423-435`）在容错和严格之间权衡 —— 用户取消（Canceled）绝不恢复，超时（DeadlineExceeded）可以恢复。

### Q2：agent_judge 怎么防止 LLM 偷懒作弊？

**答**：三道防线：

1. **prompt 引导**（`buildJudgePrompt`）：明确要求"MUST NOT pass without evidence"、JSON 格式约束。
2. **数量校验**（`validateAgentJudgeResponse`）：返回的 results 数必须等于 criteria 数。少返回会让 `pass_rate` 分母变小、虚高，必须拒绝。
3. **evidence 非空校验**：每条必须有非空 evidence，避免 LLM 全标 passed 蒙混。

另外 prompt 模板里**预填 criterion 字段**（`appendRequiredResponseFormat`），强制 LLM 用我们提供的原文，避免它改写后绕开数量匹配。

### Q3：rule_based 的"failure 语义反转"是怎么实现的？为什么要这样设计？

**答**：底层 `evalOutputContains` 等 `evalXxx` 函数是"语义纯粹"的 —— `Passed=true` 表示"条件成立"。它们不知道自己被用于 success 还是 failure。

`RuleBasedJudge.Evaluate`（`rule_based.go:36-48`）在调用时做反转：

```go
for _, rule := range j.Failure {
    ar := evaluateAssertion(rule, in)
    if ar.Passed {  // 条件成立 = 坏模式命中 = FAIL
        allAssertions = append(allAssertions, AssertionResult{
            Passed: false,
            ...
        })
    }
}
```

**为什么要这样？** 复用底层算子。`evalOutputContains` 写一份，success 和 failure 都能用，避免代码重复。这是策略模式"组合优于继承"的体现。

### Q4：跨平台脚本执行最难的部分是什么？

**答**：**Windows 上 shebang 不被内核识别**，必须 Go 代码自己派发解释器。难点有三：

1. **shebang 解析**：要支持 `#!/bin/bash -eu`、`#!/usr/bin/env python3`、`#!/usr/bin/env -S pwsh -NoLogo` 三种形态。最后一种的 `-S` 是个迷你 shell tokenizer（`tokenizeShebangSplitString`），要处理单引号、双引号、反斜杠转义、`\c` 截断、`\_` 空格转义。
2. **flag 透传**：shebang 里的 `-NoLogo` / `-eu` 必须带到最终命令里（`interpreter.go:110-118`），不能丢。但又要避免把不相关 shebang 的 flag 串到错解释器（如 `.ps1` 文件带 `#!/bin/bash -eu` 不应该把 `-eu` 传给 powershell，行 113-114 守卫）。
3. **路径格式**：Windows bash（Git Bash）需要正斜杠路径才能让 POSIX 工具用，所以 `envPath: filepath.ToSlash`（`interpreter.go:177`）专门为 `.sh` 准备。

### Q5：为什么 `Input` 要做成"统一契约"？多加字段不就解决了？

**答**：契约统一 = 解耦。执行层（evaluator）只关心"把 Input 装满"，评测层（judge）只关心"从 Input 取数据"。任何字段变更都有清晰的边界：

- 加字段：所有策略都能用，向后兼容。
- 改字段语义：只改 evaluator 装配逻辑，不动 Judge 实现。

如果允许策略"额外要字段"，会出现"只有 agent_judge 需要额外 Context 对象"这种特例，工厂就要变复杂（要给不同策略传不同参数）。当前设计里 `NewJudge`（`factory.go:22`）三个参数 `(cfg, ag, rt)` 对所有策略统一 —— 简单清晰。

**Java 类比**：相当于 `Controller` → `Service` 之间用一个 DTO 传所有数据，而不是 N 个分散参数。好处是签名稳定，扩展靠加字段。

---

## 11. 一句话总结

> skill-up 的 Judge 子系统用**一个接口**（`Judge.Evaluate(Input) (*Result, error)`）+ **三种实现**（rule_based 内存算 / script 子进程跑 / agent_judge LLM 裁判），配合 **expect 预检查短路省钱**、**safePath 路径安全**、**MergeJudgeConfig 分级覆盖**、**extractJSON 四层兜底**、**profile×mode 双维物化** 这五大工程手段，把"模糊的 AI 评测"工程化成了"可复现、可调试、可省钱"的流水线 —— 这套设计可以直接搬到任何"AI 输出质量评测"系统里复用。

---

## 附：文件清单速查

| 文件 | 行数 | 主要内容 |
|---|---|---|
| `internal/judge/judge.go` | 313 | 接口、Input/Result 契约、构造器、safePath |
| `internal/judge/factory.go` | 159 | NewJudge 工厂、MergeJudgeConfig |
| `internal/judge/expect.go` | 301 | expect 预检查、ToAssertionResults、diffHint |
| `internal/judge/rule_based.go` | 492 | RuleBasedJudge、9 种断言、lookupTurn、argsMatch |
| `internal/judge/script.go` | 162 | ScriptJudge、退出码契约、buildResult |
| `internal/judge/interpreter.go` | 556 | planScript 跨平台派发、shebang/env -S tokenizer |
| `internal/judge/agent_judge.go` | 559 | AgentJudge、buildJudgePrompt、extractJSON 四层 |
| `internal/judge/context_materializer.go` | 506 | MaterializeJudgeContext、profile×mode、自动降级 |
| `internal/judge/judge_skill.go` | 42 | SkillInfo / SkillInfosFromRefs |
