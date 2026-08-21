# 04 · 入口与前端

> 本篇覆盖从用户命令到会话引擎之间的所有 crate:`cli`、`tui`、`exec`、`app-server`(+protocol/client/daemon)、`exec-server`、`mcp-server`、`arg0`、`code-mode` 系列、`responses-api-proxy`。

## 1. 调用关系总览

```
用户
 └─ codex (codex-cli)
     ├─ 无子命令 ─────────────► codex-tui::run_main ─┐
     ├─ codex exec / review ──► codex-exec::run_main ┤
     ├─ codex mcp-server ─────► codex-mcp-server ────┼─► codex-app-server-client
     ├─ codex app-server ─────► codex-app-server ────┤    (InProcess / Remote daemon)
     ├─ codex exec-server ────► codex-exec-server    │         │
     └─ (内部)ResponsesApiProxy                     ▼         ▼
                                            codex-app-server(lib) ─► codex-core
   所有二进制 main 均经 codex-arg0::arg0_dispatch_or_else 包装
```

## 2. cli — 主二进制 `codex`

- **路径**:`codex-rs/cli/`;二进制 `codex`(`default-run`)与 `logs_client`。
- **入口**:`src/main.rs` 的 `MultitoolCli`(clap Parser,flatten `CliConfigOverrides` + `FeatureToggles` + TUI 的 `TuiCli`)。
- **子命令**(`enum Subcommand`,main.rs:133):`Agents`、`Exec`(别名 `e`)、`Review`、`Login/Logout`、`Mcp`、`Plugin`、`McpServer`、`AppServer`、`RemoteControl`、`App`、`Completion`、`Update`、`Doctor`、`Sandbox`、`Debug`、`Apply`(别名 `a`)、`Resume`、`Queue`、`Archive/Unarchive`、`Delete`、`MigrateRollouts`、`Fork`、`Cloud`、`ResponsesApiProxy`(隐藏)、`StdioToUds`(隐藏)、`ExecServer`、`Features`。
- **分发**:默认→`codex_tui::run_main`;`Exec/Review`→`codex_exec::run_main`;`McpServer`→`codex_mcp_server::run_main`;`ResponsesApiProxy`→`codex_responses_api_proxy::run_main`。
- **lib 侧模块**:`login.rs`、`debug_sandbox.rs`、`mcp_cmd.rs`、`plugin_cmd.rs`、`doctor.rs`、`migrate_rollouts.rs`、`state_db_recovery.rs`、`wsl_paths.rs` 等。
- **依赖**:聚合几乎所有前端 crate(tui、exec、app-server 全家、mcp-server、codex-mcp、exec-server、responses-api-proxy、arg0、core、cloud-tasks…)。

## 3. tui — 终端交互 UI

- **路径**:`codex-rs/tui/`;二进制 `codex-tui`、`md-events`;入口 `pub async fn run_main(cli: Cli, ...) -> io::Result<AppExitInfo>`(lib.rs,内部 `startup_orchestration::run_main_inner`)。
- **关键类型**:`Cli`(TUI 专属 clap 参数)、`AppExitInfo`、`ExitReason`、`TokenUsage`。
- **组件结构**:
  - `app.rs` + `app/`:全局编排(事件循环、按键路由、`event_dispatch.rs`、`thread_events.rs`、`session_lifecycle.rs`、`history_ui.rs`、`transcript_export.rs`…);
  - `chatwidget.rs` + `chatwidget/`:聊天历史渲染与回合编排(审批弹窗、`slash_dispatch.rs`、`model_popups.rs`、`replay.rs`…;**注意:仓库规范禁止继续膨胀此文件**);
  - `bottom_pane/`:输入区(`chat_composer.rs`、`textarea.rs`)、footer、approval overlay、各类 popup;
  - `markdown.rs` / `markdown_stream.rs`(流式 Markdown)、`diff_render.rs`。
- **后端连接**:`codex-app-server-client` 的 `AppServerClient` / `InProcessAppServerClient`(进程内)/ `RemoteAppServerClient`(连接本地共享 daemon);并直接用 `codex-exec-server` 的 `EnvironmentManager`。
- **重要**:**不依赖 codex-core**(仅 dev 依赖测试支持),风格与快照规范见 `tui/styles.md` 与 [07 §测试](./07-build-test-run.md)。

## 4. exec — 非交互(headless)模式

- **路径**:`codex-rs/exec/`;入口 `run_main(cli, arg0_paths)`。
- **CLI**:`codex exec [prompt]`,支持 `--json`(JSONL 事件输出)、`--output-last-message`、`--output-schema`、`--ephemeral`、`--oss`,以及 `Resume/Fork/Review` 子命令。
- **流程**:加载配置(默认 `AskForApproval::Never`)→ OTEL → `EnvironmentManager` → `InProcessAppServerClient::start` → `thread/start`(或 resume/fork)→ `turn/start` → 循环消费 `ServerNotification` → `thread/unsubscribe` + shutdown。所有交互式 `ServerRequest`(审批等)在 exec 模式一律拒绝。
- **输出**:`event_processor.rs`(trait `EventProcessor`)、人类可读与 JSONL 两种实现;`exec_events.rs` 的类型经 ts-rs 导出 TS。

## 5. app-server + app-server-protocol — JSON-RPC 服务

### app-server(`codex-rs/app-server/`)

- **二进制**:`codex-app-server`(独立)与 `exec-server`(fixture);生产入口为 `codex app-server` 子命令。
- **用途**:驱动 VS Code 扩展、桌面 App、TUI/exec 等富客户端的服务进程。三大原语 **Thread / Turn / Item**;传输:stdio(默认 JSONL)、websocket(实验)、unix socket(控制面)、off。
- **入口**:`run_main_with_transport_options(...)`;`AppServerArgs`(`--listen`、`--session-source`、`--code-mode-host`、`--strict-config`、`--remote-control` 等)。
- **内部结构**:`message_processor.rs`(请求分发)+ `request_processors/`(按资源:thread/turn/account/config/fs/git/mcp/environment/process_exec/command_exec/marketplace/remote_control…)+ `transport.rs`(连接/背压/-32001 过载)+ `in_process.rs`(供进程内 client)。

### app-server-protocol(`codex-rs/app-server-protocol/`,纯类型库)

- `protocol/common.rs`:`client_request_definitions!` 宏生成 `ClientRequest/ClientNotification/ServerRequest/ServerNotification`、`JSONRPCMessage` 信封、experimental gating;
- `protocol/v1.rs`:旧版 API(仅兼容保留,**新开发一律进 v2**);
- `protocol/v2/`:按资源分模块(thread/turn/item/model/mcp/fs/account/config/review/plugin/environment…)+ mappers 与 core 协议互转;
- 代表性 RPC:`thread/start|resume|fork|read|list`、`thread/items/list`、`turn/start|steer|interrupt`、`review/start`、`model/list`、`account/login/*`、`config/read`、`fs/readFile`、`process/spawn`、`item/commandExecution/requestApproval`(server→client)等;
- 生命周期:连接后必须 `initialize` → `initialized`,否则其他请求被拒。

### 配套 crate

- `app-server-client`:进程内/远程 client 外观(bootstrap + initialize 握手 + 类型化通道 + 优雅关闭),exec/tui 共用;
- `app-server-daemon`:共享后台 daemon;
- `app-server-transport` / `app-server-test-client` / `app-server-protocol-noop-macros`。

## 6. exec-server — 进程与文件执行服务

- **路径**:`codex-rs/exec-server/`(纯 lib;生产入口 `codex exec-server` 子命令)。
- **职责**:spawn/控制子进程(经 `codex-utils-pty`)与沙箱化文件系统操作的小型 JSON-RPC 服务;支持本地 `ws://IP:PORT`、远程 `--remote URL --environment-id`(environment registry + **Noise 加密 relay**,protobuf `codex.exec_server.relay.v1`)、`forward` 转发。
- **关键类型**:`run_main(listen_url, runtime_paths, http_client_factory)`、`ExecServerClient`、`EnvironmentManager`/`Environment`/`EnvironmentProvider`、`ExecProcess`/`ExecBackend`、`LocalFileSystem`/`RemoteFileSystem`/`SandboxedFileSystem`、arg0 helper 入口(`run_fs_helper_main` / `run_arg0_exec_helper_main`)。
- **被依赖方**:app-server、tui、cli、mcp-server、arg0。

## 7. arg0 — 单二进制多身份分发

- **路径**:`codex-rs/arg0/`(纯 lib)。
- **机制("arg0 trick")**:
  1. `arg0_dispatch()`:按 `argv[0]`/`argv[1]` 分发 helper 身份:`codex-linux-sandbox`、`apply_patch`、`codex-execve-wrapper`、exec-server 的 exec/fs helper、Windows sandbox wrapper 等;
  2. 正常调用:加载 `~/.codex/.env` → 在 `$CODEX_HOME/tmp/arg0/<session>/` 创建符号链接别名并前置 `PATH`(`Arg0PathEntryGuard` 保活、`janitor_cleanup` 清理);
  3. `arg0_dispatch_or_else(main_fn)`:在 16MB 栈的 `codex-main` 线程上构建 tokio runtime 运行 async main,并把 `Arg0DispatchPaths` 传给 main。
- **被使用方**:cli、tui、exec、app-server、mcp-server 等所有二进制的 `main()`。

## 8. code-mode 系列(4 个 crate)

链路:`app-server →(codex-code-mode provider)→ codex-code-mode-host(独立进程,ws/gRPC)→ codex-code-mode-runtime(V8 沙箱)`。

| crate | 职责 |
|---|---|
| `code-mode-protocol` | 协议与抽象:`CodeModeSession` trait(启动 cell、Execute/Wait 请求)、host↔client 握手(`HostHello`/`Capability`)、gRPC 定义(build.rs + tonic-prost) |
| `code-mode-runtime` | 基于 V8(`v8` crate)的进程内实现:`InProcessCodeModeSession`、`initialize_v8`、JS cell 执行 actor、module loader/timers |
| `code-mode-host` | 独立宿主二进制 `codex-code-mode-host`(`--listen stdio|ws|grpc`),把 runtime 暴露给远端客户端;资源上限(MAX_IN_FLIGHT_REQUESTS=256、MAX_ACTIVE_CELLS=128) |
| `code-mode` | 客户端 provider:`WebSocketCodeModeSessionProvider`、`GrpcCodeModeSessionProvider`、`ProcessOwnedCodeModeSession`、`DisabledCodeModeSessionProvider`(被 app-server 消费) |

语义:模型以**代码形式**表达工具调用,在受资源限额约束的 V8 cell 中执行 `exec`/`wait` 工具。

## 9. mcp-server 与 codex-mcp(方向相反,勿混淆)

| crate | 方向 | 说明 |
|---|---|---|
| `mcp-server`(`codex-mcp-server`) | **Codex 作为 MCP 服务端** | `codex mcp-server` 启动;stdin/stdout 上的 rmcp JSON-RPC;向外部 MCP 宿主暴露 `codex` 工具(`CodexToolCallParam`),并经 MCP elicitation 做命令/补丁审批 |
| `codex-mcp` | **Codex 作为 MCP 客户端** | 会话内连接外部 MCP server;详见 [05 §9](./05-supporting-crates.md) |

## 10. responses-api-proxy

- **路径**:`codex-rs/responses-api-proxy/`;二进制 `codex-responses-api-proxy`(`#[ctor]` 先跑 `codex_process_hardening::pre_main_hardening()`)。
- **用途**:严格本地 HTTP 代理:仅接受 `POST /v1/responses` 转发上游(默认 `https://api.openai.com/v1/responses`,可指 Azure),注入 `Authorization: Bearer <key>`;key 从 **stdin** 读取并 zeroize/mlock 加固 —— 使特权用户持密钥、非特权用户经代理使用 Codex。支持 `--dump-dir` 录制用于测试。
- 由 cli 的隐藏子命令 `ResponsesApiProxy` 拉起;仅依赖 `codex-process-hardening`。
