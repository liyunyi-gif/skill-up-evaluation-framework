# 07 - Report 生成与 Anthropic 兼容（Reporter 策略 + JSON/JUnit/HTML + Benchmark A/B + IterationWorkspace）

> 面向 Java 后端/全栈工程师的 skill-up 源码逐行深度讲解系列。
> 本文聚焦评测流水线的「最后一公里」——报告与产物生成：策略模式多 Reporter、Anthropic 兼容 JSON schema、A/B Benchmark 量化、Workspace 目录治理。

---

## 0. 一句话定位

> **Report 子系统是 skill-up 评测流水线的「末端渲染层」**：它把 `evaluator.EvalResult` 这堆"事实数据"按三种视图（JSON 事实源 / JUnit CI 拍板 / HTML 给人看）渲染出来，同时落盘成 Anthropic skill-creator 兼容的 `grading.json` / `eval_metadata.json` / `benchmark.json`，让一个本地 Go 评测工具能直接吃进 Anthropic 官方 eval-viewer 生态。

在整条流水线中的位置（来自 `internal/runner/runner.go:1-13` 的包注释）：

```
Load Config → Validate → For each case:
  → Prepare Runtime
  → Install Skills
  → Provision MCP
  → Execute Engine
  → Judge Output
  → Collect Result
→ Generate Report   ← 你在这里
```

报告层不是"打印日志"，而是 **Skill 有效性量化的最终兑现**——`pass_rate +Δ%`、token 是否暴涨、延迟是否拖垮——这些数字都在这一层算出来。

---

## 1. Reporter 策略接口：三视图分工

### 1.1 接口定义（`internal/report/reporter.go:12-14`）

```go
// Reporter writes evaluation output to a chosen format.
type Reporter interface {
    Write(ctx context.Context, in Input) error
}
```

只有 1 个方法、1 个入参 `Input`、1 个 error 返回。**Java 类比**：经典的策略模式 `Strategy` 接口，配合 `Context` 传递上下文。这里 `Input` 就是上下文对象。

> Java 工程师可类比为：
> ```java
> interface Reporter { void write(ReportInput in) throws IOException; }
> ```
> 只是 Go 把"异常"折叠成 `error` 返回值，并且不用 `throws` 声明。

### 1.2 输入对象 `Input`（`reporter.go:17-27`）

```go
type Input struct {
    SkillName     string           `json:"skill_name"`
    SchemaVersion string           `json:"schema_version"`
    EngineName    string           `json:"engine_name"`
    ModelName     string           `json:"model_name"`
    StartTime     time.Time        `json:"start_time"`
    EndTime       time.Time        `json:"end_time"`
    CaseResults   []CaseResult     `json:"case_results"`
    TotalTokens   int              `json:"total_tokens"`
    Benchmark     *BenchmarkResult `json:"benchmark,omitempty"`
}
```

注意 3 个细节：

- `Benchmark` 字段是指针 `*BenchmarkResult` + `omitempty`：当没开启 A/B 实验时，JSON 里**根本不出现**这个 key，而不是 `null`。这是 Go 处理"可选对象"的惯用法（Java 里类似 `Optional<BenchmarkResult>` 但语义更轻）。
- 字段同时挂了 `json` tag —— `Input` 本身也是 `JSONReporter` 直接 `json.MarshalIndent` 的产物（`json.go:20`），这意味着 **`Input` 既是 Reporter 的输入，也是 JSON 的输出 schema**。这是一处很巧的"零冗余"。
- 两个派生方法 `TotalDuration()`（`reporter.go:31-40`）和 `OverallPassRate()`（`reporter.go:43-54`）是"按需计算"的——优先用 `StartTime/EndTime`，否则 fallback 成各 case duration 之和。fallback 的存在是为了容错（防止上游忘填起止时间）。

### 1.3 三 Reporter 分工

| Reporter | 文件 | 受众 | 角色 |
|---|---|---|---|
| `JSONReporter` | `json.go:12-32` | 程序/下游脚本 | **事实源（fact source）**，所有视图的母版 |
| `JUnitReporter` | `junit.go:18-40` | CI / Jenkins / GitHub Actions | 通过 `<failure>` 拍板 CI 红/绿 |
| `HTMLReporter` | `html.go:19-46` | 工程师肉眼 | 单页交互式报告（侧边栏 case 导航 + 折叠 grading） |

注释 `json.go:11` 直接挑明：

```go
// JSONReporter writes machine-readable JSON results.
// JSON is the "fact source" — JUnit and HTML are derived views.
```

**Java 类比**：MVC 里 Model=JSON，View=JUnit/HTML。JUnit 和 HTML 都从同一份 `Input` 派生，谁也不存"私房数据"，避免了多源真相（multi-source-of-truth）。

### 1.4 管道友好的 `writeToOutput`（`helpers.go:28-44`）

这是整个 report 包的「写入门面」，三个 Reporter 都从它走：

```go
func writeToOutput(path, desc string, fn func(w io.Writer) error) (err error) {
    if path == "" {
        return fn(os.Stdout)          // 空路径 → stdout（管道友好）
    }
    f, ferr := os.Create(path)
    if ferr != nil {
        return fmt.Errorf("create %s file: %w", desc, ferr)
    }
    defer func() {
        if cerr := f.Close(); cerr != nil && err == nil {
            err = fmt.Errorf("close %s file: %w", desc, cerr)
        }
    }()
    return fn(f)
}
```

逐行拆解：

1. **空路径写 stdout**（`path == ""`）：这是**Unix 管道哲学**的体现——当用户没指定 `-o report.json` 时，结果直接打到 stdout，可以被 `| jq` 或重定向消费。Java 里类似 `System.out` 兜底，但 Go 这里做得更优雅：调用方传一个 `func(w io.Writer)`，对文件和 stdout 完全无感。
2. **命名的返回值 `err`**：注意签名是 `(err error)`，这是为了配合 `defer` 里"覆盖返回值"——如果 `fn` 已经返回了非空 error，`Close` 错误就**不要覆盖**（`err == nil` 时才赋值），这避免了"关闭失败把真正的业务错误吞掉"。
3. **`defer` 关闭文件**：Go 处理资源释放的标准范式。Java 等价于 `try-with-resources`，但 Go 没有 ARM 块，只能 `defer`。
4. **`%w` 包装错误**：错误信息层层包装但保留原始 chain，可以用 `errors.Unwrap` 还原。Java 没原生等价物，类似 `new IOException("...", cause)`。

**为什么叫"管道友好"？** 因为：

- 三个 Reporter 共享同一个出口语义（"有路径写文件，没路径写 stdout"）；
- 调用方（`runner.writeFormattedReports` `runner.go:323-344`）只管把 `iterDir/report.json` 这种路径塞进去，毫不用关心 stdout fallback；
- CLI 测试时不用真去写文件，传空 `OutputPath` 就能在 stdout 上看到完整 JSON。

---

## 2. JUnit XML 映射细节（CI 拍板）

JUnit XML 是 CI 系统的"通用测试语言"。skill-up 把每个 case 映射成一个 `<testcase>`，并把 judge（断言）结果编码进 `<failure>` / `<error>` / `<skipped>` 三态。

### 2.1 三态映射（`junit.go:117-154`）

核心是这段 switch：

```go
switch cr.Status {
case judge.StatusFail:
    failures++
    tc.Failure = &junitFailure{
        Message: fmt.Sprintf("case %s failed", cr.CaseID),
        Type:    "AssertionFailure",
        Body:    buildFailureBody(cr),
    }
case judge.StatusError:
    errors++
    errMsg := cr.Error
    if errMsg == "" && cr.Grading != nil && cr.Grading.ErrorReason != nil {
        errMsg = *cr.Grading.ErrorReason
    }
    tc.Error = &junitError{
        Message: errMsg,
        Type:    "ExecutionError",
        Body:    errMsg,
    }
case judge.StatusSkip:
    skipped++
    msg := ""
    if cr.Grading != nil && cr.Grading.SkipReason != nil {
        msg = *cr.Grading.SkipReason
    }
    tc.Skipped = &junitSkipped{Message: msg}
}
```

四态状态机（`internal/judge/judge.go:21-27`）：`PASS / FAIL / SKIP / ERROR`。其中：

- `PASS`：什么都不挂，testcase 干净（CI 视为通过）。
- `FAIL`（**断言失败**）：业务上的"答错了"，挂 `<failure type="AssertionFailure">` —— CI 视为**红灯**但构建仍成功。
- `ERROR`（**执行异常**）：runtime 挂了，比如 agent 进程崩了。挂 `<error type="ExecutionError">` —— CI 视为**构建失败**，比 failure 更严重。
- `SKIP`：主动跳过（比如配置了 `skip`）。挂 `<skipped>` —— CI 视为**忽略**，不亮红。

> **JUnit 协议本身**就区分 `failures`（断言失败）和 `errors`（意外异常）。这是测试框架的共识——JUnit Java 原生也是 `AssertionFailedError` vs `RuntimeException` 的二元区分。skill-up 完整地还原了这套语义，没有任何信息损失。

### 2.2 数据结构对应 XML（`junit.go:46-103`）

每个 Go struct 字段都带 `xml:` tag，对应 XML 元素/属性：

```go
type junitTestSuite struct {
    XMLName   xml.Name        `xml:"testsuite"`
    Name      string          `xml:"name,attr"`
    Tests     int             `xml:"tests,attr"`
    Failures  int             `xml:"failures,attr"`
    Errors    int             `xml:"errors,attr"`
    Skipped   int             `xml:"skipped,attr"`
    Time      string          `xml:"time,attr"`
    Timestamp string          `xml:"timestamp,attr"`
    TestCases []junitTestCase `xml:"testcase"`
}

type junitTestCase struct {
    XMLName    xml.Name         `xml:"testcase"`
    Name       string           `xml:"name,attr"`
    ClassName  string           `xml:"classname,attr"`
    Time       string           `xml:"time,attr"`
    Properties *junitProperties `xml:"properties,omitempty"`
    Failure    *junitFailure    `xml:"failure,omitempty"`
    Error      *junitError      `xml:"error,omitempty"`
    Skipped    *junitSkipped    `xml:"skipped,omitempty"`
}
```

注意 3 个 Go XML 编码的特殊点：

- `XMLName xml.Name xml:"testsuite"`：声明根元素名。
- `,attr`：表示作为**属性**而非子元素。
- `,omitempty`：指针为 `nil` 时该元素**完全不出现**。比如 `Properties *junitProperties ... omitempty`，没有 properties 时整个 `<properties>` 块都省略。

> **Java 类比**：JAXB 的 `@XmlAttribute` / `@XmlElement`。Go 的 `xml` tag 是它的"低配 JAXB"——无需注解类、纯靠反射 + struct tag 驱动。

### 2.3 `properties` 携带 judge.skills（`junit.go:172-186`）

这是把"用到了哪些 Skill 文件"这种**结构化元数据**塞进 JUnit 的手段：

```go
func buildJudgeSkillProperties(cr CaseResult) *junitProperties {
    props := []junitProperty{
        {Name: "judge.skills.count", Value: strconv.Itoa(len(cr.JudgeSkills))},
    }
    for i, skill := range cr.JudgeSkills {
        prefix := fmt.Sprintf("judge.skills.%d.", i)
        props = append(props,
            junitProperty{Name: prefix + "source", Value: skill.Source},
            junitProperty{Name: prefix + "path",   Value: skill.Path},
            junitProperty{Name: prefix + "target", Value: skill.Target},
            junitProperty{Name: prefix + "name",   Value: skill.Name},
        )
    }
    return &junitProperties{Properties: props}
}
```

输出 XML 长这样：

```xml
<properties>
  <property name="judge.skills.count" value="2"/>
  <property name="judge.skills.0.source" value="local"/>
  <property name="judge.skills.0.path" value="/abs/skill.md"/>
  <property name="judge.skills.0.target" value="agent"/>
  <property name="judge.skills.0.name" value="code-review"/>
  ...
</properties>
```

**为什么用扁平的 `judge.skills.0.name` 而不是嵌套 `<skill>` 元素？** 因为 JUnit 的 `<properties>` schema 只允许扁平 key-value——这是协议约束。skill-up 用"点号路径 + 数字下标"模拟出数组结构，CI 工具（如 Jenkins）解析 properties 时可以直接拿到这些元数据，用于构建失败原因 dashboard。

### 2.4 `buildFailureBody` 把失败证据写进 XML（`junit.go:191-210`）

```go
func buildFailureBody(cr CaseResult) string {
    if cr.Grading == nil {
        return ""
    }
    var lines []string
    for _, ar := range cr.Grading.AssertionResults {
        if !ar.Passed {
            lines = append(lines, fmt.Sprintf("- %s: %s", ar.Text, ar.Evidence))
        }
    }
    // Append turn summary when multi-turn results are present.
    if len(cr.TurnResults) > 0 {
        for _, tr := range cr.TurnResults {
            if tr.Status != "completed" {
                lines = append(lines, fmt.Sprintf("- turn %d: status=%s reason=%s", tr.TurnNumber, tr.Status, tr.Reason))
            }
        }
    }
    return strings.Join(lines, "\n")
}
```

注意注释 `junit.go:188-190`：

> Turn-scoped assertions already include turn numbers in their text field, making CI failure output directly actionable.

——意思是：**断言文本本身已经带了 "turn 3" 之类的轮次信息**，所以这里直接 `- {Text}: {Evidence}` 拼起来，CI 上点开 `<failure>` 的 body 就能直接定位"哪一轮的哪一条断言挂了"，不需要点开 HTML 报告再翻一次。这是面向工程师的"可操作性"设计——优先服务于 CI 流量入口。

### 2.5 多轮追加（`<testsuites>` 容器）

最外层是 `<testsuites>` 而非 `<testsuite>`：

```go
return junitTestSuites{Suites: []junitTestSuite{suite}}
```

虽然当前只塞了一个 suite，但保留了"未来多 suite（比如多 Skill 一起评测）"的扩展空间。这是协议推荐的容器层级——CI 工具一般都期望根是 `<testsuites>`。

### 2.6 头部 + Indent（`junit.go:28-39`）

```go
return writeToOutput(r.OutputPath, "junit report", func(w io.Writer) error {
    if _, err := w.Write([]byte(xml.Header)); err != nil {
        return fmt.Errorf("write junit xml header: %w", err)
    }
    enc := xml.NewEncoder(w)
    enc.Indent("", "  ")
    if err := enc.Encode(suite); err != nil {
        return fmt.Errorf("encode junit xml: %w", err)
    }
    return nil
})
```

- `xml.Header` 是 `<?xml version="1.0" encoding="UTF-8"?>\n`，必须**单独手动写**——Go 的 `xml.Encoder` 默认**不输出 XML 声明**，这是已知的设计取舍。
- `enc.Indent("", "  ")` 输出带 2 空格缩进的"漂亮版" XML——便于 git diff 和人肉排查。

---

## 3. HTML 工程亮点（自包含 + 前端接管）

### 3.1 `//go:embed` 自包含（`html.go:304-314`）

```go
//go:embed templates/report.html
var htmlTemplate string

//go:embed templates/logo.png
var htmlLogoPNG []byte

// logoDataURI is the project logo encoded as a base64 data URI, embedded into
// the HTML report so the output file is self-contained and viewable offline.
var logoDataURI = template.URL("data:image/png;base64," + base64.StdEncoding.EncodeToString(htmlLogoPNG))
```

3 个关键工程点：

1. **`//go:embed`**：Go 1.16+ 的"编译期资源嵌入"。`templates/report.html` 和 `templates/logo.png` 在编译时被打进二进制，部署时**单文件可执行**——不需要带资源目录。Java 等价物：Maven 的 `src/main/resources` + ClassLoader 加载，但 Go 这里更彻底，没有"资源目录"概念，直接是字节数组。
2. **`var logoDataURI = ...`**：包级全局变量，**进程启动时计算一次**（PNG → base64 → data URI），后续每个 HTML 报告共享这一份。注释 `//nolint:gochecknoglobals,gosec // G203: precomputed once at init from embedded asset` 说明 linter 被显式豁免——理由是"输入是编译期常量，不接受用户控制，没有 XSS 风险"。
3. **`template.URL`**：绕过 Go `html/template` 的自动转义。因为 data URI 是给 `<img src="...">` 用的，自动转义会破坏 URL。

### 3.2 数据塞 `<script>` JSON（`html.go:249-283`）

这是 HTML 报告最巧的设计——**Go 端不做 HTML 渲染，只把 JSON 塞进 `<script>` 标签，前端 JS 接管所有渲染**：

```go
func (r *HTMLReporter) buildTemplateData(in Input) (htmlReportData, error) {
    grouped, orderedIDs := groupCaseResults(in.CaseResults)
    counts := countCaseStatuses(grouped, orderedIDs)
    cases := buildEmbeddedCases(grouped, orderedIDs)

    ed := embeddedReportData{
        SkillName:   in.SkillName,
        EngineName:  in.EngineName,
        ModelName:   in.ModelName,
        StartTime:   in.StartTime.Format(time.RFC3339),
        Duration:    fmt.Sprintf("%.1fs", in.TotalDuration().Seconds()),
        TotalTokens: in.TotalTokens,
        Summary: embeddedSummary{
            Total:    len(cases),
            Passed:   counts.passed,
            Failed:   counts.failed,
            Skipped:  counts.skipped,
            Errors:   counts.errored,
            PassRate: fmt.Sprintf("%.0f%%", in.OverallPassRate()*100),
        },
        Cases:     cases,
        Benchmark: in.Benchmark,
    }

    jsonBytes, err := json.Marshal(ed)
    if err != nil {
        return htmlReportData{}, fmt.Errorf("marshal embedded report data: %w", err)
    }

    return htmlReportData{
        SkillName:        in.SkillName,
        LogoDataURI:      logoDataURI,
        EmbeddedDataJSON: template.JS(jsonBytes), //nolint:gosec // trusted internal data, not user input
    }, nil
}
```

- `embeddedReportData`（`html.go:60-70`）是给前端用的"瘦过身"的数据模型——只挑前端要用的字段（隐藏掉 `judge.Result` 内部细节），是一种**显式的契约边界**。
- `template.JS(jsonBytes)`：把 JSON 字节流标记为"安全的 JS 字面量"，Go 模板会原样输出到 `<script>` 中，形如 `const DATA = {...};`。这种"JSON in script"模式避免了在 HTML 里散落渲染数据，前端只需读 `DATA` 全局变量。

> **Java 类比**：Spring MVC 的 `@RestController` 返回 JSON + 前端 React/Vue 自己渲染。Go 这里类似——template 只搭骨架（CSS、布局、logo），数据是 JS 端读取的。这种"服务端模板 + 客户端渲染"的混合模式（**Isomorphic-lite**）让单一 HTML 文件既有交互又自包含。

### 3.3 `with/without` 配对（`html.go:185-247`）

A/B 实验模式下，同一个 case 有两个 `CaseResult`（一个 `with_skill`，一个 `without_skill`）。HTML 需要把它们**按 case ID 配对**展示：

```go
func groupCaseResults(results []CaseResult) (map[string]*caseGroup, orderedIDs []string) {
    grouped := make(map[string]*caseGroup)
    orderedIDs := make([]string, 0, len(results))

    for i := range results {
        cr := &results[i]
        g, ok := grouped[cr.CaseID]
        if !ok {
            g = &caseGroup{}
            grouped[cr.CaseID] = g
            orderedIDs = append(orderedIDs, cr.CaseID)
        }
        if cr.Configuration == "without_skill" {
            g.withoutSkill = cr
        } else {
            g.withSkill = cr
        }
    }
    return grouped, orderedIDs
}
```

注意 3 个细节：

1. **取指针 `&results[i]` 而非 `results[i]`**：Go 的 `for i := range` 取地址安全（`for _, v := range` 取 `&v` 会得到最后一个 v 的地址，是经典坑）。这里写法正确。
2. **`orderedIDs` 单独维护顺序**：Go 的 `map` 迭代顺序是**随机**的。如果直接遍历 `grouped`，每次 HTML 报告的 case 顺序都不同——很糟糕的体验。`orderedIDs` 记录"首次见到"的顺序，保证可重复。
3. **`Configuration == "without_skill"` 判定**：把字符串硬编码 `"without_skill"` 当作 baseline 标识。`primaryCaseResult`（`html.go:207-212`）优先返回 `withSkill`——意思是**主视图永远展示"开 Skill"的版本**，"未开 Skill"作为 baseline 挂在 `Baseline` 字段下供对比展示。

### 3.4 logo data URI（`html.go:307-314`）

`logoDataURI` 是 `template.URL` 类型（`htmlReportData.LogoDataURI`），模板里 `<img src="{{.LogoDataURI}}">`。最终 HTML 文件里这一段长这样：

```html
<img src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA..." />
```

**单文件可分享**：把 `report.html` 邮件发送给同事，他在断网环境打开都能看到 logo。这就是注释 `viewable offline` 的含义。

### 3.5 模板函数（`html.go:32-34` + `template_helpers.go:12-53`）

```go
funcMap := SharedTemplateFuncs()
funcMap["statusIcon"] = statusIcon
tmpl, err := template.New("report").Funcs(funcMap).Parse(htmlTemplate)
```

`SharedTemplateFuncs`（`template_helpers.go:12-53`）提供 7 个共享模板函数：`fmtDuration / fmtPercent / fmtPercentSigned / passFailClass / passFailIcon / notNil / derefFloat`。这套函数在 `report.html` 和 `review.html`（另一个模板）间复用，是 DRY 的体现。

`statusIcon`（`html.go:285-298`）把状态映射为 HTML 实体：

```go
case judge.StatusPass:  return template.HTML("&#x2705;") // ✅
case judge.StatusFail:  return template.HTML("&#x274C;") // ❌
case judge.StatusSkip:  return template.HTML("&#x23ED;") // ⏭
case judge.StatusError: return template.HTML("&#x26A0;") // ⚠
```

`template.HTML` 同样是绕过转义——因为这些是受信任的 HTML 实体常量。

---

## 4. Anthropic 兼容格式（生态借力）

skill-up 主动兼容 Anthropic skill-creator 的产物 schema，让本地评测产物可以**直接被 eval-viewer 等官方工具消费**——这是一招"借力生态"。

### 4.1 三种 Anthropic 兼容产物

| 文件 | 位置 | Go 类型 | 写入函数 |
|---|---|---|---|
| `grading.json` | `<case>/<config>/grading.json` | `AnthropicGrading`（`grading.go:26-30`） | `WriteGradingJSON`（`grading.go:86-88`） |
| `eval_metadata.json` | `<case>/eval_metadata.json` | `EvalMetadata`（`grading.go:104-109`） | `WriteEvalMetadata`（`grading.go:113-115`） |
| `benchmark.json` | `iteration-N/benchmark.json` | `AnthropicBenchmark`（`benchmark_anthropic.go:22-27`） | `WriteAnthropicBenchmark`（`benchmark_anthropic.go:218-220`） |

### 4.2 `AnthropicGrading` schema（`grading.go:18-45`）

```go
type AnthropicGrading struct {
    Expectations []AnthropicExpectation `json:"expectations"`
    Summary      AnthropicSummary       `json:"summary"`
    JudgeContext *judge.ContextMetadata `json:"judge_context,omitempty"`
}

type AnthropicExpectation struct {
    Text     string `json:"text"`
    Passed   bool   `json:"passed"`
    Evidence string `json:"evidence"`
}

type AnthropicSummary struct {
    Passed   int     `json:"passed"`
    Failed   int     `json:"failed"`
    Total    int     `json:"total"`
    PassRate float64 `json:"pass_rate"`
}
```

字段命名是关键——"expectations / passed / evidence / pass_rate"这些**精确对齐** Anthropic skill-creator 的输出 schema。文档注释 `grading.go:18-24` 还贴了 demo 例子佐证。

### 4.3 `ConvertToAnthropicGrading` 适配器（`grading.go:47-82`）

```go
func ConvertToAnthropicGrading(result *judge.Result) *AnthropicGrading {
    if result == nil {
        return &AnthropicGrading{
            Expectations: []AnthropicExpectation{},
            Summary:      AnthropicSummary{},
        }
    }

    expectations := make([]AnthropicExpectation, 0, len(result.AssertionResults))
    for _, ar := range result.AssertionResults {
        expectations = append(expectations, AnthropicExpectation{
            Text:     ar.Text,
            Passed:   ar.Passed,
            Evidence: ar.Evidence,
        })
    }

    return &AnthropicGrading{
        Expectations: expectations,
        Summary: AnthropicSummary{
            Passed:   result.Summary.Passed,
            Failed:   result.Summary.Failed,
            Total:    result.Summary.Total,
            PassRate: result.Summary.PassRate,
        },
        JudgeContext: result.JudgeContext,
    }
}
```

这是经典的**适配器模式（Adapter Pattern）**——`judge.Result` 是内部模型，`AnthropicGrading` 是外部 schema，转换函数负责字段级映射。注释 `grading.go:49-54` 用伪图描述映射：

```
judge.AssertionResult.Text     -> AnthropicExpectation.Text
judge.AssertionResult.Passed   -> AnthropicExpectation.Passed
judge.AssertionResult.Evidence -> AnthropicExpectation.Evidence
judge.ResultSummary            -> AnthropicSummary (direct field mapping)
```

3 个细节：

1. **`nil` 安全**（`result == nil`）：上游忘填 grading 时，返回空结构（`Expectations: []`——非 nil 空切片），保证 JSON 输出 `"expectations": []` 而非 `"expectations": null`。Anthropic 工具解析时空数组比 null 更友好。
2. **预分配容量** `make([]..., 0, len(...))`：避免 append 时的多次扩容拷贝。
3. **`JudgeContext` 透传**：这是 skill-up 在 Anthropic schema 上的**扩展字段**——`omitempty` 意味着 Anthropic eval-viewer 会忽略这个字段，但 skill-up 自己的工具能消费。这是"兼容 + 扩展"的平衡。

> **Java 类比**：MapStruct 的 `@Mapper` 自动生成的字段映射代码。Go 没有 MapStruct，但这种 1:1 字段映射的样板代码很短（10 行），手写比引入框架更划算。

### 4.4 `EvalMetadata`（`grading.go:94-115`）

```go
type EvalMetadata struct {
    EvalID     int      `json:"eval_id"`
    EvalName   string   `json:"eval_name"`
    Prompt     string   `json:"prompt"`
    Assertions []string `json:"assertions"`
}
```

这个文件描述"这个 case 测了什么"——id、名字、原始 prompt、断言文本数组。它是 eval-viewer 的"目录索引"，让人能脱离评测工具快速翻阅 case 内容。

### 4.5 为什么"兼容 = 生态借力"

如果 skill-up 自己发明一套报告 schema（比如叫 `skill-up-result.json`），用户拿到产物只能用 skill-up 自己的工具看，**生态被锁死**。而兼容 Anthropic schema 意味着：

- 用户已有 Anthropic eval-viewer / CI 集成 → 直接复用，**迁移成本为零**；
- Anthropic 未来更新 schema → skill-up 跟着升级适配器即可，**不破坏前端工具链**；
- skill-up 在 Anthropic 生态里能直接被讨论、对比、引用。

> 这是开源项目**进入主流生态**的关键策略——**"少发明轮子，多对齐标准"**。Java 类比：Spring Data 对齐 JPA 标准、Spring Boot Starter 对齐 MicroProfile 等。

---

## 5. Benchmark A/B 计算（Skill 有效性量化）

Benchmark 是评测的"**结论性数字**"：开 Skill 比 baseline 到底好多少？

### 5.1 两套 Benchmark：内部 vs Anthropic

skill-up 有两套 Benchmark 数据结构：

- **内部 Benchmark**（`benchmark.go` + `reporter.go:83-115`）：用于 `result.json` 内嵌的 `benchmark` 字段。
- **Anthropic Benchmark**（`benchmark_anthropic.go`）：用于独立 `benchmark.json` 文件，schema 完全对齐 Anthropic。

> 为什么两套？因为 `result.json` 内嵌版本面向程序消费（数字精确），Anthropic 版本面向 eval-viewer（schema 兼容 + min/max + 字符串格式化 delta）。

### 5.2 4 指标 + mean/stddev（`benchmark.go:32-109`）

3 个统计基础函数：

```go
// mean computes the arithmetic mean of a float64 slice.
func mean(values []float64) float64 {
    if len(values) == 0 {
        return 0
    }
    var sum float64
    for _, v := range values {
        sum += v
    }
    return sum / float64(len(values))
}

// stdDev computes the population standard deviation of a float64 slice.
func stdDev(values []float64) float64 {
    n := len(values)
    if n < 2 {
        return 0
    }
    avg := mean(values)
    var sumSq float64
    for _, v := range values {
        d := v - avg
        sumSq += d * d
    }
    return math.Sqrt(sumSq / float64(n))
}

// passRate computes the fraction of true values in a bool slice.
func passRate(passed []bool) float64 { ... }
```

**3 个工程细节**：

1. **总体标准差**（除以 `n`，不是 `n-1`）：注释 `population standard deviation`。这是统计学上的取舍——我们关注的是**这 N 次运行本身**的离散度（描述统计），而不是**估计总体**（推断统计）。Java 类似 Apache Commons Math 的 `StandardDeviation` 默认是总体版本。
2. **`n < 2` 返回 0**：单点没法算方差。
3. **空切片返回 0** 而非 panic：函数式友好的容错。

### 5.3 `ComputeBenchmarkStats` 4 指标（`benchmark.go:79-109`）

```go
func ComputeBenchmarkStats(metrics []CaseMetrics) BenchmarkStats {
    if len(metrics) == 0 {
        return BenchmarkStats{}
    }

    passed := make([]bool, len(metrics))
    times := make([]float64, len(metrics))
    inputTokens := make([]float64, len(metrics))
    outputTokens := make([]float64, len(metrics))

    for i, m := range metrics {
        passed[i] = m.Passed
        times[i] = m.TimeSeconds
        inputTokens[i] = m.InputTokens
        outputTokens[i] = m.OutputTokens
    }

    passFloats := make([]float64, len(passed))
    for i, p := range passed {
        if p {
            passFloats[i] = 1
        }
    }

    return BenchmarkStats{
        PassRate:     StatValue{Mean: passRate(passed), StdDev: stdDev(passFloats)},
        TimeSeconds:  StatValue{Mean: mean(times), StdDev: stdDev(times)},
        InputTokens:  StatValue{Mean: mean(inputTokens), StdDev: stdDev(inputTokens)},
        OutputTokens: StatValue{Mean: mean(outputTokens), StdDev: stdDev(outputTokens)},
    }
}
```

**4 个指标**：`pass_rate / time_seconds / input_tokens / output_tokens`——这 4 个数字回答了 Skill 评测最关心的 4 个问题：

| 指标 | 业务含义 |
|---|---|
| `pass_rate` | **质量**：Skill 有没有提升任务完成率？ |
| `time_seconds` | **延迟代价**：Skill 加进去后变快还是变慢？ |
| `input_tokens` | **prompt 成本**：Skill 注入的上下文贵不贵？ |
| `output_tokens` | **响应膨胀**：Skill 有没有诱导模型啰嗦？ |

**关键巧思**（`benchmark.go:96-101`）：`pass_rate` 的 stddev 用的是"布尔转 0/1 浮点数"再算 stddev——这等价于**伯努利分布的标准差** `sqrt(p*(1-p))`。这种"借用数值统计学公式表达分类变量离散度"的小技巧避免了单独写一份伯努利 stddev 函数。

### 5.4 `ComputeBenchmarkDelta`（`benchmark.go:112-119`）

```go
func ComputeBenchmarkDelta(withSkill, withoutSkill BenchmarkStats) BenchmarkDelta {
    return BenchmarkDelta{
        PassRate:     withSkill.PassRate.Mean - withoutSkill.PassRate.Mean,
        TimeSeconds:  withSkill.TimeSeconds.Mean - withoutSkill.TimeSeconds.Mean,
        InputTokens:  withSkill.InputTokens.Mean - withoutSkill.InputTokens.Mean,
        OutputTokens: withSkill.OutputTokens.Mean - withoutSkill.OutputTokens.Mean,
    }
}
```

delta 是"减法"——`with_skill.mean - without_skill.mean`。注意：**只用 mean，不用 stddev**——delta 本身不带置信区间，这是"描述性差异"，不是"统计显著性检验"。生产级 A/B 一般还要做 t-test，但 skill-up 选择了"够用即可"。

### 5.5 `ComputeBenchmark`：简化模式 vs 完整模式（`benchmark.go:121-144`）

```go
func ComputeBenchmark(withSkillMetrics []CaseMetrics, withoutSkillMetrics []CaseMetrics) *BenchmarkResult {
    withStats := ComputeBenchmarkStats(withSkillMetrics)

    result := &BenchmarkResult{
        RunSummary: BenchmarkRunSummary{
            WithSkill:    withStats,
            WithoutSkill: nil,
            Delta:        nil,
        },
    }

    if withoutSkillMetrics != nil {
        withoutStats := ComputeBenchmarkStats(withoutSkillMetrics)
        delta := ComputeBenchmarkDelta(withStats, withoutStats)
        result.RunSummary.WithoutSkill = &withoutStats
        result.RunSummary.Delta = &delta
    }

    return result
}
```

两种模式：

- **简化模式**（默认）：`withoutSkillMetrics == nil` → `WithoutSkill` 和 `Delta` 都是 `nil`，JSON 输出为 `null`。
- **完整模式**（`benchmark.enabled=true`）：传入 baseline metrics，计算 `withoutStats` 和 `delta`，挂上指针。

> **Java 类比**：策略模式 + 空对象模式。`null` 在 Java 里很危险，但 Go 配合 `omitempty` + 指针 + nil check 是一种受控的"可选性"表达。

### 5.6 `ExtractMetrics` 从 CaseResult 提取指标（`benchmark.go:151-162`）

```go
func ExtractMetrics(results []CaseResult) []CaseMetrics {
    metrics := make([]CaseMetrics, len(results))
    for i, r := range results {
        metrics[i] = CaseMetrics{
            Passed:       r.Status == judge.StatusPass,
            TimeSeconds:  float64(r.DurationMs) / 1000.0,
            InputTokens:  float64(r.InputTokens),
            OutputTokens: float64(r.OutputTokens),
        }
    }
    return metrics
}
```

这是 `CaseResult → CaseMetrics` 的 **ETL 转换层**——把"展示型"数据（status 枚举、DurationMs 毫秒）转成"统计型"数据（bool、float seconds）。是适配器模式的又一处应用。

### 5.7 Anthropic 版：min/max + 字符串 delta（`benchmark_anthropic.go`）

Anthropic 版本比内部版本多了 2 类信息：

```go
type AnthropicStatValue struct {
    Mean   float64 `json:"mean"`
    StdDev float64 `json:"stddev"`
    Min    float64 `json:"min"`
    Max    float64 `json:"max"`
}

type AnthropicDelta struct {
    PassRate    string `json:"pass_rate"`
    TimeSeconds string `json:"time_seconds"`
    Tokens      string `json:"tokens"`
}
```

- `Min/Max`：内部版本只算 mean/stddev，Anthropic 版本加上**极值**——eval-viewer 用 min/max 画箱线图。
- `Delta` 是 **`string`** 而非 `float64`：因为 delta 要带符号显示（`+5.2%` / `-3.1s`），用字符串预格式化避免前端再处理。

#### `sliceMin/sliceMax`（`benchmark_anthropic.go:93-118`）

```go
func sliceMin(values []float64) float64 {
    if len(values) == 0 {
        return 0
    }
    m := values[0]
    for _, v := range values[1:] {
        if v < m {
            m = v
        }
    }
    return m
}
```

手写 min/max 而非用 `slices.Min`（Go 1.21+）—— 大概率是为了**显式控制空切片行为**（Go `slices.Min` 在空切片上 panic，这里返回 0）。

#### `ComputeAnthropicStatValue`（`benchmark_anthropic.go:121-128`）

```go
func ComputeAnthropicStatValue(values []float64) AnthropicStatValue {
    return AnthropicStatValue{
        Mean:   mean(values),
        StdDev: stdDev(values),
        Min:    sliceMin(values),
        Max:    sliceMax(values),
    }
}
```

复用了 `benchmark.go` 里的 `mean/stdDev`，**不重复造轮子**——这是同包内代码复用的自然结果。

#### Delta 字符串格式化（`benchmark_anthropic.go:179-183`）

```go
bm.RunSummary.Delta = &AnthropicDelta{
    PassRate:    fmt.Sprintf("%+.2f", withSummary.PassRate.Mean-withoutSummary.PassRate.Mean),
    TimeSeconds: fmt.Sprintf("%+.1f", withSummary.TimeSeconds.Mean-withoutSummary.TimeSeconds.Mean),
    Tokens:      fmt.Sprintf("%+.1f", withSummary.Tokens.Mean-withoutSummary.Tokens.Mean),
}
```

- `%+` 中的 `+` 表示**强制输出正负号**——`+5.20` 和 `-3.10`，让人一眼看出方向。
- 不同精度（PassRate 用 `.2` 即 2 位小数，Time/Tokens 用 `.1` 即 1 位）—— 根据"业务关心的小数位"调整。

### 5.8 Functional Options：`WithTimestamp`（`benchmark_anthropic.go:137-199`）

```go
func ComputeAnthropicBenchmark(
    skillName, skillPath string,
    withSkillRuns []BenchmarkRun,
    withoutSkillRuns []BenchmarkRun,
    opts ...func(*benchmarkOptions),
) *AnthropicBenchmark {
    o := benchmarkOptions{timestamp: time.Now().UTC()}
    for _, fn := range opts {
        fn(&o)
    }
    timestamp := o.timestamp.Format(time.RFC3339)
    ...
}

type benchmarkOptions struct {
    timestamp time.Time
}

func WithTimestamp(t time.Time) func(*benchmarkOptions) {
    return func(o *benchmarkOptions) {
        o.timestamp = t
    }
}
```

这是 Go 的 **Functional Options 模式**（Dave Cheney 推广）：

- 必填参数（skillName/skillPath/runs）走位置参数；
- 可选参数（timestamp）走 `opts ...func(*T)` 变长参数；
- 调用方：`ComputeAnthropicBenchmark(name, path, withRuns, nil, report.WithTimestamp(someTime))` 或不传最后一个 → 用默认 `time.Now().UTC()`。

> **Java 类比**：Builder 模式。Go 没有 builder，但 Functional Options 达到了相同效果——**可选参数 + 默认值 + 易扩展**。未来加 `WithNote()`、`WithRunsPerConfiguration()` 都不需要改函数签名。
>
> 测试代码尤其喜欢这种模式——可以注入固定 timestamp 让输出可断言。

### 5.9 `extractRunMetric`：函数作为参数（`benchmark_anthropic.go:209-215`）

```go
func extractRunMetric(runs []BenchmarkRun, fn func(BenchmarkRun) float64) []float64 {
    values := make([]float64, len(runs))
    for i, r := range runs {
        values[i] = fn(r)
    }
    return values
}
```

调用方：

```go
computeAnthropicRunSummary := func(runs []BenchmarkRun) AnthropicStatSummary {
    return AnthropicStatSummary{
        PassRate:    ComputeAnthropicStatValue(extractRunMetric(runs, func(run BenchmarkRun) float64 { return run.Result.PassRate })),
        TimeSeconds: ComputeAnthropicStatValue(extractRunMetric(runs, func(run BenchmarkRun) float64 { return run.Result.TimeSeconds })),
        Tokens:      ComputeAnthropicStatValue(extractRunMetric(runs, func(run BenchmarkRun) float64 { return float64(run.Result.Tokens) })),
    }
}
```

**Java 类比**：`stream.map(run -> run.getResult().getPassRate()).collect(toList())`。Go 没有 Stream API，但这种"传一个 getter lambda 给提取器"的写法达到了同样的效果，且零依赖。

---

## 6. IterationWorkspace 目录管理

`IterationWorkspace`（`internal/report/workspace.go:54-86`）是 skill-up 落盘产物的**目录治理者**，它把所有 case 产物组织成 Anthropic 兼容的目录树。

### 6.1 目录结构图（`workspace.go:1-21` 文件头注释）

```
<skill-name>-workspace/
  iteration-<N>/
    benchmark.json
    benchmark.md
    report.html
    result.json
    report.xml           (可选, --format junit)
    report.json          (可选, --format json)
    <case-id>/
      eval_metadata.json
      with_skill/
        outputs/
          response.md
        grading.json
      without_skill/          # 可选, 仅当 benchmark.enabled=true
        outputs/
          response.md
        grading.json
```

设计要点：

- **workspace 与 skill 目录平级**：`<skill>/` 和 `<skill>-workspace/` 是兄弟（`runner.go:122-126` 明确："workspace sits alongside the skill directory"）——这是 Anthropic eval layout 的约定，便于将整个 workspace 一起 zip 分享。
- **iteration-N 是版本号**：每次评测递增（`runner.go:493-511` `nextIterationNumber` 扫描 `iteration-N` 目录取 max+1）。多轮迭代可以对比"v1 的 Skill vs v2 的 Skill"。
- **case 维度子目录**：每个 case 一个目录，互不干扰。
- **with_skill/without_skill** 双分支：仅当 `benchmark.enabled=true` 时才创建 `without_skill` 分支（节省磁盘 + 表达"是否做 A/B"）。

### 6.2 `IterationWorkspace` 结构（`workspace.go:54-86`）

```go
type IterationWorkspace struct {
    RootDir      string
    IterationNum int
    SkillName    string
}

func NewIterationWorkspace(outputDir, skillName string, iterNum int) (*IterationWorkspace, error) {
    if outputDir == "" {
        outputDir = skillName + "-workspace"
    }
    if iterNum < 1 {
        return nil, fmt.Errorf("iteration number must be >= 1, got %d", iterNum)
    }
    if err := os.MkdirAll(outputDir, dirPerm); err != nil {
        return nil, fmt.Errorf("create workspace root %s: %w", outputDir, err)
    }
    return &IterationWorkspace{
        RootDir:      outputDir,
        IterationNum: iterNum,
        SkillName:    skillName,
    }, nil
}
```

3 个不变量：

1. **`outputDir` 空时默认 `<skill>-workspace`**：和 `Evaluate` 中 `runner.go:122-126` 的默认行为呼应。
2. **`iterNum < 1` 拒绝**：1-based 索引，避免 `iteration-0` 这种容易混淆的目录。
3. **构造时即创建 root**：`MkdirAll` 幂等，存在不报错。

权限常量（`workspace.go:30-33`）：

```go
const (
    dirPerm  = 0o755  // rwxr-xr-x
    filePerm = 0o644  // rw-r--r--
)
```

**0o755 / 0o644** 是 Unix 标准：目录 755（owner 可写，其他可读可进入），文件 644（owner 可写，其他只读）。Go 的 `0o` 前缀是 Go 1.13+ 的八进制字面量，比 `0755` 更醒目。

### 6.3 路径计算（`workspace.go:88-111`）

```go
func (w *IterationWorkspace) IterationDir() string {
    return filepath.Join(w.RootDir, fmt.Sprintf("iteration-%d", w.IterationNum))
}

func (w *IterationWorkspace) CaseDir(caseID string) string {
    return filepath.Join(w.IterationDir(), filepath.Clean(caseID))
}

func (w *IterationWorkspace) WithSkillDir(caseID string) string {
    return filepath.Join(w.CaseDir(caseID), "with_skill")
}

func (w *IterationWorkspace) WithoutSkillDir(caseID string) string {
    return filepath.Join(w.CaseDir(caseID), "without_skill")
}

func (w *IterationWorkspace) ConfigDir(caseID, config string) string {
    return filepath.Join(w.CaseDir(caseID), config)
}
```

每个方法都是纯函数式路径拼接，**无副作用**——可以在不创建目录的情况下预算路径。`filepath.Join` 自动处理跨平台分隔符（Windows `\` vs Unix `/`）。

注意 `CaseDir` 调用了 `filepath.Clean(caseID)`——这是**第一道防御**，把 `./foo` 或 `foo/` 之类的脏路径清掉。但还不够（见下文 `validateCaseID`）。

### 6.4 `EnsureDirs` vs `EnsureDirsWithBaseline`（`workspace.go:113-140`）

```go
func (w *IterationWorkspace) EnsureDirs(caseIDs []string) error {
    if err := w.ensureIterationDir(); err != nil {
        return err
    }
    for _, caseID := range caseIDs {
        if err := validateCaseID(caseID); err != nil {
            return err
        }
        if err := w.ensureWithSkillOutputs(caseID); err != nil {
            return err
        }
    }
    return nil
}

func (w *IterationWorkspace) EnsureDirsWithBaseline(caseIDs []string) error {
    if err := w.EnsureDirs(caseIDs); err != nil {
        return err
    }
    for _, caseID := range caseIDs {
        if err := w.ensureWithoutSkillOutputs(caseID); err != nil {
            return err
        }
    }
    return nil
}
```

- `EnsureDirs`：只创建 `with_skill/outputs`。
- `EnsureDirsWithBaseline`：先调 `EnsureDirs`，再补 `without_skill/outputs`。

`EnsureDirsWithBaseline` 复用 `EnsureDirs` 而非重写——这是**模板方法模式**的轻量实现（先做 A，再做 B = A + 增量）。

**自适应调度**（`runner.go:268-279`）：

```go
func ensureCaseDirs(ws *report.IterationWorkspace, results []evaluator.EvalResult, caseIDs []string) error {
    var err error
    if hasBaselineConfiguration(results) {
        err = ws.EnsureDirsWithBaseline(caseIDs)
    } else {
        err = ws.EnsureDirs(caseIDs)
    }
    ...
}
```

runner 不写死"用哪个"，而是**根据 results 里有没有 `without_skill` 配置动态选择**。这种"数据驱动布局"的好处是：评测逻辑改了（开启/关闭 benchmark），目录布局自动跟上，零代码改动。

### 6.5 `validateCaseID` 防穿越（`workspace.go:45-51`）

```go
func validateCaseID(caseID string) error {
    cleaned := filepath.Clean(caseID)
    if filepath.IsAbs(cleaned) || strings.Contains(cleaned, "..") || strings.ContainsAny(cleaned, `/\`) {
        return fmt.Errorf("invalid case ID: %q", caseID)
    }
    return nil
}
```

3 重拦截：

1. **`filepath.IsAbs`**：拒绝绝对路径（如 `/etc/passwd` 或 `C:\Windows`）。
2. **`strings.Contains(cleaned, "..")`**：拒绝"../"穿越（即使 Clean 之后还残留，比如 `foo/../etc`）。
3. **`ContainsAny(cleaned, `/\`)`**：拒绝**任何**分隔符——意味着 case ID 只能是单层目录名，不能是 `sub/foo`。

**为什么这么严？** 因为 case ID 来自 YAML 配置文件（`evals[].id`），是**用户输入**。如果用户写 `id: ../../../etc/passwd`，写入 grading.json 时会逃出 workspace 边界——这是经典的**路径穿越攻击（Path Traversal）**。

> **Java 类比**：Spring 的 `Paths.get(base).resolve(userInput).normalize()` + 检查 `startsWith(base)`。OWASP 推荐的"规范化后比对前缀"策略。Go 这里更严，直接禁止所有分隔符。

### 6.6 Windows 拦截：`isRootedPath`（`workspace.go:33-42`）

```go
// isRootedPath reports whether p begins with either path separator. On
// Windows filepath.IsAbs requires a volume name (e.g. `C:\…`), so a POSIX-
// style "/abs.txt" passed in from user-supplied YAML would otherwise be
// accepted as a relative path; treat any leading `/` or `\` as rooted to
// keep the path-traversal guard OS-independent.
func isRootedPath(p string) bool {
    return strings.HasPrefix(p, "/") || strings.HasPrefix(p, `\`)
}
```

注释非常清楚地解释了动机：**`filepath.IsAbs` 在 Windows 上要求带盘符**，所以 `/etc/passwd` 在 Windows 上会被误判为"相对路径"！如果只依赖 `IsAbs`，攻击者用 POSIX 风格路径就能绕过。这里**额外**用 `isRootedPath` 拦截"任何以 `/` 或 `\` 开头的路径"，做到 OS 无关。

这是 `WriteFile`（`workspace.go:215-234`）的防御：

```go
func (w *IterationWorkspace) WriteFile(relPath string, data []byte) error {
    cleaned := filepath.Clean(relPath)
    if filepath.IsAbs(cleaned) || isRootedPath(cleaned) || strings.HasPrefix(cleaned, "..") {
        return fmt.Errorf("invalid relative path: %s", relPath)
    }
    path := filepath.Join(w.IterationDir(), cleaned)
    dir := filepath.Dir(path)
    if err := os.MkdirAll(dir, dirPerm); err != nil {
        return fmt.Errorf("create dir for %s: %w", relPath, err)
    }
    if err := os.WriteFile(path, data, filePerm); err != nil {
        return fmt.Errorf("write file %s: %w", path, err)
    }
    return nil
}
```

注意：`relPath` 可以是多层（`sub/dir/file.txt`），所以**不**像 `validateCaseID` 那样禁止所有分隔符——只禁止"逃逸"（绝对路径、`..` 前缀）。这是 `result.json` / `report.html` 等文件落盘的通用入口。

### 6.7 写入方法族（`workspace.go:166-212`）

```go
func (w *IterationWorkspace) WriteResponse(caseID, config, content string) error { ... }
func (w *IterationWorkspace) WriteGrading(caseID, config string, grading *AnthropicGrading) error { ... }
func (w *IterationWorkspace) WriteEvalMeta(caseID string, meta *EvalMetadata) error { ... }
func (w *IterationWorkspace) WriteBenchmark(bm *AnthropicBenchmark) error { ... }
func (w *IterationWorkspace) WriteBenchmarkMD(bm *AnthropicBenchmark) error { ... }
```

每个方法**先 `validateCaseID` 再写**——这是"防御编程"的体现，不假设调用方都干净。即使 runner 已经在外面 validate 过，这里再 validate 一次是双保险（也方便单测调用）。

---

## 7. Runner.WriteResults 编排（`runner.go:191-238`）

这是把上面所有组件串起来的总入口。**逐段拆解**：

### 7.1 入口 + 校验

```go
func (r *Runner) WriteResults(ctx context.Context, results []evaluator.EvalResult,
    skillName, skillPath string, runNumber int, formats []string) error {
    if r.workspace == nil {
        return errors.New("workspace not initialized; call InitWorkspace first")
    }
    ws := r.workspace
```

前置校验：workspace 必须存在。错误信息是**给开发者的指令**（"call InitWorkspace first"），不是给用户看的——这种"指出修复路径"的错误信息能极大缩短排错时间。

### 7.2 第 1 步：收集 case ID + 创建目录（`runner.go:198-201`）

```go
caseIDs := indexCaseResults(results)
if err := ensureCaseDirs(ws, results, caseIDs); err != nil {
    return err
}
```

`indexCaseResults`（`runner.go:241-252`）—— 按**首次见到**顺序去重：

```go
func indexCaseResults(results []evaluator.EvalResult) []string {
    seen := make(map[string]struct{}, len(results))
    caseIDs := make([]string, 0, len(results))
    for _, res := range results {
        if _, ok := seen[res.CaseID]; ok {
            continue
        }
        seen[res.CaseID] = struct{}{}
        caseIDs = append(caseIDs, res.CaseID)
    }
    return caseIDs
}
```

`struct{}` 是零大小类型——map 当 set 用，节省内存。Java 等价：`Set<String> seen = new HashSet<>()`。

### 7.3 第 2 步：分组（`runner.go:203`）

```go
grouped := groupResultsByCase(results)
```

`groupResultsByCase`（`runner.go:283-297`）—— 把扁平的 `[]EvalResult` 按 case ID 装进 `caseResults{withSkill, withoutSkill}` 桶：

```go
func groupResultsByCase(results []evaluator.EvalResult) map[string]*caseResults {
    grouped := make(map[string]*caseResults)
    for i := range results {
        res := &results[i]
        if _, ok := grouped[res.CaseID]; !ok {
            grouped[res.CaseID] = &caseResults{}
        }
        if res.Configuration == "without_skill" {
            grouped[res.CaseID].withoutSkill = res
        } else {
            grouped[res.CaseID].withSkill = res
        }
    }
    return grouped
}
```

注意这里和 `html.go:185-205` 的 `groupCaseResults` **几乎是同一段代码**——一个作用于 `evaluator.EvalResult`（runner 层），一个作用于 `report.CaseResult`（report 层）。这种"两层分组"略有一点重复，但**好处是解耦**——report 包不依赖 runner 包的类型。

### 7.4 第 3 步：写 per-case 产物 + 累积 benchmark runs（`runner.go:205-208`）

```go
withSkillRuns, withoutSkillRuns, err := r.writeGroupedCaseArtifacts(ws, grouped, caseIDs, runNumber)
```

`writeGroupedCaseArtifacts`（`runner.go:301-318`）—— 遍历每个 case，调 `writeCaseArtifacts` 落盘 response/grading/eval_metadata，同时 `resultToBenchmarkRun` 把 result 转成 Anthropic 格式的 run：

```go
for _, caseID := range caseIDs {
    cr := grouped[caseID]
    if cr.withSkill != nil {
        if writeErr := r.writeCaseArtifacts(ws, cr.withSkill, "with_skill"); writeErr != nil { ... }
        withSkillRuns = append(withSkillRuns, resultToBenchmarkRun(cr.withSkill, runNumber))
    }
    if cr.withoutSkill != nil {
        if writeErr := r.writeCaseArtifacts(ws, cr.withoutSkill, "without_skill"); writeErr != nil { ... }
        withoutSkillRuns = append(withoutSkillRuns, resultToBenchmarkRun(cr.withoutSkill, runNumber))
    }
}
```

注意 **named return**（`withSkillRuns, withoutSkillRuns, err`）+ 注释 `//nolint:revive,nonamedreturns`：revive linter 默认禁止 named returns（怕混乱），这里显式豁免——因为签名长 + 文档化的语义（注释 `named returns required by revive confusing-results`）。这是"明知规则、显式破例"的工程纪律。

`writeCaseArtifacts`（`runner.go:346-370`）落盘三件套：

1. `ws.WriteResponse` → `<case>/<config>/outputs/response.md`
2. `ws.WriteGrading`（如果 grading 非 nil）→ `<case>/<config>/grading.json`
3. `ws.WriteEvalMeta` → `<case>/eval_metadata.json`

`resultToBenchmarkRun`（`runner.go:372-406`）—— 把 `EvalResult` 转 `BenchmarkRun`（Anthropic 格式）。注意 `Tokens: res.InputTokens + res.OutputTokens`——Anthropic schema 把 input/output tokens 合并成一个数字，和内部 Benchmark（分开）不同。

### 7.5 第 4 步：计算 + 写 Benchmark（`runner.go:210-216`）

```go
bm := report.ComputeAnthropicBenchmark(skillName, skillPath, withSkillRuns, withoutSkillRuns)
if err := ws.WriteBenchmark(bm); err != nil {
    return fmt.Errorf("failed to write benchmark.json: %w", err)
}
if err := ws.WriteBenchmarkMD(bm); err != nil {
    return fmt.Errorf("failed to write benchmark.md: %w", err)
}
```

同时写 **JSON + Markdown** 两个版本：JSON 给工具，Markdown 给人。Markdown 渲染逻辑在 `benchmark_md.go`，根据 with/without 是否齐全渲染**单列 or 对比表格**（`benchmark_md.go:45-70`）。

### 7.6 第 5 步：构建 Input + 写 result.json（`runner.go:218-227`）

```go
input := buildReportInput(skillName, grouped, caseIDs, r.startTime, r.endTime, r.evalCfg)

resultJSON, err := json.MarshalIndent(input, "", "  ")
if err != nil {
    return fmt.Errorf("failed to marshal result.json: %w", err)
}
if err := ws.WriteFile("result.json", append(resultJSON, '\n')); err != nil {
    return fmt.Errorf("failed to write result.json: %w", err)
}
```

`result.json` 是 **总事实源**——比 `report.json` 更权威（后者由 `--format json` 触发，是冗余副本）。注释 `runner.go:220-221` 明确：

> result.json is always written as the raw evaluation data source.

`buildReportInput`（`runner.go:408-437`）构造 `report.Input`：

```go
modelName := evalCfg.Engine.Model.Name
if evalCfg.Engine.Model.Provider != "" && modelName != "" {
    modelName = evalCfg.Engine.Model.Provider + "/" + modelName   // e.g. "openai/gpt-4o"
}
```

这里把 provider/name 拼成 `openai/gpt-4o` 这种"slash path"——是 LLM 社区的通用约定（litellm、openrouter 都用这种格式）。

### 7.7 第 6 步：多格式报告（`runner.go:229-231`）

```go
if err := writeFormattedReports(ctx, ws.IterationDir(), formats, input); err != nil {
    return err
}
```

`writeFormattedReports`（`runner.go:323-344`）—— 遍历 `formats` slice，对每个 format 实例化对应 Reporter：

```go
for _, format := range formats {
    switch format {
    case "json":
        reporter := &report.JSONReporter{OutputPath: filepath.Join(iterDir, "report.json")}
        ...
    case "junit":
        reporter := &report.JUnitReporter{OutputPath: filepath.Join(iterDir, "report.xml")}
        ...
    case "html":
        reporter := &report.HTMLReporter{OutputPath: filepath.Join(iterDir, "report.html")}
        ...
    }
}
```

3 个观察：

1. **switch + 类型字面量实例化**：根据 format 字符串选择具体 Reporter 类型。这是**工厂方法模式**的轻量版（没有显式 Factory 类，直接 switch）。
2. **每个 Reporter 都 new 一个**：没有复用——但因为只调用一次 `Write`，复用没有意义。
3. **路径拼接用 `filepath.Join`**：跨平台安全。
4. **未知 format 静默忽略**（switch 没有 default）——这是宽容解析，避免 CLI 拼写错误阻塞整个流程（前面已经有 result.json 兜底）。

### 7.8 第 7 步：UI 反馈（`runner.go:233-237`）

```go
ui.Blank()
ui.Step("📊", "Generating reports...")
logging.DebugContextf(ctx, "Runner: results written to %s", ws.IterationDir())
ui.Statusf("✅", "Reports saved to %s", ws.IterationDir())
```

最后一步是**用户体验**——空行 + 进度 emoji + 完成状态。这是 CLI 工具的"软实力"。

### 7.9 完整编排时序

```
WriteResults(results, formats)
  │
  ├─ indexCaseResults         ──→ caseIDs (有序去重)
  ├─ ensureCaseDirs           ──→ 创建目录树 (with/without 自适应)
  ├─ groupResultsByCase       ──→ 按 case 分组
  │
  ├─ writeGroupedCaseArtifacts
  │     ├─ writeCaseArtifacts (per case × per config)
  │     │     ├─ WriteResponse     → response.md
  │     │     ├─ WriteGrading      → grading.json  (Anthropic 兼容)
  │     │     └─ WriteEvalMeta     → eval_metadata.json
  │     └─ resultToBenchmarkRun    → 累积 with/without runs
  │
  ├─ ComputeAnthropicBenchmark ──→ benchmark.json schema
  ├─ WriteBenchmark / WriteBenchmarkMD  → benchmark.{json,md}
  │
  ├─ buildReportInput          ──→ report.Input (含 Cases, TotalTokens)
  ├─ ws.WriteFile("result.json")  ──→ 总事实源
  │
  ├─ writeFormattedReports     ──→ report.{json,xml,html} (按 formats)
  │     ├─ JSONReporter.Write
  │     ├─ JUnitReporter.Write
  │     └─ HTMLReporter.Write
  │
  └─ ui.Step / ui.Statusf      ──→ CLI 反馈
```

---

## 8. 关键设计点（带 Java 类比）

### 8.1 策略模式多 Reporter（OCP 开闭原则）

`Reporter` 接口 + 3 个实现（JSON / JUnit / HTML）+ `writeFormattedReports` 工厂调度。

- **新增格式（如 Sarif / CSV）只需**：写一个新 struct 实现 `Reporter`，在 switch 里加一个 case。**不动现有代码**。
- **Java 类比**：Spring 的 `@Component` + `Map<String, Reporter>` 注入。Go 没有依赖注入容器，用 switch 字符串分发更直接，但失去了"自动注册"的能力（要改 switch）—— trade-off。

### 8.2 模板渲染 + 数据/视图分离

`Input` 是 Model，`JSONReporter` 是直接的 Model 序列化，`JUnitReporter` / `HTMLReporter` 是 View。**JUnit 和 HTML 都从同一份 Input 派生**，避免多源真相。

- **Java 类比**：Spring MVC 的 `@Controller` 返回 `ModelAndView`。Go 这里没有 Controller，但 Reporter 接口扮演了"格式化视图"的角色。
- HTML 进一步把"渲染"让渡给前端 JS（`template.JS` 塞 JSON）—— 是 **Isomorphic** 思想的简化版。

### 8.3 A/B 实验量化 Skill 有效性

`ExtractMetrics → ComputeBenchmarkStats (mean+stddev) → ComputeBenchmarkDelta` 这条链路是 Skill 评测的**核心价值兑现**。没有它，skill-up 只是个"跑用例"工具；有了它，skill-up 是个"**Skill ROI 评估平台**"。

- `with_skill - without_skill` 的 delta 回答"**这个 Skill 值得上线吗**"？
- min/max/stddev 回答"**结果稳吗**"？
- **Java 类比**：A/B testing 框架（如 Optimizely、Statscount）的核心数据管线。skill-up 简化了——没有 t-test，但 mean+stddev 足够给"肉眼判断"用。

### 8.4 生态兼容选型（Anthropic schema）

`grading.json` / `eval_metadata.json` / `benchmark.json` 全部对齐 Anthropic skill-creator 的 schema。

- **战略意义**：进入主流生态 > 自建生态。让用户已有的 Anthropic 工具链（eval-viewer 等）零成本接入。
- **Java 类比**：Spring 对齐 JPA / Hibernate；Micronaut 对齐 CDI；Apache 项目几乎都对齐各自的工业标准。**"少发明标准，多对齐事实标准"** 是开源项目成功的关键。

### 8.5 Functional Options + 数据驱动布局

- `ComputeAnthropicBenchmark` 用 Functional Options（`WithTimestamp`）表达可选参数—— **Go 风格的 Builder**。
- `ensureCaseDirs` 根据 results 数据动态选择 `EnsureDirs` vs `EnsureDirsWithBaseline`——**数据驱动布局**，配置变了目录自适应。

---

## 9. 面试 Q&A

### Q1：为什么 skill-up 同时输出 JSON / JUnit / HTML 三种格式？能不能合并？

**A**：三种格式服务三种不同场景，**无法合并**：

- **JSON** 是**事实源（fact source）**，给程序消费（下游脚本、eval-viewer）。结构化、可 diff、可重处理。
- **JUnit XML** 是 **CI 系统通用语言**——Jenkins / GitHub Actions 直接根据 `<failure>` / `<error>` 拍板红绿。没有它，CI 集成需要自己写解析器。
- **HTML** 是**人类可读视图**——单文件可分享、有交互（侧边栏导航 + 折叠 grading），方便 code review 时肉眼对比。

它们对应 MVC 的 Model / View-for-machine / View-for-human。共用同一份 `Input`（单一真相源），通过策略模式分发，**零信息冗余**。

### Q2：HTML 报告里 `template.JS` 把整个 JSON 塞进 `<script>` 是什么套路？为什么不直接用模板渲染？

**A**：这是 "**server-side template + client-side rendering**" 的混合模式：

- **Go 模板只渲染骨架**（HTML 结构、CSS、logo），不渲染 case 数据；
- **数据塞进 `<script>const DATA = {...}</script>`**，前端 JS 读 `DATA` 自己渲染交互（折叠、切换 tab、对比 baseline）。

**好处**：
1. 单文件自包含（不依赖外部 JSON 文件），可邮件分享；
2. 交互逻辑（点击展开/折叠）放前端，Go 端无状态；
3. 前端可以独立迭代（改 UI 不用重跑评测）。

**代价**：HTML 体积大（嵌入完整 JSON），不适合巨型 case 集合。

**Java 类比**：Spring `@RestController` 返 JSON + React 前端。只不过这里是单 HTML 文件嵌入。

### Q3：`validateCaseID` 为什么要查 `..` 和 `/\`？`filepath.Clean` 不就够了吗？

**A**：`filepath.Clean` **只规范化路径**，不"拒绝"危险路径。比如：

- `../etc/passwd` → Clean 后还是 `../etc/passwd`（不报错，但仍是逃逸路径）。
- `/etc/passwd` → Clean 后还是 `/etc/passwd`（绝对路径，会覆盖系统文件）。

`validateCaseID` 的 3 重检查：

1. `filepath.IsAbs`：拒绝对对路径（POSIX `/x` 和 Windows `C:\x`）。
2. `strings.Contains(cleaned, "..")`：拒绝穿越（`foo/../bar`）。
3. `ContainsAny(cleaned, "/\\")`：拒绝任何分隔符——把 case ID 限制为**单层目录名**。

**Windows 特例**（`isRootedPath`）：`filepath.IsAbs` 在 Windows 上**要求带盘符**，所以 `/abs.txt` 在 Windows 上会被误判为相对路径！需要**额外**用 `isRootedPath` 拦截"任何以 `/` 或 `\` 开头的路径"，做到 OS 无关。

**Java 类比**：OWASP 推荐的"规范化 → 比对前缀"策略。Go 这里更严，直接禁止所有分隔符，把 case ID 锁死为文件名。

### Q4：为什么 `ComputeAnthropicBenchmark` 用 Functional Options，而不是直接多加一个 `timestamp time.Time` 参数？

**A**：Functional Options（`opts ...func(*benchmarkOptions)`）的好处：

1. **默认值清晰**：不传时用 `time.Now().UTC()`，签名上看不出来，但代码里 `o := benchmarkOptions{timestamp: time.Now().UTC()}` 显式。
2. **易扩展**：未来要加 `WithNote / WithRunsPerConfiguration / WithAuthor`，**不用改签名**——只是新增一个 `WithXxx` 函数。调用方不传就走默认。
3. **可读性**：`ComputeAnthropicBenchmark(name, path, with, without, WithTimestamp(t))` 比 `ComputeAnthropicBenchmark(name, path, with, without, t, "", 1, "")` 可读多了——位置参数多了就记不住。
4. **测试友好**：测试可以注入固定 timestamp，让输出可断言。

**Java 类比**：Builder 模式（`new Builder().withTimestamp(t).build()`）。Go 没有 builder 语法糖，但 Functional Options 达到了同样的"可选 + 链式 + 默认值"效果。

### Q5：result.json 和 report.json 都写出来了，不是冗余吗？

**A**：是有**轻微冗余**，但语义不同：

- **`result.json`**：runner.go:220 注释明确—— "raw evaluation data source"，**永远写**，是评测的"权威档案"。包含 `Input` 的完整 JSON 序列化。
- **`report.json`**：只在 `--format json` 时才写，内容**和 `result.json` 一致**。它的存在是为了**和其他格式（report.xml / report.html）对齐命名**——CLI 上 `--format json,junit,html` 三件套对应 `report.{json,xml,html}`，对称美感 + 一致性。

如果觉得冗余可以**省略 `--format json`**（默认就有 result.json）。但保留 report.json 让"format 列表"语义统一——所有格式都产生 `report.<ext>` 文件，CLI 不需要特殊处理 json。

### Q6：为什么 `IterationWorkspace` 用 1-based iteration number？0 不行吗？

**A**：`workspace.go:72-74` 显式拒绝 `iterNum < 1`：

```go
if iterNum < 1 {
    return nil, fmt.Errorf("iteration number must be >= 1, got %d", iterNum)
}
```

理由：

1. **人类直觉**：用户看到的目录是 `iteration-1, iteration-2`——"第 1 次迭代"，从 1 开始符合自然语言习惯。`iteration-0` 容易让人误以为是"基线"或"未开始"。
2. **对齐 Anthropic**：Anthropic skill-creator 的 eval 输出也是 1-based。
3. **runner 自动检测**（`runner.go:493-511`）：`nextIterationNumber` 扫描目录取 max+1，第一次扫描到 0 个目录就返回 1——和"无 baseline 时第一次跑就是 iteration-1"自洽。

---

## 10. 一句话总结

> **skill-up 的 Report 子系统是"流水线末端的多视图渲染层 + Anthropic 生态兼容层 + Skill A/B 量化层"三合一**：通过 `Reporter` 策略接口把同一份事实数据（`Input`）渲染成 JSON（机器）/ JUnit（CI）/ HTML（人）三种视图，通过 `IterationWorkspace` 把产物组织成 Anthropic 兼容的目录树（`grading.json` / `eval_metadata.json` / `benchmark.json`），通过 `ComputeBenchmark` 系列把"开 Skill vs 不开 Skill"的差异量化成 `mean ± stddev` + `delta`——最终让一个 Go CLI 工具能直接被 Anthropic eval-viewer 消费，把"Skill 有没有用"这件事变成可量化、可对比、可分享的工程数据。

---

## 附：关键文件速查

| 关注点 | 文件:行号 |
|---|---|
| Reporter 接口 | `internal/report/reporter.go:12-14` |
| Input 数据模型 | `internal/report/reporter.go:17-54` |
| JSONReporter | `internal/report/json.go:12-32` |
| JUnitReporter + 三态映射 | `internal/report/junit.go:18-210` |
| HTMLReporter + //go:embed | `internal/report/html.go:19-46, 304-314` |
| with/without 配对 | `internal/report/html.go:185-247` |
| AnthropicGrading 适配器 | `internal/report/grading.go:26-82` |
| EvalMetadata | `internal/report/grading.go:94-115` |
| Benchmark 4 指标 + mean/stddev | `internal/report/benchmark.go:32-162` |
| Anthropic Benchmark + min/max | `internal/report/benchmark_anthropic.go:22-220` |
| Functional Options WithTimestamp | `internal/report/benchmark_anthropic.go:189-199` |
| IterationWorkspace 目录治理 | `internal/report/workspace.go:54-234` |
| validateCaseID 防穿越 | `internal/report/workspace.go:45-51` |
| isRootedPath Windows 防御 | `internal/report/workspace.go:33-42` |
| writeToOutput 管道友好 | `internal/report/helpers.go:28-44` |
| Runner.WriteResults 编排 | `internal/runner/runner.go:191-238` |
| writeFormattedReports 工厂 | `internal/runner/runner.go:323-344` |
| nextIterationNumber 自动编号 | `internal/runner/runner.go:493-511` |
