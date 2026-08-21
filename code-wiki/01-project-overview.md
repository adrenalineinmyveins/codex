# 01 · 项目总览

## 1. 项目是什么

**Codex CLI** 是 OpenAI 开源的本地编码智能体(Apache-2.0 许可)。核心形态:

- **终端智能体**:用户与模型多轮对话,模型通过工具(shell 命令、`apply_patch` 文件修改、MCP 工具、web 搜索等)操作本地代码库;
- **安全优先**:所有命令默认在沙箱(macOS Seatbelt / Linux Landlock+bubblewrap / Windows AppContainer)中运行,越权操作需经审批(approval)流程;
- **多形态接入**:终端 TUI、非交互 `exec`、JSON-RPC app-server(驱动 IDE 扩展/桌面 App)、MCP server(把 Codex 暴露给其他 MCP 宿主)、Python/TypeScript SDK。

## 2. 顶层仓库布局

```
/workspace
├── AGENTS.md              # 贡献规范(必读:代码风格、测试、review 规则)
├── justfile               # 任务入口(fmt/test/fix/bazel/... 默认 cd 到 codex-rs)
├── MODULE.bazel           # Bazel 模块定义(封闭工具链、crate.from_cargo)
├── defs.bzl               # codex_rust_crate 宏(Cargo↔Bazel 对齐)
├── codex-rs/              # ★ Rust workspace(约 140 个 crate,全部业务代码)
│   ├── core/              # ★ 会话引擎(最大、最核心的 crate)
│   ├── cli/ tui/ exec/    # 用户入口
│   ├── app-server*/       # JSON-RPC 服务与协议
│   ├── protocol/ config/  # 核心协议与配置
│   ├── ext/ utils/        # 扩展实现与工具库
│   └── docs/              # bazel.md、protocol_v1.md 等内部文档
├── codex-cli/             # npm 根包 @openai/codex(仅含 bin/codex.js 启动器)
├── sdk/
│   ├── python/            # openai-codex(基于 app-server 协议)
│   └── typescript/        # @openai/codex-sdk(基于 CLI 子进程 + JSONL)
├── scripts/               # 构建/打包/一致性校验脚本(Python)
├── tools/                 # argument-comment-lint 自定义 linter 等
├── docs/                  # 对外文档(多为外链存根,实质内容在 developers.openai.com)
└── third_party/ patches/  # Bazel 依赖(V8/libc++ 等)与补丁
```

## 3. 技术栈

| 领域 | 选型 |
|---|---|
| 语言 | Rust(edition 2024,workspace 统一管理版本/lints) |
| 异步运行时 | tokio(多线程,主线程 16MB 栈,见 `codex-rs/arg0`) |
| CLI 参数 | clap 4(derive) |
| TUI | ratatui + crossterm(OpenAI fork) |
| 模型 API | OpenAI Responses API;SSE(`eventsource-stream`)与 WebSocket 双通道 |
| RPC | JSON-RPC 2.0(app-server/exec-server/mcp-server 各自实现) |
| MCP | rmcp SDK(rust 实现) |
| 沙箱 | Seatbelt(profiles)/ Landlock+seccomp+bubblewrap / Windows AppContainer+WFP |
| 持久化 | JSONL rollout 文件 + SQLite(sqlx,`~/.codex` 下) |
| 可观测 | tracing + OpenTelemetry(OTLP) |
| 构建 | Cargo(开发)+ Bazel(rules_rs,封闭构建/CI/发布) |
| 测试 | cargo-nextest(强制)、insta 快照、wiremock、pytest/ts-jest |
| 类型导出 | ts-rs(Rust→TS 类型,app-server 协议双端共享) |
| 代码执行沙箱 | V8(rusty_v8,Code Mode 中执行模型生成的 JS) |

## 4. 产物形态(从源码到用户)

```
Rust 源码 (codex-rs/)
  → cargo build / bazel build
      → `codex` 单一多用途二进制(含全部子命令;发布名带平台 triple)
  → release 流程(rust-release.yml)
      → codex-package-<target>.tar.gz(GitHub Release)
      → npm 平台包 @openai/codex-{linux,win32,darwin}-{x64,arm64} + 根包 @openai/codex
      → dotslash / Homebrew cask / winget / R2 镜像
      → Python runtime wheel(openai-codex-cli-bin,SDK 依赖)
```

`codex` 二进制内部通过 **arg0 trick**(见 [04-入口与前端](./04-entrypoints-and-frontends.md) §7)在运行时伪装出多个 helper 身份:`apply_patch`、`codex-linux-sandbox`、`codex-execve-wrapper` 等,避免分发多个可执行文件。

## 5. 用户视角的运行方式(速览)

```bash
# 安装(终端用户)
curl -fsSL https://chatgpt.com/codex/install.sh | sh   # 或 npm i -g @openai/codex

# 认证
codex login        # 浏览器 ChatGPT 登录(推荐)/ API key / 设备码

# 使用
codex              # 交互式 TUI
codex exec "修复失败的单测"   # 非交互
codex resume       # 恢复历史会话
codex mcp / codex plugin / codex app-server / codex doctor ...
```

开发者本地源码构建与测试见 [07-构建测试与运行](./07-build-test-run.md)。

## 6. 关键设计原则(摘自 AGENTS.md,影响所有代码)

1. **抑制 codex-core 膨胀**:新功能优先落到兄弟 crate,而不是塞进 `core/`;
2. **模型上下文治理**:历史只增不改、避免缓存失效、注入片段必须实现 `ContextualUserFragment`、有硬上限;
3. **app-server API 只在 v2 开发**,命名/serde/ts-rs 约定严格(camelCase、`*Params/*Response/*Notification`);
4. **测试优先集成测试**(`core/suite`、app-server 公共 API),UI 变更必须带 insta 快照;
5. Cargo 是依赖事实来源,改 `Cargo.toml`/`Cargo.lock` 后必须 `just bazel-lock-update` 同步 Bazel 锁文件。
