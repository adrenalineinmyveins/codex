# 03 · 核心引擎:codex-core

> 路径:`codex-rs/core/`(crate 名 `codex-core`,约 70+ 内部依赖)。它是整个产品的"编排层":会话状态机、Turn 循环、审批流、上下文组装都在这里;而模型传输、rollout 格式、MCP 连接、沙箱策略等已下沉为兄弟 crate(见 [05-支撑模块](./05-supporting-crates.md))。

## 1. 三层会话结构

```
ThreadManager(进程级,聚合多线程 + 共享服务初始化)
   └── CodexThread(线程级门面,公开 API:submit/next_event)
          ├── SessionIo(通道端点:tx_sub / rx_event / agent_status)
          └── Session(运行时状态机 + SessionServices 共享服务集)
                 └── ActiveTurn / RunningTask(SessionTask: Regular/Review/Compact/…)
```

> 历史演进:`ConversationManager → ThreadManager`、`CodexConversation → CodexThread`、旧 `codex.rs → session/`、旧 `event_processor → session/handlers.rs`、旧 `turns/ → session/turn.rs + tasks/`、`mcp_connection_manager → codex-mcp` crate。`lib.rs` 保留废弃别名。

## 2. 模块地图

### 2.1 顶层关键文件

| 模块 | 职责 |
|---|---|
| `codex_thread.rs` | **`CodexThread`**:单线程门面。`submit(Op)`、`start_or_steer_turn`/`start_turn_if_idle`/`steer_turn`、`next_event`、rollout 持久化、MCP 资源读取 |
| `thread_manager.rs` | **`ThreadManager`**:`start_thread`、`spawn_subagent`、`resume_thread_from_rollout`、`fork_thread`、`shutdown_all_threads_bounded`、`list_models`;初始化 AuthManager/ModelsManager/McpManager/PluginsManager/SkillsService 等共享服务 |
| `client.rs` | **`ModelClient` + `ModelClientSession`**:`stream()` 双通道(SSE/WS)、`prewarm_websocket`、压缩/记忆/realtime 客户端 |
| `client_common.rs` | `Prompt`(单次请求载荷:input + tools + base_instructions + output_schema)、`ResponseEvent/ResponseStream` 重导出 |
| `compact*.rs`(约 8 个文件) | 上下文压缩管线:本地 compact、远端 compact v1/v2、token 预算、模型回退 |
| `rollout.rs` | rollout 门面(类型主要 re-export 自 `codex-rollout`) |
| `shell.rs` / `shell_snapshot.rs` | 用户 shell 检测与命令派生(zsh/bash `-lc`、PowerShell) |
| `exec.rs` / `exec_env.rs` / `spawn.rs` | 子进程执行:spawn(stdio 策略、sandbox 环境)、输出聚合、超时;`spawn.rs` 定义 `CODEX_SANDBOX_ENV_VAR` 等 |
| `safety.rs` / `apply_patch.rs` | apply_patch 安全评估(`SafetyCheck::AutoApprove/AskUser/Reject`)与补丁执行 |
| `mcp.rs` | **`McpManager`**:config→`McpRuntimeInput` 投影、MCP 配置分层(用户/项目/插件/client extensions) |
| `event_mapping.rs` | 协议事件→turn item 映射(`parse_turn_item`) |
| `agents_md.rs` / `agents_md_manager.rs` | AGENTS.md 发现与加载(向上查找、缓存) |
| `hook_runtime.rs` | SessionStart/PreToolUse/PostToolUse/Stop hooks 运行时 |
| `skills.rs` / `plugins/` | `@skill` mention 解析与隐式调用;插件注入 |
| `guardian/` | Guardian 安全审查器:policy、prompt、review session、审批 |
| `realtime_conversation*` | 实时语音对话(Realtime API)管理器 |
| `unified_exec/` | 统一执行后台进程管理(`UnifiedExecProcessManager`、head/tail 缓冲) |
| `exec_policy.rs` + `exec_policy/` | execpolicy(WASM 规则)管理器 |
| `codex_delegate.rs` | `codex exec` 子进程委托模式 |
| `state_db_bridge.rs` | SQLite 状态库桥(`StateDbHandle`) |

### 2.2 `session/` — 会话运行时

| 模块 | 职责 |
|---|---|
| `mod.rs` | `SessionIo`(通道端点)、`SessionSpawnArgs`、`Session::spawn()`(exec policy、模型解析、SessionConfiguration 校验、启动 submission_loop) |
| `session.rs` | `Session` + `SessionConfiguration`(provider、collaboration_mode、approval_policy、permission_profile_state、dynamic_tools 等 40+ 字段) |
| `handlers.rs` | **`submission_loop`**:逐个消费 `Submission`,按 Op 变体分发 |
| `turn_input.rs` | `Op::TurnInput` 路由决策(Start / Steer / StartIfIdle / NotSubmitted) |
| `turn.rs` | **`run_turn`** — Turn 主循环(见 [02 §3](./02-architecture.md)) |
| `turn_context.rs` / `step_context.rs` | Turn 级 / 采样级的上下文快照 |
| `input_queue.rs` | `TurnInputQueue`(pending input / mailbox / steer 队列) |
| `world_state.rs` | WorldState 分节快照与 diff |
| `context_window.rs` / `token_budget.rs` | 上下文窗口滚动与 token 预算 |
| `rollout_reconstruction.rs` | resume 时从 rollout 重建历史 |
| `mcp*.rs` | MCP refresh / prewarm / runtime 集成 |
| `multi_agents.rs` / `review.rs` / `environment.rs` / `thread_settings.rs` / `inject.rs` / `time_reminder.rs` / `rollout_budget.rs` | 多 agent、审查、环境、线程设置、注入、时间提醒、rollout 预算 |

### 2.3 `state/` — 状态容器

| 模块 | 职责 |
|---|---|
| `session.rs` | `SessionState`:history(ContextManager)、rate limits、auto-compact window、startup prewarm 等 |
| `turn.rs` | `ActiveTurn`(task + `TurnState`)、`RunningTask`(Notify、CancellationToken、AbortOnDropHandle)、`TaskKind`(Regular/Review/Compact) |
| `service.rs` | `SessionServices`:50+ 共享服务句柄(mcp_runtime、model_client、unified_exec_manager、auth_manager、hooks、thread_store、network_proxy、extensions…) |

### 2.4 `tasks/` — Turn 任务

- `mod.rs`:**`SessionTask` trait**(`kind/span_name/run/abort`,RPITIT:`impl Future + Send`)+ `Session::spawn_task/start_task/abort_all_tasks/on_task_finished`;
- `regular.rs`:`RegularTask`(常规用户 Turn);
- `compact.rs` / `review.rs` / `user_shell.rs`:`CompactTask`(`/compact`)、`ReviewTask`、`!cmd` 用户 shell;
- `lifecycle.rs`:Turn/线程生命周期事件向 extension contributors 广播。

### 2.5 `tools/` — 工具系统

| 模块 | 职责 |
|---|---|
| `registry.rs` | `ToolRegistry` + `CoreToolRuntime` trait(在 `codex-tools::ToolExecutor` 上叠加 hooks/遥测/工具搜索元数据) |
| `router.rs` | **`ToolRouter`**:最终对模型广播的工具 spec 集合 + 执行路由;`ToolCall` |
| `orchestrator.rs` | **`ToolOrchestrator`**:审批→sandbox 选择→尝试→拒绝后升级 sandbox 重试(免重复审批) |
| `parallel.rs` | `ToolCallRuntime`:并行执行(保序结果) |
| `handlers/` | 每个工具一个文件:`*_spec.rs` 定义模型可见 schema,同名无后缀文件是执行 handler(shell、unified_exec(exec_command/write_stdin)、apply_patch、mcp、view_image、request_user_input、tool_search、new_context_window、sleep、wait_for_environment…) |
| `runtimes/` | apply_patch / unified_exec / zsh_fork 的进程运行时 |
| `code_mode/` | Code Mode(execute/wait 委托给 codex-code-mode 进程) |

### 2.6 `context/` + `context_manager/` — 模型可见上下文

- `context/`:38 个片段模块,全部实现 `ContextualUserFragment` trait(trait 定义在 `codex-context-fragments`);
- `context/world_state/`:WorldState 分节快照,是"上下文增量更新"的基线;
- `context_manager/history.rs`:**`ContextManager`** — 历史条目(`Arc<Vec<ResponseItemEnvelope>>` 写时共享)、history_version、token_info、diff 基线;
- `context_manager/normalize.rs` / `updates.rs`:历史规范化、增量 update 生成。

> 对应 AGENTS.md 的模型上下文治理规则:历史只增不改、片段结构化、有界、≤10K token。

### 2.7 其他子目录

`agent/`(多 agent 注册表/角色/控制/spawn 深度限制)、`apps/`(Codex Apps 渲染)、`config/`(Config 加载编辑、权限画像、keyring auth)、`plugins/`、`guardian/`、`realtime_conversation/`、`exec_policy/`、`mcp_tool_call/`、`sandboxing/`、`unified_exec/`、`utils/`。

## 3. 关键函数速查

| 函数/方法 | 位置 | 说明 |
|---|---|---|
| `ThreadManager::start_thread` | `thread_manager.rs` | 创建新 `CodexThread`(组装共享服务) |
| `CodexThread::submit(op)` | `codex_thread.rs` | 提交 Op,返回 submission id |
| `CodexThread::next_event()` | `codex_thread.rs` | 阻塞等待下一个 Event |
| `Session::spawn(SessionSpawnArgs)` | `session/mod.rs` | 创建会话并 spawn submission_loop 任务 |
| `submission_loop` | `session/handlers.rs` | Op 分发主循环(每线程一个) |
| `Session::start_task` | `tasks/mod.rs` | 注册 ActiveTurn 并 spawn RunningTask |
| `run_turn` | `session/turn.rs` | Turn 采样主循环(压缩→上下文→采样→工具→续跑判定) |
| `run_sampling_request` / `try_run_sampling_request` | `session/turn.rs` | 发起单次流式模型请求并消费 ResponseEvent |
| `ModelClient::stream` | `client.rs` | WS 优先、SSE 回退的模型流请求 |
| `ToolRouter` / `ToolOrchestrator` | `tools/` | 工具广播与审批/沙箱执行编排 |
| `ContextManager::for_prompt` | `context_manager/history.rs` | 由历史构造 Prompt 输入 |
| `handle`(`turn_input.rs`) | `session/turn_input.rs` | TurnInput 的 Start/Steer 路由 |

## 4. core 与外部 crate 的关系(按领域)

| 领域 | crate | 提供 |
|---|---|---|
| 协议 | `codex-protocol` | Op/Submission/Event/ResponseItem/审批与沙箱类型/CodexErr |
| 配置 | `codex-config` | 分层 config.toml 加载;core 再包装为 `Config` + 权限画像 + `config.schema.json` |
| 工具 | `codex-tools` | ToolName/ToolSpec/ToolExecutor trait/ResponsesApiTool |
| 模型 | `codex-model-provider` + `-info` + `codex-models-manager` + `codex-api` + `codex-client` | provider 工厂、模型目录、SSE/WS 传输、ResponsesApiRequest |
| 持久化 | `codex-rollout` + `codex-thread-store` + `codex-history` + `codex-state` + `codex-rollout-trace` | RolloutRecorder、ThreadStore trait、历史类型、SQLite 镜像、trace |
| MCP | `codex-mcp` + `codex-rmcp-client` | McpRuntime/McpBinding/McpConnectionSet |
| 沙箱/执行 | `codex-sandboxing`、`codex-exec-server`、`codex-execpolicy`、`codex-apply-patch`、`codex-windows-sandbox`、`codex-network-proxy` | 沙箱包装、执行环境、WASM 规则、补丁、网络代理 |
| 上下文/提示 | `codex-context-fragments`、`codex-prompts`、`codex-skills(-extension)`、`codex-core-plugins`、`codex-connectors` | 片段契约、prompt 模板(core/ 根目录 `*.md`)、技能、插件、连接器 |
| 可观测 | `codex-otel`、`codex-analytics`、`codex-diagnostics` | SessionTelemetry、turn/tool 指标 |
| 其他 | `codex-login`(AuthManager)、`codex-extension-api`(扩展注册表)、`codex-code-mode`、`codex-app-server-protocol` | 认证、扩展点、Code Mode、app-server 类型 |

## 5. 测试结构(`core/tests/`)

```
core/tests/
├── all.rs                  # 单测试二进制入口
├── common/                 # core_test_support 支持库(独立 Cargo.toml)
│   ├── test_codex.rs       #   TestCodex / TestCodexBuilder.build_with_auto_env()
│   ├── responses.rs        #   ResponseMock、mount_sse*、ResponsesRequest 断言、WS 测试服务
│   ├── streaming_sse.rs    #   ev_* / sse() SSE payload 构造
│   ├── context_snapshot.rs #   模型可见布局快照断言
│   └── hooks.rs / process.rs / test_environment.rs / …
├── remote_env_windows/     # 跨 OS 远程执行环境测试
└── suite/                  # ~100-120 个集成测试主题模块 + snapshots/(insta)
```

典型模式:

```rust
// 1. 用 wiremock MockServer 挂一次 SSE 响应
let mock = responses::mount_sse_once(&server, responses::sse(vec![
    responses::ev_response_created("resp-1"),
    responses::ev_function_call(call_id, "shell", &serde_json::to_string(&args)?),
    responses::ev_completed("resp-1"),
])).await;

// 2. 提交用户输入
codex.submit(Op::UserTurn { .. }).await?;

// 3. 断言事件与请求体
wait_for_event!(codex, EventMsg::TurnComplete { .. });
let request = mock.single_request();
assert!(request.function_call_output(call_id).is_some());
```

环境感知:`skip_if_sandbox!` / `skip_if_no_network!` 宏在 Seatbelt/无网沙箱中提前退出;`TestCodexBuilder::build_with_auto_env()` 保证与 app/exec-server 异 OS 部署兼容。
