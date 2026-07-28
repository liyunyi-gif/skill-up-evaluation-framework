# 06 · Runtime 沙箱三后端 — 统一接口 + none/opensandbox/docker + platform.Shell 跨平台抽象

> 「逐行深度讲解」系列第 6 篇。本篇下钻 `internal/runtime/` 整个包：一个 `Runtime` 接口（13 方法 / 4 组）+ 三个具体后端（`NoneRuntime` / `OpenSandboxRuntime` / `DockerRuntime`）+ 一层 `platform.Shell` 跨平台 shell 抽象。
> 面向：把"沙箱/隔离层"这块彻底吃透、面试能讲到底的工程师。

---

## 0. 一句话定位 + 为什么需要 Runtime 抽象

**一句话**：`Runtime` 是 skill-up 给"被测 Agent 跑命令"画的一层**统一隔离边界**——上层（evaluator / agent）只认 `Exec / Upload / Download` 这套 API，不关心底下是本机直接跑、是远端 OpenSandbox、还是本地 Docker 容器。

**为什么必须抽象**（三个硬动机，对应 Java 类比）：

1. **安全隔离**。被测 Skill 会让 Agent 跑任意 shell 命令（`rm -rf`、`curl`、`npm install` 都可能）。直接在 CI/开发机上裸跑等于把主机交给不可信代码。三种后端代表三档隔离强度：
   - `none`：本机执行，**无隔离**（靠临时目录 + 进程组兜底，主要给本地开发/CI 信任场景）
   - `docker`：本地容器，**OS 级隔离**（`--network none` 断网）
   - `opensandbox`：Alibaba 远端沙箱服务，**最强隔离 + 集中管控**

2. **跨平台**。skill-up 是 Go 项目，要跑在 Linux/macOS/**Windows** 开发机上。但被测 Skill 的脚本几乎都是 POSIX shell（`set -eu`、`if/then`、单引号引用）。Runtime 层必须在"宿主是 Windows"时也能找到 bash（Git Bash），把命令经 bash 调度，避免把 POSIX 语法喂给 `cmd.exe`。这就是 `platform.Shell` 存在的理由。

3. **测试注入**。整个评测框架要能在没有 Docker、没有 OpenSandbox 网络的单元测试里跑通。统一接口 + 几个清晰的 mock seam（`dockerCommandRunner` 函数字段、`createOpenSandbox` 包级变量）让三后端全部可 mock。**类似 Java 里 `DataSource` 接口屏蔽 H2/MySQL/PG**：业务代码只 `getConnection()`，测试注入嵌入式 H2。

> Java 类比：Runtime 接口 ≈ Spring 的 `Resource` / `TransactionManager` 抽象；三后端 ≈ Testcontainers 提供的不同容器类型；`platform.Shell` ≈ Spring 的 `Environment` profile 决定用哪套配置。

---

## 1. Runtime 接口：13 方法 / 4 组逐方法讲

接口定义在 `internal/runtime/runtime.go:115-153`（带 `//nolint:interfacebloat` 注释，作者明知 13 方法偏胖，但坚持把它作为"生命周期 + 传输 + 执行 + 元信息"的共享契约）：

```go
// runtime.go:117-118
//nolint:interfacebloat // Runtime is the shared contract for lifecycle, transfer, exec, and agent policy.
type Runtime interface {
```

### 4 组方法分工

| 组 | 方法 | 行号 | 作用 |
|---|---|---|---|
| **生命周期 (4)** | `Create(ctx)` | 119 | 建沙箱（建临时目录/建容器/建远端 sandbox） |
| | `Close() error` | 120 | 销毁沙箱（受 `cfg.Delete` 控制，可保留现场排障） |
| | `Start(ctx)` | 122 | 启动（none 无操作；docker `docker start`；opensandbox `Ping`） |
| | `Stop(ctx)` | 123 | 暂停（none 无操作；docker `docker stop`；opensandbox `Pause`） |
| **文件传输 (4)** | `UploadFile(ctx, src, dst)` | 131 | 单文件：宿主 → 沙箱 |
| | `UploadDir(ctx, srcDir, dstDir)` | 132 | 整棵目录树：宿主 → 沙箱 |
| | `DownloadFile(ctx, src, dst)` | 133 | 单文件：沙箱 → 宿主 |
| | `DownloadDir(ctx, srcDir, dstDir)` | 136 | 整棵目录树：沙箱 → 宿主 |
| **执行 (2)** | `Exec(ctx, command, opts)` | 138 | 跑一条 shell 命令，返回 `ExecResult{Stdout, Stderr, ExitCode}` |
| | `MergeEnv(env)` | 145 | 把若干环境变量合并进"持久 baseline"，后续所有 `Exec` 都能看见 |
| **元信息 (3)** | `Workspace() string` | 146 | 返回沙箱内工作目录绝对路径 |
| | `RequiresProcessSandbox() bool` | 148 | **告诉 Agent：要不要再套一层进程沙箱？** |
| | `Shell() platform.Shell` | 152 | 返回**目标 shell 描述**（注意：不是宿主 shell！） |

### 1.1 生命周期 4 方法

- **`Create` vs `Start` 为什么分开？** `Create` 是"分配资源"（一次性，幂等——见 `opensandbox.go:153-155` 的 `if r.sandbox != nil { return nil }`），`Start` 是"让资源进入运行态"。对 docker，`Create`=`docker create`、`Start`=`docker start`；对 opensandbox，`Create`=`CreateSandbox`+`ConnectSandbox`、`Start`=`Ping`；对 none，`Create`=`os.MkdirTemp`、`Start`=no-op。
- **`Close` 的 `cfg.Delete` 语义**：`Delete=false` 时**不删现场**，留给用户 `docker exec` 进去排障。这个开关在所有三后端都一致（`none.go:72-75`、`opensandbox.go:205-213`、`docker.go:226-237`），是"测试失败保留现场"的核心机制。

### 1.2 文件传输 4 方法

接口注释里有一条**强约束**（`runtime.go:125-130`）：

```go
// UploadFile and UploadDir copy host files into the runtime workspace.
// Implementations MUST preserve the source permission bits — in
// particular the executable bit — so skills that ship runnable helper
// scripts install without a chmod workaround. This holds for every
// runtime whose target filesystem supports Unix file modes (none,
// opensandbox, docker).
```

**"必须保留权限位"是契约级别的要求**。原因：很多 Skill 仓库带可执行 helper 脚本（`scripts/build.sh`），如果上传后 executable bit 丢了，Agent 跑它会报 `permission denied`，需要额外 `chmod +x` 兜底——这是典型的"抽象泄漏"。三后端都遵守这条（none 在 `none.go:111-120`、opensandbox 在 `opensandbox.go:347-349` 把 `info.Mode()` 作为 metadata 传给 SDK、docker 靠 `docker cp` 自带权限保留）。

### 1.3 执行：`Exec` + `MergeEnv`

`Exec` 是整个框架**调用频次最高**的方法（setup_steps、agent install、judge 脚本、fixture 准备全走它）。签名：

```go
// runtime.go:138
Exec(ctx context.Context, command string, opts ExecOptions) (ExecResult, error)
```

`ExecOptions`（`runtime.go:93-104`）：

```go
type ExecOptions struct {
    Cwd         string               // 工作目录（相对 workspace 或绝对）
    Env         map[string]string    // 本次调用的临时 env（覆盖 baseline）
    TimeoutSec  int                  // 本次超时
    ArtifactDir string               // 产物落盘目录
    AgentMetadata *AgentMetadata     // 给 Custom Engine 用的 case 级元数据
}
```

**`MergeEnv` 的设计动机**（`runtime.go:139-145` 注释讲得很清楚）：evaluator 探测出 Agent 在目标 shell 下展开后的 `PATH`，想把它塞进所有后续 `Exec` 的环境里，但又不想每个 `Exec` 调用方都记得传。于是有"持久 baseline"（`cfg.Env`）这层：`MergeEnv` 写进 baseline，`Exec` 时再叠加 `opts.Env`，`opts.Env` 优先级最高。

合并逻辑实现在 `mergeEnv`（`runtime.go:28-38`）：

```go
func mergeEnv(persistentEnv, callEnv map[string]string) []string {
    envMap := envMapFromList(os.Environ())  // 1. 主机全量 env 做底
    maps.Copy(envMap, persistentEnv)         // 2. baseline 覆盖
    maps.Copy(envMap, callEnv)               // 3. 本次 callEnv 最后覆盖（优先级最高）
    ...
}
```

> 注意：none runtime 走 `mergeEnv`（带主机 env），docker/opensandbox 走 `overlayEnvList` / `mergeEnvMaps`（**只传 cfg.Env + opts.Env，不带主机 env**，见 `docker.go:417-424` 注释——这是为了不污染容器的 PATH/HOME）。

### 1.4 元信息：`Workspace` / `RequiresProcessSandbox` / `Shell`

这三个是**只读探针**，上层据此调整行为。

**`RequiresProcessSandbox()`** 语义（接口注释 `runtime.go:147-148`）：

```go
// RequiresProcessSandbox reports whether agents should enable their own process sandbox.
```

三后端的返回值差异是核心：

| 后端 | 返回值 | 含义 |
|---|---|---|
| `NoneRuntime` (`none.go:380-382`) | **`true`** | 命令在主机直接跑，Agent 必须自己再套一层进程沙箱（如 Codex 的 sandbox）兜底 |
| `OpenSandboxRuntime` (`opensandbox.go:551-553`) | `false` | 远端容器已隔离，Agent 不必重复套 |
| `DockerRuntime` (`docker.go:498-500`) | `false` | 本地容器已隔离，同上 |

谁消费这个值？Codex/Qwen Code 等 Agent 在决定要不要开自己的进程沙箱时调它（`internal/agent/codex.go:311`、`internal/agent/qwen_code.go:152`）。这是典型的"分层防御"——每层独立判断是否需要再隔离一次。

**`Shell()`** 语义（接口注释 `runtime.go:149-152` 极重要）：

```go
// Shell describes the target command interpreter used by Exec. The target
// may differ from the skill-up host (for example, a Linux container on a
// Windows host), so callers must not re-derive it from platform.Host.
```

**关键陷阱**：`Shell()` 返回的是**目标 shell**（命令实际跑的那个 shell），不是 skill-up 进程所在主机的 shell。比如你在 Windows 笔记本上跑 skill-up 评测一个 Docker 镜像，`platform.Host()` 是 Windows bash/cmd，但 `DockerRuntime.Shell()` 永远返回 `Linux/POSIX`（`docker.go:251-253`），因为命令是在 Linux 容器里执行的。上层做参数引用（quote）时**必须**用 `rt.Shell()` 而不是 `platform.Host()`，否则会把 Windows 风格引用喂给 Linux bash。

### 1.5 工厂 switch：`NewRuntime`（runtime.go:216-238）

```go
// runtime.go:216-238
func NewRuntime(cfg Config) (Runtime, error) {
    var (
        rt  Runtime
        err error
    )
    switch cfg.Type {
    case "none":
        rt = &NoneRuntime{cfg: cfg}
    case "opensandbox":
        rt, err = NewOpenSandboxRuntime(cfg)
    case "docker":
        rt, err = NewDockerRuntime(cfg)
    default:
        return nil, errors.New("unknown runtime type: " + cfg.Type)
    }
    if err != nil {
        return nil, err
    }
    // ★ 构造期立刻校验 Shell，坏配置 fail-fast
    if err := rt.Shell().Validate(); err != nil {
        return nil, fmt.Errorf("invalid %s runtime shell: %w", cfg.Type, err)
    }
    return rt, nil
}
```

两个设计点：

1. **`switch` 而非 map/registry**：三后端是闭集，未来不会动态扩展，用 switch 最直白。这跟 skill-up 里 Agent 工厂的风格一致。
2. **构造期 `rt.Shell().Validate()`（L234-236）**：拿到 Runtime 实例后**立刻**校验它的目标 shell 描述合法（比如 Windows 上配了个 cmd 但又填了 `BashPath`，会被 `platform.Shell.Validate` 直接拒掉）。**fail-fast 原则**——坏配置在 `NewRuntime` 就报错，而不是等到第一次 `Exec` 才崩。Java 类比：类似 Spring 在 `ApplicationContext.refresh()` 阶段就把 bean 配置错误暴露，不等到第一次注入使用。

### 1.6 `Config` 结构（runtime.go:166-188）

```go
type Config struct {
    Type           string         // "none" / "opensandbox" / "docker"
    Image          string         // docker/opensandbox 镜像
    WorkspaceMount string         // 容器内工作目录（默认 /workspace）
    Env            map[string]string  // 持久 baseline env
    SetupSteps     []SetupStep    // 启动后要跑的初始化命令

    SandboxTemplate string        // opensandbox 模板
    UseServerProxy  bool
    ReadyTimeout    time.Duration
    SandboxTimeout  time.Duration
    Entrypoint      []string      // 容器 entrypoint 覆盖
    Metadata        map[string]string
    Kwargs          map[string]string  // 后端专属参数（base_url 等）

    NetworkPolicy string   // deny_all / allow_declared
    AllowedEgress []string // allow_declared 时的白名单 FQDN

    SkillPath string
    Delete    bool          // 是否在 Close 时销毁沙箱
}
```

注意 `Kwargs` 这个字段——它是"后端专属参数的逃生舱"。opensandbox 的 `base_url` / `api_key` / `extensions` / `parallelism` 全塞在这里（见 `opensandbox.go:34-37` 常量），避免给共享 `Config` 加一堆 opensandbox 专属字段污染接口。这是**接口稳定性 vs 配置扩展性**的典型折中。

---

## 2. NoneRuntime：本机执行 + 临时目录隔离

定位（`none.go:43-47`）：

```go
// NoneRuntime executes commands directly on the host with an isolated temp workspace.
type NoneRuntime struct {
    cfg       Config
    workspace string
}
```

**没有进程级隔离**，靠两点兜底：(1) 每次跑用独立临时目录避免文件串；(2) 进程组 + SIGTERM/SIGKILL 保证超时命令能被收掉。仅用于信任场景（本地开发、CI、被测 Skill 已知安全）。

### 2.1 `Create`：建隔离临时目录（none.go:57-65）

```go
func (r *NoneRuntime) Create(ctx context.Context) error {
    dir, err := os.MkdirTemp("", "skill-up-*")
    if err != nil {
        return fmt.Errorf("failed to create temp dir: %w", err)
    }
    r.workspace = dir
    return nil
}
```

`os.MkdirTemp("", "skill-up-*")` 在系统 temp 目录（Linux `/tmp`、Windows `%TEMP%`）下建一个 `skill-up-<随机>` 目录。每个 case 一个独立目录，互相隔离。**Java 类比**：类似 `Files.createTempDirectory("skill-up-")`。

### 2.2 `Close`：受 `cfg.Delete` 控制（none.go:68-77）

```go
func (r *NoneRuntime) Close() error {
    if r.workspace == "" {
        return nil
    }
    if !r.cfg.Delete {
        logging.Debugf("NoneRuntime.Close: skipping cleanup, workspace preserved at %s", r.workspace)
        return nil
    }
    return os.RemoveAll(r.workspace)
}
```

`Delete=false` 时**保留工作区**，方便你手动进去看 Agent 改了哪些文件。这条"排障模式"贯穿三后端。

### 2.3 `UploadFile`：保留权限位（none.go:97-121）

```go
func (r *NoneRuntime) UploadFile(ctx context.Context, sourcePath, targetPath string) error {
    info, err := os.Stat(sourcePath)         // 1. 取源文件 mode
    if err != nil { return ... }
    data, err := os.ReadFile(sourcePath)
    if err != nil { return ... }
    target := r.pathInWorkspaceOrAbs(targetPath)
    if err := os.MkdirAll(filepath.Dir(target), noneDirMode); err != nil { return ... }

    mode := info.Mode().Perm()
    //nolint:gosec // target is joined with workspace; info.Mode() preserves source permissions
    if err := os.WriteFile(target, data, mode); err != nil { return err }
    // WriteFile only applies the mode when it creates the file and is subject to
    // the process umask; Chmod makes the executable bit deterministic even when
    // re-installing over an existing file or under a restrictive umask.
    //nolint:gosec // target is joined with workspace
    return os.Chmod(target, mode)
}
```

**为什么 `WriteFile` 之后还要 `Chmod`？** 注释（L116-118）讲得很细：`os.WriteFile` 只在**创建**文件时应用 mode，而且会被进程 umask 削减。如果文件已存在（重装），`WriteFile` **根本不改 mode**。所以必须显式 `Chmod` 才能让 executable bit 在重装场景下也确定性地保留。这是非常隐蔽的 bug 防御——典型"经验之谈"，面试可以拿来展开。

`pathInWorkspaceOrAbs`（L35-41）的逻辑：相对路径 join 到 workspace 下，绝对路径原样写——允许调用方写到 workspace 外（比如系统的 `/tmp/fixture`）。

### 2.4 `Exec`：核心方法逐段（none.go:200-283）

```go
func (r *NoneRuntime) Exec(ctx context.Context, command string, opts ExecOptions) (ExecResult, error) {
    ctx, span := observability.Tracer().Start(ctx, "runtime.exec")  // OpenTelemetry 追踪
    defer span.End()
    startTime := time.Now()

    // ① 选 shell —— platform.Host() 决定本机用什么 shell 调度
    shell := platform.Host()
    cmd := shell.Cmd(ctx, command)
    // ② 进程组配置（POSIX 才有，Windows no-op）
    done := make(chan struct{})
    configureProcessGroup(cmd, done)
    // ③ WaitDelay：ctx 取消后最多等 2s 强关 stdio 管道
    cmd.WaitDelay = noneExecWaitDelay
    if opts.Cwd != "" {
        cmd.Dir = opts.Cwd
    } else {
        cmd.Dir = r.workspace                       // 默认工作目录 = 临时 workspace
    }
    // ④ 合并 env：主机 env ← cfg.Env ← opts.Env ← shell.Env
    env := mergeEnv(r.cfg.Env, opts.Env)
    env = append(env, shell.Env...)                 // MSYS_NO_PATHCONV 之类
    cmd.Env = env

    var stdout, stderr bytes.Buffer
    cmd.Stdout = &stdout
    cmd.Stderr = &stderr

    // ⑤ 分类错误：把 *exec.ExitError / ctx.Err / ErrWaitDelay 翻译成 (exitCode, err)
    exitCode, execErr := classifyExecError(ctx, runCmd(cmd, done))
    ...
}
```

**逐行要点**：

- **`platform.Host()`（L205）**：这是 NoneRuntime 选 shell 的核心。它返回一个 `HostShell`，其中 `Cmd` 字段是个闭包，已经把"用 bash 还是 cmd、带哪些 env"全封装好了。下一节细讲。
- **`cmd.WaitDelay = noneExecWaitDelay`（L216）**：Go 1.20+ 的 `exec.Cmd.WaitDelay` 字段。含义：ctx 取消后，`Wait` 最多再等这么久（2s）就强关子进程的 stdio 管道。**为什么需要**：注释（L24-29）解释——Windows 上没有进程组，Git Bash 的孙进程（`ping`/`sleep`/`git`）可能持有 stderr 管道不让 `Wait` 返回，`WaitDelay` 兜底强关管道让 pipe-reader 协程解锁。
- **`env = append(env, shell.Env...)`（L231）**：shell 自己需要的 env（如 Git Bash 要 `MSYS_NO_PATHCONV=1` 防止 MSYS 把 `/x` 改写）在这里一次性追加。这个决策集中在一处，避免散落到各调用方。

### 2.5 进程组：`configureProcessGroup`（none_exec_unix.go / none_exec_other.go）

这是个**用 build tag 切换**的双实现文件：

```go
// none_exec_unix.go:1
//go:build unix

// none_exec_other.go:1
//go:build !unix
```

`unix` build tag 在 Linux/macOS/BSD 上为真，Windows 上为假。所以编译期就决定走哪份实现。

**POSIX 版（none_exec_unix.go:31-58）**：

```go
func configureProcessGroup(cmd *exec.Cmd, done <-chan struct{}) {
    cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}  // ① 让子进程成为新进程组 leader
    cmd.Cancel = func() error {                            // ② ctx 取消时调
        if cmd.Process == nil { return nil }
        pid := cmd.Process.Pid
        // 负 PID = 给整个进程组发信号
        if err := signalGroupOrProcess(cmd, pid, syscall.SIGTERM); err != nil { return err }
        // ③ 起一个独立 goroutine，1s 后还没退出就 SIGKILL
        go func() {
            timer := time.NewTimer(noneExecKillGrace)  // 1s
            defer timer.Stop()
            select {
            case <-done:                               // 进程已退出，停止升级
            case <-timer.C:
                _ = signalGroupOrProcess(cmd, pid, syscall.SIGKILL)
            }
        }()
        return nil
    }
}
```

**这段是面试黄金矿**。逐行讲为什么：

1. **`Setpgid: true`（L32）**：让子进程成为一个新进程组的 leader（PGID = 子 PID）。Go 默认是 `exec.CommandContext` 只杀直接子进程，**不杀孙进程**。但 shell 命令通常是 `bash -c "sleep 100"`——`sleep` 是 `bash` 的子进程。如果只杀 `bash`，`sleep` 成孤儿继续跑，工作区文件可能被它继续改。`Setpgid` 让整个树成为一个组，可以被一起信号。

2. **`signalGroupOrProcess` 用负 PID（none_exec_unix.go:62-67）**：

   ```go
   func signalGroupOrProcess(cmd *exec.Cmd, pid int, sig syscall.Signal) error {
       if err := syscall.Kill(-pid, sig); err != nil {  // ★ 负 PID = 整组
           return cmd.Process.Signal(sig)               // 失败回退到只发主进程
       }
       return nil
   }
   ```

   `syscall.Kill(-pid, sig)` 是 POSIX 规范——负 PID 表示"给 PGID == pid 的整个进程组发信号"。**Java 类比**：类似 Linux 命令 `kill -TERM -<pgid>`。

3. **SIGTERM → SIGKILL 升级（L47-55）**：先发 SIGTERM 给 1s 优雅退出机会（让 trap handler 跑、释放锁），超时还没死就 SIGKILL 强杀。**`noneExecKillGrace = 1s`**（L18）必须 **< `noneExecWaitDelay = 2s`**（none.go:30），保证升级在 `WaitDelay` 强关管道前完成。注释（L13-18）专门讲了这两个常量的耦合关系——典型的"魔法数字必须协同"。

4. **`done` channel 防 PID 复用（L47-51 的 select）**：`runCmd`（none.go:289-295）在 `Wait` 返回后 `close(done)`，于是升级 goroutine 立刻退出，**不会**在 PID 被内核回收并复用给别的进程后还发 SIGKILL。这是一个**TOCTOU（time-of-check-time-of-use）竞态**的优雅防御：

   ```go
   // none.go:289-295
   func runCmd(cmd *exec.Cmd, done chan<- struct{}) error {
       defer close(done)   // ★ Wait 返回后立刻 close，停止升级定时器
       if err := cmd.Start(); err != nil { return err }
       return cmd.Wait()
   }
   ```

**Windows 版（none_exec_other.go:8）**：

```go
//go:build !unix
func configureProcessGroup(_ *exec.Cmd, _ <-chan struct{}) {}
```

直接 no-op。Windows 没有 POSIX 进程组语义，靠 `cmd.WaitDelay`（2s）兜底——强关 stdio 管道，让被卡住的子进程最终随管道关闭被回收。这是个**已知妥协**，注释（none.go:24-29）坦白说了。

### 2.6 `classifyExecError`：四态错误分类（none.go:307-334）

这是 NoneRuntime 里**最值得拿出来讲**的函数。它把 `*exec.Cmd.Run()` 的错误翻译成 `(exitCode, error)` 二元组，供上层判断"是命令本身失败（评分 FAIL）还是基础设施故障（重试）"。

```go
// Precedence (matches the legacy inline behaviour):
//
//	nil err                            → (0, nil)              // 正常退出 0
//	*exec.ExitError + ctx.Err() == nil → (exitCode, nil)       // 命令非零退出，业务失败
//	*exec.ExitError + ctx.Err() != nil → (exitCode, ctxErr)    // 被 ctx 杀，但已包了 ExitError
//	exec.ErrWaitDelay (no ExitError)   → (0, nil)              // 进程正常退 0，只是管道滞后
//	non-ExitError                      → (-1, ctxErr or err)   // 启动失败之类
func classifyExecError(ctx context.Context, runErr error) (int, error) {
    if runErr == nil {
        return 0, nil
    }
    // ★ ctx 取消时强制返回 -1，无论 OS 报什么 exit code
    if ctxErr := ctx.Err(); ctxErr != nil {
        return -1, ctxErr
    }
    var exitErr *exec.ExitError
    if errors.As(runErr, &exitErr) {
        return exitErr.ExitCode(), nil
    }
    // ErrWaitDelay：进程已正常退出 0，只是孙进程还持着管道
    if errors.Is(runErr, exec.ErrWaitDelay) {
        return 0, nil
    }
    return -1, runErr
}
```

**关键设计点**——`ctx.Err() != nil` 时强制 `-1`（L317-319）。注释（L311-316）讲了为什么：POSIX 上被信号杀的 `sleep` 报 -1（信号），但 **Windows 上 Git Bash 的 `sleep 1` 被 ctx 杀会报 exit 1**——和"命令正常失败"无法区分。所以这里**无视 OS 报的 exit code**，只要 ctx 取消了就强制 `-1`。`-1 + ctxErr` 是上层（`Exec` 的 switch，L247-269）识别"被超时杀"的统一信号。

**`exec.ErrWaitDelay` 分支（L330-332）**：Go 1.20+ 引入。`WaitDelay` 触发强关管道后，`Wait` 会返回一个包装了 `ErrWaitDelay` 的错误，但**进程本身可能已经正常退 0**——只是孙进程还抱着管道。这种情况应判为成功（`return 0, nil`），否则一个本来成功的 Agent 跑因为孙进程滞后就被误判失败。注释（L324-329）讲得很细。

> Java 类比：这相当于你在 `Process.waitFor()` 之后还要区分"进程正常 exit"、"进程被 `destroyForcibly` 杀"、"Future.cancel 触发的中断"——Go 这里用四种错误类型表达，比 Java 的 `InterruptedException` 单一信号更精细。

### 2.7 `Exec` 的日志 switch（none.go:247-280）

```go
switch {
case result.ExitCode == -1 && ctx.Err() != nil:
    // 被超时杀，专门记 "deadline elapsed by Xs"
    reason := ctx.Err().Error()
    if errors.Is(ctx.Err(), context.DeadlineExceeded) {
        if deadline, ok := ctx.Deadline(); ok {
            reason = fmt.Sprintf("%s (deadline elapsed by %s)", reason, time.Since(deadline).Round(time.Millisecond))
        }
    }
    logging.ErrorContextf(ctx, "command killed by context (%s); command: %s", reason, maskCommand(command))
    ...
case result.ExitCode != 0:
    logNonZeroExit(...)   // 普通非零退出
case result.Stderr != "":
    logging.WarnContextf(ctx, "stderr: %s", result.Stderr)  // 退 0 但有 stderr，降级 warn
}
```

**两个细节**：

- **`maskCommand`（none.go:336-343）**：日志里截断超 200 rune 的命令，避免巨型脚本刷爆日志。
- **`logNonZeroExit/Stderr/Stdout` 按 exit code 分级（none.go:345-372）**：`exit 1` 算 `Warn`（很多工具用 1 表示"有警告但能跑"），其他非零算 `Error`。**stdout 也被记录**——注释（L362-366）解释：很多工具（构建器、测试框架）的诊断信息写到 stdout 而非 stderr，丢了就让用户只剩 exit code 和空 stderr，没法排错。

---

## 3. OpenSandboxRuntime：对接 Alibaba OpenSandbox HTTP API

定位（`opensandbox.go:131-140`）：把命令送到 Alibaba OpenSandbox 远端服务跑，最强隔离 + 集中化镜像管理。

```go
type OpenSandboxRuntime struct {
    cfg             Config
    sandbox         openSandboxClient   // 接口！测试可 mock
    workspace       string
    baseURL         string
    extensions      map[string]string
    requestTimeout  time.Duration
    fileParallelism int
}
```

### 3.1 `openSandboxClient` 接口 + mock seam（opensandbox.go:50-63）

```go
type openSandboxClient interface {
    ID() string
    Close() error
    Kill(ctx context.Context) error
    Pause(ctx context.Context) error
    Ping(ctx context.Context) error
    CreateDirectory(ctx context.Context, remotePath string, mode int) error
    UploadFiles(ctx context.Context, entries []opensandbox.UploadFileEntry) error
    DownloadFile(ctx context.Context, remotePath, rangeHeader string, opts ...opensandbox.DownloadFileOptions) (io.ReadCloser, error)
    SearchFiles(ctx context.Context, dir, pattern string) ([]opensandbox.FileInfo, error)
    RunCommandWithOpts(ctx context.Context, req opensandbox.RunCommandRequest, handlers *opensandbox.ExecutionHandlers) (*opensandbox.Execution, error)
}

// ★ 包级变量，测试可替换
var createOpenSandbox = createOpenSandboxCompat
```

**两层可测试性设计**：

1. `openSandboxClient` 是个**接口**（生产代码用真实 SDK，测试用 fake 实现）。
2. `createOpenSandbox` 是个**包级函数变量**（`var`，不是 `const`），测试里 `t.Cleanup` 替换成假工厂（见 `opensandbox_test.go:59-66`）。

```go
// 测试里的替换模式
origCreate := createOpenSandbox
t.Cleanup(func() { createOpenSandbox = origCreate })
createOpenSandbox = func(ctx, cfg, opts) (openSandboxClient, error) {
    return &fakeSandbox{...}, nil
}
```

**Java 类比**：这相当于 Spring 里 `@MockBean` 替换 `@Service`——但 Go 没有 DI 容器，靠"接口 + 包级变量"两件套实现等效效果。这是个**值得在面试里讲的"Go 风格可测试设计"**。

### 3.2 `createOpenSandboxCompat`：Create + Connect 轮询（opensandbox.go:65-112）

```go
func createOpenSandboxCompat(ctx context.Context, cfg opensandbox.ConnectionConfig, opts opensandbox.SandboxCreateOptions) (openSandboxClient, error) {
    if opts.Image == "" {
        return nil, errors.New("opensandbox image is required")
    }
    entrypoint := opts.Entrypoint
    if len(entrypoint) == 0 { entrypoint = opensandbox.DefaultEntrypoint }
    limits := opts.ResourceLimits
    if limits == nil { limits = opensandbox.DefaultResourceLimits }
    timeout := opts.TimeoutSeconds
    if opts.ManualCleanup {
        timeout = nil                          // ★ 手动清理模式不设超时
    } else if timeout == nil {
        t := opensandbox.DefaultTimeoutSeconds
        timeout = &t
    }

    // ① 通过 LifecycleClient 发 CreateSandbox HTTP 请求
    lifecycle := newOpenSandboxLifecycleClient(cfg)
    created, err := lifecycle.CreateSandbox(ctx, opensandbox.CreateSandboxRequest{
        Image:          &opensandbox.ImageSpec{URI: opts.Image, Auth: opts.ImageAuth},
        Entrypoint:     entrypoint,
        ResourceLimits: limits,
        Timeout:        timeout,
        Env:            opts.Env,
        Metadata:       opts.Metadata,
        NetworkPolicy:  opts.NetworkPolicy,
        Volumes:        opts.Volumes,
        Extensions:     opts.Extensions,
    })
    if err != nil { return nil, fmt.Errorf("opensandbox: create sandbox: %w", err) }

    // ② 轮询直到 sandbox ready（带超时）
    readyOpts := opensandbox.ReadyOptions{
        Timeout:         opts.ReadyTimeout,
        PollingInterval: opts.HealthCheckInterval,
        HealthCheck:     opts.HealthCheck,
    }
    sb, err := opensandbox.ConnectSandbox(ctx, cfg, created.ID, readyOpts)
    if err != nil {
        // ★ ready 失败：用 detach 的 ctx 删掉刚建的，避免泄漏
        _ = lifecycle.DeleteSandbox(context.WithoutCancel(ctx), created.ID)
        return nil, err
    }
    return sb, nil
}
```

**三个设计点**：

1. **`LifecycleClient` 单独建（L85）**：`newOpenSandboxLifecycleClient` 把 base_url、auth header、timeout、retry 等可配置项统一在一处组装。`cfg.GetBaseURL()+"/"+opensandbox.APIVersion` 拼版本化 URL（L128），保证 API 版本可演进。
2. **`CreateSandbox` → `ConnectSandbox` 轮询**：远端建容器是异步的，`Create` 立刻返回 ID，但容器要等一段时间才 ready。`ConnectSandbox` 内部轮询健康检查直到 ready 或 `ReadyTimeout` 超时。
3. **`context.WithoutCancel(ctx)` 善后（L108）**：如果 `ConnectSandbox` 失败（比如超时），必须把刚建的 sandbox 删掉避免泄漏。但**用原 ctx 会被一起取消**，导致 `DeleteSandbox` 也跑不动。`context.WithoutCancel`（Go 1.21+）派生一个**不被父 ctx 取消**的 ctx，让善后能正常跑。这是个值得记的模式。

### 3.3 `configure`：解析 base_url / api_key / extensions / parallelism（opensandbox.go:576-616）

```go
func (r *OpenSandboxRuntime) configure(ctx context.Context) {
    r.baseURL = r.resolveBaseURL()
    r.requestTimeout = r.resolveRequestTimeout()
    r.extensions = r.resolveExtensions(ctx)
    r.fileParallelism = r.resolveFileTransferParallelism()
}

func (r *OpenSandboxRuntime) resolveBaseURL() string {
    if baseURL := strings.TrimSpace(r.cfg.Kwargs[openSandboxKwargBaseURL]); baseURL != "" {
        return baseURL                                // kwargs 优先
    }
    return os.Getenv(openSandboxBaseURLEnv)            // 其次环境变量
}

func (r *OpenSandboxRuntime) resolveFileTransferParallelism() int {
    raw := strings.TrimSpace(r.cfg.Kwargs[openSandboxKwargParallelism])
    if raw == "" { return openSandboxDefaultFileTransferParallelism }  // 默认 8
    parallelism, err := strconv.Atoi(raw)
    if err != nil || parallelism <= 0 {
        return openSandboxDefaultFileTransferParallelism
    }
    return min(parallelism, 64)                        // ★ 上限 64，防止滥用
}
```

**配置优先级**：`Kwargs[...]` > 环境变量 > 默认值。这是个清晰的"配置层级"，类似 Java 里 Spring Boot 的 `application.yml` > 环境变量 > 默认。

`resolveExtensions`（L650-670）解析 `extensions` JSON 字符串：

```go
func parseExtensions(ctx context.Context, source, raw string) map[string]string {
    if raw == "" { return nil }
    extensions := map[string]string{}
    if err := json.Unmarshal([]byte(raw), &extensions); err != nil {
        logging.WarnContextf(ctx, "%s is not valid JSON object, ignoring: %v", source, err)
        return nil                                     // ★ 解析失败降级为 nil，不致命
    }
    if len(extensions) == 0 { return nil }
    return extensions
}
```

JSON 解析失败**只 warn 不报错**——extensions 是可选增强，不应让一次配置手抖搞崩整个评测。这是个"**优雅降级**"哲学。

### 3.4 `Create`：编排（opensandbox.go:152-193）

```go
func (r *OpenSandboxRuntime) Create(ctx context.Context) error {
    if r.sandbox != nil { return nil }              // 幂等
    r.configure(ctx)
    image := strings.TrimSpace(r.cfg.Image)
    if image == "" { image = strings.TrimSpace(r.cfg.SandboxTemplate) }
    if image == "" { image = openSandboxDefaultImage }  // ""
    if r.baseURL == "" {
        return errors.New("opensandbox runtime requires environment.kwargs.base_url or OPENSANDBOX_BASE_URL")
    }
    if openSandboxAuthKey() == "" {
        return errors.New("opensandbox runtime requires OPENSANDBOX_API_KEY")
    }
    if image == "" {
        return errors.New("opensandbox runtime requires environment.image or environment.sandbox_template to be set")
    }
    sb, err := createOpenSandbox(ctx, r.connectionConfig(), opensandbox.SandboxCreateOptions{
        Image:          image,
        Entrypoint:     r.entrypoint(),
        TimeoutSeconds: durationSecondsPtr(r.cfg.SandboxTimeout),
        Env:            r.cfg.Env,
        Metadata:       r.cfg.Metadata,
        Extensions:     r.extensions,
        ReadyTimeout:   r.cfg.ReadyTimeout,
        NetworkPolicy:  r.networkPolicy(),
    })
    if err != nil { return fmt.Errorf("failed to create opensandbox: %w", err) }
    r.sandbox = sb

    // 建工作区目录，失败要回滚销毁 sandbox
    if err := r.ensureDirectory(ctx, r.workspace, 755); err != nil {
        _ = r.cleanupCreatedSandbox(ctx)
        return fmt.Errorf("failed to create opensandbox workspace %s: %w", r.workspace, err)
    }
    return nil
}
```

**注意 `openSandboxDefaultImage = ""`**（L32-33）的注释特别讲为什么：

```go
// openSandboxDefaultImage is intentionally empty: users must supply a
// concrete image via environment.image / environment.sandbox_template or
// the config schema. Shipping a vendor-specific default leaks internal
// infrastructure to OSS users and gives a confusing failure mode because
// the image would not be reachable from their network.
```

**不内置默认镜像**——内置的话 OSS 用户会拿到一个他们网络根本访问不到的内部镜像，错误信息会很迷惑。宁可让用户显式配。这是个"OSS 项目国际化"的细节考量。

### 3.5 `networkPolicy` 映射（opensandbox.go:626-648）

```go
func (r *OpenSandboxRuntime) networkPolicy() *opensandbox.NetworkPolicy {
    switch strings.ToLower(strings.TrimSpace(r.cfg.NetworkPolicy)) {
    case "deny_all":
        return &opensandbox.NetworkPolicy{DefaultAction: "deny"}
    case "allow_declared":
        // ★ 默认拒绝，只放行声明的目标——空白名单 = 全断网
        policy := &opensandbox.NetworkPolicy{DefaultAction: "deny"}
        for _, target := range r.cfg.AllowedEgress {
            t := strings.TrimSpace(target)
            if t == "" { continue }
            policy.Egress = append(policy.Egress, opensandbox.NetworkRule{
                Action: "allow",
                Target: t,
            })
        }
        return policy
    default:
        return nil                                     // nil = 服务端默认策略
    }
}
```

**安全设计**：`allow_declared` 模式下**默认拒绝**，只把 `AllowedEgress` 列表里的目标加成 allow。**空白名单 = 全断网**，不是"全放行"。这是"白名单优先于黑名单"的安全原则。

### 3.6 `Exec`：调远端命令（opensandbox.go:482-543）

```go
func (r *OpenSandboxRuntime) Exec(ctx context.Context, command string, opts ExecOptions) (ExecResult, error) {
    if err := r.ensureCreated(); err != nil { return ExecResult{}, err }
    ctx, span := observability.Tracer().Start(ctx, "runtime.exec")
    defer span.End()
    startTime := time.Now()
    env := mergeEnvMaps(r.cfg.Env, opts.Env)              // ★ 不带主机 env

    req := opensandbox.RunCommandRequest{
        Command: command,
        Cwd:     r.execCwd(opts.Cwd),
        Timeout: int64(opts.TimeoutSec) * 1000,            // 毫秒
        Envs:    env,
    }
    exec, err := r.sandbox.RunCommandWithOpts(ctx, req, nil)
    result := executionToResult(exec)
    // ★ 区分两种"看起来 exit -1"的情况
    sandboxCallFailed := err != nil || exec == nil
    if sandboxCallFailed && result.ExitCode == 0 {
        result.ExitCode = -1                              // SDK 失败强制 -1
    }
    ...
}
```

**关键逻辑**（L512-515）：区分两种失败：

- **(a) SDK 调用本身失败 / 返回 nil**——远端命令可能根本没跑，没有真实 exit code。强制 `-1`，避免误导日志。
- **(b) 远端命令跑了但非零退出**——保留真实 exit code，按业务失败处理。

`executionToResult`（L766-785）把 SDK 的 `*opensandbox.Execution` 转成统一 `ExecResult`：

```go
func executionToResult(exec *opensandbox.Execution) ExecResult {
    if exec == nil { return ExecResult{ExitCode: -1} }
    result := ExecResult{
        Stdout:   joinOutputMessages(exec.Stdout),
        Stderr:   joinOutputMessages(exec.Stderr),
        ExitCode: 0,                                     // 默认 0
    }
    if exec.ExitCode != nil { result.ExitCode = *exec.ExitCode }
    if exec.Error != nil && result.Stderr == "" {
        result.Stderr = strings.Join(exec.Error.Traceback, "\n")
        if result.Stderr == "" { result.Stderr = exec.Error.Value }
    }
    return result
}
```

`joinOutputMessages`（L838-849）把 SDK 返回的多条 output message 拼成一个字符串（预算 total 一次性 `Grow`，避免反复扩容）。

### 3.7 批量上传：控 fd（opensandbox.go:305-358）

```go
const openSandboxUploadBatchSize = 128

func (r *OpenSandboxRuntime) uploadFiles(ctx context.Context, items []uploadItem) error {
    for start := 0; start < len(items); start += openSandboxUploadBatchSize {
        end := min(start+openSandboxUploadBatchSize, len(items))
        if err := r.uploadFileBatch(ctx, items[start:end]); err != nil {
            return err
        }
    }
    return nil
}

func (r *OpenSandboxRuntime) uploadFileBatch(ctx context.Context, items []uploadItem) error {
    if len(items) == 0 { return nil }
    entries := make([]opensandbox.UploadFileEntry, 0, len(items))
    closers := make([]io.Closer, 0, len(items))
    defer func() {
        for _, c := range closers { _ = c.Close() }      // ★ 批结束统一关 fd
    }()

    for _, item := range items {
        file, err := os.Open(item.source)                // 同时打开 128 个
        if err != nil { return ... }
        closers = append(closers, file)
        info, err := file.Stat()
        if err != nil { return ... }
        entries = append(entries, opensandbox.UploadFileEntry{
            File: file,
            Options: opensandbox.UploadFileOptions{
                FileName: filepath.Base(item.source),
                Metadata: opensandbox.FileMetadata{
                    Path: item.target,
                    Mode: opensandbox.OctalMode(info.Mode()),  // ★ 保留权限位
                },
            },
        })
    }
    if err := r.sandbox.UploadFiles(ctx, entries); err != nil { return ... }
    return nil
}
```

**为什么要分批？** 注释（L46-48）：每批最多同时打开 128 个文件，**控制同时打开的 fd 数量在 `ulimit -n` 之下**。一次性传几千个 Skill 仓库文件会把进程 fd 表撑爆。每批结束用 `defer` 统一关掉这批的 fd。

`Mode: opensandbox.OctalMode(info.Mode())` 把源文件权限位作为 metadata 传给 SDK——这呼应了 §1.2 说的"必须保留权限位"契约。

### 3.8 并发下载：worker pool + fail-fast（opensandbox.go:411-464）

```go
func (r *OpenSandboxRuntime) downloadFiles(ctx context.Context, source, targetDir string, files []opensandbox.FileInfo) error {
    if len(files) == 0 { return nil }
    parallelism := r.fileParallelism
    if parallelism <= 0 { parallelism = openSandboxDefaultFileTransferParallelism }  // 8
    parallelism = min(parallelism, len(files))

    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    jobs := make(chan opensandbox.FileInfo)
    errCh := make(chan error, 1)                          // ★ buffer 1，第一个错误就 cancel
    var wg sync.WaitGroup
    for range parallelism {
        wg.Go(func() {                                    // Go 1.25+ 的 wg.Go
            for file := range jobs {
                if err := r.downloadSearchResult(ctx, source, targetDir, file); err != nil {
                    select {
                    case errCh <- err:                    // 第一个错误塞进去
                        cancel()                          // 然后立刻 cancel 整个 ctx
                    default:                              // 后续错误直接丢弃（已经有 1 个了）
                    }
                    return
                }
            }
        })
    }

    for _, file := range files {
        select {
        case <-ctx.Done():                                // 已失败，停止投递
            close(jobs)
            wg.Wait()
            select {
            case err := <-errCh: return err
            default: return ctx.Err()
        case jobs <- file:                                // 投递任务
        }
    }
    close(jobs)
    wg.Wait()
    select {
    case err := <-errCh: return err
    default: return nil
    }
}
```

**这是经典的 Go worker pool + fail-fast 模式**：

1. **`for range parallelism` + `wg.Go`**：起 N 个 worker（默认 8），从 `jobs` channel 拉任务。`wg.Go` 是 Go 1.25 新加的 `WaitGroup.Go` 方法，比 `wg.Add(1); go func() { defer wg.Done(); ... }()` 更简洁。
2. **`errCh` buffer=1（L425）**：第一个错误占满，后续 worker `select default` 直接丢——**只有第一个错误会被报告**。
3. **`cancel()` 立刻取消整个 ctx（L433）**：第一个错误触发 `cancel()`，所有 worker 的 `downloadSearchResult` 内部 HTTP 调用立刻失败返回，投递循环也通过 `<-ctx.Done()` 退出。
4. **三段 select 收尾（L442-456 + L458-463）**：投递循环和收尾各一次 select，确保任何路径都把 worker 拉完（`wg.Wait()`）再返回。

**Java 类比**：相当于 `ExecutorService.invokeAll` + `Future` 里第一个抛异常就 `shutdownNow`。Go 用 channel + ctx 表达更原生。

### 3.9 `ensureDirectory` 优雅降级（opensandbox.go:672-690）

```go
func (r *OpenSandboxRuntime) ensureDirectory(ctx context.Context, dir string, mode int) error {
    err := r.sandbox.CreateDirectory(ctx, dir, mode)
    if err == nil { return nil }
    // ★ API 失败 → fallback 到 shell mkdir -p
    quoted := shellquote.QuotePOSIX(dir)
    result, execErr := r.runCommand(ctx, "/", "mkdir -p "+quoted+" && test -d "+quoted+" && test -w "+quoted, 30)
    if execErr != nil { return err }                      // shell 也跑不动，返回原始 API 错误
    if result.ExitCode == 0 {
        logging.WarnContextf(ctx, "OpenSandboxRuntime: directory API failed for %s, continuing after shell verification: %v", dir, err)
        return nil                                        // shell 验证通过，降级成功
    }
    if result.Stderr != "" {
        return fmt.Errorf("%w; shell verification failed with exit code %d: %s", err, result.ExitCode, result.Stderr)
    }
    return fmt.Errorf("%w; shell verification failed with exit code %d", err, result.ExitCode)
}
```

**优雅降级两步走**：(1) 先调 SDK 的 `CreateDirectory` API；(2) 失败的话 fallback 到 `mkdir -p && test -d && test -w` shell 命令验证目录确实存在且可写。只要任一条路成功就算 OK。**为什么这么写**——OpenSandbox 服务端某些 profile 可能没实现 directory API，但 shell 一定有，这样兼容性最好。注释（L683）专门记了 `warn` 提醒用户 API 失败但继续了。

### 3.10 路径安全：`safeLocalTarget`（opensandbox.go:800-814）

```go
func safeLocalTarget(root, rel string) (string, error) {
    clean := filepath.Clean(filepath.FromSlash(rel))
    if clean == "." { return root, nil }
    // Windows 上 filepath.IsAbs 要 volume name，所以 POSIX 风格的 "/absolute" 会漏过
    rooted := strings.HasPrefix(clean, "/") || strings.HasPrefix(clean, `\`)
    if filepath.IsAbs(clean) || rooted || clean == ".." || strings.HasPrefix(clean, ".."+string(filepath.Separator)) {
        return "", fmt.Errorf("unsafe sandbox file path: %s", rel)
    }
    return filepath.Join(root, clean), nil
}
```

**这是防 Zip Slip / 路径穿越攻击**的核心。SDK 返回的文件路径如果包含 `../../etc/passwd`，简单 `filepath.Join(root, rel)` 会逃出 `root`。这里显式拒绝：

1. 绝对路径（含 POSIX `/abs` 和 Windows `\abs`）
2. `..` 或 `../...` 开头（向上跳）

注释（L805-808）特别讲了 Windows 的坑：`filepath.IsAbs` 在 Windows 上要求 volume name（`C:\`），所以 POSIX 风格 `/etc/passwd` 会被判相对路径——必须额外用 `strings.HasPrefix(clean, "/")` 兜底。这是个**跨平台安全检查**的细节。

### 3.11 `safeFileMode`：八进制解析（opensandbox.go:816-836）

```go
func safeFileMode(rawMode int) os.FileMode {
    if rawMode <= 0 { return noneFileMode }
    if rawMode > 777 { return noneFileMode }              // ★ 八进制上限
    digitModes := [...]os.FileMode{0, 1, 2, 3, 4, 5, 6, 7}
    var mode os.FileMode
    shift := 0
    for rawMode > 0 {
        digit := rawMode % 10
        if digit > 7 { return noneFileMode }              // 任一位 > 7 非法
        mode |= digitModes[digit] << shift
        shift += 3
        rawMode /= 10
    }
    return mode
}
```

SDK 返回的 mode 是十进制表示的八进制数（比如 `755` 表示 rwxr-xr-x）。这里逐位拆解，每位必须 ≤ 7，左移 3 位拼接成 Go 的 `os.FileMode`。任何非法（≤0、>777、某位 >7）回退到默认 `0o600`。**防御式编程**——不相信远端返回的格式。

---

## 4. DockerRuntime：shell out docker CLI（非 SDK）

定位（`docker.go:44-58` 注释）：

```go
// DockerRuntime executes commands inside a local Docker container.
//
// Lifecycle:
//   - Create: docker create with the configured image, entrypoint sleep loop,
//     workspace dir, optional --network=none, optional env from cfg.Env.
//   - Start: docker start.
//   - Stop:  docker stop --time=5.
//   - Close: docker rm -f (skipped when cfg.Delete is false; the container
//     is just stopped so the user can inspect it).
//
// All cross-environment access flows through `docker exec` / `docker cp`, so
// no host bind mount is required.
```

**关键设计选择**：**shell out 调 `docker` CLI，不用 Docker SDK**。原因：

1. CLI 几乎所有开发机都装了，SDK 要拉一堆依赖。
2. CLI 行为可预测、可手动复现（用户能直接 `docker exec -it ...` 进去排障）。
3. 单测直接 mock 那个函数字段，无需起真容器。

### 4.1 结构 + mock seam（docker.go:38-73）

```go
// dockerCommandRunner is the seam unit tests use to capture or fake the
// invocations DockerRuntime issues against the `docker` CLI.
type dockerCommandRunner func(ctx context.Context, name string, args ...string) (stdout, stderr string, exitCode int, err error)

type DockerRuntime struct {
    cfg       Config
    workspace string
    cli string                                         // 默认 "docker"，测试可注入
    run dockerCommandRunner                            // ★ 函数字段：生产 runDockerCommand，测试 fake
    mu          sync.Mutex
    containerID string                                 // Create 后才有
    started     bool
}
```

**两个 mock seam**：

- `cli string`：默认 `"docker"`，测试可换成 fake binary 名。
- `run dockerCommandRunner`：函数字段，生产代码用 `runDockerCommand`（L674），测试注入 fake（`docker_test.go:51` 的 `fakeDocker.runner()`）。

**Java 类比**：相当于把 `DockerClient` 接口做成一个字段，测试里换 Mockito mock。Go 用函数类型表达更轻量。

### 4.2 `NewDockerRuntime`：构造期校验（docker.go:77-102）

```go
func NewDockerRuntime(cfg Config) (*DockerRuntime, error) {
    if strings.TrimSpace(cfg.Image) == "" {
        return nil, errors.New("docker runtime requires environment.image")
    }
    // ★ allow_declared 暂不支持，拒绝而不是静默放行
    if policy := strings.TrimSpace(strings.ToLower(cfg.NetworkPolicy)); policy == "allow_declared" {
        return nil, errors.New("docker runtime: network_policy=allow_declared is not yet supported; use deny_all or run on opensandbox")
    }
    workspace := strings.TrimSpace(cfg.WorkspaceMount)
    if workspace == "" { workspace = dockerDefaultWorkspace }
    if !path.IsAbs(workspace) {
        return nil, fmt.Errorf("docker runtime: workspace_mount must be absolute, got %q", workspace)
    }
    return &DockerRuntime{
        cfg:       cfg,
        workspace: path.Clean(workspace),
        cli:       "docker",
        run:       runDockerCommand,
    }, nil
}
```

**`allow_declared` 拒绝（L81-88）的注释值得读**：

```go
// allow_declared needs FQDN-level egress filtering. Implementing
// that for the local docker runtime requires either an egress
// proxy sidecar or iptables rules in the container — both out
// of scope for the initial cut. Refuse loudly instead of silently
// allowing all egress, which would violate the user's policy.
```

**Refuse loudly instead of silently allowing**——这是安全设计的金科玉律。功能没实现就明确报错，**绝不静默降级到不安全状态**。否则用户以为白名单生效了，实际全放行。面试可重点讲这条。

### 4.3 `Create`：create → start → exec mkdir（docker.go:109-154）

```go
func (r *DockerRuntime) Create(ctx context.Context) error {
    r.mu.Lock()
    defer r.mu.Unlock()

    if r.containerID != "" { return nil }               // 幂等

    name, err := dockerContainerName()                   // skill-up-<rand hex>
    if err != nil { return ... }

    args := r.buildCreateArgs(name)
    stdout, stderr, exitCode, err := r.run(ctx, r.cli, args...)
    if err != nil || exitCode != 0 {
        return dockerCLIErr(stderr, exitCode, err, "docker create failed")
    }
    id := strings.TrimSpace(stdout)
    if id == "" { id = name }                            // 某些 docker 版本 stdout 空，回退 name
    r.containerID = id

    // ★ start 失败要回滚 rm
    _, startStderr, startExit, startErr := r.run(ctx, r.cli, "start", id)
    if startErr != nil || startExit != 0 {
        primary := dockerCLIErr(startStderr, startExit, startErr, "docker start %s failed", id)
        if rbErr := r.rollbackRemove(ctx, id); rbErr != nil {
            return errors.Join(primary, rbErr)           // ★ errors.Join 把两个错误合并
        }
        r.containerID = ""
        return primary
    }
    r.started = true

    // ★ mkdir -p workspace，失败也要回滚
    if _, mkStderr, mkExit, mkErr := r.run(ctx, r.cli, "exec", id, "mkdir", "-p", r.workspace); mkErr != nil || mkExit != 0 {
        primary := dockerCLIErr(mkStderr, mkExit, mkErr, "docker exec mkdir -p %s failed", r.workspace)
        if rbErr := r.rollbackRemove(ctx, id); rbErr != nil {
            return errors.Join(primary, rbErr)
        }
        r.containerID = ""
        r.started = false
        return primary
    }
    return nil
}
```

**三步编排**：(1) `docker create` 拿容器 ID；(2) `docker start` 启动；(3) `docker exec mkdir -p` 建工作区。**任何一步失败都 `rollbackRemove`**。

`errors.Join`（Go 1.20+）把主错误和回滚错误合并报告——如果回滚也失败了，两个错误都暴露给调用方，便于排障。

**为什么 Create 内部就把 mkdir 干了？** 注释（L104-108）解释：evaluator 不在 Create 和第一次 Exec 之间调 Start，所以 Create 必须把容器带到"完全可用"状态（和 OpenSandboxRuntime 一致，那里 Create 同时 provision + connect）。

### 4.4 `buildCreateArgs`：`--network none` / env / `sleep infinity`（docker.go:159-181）

```go
func (r *DockerRuntime) buildCreateArgs(name string) []string {
    args := []string{
        "create",
        "--name", name,
        "--workdir", r.workspace,
    }
    if policy := strings.TrimSpace(strings.ToLower(r.cfg.NetworkPolicy)); policy == "deny_all" {
        args = append(args, "--network", "none")        // ★ 完全断网
    }
    for k, v := range r.cfg.Env {
        args = append(args, "--env", k+"="+v)            // 字面透传
    }
    entry := r.cfg.Entrypoint
    if len(entry) == 0 {
        entry = []string{"sleep", "infinity"}            // 让容器一直挂着
    }
    args = append(args, "--entrypoint", entry[0])
    args = append(args, r.cfg.Image)
    if len(entry) > 1 {
        args = append(args, entry[1:]...)
    }
    return args
}
```

**`--network none`（L166-167）**：`deny_all` 策略映射到 Docker 的 `none` 网络——容器没有任何网络接口，连 localhost 都没有。最严格隔离。

**`sleep infinity` entrypoint（L172-174）**：默认让容器主进程一直挂着（`sleep infinity` 永不退出），后续靠 `docker exec` 跑命令。如果用户自定义了 entrypoint 就用用户的。

### 4.5 `rollbackRemove`：detach ctx（docker.go:192-201）

```go
//nolint:contextcheck // Intentionally detached: parent cancellation must not skip cleanup.
func (r *DockerRuntime) rollbackRemove(_ context.Context, id string) error {
    cleanupCtx, cancel := context.WithTimeout(context.Background(), dockerCleanupTimeout)  // 30s
    defer cancel()
    _, stderr, exitCode, err := r.run(cleanupCtx, r.cli, "rm", "-f", id)
    if err != nil || exitCode != 0 {
        logging.DebugContextf(cleanupCtx, "DockerRuntime.rollbackRemove: rm -f %s failed (exit=%d): %s", id, exitCode, strings.TrimSpace(stderr))
        return dockerCLIErr(stderr, exitCode, err, "docker rm -f %s failed during rollback", id)
    }
    return nil
}
```

**`_ context.Context`**：故意不用调用方的 ctx！如果用原 ctx，调用方 cancel 后这个回滚也跑不动（和 OpenSandbox 的 `context.WithoutCancel` 同理）。这里用 `context.Background()` + 30s 超时（`dockerCleanupTimeout`），保证回滚不被父 ctx 取消影响。`nolint:contextcheck` 注释告诉 linter"我知道这违反规则，是故意的"。

`dockerCleanupTimeout = 30s`（L30-35）的注释讲：够 `docker rm -f` 正常完成（几秒）+ 些余量给繁忙 daemon；不至于让卡死的 daemon 把 runtime 锁死太久。

### 4.6 `snapshotContainerID`：短锁防串行（docker.go:520-527）

```go
func (r *DockerRuntime) snapshotContainerID() (string, error) {
    r.mu.Lock()
    defer r.mu.Unlock()
    if r.containerID == "" {
        return "", errors.New("docker runtime: container not created (call Create first)")
    }
    return r.containerID, nil
}
```

注释（L511-519）非常详细，值得整段读：

```go
// snapshotContainerID returns the current container id under the mutex,
// so callers can use the captured local value through the rest of their
// method without racing a concurrent Close. The lock is released on
// return — `docker exec` / `docker cp` may take many seconds and holding
// the lock for their duration would serialise every operation on the
// runtime. If Close fires while exec/cp is in flight, the captured id
// becomes stale and docker returns a "No such container" error, which
// dockerExecLayerError catches and surfaces as a layer fault — much
// better than a panic on a half-cleared field.
```

**核心权衡**：mu 只在**读 containerID 那一瞬间**持有，拿到本地变量后立刻释放。后续 `docker exec` 可能跑几秒甚至几分钟，**绝不能持锁等它**——否则所有操作串行化，并发评测直接退化成单线程。

**代价**：如果 `docker exec` 跑一半被 `Close` 抢了，containerID 已被清空，但本地 snapshot 还在用旧 ID，docker 会返回 "No such container"。这个错误被 `dockerExecLayerError`（L622-633）识别为"基础设施故障"，上层走重试路径——比"在半清空的字段上 panic"好得多。

**Java 类比**：相当于 `AtomicReference<String>` 的 `getAndSet` —— 拿到 snapshot 后释放锁，后续操作基于 snapshot，可能"看到已失效的状态"但不会崩。

### 4.7 `Exec`：`sh -c` 兼容 alpine（docker.go:388-452）

```go
func (r *DockerRuntime) Exec(ctx context.Context, command string, opts ExecOptions) (ExecResult, error) {
    id, err := r.snapshotContainerID()
    if err != nil { return ExecResult{}, err }

    ctx, span := observability.Tracer().Start(ctx, "runtime.exec")
    defer span.End()
    startTime := time.Now()

    if opts.TimeoutSec > 0 {
        var cancel context.CancelFunc
        ctx, cancel = context.WithTimeout(ctx, time.Duration(opts.TimeoutSec)*time.Second)
        defer cancel()
    }

    args := []string{"exec"}
    cwd := opts.Cwd
    if cwd == "" { cwd = r.workspace }
    if !path.IsAbs(cwd) {
        cwd = path.Join(r.workspace, cwd)
        if !isSubPath(r.workspace, cwd) {                // ★ 防 cwd 逃逸
            return ExecResult{}, fmt.Errorf("docker runtime: cwd %q escapes workspace %q", opts.Cwd, r.workspace)
        }
    }
    args = append(args, "--workdir", cwd)
    for _, kv := range overlayEnvList(r.cfg.Env, opts.Env) {
        args = append(args, "--env", kv)                 // 字面透传
    }
    // ★ 用 sh -c 而非 bash -c，兼容 alpine/busybox
    args = append(args, id, "sh", "-c", command)
    ...
}
```

**`sh -c` 而非 `bash -c`（L435）的注释值得整段**：

```go
// Use `sh -c` rather than `bash -c` so the docker runtime works on
// minimal base images (alpine, busybox, plain debian without bash).
// All shell snippets the rest of the codebase ships (setup_steps,
// judge scripts, agent commands) are POSIX-compatible.
//
// Shell-less images (distroless, scratch + a single binary) are NOT
// supported: skill-up's evaluation lifecycle (setup_steps, agent
// install, MCP install, judge scripts) is shell-driven end-to-end,
// so an image without /bin/sh isn't usable here even if Exec
// itself bypassed `sh -c`. Pick a base image with a POSIX shell.
```

**关键决策**：选 `sh -c` 而不是 `bash -c` 是为了兼容最小镜像（alpine 默认没装 bash，只有 busybox sh）。skill-up 的所有 shell 片段（setup_steps、judge 脚本）都刻意保持 POSIX 兼容，所以 `sh` 够用。**distroless 镜像不支持**——因为整个评测生命周期都是 shell 驱动的，没 `/bin/sh` 就跑不动。

**cwd 逃逸检查（L411-413）**：相对 cwd join 到 workspace 后，必须还在 workspace 子树内。`isSubPath`（L563-570）做这个判断。否则用户传 `../etc` 会被规范化成 `/etc`，逃出 workspace。

### 4.8 env 字面透传：`overlayEnvList`（docker.go:657-667）

```go
// overlayEnvList returns just the union of persistentEnv and callEnv as
// KEY=VALUE strings (callEnv wins on key conflicts). Values are passed
// LITERALLY — no host-env expansion happens here.
//
// Expanding `$VAR` against the host's environment before passing the value
// to `docker --env` is dangerous: it silently rewrites container-relative
// variables like PATH/HOME/USER with whatever the host's value happens to
// be, which breaks command lookup inside otherwise valid images (e.g.
// passing `PATH=$PATH:/tooling` would clobber the image's PATH with the
// macOS `/opt/homebrew/...`).
func overlayEnvList(persistentEnv, callEnv map[string]string) []string {
    overlay := mergeEnvMaps(persistentEnv, callEnv)
    if len(overlay) == 0 { return nil }
    out := make([]string, 0, len(overlay))
    for k, v := range overlay {
        out = append(out, k+"="+v)
    }
    return out
}
```

**注释（L643-656）讲得很清楚为什么字面透传**：如果在宿主展开 `$PATH`，会把容器内的 `PATH`（比如 `/usr/local/sbin:/usr/local/bin:...`）覆盖成宿主的（macOS 上 `/opt/homebrew/...`），容器里命令找不到。所以**只传 cfg.Env + opts.Env 的并集，且不展开任何 `$VAR`**。需要展开的调用方必须自己先展开成字面值（参见 `internal/agent.probeAndMergePATH`——它先在容器里探测出真实 PATH 再合并）。

**这是抽象边界的体现**：Runtime 不偷偷做"贴心"的变量展开，把这个责任明确留给上层。Java 类比：类似 JPA 不替你处理 `@Query` 里的参数替换，要你显式用 `:param` 绑定。

### 4.9 `dockerExecLayerError`：5 个错误前缀（docker.go:622-633）

```go
func dockerExecLayerError(stderr string) bool {
    s := strings.TrimSpace(stderr)
    switch {
    case strings.HasPrefix(s, "Error response from daemon:"),
        strings.HasPrefix(s, "Error: No such container"),
        strings.HasPrefix(s, "OCI runtime exec failed:"),
        strings.HasPrefix(s, "Cannot connect to the Docker daemon"),
        strings.HasPrefix(s, "error during connect:"):
        return true
    }
    return false
}
```

注释（L586-621）非常详细：**这五个前缀都是 docker/容器运行时层面的故障**（daemon 拒绝、容器消失、OCI runtime 失败、daemon 连不上），不是用户命令自身失败。它们必须被识别为"基础设施故障"让上层重试，而不是当成业务 FAIL。

**只匹配前缀**（注释 L593-594）：用户脚本合法地打印 `redis is not running` 然后退出 1，不应该被误判为基础设施故障。所以用 `HasPrefix` 而不是 `Contains`。

**不用 exit code**（注释 L617-621）：docker CLI 只保留 126/127 给自己用，125 空stderr的 daemon 失败和用户命令 exit 125 无法区分，所以保守地只按 stderr 分类。

`dockerCLIErr`（L577-584）是个贴心的错误格式化函数：

```go
func dockerCLIErr(stderr string, exitCode int, err error, format string, args ...any) error {
    cleanStderr := strings.TrimSpace(stderr)
    args = append(args, exitCode, cleanStderr)
    if err == nil {
        return fmt.Errorf(format+" (exit=%d): %s", args...)  // 无 err 不加 %w
    }
    return fmt.Errorf(format+" (exit=%d): %s: %w", append(args, err)...)  // 有 err 才 %w
}
```

CLI 非零退出时 `err == nil`（只有 docker 二进制都跑不起来时才有 err），如果机械地用 `: %w` 会渲染成 `%!w(<nil>)` 破坏 `errors.Unwrap`。**条件性加 `%w`** 是个 Go error 处理的细节技巧。

---

## 5. platform.Shell 跨平台抽象

`internal/platform/` 整个包只做一件事：**把"宿主是什么 OS、用哪个 shell、怎么引用参数"集中到一处**，让 `runtime/` 包不再散落 `runtime.GOOS` 分支。

### 5.1 三元组：`GOOS` + `Family` + `BashPath`（platform.go:44-50）

```go
// Shell describes the target command interpreter used by a runtime's Exec
// method. GOOS is the target environment, not necessarily the skill-up host.
type Shell struct {
    GOOS   string
    Family ShellFamily
    // BashPath is set when a Windows POSIX target explicitly invokes bash.
    // POSIX targets may leave it empty when the runtime owns shell selection.
    BashPath string
}

type ShellFamily string

const (
    ShellPOSIX ShellFamily = "posix"   // sh 兼容
    ShellCmd   ShellFamily = "cmd"     // Windows cmd.exe
)
```

**为什么是三元组？**

- `GOOS`：目标文件路径风格、行分隔符、`exec.LookPath` 行为都依赖它。
- `Family`：决定参数引用规则（POSIX 单引号 vs Windows 双引号 + CommandLineToArgvW）。
- `BashPath`：**Windows 上跑 POSIX 命令时必须知道 bash.exe 在哪**。Windows 自带 `cmd.exe` 不懂 POSIX 语法，但用户可能装了 Git Bash。`BashPath` 就是 bash 的绝对路径。

这三者解耦后能表达"Windows 主机 + POSIX 命令语言 + Git Bash 调度"这种混合状态——skill-up 在 Windows 上跑评测的关键。

### 5.2 `Shell.Validate`：构造期拒绝（platform.go:53-75）

```go
func (s Shell) Validate() error {
    if s.GOOS == "" {
        return errors.New("shell GOOS must not be empty")
    }
    switch s.Family {
    case ShellPOSIX:
        if s.GOOS == GOOSWindows && s.BashPath == "" {
            return errors.New("windows POSIX shell requires BashPath")
        }
    case ShellCmd:
        if s.GOOS != GOOSWindows {
            return fmt.Errorf("cmd shell requires windows GOOS, got %q", s.GOOS)
        }
        if s.BashPath != "" {
            return errors.New("cmd shell must not set BashPath")
        }
    case "":
        return errors.New("shell family must not be empty")
    default:
        return fmt.Errorf("unsupported shell family %q", s.Family)
    }
    return nil
}
```

**拒绝矛盾组合**：

- Windows + POSIX 但没 bash 路径——拒绝（POSIX 命令谁来执行？）
- cmd 但 GOOS 不是 Windows——拒绝（cmd.exe 只在 Windows）
- cmd 又设了 BashPath——拒绝（矛盾）

**这个 Validate 被 `NewRuntime` 在工厂里立刻调用**（runtime.go:234），坏配置 fail-fast，绝不带到运行时。

### 5.3 `Quoter()`：自动选引用函数（platform.go:78-89）

```go
func (s Shell) Quoter() (func(string) string, error) {
    if err := s.Validate(); err != nil {
        return nil, err
    }
    if s.Family == ShellCmd {
        return shellquote.QuoteWindows, nil          // cmd → CommandLineToArgvW 规则
    }
    if s.GOOS == GOOSWindows {
        return quoteForBashDoubleQuote, nil          // Windows + POSIX → bash 双引号
    }
    return shellquote.QuotePOSIX, nil                 // 其他 → POSIX 单引号
}
```

**三种引用策略，按 Shell 描述自动选**：

1. **`shellquote.QuotePOSIX`（shellquote.go:7-9）**：POSIX 单引号包裹，`'` 用 `'\''` 转义。最简单可靠。
   ```go
   func QuotePOSIX(s string) string {
       return "'" + strings.ReplaceAll(s, "'", `'\''`) + "'"
   }
   ```

2. **`shellquote.QuoteWindows`（shellquote.go:24-49）**：Windows cmd.exe 的 `CommandLineToArgvW` 规则——双引号包裹，反斜杠在引号前要双倍，内嵌 `"` 转义为 `\"`。注释（L17-23）专门讲：哪怕字符串没空格也要包双引号，因为 Git Bash 会把 `C:\tmp\script.cmd` 的反斜杠当转义符剥掉，包双引号能让反斜杠在 bash 和 cmd 两种语境下都保持字面。

3. **`quoteForBashDoubleQuote`（platform.go:115-128）**：Windows + POSIX 目标专用。bash 双引号内只 `\ " $ \`` 四个字符需要转义：
   ```go
   func quoteForBashDoubleQuote(s string) string {
       var b strings.Builder
       b.Grow(len(s) + 2)
       b.WriteByte('"')
       for i := range len(s) {
           c := s[i]
           if c == '\\' || c == '"' || c == '$' || c == '`' {
               b.WriteByte('\\')
           }
           b.WriteByte(c)
       }
       b.WriteByte('"')
       return b.String()
   }
   ```

**为什么 Windows + POSIX 要单独一套？** 因为命令最终是 `bash -c "<command>"`，参数要塞进 bash 双引号字符串里，bash 双引号语义（`$VAR` 展开、`` `cmd` `` 执行）必须被转义；但 cmd.exe 的 `CommandLineToArgvW` 规则又同时生效（因为 bash.exe 是 Windows 进程）。两套规则叠加，单独写一个 quoter 最清晰。

### 5.4 `Host()`：缓存 + 三平台 dispatch

`Host()` 是个**进程级 cache**（`shell_other.go:24`）：

```go
var hostShell = sync.OnceValue(buildHostShell)

func Host() HostShell { return hostShell() }
```

`sync.OnceValue`（Go 1.21+）保证 `buildHostShell` 只跑一次，结果全进程共享。注释（shell_windows.go:31-34）讲为什么 cache：`PATH` 和 `SKILL_UP_BASH` 设计为启动时读一次，主机上的 bash 安装在 skill-up 跑的过程中不会变；cache 避免每次 `NoneRuntime.Exec` 都重复 `LookPath`/`Stat`。

**POSIX 版（shell_other.go:29-45）**：

```go
func buildHostShell() HostShell {
    bash, hasBash := DiscoverBash()                    // 找 bash，找不到退回 sh
    shell := "sh"
    if hasBash { shell = bash }
    return HostShell{
        Target: Shell{
            GOOS:     runtime.GOOS,
            Family:   ShellPOSIX,
            BashPath: bash,
        },
        Cmd: func(ctx context.Context, command string) *exec.Cmd {
            return exec.CommandContext(ctx, shell, "-c", command)
        },
    }
}
```

简单：能找到 bash 就用 bash，否则用 sh。命令统一 `bash -c "<command>"`。

**Windows 版（shell_windows.go:40-66）复杂得多**：

```go
func buildHostShell() HostShell {
    bash, hasBash := DiscoverBash()
    if hasBash {
        return HostShell{
            Target: Shell{GOOS: GOOSWindows, Family: ShellPOSIX, BashPath: bash},
            Cmd: func(ctx context.Context, command string) *exec.Cmd {
                return exec.CommandContext(ctx, bash, "-c", command)
            },
            Env: []string{
                "MSYS_NO_PATHCONV=1",                  // ★ 防 MSYS 把 /x 改写成 Windows 路径
                "MSYS2_ARG_CONV_EXCL=*",
            },
        }
    }
    // cmd 兜底
    return HostShell{
        Target: Shell{GOOS: GOOSWindows, Family: ShellCmd},
        Cmd: func(ctx context.Context, command string) *exec.Cmd {
            cmd := exec.CommandContext(ctx, "cmd")
            cmd.SysProcAttr = &syscall.SysProcAttr{
                CmdLine: `cmd /d /s /c "` + command + `"`,
            }
            return cmd
        },
    }
}
```

注释（L12-29）讲得很清楚：

- **优先 Git Bash**：很多内部命令字符串带单引号、`set -eu`、`if/then`、git fixture——cmd.exe 跑不动，bash 才行。
- **`MSYS_NO_PATHCONV=1` + `MSYS2_ARG_CONV_EXCL=*`（L51-55）**：Git Bash 的 MSYS 层会把 `-c /x/y` 这种"看起来像 Unix 路径"的参数自动改成 `C:\Program Files\Git\x\y`——这在调原生 Windows 二进制（cmd.exe、powershell）时会出问题。这两个 env 关掉改写。
- **cmd 兜底用 `SysProcAttr.CmdLine`（L60-62）**：Go 默认 `exec.Command("cmd", "/d", "/s", "/c", command)` 在某些情况下 escape 不对。直接设 `CmdLine` 把整个命令行作为原始字符串传，遵循 `/d`（禁用 AutoRun）+ `/s`（强制 strip 首尾引号规则）+ 双引号包裹 + CommandLineToArgvW 解析。

### 5.5 `DiscoverBash`：Windows 上排除 WSL bash

`bash_windows.go:28-43` 的查找顺序：

```go
func DiscoverBash() (string, bool) {
    if v := os.Getenv(BashEnvOverride); v != "" {       // 1. SKILL_UP_BASH 覆盖
        if isRegularFile(v) && !isWSLBash(v) {
            return v, true
        }
    }
    if p, err := exec.LookPath("bash"); err == nil && !isWSLBash(p) {  // 2. PATH 排除 WSL
        return p, true
    }
    for _, p := range knownWindowsBashPaths {            // 3. 已知 Git Bash 安装位置
        if isRegularFile(p) {
            return p, true
        }
    }
    return "", false
}
```

**`isWSLBash` 排除 WSL 的 `C:\Windows\System32\bash.exe`**（L51-61）的注释值得读：

```go
// The shim lives under %SystemRoot%\System32, which is on PATH by default on every
// Windows host, so PATH-based discovery would otherwise prefer it. The shim
// expects Linux-format paths (`/mnt/<drive>/...`) and silently fails on the
// Windows host paths we pass through, so we treat it as "no bash found" and
// fall through to the known Git Bash locations.
```

WSL 的 `bash.exe` 在 PATH 上排很靠前，但**它期望 Linux 路径**（`/mnt/c/...`），把 Windows 路径 `C:\Users\...` 喂给它直接失败。所以必须显式排除——这是个**踩过坑才写得出的代码**。

### 5.6 命令永远经 bash（即便宿主是 Windows）

这是整个抽象的**最终效果**：

- evaluator / agent 调 `rt.Exec(ctx, "set -eu; npm install", opts)`。
- `NoneRuntime.Exec` 调 `platform.Host().Cmd(ctx, command)` → 在 Windows 上变成 `bash.exe -c "set -eu; npm install"`。
- 命令字符串是 POSIX 语法，bash 能直接执行，**完全绕过 cmd.exe 的语法不兼容问题**。
- 只有在用户机器上**完全没装 Git Bash** 时，才退化到 cmd.exe fallback（注释 `shell_windows.go:26-29` 坦白这是已知限制，写在 `docs/guide/windows.md`）。

> Java 类比：相当于你写了一套基于 Bash 的部署脚本，跨平台跑靠"在 Windows 上偷偷用 Git Bash"实现。这种"目标方言优先于宿主方言"的设计，类似 JVM 让 Java 代码无视底层 OS 差异——只不过这里"虚拟机"是 bash 本身。

---

## 6. 关键设计点（3-5 条，带 Java 类比）

### 6.1 统一接口屏蔽三后端（≈ Spring `DataSource` / `TransactionManager`）

13 方法接口 + 工厂 switch，让 evaluator 和 agent 代码完全不感知底下是本机、容器还是远端服务。**新加一个后端（比如 firecracker microVM）只需：实现接口 + 在工厂 switch 加一个 case + 配置层加一个 type**。evaluator 和 agent 一行不用改。

**对比 Java**：这跟 `DataSource` 屏蔽 H2/MySQL/PG、`TransactionManager` 屏蔽 JTA/JDBC 本地事务是同一个套路——"统一契约 + 多 Provider + 工厂选择"。skill-up 进一步把它做到 13 方法大接口（接受 `//nolint:interfacebloat`），换得 evaluator 调用方极简。

### 6.2 进程组 + WaitDelay + Cancel 升级（≈ Testcontainers 的容器清理）

NoneRuntime 的进程终止是个**三层防御**：

1. `Setpgid` 把子树变进程组（POSIX）；
2. `cmd.Cancel` 在 ctx 取消时给整组发 SIGTERM；
3. 1s 后还没死，goroutine 升级 SIGKILL；
4. 兜底 `cmd.WaitDelay` 强关 stdio 管道（Windows 上特别需要）。

这套机制保证超时命令**真的会被收掉**，不会留孤儿进程继续改工作区文件、污染下一个 case。**对比 Java**：Testcontainers 的 Ryuk 容器做类似的事——测试 JVM 崩了之后 Ryuk 还能清理容器。skill-up 这里是单进程内的"清理 Ryuk"。

### 6.3 跨平台 shell 抽象（≈ Spring `Environment` profile）

`platform.Shell` 三元组 + `Host()` cache + 三种 quoter，把"宿主是什么 OS、用哪个 shell、怎么 quote"彻底集中。**evaluator 写一次 `rt.Exec(cmd, ...)`，跨 Linux/macOS/Windows 都能跑**，命令永远经 bash（除非用户机器连 Git Bash 都没装）。

这个抽象的关键洞察是"**目标 shell ≠ 宿主 shell**"——`Shell()` 接口注释（runtime.go:149-152）专门警告调用方别用 `platform.Host()` 重新推导。这是**抽象边界清晰**的体现。

### 6.4 全 mock seam 的可测试性（≈ Mockito + Spring `@MockBean`）

三个后端都有清晰的 mock 注入点：

- **NoneRuntime**：13 方法都是普通方法，测试直接构造 `&NoneRuntime{cfg: Config{...}}`。
- **DockerRuntime**：`run dockerCommandRunner` 函数字段 + `cli string` 字段，测试注入 `fakeDocker.runner()`（`docker_test.go:51`）——**无需起真容器**。
- **OpenSandboxRuntime**：`openSandboxClient` 接口 + `var createOpenSandbox` 包级变量，测试替换工厂（`opensandbox_test.go:59-66`）——**无需网络**。

**对比 Java**：Go 没有 DI 容器，靠"接口 + 函数字段 + 包级 var"三件套实现等效 Mockito 的效果。比 Java 反射 mock 更轻量、更类型安全。这是**Go 工程文化的典型体现**——可测试性靠"显式依赖注入"而不是"反射魔法"。

### 5.5 安全防御：路径穿越 + 字面 env + 白名单优先

四散各处的安全细节，组合起来构成可信边界：

- **`safeLocalTarget`（opensandbox.go:800-814）**：防 Zip Slip。
- **`overlayEnvList` 字面透传（docker.go:657-667）**：不展开 `$VAR`，避免宿主 PATH 污染容器。
- **`networkPolicy` 默认拒绝（opensandbox.go:633-644）**：白名单优先，空白名单 = 全断网。
- **`allow_declared` 不支持就拒掉（docker.go:81-88）**：绝不静默降级到不安全状态。
- **`isWSLBash` 排除（bash_windows.go:51-61）**：避免误用语义不一致的 bash。

**每一条都是踩过坑才写得出的**。这种"防御性编程 + 经验沉淀"是面试可以重点展开的，体现工程成熟度。

---

## 7. 面试 Q&A（3-5 条）

### Q1：为什么 Runtime 接口有 13 个方法？是不是违反接口隔离原则（ISP）？

**A**：是的，作者用 `//nolint:interfacebloat` 显式承认了这一点。但这是**有意的折中**：

- 三个后端的语义高度一致——它们都需要"生命周期 + 文件传输 + 执行 + 元信息"，每个后端都实现了几乎全部方法。
- 如果拆成 4 个小接口（`LifecycleRuntime` / `TransferRuntime` / `ExecRuntime` / `MetaRuntime`），调用方就要写一堆类型断言或组合，得不偿失。
- 单一大接口换来了 evaluator 调用方极简：`rt.Exec(...)`、`rt.UploadFile(...)`、`rt.Close()` 全在一行。

**对比 Java**：Spring 的 `ApplicationContext` 也是个巨型接口（几十个方法），同样是"为了大多数调用方方便"而牺牲 ISP。这是工程实用主义。如果面试官追问"什么时候会拆"，答："如果未来某个后端只支持执行不支持文件传输（比如 serverless 函数），就该拆了。"

### Q2：NoneRuntime 怎么保证超时命令真的被杀掉？

**A**：三层防御 + 一个兜底（背 `none_exec_unix.go`）：

1. **`Setpgid: true`** 让子进程成为新进程组 leader，整个子树成为一个组。
2. **`cmd.Cancel`** 在 ctx 取消时给整个组（负 PID）发 SIGTERM，给 trap handler 1s 优雅退出。
3. **goroutine 升级 SIGKILL**：1s 后没死就强杀（`noneExecKillGrace = 1s`）。
4. **`cmd.WaitDelay = 2s`** 兜底强关 stdio 管道（Windows 上 Git Bash 孙进程可能持着管道）。

`done` channel 在 `cmd.Wait` 返回后立刻 close，停止升级定时器，**防 PID 复用 TOCTOU**——内核回收 PID 后给新进程发 SIGKILL 是灾难。

**追问 "Windows 怎么办"**：Windows 没 POSIX 进程组，`configureProcessGroup` 是 no-op，只能靠 `WaitDelay=2s` 强关管道让子进程随管道一起被回收。这是已知妥协，注释明说了。

### Q3：为什么 OpenSandbox 用批量上传 + 并发下载，而不是统一一种？

**A**：因为两个方向的**瓶颈不同**：

- **上传瓶颈是 fd**：每个上传要打开本地文件读。一次性传几千个 Skill 文件会撑爆 `ulimit -n`。所以分批，每批最多 128 个文件，批结束统一关 fd（`defer close closers`）。
- **下载瓶颈是网络**：每个文件一次 HTTP GET，串行慢。所以并发，最多 8 个 worker（可配，上限 64），fail-fast（第一个错就 cancel 整批）。

**对比 Java**：上传 ≈ `Stream` + 分批处理防 OOM；下载 ≈ `ExecutorService.invokeAll` + `Future` 第一个异常就 `shutdownNow`。两套机制针对不同资源约束，不能混用。

### Q4：Docker 后端为什么 shell out CLI 而不用官方 Docker SDK？

**A**：四个理由：

1. **依赖轻**：CLI 几乎所有开发机都装了，SDK 要拉一堆 transitive deps，构建变慢。
2. **可手动复现**：用户能直接 `docker exec -it <id> sh` 进去排障，看到的和代码跑的一致。
3. **测试简单**：mock 一个函数字段就行，不用 mock SDK 的复杂 client。
4. **行为可预测**：CLI 输出对人友好，stderr 的前缀（`Error response from daemon:`）稳定，便于 `dockerExecLayerError` 这种基于前缀的分类。

**代价**：每次调用 fork 一个 docker 进程，比 SDK 直接走 HTTP 慢一点。但 skill-up 不是高频调用，几秒一次 `docker exec` 完全可接受。

**对比 Java**：相当于选 ProcessBuilder 调 `docker` 命令还是选 `docker-java` 客户端库——前者依赖少行为直观，后者类型安全但重。skill-up 选前者。

### Q5：`Shell()` 为什么不直接返回 `platform.Host()`？接口注释为什么专门警告"target ≠ host"？

**A**：因为**目标 shell 和宿主 shell 可能不一样**。典型场景：你在 Windows 笔记本上跑 skill-up 评测一个 Linux Docker 镜像——

- `platform.Host()` 返回 Windows（因为 skill-up 进程跑在 Windows 上）；
- 但 `DockerRuntime.Shell()` 返回 `Linux/POSIX`（因为命令实际跑在 Linux 容器里）。

如果调用方按 `platform.Host()` 选 quoter，会用 Windows 引用规则生成 `"C:\tmp\file"`，结果送进 Linux bash 直接报错（bash 不认 `C:\` 路径，反斜杠被当转义）。

所以接口注释（runtime.go:149-152）专门警告：**callers must not re-derive it from platform.Host**。正确做法是 `rt.Shell().Quoter()` 拿到目标 shell 的 quoter。

**这是抽象边界清晰**的体现——Shell 描述的是"Exec 实际跑在什么环境"，而不是"skill-up 进程跑在什么环境"。两者必须解耦，跨平台评测才可能。

---

## 8. 一句话总结

Runtime 层是 skill-up 给"不可信代码执行"画的**统一隔离边界**：13 方法接口 + 工厂 switch 屏蔽本机/容器/远端三后端，`platform.Shell` 三元组屏蔽 Linux/macOS/Windows 三平台，进程组 + WaitDelay 保证超时命令被收掉，全 mock seam 让三后端都可单测。整层的设计目标是——**让 evaluator 写一次代码，跨任意后端、任意宿主 OS 都能跑同一个不可信 Skill**。

> **核心源码索引**（按本文出现顺序）：
> - `internal/runtime/runtime.go`：接口 / Config / 工厂 / env merge
> - `internal/runtime/none.go`：NoneRuntime / classifyExecError / WaitDelay
> - `internal/runtime/none_exec_unix.go` + `none_exec_other.go`：进程组双实现（build tag 切换）
> - `internal/runtime/opensandbox.go`：OpenSandbox HTTP 对接 / 批量上传 / 并发下载
> - `internal/runtime/docker.go`：shell out docker CLI / snapshotContainerID / dockerExecLayerError
> - `internal/platform/platform.go` + `shell_*.go` + `bash_*.go`：Shell 三元组 / Host() cache / DiscoverBash
> - `internal/shellquote/shellquote.go`：QuotePOSIX / QuoteWindows
