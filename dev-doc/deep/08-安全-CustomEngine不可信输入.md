# 08 · 安全工程 —— Custom Engine 不可信输入全链路防御

> 这一篇是 skill-up 安全工程的**源码级**深挖：Custom Engine 作为「用户自定义、可返回任意 JSON / 任意文件」的外部组件，是整个项目最大的**不可信边界**。本篇逐行讲清这条边界上的每一道闸门 —— TOCTOU 防御、凭据脱敏、SSRF 防护、内存上限、配置侧防走私。
>
> 面向：把「我做过安全工程」写进简历、面试能下钻到「TOCTOU 窗口怎么关」「凭据为什么不泄漏到 ps」这一层的工程师。

---

## 1. 一句话定位

**Custom Engine = 用户用 YAML 声明的一个外部 Agent（一条 shell 命令 / 一个 HTTP 服务），skill-up 把 case 输入交给它、把它吐回的 JSON/文件当作 `SessionResult` 去打分。** 它在数据流上处于**完全不可信**的位置 —— 它返回的每一个字段、每一个文件路径、每一个 URL 都可能是有意或无意构造出来的攻击向量。

为什么不可信？看它返回的结构（`internal/agent/custom.go:480-492`）：

```go
type parsedSessionResult struct {
    Engine       string                `json:"engine"`
    Model        string                `json:"model"`
    ExitCode     *int                  `json:"exit_code"`
    FinalMessage string                `json:"final_message"`
    Stderr       string                `json:"stderr"`
    Transcript   transcript.Transcript `json:"transcript"`
    Artifacts    *SessionArtifacts     `json:"artifacts"`
    // ...
}
```

每一行都是**攻击面**：

- `engine` / `model` —— 可能 echo 回配置的 API key（skill-up 会把它写进 `result.json` 和 OTel span）；
- `final_message` / `stderr` / `transcript` —— 同上，且会**流给 Judge LLM** 和 CI 日志；
- `artifacts.files[].path` —— 可能是 `/etc/passwd` 或 `../../../.ssh/id_rsa`，让 skill-up 把宿主机文件拷进报告目录（**路径穿越**）；
- `artifacts.files[].url` —— 可能指向 `http://169.254.169.254/...`（云元数据服务，**SSRF**）或一个会 302 跳到内网的入口；
- 一个巨大的 inline `content_base64` 或一个无限的 HTTP 响应体 —— **内存耗尽**（OOM）。

而本地传输（`transport: local`）更凶：skill-up 要按它声明的 `command` 在宿主机 / 容器里起进程，命令行如果包含 API key，会**直接出现在 `ps`、exec span、shell 历史**里。

> **Java 类比**：这就像你写了一个接收第三方回调的 Webhook 服务，对方的 JSON 体里任何一个字段都可能带攻击载荷。差别是，Webhook 还能躲在 HTTPS 后面，Custom Engine 是**本地起进程 + 读文件 + 发 HTTP**，攻击面是三维的。

---

## 2. 威胁模型

把上面散落的攻击面整理成 6 类威胁，每类对应后面的防御章节：

| # | 威胁 | 攻击场景 | 受害资产 | 防御章节 |
|---|------|----------|----------|----------|
| T1 | **路径穿越 / 符号链接 TOCTOU** | engine 返回 `artifacts.files[].path: "/etc/passwd"` 或在 workspace 里种一个指向 `~/.ssh` 的符号链接，skill-up 的报告归档器跟着把宿主文件拷出来 | 宿主机敏感文件外泄 | §3.1 `workspacePath` |
| T2 | **stale fixture 误读** | 上次跑残留的 `outputs/result.json` 被当成这次的成功结果，让失败的 case 骗过评测 | 评测结果失真 | §3.2 `clearStaleOutputFile` |
| T3 | **凭据泄漏到日志/报告/Judge** | engine 把 `$ANTHROPIC_API_KEY` 原样 echo 进 `final_message`，流向 Judge LLM 和 CI 日志 | API key 泄漏 | §3.3 `maskAPIKey` |
| T4 | **凭据进命令行 / URL** | 用户把 `${api_key}` 写进 `local.command` 或 `http.url`，进程一启动 key 就出现在 `ps` / request log / proxy 日志 | API key 泄漏 | §3.4 `containsAPIKey` / `errSecretInCommand` / `errSecretInURL` |
| T5 | **SSRF（artifact 重定向到内网）** | engine 返回 `url: http://internal-metadata/latest/credentials`，下载器跟随 302 跳到云元数据服务 | 内网探测 / 凭据窃取 | §3.5 不跟随 3xx |
| T6 | **内存耗尽** | engine 返回 800MB 的 inline `content_base64`，或 HTTP 服务回一个无上限的流，skill-up `io.ReadAll` 直接 OOM | DoS | §3.6 size cap |

外加两条**配置侧**的预防性威胁（不是 engine 返回值，而是**用户写错配置**把自己 leak 了）：

| # | 威胁 | 场景 | 防御 |
|---|------|------|------|
| T7 | 凭证写进会被日志暴露的字段 | 用户把 `OPENAI_API_KEY=sk-real-key` 直接写进 `args: ["--token", "sk-real-key"]` | §4 `resolveEnvRefsStrict` |
| T8 | 间接走私 | 用 `${WRAPPER}` 包装一个敏感名，绕过单次检查 | §4 `resolveEnvRefsStrict` 迭代展开 |

---

## 3. 逐条防御讲解（每条贴真实代码 + file:line + 攻击场景 → 防御）

### 3.1 `workspacePath`：路径穿越 + 符号链接 TOCTOU（T1）

这是整篇文档最硬核的一段，因为 TOCTOU（Time-of-Check to Time-of-Use）是少数**只能靠「在 use 时刻重新 check」才能堵住**的漏洞类别 —— 一次检查不够。

#### 攻击场景

1. **静态穿越**：engine 返回 `path: "../../../etc/passwd"`，归档器 `filepath.Join(workspace, path)` 后拼出一个 workspace 外的路径，把 `/etc/passwd` 拷进报告目录。
2. **符号链接外指**：engine 在 workspace 里 `ln -s /root/.ssh id_rsa`，然后返回 `path: "id_rsa"`。`filepath.Clean` 看不出问题（路径就是 `id_rsa`），但 `read` 会跟随链接读到 `/root/.ssh/id_rsa`。
3. **TOCTOU**：即使你在 use 之前 `EvalSymlinks` 检查过 `outputs/newdir/result.json` 是干净的，engine 在你**检查完到真正 `rm` / `read` 之间**把 `outputs/newdir` 创建成指向 `/tmp` 的符号链接，你的 use 就跟随了被种的链接 —— 这就是经典的 check-then-use 竞态。

#### 防御代码（`internal/agent/custom.go:1115-1149`）

```go
// workspacePath resolves a path against the runtime workspace and confines it
// to that workspace. ...
func workspacePath(rt Runtime, p string) (string, error) {
    if p == "" {
        return p, nil
    }
    // (1) 先把 workspace 根本身用 EvalSymlinks 解析 —— 防止 workspace 本身就是一个软链
    ws, err := filepath.EvalSymlinks(filepath.Clean(rt.Workspace()))
    if err != nil {
        // Workspace must exist; if EvalSymlinks fails fall back to the
        // lexical root so we still reject obvious escapes.
        ws = filepath.Clean(rt.Workspace())
    }
    var abs string
    if filepath.IsAbs(p) {
        abs = filepath.Clean(p)
    } else {
        abs = filepath.Clean(filepath.Join(ws, p))   // (2) 相对路径 join 到 workspace
    }
    // (3) 解析「最深的存在祖先」的符号链接，把中间目录里藏的软链也展平
    resolved := resolveExistingPrefix(abs)
    rel, err := filepath.Rel(ws, resolved)
    if err != nil || rel == ".." || strings.HasPrefix(rel, ".."+string(filepath.Separator)) {
        return "", fmt.Errorf("path %q escapes the runtime workspace", p)
    }
    // (4) 关键：返回 resolved-prefix 路径，而不是 lexical abs
    return resolved, nil
}
```

四个动作逐条拆：

**(1) `EvalSymlinks` workspace 根**：`rt.Workspace()` 返回的可能是 `/var/ws`，但 `/var/ws` 本身可能是个软链指向 `/host/home attacker`。先把根展平，后续 `Rel` 比较才有效。

**(2) 相对路径 join**：把 engine 给的相对路径钉死在 workspace 下；绝对路径走 `(3)` 的祖先解析，最后 `Rel` 比较仍然要求落在 ws 内。

**(3) `resolveExistingPrefix` —— 解析「最深的存在祖先」**（`custom.go:1156-1173`）：

```go
func resolveExistingPrefix(abs string) string {
    suffix := ""
    cur := abs
    for {
        if resolvedAncestor, err := filepath.EvalSymlinks(cur); err == nil {
            if suffix == "" {
                return resolvedAncestor
            }
            return filepath.Join(resolvedAncestor, suffix)
        }
        parent := filepath.Dir(cur)
        if parent == cur {
            return abs
        }
        suffix = filepath.Join(filepath.Base(cur), suffix)
        cur = parent
    }
}
```

**为什么要走这一步？** 因为 output file 在 engine 跑之前**还不存在**（`outputs/newdir/result.json` 整条路径可能一个文件都没有），`EvalSymlinks` 直接对它调用会失败。但**中间目录**（`outputs/newdir`）可能已经被 engine 种成了软链。所以这个函数**从叶子向上走**，找到第一个存在的祖先，对祖先 `EvalSymlinks`，再把不存在的尾巴接回去。

**攻击场景对照**：engine 在 workspace 里建了 `outputs/evil -> /etc`，然后让 output_file 是 `outputs/evil/passwd`。`resolveExistingPrefix` 会找到 `outputs/evil`（它存在），`EvalSymlinks` 把它解析成 `/etc`，最后 `filepath.Rel("/var/ws", "/etc/passwd")` 返回 `../etc/passwd`，触发 `(4)` 的拒绝。

**(4) 返回 resolved 而不是 lexical abs**：这一步是**关 TOCTOU 窗口的关键**。注释（`custom.go:1140-1148`）原话：

> Returning the lexical `abs` instead would re-introduce a TOCTOU window: the custom command could create a not-yet-existing parent as a symlink after this check, and a later `DownloadFile` / `rm` against `abs` would follow it.

换句话说：如果你返回 lexical abs，调用方拿着这个字符串去做 `rm` / `read` 时，engine 可能在这中间又种了链接 —— 你这一次的检查白做了。返回**已经把祖先软链展平**的路径，调用方 follow 时走的是已经被验证过的物理位置。

#### 真正关 TOCTOU 的是「每次 use 前重新解析」

光有 `workspacePath` 还不够 —— 它只是「某次调用时刻的 snapshot」。TOCTOU 的核心是**两次调用之间的时间差**。skill-up 的做法是：**在每一个真正会触碰文件系统的 use site 都重新调用 `workspacePath`**，而不是缓存第一次的结果。三个 use site：

1. **`clearStaleOutputFile`**（`custom.go:258-279`）—— 清理上次残留输出文件前：
   ```go
   func (a *CustomAgent) clearStaleOutputFile(ctx context.Context, rt Runtime, outputFile string) bool {
       // Re-resolve symlinks now: resolveCustomIOFiles validated the path
       // pre-run, but a not-yet-existing parent (e.g. outputs/new/result.json)
       // could be planted as an outward symlink by a concurrent case or any
       // other process sharing the workspace between then and now.
       safe, err := workspacePath(rt, outputFile)
       if err != nil {
           logging.WarnContextf(ctx, "CustomAgent: refusing to clear output file %s before run: %v", outputFile, err)
           return false
       }
       outputFile = safe
       q := shellQuote(outputFile)
       cmd := "if [ -e " + q + " ]; then rm -f -- " + q + " && printf %s " + shellQuote(customStaleClearedMarker) + "; fi"
       // ...
   }
   ```
   注意注释明确写「`resolveCustomIOFiles` 在 run 前验证过，但一个尚未存在的父目录可能**在那时到现在之间**被并发 case 或任何共享 workspace 的进程种成外向软链」。**这次的 re-resolve 才是真正堵窗口的那一刀。**

2. **`readRawResult`**（`custom.go:433-446`）—— engine 跑完读结果文件前：
   ```go
   func (a *CustomAgent) readRawResult(ctx context.Context, rt Runtime, custom *customengine.Config, result ExecResult, outputFile string) (raw string, produced bool) {
       // Re-resolve symlinks now that the engine has run. workspacePath was
       // called pre-run, but the engine could have created a not-yet-existing
       // parent (e.g. outputs/newdir) as a symlink pointing outside the
       // workspace between then and now.
       if outputFile != "" {
           safe, err := workspacePath(rt, outputFile)
           if err != nil {
               logging.WarnContextf(ctx, "CustomAgent: refusing to read output file %s after run: %v", outputFile, err)
               return strings.TrimSpace(result.Stdout), false
           }
           outputFile = safe
       }
       // ...
   }
   ```
   这次是**engine 已经跑过之后**的 re-resolve —— engine 在运行中种下的链接，这里会被看见。**这才是真正的 post-engine TOCTOU 关闭点。**

3. **`archiveRenamedPathArtifact`**（`custom.go:763-782`）—— 归档 path 类 artifact 前：
   ```go
   // f.Path has already passed workspacePath in collectArtifacts, but we
   // re-check here so a follow-up symlink racing the DownloadFile cannot
   // follow a planted parent symlink out of the workspace.
   safePath, err := workspacePath(rt, f.Path)
   ```

**核心设计**：**每一次 `DownloadFile` / `rm` 之前都重新解析**。即使攻击者在两次解析之间种链接，下一次解析也会发现并拒绝。窗口被压缩到「单次解析 + 单次 IO」之间，而 Go 的 `EvalSymlinks` + 紧接着的系统调用在这个尺度上竞态成功的概率已经低到可以忽略（而且 `resolveExistingPrefix` 返回的是已展平路径，follow 时走的是物理路径，连这层窗口都消掉了）。

> **Java 类比**：Java 里写文件前用 `Path.normalize()` / `toRealPath()` 防穿越是常见做法，但大多数教程只检查一次。skill-up 这套「每个 use site 都 re-resolve」相当于在每一次 `Files.copy` 前都重新 `toRealPath()` —— 这才是堵 TOCTOU 的正确姿势。面试可以讲：「TOCTOU 不是靠更严格的检查堵的，是靠**把 check 挪到离 use 越近越好**堵的，理想是 check 和 use 之间没有任何 attacker-controllable 的 IO 间隔。」

#### 配套：`filterWorkspacePaths`（`custom.go:743-757`）

engine 返回的 `artifacts.generated_files` 是个 `[]string`，每一个都要过 `workspacePath`：

```go
func (a *CustomAgent) filterWorkspacePaths(ctx context.Context, rt Runtime, paths []string, field string) []string {
    if len(paths) == 0 {
        return paths
    }
    out := paths[:0]
    for _, p := range paths {
        safe, err := workspacePath(rt, p)
        if err != nil {
            logging.WarnContextf(ctx, "CustomAgent: dropping %s entry %q: %v", field, p, err)
            continue
        }
        out = append(out, safe)
    }
    return out
}
```

**注释里直接点名 `/etc/passwd` 攻击**（`custom.go:706-710`）：

> Drop any engine-supplied generated_files paths that escape the workspace. On the none runtime an absolute path is the host path, so an untrusted custom engine could return /etc/passwd here to have the evaluator copy host files into the report.

**攻击场景**：`none` runtime 下，engine 返回 `generated_files: ["/etc/passwd", "/home/user/.aws/credentials"]`，没有这道闸，归档器会老老实实把这两个文件拷进报告目录，评测报告里就**直接能看到宿主密钥**。

---

### 3.2 `clearStaleOutputFile`：防 stale fixture 误读（T2）

#### 攻击场景

上次 case 跑完留下了一个 `outputs/result.json`（exit_code=0，答案也对）。这次 case 跑，engine 因为配置错误根本没启动 / 启动就崩了 / 输出写到别的地方，`outputs/result.json` **原地不动**。如果 skill-up 直接读这个文件，就会把**上次的结果当成这次的成功**，评测完全失真 —— 失败 case 骗过 pass rate。

#### 防御（`custom.go:254-279` + 调用点 `custom_local.go:53-56`）

```go
// localTransport.run (custom_local.go:53-56)
clearedStaleOutput := false
if custom.Local.OutputFile != "" && filepath.Clean(outputFile) != filepath.Clean(inputFile) {
    clearedStaleOutput = a.clearStaleOutputFile(ctx, rt, outputFile)
}
```

两个**前置条件**很关键：

1. **`custom.Local.OutputFile != ""`**：只有**用户显式配置了** output_file 才清理。默认的 `${output_file}` 路径（`outputs/session-result.json`）不动 —— 它可能是用户作为 fixture 输入放进来的合法文件（「这个文件是我 case 的输入，engine 会读它」）。
2. **`outputFile != inputFile`**：当 output 和 input 是同一个文件时**不清理** —— 否则会把刚刚写好的 SessionInput 自己删掉。

`clearStaleOutputFile` 的实现用了一个 marker 技巧（`custom.go:271-278`）：

```go
q := shellQuote(outputFile)
cmd := "if [ -e " + q + " ]; then rm -f -- " + q + " && printf %s " + shellQuote(customStaleClearedMarker) + "; fi"
result, err := rt.Exec(ctx, cmd, ExecOptions{})
// ...
return strings.Contains(result.Stdout, customStaleClearedMarker)
```

返回 bool 表示**是否真的删过**，因为后面（`custom_local.go:94-101`）的「framework-written files」记录要依赖它：

```go
usedOutputFile := outputFileProduced || clearedStaleOutput
frameworkFiles := make([]string, 0, 2)
if inputFile != "" {
    frameworkFiles = append(frameworkFiles, inputFile)
}
if usedOutputFile && outputFile != "" {
    frameworkFiles = append(frameworkFiles, outputFile)
}
```

这个 `frameworkFiles` 的用途是告诉 workspace-diff 收集器「这些文件是框架写的，别算成用户的产出」。**「produced or cleared」规则**保证：即使文件被清空了，它也被记成 framework 文件 —— 否则 diff 收集器看到一个被清空的文件，会当成「用户删了某个文件」记进 diff，污染评测。

> **Java 类比**：这就像 JUnit 跑测前 `@BeforeEach` 里清空 `target/test-out/`，避免上次跑的产物干扰这次断言。多了一层「记录哪些是框架自己写的」是为了让 diff 类断言（「这次跑新增了哪些文件」）不被框架的中间产物污染。

---

### 3.3 `maskAPIKey`：全字段脱敏（T3）

#### 攻击场景

engine 进程的 env 里有 `$ANTHROPIC_API_KEY=sk-ant-xxx`（这是 skill-up 注入的）。engine 的工作里可能**完全无意**地把它 echo 回来 —— 比如它跑一个 wrapper 脚本，wrapper 在 debug 模式下打印环境变量；或者它用 `ps` 看到了父进程的 env；又或者 LLM 输出里恰好把 key 复述了一遍。这个字符串一旦进入 `final_message` / `stderr` / `transcript`，就会：

1. 写进 `result.json`（落盘）；
2. 发给 Judge LLM（**第三个组织的 API**，可能记录输入）；
3. 进 OTel span 属性（可观测性后端）；
4. 进 CI 日志（GitHub Actions 日志公开可见）。

#### 防御代码（`custom.go:942-962`）

```go
func (a *CustomAgent) maskAPIKey(s string) string {
    if s == "" {
        return s
    }
    const redacted = "***REDACTED***"
    const minMaskLen = 8
    // (1) 配置的模型层 API key
    if a.Cfg.APIKey != "" && strings.Contains(s, a.Cfg.APIKey) {
        s = strings.ReplaceAll(s, a.Cfg.APIKey, redacted)
    }
    // (2) 所有 engine.custom.env 的 value
    if a.Cfg.Custom != nil {
        for _, v := range a.Cfg.Custom.Env {
            if len(v) < minMaskLen {
                continue
            }
            if strings.Contains(s, v) {
                s = strings.ReplaceAll(s, v, redacted)
            }
        }
    }
    return s
}
```

两个脱敏源：

1. **`a.Cfg.APIKey`**：模型层凭据。**即使 custom engine 从来没拿到它**（model 层和 engine.custom 是分开的），也照脱 —— 注释说「防止一个 inspect 父进程 env 的 wrapper 把它 echo 出来」。这是**纵深防御**（defense in depth）：不假设 engine 看不到，而是假设它可能看到，所以无论看到没看到，输出里都不能有。
2. **`a.Cfg.Custom.Env` 的每个 value**：用户在 `engine.custom.env` 里声明的所有 env 变量值。这是用户**显式**的 secret channel（`MY_TOKEN: ${MY_TOKEN}`），engine 拿到后如果 echo 回 `$MY_TOKEN` 的值，必须脱掉。

**`minMaskLen = 8` 的精妙**：注释解释「短于 8 字符的 value 跳过，避免一个非敏感的 flag 或 model 名把输出里所有短词都变成 `***REDACTED***`」。比如 env 里有个 `VERBOSE=1`，不跳过的话，输出里每个 `1` 都会被替换成 `***REDACTED***` —— 既要防泄漏又要保证可用性。

#### 全字段覆盖的调用点

`maskAPIKey` 不是只用在某个字段，而是**每一个流向外部的不可信字符串**都过一遍。`assembleResult` 和 `parseSessionResult` 里密集出现：

```go
// custom.go:355-359 (assembleResult, execErr 路径)
if res.Stderr == "" {
    res.Stderr = a.maskAPIKey(o.stderr)
} else {
    res.Stderr = a.maskAPIKey(res.Stderr)
}

// custom.go:376 (非零退出)
return res, fmt.Errorf("custom engine command exited %d: %s", o.exitCode, a.maskAPIKey(o.stderr))

// custom.go:400-409 (text 格式的 final message)
finalMsg := a.maskAPIKey(strings.TrimSpace(stdout))
return &SessionResult{
    // ...
    FinalMessage: finalMsg,
    Stderr:       a.maskAPIKey(stderr),
    Transcript:   minimalCustomTranscript(messages, finalMsg),
    // ...
}

// custom.go:520-557 (session_result 格式)
finalMsg := a.maskAPIKey(parsed.FinalMessage)
trans := a.maskTranscript(parsed.Transcript)
// ...
res := &SessionResult{
    Engine:       firstNonEmpty(a.maskAPIKey(parsed.Engine), a.Name()),
    Model:        firstNonEmpty(a.maskAPIKey(parsed.Model), formatAgentModel(...)),
    // ...
    FinalMessage: finalMsg,
    Stderr:       a.maskAPIKey(parsed.Stderr),
    Transcript:   trans,
    // ...
}
```

注意 **`Engine` 和 `Model` 也脱敏**（`custom.go:546-548`）—— 注释明确写了为什么：

> Engine and Model are engine-supplied untrusted strings that flow into result.json, judges, and OTel span attributes. Mask them too so a custom engine that echoes a credential into its self-identification fields cannot bypass the masking discipline applied to FinalMessage / Stderr / Transcript content above.

**攻击场景**：malicious engine 返回 `{"engine": "my-wrapper sk-ant-real-key", ...}`，如果不脱 Engine 字段，这个 key 就会写进 `result.json` 的 engine 字段 + OTel span 的 `skill_up.engine` 属性。脱敏后变成 `my-wrapper ***REDACTED***`。

`maskTranscript`（`custom.go:917-927`）单独处理 transcript —— 因为 transcript 是个 `[]Message`，要拷一份逐条 mask Content，**返回新 slice** 避免调用方误用 mask 前的版本：

```go
func (a *CustomAgent) maskTranscript(t transcript.Transcript) transcript.Transcript {
    if len(t) == 0 {
        return t
    }
    out := make(transcript.Transcript, len(t))
    for i, m := range t {
        m.Content = a.maskAPIKey(m.Content)
        out[i] = m
    }
    return out
}
```

#### HTTP transport 错误也脱敏

`custom_http.go:134-144` 里，HTTP 请求失败时 `net/http` 的 error 字符串会**内嵌 request URL**：

```go
resp, err := client.Do(req)
if err != nil {
    // The net/http error embeds the request URL; mask it like stderr so a
    // secret a user mistakenly put in the URL cannot leak into result.Error
    // / reports.
    return &transportOutcome{
        exitCode: 1,
        stderr:   a.maskAPIKey(err.Error()),
        execErr:  errors.New(a.maskAPIKey("custom engine http request failed: " + err.Error())),
    }
}
```

注释直指威胁：「用户不小心把 secret 放进 URL 时，`net/http` 的 error 会带着 URL 一起被打印」。`errors.New` 而不是 `fmt.Errorf("...: %w", err)` —— 因为 mask 后的字符串已经是 payload，没必要保留 wrap 链。

> **Java 类比**：这就像 Spring 里写一个 `@ControllerAdvice` 全局异常处理器，但**主动重写** error message，把任何匹配 credential 模式的子串替换成 `***REDACTED***` 再返回给客户端。差别是 skill-up 是**字段级别、按数据流贴**的，不靠全局拦截 —— 这样即使将来加了新字段，老字段的脱敏也不会失效。

---

### 3.4 凭据不进命令行 / URL（T4）

#### 攻击场景

本地 transport 要起进程。如果用户配置：

```yaml
engine:
  custom:
    transport: local
    local:
      command: "curl https://api.x.com/v1/chat -H 'Authorization: Bearer ${api_key}'"
```

`${api_key}` 被渲染成真实 key 后，整个字符串变成 `curl ... -H 'Authorization: Bearer sk-real-key'`。这个字符串会：

1. 进入 `rt.Exec(ctx, cmd, ...)` 的命令行；
2. 出现在**进程列表** `ps aux`（同主机任何用户都能看）；
3. 进入 exec span 的 `process.command` 属性（OTel 后端）；
4. 进入失败时的 error 日志。

#### 防御 1：`containsAPIKey` + `errSecretInCommand`（`custom.go:908-910, 965-970, 288-300`）

```go
// custom.go:908-910
func (a *CustomAgent) containsAPIKey(s string) bool {
    return a.Cfg.APIKey != "" && strings.Contains(s, a.Cfg.APIKey)
}

// custom.go:965-970
func errSecretInCommand(field string) error {
    return fmt.Errorf(
        "engine.custom.%s renders the API key into the command line, which would expose it in process listings and traces; reference ${api_key} from engine.custom.env instead",
        field,
    )
}
```

在 `buildLocalExec` 里，**渲染后的** command / 每个 arg 都过一遍（`custom.go:284-300`）：

```go
command, err := renderTemplate(local.Command, vars)
if err != nil {
    return "", ExecOptions{}, fmt.Errorf("render local.command: %w", err)
}
if a.containsAPIKey(command) {
    return "", ExecOptions{}, errSecretInCommand("local.command")
}
parts := []string{shellQuote(command)}
for i, raw := range local.Args {
    arg, rErr := renderTemplate(raw, vars)
    if rErr != nil { /* ... */ }
    if a.containsAPIKey(arg) {
        return "", ExecOptions{}, errSecretInCommand(fmt.Sprintf("local.args[%d]", i))
    }
    parts = append(parts, shellQuote(arg))
}
```

**关键设计**：检查的是**渲染后**的字符串，不是模板原文。这样无论是 `${api_key}` 直接展开，还是间接拼出来（比如 kwargs 里塞了 key 再 `${kwargs.token}` 展开），只要最终 command/arg 里出现了 API key 的完整值，就拒绝。

#### 防御 2：`errSecretInURL`（`custom_http.go:51-53, 254-258`）

HTTP transport 同理 —— URL 里的 key 会进 request log / proxy / trace：

```go
// custom_http.go:51-53
if a.containsAPIKey(url) {
    return nil, errSecretInURL()
}

// custom_http.go:254-258
func errSecretInURL() error {
    return errors.New(
        "engine.custom.http.url renders the API key into the URL, which would expose it in request logs and traces; reference ${api_key} from engine.custom.http.headers or request_body instead",
    )
}
```

**正确的 channel** 在 error message 里直接告诉用户：用 `engine.custom.env`（本地）或 `http.headers` / `http.request_body`（HTTP）传 key，这些字段不会进命令行/URL。

#### 防御 3：artifact URL 也查（`custom.go:846-849`）

engine 返回的 `artifacts.files[].url` 在下载前也过一遍：

```go
if a.containsAPIKey(f.URL) {
    logging.WarnContextf(ctx, "CustomAgent: artifact %q url embeds the API key; skipping download", f.Name)
    return ""
}
```

否则这个 key 会被传到 engine 声明的 endpoint，以及它的 request log 和代理里。

> **Java 类比**：这相当于在 `Runtime.exec` 之前做一道 gate，扫描命令行参数里有没有出现 `System.getenv("API_KEY")` 的值。Java 圈类似的实践是「永远不要把 secret 拼进命令行，要用 stdin / env var 传」—— 但很少有项目显式做这种「渲染后扫描」的检查。面试可以讲：「`ps` 是同主机最古老的 secret leakage 渠道，Go 的 `os/exec` 把 args 当 argv 传本来已经避开了 shell 注入，但 args 仍然会出现在 `/proc/<pid>/cmdline`，所以渲染后的字符串本身不能含 key。」

---

### 3.5 artifact URL 不跟随 3xx 重定向（T5 / SSRF）

#### 攻击场景

engine 返回 `artifacts.files[].url: "http://attacker.com/redirect"`，attacker.com 返回 `302 Location: http://169.254.169.254/latest/meta-data/iam/security-credentials/`（AWS 元数据服务，云主机内网地址）。如果下载器跟随重定向，它会把云主机的临时凭据下载下来，作为 artifact 写进报告目录 —— 攻击者读一下评测报告就拿到了 IAM 凭据。

即使不针对云元数据，重定向也能让下载器**绕过 skill-up 在原 URL 上做的所有检查**（scheme 白名单、api-key 检查）。

#### 防御（`custom.go:829-904`）

整个 `downloadURLArtifact` 的安全姿态在注释里写得很清楚：

```go
// custom.go:821-828
// It is best-effort, mirroring the rest of artifact handling: a non-http(s)
// scheme, a transport error, a non-2xx status, or an over-cap body is logged as
// a warning and yields an empty string so the run still succeeds. Only the exact
// URL the engine declared is fetched — SSRF is out of scope by the same posture
// as the http transport, where the engine/operator is the trust boundary. A URL
// that embeds the configured API key is refused up front ...
```

**trust boundary 设定**：engine（和操作者）就是信任边界 —— engine 声明什么 URL 就 fetch 什么 URL，这是「exact declared URL」契约。但**重定向打破这个契约**：一旦跟随 3xx，实际 fetch 的就不再是 engine 声明的那个 URL，而是某个第三方（attacker 或被劫持的 host）指定的任意地址。所以核心防御是**关闭重定向跟随**（`custom.go:867-872`）：

```go
// Redirects are NOT followed: a 3xx would let the artifact host (or the engine)
// swap the archived bytes for a Location pointing at a different — possibly
// internal — endpoint, breaking the "exact declared URL" boundary and the
// up-front api-key/scheme checks. ErrUseLastResponse returns the 3xx as-is so
// it falls through to the non-2xx skip path below.
client := &http.Client{
    Timeout: customArtifactDownloadTimeout,
    CheckRedirect: func(*http.Request, []*http.Request) error {
        return http.ErrUseLastResponse
    },
}
```

`http.ErrUseLastResponse` 是 Go stdlib 的一个哨兵 error，告诉 `http.Client`：「不要跟随，把最近的响应原样返回给我」。这样 3xx 就会落到下面 `if resp.StatusCode < 200 || resp.StatusCode >= 300` 的分支，被当作非 2xx 跳过。

**前置检查（纵深防御）**：

```go
// custom.go:833-849
parsed, err := url.Parse(f.URL)
// ...
if parsed.Scheme != "http" && parsed.Scheme != "https" {
    logging.WarnContextf(ctx, "CustomAgent: artifact %q url scheme %q is not http(s); skipping", f.Name, parsed.Scheme)
    return ""
}
if a.containsAPIKey(f.URL) {
    logging.WarnContextf(ctx, "CustomAgent: artifact %q url embeds the API key; skipping download", f.Name)
    return ""
}
```

- **scheme 白名单**：只允许 `http` / `https`。`file://` / `gopher://` / `dict://` 这些会被 url 驱动访问本地文件或奇怪协议的 scheme 一律拒绝。
- **api-key 检查**：见 §3.4。

**为什么要多层检查 + 关重定向？** 因为任何一层都不是完整的 —— scheme 白名单挡不住 `http://169.254.169.254`（它就是合法的 http），不跟随重定向也挡不住 engine **直接**返回内网 URL（它就是声明的那个 URL）。组合起来才能覆盖：「engine 声明的 URL 必须是 http(s)、不能含 key；只 fetch 这个 URL，不跟随任何跳转」。

#### 下载后还有 size cap（`custom.go:886-896`）

```go
body, err := io.ReadAll(io.LimitReader(resp.Body, maxArtifactDownloadBytes+1))
if err != nil {
    logging.WarnContextf(ctx, "CustomAgent: cannot read artifact %q body: %v", f.Name, a.maskAPIKey(err.Error()))
    return ""
}
if int64(len(body)) > maxArtifactDownloadBytes {
    logging.WarnContextf(ctx, "CustomAgent: artifact %q exceeds download size limit %d bytes; dropping", f.Name, maxArtifactDownloadBytes)
    return ""
}
```

`maxArtifactDownloadBytes = 256 * 1024 * 1024`（256MB，`custom.go:63`）。**多读 1 字节**是为了能区分「正好达到上限」和「超过上限」—— 只读到上限会**静默截断**，下载者得到一个被截断的文件还以为下载完了；多读 1 字节后只要 `len(body) > cap` 就 100% 是超限，直接丢弃。error 也过 `maskAPIKey`（URL 嵌在 error 里）。

> **Java 类比**：Java 圈防 SSRF 的标准做法是「自定义 `HttpClient.setRedirectPolicy(NEVER)` + URL 白名单解析 + 禁用非 http(s) scheme」。Apache HttpClient 的 `HttpClientBuilder.disableRedirectHandling()` 等价于这里的 `ErrUseLastResponse`。skill-up 多了一层「这是 engine 声明的 URL，操作者是信任边界」的**姿态声明** —— 它不试图枚举所有内网段（169.254.169.254 / 10.0.0.0/8 / 127.0.0.1 ...）做黑名单，因为那种黑名单永远漏；而是用「不跟随重定向 + scheme 白名单 + size cap」把可控范围卡死。这是更**工程务实**的姿态。

---

### 3.6 multipart size cap 256MB（T6 / 内存耗尽）

#### 攻击场景

HTTP transport 的 `http.files` 支持用 glob 上传 workspace 文件（比如 `**/*`）。如果 workspace 很大（比如 engine 在跑的过程中生成了几个 GB 的中间产物），整个 multipart body 在内存里组装（`bytes.Buffer`），直接 OOM。同样，一个恶意或配置错误的 engine HTTP 服务可以返回一个无上限的响应体。

#### 防御 1：上传侧累计上限（`custom_http_files.go:21, 44-49, 62-96`）

```go
// custom_http_files.go:21
const maxHTTPUploadBytes = 256 * 1024 * 1024

// custom_http_files.go:44-49 (buildMultipartBody)
var total int64
for _, rel := range relPaths {
    if err := addFilePart(ctx, rt, mw, rel, &total); err != nil {
        return nil, "", err
    }
}

// custom_http_files.go:62-81 (addFilePart)
func addFilePart(ctx context.Context, rt Runtime, mw *multipart.Writer, rel string, total *int64) error {
    // 先下到临时文件，再用 os.Stat 拿到真实大小
    tmpFile, err := os.CreateTemp("", "skill-up-http-upload-*")
    // ...
    if err := rt.DownloadFile(ctx, rel, tmp); err != nil { /* ... */ }
    info, err := os.Stat(tmp)
    // ...
    *total += info.Size()
    if *total > maxHTTPUploadBytes {
        return fmt.Errorf("http.files: total upload exceeds %d bytes", maxHTTPUploadBytes)
    }
    // ... 才 io.Copy 进 multipart part
}
```

**关键设计**：先 `DownloadFile` 到临时文件 + `os.Stat` 拿大小，**累计判断**通过后才 `io.Copy` 进 multipart。如果先 copy 再判断，一个大文件已经把内存吃光了 —— 累计判断必须在 **copy 前**用文件大小做。注释（`custom_http_files.go:57-61`）也承认这是可改进点：

> The whole multipart body is still buffered in memory, bounded by maxHTTPUploadBytes; with many parallel cases a streaming (io.Pipe) body would lower peak RAM — a future improvement.

即：上限保证了不 OOM，但峰值 = 256MB × 并发 case 数；未来用 `io.Pipe` 流式拼 body 可以进一步降峰值。

#### 防御 2：HTTP 响应体上限（`custom_http.go:47-50, 149-160`）

```go
// custom_http.go:47-50
const maxHTTPResponseBytes = 64 * 1024 * 1024  // 64MB

// custom_http.go:149-160 (doRequest)
respBody, readErr := io.ReadAll(io.LimitReader(resp.Body, maxHTTPResponseBytes+1))
if readErr != nil { /* ... */ }
if int64(len(respBody)) > maxHTTPResponseBytes {
    msg := fmt.Sprintf("custom engine http response exceeded %d bytes", maxHTTPResponseBytes)
    return &transportOutcome{exitCode: 1, stderr: msg, execErr: errors.New(msg)}
}
```

同样**多读 1 字节**避免静默截断。一个 SessionResult JSON 64MB 已经远超合理大小（典型几 KB 到几 MB），超过就是配置错误或恶意。

#### 防御 3：inline artifact 上限（`custom.go:39-46, 678-697, 787-813`）

三层 cap，逐层收紧：

```go
// custom.go:39-46
maxInlineArtifactBytes  = 50 * 1024 * 1024   // 单个 inline artifact
maxInlineArtifactTotal  = 200 * 1024 * 1024  // 单个 SessionResult 内所有 inline 总和
```

- `validateArtifactSize`（`custom.go:682-697`）：解析时用 base64 长度**估算**解码后大小，超限直接 reject（不付解码代价）；
- `writeInlineArtifact`（`custom.go:802-806`）：解码后**真实大小**再校验一次 —— 因为 `len(content_base64) * 3/4` 是估算，padding 和实际解码可能差一点，真实大小再校验保证 100% 拦得住。

#### 防御 4：HTTP transport 即使非 2xx 也读 body（`custom_http.go:165-174`）

```go
outcome := &transportOutcome{raw: raw, stdout: raw}
if resp.StatusCode < 200 || resp.StatusCode >= 300 {
    outcome.exitCode = resp.StatusCode
    outcome.stderr = a.maskAPIKey(strings.TrimSpace(raw))
    outcome.execErr = fmt.Errorf("custom engine http status %d", resp.StatusCode)
}
```

非 2xx 也读 body —— 因为 body 里可能带 partial SessionResult（engine 中途崩了但已经吐了部分 JSON），让 `assembleResult` 有机会挽救。

> **Java 类比**：这就像在 Spring 里给 `WebClient` 配 `maxInMemorySize(64MB)`（默认 256KB，太小）—— 任何反序列化前的缓冲都要有上限，否则一个恶意 / 失控的 peer 一个 HTTP 响应就让你 OOM。多读 1 字节区分「达到上限 vs 超过上限」是处理 bounded read 的通用技巧。

---

## 4. 配置侧安全 customengine.go（防用户自己 leak 自己，T7 / T8）

这一节关心的是**另一类威胁**：不是 engine 返回值攻击 skill-up，而是**用户写错配置**把自己 leak 了。开源工具被各种水平的用户配置，"防止用户把明文密钥写进会被日志/进程列表暴露的地方"是成熟产品的标配意识。

> 注意路径：实际的敏感模式匹配逻辑在 `internal/config/customengine.go`（不是 `internal/customengine/`，后者是依赖无关的配置结构体 + 路径检查 + 模板 token 解析的叶子包）。这是**分层**：`internal/customengine/` 不 import `os` / `regexp`，可以被任何层（包括 `pkg/`）安全引用；带副作用的解析逻辑在 `internal/config/`。

### 4.1 敏感变量名识别 `sensitiveEnvNamePattern`（`config/customengine.go:18-24`）

```go
var sensitiveEnvNamePattern = regexp.MustCompile(
    `(?i)(^|_)(API_?KEY|ACCESS_?KEY|KEY|TOKEN|SECRET|PASSWORD|PASSWD|CREDENTIALS?|AUTHORIZATION)(_|$)`,
)

func isSensitiveEnvName(name string) bool {
    return sensitiveEnvNamePattern.MatchString(name)
}
```

匹配 `API_KEY` / `TOKEN` / `SECRET` / `PASSWORD` / `CREDENTIALS` / `AUTHORIZATION` 等敏感词，用 `(^|_)...(_|$)` 词边界锚定。注释特别提到「Word boundaries avoid false positives like MONKEY_PATH」—— 不用 `(?i).*key.*` 这种粗暴正则，否则 `MONKEY_PATH`、`KEYBOARD_LAYOUT` 全部误判。

`(?i)` 大小写不敏感，覆盖 `apiKey` / `Api_Key` / `API_KEY` 各种风格。

### 4.2 凭证形状识别 `secretLiteralPatterns`（`config/customengine.go:30-41`）

光靠变量名不够 —— 用户可能把 key 写进一个**看起来无害的变量名**的默认值：`${INNOCUOUS_NAME:-sk-ant-real-key}`。所以还要识别**真实凭证的形状**：

```go
var secretLiteralPatterns = []*regexp.Regexp{
    // Vendor prefixes (Anthropic sk-ant-, OpenAI sk-, GitHub ghp_/gho_/ghu_/
    // ghs_/ghr_, Google AIza, Slack xox[abp]-, AWS AKIA/ASIA) — case-sensitive
    // on purpose: lowercase prefixes match real keys, uppercase ones AWS IDs.
    regexp.MustCompile(`(?:^|[^A-Za-z0-9_])sk-[A-Za-z0-9_\-]{16,}`),
    regexp.MustCompile(`(?:^|[^A-Za-z0-9_])gh[pousr]_[A-Za-z0-9]{20,}`),
    regexp.MustCompile(`(?:^|[^A-Za-z0-9_])AIza[A-Za-z0-9_\-]{20,}`),
    regexp.MustCompile(`(?:^|[^A-Za-z0-9_])xox[abposr]-[A-Za-z0-9\-]{10,}`),
    regexp.MustCompile(`(?:^|[^A-Za-z0-9_])(?:AKIA|ASIA)[A-Z0-9]{16}`),
    // JWT: three base64url segments separated by dots.
    regexp.MustCompile(`(?:^|[^A-Za-z0-9_])ey[A-Za-z0-9_\-]{10,}\.[A-Za-z0-9_\-]{10,}\.[A-Za-z0-9_\-]{10,}`),
}
```

覆盖主流厂商：

| 模式 | 厂商 |
|------|------|
| `sk-...` | OpenAI / Anthropic |
| `gh[pousr]_...` | GitHub token（PAT / OAuth / app / server / refresh） |
| `AIza...` | Google API key |
| `xox[abposr]-...` | Slack token |
| `AKIA/ASIA...` | AWS access key ID |
| `ey...\.xxx\.xxx` | JWT（三段 base64url 用 `.` 分隔，以 `ey` 开头因为 `{"..."` base64url 编码后是 `ey`） |

**case-sensitive** 是有意的：lowercase 前缀匹配真实 key，uppercase 的 `AKIA` 匹配 AWS ID（AWS ID 一律大写）。`(?:^|[^A-Za-z0-9_])` 前缀避免匹配到 `mysk-xxx` 这种变量名中段 —— 必须前面是非单词字符或字符串开头。

### 4.3 严格模式 `resolveEnvRefsStrict`（`config/customengine.go:397-410`）

```go
func resolveEnvRefsStrict(s string) (string, error) {
    const maxStrictExpansionDepth = 10
    for range maxStrictExpansionDepth {
        resolved, err := resolveEnvRefsWith(s, true)
        if err != nil {
            return "", err
        }
        if resolved == s || !strings.Contains(resolved, "${") {
            return resolved, nil
        }
        s = resolved
    }
    return "", errors.New("strict env resolution exceeded maximum depth (possible reference cycle)")
}
```

**「严格」体现在两件事**：

1. **拒绝敏感名**：单次 `resolveEnvRefsWith(s, true)` 里（`config/customengine.go:492-497`），如果 token 名匹配 `isSensitiveEnvName`（如 `${MY_API_KEY}`），直接报错：
   ```go
   if rejectSecrets && isSensitiveEnvName(tok.Name) {
       return "", false, fmt.Errorf(
           "secret-like environment variable %q must not be referenced in a command line; pass credentials via engine.custom.env instead",
           tok.Name,
       )
   }
   ```

2. **迭代展开防走私（T8）**：这是注释里讲得最透的一点：

> The resolver iterates: if the produced value itself embeds further ${...} references (a wrapper env var whose value is "${CUSTOM_AGENT_TOKEN}"), the next pass re-checks them, so a non-sensitive wrapper cannot smuggle a sensitive name through to run-time rendering.

**攻击场景（T8）**：用户在 shell env 里设了 `WRAPPER=${ANTHROPIC_API_KEY}`（`WRAPPER` 这个名字不敏感），然后在 skill-up 配置里写 `command: ${WRAPPER}`。如果只做**单次**检查，`WRAPPER` 通过（名字不敏感），渲染后变成 `${ANTHROPIC_API_KEY}` 字面量，再被运行时展开 —— key 进了命令行。**迭代展开**后第二轮看到 `${ANTHROPIC_API_KEY}`，被 `isSensitiveEnvName` 拦下。最多 10 轮，到顶报「possible reference cycle」（防 `A=${B}, B=${A}` 死循环）。

#### 严格模式用在哪：所有「会变成命令行」的字段

`resolveLocalEnv`（`config/customengine.go:274-289`）对 **command / args / cwd / input_file / output_file** 全用 strict：

```go
func resolveLocalEnv(l *customengine.LocalConfig) []string {
    var errs []string
    errs = append(errs, resolveScalarEnvStrict("local.command", &l.Command)...)
    errs = append(errs, resolveScalarEnvStrict("local.cwd", &l.Cwd)...)
    errs = append(errs, resolveScalarEnvStrict("local.input_file", &l.InputFile)...)
    errs = append(errs, resolveScalarEnvStrict("local.output_file", &l.OutputFile)...)
    for i := range l.Args {
        rv, err := resolveEnvRefsStrict(l.Args[i])
        // ...
    }
    return errs
}
```

而 `env` / `http.headers` 这些**合法持有凭据**的字段用普通模式（`resolveStringMapEnv`）。**kwargs 用 strict**（`resolveStringMapEnvStrict`），因为 kwargs 会通过 `${kwargs.token}` 展开进命令行。

#### `resolveEnvRefsWith` 的三段检查（`config/customengine.go:417-471`）

```go
func resolveEnvRefsWith(s string, rejectSecrets bool) (string, error) {
    if s == "" {
        return s, nil
    }
    if !strings.Contains(s, "${") {
        // (a) 没有引用，但 raw literal 也可能是凭证
        if rejectSecrets && looksLikeSecret(s) {
            return "", errors.New(
                "value looks like a credential literal; pass secrets via engine.custom.env instead of inline command-line literals",
            )
        }
        return s, nil
    }
    // (b) 逐 token 解析
    var b strings.Builder
    for i := 0; i < len(s); {
        // ... 取出 inner ...
        value, leaveIntact, err := resolveEnvToken(inner, rejectSecrets)
        // ...
    }
    out := b.String()
    // (c) 渲染后再扫一遍 —— 防 "Bearer ${PREFIX}sk-ant-real-key" 这种拼接
    if rejectSecrets && looksLikeSecret(out) {
        return "", errors.New(
            "resolved value looks like a credential literal; pass secrets via engine.custom.env instead of inline command-line literals",
        )
    }
    return out, nil
}
```

**三段**：
- **(a)** 没有引用时也扫：纯字面量 `args: ["--token", "sk-ant-xxx"]` 也拦。
- **(b)** 每个 token 内部检查（敏感名 / 内置敏感变量 / 默认值形状）。
- **(c)** 拼接后再扫：`Bearer ${PREFIX}sk-ant-real-key`，`PREFIX` 不敏感通过，但拼接出的最终字符串匹配 `looksLikeSecret` —— 拦下。

#### `isSensitiveTemplateVar`：内置变量也要查（`config/customengine.go:87-98`）

```go
func isSensitiveTemplateVar(name string) bool {
    switch name {
    case "api_key", "kwargs", "kwargs_json", "session_input", "session_input_json":
        return true
    }
    if key, ok := strings.CutPrefix(name, "kwargs."); ok {
        return isSensitiveEnvName(normalizeKeyForSensitiveCheck(key))
    }
    return false
}
```

- `${api_key}` —— 显然敏感；
- `${kwargs}` / `${kwargs_json}` / `${session_input}` / `${session_input_json}` —— 整个 map 序列化，里面**可能**含敏感 kwargs key，按敏感处理；
- `${kwargs.<key>}` —— 看 `<key>` 的名字（归一化后）是否敏感。

### 4.4 命名归一化 `normalizeKeyForSensitiveCheck`（`config/customengine.go:104-126`）

```go
// normalizeKeyForSensitiveCheck converts a kwarg key into UPPER_SNAKE_CASE so
// the sensitive-name pattern recognizes alternative naming conventions —
// hyphenated ("api-key"), dotted ("api.key"), camelCase ("apiKey",
// "bearerToken"), and other non-alphanumeric separators.
func normalizeKeyForSensitiveCheck(key string) string {
    var sep strings.Builder
    sep.Grow(len(key))
    for _, r := range key {
        if unicode.IsLetter(r) || unicode.IsDigit(r) {
            sep.WriteRune(r)
        } else {
            sep.WriteByte('_')        // 非字母数字 → _
        }
    }
    normalized := sep.String()
    var b strings.Builder
    b.Grow(len(normalized) + 4)
    prevLower := false
    for _, r := range normalized {
        if prevLower && unicode.IsUpper(r) {
            b.WriteByte('_')           // camelCase 边界 → _
        }
        b.WriteRune(r)
        prevLower = unicode.IsLower(r)
    }
    return strings.ToUpper(b.String())
}
```

**为什么要归一化？** 用户写 kwargs key 风格五花八门：`api-key`（hyphen）、`apiKey`（camelCase）、`bearerToken`、`API_KEY`（已经规范）。`sensitiveEnvNamePattern` 只认 UPPER_SNAKE，所以要把所有风格统一过去：

- `api-key` → `api_key` → `API_KEY` ✓
- `apiKey` → `api_Key` → `API_KEY` ✓（camelCase 边界插入 `_`）
- `bearerToken` → `bearer_Token` → `BEARER_TOKEN`，匹配 `TOKEN` 词 ✓

**攻击场景**：用户写 `kwargs: {apiKey: "${ANTHROPIC_API_KEY}"}`，然后 `command: "wrapper --token ${kwargs.apiKey}"`。如果归一化只认字面 `API_KEY`，`apiKey` 这个 camelCase key 就**绕过 kwargs 敏感检查**，把 key 渲染进命令行。归一化后 `apiKey` 被识别为敏感，拦下。

> **Java 类比**：这相当于 Spring 的 `RelaxedPropertyResolver` / `Binder` —— 它能把 `api-key` / `apiKey` / `API_KEY` 都绑定到同一个 `@ConfigurationProperties`。差别是 skill-up 用归一化做**安全检查**而不是绑定，但思路一致：「不能因为用户命名风格不同就让检查失效」。

---

## 5. 凭据管理 credential.go（多源 + 脱敏日志 + 从不打明文）

### 5.1 多源优先级（`credential/credential.go` + `agent_init.go`）

凭据来源按优先级（高 → 低）：

| 源 | 入口 | 备注 |
|----|------|------|
| **CLI flag** `--api-key` | `applyCLIOverrides`（`agent_init.go:181-200`） | 最高，覆盖一切 |
| **Provider-scoped env** `${PROVIDER}_API_KEY` | `lookupProviderEnv`（`agent_init.go:245-255`） | 比如 `ANTHROPIC_API_KEY`、`OPENAI_API_KEY` |
| **Resolver**（`~/.skill-up/credentials.yaml`） | `resolveValue`（`agent_init.go:220-243`） | 用户私有配置文件 |
| **Runner fallback** | `applyFallbackCredentials`（`agent_init.go:171-179`） | Judge 复用 Runner 的 key |

`.env` 通过 `godotenv.Load()` 加载（`credential.go:61`），`credentials.yaml` 通过 `loadFromFile` 解析（`credential.go:138-178`）。两个源**互补**：env 适合 CI（环境变量注入），yaml 适合本地开发（一个文件管多 provider）。

### 5.2 `MaskAPIKey`：首 2 后 2（`credential.go:91-98`）

```go
const apiKeyMaskLen = 4 // Length for API key masking

func MaskAPIKey(key string) string {
    if len(key) <= apiKeyMaskLen {
        return "****"
    }

    return key[:2] + "****" + key[len(key)-2:]
}
```

**短于等于 4 字符**全打码（避免 `key[:2]` 越界 + 短 key 暴露比例太大）；长 key 显示首 2 后 2。比如 `sk-ant-api03-xxxxxxxxxxxx` → `sk****xx`。

**为什么是首尾各 2 而不是「中间打码保留前缀」？** 因为厂商前缀本身就识别身份（`sk-ant-` = Anthropic），保留前 2 字符刚好够让人**认出这是哪种 key**（debug 用），但不够复原。完全打码（`****`）反而难 debug（不知道是哪个 key）。

> 注意与 `CustomAgent.maskAPIKey`（§3.3）的区别：`MaskAPIKey` 是**配置层日志脱敏**（保留首尾可识别），`maskAPIKey` 是**结果层全替换**（`***REDACTED***`，更激进）。两者用在不同位置是有意的 —— 日志保留可识别度便于排错，结果文件 / Judge 输入要彻底不能复原。

### 5.3 脱敏日志（`credential.go:101-115` + `agent_init.go:289-306`）

`logEffectiveConfig`（`credential.go:101-115`）：

```go
func (r *Resolver) logEffectiveConfig() {
    if len(r.creds) == 0 {
        logging.Debugf("No credentials loaded from config file.")
        return
    }
    for name, cred := range r.creds {
        if cred.APIKey != "" {
            logging.Debugf("CREDENTIAL_DISCOVERED provider=%s source=resolver api_key=%s", name, MaskAPIKey(cred.APIKey))
        }
        if cred.BaseURL != "" {
            logging.Debugf("CREDENTIAL_DISCOVERED provider=%s source=resolver base_url=%s", name, cred.BaseURL)
        }
    }
}
```

`logResolvedAgentConfig`（`agent_init.go:289-306`）：

```go
if params.APIKey != "" {
    logging.Debugf("AGENT_CONFIG kind=%s engine=%s api_key=%s source.api_key=%s",
        params.Kind, params.Engine, MaskAPIKey(params.APIKey), params.APIKeySource)
}
```

**每一个打印 api_key 的地方都过 `MaskAPIKey`** —— 没有 `Debugf("api_key=%s", cred.APIKey)` 这种裸打。注释（`agent_init.go:281-282`）连字段名都换成 `auth_env`：

```go
logging.Debugf("AGENT_CONFIG kind=%s engine=%s provider=%s auth_env=%s source.auth=%s",
    params.Kind, params.Engine, params.Provider, envVar, ValueSourceEnv)
```

注意 `auth_env` 而不是 `api_key_env` —— **连环境变量名都不暴露**，因为某些环境变量名（如 `ANTHROPIC_API_KEY`）本身能泄露 provider 信息。

### 5.4 ProviderConfig 双格式（`credential.go:124-136`）

```go
type ProviderConfig struct {
    APIKey  string `yaml:"api_key,omitempty"`
    BaseURL string `yaml:"base_url,omitempty"`

    OpenAI    *ProviderEndpointConfig `yaml:"openai,omitempty"`
    Anthropic *ProviderEndpointConfig `yaml:"anthropic,omitempty"`
}
```

同时支持两种 yaml 写法：

**Flat 格式**：
```yaml
providers:
  my-proxy:
    api_key: sk-xxx
    base_url: https://proxy.internal
```

**Nested 格式**（一个 provider 下挂多个 endpoint）：
```yaml
providers:
  my-proxy:
    openai:
      api_key: sk-xxx
      base_url: https://proxy.internal/openai
    anthropic:
      api_key: sk-ant-yyy
```

`loadFromFile`（`credential.go:138-178`）的合并逻辑：flat 优先，nested 补充（`if cred.APIKey == "" && p.OpenAI.APIKey != ""`）。这照顾了两种使用场景 —— 简单的单 endpoint 用 flat，复杂的代理服务用 nested。

### 5.5 文件位置 `DefaultConfPath`（`credential.go:31-38`）

```go
func DefaultConfPath() string {
    home, err := os.UserHomeDir()
    if err != nil || home == "" {
        return ""
    }
    return filepath.Join(home, ".skill-up", "credentials.yaml")
}
```

`~/.skill-up/credentials.yaml` —— 跟 `~/.docker/config.json`、`~/.aws/credentials`、`~/.gitconfig` 同一套约定。**取不到 home 时返回空串**，调用方跳过文件层（不会 panic 或 fatal）。

> **Java 类比**：整套设计就是 Spring Security 里 `PropertySource` 优先级链（CLI > env > yaml > default）+ 日志脱敏的组合。差别是 skill-up 是无框架、纯 Go 实现 —— 自己写 Resolver、自己写 MaskAPIKey、自己确保每一处 `Debugf` 都过 mask。面试可以讲：「凭据管理的核心不是加密存储（那是 vault / KMS 干的），而是**确保明文在内存里流转时永远不出现在日志 / 进程列表 / 报告文件里**——这要靠『每一个 print 点都过 mask』的纪律，不是靠一处全局拦截。」

---

## 6. shellquote：统一引用，永不字符串拼接不可信命令（`internal/shellquote/shellquote.go`）

### 6.1 POSIX 引用（`shellquote.go:7-9`）

```go
func QuotePOSIX(s string) string {
    return "'" + strings.ReplaceAll(s, "'", `'\''`) + "'"
}
```

把任意字符串包成 POSIX shell 单引号字符串。唯一需要 escape 的是单引号本身 —— 用 `'\''`（关闭引号 → 转义单引号 → 重开引号）这个标准技巧。

**为什么单引号而不是双引号？** 单引号里**没有任何转义**（除了单引号本身），shell 不会解释 `$var`、反引号、`\`，最安全。双引号会展开 `$`、反引号，留攻击面。

### 6.2 Windows 引用（`shellquote.go:24-49`）

```go
func QuoteWindows(s string) string {
    if s == "" {
        return `""`
    }
    var b strings.Builder
    b.WriteByte('"')
    backslashes := 0
    for i := range len(s) {
        switch c := s[i]; c {
        case '\\':
            backslashes++
        case '"':
            b.WriteString(strings.Repeat(`\`, backslashes*2+1))
            b.WriteByte('"')
            backslashes = 0
        default:
            b.WriteString(strings.Repeat(`\`, backslashes))
            backslashes = 0
            b.WriteByte(c)
        }
    }
    // Double trailing backslashes so they do not escape the closing quote.
    b.WriteString(strings.Repeat(`\`, backslashes*2))
    b.WriteByte('"')
    return b.String()
}
```

按 `CommandLineToArgvW` 的解析规则引用 —— 这是 Windows 命令行参数解析的事实标准。核心规则：

- 用双引号包裹；
- **反斜杠只在「紧邻双引号」时才特殊**：`2n` 个 `\` 后接 `"` → `n` 个 `\` + 一个字面 `"`；其他位置的 `\` 原样。
- 因此 `\` 紧邻 `"` 时翻倍（`backslashes*2+1`），其他位置的 `\` 原样输出。

**注释里的真实 bug 故事**（`shellquote.go:18-23`）：

> The result is always wrapped, even when s has no whitespace or metacharacter triggers: when NoneRuntime.Exec routes through bash on Windows (the default when Git Bash is discoverable), an unquoted backslash-bearing path such as `C:\tmp\script.cmd` would have its backslashes stripped by bash and reach the downstream `cmd /c` / `powershell -File` as `C:tmpscript.cmd`. Wrapping in double quotes keeps backslashes literal under both bash and cmd, and is equally safe for CommandLineToArgvW consumers.

**永远包裹**（即使没有空格）—— 因为 bash 在 Windows 上会吃掉裸路径里的 `\`。这是从真实 bug 里学出来的规则。

### 6.3 在 CustomAgent 里的使用（`custom.go:271-272, 291, 300` + `agent.go:391-393`）

```go
// agent.go:391-393 —— 项目内统一包装函数
func shellQuote(s string) string {
    return shellquote.QuotePOSIX(s)
}
```

`buildLocalExec` 里**每个 arg 都过 shellQuote**（`custom.go:291, 300`）：

```go
parts := []string{shellQuote(command)}
for i, raw := range local.Args {
    arg, rErr := renderTemplate(raw, vars)
    // ...
    parts = append(parts, shellQuote(arg))
}
```

`clearStaleOutputFile` 里拼 `rm` 命令也用（`custom.go:271-272`）：

```go
q := shellQuote(outputFile)
cmd := "if [ -e " + q + " ]; then rm -f -- " + q + " && printf %s " + shellQuote(customStaleClearedMarker) + "; fi"
```

**核心纪律**（`AGENTS.md` 原话）：

> When adding new shell execution paths, use `internal/shellquote` — never concatenate untrusted strings into a shell command.

这是项目级的硬规则。任何「把外部字符串拼进 shell 命令」的地方都必须过 `shellQuote`，否则 revview 会被打回。

> **Java 类比**：这相当于永远用 `ProcessBuilder(List<String>)` 而不是 `Runtime.exec(String)` + 字符串拼接。Go 的 `os/exec` 默认走 `execve`（argv 数组，不经 shell），但 skill-up 的 NoneRuntime 在 Windows 上要路由经过 bash，所以仍需要 shell 引用。关键是「**不可信字符串永远只作为单个 argv 元素**，绝不拼进 shell 字符串」—— 这条规则在 Java 圈也是一样。

---

## 7. CustomAgent 的 no-op 设计（为何 Install/Check/CheckCredentials 是 no-op）

`internal/agent/custom.go:89-114`：

```go
// Install is a no-op: a custom engine's command is provided and managed by the
// user, not installed by skill-up.
func (a *CustomAgent) Install(_ context.Context, _ Runtime) error { return nil }

// InstallMCP is a no-op for custom engines: a custom engine discovers MCP
// servers from the runtime environment itself. Declared servers are logged so
// an eval is not silently missing expected MCP wiring, but no installation is
// attempted (unlike the inherited CLIAgent.InstallMCP, which would error).
func (a *CustomAgent) InstallMCP(ctx context.Context, _ Runtime, mcpCfg runtime.MCPConfig) error {
    if len(mcpCfg.Servers) > 0 {
        logging.InfoContextf(ctx, "CustomAgent %q: %d MCP server(s) declared; a custom engine manages MCP itself, skipping installation", a.Name(), len(mcpCfg.Servers))
    }
    return nil
}

// Check is a no-op for custom engines.
func (a *CustomAgent) Check(_ context.Context, _ Runtime) error { return nil }

// CheckCredentials is a no-op: a custom engine references credentials
// explicitly via ${api_key}, so skill-up does not pre-validate them.
func (a *CustomAgent) CheckCredentials(_ context.Context) error { return nil }
```

### 7.1 为什么是 no-op

| 方法 | CLIAgent 行为 | CustomAgent 行为 | 原因 |
|------|--------------|------------------|------|
| `Install` | 装 CLI（npm install / brew install） | no-op | 用户的命令是**用户自己管的**，skill-up 不知道装什么、也没权限装 |
| `InstallMCP` | 注册 MCP server 到 CLI 配置 | no-op（只 log） | custom engine 自己从 runtime env 发现 MCP，skill-up 不插手；**故意不报错**避免误伤 |
| `Check` | 检查 CLI 是否可用（`--version`） | no-op | 没有统一的「检查」契约 —— 一条 shell 命令检查不了「能不能跑」，跑了等于真跑 |
| `CheckCredentials` | 预验证 API key 有效（调一次轻量 API） | no-op | custom engine 通过 `${api_key}` **显式**引用凭据，skill-up 不知道 endpoint，无法预验证 |

### 7.2 这背后是「不嵌 CLIAgent」的接口设计（`custom.go:71-79`）

```go
// CustomAgent embeds BaseAgent directly (not CLIAgent) so it inherits only
// shared agent infrastructure — Name, credential helpers, exec-options merge,
// installSkillDefault. Run / Install / InstallMCP / InstallSkill / Check /
// CheckCredentials are all defined explicitly, so future additions to
// CLIAgent do not accidentally leak in with the wrong semantics for a custom
// engine.
type CustomAgent struct {
    BaseAgent
}
```

**关键设计**：CustomAgent **故意不嵌 CLIAgent**，只嵌 BaseAgent。这样 CLIAgent 未来加方法时（比如加一个 `Login()`），不会**意外继承**到 CustomAgent 上 —— CLIAgent 的方法语义是「CLI 类 Agent」（有版本检查、有登录态），对 custom engine 全是错的。

**安全视角的好处**：no-op 化让 custom engine 的**接口表面最小**。CustomAgent 不会主动调任何「检查 / 安装 / 验证」逻辑 —— 这些逻辑可能假设 CLI 行为，对 custom engine 会做出错误假设（比如 `Check` 调 `--version` 触发了用户脚本的副作用）。**接口表面越小，攻击面越小**。

> **Java 类比**：这是「接口隔离原则（ISP）」的落地 —— 不让一个类被迫实现它用不到的方法。Java 里等价做法是 `interface CustomAgent extends BaseAgentOnly`，而不是 `extends CLIAgent` 然后把所有 CLI 方法 override 成抛 `UnsupportedOperationException`。Go 没有「extends」，靠「组合 + 重新定义方法」实现：嵌入 BaseAgent 拿到基础方法，自己定义 Install/Check 等覆盖掉（Go 的 embedding 里外层方法会 shadow 内层）。

---

## 8. 关键设计点（带 Java 类比）

### 8.1 TOCTOU 防御：check 挪到离 use 越近越好（不是 check 更严格）

**做法**：不是「检查一次 + 缓存」，而是「每个 use site 都 `workspacePath` 重新解析」。`resolveCustomIOFiles`（run 前）→ `clearStaleOutputFile`（清前）→ `readRawResult`（读前）→ `collectArtifacts`（归档前）→ `archiveRenamedPathArtifact`（拷前），**五次重解析**。

**Java 类比**：`Files.copy` 前 `toRealPath()`，而不是「函数入口处 `toRealPath()` 一次，后面 5 个 IO 都用缓存值」。**check 严格度提升没用，要靠时间窗口压缩**。

### 8.2 不可信输入：纵深防御，多层独立可绕过

**做法**：URL artifact 下载同时上 (1) scheme 白名单、(2) api-key 检查、(3) 不跟随重定向、(4) size cap、(5) error 也脱敏。每一层都假设其他层失效也能挡。

**Java 类比**：OWASP 推荐 SSRF 防御就是「URL 白名单 + 不跟随重定向 + 响应大小限制 + 不返回原始 error」组合，**没有任何单点能完整防御**。Spring Security 的 filter chain 也是这种思路。

### 8.3 凭据脱敏：字段级贴面，不靠全局拦截

**做法**：`maskAPIKey` 不是「写一个 log filter 全局替换」，而是**每个不可信字段流入结果对象时手动调用**。FinalMessage / Stderr / Transcript / Engine / Model / OTel attribute 各自贴。好处是「新加字段忘了 mask」只影响那个字段，老字段仍然安全；坏处是要靠 code review 保证新字段也贴 —— 这是工程权衡。

**Java 类比**：相当于在每个 DTO 的 getter 里脱敏，而不是配置一个 Jackson `@JsonSerialize` 全局 serializer。差别是前者更显式、更难被绕过（你必须显式调用），后者更省代码但容易漏（不标解解就泄漏）。

### 8.4 trust boundary 显式声明：engine / operator 是边界

**做法**：注释里反复出现「the engine/operator is the trust boundary」「exact declared URL」。skill-up **不试图**枚举所有内网 IP / 恶意 domain 做 SSRF 黑名单，因为那种黑名单永远漏；而是声明「engine 声明的 endpoint 就是要访问的，操作者负责 endpoint 可信」，然后在这个假设下卡死「重定向 / scheme / size」三个开关。

**Java 类比**：Cloudflare / nginx 配置里 `proxy_pass` 信任上游，不试图在反代里做内网过滤。这是**分层信任模型**：每一层只管自己那一层的事。

### 8.5 no-op 即安全：接口表面最小化

**做法**：CustomAgent 的 Install/Check/CheckCredentials 全是 no-op。这些方法对 CLI Agent 有意义（装 / 验证 CLI / 验证凭据），对 custom engine 没意义（用户的命令用户自己管）。强行实现会引入错误假设。

**Java 类比**：「最小权限」不只适用于运行时权限，也适用于**接口表面**。一个类暴露的方法越多，调用方能触发的行为越多，攻击面越大。Go 通过「不嵌 CLIAgent」实现，Java 通过「只 implements 真正需要的 interface」实现。

---

## 9. 面试 Q&A

### Q1：TOCTOU 是什么？skill-up 怎么防的？

**A**：TOCTOU = Time-of-Check to Time-of-Use，是「检查时刻」和「使用时刻」之间攻击者改了状态的竞态漏洞。skill-up 防御 `workspacePath` 路径穿越时（`internal/agent/custom.go:1115`）：

1. **`EvalSymlinks` 展平 workspace 根和中间祖先**（`resolveExistingPrefix`），把已存在的符号链接物理位置定下来；
2. **返回已展平的路径**（不是 lexical abs），调用方 follow 时走物理路径；
3. **核心**：每个真正会 `rm` / `read` 的 use site（`clearStaleOutputFile`、`readRawResult`、`collectArtifacts`、`archiveRenamedPathArtifact`）**都重新调用 `workspacePath`** —— 即使 engine 在两次调用之间种了链接，下一次调用会看见并拒绝。

**关键认知**：TOCTOU 不是靠「更严格的检查」堵的，是靠「把 check 挪到离 use 越近越好」堵的。理想是 check 和 use 之间没有任何 attacker-controllable IO 间隔。

### Q2：Custom Engine 的凭据为什么不会泄漏到日志 / ps / Judge？

**A**：三层防御（`internal/agent/custom.go`）：

1. **不进命令行**：`buildLocalExec` 渲染完 command / args 后用 `containsAPIKey` 扫描（`custom.go:288, 297`），命中就 `errSecretInCommand` 拒绝运行 —— 否则 key 会进 `ps` 和 exec span。
2. **不进 URL**：`httpTransport.run` 用 `errSecretInURL`（`custom_http.go:51-53`）挡 URL 里的 key —— 否则会进 request log 和代理日志。
3. **结果全字段脱敏**：`maskAPIKey`（`custom.go:942-962`）对 FinalMessage / Stderr / Transcript.Content / Engine / Model **每一个流向 result.json / Judge / OTel 的字段**都做替换。配置的 API key + 所有 `engine.custom.env` 里 ≥8 字符的 value 都会被替换成 `***REDACTED***`。

**配套**：HTTP error 字符串里嵌的 URL 也过 `maskAPIKey`（`custom_http.go:142`），因为 `net/http` 的 error 会带 request URL。

### Q3：SSRF 怎么防？为什么不做内网 IP 黑名单？

**A**：artifact URL 下载（`custom.go:829-904`）用**四层组合**，不靠 IP 黑名单：

1. **scheme 白名单**：只允许 `http` / `https`（`custom.go:838-841`），挡 `file://` / `gopher://`；
2. **不跟随重定向**（`custom.go:867-872`）：`http.Client.CheckRedirect` 返回 `http.ErrUseLastResponse`，3xx 原样返回走非 2xx 分支跳过 —— 否则 attacker.com 302 跳到 `169.254.169.254` 元数据服务，会下云凭据；
3. **api-key 检查**（`custom.go:846-849`）：URL 含 key 不下载；
4. **size cap**：256MB 上限多读 1 字节判断（`custom.go:888-896`），防内存耗尽。

**不做内网 IP 黑名单的原因**：黑名单永远漏（云厂商不断新增元数据 IP、内网段、IPv6 ULA），且 engine 声明的 URL 本来就是操作者负责的（trust boundary）。组合策略把可控范围卡死在「声明的 URL + 不跳转 + 不超大」，比枚举内网段务实。

### Q4：用户如果用 wrapper 变量间接走私 key，怎么发现？

**A**：`resolveEnvRefsStrict`（`config/customengine.go:397-410`）**迭代展开**：

```go
const maxStrictExpansionDepth = 10
for range maxStrictExpansionDepth {
    resolved, err := resolveEnvRefsWith(s, true)
    // ...
    if resolved == s || !strings.Contains(resolved, "${") {
        return resolved, nil
    }
    s = resolved
}
```

用户设 `WRAPPER=${ANTHROPIC_API_KEY}`（名字不敏感），配置里写 `command: ${WRAPPER}`。第一轮展开得到 `${ANTHROPIC_API_KEY}` 字面，单次检查会通过（`WRAPPER` 不敏感）。迭代第二轮看到 `${ANTHROPIC_API_KEY}`，被 `isSensitiveEnvName` 拦下。最多 10 轮，到顶报「possible reference cycle」。

外加 `resolveEnvRefsWith` 末尾的「拼接后扫描」（`config/customengine.go:465`）：`Bearer ${PREFIX}sk-ant-real-key` 拼接完整体匹配 `looksLikeSecret`，挡住「分段走私」。

### Q5：maskAPIKey 为什么只替换 ≥8 字符的 env value？

**A**：避免**过度脱敏导致不可用**（`custom.go:947-948`）。如果 `VERBOSE=1` 这种短 env value 也替换，输出里所有 `1` 都会变成 `***REDACTED***`，整个 final_message 变成乱码，Judge LLM 看不懂、报告也读不了。8 字符是经验阈值 —— 真实 secret（API key、token）几乎都远长于 8 字符，而短 value 基本是 flag / model name / 路径片段，泄漏意义不大。这是**可用性 vs 安全性的工程权衡**。

---

## 一句话总结

> skill-up 把 Custom Engine 当成一个**返回任意 JSON、能在 workspace 里种文件、会 echo env** 的完全不可信外部组件，用**「TOCTOU 重解析 + 凭据不进命令行/URL + 结果字段全脱敏 + URL 不跟随重定向 + 多层 size cap + 配置侧严格模式防走私」**这一整套纵深防御，把路径穿越 / 凭据泄漏 / SSRF / OOM 四类威胁系统性堵死；面试讲这条线时，能讲清 TOCTOU 怎么关窗口、凭据为什么不进 ps、SSRF 为什么不做 IP 黑名单，就是合格的安全工程意识。
