# 03 · Agent 多引擎适配层 · 源码逐行深度讲解

> **一句话定位**：这层是 skill-up 与「外界各种 Coding Agent CLI（claude-code / codex / qodercli / qwen_code / 你自定义的引擎）」之间的**适配器层**——把"各家 CLI 长得完全不一样的安装/运行/输出/会话"四件事，归一成一个稳定的 Go 接口 `Agent`，让上游 evaluator 不用关心底下到底是哪家。
>
> **为什么需要它**：skill-up 的核心业务是"评测"——同一道题，喂给 N 个 Agent，比谁答得好。如果 evaluator 直接 `exec("claude -p ...")`，每加一个引擎就要改一遍 evaluator；有了适配层，**evaluator 永远只调 `agent.Run(...)`**，新增引擎只是再实现一次接口。这是经典「开闭原则 + 依赖倒置」落地，Java 同学可以直接类比 Spring 的 `DataSource → JdbcTemplate → Driver`。

---

## 0. 全景图：一张图看懂四层

```
                    ┌─────────────────────────────────────────┐
   evaluator ──────►│  Agent 接口 (7 方法) + SessionResumer    │  ← 契约
                    │  internal/agent/agent.go:133             │
                    └─────────────────┬───────────────────────┘
                                      │ 被这 5 个实现满足
            ┌─────────────┬───────────┴───────────┬─────────────┐
            ▼             ▼                       ▼             ▼
       ClaudeCode     Codex                  QoderCLI       QwenCode     CustomAgent
            └─────┬───────┴───────────┬────────┴────────┬──────┘             │
                  │ 嵌入              │ 嵌入            │ 嵌入               │ 嵌入
                  ▼                   ▼                 ▼                    ▼
              ┌──────────────────────────────────────────────┐  ┌────────────────────┐
              │  CLIAgent (模板方法)                          │  │  BaseAgent          │
              │  InstallMCP/InstallSkill/Run/Check            │  │  (直接嵌入，跳过 CLI)│
              │  internal/agent/cli.go:20                     │  └────────────────────┘
              └──────────────────────┬───────────────────────┘
                                     │ 嵌入
                                     ▼
              ┌──────────────────────────────────────────────┐
              │  BaseAgent (基础设施)                         │
              │  Config/Name/SkillPath/CheckCredentials       │
              │  mergeExecOptionsEnv / probeAndMergePATH      │
              │  internal/agent/agent.go:159                  │
              └──────────────────────────────────────────────┘

   生成入口：DetectAgent / DetectAgentWithInitParams （factory.go）
   别名归一：agentkind 包（叶子包，避免常量重复）
```

四个角色对应 Java 经典分层：

| skill-up            | Java/Spring 类比                          | 职责                        |
|---------------------|------------------------------------------|----------------------------|
| `Agent` 接口        | `javax.sql.DataSource`                   | 契约                       |
| `CLIAgent`          | `JdbcTemplate`（模板方法）               | 通用流程骨架               |
| `ClaudeCodeAgent` 等 4 个 | 各家 JDBC `Driver`（MySQL/PG/Oracle） | 具体实现                   |
| `DetectAgent`       | `DriverManager.getConnection`            | 工厂 + 路由                |

---

## 1. Agent 接口：7 个方法的契约

源码：`internal/agent/agent.go:133-148`

```go
type Agent interface {
    Name() string
    Install(ctx context.Context, rt Runtime) error
    InstallMCP(ctx context.Context, rt Runtime, mcpCfg runtime.MCPConfig) error
    InstallSkill(ctx context.Context, rt Runtime, skillCfg runtime.SkillConfig) error
    Run(ctx context.Context, rt Runtime, opts ExecOptions, messages []transcript.Message) (*SessionResult, error)
    Check(ctx context.Context, rt Runtime) error
    CheckCredentials(ctx context.Context) error
}
```

### 1.1 逐方法讲清职责

| 方法 | 职责 | 实现层 | 何时被调 |
|------|------|--------|----------|
| `Name()` | 返回引擎名（claude_code / codex / …） | `BaseAgent.Name` (`agent.go:189`) | 全链路埋点、报告 |
| `Install(ctx, rt)` | 在 runtime 里把 CLI 装好（apt/npm/curl） | 每个**具体引擎**自己重写 | eval 开始前的 setup 阶段 |
| `InstallMCP(ctx, rt, mcpCfg)` | 把 MCP server 注册进 agent | `CLIAgent` 模板方法 + 各引擎自定义 | setup 阶段 |
| `InstallSkill(ctx, rt, skillCfg)` | 把 skill 目录同步进 agent 的 skill 路径 | `CLIAgent` + `installSkillDefault` | setup 阶段 |
| `Run(ctx, rt, opts, messages)` | **核心**：跑一次 agent，拿回 `SessionResult` | 每个具体引擎重写（差异最大） | 每个 case 的执行步 |
| `Check(ctx, rt)` | 检测 CLI 是否在 PATH 里（`command -v X`） | `CLIAgent.Check` (`cli.go:187`) | setup 前 + 故障诊断 |
| `CheckCredentials(ctx)` | 检查 API token 是否设置 | `BaseAgent.CheckCredentials` (`agent.go:199`) | setup 前 |

### 1.2 为什么要这 7 个，不多不少

这 7 个对应了 **"评测一个 Agent 需要的全部副作用"**：

- **3 个准备副作用**：`Install`（装 CLI）、`InstallMCP`（注册 MCP 工具）、`InstallSkill`（装被测 skill 本身）—— skill-up 评的就是 skill，所以 skill 必须能装到 agent 里。
- **1 个核心计算**：`Run` —— 真正跑 prompt，拿回 transcript。
- **2 个自检**：`Check`（CLI 在不在）、`CheckCredentials`（token 在不在）—— 失败时给用户**精确**报错，而不是让命令跑到一半挂掉。
- **1 个元信息**：`Name()`。

> **Java 类比**：这就是 JDBC 的 `Connection` 接口思路——`createStatement / close / setAutoCommit / commit / rollback`，每个方法对应一类副作用；不能少，多了又会限制实现。

### 1.3 SessionResumer：可选能力接口

源码：`internal/agent/agent.go:72-74`

```go
type SessionResumer interface {
    RunTurn(ctx context.Context, rt Runtime, opts ExecOptions,
            message transcript.Message, sessionID string) (*SessionResult, error)
}
```

**关键设计：它和 `Agent` 是两个接口，不是把 `RunTurn` 塞进 `Agent` 里。**

为什么？因为「能不能续接上一轮对话」是**部分引擎**才有的能力：
- claude-code 有 `--resume <sessionID>`
- codex 有 `codex exec resume <sessionID>`
- qodercli 有 `-r <sessionID>`
- qwen_code 和 CustomAgent **不支持**多轮

如果把 `RunTurn` 放进 `Agent` 接口，qwen_code 就得写一个永远 panic 或返回 `errors.New("not supported")` 的空实现——**接口臃肿**。拆成可选接口后，evaluator 用**类型断言**探测：

```go
if resumer, ok := agent.(SessionResumer); ok {
    resumer.RunTurn(...)   // 支持多轮
} else {
    agent.Run(...)         // 退化成单轮
}
```

源码里用 **compile-time 断言**锁死"哪几个引擎实现了 SessionResumer"（`agent.go:151-155`）：

```go
var (
    _ SessionResumer = (*ClaudeCodeAgent)(nil)
    _ SessionResumer = (*QoderCLIAgent)(nil)
    _ SessionResumer = (*CodexAgent)(nil)
)
```

这种 `var _ Interface = (*Type)(nil)` 写法是 Go 的"静态自检"惯用法：编译期就保证类型实现了接口，**不用等到运行时调用才发现漏了方法**。等价于 Java 的 `implements List<String>` 显式声明，但是是用"赋值给空白变量"模拟的。

---

## 2. 四层组合：BaseAgent → CLIAgent → 具体引擎

Go 没有 `extends`，复用靠**结构体嵌入（embedding）**——这等价于 Java 的"组合 + 语言级委托"，但比继承更灵活。

### 2.1 BaseAgent：基础设施层

源码：`internal/agent/agent.go:159-186`

```go
type BaseAgent struct {
    Cfg     Config
    Version string
}

func NewBaseAgent(cfg Config) BaseAgent {
    if cfg.EnvVars == nil {
        cfg.EnvVars = make(map[string]string)
    }
    return BaseAgent{Cfg: cfg, Version: cfg.Version}
}
```

它在 `Config`（`agent.go:96-121`，一坨字段）上提供几个**所有引擎都需要**的方法：

| 方法 | 行号 | 作用 |
|------|------|------|
| `Name()` | `agent.go:189` | 直接返回 `Cfg.Name` |
| `SkillPath()` | `agent.go:194` | 返回 `Cfg.SkillPath`（agent 读 skill 的目录） |
| `CheckCredentials()` | `agent.go:199` | 按 provider（openai/anthropic）查对应 env 变量 |
| `credentialEnvVars()` | `agent.go:240` | 把 API key/baseURL 拼进 env map |
| `mergeExecOptionsEnv()` | `agent.go:332` | 把多个 env 来源合并进 `ExecOptions.Env` |
| `probeAndMergePATH()` | `agent.go:349` | 跑一次探针命令，把 `~/.local/bin` 等加进 PATH |
| `buildAgentObservabilityAttrs()` | `agent.go:362` | 给 OTel trace 加 `engine=xxx` 属性 |
| `installSkillDefault()` | `agent.go:257` | 默认 skill 安装实现（拷目录） |

**关键细节**：`installSkillDefault` 故意**挂在 BaseAgent 而不是 CLIAgent** 上（注释 `agent.go:253-256` 写得很直白）：

> Defined on BaseAgent so both CLIAgent and CustomAgent share it via embedding,
> without making CustomAgent inherit the rest of CLIAgent.

这是后面要讲的"CustomAgent 故意不嵌 CLIAgent"的伏笔——通过把共享方法**下推到最底层**，避免被迫继承一整套 CLI 行为。Java 同学可以理解为：把通用方法从 `AbstractJdbcTemplate` 挪到更基础的 `AbstractDataSource`，让"不走 JDBC 模板的自定义实现"也能用。

### 2.2 CLIAgent：模板方法层

源码：`internal/agent/cli.go:20-22`

```go
type CLIAgent struct {
    BaseAgent
}
```

它只为"**通过 CLI 调用**"的引擎提供模板方法，定义了 4 个流程骨架：

#### 2.2.1 InstallMCP（`cli.go:25-79`）

模板方法逻辑：
1. 检查 `Cfg.InstallMCPCmd` 模板有没有配；没配但有 server 要装 → 报错。
2. 用 **`text/template`** 解析模板（注意：和 `Run` 用的 `fmt.Sprintf` 不是一套，下文细讲）。
3. 对每个 server 渲染模板 → `rt.Exec` 执行。

```go
tmpl, err := template.New("installMCP").Parse(a.Cfg.InstallMCPCmd)
// ...
data := struct {
    Name, Transport, Command string
    Args []string
    Endpoint, ConfigRef, Workspace string
    Env, Headers map[string]string
}{ /* 填充 server 字段 */ }
tmpl.Execute(&buf, data)
```

模板里可以写 `{{.Name}}`、`{{.Endpoint}}` 这种 Go template 语法。

#### 2.2.2 InstallSkill（`cli.go:82-123`）

和 InstallMCP 同构：模板没配 → 走 `installSkillDefault`；配了就用 `text/template` 渲染 `{{.Source}} {{.Target}} {{.Workspace}}`。

#### 2.2.3 Run（`cli.go:126-158`）—— **最关键的模板方法**

```go
func (a *CLIAgent) Run(ctx context.Context, rt Runtime, opts ExecOptions,
                       messages []transcript.Message) (*SessionResult, error) {
    if err := requireBashTargetShell(rt); err != nil { return nil, ... }   // ① Windows 必须 bash
    start := time.Now()

    instruction := BuildInstructionFromMessages(messages)                  // ② messages 拼成单串
    cmd := fmt.Sprintf(a.Cfg.RunCmd, shellQuote(instruction))              // ③ 用 %s 模板渲染
    opts = a.mergeExecOptionsEnv(ctx, opts, ...)                           // ④ 合并 env
    ctx = observability.ContextWithConfiguredAgentSpanAttributes(ctx, opts.Env)
    result, err := rt.Exec(ctx, cmd, opts)                                 // ⑤ 真正执行

    sessionResult := &SessionResult{
        Engine: a.Name(), ExitCode: result.ExitCode, DurationMs: ...,
        FinalMessage: result.Stdout, Stderr: result.Stderr,
        Artifacts: &SessionArtifacts{},
    }
    // ⑥ 错误处理：Exec 失败 / ExitCode != 0
    return sessionResult, nil
}
```

**注意**：这是"通用 Run"——只把 stdout 当 final message，**不解析 transcript**。所以 4 个具体引擎**全都重写了 Run**，因为每家的输出格式完全不一样（claude 有 stream-json、codex 有 NDJSON events、qwen 有 Gemini-shape JSONL、qoder 也是 jsonl）。CLIAgent.Run 实际只在最朴素的"RunCmd 模板引擎"场景才用，更多是**默认基线**作用。

> **Java 类比**：这就是 `JdbcTemplate.execute(String sql)` 的位置——给你个能直接跑 SQL 的兜底，但真要拿结构化结果，你得用 `query(sql, RowMapper)` 这种更具体的方法（对应每个引擎自己重写的 Run）。

#### 2.2.4 Check（`cli.go:164-206`）

```go
checkCmd := checkCommandForOS(a.Cfg.CheckCmd, rt.Shell().GOOS)
```

这里有个**跨平台细节**值得讲（`cli.go:160-184`）：

- 默认 `CheckCmd` 是 `command -v claude`（POSIX）。
- 但 **Windows cmd.exe 没有 `command` 内建**，只有 `where`。
- `checkCommandForOS` 用正则把 `command -v X` 改写成 `where X`，**还会把 `>/dev/null` 改成 `>nul`**（否则 cmd 找不到 `/dev/null` 路径会报错）。

这是个典型的"**跨平台命令归一**"细节，面试可以讲：**不要硬编码平台分支，而要把命令做成可重写的字符串**。

### 2.3 四个具体引擎：嵌入 CLIAgent

每个引擎的写法都极其规整（4 个文件长得几乎一样）：

```go
// claude_code.go:22-25
type ClaudeCodeAgent struct {
    CLIAgent
}

func NewClaudeCodeAgent(cfg Config) *ClaudeCodeAgent {
    if cfg.Name == "" { cfg.Name = "claude-code" }
    cfg.CheckCmd = "command -v claude"
    cfg.SkillPath = ".claude/skills"
    return &ClaudeCodeAgent{
        CLIAgent: CLIAgent{BaseAgent: NewBaseAgent(cfg)},
    }
}
```

`CodexAgent`（`codex.go:24`）、`QoderCLIAgent`（`qodercli.go:19`）、`QwenCodeAgent`（`qwen_code.go:24`）结构完全一致，只是构造时填的默认值不同。

**为什么这样写"好"**：
1. `New*Agent` 是个**便利构造函数**（Java 里叫 static factory method），把"默认值 + 嵌入初始化"一次性封死。
2. 想加第 5 个引擎？复制一份文件，改 5 个常量即可，**不动任何已有代码**——开闭原则。
3. 每个引擎可以**按需重写**任意方法（事实上 4 个都重写了 `Run` / `Install` / `InstallMCP`），不重写的就用 CLIAgent 模板。

### 2.4 CustomAgent：故意不嵌 CLIAgent

这是整个适配层**最值得讲的设计决策**。源码：`internal/agent/custom.go:77-79`：

```go
type CustomAgent struct {
    BaseAgent   // ← 注意：嵌的是 BaseAgent，不是 CLIAgent！
}
```

为什么？看注释（`custom.go:65-78`）说得非常清楚：

> CustomAgent embeds BaseAgent directly (not CLIAgent) so it inherits only
> shared agent infrastructure — Name, credential helpers, exec-options merge,
> installSkillDefault. Run / Install / InstallMCP / InstallSkill / Check /
> CheckCredentials are all defined explicitly, so future additions to CLIAgent
> do not accidentally leak in with the wrong semantics for a custom engine.

翻译一下："Custom Engine 的执行模型**根本不是 CLI**——它可能是本地命令、可能是 HTTP 服务、可能压根没有'安装'这个概念。如果让它嵌 CLIAgent，将来给 CLIAgent 加方法时会**意外漏进来**，对一个 HTTP 引擎来说，'跑 InstallMCP 命令'这种语义完全是错的。"

于是 CustomAgent 把所有方法都**显式实现**了一遍（`custom.go:89-114`）：

```go
func (a *CustomAgent) Install(_ context.Context, _ Runtime) error { return nil }       // no-op
func (a *CustomAgent) InstallMCP(...) error { /* 只打日志，不装 */ }
func (a *CustomAgent) InstallSkill(...) error { return a.installSkillDefault(...) }    // 复用 BaseAgent 的
func (a *CustomAgent) Check(...) error { return nil }                                  // no-op
func (a *CustomAgent) CheckCredentials(...) error { return nil }                       // no-op
func (a *CustomAgent) Run(...) (*SessionResult, error) { /* 大量逻辑 */ }
```

5 个 no-op 看起来"浪费"，但这是**用代码自证意图**："Custom Engine 没有安装/检测概念，只是占个位返回 nil"。比"继承了一堆错误行为"安全得多。

> **Java 类比**：这相当于 `MyBatis` 这种非 JDBC 模板的 ORM，**直接实现 `DataSource` 接口**而不是继承 `AbstractDataSource`——只复用真正需要的基础能力，避免被模板类的默认行为绑架。**"组合优于继承"** 的经典体现，Effective Java Item 18。

---

## 3. 工厂：DetectAgent + agentkind 别名归一

### 3.1 agentkind：消除"两处常量同步"的叶子包

源码：`internal/agentkind/agentkind.go`

这是一个**只有常量**的叶子包，存在的唯一理由（看包注释 `agentkind.go:1-8`）：

> It is a tiny leaf package with no internal dependencies so both
> internal/agent (factory dispatch) and internal/config (validation) can
> import the same source of truth without creating an import cycle —
> previously the list was duplicated in both packages with a "keep in sync"
> comment.

```go
const (
    ClaudeCode      = "claude_code"
    ClaudeCodeAlias = "claude-code"
    Codex           = "codex"
    QoderCLI        = "qodercli"
    QoderAlias      = "qoder"
    QoderCLIAlias   = "qoder-cli"
    QwenCode        = "qwen_code"
    QwenCodeAlias   = "qwen-code"
    QwenAlias       = "qwen"
)
```

**亮点设计**：
- 每个引擎**同时给 `_` 和 `-` 两种写法**的别名（`claude_code` / `claude-code`），因为 YAML 历史上两种都有人写。
- 把常量放在**最底层的叶子包**，避免 `agent` 和 `config` 互相 import 造成循环依赖。
- `IsBuiltin(name)` 函数（`agentkind.go:38-41`）用 map 查表，O(1)。

> **Java 类比**：这相当于把"协议号 / MIME 类型"这种跨模块共享常量放到 `commons-lang3` 之类的叶子 jar——谁都能依赖它，它不依赖任何人。**消除"keep in sync"注释**是关键：注释不能保证同步，编译器才能。

### 3.2 DetectAgent：纯路由

源码：`internal/agent/factory.go:12-34`

```go
func DetectAgent(engineName string, cfg Config) (Agent, error) {
    if cfg.Name == "" { cfg.Name = engineName }
    switch engineName {
    case agentkind.QoderCLIAlias, agentkind.QoderAlias, agentkind.QoderCLI:
        return NewQoderCLIAgent(cfg), nil
    case agentkind.ClaudeCodeAlias, agentkind.ClaudeCode:
        return NewClaudeCodeAgent(cfg), nil
    case agentkind.Codex:
        return NewCodexAgent(cfg), nil
    case agentkind.QwenCode, agentkind.QwenCodeAlias, agentkind.QwenAlias:
        return NewQwenCodeAgent(cfg), nil
    default:
        if cfg.Custom != nil {
            return NewCustomAgent(cfg), nil
        }
        return nil, &UnsupportedAgentError{Name: engineName}
    }
}
```

就是个**大 switch**——但有几个细节：

1. **别名归一**：每个 case 都列出所有别名（`QoderCLIAlias, QoderAlias, QoderCLI`），不管用户写哪种，都路由到同一个 `NewXxxAgent`。
2. **default 走 Custom**：如果引擎名不是内置的，但 `cfg.Custom != nil`（用户在 YAML 里配了 `engine.custom` 块），就当 Custom Engine 处理。
3. **彻底不支持 → `UnsupportedAgentError`**（`errors.go:7-13`）：

```go
type UnsupportedAgentError struct{ Name string }
func (e *UnsupportedAgentError) Error() string {
    return fmt.Sprintf("unsupported agent %q: missing engine.custom", e.Name)
}
```

这是个**可导出的错误类型**（不是普通的 `errors.New`），上游可以用 `errors.As(err, &UnsupportedAgentError{})` 精准识别这类错误并给出"是不是忘了配 engine.custom？"的友好提示。**Go 错误处理的惯用法**：错误类型化 > 字符串匹配。

### 3.3 DetectAgentWithInitParams：处理"auto"串味

源码：`internal/agent/factory.go:39-77`

这个函数比 `DetectAgent` 多做了一步——**把"凭据解析后的初始参数"映射到 Config**。最有意思的是**"auto" 模型剥离**（`factory.go:45-47`）：

```go
model := params.Model
// "auto" is a QoderCLI-specific model tier; strip it for other built-in
// engines so they don't need to hard-code awareness of it.
if model == "auto" && !isQoderCLIEngine(engineName) && params.Custom == nil {
    model = ""
}
```

为什么？`qodercli` 有个"auto"模型档位（自动选 lite/efficient/performance/ultimate），但**别的引擎不懂"auto"是啥**。如果原样把 `"auto"` 传给 claude-code，它会把 `"auto"` 当 model 名发给 Anthropic API，直接 400。

**两种处理思路对比**：
- ❌ 错的：让每个引擎都写一行 `if model == "auto" { model = "" }` —— 4 处重复，未来加引擎还会忘。
- ✅ 对的：在**工厂层一次性剥离**，下游引擎永远看不到 "auto"。

但是！**Custom Engine 保留 "auto"**（`params.Custom == nil` 的判断）——因为 custom engine 是用户自定义的，`auto` 可能是它自己定义的合法档位，由用户通过 `${model}` 自己解读。**"内置归一，自定义透传"** 的边界划得非常清晰。

`QODER_PERSONAL_ACCESS_TOKEN` 处理也是同款思路（`factory.go:62-74`）：

```go
if token := os.Getenv(credential.EnvQoderPersonalAccessToken); token != "" {
    cfg.EnvVars[credential.EnvQoderPersonalAccessToken] = token
}
```

注释（`factory.go:62-66`）讲了一个**真实踩过的坑**：

> QODER_PERSONAL_ACCESS_TOKEN 是 qodercli 自己的鉴权凭证，**独立于**底层 model provider（比如 anthropic）。
> params.APIKey 可能装的是 provider-scoped key（比如 ANTHROPIC_API_KEY），**不能**当成 qodercli 的 token 转发。

也就是说，曾经有过 bug：把 `ANTHROPIC_API_KEY` 错当成 `QODER_PERSONAL_ACCESS_TOKEN` 传给了 qodercli，导致鉴权失败。修复就是：**只从 process env 取 QODER 自己的 token，绝不用 provider API key 顶替**。这种"bug-driven design"在源码里随处可见，注释极有教育意义。

---

## 4. CLIAgent 模板方法 + 两套模板语法

### 4.1 两套模板的分工

这是源码里**很容易被忽略但极其重要**的设计——**同一个项目里有两套模板语法**：

| 模板语法 | 用在哪 | 示例 | 为什么 |
|----------|--------|------|--------|
| `text/template` (`{{.Name}}`) | `InstallMCPCmd` / `InstallSkillCmd` | `claude mcp add {{.Name}} --transport http {{.Endpoint}}` | 字段多（10+），需要结构化访问 |
| `fmt.Sprintf` (`%s`) | `RunCmd` | `qodercli -p "%s" 2>&1` | 只有一个 instruction 占位符，过度用 template 反而啰嗦 |

**源码证据**：

`cli.go:33`（InstallMCP，用 text/template）：
```go
tmpl, err := template.New("installMCP").Parse(a.Cfg.InstallMCPCmd)
```

`cli.go:133`（Run，用 fmt.Sprintf）：
```go
cmd := fmt.Sprintf(a.Cfg.RunCmd, shellQuote(instruction))
```

**Java 类比**：这相当于 Spring 的 `JdbcTemplate.update(String sql, Object... args)`（`?` 占位）vs `MessageFormat.format`（`{0}` 占位）——**简单场景用 printf，复杂场景用 template engine**，是工程上的合理取舍。

### 4.2 各引擎的 buildXxxRunCmd builder

虽然 CLIAgent.Run 用 `fmt.Sprintf` 渲染 `RunCmd` 模板，但**4 个具体引擎都重写了 Run**，各自调用更专用的 `buildXxxRunCmd` 函数。原因：命令构造有**复杂的条件分支**，用模板字符串已经表达不了。

以 **claude_code** 为例（`claude_code.go:311-318`）：

```go
func buildClaudePrintCmd(sessionID, model, instruction string) string {
    cmd := "claude --settings " + shellQuote(`{"disableAllHooks":true}`) +
           " --session-id " + sessionID +
           " -p --permission-mode=bypassPermissions"
    if model != "" {
        cmd += " --model " + shellQuote(model)
    }
    cmd += " " + shellQuote(instruction)
    return cmd
}
```

亮点：
- `--settings {"disableAllHooks":true}` —— 关掉所有用户 hooks（防止用户配置干扰评测）。
- `--permission-mode=bypassPermissions` —— 跳过权限确认，无人值守。
- `shellQuote(instruction)` —— 用 POSIX 安全转义包起来（防止 prompt 里有 `;` `&` 等被 shell 解析）。

**codex** 更复杂（`codex.go:536-551`）：

```go
func buildCodexRunCmdWithLastMessage(instruction, model string,
        provider codexProviderConfig, sandboxFlag, lastMessagePath string) string {
    cmd := "codex exec --json --skip-git-repo-check"
    if sandboxFlag != "" { cmd += " " + sandboxFlag }
    cmd += codexProviderFlags(provider)              // 动态拼 -c model_provider=...
    if model != "" { cmd += " -m " + shellQuote(model) }
    if lastMessagePath != "" {
        cmd += " --output-last-message " + shellQuote(lastMessagePath)
    }
    cmd += " " + shellQuote(instruction)
    return cmd
}
```

注意 `codexProviderFlags` 是**运行时动态生成的**——根据用户配的 provider/baseURL 决定要不要加 `-c model_providers.xxx.xxx=...`，这种逻辑根本没法塞进一个模板字符串。

**结论**：模板方法（CLIAgent.Run 的 `fmt.Sprintf`）只是**兜底基线**；真正干活的 builder 函数才是 4 引擎的差异核心。这就是"模板方法 + 钩子（hook）"模式的活用——CLIAgent 定义骨架，具体引擎用 builder 函数注入差异。

---

## 5. 4 引擎差异对比表 + 3 个亮点细节

### 5.1 差异对比表（一图看完）

| 维度 | claude_code | codex | qodercli | qwen_code |
|------|-------------|-------|----------|-----------|
| **安装方式** | `npm install -g @anthropic-ai/claude-code` | `npm install -g @openai/codex@0.80.0` | `curl https://qoder.com/install \| bash` | `npm install -g @qwen-code/qwen-code` |
| **安装源** | nvm + node 引导 | nvm + node 引导 + 锁版本 | 官方二进制脚本（自包含） | nvm + node 引导 |
| **SkillPath** | `.claude/skills` | `.codex/skills` | `.qoder/skills` | `.qwen/skills` |
| **凭据 env** | `ANTHROPIC_API_KEY` + **双写 `ANTHROPIC_AUTH_TOKEN`** | `OPENAI_API_KEY` | `QODER_PERSONAL_ACCESS_TOKEN` | `OPENAI_API_KEY` + `OPENAI_MODEL` |
| **输出格式** | stream-json / session jsonl | NDJSON events（`--json`） | session jsonl（同 Claude schema） | Gemini-shape jsonl |
| **SessionID 来源** | 自己 `uuid.New()` 生成 | codex 输出的 `thread_id` | 从 session 文件路径提取 | 不支持多轮 |
| **resume 命令** | `claude --resume <id> -p` | `codex exec resume <id>` | `qodercli -r <id> -p` | — |
| **特殊能力** | 双写 AUTH_TOKEN；provider 错误识别 | 合成 `skill-up-openai` provider；MCP 桥接脚本 | 自包含二进制；模型档位（lite/eff/.../ulti） | Gemini schema 兼容；`--yolo` |

### 5.2 亮点细节 1：Claude 双写 `ANTHROPIC_AUTH_TOKEN`

源码：`claude_code.go:119-128`

```go
envVars := a.credentialEnvVars(credential.EnvAnthropicAPIKey, credential.EnvAnthropicBaseURL)
// Some Anthropic-compatible proxies (e.g. internal anthropic-proxy gateways)
// only validate `Authorization: Bearer <token>` and do not recognise the
// `x-api-key` header. The Anthropic SDK switches to Bearer when
// ANTHROPIC_AUTH_TOKEN is set, and the official Anthropic API also accepts
// Bearer, so writing both env vars covers both flavours without forcing
// operators to maintain two credentials.
if a.Cfg.APIKey != "" {
    envVars[credential.EnvAnthropicAuthToken] = a.Cfg.APIKey
}
```

**故事**：Anthropic 官方有**两套**鉴权 header：
- `x-api-key: <key>` —— 官方 SDK 默认（设置 `ANTHROPIC_API_KEY` 时走这个）
- `Authorization: Bearer <token>` —— 设置 `ANTHROPIC_AUTH_TOKEN` 时切换到这个

问题：**很多企业内部代理（anthropic-proxy 网关）只认 Bearer，不认 x-api-key**。如果只设 `ANTHROPIC_API_KEY`，请求会被代理拒掉。

**解法**：把**同一个 API key 同时写到两个 env 变量**——官方 API 两个都接受，代理也能找到 Bearer。**一份凭证，双 header 兼容**。

> **Java 类比**：相当于 Spring Security 里同时配置 `BasicAuth` 和 `Bearer Token` 过滤器，让一套凭证穿过不同的鉴权网关。这种"**兼容性高于纯粹性**"的工程权衡，面试可以重点讲。

### 5.3 亮点细节 2：Codex 合成 `skill-up-openai` provider

源码：`codex.go:397-438`（核心逻辑）+ `codex.go:588-606`（拼命令）

故事背景：codex CLI 默认用 OpenAI 的 `/responses` 端点（`wire_api="responses"`），但很多兼容服务（阿里 dashscope OpenAI-compat、idealab 等）**只支持 `/chat/completions`**。

如果直接配：
```yaml
engine:
  name: codex
  base_url: https://dashscope.aliyuncs.com/compatible-mode/v1
```

codex 会**忽略** `base_url`，继续打官方 `api.openai.com`，返回 400。

**错误思路**：把 provider 名设为 `"openai"`：
```go
-c model_provider="openai"
-c model_providers.openai.base_url="..."
```
不行——codex 内置了 `"openai"` provider 定义，**合并覆盖行为跨版本不稳定**。

**正确思路**（codex.go:39-40 注释）：
```go
codexOpenAIOverrideProvider = "skill-up-openai"
```

用 `skill-up-` 前缀造一个**全新的 provider 名**，强制 codex 从零创建一个 provider 定义（不存在合并问题），并通过 `wire_api="chat"` 强制走 `/chat/completions`：

```go
return codexProviderConfig{
    Name:    codexOpenAIOverrideProvider,    // "skill-up-openai"
    Label:   codexOpenAIOverrideProvider,
    BaseURL: a.Cfg.BaseURL,
    EnvKey:  credential.EnvOpenAIAPIKey,
    WireAPI: codexCustomWireAPI,              // "chat"
}
```

然后 `codexProviderFlags`（codex.go:588）拼成 5 个 `-c` 参数：
```bash
codex exec --json \
  -c 'model_provider="skill-up-openai"' \
  -c 'model_providers.skill-up-openai.name="skill-up-openai"' \
  -c 'model_providers.skill-up-openai.base_url="https://..."' \
  -c 'model_providers.skill-up-openai.env_key="OPENAI_API_KEY"' \
  -c 'model_providers.skill-up-openai.wire_api="chat"' \
  ...
```

**`skill-up-` 前缀**还有个好处：在 codex 日志/命令行里一眼能看出"这是 skill-up 合成的"，方便排错。这种"**给合成物加命名空间前缀**"是个非常实战的工程习惯。

### 5.4 亮点细节 3：Codex MCP 桥接脚本

源码：`codex.go:152-212`

故事：codex 的 `codex mcp add` **不支持 `--header` 参数**给 HTTP MCP server 加自定义 header。但评测时我们就是要测带鉴权 header 的 MCP server，怎么办？

**思路**：用一个**中间人桥接脚本**——用 `npx mcp-remote` 这个工具做 HTTP MCP client，由它带 header，然后用 stdio 把协议转给 codex。

```go
func buildCodexMCPRemoteBridgeScript(server runtime.MCPServerConfig) (string, error) {
    var script strings.Builder
    script.WriteString("exec npx mcp-remote \"$1\"")
    for _, key := range sortedMapKeys(server.Headers) {
        // ...
        script.WriteString(" --header ")
        script.WriteString(header)
    }
    script.WriteString(" 2>/dev/null")
    return script.String(), nil
}
```

然后 `buildCodexMCPRemoteInstallCmd`（codex.go:152）把这段脚本**作为 stdio 命令注册给 codex**：

```go
cmd.WriteString(" -- 'sh' '-c' ")
cmd.WriteString(shellQuote(bridgeScript))    // 桥接脚本
cmd.WriteString(" 'mcp-remote' ")
cmd.WriteString(endpoint)                     // HTTP endpoint 作为 $1
```

最终生成的命令大概是这样：
```bash
codex mcp add my-server \
  --env KEY=val \
  -- 'sh' '-c' 'exec npx mcp-remote "$1" --header "Authorization: Bearer xxx" 2>/dev/null' \
  'mcp-remote' \
  https://mcp.example.com/sse
```

**精彩之处**：
- 用 **shell `-c` + `--` 分隔符**把"stdio 命令"伪装成一个本地程序，但实际它内部转 HTTP。
- `$1` 是 shell 的位置参数，指向后面的 endpoint URL。
- `2>/dev/null` 静音 mcp-remote 的 stderr 噪音。

这是**"用组合代替修改"**的极致——codex 不支持？没关系，外面套一层 shell 脚本就行。**适配器模式的终极体现**。

---

## 6. transcript 归一：把 4 种 JSON 翻译成同一份 Transcript

### 6.1 为什么这是核心难点

`SessionResult.Transcript`（`agent.go:60`）是个统一结构：
```go
type SessionResult struct {
    // ...
    Transcript transcript.Transcript `json:"transcript,omitempty"`
    InputTokens  int
    OutputTokens int
    Turns        int
    FinalMessage string
}
```

但 4 家 Agent 的**原始输出格式完全不同**：

| 引擎 | 原始格式 | 文件位置 |
|------|----------|----------|
| claude_code | stream-json（NDJSON，type=user/assistant/tool_call/tool_result/result） | `~/.claude/projects/<wskey>/*.jsonl` |
| qodercli | 同 claude schema | `~/.qoder/projects/<wskey>/*.jsonl` |
| codex | NDJSON events（type=thread.started/turn.started/item.completed/...） | stdout + `~/.codex/sessions/*<thread_id>*.jsonl` |
| qwen_code | Gemini schema（role ∈ {user, model}，parts 数组） | `~/.qwen/projects/<wskey>/chats/*.jsonl` |

适配层要做的是：**把这些完全不同的 JSON，翻译成 `[]transcript.Message`**——一个统一的 `{Role, Content, Turn, ToolCall, ToolResult}` 结构。

### 6.2 Claude / Qoder 共享 parseSessionFile

源码：`claude_code.go:843-896`

由于 qodercli 的 session 文件 schema **和 claude 完全一样**（都是 Anthropic 协议），它**直接复用** `parseSessionFile`：

```go
// qodercli.go:194-204
withDownloadedSession(..., func(artifactPath string) {
    t, f, inTok, outTok := parseSessionFile(artifactPath)   // ← 复用 claude 的
    if len(t) > 0 { trans = t; finalMsg = f }
    inputTokens, outputTokens = inTok, outTok
})
```

`parseSessionFile` 的核心循环（claude_code.go:858-882）：
```go
for scanner.Scan() {
    line := strings.TrimSpace(scanner.Text())
    // ...
    var event claudeEvent
    if err := json.Unmarshal([]byte(line), &event); err != nil { continue }

    var final string
    var turnMsgs []transcript.Message
    turnMsgs, final, currentTurn = processClaudeEvent(event, currentTurn)  // 分发
    messages = append(messages, turnMsgs...)
    if final != "" { finalMsg = final }

    // token 累计
    if event.Type == "assistant" && event.Message != nil {
        inputTokens = max(inputTokens, sessionUsageInputTotal(event.Message.Usage))
        outputTokens = max(outputTokens, event.Message.Usage.OutputTokens)
    }
}
```

**最妙的是 token 聚合用 `max` 而不是累加**——源码注释（claude_code.go:828-842）讲了一个非常细的语义：

> Session file semantics (verified against local ~/.claude/projects/*.jsonl):
> message.usage on successive "assistant" lines is a **running/cumulative meter snapshot**,
> not a per-line delta — e.g. the same input_tokens often repeats across many lines
> while output_tokens ratchets up within one model step.

翻译："Claude 的 session 文件里，每条 assistant 事件带的 usage **不是这一条花的 token，而是从会话开始到这一条的累计快照**。所以你看到 input_tokens 在很多行里**完全相同**（因为 prompt 缓存命中），output_tokens 单调递增。如果直接 sum，会**重复计算多次**；用 max 取高峰水位才对。"

这是个**只有真的解析过真实数据才会知道**的坑。源码注释把它讲得清清楚楚，是面试的**金矿**——展示你不是"会用 API"，而是"懂底层语义"。

### 6.3 Codex：双解析器 + 两阶段关联

源码：`codex.go:683-710`（stdout 解析）+ `codex.go:929-993`（session 文件解析）

codex **比 claude 复杂**——它有两个数据源：

1. **stdout 的 NDJSON events**：包含 `thread.started` / `turn.started` / `item.started` / `item.completed` / `turn.completed` 等。
2. **session 文件**（`~/.codex/sessions/*<thread_id>*.jsonl`）：包含 `response_item` / `event_msg` 两种类型。

为什么有两个？因为：
- stdout 是**实时事件流**，但有些 item 只在 `item.started` 出现（如 command_execution 的 command），完成事件只有 exit code。
- session 文件是**事后落盘的整理结果**，更完整。

**两阶段关联策略**（codex.go:465-476）：
```go
withDownloadedSession(..., findCodexSessionPath(..., streamParsed.threadID), ...,
    func(artifactPath string) {
        sessionParsed := parseCodexSessionFile(artifactPath)
        // 只有当 session 文件解析出的 transcript 更长时才覆盖
        if len(sessionParsed.transcript) > len(streamParsed.transcript) &&
           (sessionParsed.finalMsg != "" || result.ExitCode != 0 || finalMsg == "") {
            trans, finalMsg = sessionParsed.transcript, sessionParsed.finalMsg
            inputTokens = sessionParsed.inputTokens
            outputTokens = sessionParsed.outputTokens
        }
    })
```

注意几个细节：
1. **threadID 是从 stdout 解析出来的**（`thread.started` 事件），然后用它去**找 session 文件**（文件名包含 thread_id）。
2. **不是无脑覆盖**：只有 session 文件的 transcript **更长**（信息更全）且最终消息非空时才用。
3. **command_execution 用 map 关联 started/completed 事件**（`codex.go:659-662, 787-818`）：

```go
type codexCommandState struct {
    command string
    turn    int
}
// applyCodexItemStarted 里记录
state.commandExecutions[event.Item.ID] = codexCommandState{command: ..., turn: turn}
// applyCodexItemCompleted 里查回
cmdState, ok := state.commandExecutions[event.Item.ID]
if ok { turn = cmdState.turn }    // 保证 tool_call 和 tool_result 在同一 turn
```

因为 codex 的 stdout 里，"命令开始"和"命令结束"是**两个独立事件**，要靠 ID 关联才能拼成一个完整的 `ToolCall + ToolResult` 对——这是事件溯源（event sourcing）模式的标准做法。

### 6.4 Qwen：Gemini schema 的翻译

源码：`qwen_code.go:302-389`

qwen_code 是 **Gemini CLI 的 fork**（注释 `qwen_code.go:22`），所以它的 session 文件用的是 **Gemini 的 schema**：

```go
type qwenSessionEvent struct {
    Type          string              `json:"type"` // user | assistant | system
    Model         string              `json:"model,omitempty"`
    Message       *qwenSessionMessage `json:"message,omitempty"`
    UsageMetadata *qwenUsageMetadata  `json:"usageMetadata,omitempty"`
}

type qwenSessionMessage struct {
    Role  string            `json:"role"`  // user | model（注意：assistant 在 Gemini 里叫 model！）
    Parts []qwenSessionPart `json:"parts"`
}
```

注意 `Role` 在 Gemini 里是 `"model"` 而不是 `"assistant"`——`appendQwenMessageParts`（qwen_code.go:395）做了**角色映射**：

```go
role := transcript.RoleAssistant
if ev.Type == qwenTypeUser { role = transcript.RoleUser }
// 用 ev.Type 而不是 ev.Message.Role 判定，绕开 Gemini 的 "model" 命名差异
```

`Parts` 是个**异构数组**，每个元素可能是：
- `{text: "..."}` —— 文本
- `{functionCall: {id, name, args}}` —— 工具调用
- `{functionResponse: {id, name, response}}` —— 工具返回

`appendQwenMessageParts`（qwen_code.go:395-438）用一个 `switch` 翻译：

```go
for _, p := range ev.Message.Parts {
    switch {
    case p.FunctionCall != nil:
        *messages = append(*messages, transcript.Message{
            Role: transcript.RoleToolCall,
            ToolCall: &transcript.ToolCallInfo{ID: p.FunctionCall.ID, Name: ..., Arguments: ...},
        })
    case p.FunctionResponse != nil:
        *messages = append(*messages, transcript.Message{Role: transcript.RoleToolResult, ...})
    case p.Text != "":
        textParts = append(textParts, p.Text)
    }
}
```

**Token 语义也归一**（qwen_code.go:347-349 注释）：
```go
// totalTokenCount == promptTokenCount + candidatesTokenCount (+ thoughtsTokenCount);
// cachedContent is a subset of prompt, so it is NOT added again to avoid double counting.
```

`thoughtsTokenCount` 是 Gemini 特有的"思考 token"（reasoning model 的内部推理），算作输出。**注释专门说明"不要把 cachedContent 加进 total，否则双重计算"**——又一次"踩过坑所以写注释"的典范。

---

## 7. 三个支撑模块：prompt_delivery / node_install / session_lookup

### 7.1 prompt_delivery：超长 prompt 的双模式

源码：`internal/agent/prompt_delivery.go`

**问题**：CLI 命令行有长度上限（Linux `ARG_MAX` 通常 128KB~2MB）。评测时 prompt 可能是几万字的代码评审任务，直接塞进 `claude -p "<几万字>"` 会**触发 ARG_MAX，命令直接报错**。

**解法**：双模式自适应（`prompt_delivery.go:33-67`）：

```go
const defaultPromptInlineMaxBytes = 32 * 1024  // 32KB 阈值

func deliverPrompt(ctx context.Context, rt Runtime, opts ExecOptions,
        instruction string, builder promptCommandBuilder) (string, *PromptDeliveryMetadata, error) {
    threshold := promptInlineMaxBytes()
    promptBytes := len([]byte(instruction))
    meta := &PromptDeliveryMetadata{
        Mode: "inline", PromptBytes: promptBytes, InlineMaxBytes: threshold,
    }
    if builder.Inline == nil { return "", meta, errors.New("...") }

    if promptBytes <= threshold {
        return builder.Inline(instruction), meta, nil        // ① 短 prompt：内联
    }
    if builder.StdinFile == nil { return "", meta, fmt.Errorf("...") }

    // ② 长 prompt：写文件 + cat 管道
    runtimePath := filepath.Join(rt.Workspace(), ".skill-up", "prompts", "prompt.txt")
    persistRuntimeArtifact(ctx, rt, runtimePath, instruction)
    // ...
    meta.Mode = "file"
    return builder.StdinFile(runtimePath), meta, nil
}
```

每个引擎的 Run 用一个 `promptCommandBuilder` 提供**两种渲染方式**（claude_code.go:150-157）：

```go
deliverPrompt(ctx, rt, opts, instruction, promptCommandBuilder{
    Inline: func(prompt string) string {
        return buildClaudePrintCmd(sessionID, model, prompt)         // claude -p "<prompt>"
    },
    StdinFile: func(path string) string {
        return buildClaudePrintStdinCmd(sessionID, model, path)      // cat <path> | claude -p
    },
})
```

**模式选择由 deliverPrompt 决定**，引擎自己不用关心 prompt 多长——又是**模板方法 + 钩子**的体现。

**返回的 PromptDeliveryMetadata**（prompt_delivery.go:20-26）会被记进 `SessionResult.PromptDelivery`，让下游报告能展示"这个 case 是 inline 还是 file 模式跑的"——**可观测性贯穿始终**。

> **Java 类比**：相当于 Spring 的 `MultipartFile` 上传——小于阈值的用 inline 字段，大于阈值的自动转临时文件。**用户无感**。

### 7.2 node_install：nvm/node 引导 + SHA256 + 镜像回退

源码：`internal/agent/node_install.go`

claude_code / codex / qwen_code 都是 **Node.js 写的 CLI**（codex 也有 node shebang）。但**评测环境的 runtime（docker/opensandbox）可能没装 node**——所以 skill-up 要自己引导 nvm + node。

**核心函数**：`nodeBootstrapLines(nodeVersion)` 返回一段 shell 脚本（node_install.go:37-71），逻辑：

```
1. 设置 NVM_DIR、NVM_SOURCE、NVM_NODEJS_ORG_MIRROR、npm_config_registry（全部支持 env 覆盖）
2. 探测当前 node 版本：node -p 'process.versions.node.split(".")[0]'
3. 如果 npm 不存在 或 node 主版本 < 22:
   a. 如果 nvm 不存在:
      - curl 下载 nvm install.sh (失败 → 回退到 gitee 镜像)
      - sha256sum 校验（agentNVMInstallSHA256）
      - bash 运行 install.sh
   b. nvm install 22 + nvm use 22
4. 配置 npm_config_prefix=$HOME/.local（让 npm -g 装到这里）
5. export PATH 加上 $npm_config_prefix/bin
```

亮点细节：

**(1) 双镜像回退**（node_install.go:56）：
```go
"    curl ... -fsSL " + shellQuote(agentNVMInstallURL) + ` -o "$nvm_install_script" || \
     curl ... -fsSL " + shellQuote(agentNVMFallbackURL) + ` -o "$nvm_install_script"`,
```

主源 `raw.githubusercontent.com`（国外），失败自动回退到 `gitee.com/mirrors/nvm`（国内镜像）。**中国大陆用户的网络友好性**直接拉满。

**(2) SHA256 校验**（node_install.go:20, 57-58）：
```go
agentNVMInstallSHA256 = "a909fdd01765379ebc5983674adafb8bc9de6d928bfa188761309d4a0c36be0f"
// ...
"    if [ \"$nvm_install_sha256\" != " + shellQuote(agentNVMInstallSHA256) + " ]; then echo \"nvm install checksum mismatch\" >&2; rm -f \"$nvm_install_script\"; exit 1; fi",
```

防止镜像被劫持/中间人攻击。**同时支持 sha256sum 和 shasum**（macOS 没 sha256sum），跨平台。

**(3) 三处可配置镜像**（node_install.go:21-34）：
- `NVM_SOURCE`：nvm 自更新源
- `NVM_NODEJS_ORG_MIRROR`：node 二进制下载源（可换 aliyun 镜像）
- `npm_config_registry`：npm 包源（可换 npmmirror）

**注释里专门提示"mainland China"用 aliyun 镜像**——这是真正的国际化工程经验。

### 7.3 session_lookup：独立 deadline + WithoutCancel

源码：`internal/agent/session_lookup.go`

**故事**：Agent 跑完后，我们要去 `~/.claude/projects/.../*.jsonl` 找它生成的 session 文件来解析 transcript。但这一步**发生在 agent 进程结束之后**——如果 agent 是因为超时被 kill 的，**当时的 ctx 已经被取消了**！

如果继续用这个 ctx 去 `rt.Exec`（找文件）、`rt.DownloadFile`（下载文件），**所有调用立刻失败**——而且错误信息是误导性的"command exited with code -1"，掩盖了真正的超时原因。

**解法**（session_lookup.go:18-26）：

```go
const sessionCleanupTimeout = 30 * time.Second

func sessionCleanupContext(ctx context.Context) (context.Context, context.CancelFunc) {
    return context.WithTimeout(context.WithoutCancel(ctx), sessionCleanupTimeout)
}
```

**`context.WithoutCancel`**（Go 1.21+）是个**关键 API**——它返回一个**继承 ctx 所有 value（trace ID 等）但不继承取消信号**的新 ctx。再加 30 秒独立 timeout，保证：
- ✅ 清理工作能跑完（不会被原 ctx 取消）。
- ✅ trace ID 能继续串联（OTel 链路不丢）。
- ✅ 不会无限挂起（独立 30 秒 deadline）。

注释（session_lookup.go:11-17）讲得很到位：

> those steps execute after the agent process has already returned (or been killed)
> and must not inherit a canceled run context — otherwise every Exec / DownloadFile
> call fires against a dead ctx and logs misleading "command exited with code -1" noise
> that hides the real timeout.

**`withDownloadedSession`**（session_lookup.go:37-66）封了"下载 → 注册 → 解析"的通用流水线，**还做了 panic safety**（session_lookup.go:58-63）：

```go
defer func() {
    if r := recover(); r != nil {
        cleanup()
        panic(r)    // 重抛让 caller 看到
    }
}()
apply(artifactPath)
```

如果 `apply`（具体引擎的解析回调）panic 了，临时文件还是要清理（否则泄漏），但 panic 不能吞掉——这种"**先清理再重抛**"的写法是 Go 异常处理的细节。

**`findAgentSessionJSONL` + `buildSessionLookupScript`**（session_lookup.go:89-149）用 shell 脚本在 runtime 内查文件：

```bash
home=$(printenv HOME); [ -n "$home" ] || exit 0
root="$home/.claude/projects/$SKILL_UP_CLAUDE_WSKEY"; [ -d "$root" ] || exit 0
tmp=$(mktemp) || exit 0; trap 'rm -f "$tmp"' 0
find "$root" -type f -name "*.jsonl" 2>/dev/null >"$tmp" || true
best=; ts=-1
while IFS= read -r p || [ -n "$p" ]; do
  m=$(stat -c %Y "$p" 2>/dev/null || stat -f %m "$p" 2>/dev/null) || continue
  if [ "$ts" -eq -1 ] || [ "$m" -gt "$ts" ]; then ts=$m; best=$p; fi
done <"$tmp"
printf %s "$best"
```

亮点：
- **HOME 和文件树只在 runtime 内读**（用 `rt.Exec`），不通过 host 的 `os.Getenv`——保持 runtime 隔离。
- **`stat -c %Y`（Linux）vs `stat -f %m`（macOS/BSD）双兼容**。
- **找最新修改时间的 jsonl**（不一定是最新创建的）——因为 resume 会修改老 session 文件。
- **`workspaceKeyForRuntime`**（session_lookup.go:111-117）：把 workspace 路径里的 `/` 替换成 `-`，就是 Claude/Qoder 用的"项目子目录名"约定。

---

## 8. 关键设计点（带 Java 类比）

### 8.1 三件套组合：适配器 + 工厂 + 模板方法

整个 `internal/agent/` 包就是这 3 个 GoF 模式的组合落地：

| 模式 | 落点 | 解决什么 |
|------|------|----------|
| **适配器** | `Agent` 接口 + 4 个具体实现 | 把"各家不一样的 CLI"适配成统一接口 |
| **工厂** | `DetectAgent` + `agentkind` 别名 | 集中创建逻辑，消除 caller 的 switch |
| **模板方法** | `BaseAgent` → `CLIAgent` → 具体引擎 | 共享流程骨架（probe→install→exec），各步可重写 |

**Java 类比（重点记忆）**：

```
DataSource (接口)                ←→  Agent
  ├─ DriverManager.getConnection ←→  DetectAgent
  ├─ JdbcTemplate (模板方法)      ←→  CLIAgent
  │   ├─ MySQL Driver            ←→  ClaudeCodeAgent
  │   ├─ PostgreSQL Driver       ←→  CodexAgent
  │   ├─ Oracle Driver           ←→  QoderCLIAgent
  │   └─ SQLServer Driver        ←→  QwenCodeAgent
  └─ 自定义 NoSQL DataSource      ←→  CustomAgent（不嵌 JdbcTemplate）
```

**面试话术**：
> "skill-up 的 agent 包是 GoF 三个经典模式的活样板——**适配器**把 4 家 Coding Agent CLI 统一成 7 方法的 `Agent` 接口；**工厂** `DetectAgent` 集中路由并消除别名；**模板方法** `CLIAgent` 给 4 个 CLI 实现共享 probe→install→exec 流程骨架。这套设计让我加一个新引擎只要 80 行代码（参考 qwen_code.go），不动任何已有逻辑。"

### 8.2 组合优于继承：CustomAgent 的反例

CustomAgent **故意不嵌 CLIAgent**，是个**反直觉但正确**的决策：
- 直觉："既然有 CLIAgent 模板，所有 agent 都嵌它不就行了？"
- 反例："Custom Engine 走 HTTP，根本没有'安装 CLI'这个概念，嵌了 CLIAgent 反而会被未来的 InstallMCP 默认实现污染。"

**Java 类比**：Effective Java Item 18 "Favor composition over inheritance"。Go 没有继承，但嵌入也会带来类似问题——**嵌入的方法会成为类型公开 API 的一部分**，未来基类加方法就是子类 API 的破坏性变更。CustomAgent 用组合（只嵌 BaseAgent）+ 显式实现每个方法，**控制接口表面**。

### 8.3 扩展点 design：怎么加第 5 个引擎

这是一个**面试必问**的设计题。源码给出的答案：

1. **加常量**：在 `agentkind/agentkind.go` 加 `MyEngine = "my_engine"` + 别名。
2. **加文件**：复制 `qwen_code.go` 改 5 处常量（包名/类名/CheckCmd/SkillPath/安装命令）。
3. **加工厂分支**：`factory.go` 的 switch 加 `case agentkind.MyEngine: return NewMyEngineAgent(cfg), nil`。
4. **（可选）加 SessionResumer**：如果支持多轮，实现 `RunTurn`，加 compile-time 断言 `_ SessionResumer = (*MyEngineAgent)(nil)`。

**就这 3 步，不动 evaluator、不动 reporter、不动 runtime**——这是开闭原则的硬指标。Java 同学应该能立刻联想到"加一个 Spring `Repository` 实现不用改 service"的体验。

### 8.4 编译期断言：用类型系统锁契约

`agent.go:151-155` 的 `var _ Interface = (*Type)(nil)` 是 Go 静态检查的精髓：
- 不占任何运行时开销（赋值给 `_`）。
- 编译器强制右侧类型实现接口，否则编译失败。
- 等价于 Java 的 `class Foo implements Bar` 显式声明，但更灵活（可以**事后**给已有类型加接口断言）。

skill-up 用它锁死了"哪几个引擎实现了 SessionResumer"，**review 时一眼能看出哪些引擎支持多轮**。

### 8.5 错误类型化：可识别的业务错误

`UnsupportedAgentError`（errors.go:7）是导出类型，上游可以 `errors.As` 精准识别。这比 `fmt.Errorf("unsupported agent: %s", name)` 强得多——后者只能字符串匹配，脆弱。

**Java 类比**：相当于 throw 一个自定义异常类型（`UnsupportedAgentException`），而不是 throw `new RuntimeException("unsupported agent ...")`。catch 端能精确处理，不用 parse 异常消息。

---

## 9. 面试 Q&A

### Q1：为什么用接口而不是基类继承？Go 的设计风格有什么取舍？

**A**：Go 没有 Java 那种"类 + 继承 + 多态"三位一体的类型系统——它**类型和接口是解耦的**。`Agent` 是接口，任何结构体只要实现 7 个方法就自动满足（duck typing）。这带来 3 个好处：
1. **解耦**：evaluator 只依赖接口，不依赖具体类，单测可以 mock。
2. **多实现零冲突**：4 个引擎互不影响，加一个不改其他。
3. **显式契约**：接口就是个"合同"，看一眼就知道 evaluator 期望什么。

Java 里要达到同样效果，得用 `interface` + 抽象类 + 依赖注入——Go 一个 `type Agent interface` 就搞定了，更轻。

### Q2：CLIAgent 和 BaseAgent 为什么要分两层？

**A**：因为它们承载**不同粒度的复用**：
- `BaseAgent`：**所有 agent**（含 CustomAgent）都要用的基础设施——Name / Config / 凭据 / env 合并 / skill 安装。
- `CLIAgent`：**只有"通过 CLI 跑"的 agent** 才需要的模板方法——InstallMCP / Run 模板等。

CustomAgent **不走 CLI**（可能走 HTTP），所以它只嵌 BaseAgent 不嵌 CLIAgent。如果不分层，CustomAgent 就会被迫继承一堆 CLI 语义，违反单一职责。**Java 类比**：`AbstractDataSource`（最通用）vs `AbstractJdbcTemplate`（只 JDBC 场景）——非 JDBC 的自定义 DataSource（比如 MongoDB Driver）只嵌前者。

### Q3：怎么处理"prompt 超过 ARG_MAX"这种边界场景？

**A**：用 `prompt_delivery.go` 的**双模式自适应**：
1. 设阈值（默认 32KB）。
2. prompt 短 → 内联进命令行（`agent -p "<prompt>"`）。
3. prompt 长 → 写到 runtime workspace 的临时文件，用 `cat file | agent -p` 管道喂入。
4. 决策由 `deliverPrompt` 统一做，调用方只提供两个 builder（Inline / StdinFile），**不感知模式选择**。

阈值可以通过 `SKILL_UP_PROMPT_INLINE_MAX_BYTES` env 调整，运维友好。返回的 `PromptDeliveryMetadata` 记进 `SessionResult`，让报告能展示每个 case 用了哪种模式——**可观测性是底线**。

### Q4：怎么解析 4 家完全不同的 session 文件？token 怎么算？

**A**：每家一个 parser，但都翻译成统一的 `transcript.Transcript`：
- claude/qoder 共享 `parseSessionFile`（schema 相同）。
- codex 有 stdout 解析器 + session 文件解析器，**两阶段关联**（用 threadID 找文件，只有更长时才覆盖）。
- qwen 用 Gemini schema 的 parser。

**Token 用 max 而不是 sum**——因为 session 文件里的 usage 是**累计快照**而非每行 delta（Claude 的 cache_read_input_tokens 经常多行重复）。这是个只有真踩过坑才知道的细节，源码注释专门讲了。

### Q5：为什么 session 文件解析要 `context.WithoutCancel`？

**A**：session 文件查找/下载发生在 agent 进程**结束后**。如果 agent 因超时被 kill，原 ctx 已取消，直接用它会让所有 cleanup 调用立刻失败，报误导性的"exit code -1"，掩盖真正的超时。

`context.WithoutCancel(ctx)` 继承 value（trace ID）但不继承取消信号，再套 30 秒独立 timeout，保证 cleanup 既能跑完又不挂死。这种"**主流程超时，cleanup 流程独立**"的设计，是高并发系统的常见模式（Java 类比：try-with-resources 里的 finally 不能被主流程的 InterruptedException 跳过）。

---

## 10. 一句话总结

> **`internal/agent/` 是 skill-up 的"多引擎适配层"——用「Agent 接口 + DetectAgent 工厂 + BaseAgent/CLIAgent 模板方法 + 4 引擎具体实现 + CustomAgent 组合」这套结构，把"4 家完全不同的 Coding Agent CLI + 用户自定义 HTTP 引擎"统一成一个稳定的 7 方法接口，让 evaluator 永远不感知底下是谁。核心难点不在接口设计本身（那就是个适配器模式），而在于把 4 家完全不同的输出格式（stream-json / NDJSON / Gemini jsonl）归一成同一份 `transcript.Transcript`，以及处理 prompt 长度、node 引导、跨平台 shell、cleanup 独立 deadline 这些"脏活累活"。**

---

## 附：核心源码文件速查

| 文件 | 行数 | 核心内容 |
|------|------|----------|
| `internal/agent/agent.go` | ~810 | `Agent` 接口、`BaseAgent`、`Config`、`SessionResult`、compile-time 断言 |
| `internal/agent/factory.go` | ~87 | `DetectAgent` / `DetectAgentWithInitParams` / "auto" 剥离 |
| `internal/agent/cli.go` | ~207 | `CLIAgent` 模板方法（InstallMCP/InstallSkill/Run/Check） |
| `internal/agent/claude_code.go` | ~1117 | ClaudeCode 引擎 + 双写 AUTH_TOKEN + parseSessionFile |
| `internal/agent/codex.go` | ~1137 | Codex 引擎 + 合成 provider + MCP 桥接 + 双解析器 |
| `internal/agent/qodercli.go` | ~330 | Qoder 引擎 + 模型档位 + 自包含二进制 |
| `internal/agent/qwen_code.go` | ~439 | Qwen 引擎 + Gemini schema 翻译 |
| `internal/agent/custom.go` | ~1281 | CustomAgent + local/http 双 transport + 安全（maskAPIKey/workspacePath） |
| `internal/agent/errors.go` | ~13 | `UnsupportedAgentError` 类型化错误 |
| `internal/agent/kwargs.go` | ~63 | 引擎 kwargs（如 `bypass_sandbox`）+ 未知 kwargs 警告 |
| `internal/agent/prompt_delivery.go` | ~93 | 超 32KB prompt 双模式（inline/file） |
| `internal/agent/node_install.go` | ~98 | nvm/node 引导 + SHA256 + 镜像回退 |
| `internal/agent/session_lookup.go` | ~149 | session JSONL 查找 + `WithoutCancel` 独立 deadline |
| `internal/agent/skill.go` | ~67 | skill 目录同步（排除 `evals/`） |
| `internal/agent/mcp.go` | ~181 | claude-compatible MCP 安装命令 + 环境变量安全引用 |
| `internal/agentkind/agentkind.go` | ~42 | 引擎名常量 + 别名（叶子包，避免循环依赖） |
