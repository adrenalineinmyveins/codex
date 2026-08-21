# 06 · 依赖关系

> 事实来源:`codex-rs/Cargo.toml`(workspace 成员与 `[workspace.dependencies]`)及各 crate 的 `Cargo.toml`。Bazel 侧经 `crate.from_cargo` 从 Cargo.lock 导入,Cargo 是唯一事实来源。

## 1. 分层依赖图(自底向上)

```
第0层 纯外部依赖    bwrap · keyring-store · file-search · execpolicy · secrets
第1层 基础协议      network-proxy → protocol / otel / ext-items
第2层 领域信息      model-provider-info · history · state · skills · plugin
第3层 组装层        config · login · model-provider · apply-patch · sandboxing
第4层 持久化/传输   rollout · rollout-trace · rmcp-client · windows-sandbox-rs
第5层 集成层        codex-mcp · tools
第6层 扩展接口      ext/extension-api · thread-store
第7层 插件宿主      core-plugins
第8层 核心          core(汇聚以上几乎所有)
第9层 应用          chatgpt(在 core 之上) · tui(不走 core,走 app-server) · cli(core+tui)
```

两个特殊点:
- `protocol` 并非最底层(依赖 execpolicy、network-proxy、ext/items、http-client);
- **tui 不依赖 codex-core**(仅 dev 依赖 core_test_support),运行时经 `codex-app-server-client` 走 app-server 协议。

## 2. 三大入口 crate 的直接依赖

### core 直接依赖(本题相关)
protocol、config、tools、apply-patch、sandboxing、windows-sandbox、model-provider、model-provider-info、rollout、rollout-trace、codex-mcp、rmcp-client、extension-api、core-plugins、plugin、skills、execpolicy、login、otel、history、thread-store、network-proxy、state、ext/skills。

**不直接依赖**:chatgpt(方向相反)、secrets/keyring-store(经 login 间接)、file-search(经 rollout 间接)、linux-sandbox/bwrap(运行时经 sandboxing 以二进制方式拉起)。

### tui 直接依赖
protocol、config、core-plugins、plugin、model-provider、model-provider-info、login、otel、history、rollout、sandboxing、state、file-search、windows-sandbox + `app-server-client`/`app-server-protocol` + `exec-server`。

### cli 直接依赖
core、tui、protocol、config、execpolicy、extension-api、login、codex-mcp、model-provider、plugin、core-plugins、rmcp-client、rollout、rollout-trace、sandboxing、state、thread-store、history、chatgpt、ext/skills、ext/git-attribution、windows-sandbox(Win)+ app-server 全家、exec、mcp-server、exec-server、responses-api-proxy、arg0 等。

## 3. 关键依赖链(示意)

```
cli ──► core ──► core-plugins ──► plugin / skills / codex-mcp / tools / model-provider ...
 │       │
 │       ├──► thread-store ──► rollout ──► state ──► history ──► protocol ──► execpolicy / network-proxy
 │       ├──► sandboxing ──► windows-sandbox-rs;(Linux 运行时)──► codex-linux-sandbox 二进制 ──► 捆绑 bwrap
 │       ├──► codex-mcp ──► rmcp-client ──► secrets / keyring-store / config / network-proxy
 │       └──► login ──► secrets + keyring-store + model-provider-info

chatgpt ──► core        (反向:chatgpt 在 core 之上)
tui ──✖ core            (tui ──► app-server-client ──► app-server ──► core)
```

## 4. 前端↔引擎↔协议的依赖方向

```
                ┌─────────────── app-server-protocol(v1/v2 + common 宏)◄──────┐
                │                                                        │
   tui / exec / cli / mcp-server ──► app-server-client ──► app-server(lib) ──► core
                                     (InProcess / Remote daemon)              │
                                                                              ▼
                                              codex-protocol(Op/Event,core 级)
```

- 对外协议(app-server-protocol)与内部协议(codex-protocol)在 `app-server-protocol/src/protocol/v2/mappers.rs`、`event_mapping.rs` 互转;
- exec-server 有独立信封 `exec-server-protocol`;code-mode 全系共用 `code-mode-protocol`。

## 5. 命名与组织约定

- crate 目录名不带前缀,crate 名一律 `codex-<name>`(如 `core/` → `codex-core`);
- workspace 统一 `edition = "2024"`、统一 clippy lints(`[workspace.lints]`,大量 deny);
- `[workspace.dependencies]` 集中管理内部路径依赖与外部版本,各 crate 以 `workspace = true` 引用;
- `ext/` 子目录放内置扩展,`utils/` 放通用小工具库,`memories/`、`tests/common` 等也是 workspace 成员;
- 改 `Cargo.toml`/`Cargo.lock` 后必须 `just bazel-lock-update` 同步 `MODULE.bazel.lock`(CI 校验漂移)。

## 6. 主要外部依赖(节选)

| 用途 | crate |
|---|---|
| 异步/通道 | tokio、tokio-stream、async-channel、futures、crossbeam-channel |
| Web/HTTP | reqwest、axum、eventsource-stream、tokio-tungstenite(OpenAI fork)、tungstenite(OpenAI fork) |
| 序列化 | serde/serde_json、toml/toml_edit、serde_yaml、schemars、ts-rs |
| TUI | ratatui、ratatui-macros、crossterm(OpenAI fork)、nucleo(模糊匹配)、textwrap、vt100 |
| 沙箱 | landlock、seccompiler、bubblewrap(vendor C 源码) |
| 数据库 | sqlx(sqlite-bundled)、libsqlite3-sys |
| 脚本/策略 | starlark(execpolicy) |
| 解析 | tree-sitter(+bash/powershell)、pulldown-cmark、similar(diff) |
| 可观测 | tracing 全家、opentelemetry 全家、sentry |
| JS 运行时 | v8(rusty_v8 150.4.0)、deno_core_icudata |
| MCP | rmcp 3.1.3 |
| gRPC | tonic、tonic-prost |
| 加密/认证 | rustls、age、ed25519-dalek、jsonwebtoken、clatter(Noise) |
