# skill-up 项目技术梳理（简历向）

> 本目录是为「理解项目 + 改简历」准备的技术梳理，独立于官方 `docs/`（那是 VitePress 用户手册）。
> 面向读者：Java 后端 / 全栈工程师，需要把这个 Go 项目写进简历并能在面试中讲清楚。

## 这是什么项目

**skill-up** 是 Alibaba 开源的 **Agent Skill 评测与演进 CLI 框架**（Go 1.25）。

一句话定位：**「给 AI Agent 的 Skill 写单元测试 / 回归测试」**。
你声明 YAML 用例 → 框架把 Skill 装进真实 Agent 引擎（Claude Code / Codex / Qoder CLI / 通义千问 qwen_code）→ 跑 prompt → 用「规则 / 脚本 / LLM 裁判」打分 → 产出结构化报告，并支持跨引擎对比、with/without Skill 基线 A/B、CI 集成、OpenTelemetry 全链路追踪。

它解决的痛点：Agent Skill（给 AI Agent 用的「技能包/插件」）的质量过去只能靠人肉试，没有可度量、可复现、可回归的工程化手段。skill-up 把这件事产品化成一个 CLI。

## 文档索引

| 文档 | 内容 | 用途 |
|------|------|------|
| [06-快速上手.md](./06-快速上手.md) | **怎么用**：安装→写用例→跑→看报告（Windows 适配） | **先看这个，跑通一遍** |
| [01-架构与核心流程.md](./01-架构与核心流程.md) | 分层架构 + `skill-up run` 端到端执行流程（时序） | **面试讲流程的主线** |
| [02-核心技术点.md](./02-核心技术点.md) | 设计模式 + 工程深度亮点（简历弹药） | **写简历/讲深度的素材** |
| [03-子系统详解.md](./03-子系统详解.md) | Judge 评分 / Agent 引擎适配 / Runtime+Report 三大子系统深入 | **被追问时能下钻** |
| [04-简历素材与面试问答.md](./04-简历素材与面试问答.md) | 项目描述模板 + STAR 话术 + 高频面试 Q&A | **直接拿来改简历** |
| [05-技术栈.md](./05-技术栈.md) | 项目技术栈 + 通俗讲解 + 精简版 | **简历技能区** |

## 技术栈速览（Go 相关）

| 类别 | 技术 |
|------|------|
| 语言/版本 | Go 1.25（用到 `maps`、`slices`、`min/max` 等新特性） |
| CLI 框架 | `spf13/cobra` + `spf13/pflag` |
| 配置 | `gopkg.in/yaml.v3`，自研 `v1alpha1` schema + 分层校验器 |
| 可观测性 | OpenTelemetry（OTLP gRPC/HTTP，trace + metric），`go.opentelemetry.io/otel` 全家桶 |
| 沙箱后端 | 自研 `internal/runtime`：none / OpenSandbox（Alibaba 远程沙箱 SDK）/ Docker（shell out） |
| 文件匹配 | `bmatcuk/doublestar`（globs） |
| 依赖注入 | 不用框架，**构造器注入 + 接口**（Go 惯例） |
| 测试 | 表驱动测试为主，`go test -race`，e2e 用 build tag 隔离 |
| 构建/发布 | GoReleaser + GitHub Actions，`-ldflags` 注入版本号 |

## 阅读建议

**推荐顺序（先用后懂）**：
1. 先看 [06-快速上手.md](./06-快速上手.md) 把项目跑通一遍（建立体感）。
2. 跑通后看 [01-架构与核心流程.md](./01-架构与核心流程.md) 建立全局画面。
3. 再看 [02-核心技术点.md](./02-核心技术点.md) 挑 3–5 个能讲清楚的亮点。
4. [03-子系统详解.md](./03-子系统详解.md) 当字典查，被追问再下钻。
5. 最后用 [04-简历素材与面试问答.md](./04-简历素材与面试问答.md) + [05-技术栈.md](./05-技术栈.md) 落地到简历。

> 注：所有 `文件名:行号` 引用都基于本仓库源码，可点击核对。
