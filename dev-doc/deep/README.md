# dev-doc/deep · 源码逐行深度讲解

> 这是 skill-up 的**源码级**深度文档，8 个模块各一篇，每篇都贴真实代码 + `文件名:行号` + 逐段讲为什么。
> 面向：想把项目彻底吃透、面试能下钻到底的工程师。

## 与 `/doc` 的区别

| 目录 | 定位 | 深度 |
|------|------|------|
| [`/doc`](../../doc/README.md) | 简历向总览（架构/流程/技术点/简历素材） | 中，好讲好记 |
| **`/dev-doc/deep`**（本目录） | 源码逐行深挖 | 深，每段贴代码 |

> 看完 `/doc/07 核心流程讲解` 建立直觉后，来这里下钻源码。

## 8 篇索引（按依赖+重要性排序）

| 文档 | 模块 | 核心源码 |
|------|------|---------|
| [01-单case生命周期-executeCaseOnce.md](./01-单case生命周期-executeCaseOnce.md) | 单 case 7 步流程 | `internal/evaluator/` |
| [02-Judge三策略.md](./02-Judge三策略.md) | rule_based / script / agent_judge + expect | `internal/judge/` |
| [03-Agent多引擎适配.md](./03-Agent多引擎适配.md) | 接口 + 工厂 + 模板方法 + 4 引擎 + Custom | `internal/agent/` |
| [04-并发重试超时.md](./04-并发重试超时.md) | 信号量 + retry + context + 进程组 | `internal/evaluator/` `internal/runtime/` |
| [05-配置加载与分层校验.md](./05-配置加载与分层校验.md) | v1alpha1 schema + Loader + Validator | `internal/config/` |
| [06-Runtime沙箱三后端.md](./06-Runtime沙箱三后端.md) | none / opensandbox / docker + Shell 抽象 | `internal/runtime/` |
| [07-Report生成与Anthropic兼容.md](./07-Report生成与Anthropic兼容.md) | Reporter 策略 + A/B benchmark + 迭代工作区 | `internal/report/` |
| [08-安全-CustomEngine不可信输入.md](./08-安全-CustomEngine不可信输入.md) | TOCTOU / 脱敏 / SSRF / 凭据 | `internal/agent/custom*` `internal/customengine/` |

## 阅读建议

1. **01 是脊椎** —— 先读，建立「一个 case 怎么跑」的完整画面。
2. 02 / 03 是**两大核心子系统**（评测 + 引擎适配），面试最常被追问。
3. 04–08 按需查：被问到并发/配置/沙箱/报告/安全时下钻对应篇。

> 所有 `文件名:行号` 基于源码实际行号，可点击核对。
