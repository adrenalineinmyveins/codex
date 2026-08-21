# Codex CLI — Code Wiki

本 Wiki 是对 [openai/codex](https://github.com/openai/codex) 仓库(本工作区)源码的结构化解读,面向需要深入理解或参与开发该仓库的工程师。

> Codex CLI 是 OpenAI 出品的本地编码智能体(coding agent):用户在终端或 IDE 中与模型对话,模型通过受沙箱保护的 shell 命令、文件补丁、MCP 工具等能力对本地代码库执行操作。

## 文档导航

| 文档 | 内容 |
|---|---|
| [01-项目总览](./01-project-overview.md) | 项目是什么、仓库顶层布局、技术栈、产物形态 |
| [02-整体架构](./02-architecture.md) | 分层架构图、Op/Event 数据流、Turn 生命周期、进程拓扑 |
| [03-核心引擎 codex-core](./03-core-engine.md) | 会话引擎详解:`ThreadManager`/`CodexThread`/`Session`、任务系统、工具编排、上下文管理 |
| [04-入口与前端](./04-entrypoints-and-frontends.md) | `cli`/`tui`/`exec`/`app-server`/`exec-server`/`mcp-server`/`arg0`/`code-mode` 等 |
| [05-支撑模块详解](./05-supporting-crates.md) | 协议、配置、沙箱、模型接入、持久化、MCP、扩展/插件/技能、认证、可观测性 |
| [06-依赖关系](./06-dependency-graph.md) | workspace 分层依赖图、关键依赖链、命名约定 |
| [07-构建测试与运行](./07-build-test-run.md) | justfile/Cargo/Bazel 双轨构建、测试体系、CI、SDK、常见开发流程 |

## 快速理解(一分钟版)

```
用户
 └─ codex (cli, Rust 多功能入口)
     ├─ 无子命令 → tui(终端交互 UI)
     ├─ codex exec/review → exec(非交互 headless)
     └─ 其他子命令(mcp-server / app-server / exec-server / ...)
            │
            ▼
   app-server(JSON-RPC 服务,thread/turn/item 三原语)
            │
            ▼
   codex-core(会话引擎:Op 队列 → submission_loop → Turn 循环
              → 模型流式请求(SSE/WebSocket)→ 工具编排与审批 → Event 流)
            │
   ├─ sandboxing(macOS Seatbelt / Linux Landlock+bwrap / Windows AppContainer)
   ├─ codex-mcp(MCP 客户端,连接外部 MCP server)
   ├─ rollout + state(JSONL 会话持久化 + SQLite 索引)
   ├─ model-provider(OpenAI / Bedrock / OSS 等 Responses API 接入)
   └─ skills / plugins / ext 扩展体系
```

## 仓库规模速览

- Rust workspace:约 140 个 crate,全部位于 `codex-rs/`,crate 名以 `codex-` 为前缀,统一 2024 edition。
- 主二进制:`codex`(Rust);通过 npm(`@openai/codex`)、Homebrew、安装脚本分发。
- SDK:`sdk/python`(基于 app-server JSON-RPC)、`sdk/typescript`(spawn CLI 进程 + JSONL 事件)。
- 构建系统:Cargo(开发)+ Bazel(封闭/CI/发布)双轨。
- 文档:`AGENTS.md`(贡献规范,必读)、`codex-rs/docs/`、`docs/`。

## 约定

- 本 Wiki 中代码引用均为相对仓库根目录的路径,例如 `codex-rs/core/src/session/turn.rs`。
- 如与源码不一致,以源码与 `AGENTS.md` 为准。
