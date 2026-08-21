# 02 · 整体架构

## 1. 分层架构总图

```
┌─────────────────────────────────────────────────────────────────────┐
│ 用户界面层                                                            │
│  tui(终端交互)   exec(headless)   IDE/桌面App   SDK(Python/TS)      │
└──────────────┬──────────────────────────────────────────────────────┘
               │ app-server 协议(JSON-RPC: thread/turn/item)
┌──────────────▼──────────────────────────────────────────────────────┐
│ 服务层  app-server(+app-server-client in-process 外观、daemon)      │
│         mcp-server(Codex 作为 MCP 服务端)  exec-server(进程/文件)    │
└──────────────┬──────────────────────────────────────────────────────┘
               │ Op(指令) / Event(事件) —— codex-protocol
┌──────────────▼──────────────────────────────────────────────────────┐
│ 会话引擎  codex-core                                                 │
│  ThreadManager → CodexThread → Session(submission_loop → Turn 循环)  │
│  上下文管理(ContextManager/WorldState) 工具编排(ToolRouter/…)       │
│  审批流 压缩(compact) 多 agent 实时语音 hooks guardian              │
└──┬───────┬────────┬─────────┬──────────┬──────────┬─────────────────┘
   │       │        │         │          │          │
┌──▼──┐ ┌──▼─────┐ ┌▼────────┐ ┌▼───────┐ ┌▼───────┐ ┌▼──────────────┐
│模型 │ │沙箱执行 │ │MCP 客户端│ │持久化   │ │扩展体系 │ │账号/认证       │
│model│ │sandbox │ │codex-mcp │ │rollout │ │ext/*   │ │login/secrets  │
│-pro-│ │-ing +  │ │rmcp-     │ │+state  │ │plugin/ │ │+keyring       │
│vider│ │linux/  │ │client    │ │+thread │ │skills/ │ │+chatgpt       │
│+api │ │windows │ │          │ │-store  │ │core-   │ │               │
│     │ │sandbox │ │          │ │        │ │plugins │ │               │
└─────┘ └────────┘ └──────────┘ └────────┘ └────────┘ └───────────────┘
   底座:codex-protocol(协议类型) codex-config(配置) codex-otel(可观测) utils/*
```

**两个关键解耦点**:

1. **tui 不依赖 core**:TUI 通过 `codex-app-server-client`(进程内或连接共享 daemon 的 app-server)交互,前端与引擎完全解耦;
2. **协议先行**:`codex-protocol`(core 级 Op/Event)与 `codex-app-server-protocol`(对外 JSON-RPC)分层定义,支持多前端复用同一引擎。

## 2. 核心数据流:Op → Event

所有前端与引擎的交互都通过一对通道完成(定义于 `codex-rs/core/src/session/mod.rs` 的 `SessionIo`):

```
前端 (tui/exec/app-server/SDK)
   │ ① codex_thread.submit(Op) 
   ▼
Submission { id: UUIDv7, op, trace, parent_turn_id, root_turn_id }
   │ async_channel(容量 512)
   ▼
Session::submission_loop(session/handlers.rs,每线程一个任务)
   │ ② 按 Op 变体分发处理
   ▼
SessionTask(RegularTask / ReviewTask / CompactTask / UserShellTask)
   │ ③ run_turn:采样循环
   ▼
Event { id: submission_id, msg: EventMsg }
   │ unbounded mpsc
   ▼
前端 codex_thread.next_event() 消费渲染
```

### Op(前端→引擎指令,`codex-rs/protocol/src/protocol.rs`)

| 类别 | 变体(节选) |
|---|---|
| 流控制 | `Interrupt` `Shutdown` `CleanBackgroundTerminals` |
| Turn 输入 | `TurnInput{request, mode, reply}`(同步回复 Started/Steered/NotSubmitted) `RecoverTurn` |
| 设置 | `ThreadSettings` `SetThreadMemoryMode` `ReloadUserConfig` |
| 审批响应 | `ExecApproval` `PatchApproval` `ResolveElicitation` `UserInputAnswer` `RequestPermissionsResponse` `DynamicToolResponse` |
| 其他 | `Compact` `ThreadRollback` `Review` `RunUserShellCommand` `RefreshMcpServers` `InterAgentCommunication` `RealtimeConversation*` |

### EventMsg(引擎→前端事件,约 60 个变体)

`TurnStarted/TurnComplete/TurnAborted`、`AgentMessage(Delta)`、`AgentReasoning*`、`ExecCommandBegin/OutputDelta/End`、`ExecApprovalRequest`、`PatchApplyBegin/Updated/End`、`McpToolCallBegin/End`、`McpStartupUpdate/Complete`、`TokenCount`、`ContextCompacted`、`ThreadRolledBack`、`Error/Warning`、raw response item 透传等。

## 3. Turn 生命周期(引擎心脏)

一个 Turn = 用户一次输入到模型完成响应(可能含多轮工具调用)的完整过程:

```
submit(Op::TurnInput)
  └─ 路由决策:空闲→启动任务 / 运行中→steer 入队(TurnInputQueue)
       └─ Session::start_task:注册 ActiveTurn、token 基线、TurnStarted 事件
            └─ RegularTask::run → 循环调用 run_turn(session/turn.rs)
                 ┌───────────────────────────────────────────────┐
                 │ 前置:hooks 结果合流 → 超限先压缩(pre-compact)      │
                 │      → @mention 驱动 MCP 预启动                    │
                 │      → 冻结 StepContext(模型/MCP/工具视图)         │
                 │      → WorldState diff 写入上下文基线              │
                 │      → skills/plugins 注入 → 记录用户输入          │
                 ├───────────────────────────────────────────────┤
                 │ 每轮:取 pending input(steer/邮箱)                 │
                 │      → ContextManager 历史 for_prompt() 构造输入   │
                 │      → run_sampling_request(SSE/WS 流式)         │
                 │      → 流事件映射为协议事件 + 记录历史              │
                 ├───────────────────────────────────────────────┤
                 │ 工具调用:FunctionCall → ToolRouter                │
                 │      → ToolCallRuntime(并行,保序)                 │
                 │      → ToolOrchestrator(审批→sandbox→重试升级)     │
                 │      → FunctionCallOutput 入历史 → 下一轮采样       │
                 ├───────────────────────────────────────────────┤
                 │ 后置:needs_follow_up?(要工具输出/pending input)     │
                 │      token 超限 → run_auto_compact(中途压缩/       │
                 │      新上下文窗口) → stop hooks(可强制续跑) → break │
                 └───────────────────────────────────────────────┘
            └─ on_task_finished:TurnComplete(或 TurnAborted{reason})
```

**三层上下文快照**贯穿其中:

| 层 | 生命周期 | 内容 |
|---|---|---|
| `TurnContext` | 整个 Turn 不变 | model_info、取消令牌、extension data、计时 |
| `StepContext` | 每次采样请求 | 冻结的模型/MCP 绑定/ToolRouter/AGENTS.md |
| `WorldState` | 跨 Turn 增量 | 环境权限/工具/AGENTS.md/人格/协作模式等分节快照,只把 diff 注入上下文 |

## 4. 模型请求通道

`codex-rs/core/src/client.rs` 实现双通道与回退:

```
ModelClient(会话级:provider、auth、传输回退状态)
  └─ ModelClientSession(每 Turn:new_session(),缓存 WS 连接与
       x-codex-turn-state 粘性路由 token)
       ├─ 首选 stream_responses_websocket(WS response.create,支持 prewarm)
       └─ 失败按 retry-budget 回退 stream_responses_api(HTTP SSE)
```

流事件为 `codex-api::ResponseEvent`(`OutputItemAdded/Done`、`Completed`、`Error`…),turn 侧消费并映射为协议事件。

## 5. 进程拓扑(运行时)

```
codex(cli 主进程)
 ├─(默认)in-process:app-server 库 + core 会话 + exec-server 库(EnvironmentManager)
 ├─(可选)连接本地共享 daemon app-server(unix socket 控制面 + stdio/websocket)
 ├─(子进程)helper 身份:apply_patch / codex-linux-sandbox / codex-execve-wrapper
 ├─(远程模式)exec-server 于另一 OS:Noise 加密 relay(protobuf)
 └─(Code Mode)codex-code-mode-host 独立进程(ws/gRPC)→ V8 runtime
```

## 6. 安全模型(贯穿全栈)

1. **沙箱分层**:每条命令按 `SandboxPolicy`(read-only / workspace-write / danger-full-access)包装;
2. **审批策略**:`ApprovalPolicy`(untrusted/on-failure/on-request/never)与沙箱正交组合,由 `ToolOrchestrator` 执行"先沙箱尝试→失败请求审批→升级重试";
3. **execpolicy**:Starlark 规则(`~/.codex/execpolicy.starlark`)可前缀放行/拒绝命令;
4. **网络**:默认禁网;放行时经本地 network-proxy(rustls MITM/CONNECT/SOCKS5)按域名策略过滤;
5. **凭据**:API key/token 存 OS keyring 或 age 加密本地文件,日志与上下文脱敏(`redact_secrets`)。
