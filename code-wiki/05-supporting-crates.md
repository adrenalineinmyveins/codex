# 05 · 支撑模块详解

> 按领域分组介绍 codex-core 与各前端共同依赖的支撑 crate。每个条目:路径 / 一句话职责 / 主要公开类型。

## 1. 协议与配置

### codex-protocol(`codex-rs/protocol/`)
**核心事件/Op 协议类型库**:前端与引擎之间全部 wire 类型。
- 主要类型:`Op`、`EventMsg`、`SessionMeta`、`SessionId`、`ThreadId`/`RolloutId`、`ToolName`、`models::ResponseItem`、`auth::AuthMode`、`config_types::*`、approvals/permissions、`mcp_policy`、`network_policy`、`error::CodexErr`、`turn_input`、`dynamic_tools`、`exec_output` 等。
- 依赖:codex-execpolicy、codex-network-proxy、ext/items、codex-http-client 等。

### codex-config(`codex-rs/config/`)
**config.toml 加载/分层合并/校验/编辑 + JSON schema**。
- 主要类型:`ConfigToml`、`loader` 模块、`ConfigLayer`/`ConfigLayerSource`、`ConfigRequirements(Toml)`、`mcp_types`/`mcp_edit`、`hook_config::HooksToml`、`cloud_config_bundle::CloudConfigBundle`、`diagnostics::{ConfigError, ConfigLoadError}`。
- 配置权威 schema 由 `codex-core` 生成:`codex-rs/core/config.schema.json`(`just write-config-schema`)。

## 2. 工具与补丁

### codex-tools(`codex-rs/tools/`)
**共享工具抽象层**:工具规格、执行 trait、MCP/动态工具到 Responses API 的转换。
- 主要类型:`ToolSpec`、`ToolDefinition`、**`ToolExecutor` trait**、`ToolCall`、`ToolEnvironment`、`TurnItemEmitter`/`ConversationHistory`、`ResponsesApiTool`/`FreeformTool`、`parse_mcp_tool`、`tool_search`(含 `TOOL_SEARCH_TOOL_NAME`)、`FunctionCallError`。

### codex-apply-patch(`codex-rs/apply-patch/`)
**解析并应用 `apply_patch` 补丁格式**(流式解析、unified diff 生成),同时提供 arg0 分发的独立 `apply_patch` 可执行。
- 主要类型:`parse_patch`、`Hunk`、`UpdateFileChunk`、`StreamingPatchParser`、`ApplyPatchOptions`、`ApplyPatchError`、`maybe_parse_apply_patch(_verified)`、`unified_diff_from_chunks`、常量 `CODEX_CORE_APPLY_PATCH_ARG1`。

## 3. 沙箱与执行安全

### codex-sandboxing(`codex-rs/sandboxing/`)
**跨平台沙箱策略→命令变换层**:把 `SandboxPolicy` 转成各平台包装命令并负责 spawn 与违规上报。
- 主要类型:`SandboxManager`、`SandboxCommand`、`SandboxExecRequest`/`SandboxTransformRequest`、`SandboxType`、`spawn_process`、模块 `seatbelt`/`landlock`/`policy_transforms`、`SandboxViolationEvent` 族。
- Linux 上把命令包一层 `codex-linux-sandbox` 可执行(`CODEX_LINUX_SANDBOX_ARG0`)。

### linux-sandbox(`codex-rs/linux-sandbox/`)
**Linux 沙箱辅助二进制**:no_new_privs + seccomp + Landlock,再 exec 捆绑/系统 bwrap;管理网络代理生命周期。入口 `run_main()`。

### windows-sandbox-rs(crate 名 `codex-windows-sandbox`)
**Windows 沙箱实现库与辅助进程**:AppContainer/restricted token、WFP 网络过滤、ConPTY 捕获、setup 安装器。bin:`codex-windows-sandbox-setup`、`codex-command-runner`。

### bwrap(`codex-rs/bwrap/`)
把 `codex-rs/vendor/bubblewrap` C 源码经 build.rs 编译为**静态捆绑的 bwrap 二进制**;运行时由 linux-sandbox 按 digest 校验后调用。

### execpolicy(`codex-rs/execpolicy/`)
**基于 Starlark 的命令审批策略引擎**(`execpolicy.starlark` 前缀规则)。
- 主要类型:`Policy`、`PolicyParser`、`Evaluation`、`Decision`、`PrefixRule`、`ExecPolicyCheckCommand`、`blocking_append_allow_prefix_rule`。

### network-proxy(`codex-rs/network-proxy/`)
**本地网络代理沙箱**(基于 rama):MITM/CONNECT/SOCKS5、域名许可策略、审计、凭据 broker。
- 主要类型:`NetworkProxy`/`NetworkProxyBuilder`/`NetworkProxyHandle`、`NetworkProxyConfig`/`NetworkMode`、`NetworkDomainPermissions`、`NetworkPolicyDecider`、`MitmHookConfig` 族。

## 4. 模型接入

### codex-model-provider(`codex-rs/model-provider/`)
**运行时模型提供方抽象**:provider 定义 + 认证 + 模型管理 → 可发起 Responses API 请求的 `ModelProvider`(含 Amazon Bedrock)。
- 主要类型:`ModelProvider` trait、`create_model_provider`、`SharedModelProvider`、`ProviderCapabilities`、`ResolvedProviderAuth`、`BearerAuthProvider`、`RemoteCompactionSupport`。

### codex-model-provider-info(`codex-rs/model-provider-info/`)
**模型提供方注册表**:内置 OpenAI/OSS/Bedrock 定义 + 用户 `model_providers` 覆盖 → `ModelProviderInfo`(`WireApi` 等)。

### 配套:`codex-api`(Responses API 请求/流事件类型)、`codex-client`(HTTP/WS 客户端)、`codex-models-manager`(模型目录刷新,内置 `models.json`)、`ollama`/`lmstudio`(本地推理接入)。

## 5. 持久化

### codex-rollout(`codex-rs/rollout/`)
**会话 rollout(JSONL)持久化与发现**:录制、zstd 压缩、列表/搜索/索引、SQLite 镜像、维护锁。
- 主要类型:`RolloutRecorder`、`RolloutLine`/`RolloutItem`、`list::{get_threads, ThreadsPage, ThreadItem, Cursor}`、`search`、`state_db`、`spawn_rollout_compression_worker`。

### codex-history(`codex-rs/history/`)
**模型历史与 rollout 领域类型**(纯数据):`ResponseItemEnvelope`、`RolloutItem`、`RolloutLine`、`InitialHistory`/`ResumedHistory`、`CompactedItem`。

### codex-state(`codex-rs/state/`)
**SQLite 状态库**:从 JSONL rollout 提取线程元数据镜像、迁移(`migrations/*.sql`)、线程历史/队列/日志 db、损坏备份恢复。
- 主要类型:`StateRuntime`(首选入口)、`SqliteConfig`、`ThreadMetadata`、`SqliteQueueStore`、`GoalStore`、`MemoryStore`。

### codex-thread-store(`codex-rs/thread-store/`)
**存储无关的线程持久化接口**:`ThreadStore` trait / `LocalThreadStore`(rollout 文件 + SQLite)/ `InMemoryThreadStore`、`StoredThread`/`StoredTurn`、fork/搜索/归档参数族、`QueueStore`/`LocalQueueStore`、projects。

### codex-rollout-trace(`codex-rs/rollout-trace/`)
**rollout trace bundle 格式**:热路径追加式 writer + 原始事件 + 语义 replay/reducer(推理、工具 dispatch、MCP、code-mode cell)。
- 主要类型:`TraceWriter`、`RawTraceEvent(Payload)`、`replay_bundle`、`*TraceContext` 族。

## 6. MCP 集成

### codex-mcp(`codex-rs/codex-mcp/`)
**Codex 的 MCP 集成中枢**:配置解析、目录(catalog)、连接管理(`mcp_connection_manager.rs` — 工具与工具调用变更的统一入口,workspace 规则要求变更优先经此处理)、工具目录缓存、资源/事件流、elicitation 审阅与 OAuth。
- 主要类型:`McpRuntime`/`McpRuntimeInput`、`McpConfig`、`effective_mcp_servers`、`ResolvedMcpCatalog`、`McpBinding`/`PreparedMcpCall`、`McpToolCatalogCache`、`ElicitationReviewer`、OAuth 族。

### codex-rmcp-client(`codex-rs/rmcp-client/`)
**rmcp SDK 客户端封装**:stdio / streamable-HTTP / in-process / executor 传输、OAuth 发现与登录、启动错误分类、elicitation client service。
- 主要类型:`RmcpClient`、`ToolWithConnectorId`、`Elicitation`/`ElicitationResponse`、`StdioServerLauncher`、`perform_oauth_login*`、`McpAuthState`。

## 7. 扩展 / 插件 / 技能

### ext/extension-api(`codex-rs/ext/extension-api/`)
**扩展 API 契约层**:`ExtensionRegistry`/`ExtensionRegistryBuilder`、contributor trait 族(`TurnLifecycleContributor`、`ToolLifecycleContributor`、`ThreadLifecycleContributor`、`ContextContributor`、`PromptFragment`、`McpServerContributor`、`ApprovalReviewContributor`…)、capabilities(`AgentSpawner`、`ResponseItemInjector`、`ExtensionEventSink`)、`ExtensionData` 共享状态。

### ext/* 内置扩展
`agent`、`connectors`、`goal`、`git-attribution`、`guardian-v2`、`history-notes`、`image-generation`、`items`(纯数据,被 protocol/tools/rollout 依赖)、`memories`、`mcp`、`queue`、`skills`、`web-search`。

### codex-plugin(`codex-rs/plugin/`)
**插件包模型层**:`PluginId`、`manifest` 模块、`LoadedPlugin`/`PluginLoadOutcome`、`PluginProvider`/`ResolvedPlugin`、`AppDeclaration`。

### codex-core-plugins(`codex-rs/core-plugins/`)
**插件宿主/管理器**(core 与 app-server 共用):`PluginsManager`(安装/卸载/列表/详情)、marketplace 族(`marketplace_add/remove/upgrade`)、`loader`/`startup_sync`/`store`、`script_attribution::{TrustedPluginRoots, PluginCommandAttribution}`。

### codex-skills(`codex-rs/skills/`)
**SKILL.md 技能模型**:frontmatter 解析、根加载与快照缓存、`@mention` 提取、隐式调用检测、内嵌系统技能安装(include_dir)。
- 主要类型:`LoadedSkills`/`SkillRootLoader`、`SkillMetadata`、`parse_skill_frontmatter_metadata`、`ToolMentions`、`ImplicitSkillLookup`、`install_system_skills`。

## 8. 认证与账号

| crate | 职责 / 主要类型 |
|---|---|
| `codex-login`(`codex-rs/login/`) | **认证中枢**:`AuthManager`、`CodexAuth`、`AuthDotJson`、`TokenData`、`LoginServer`(PKCE 本地回调)、`run_device_code_login`、`login_with_api_key/access_token/bedrock_api_key`、`logout` |
| `keyring-store` | OS 钥匙串抽象:`KeyringStore` trait、`DefaultKeyringStore` |
| `secrets` | 机密存储与脱敏:`SecretName`/`SecretScope`、`LocalSecretsBackend`(age 加密)、`redact_secrets` |
| `codex-chatgpt`(`codex-rs/chatgpt/`) | core 之上的 ChatGPT 后端客户端(connectors 列表、`get_task`、`apply_command`),供 cli 任务/连接器功能 |
| `aws-auth` / `workload-identity` / `agent-identity` | Bedrock 凭据 / 工作负载身份 / agent 身份 |

## 9. 可观测与工具库

### codex-otel(`codex-rs/otel/`)
**可观测性底座**:OTLP 导出配置与初始化、全局 metrics 客户端、会话遥测、W3C trace context 传播。
- 主要类型:`OtelProvider`、`OtelSettings`、`Timer`/`start_global_timer`、`SessionTelemetry`、`context_from_w3c_trace_context`。

### 其他常用支撑
| crate | 职责 |
|---|---|
| `file-search` | nucleo 模糊文件搜索引擎 + `codex-file-search` CLI(沙箱内搜索) |
| `hooks` | 用户 hooks 配置与执行(SessionStart/PreToolUse/PostToolUse/Stop) |
| `codex-home` | `~/.codex` 目录管理 |
| `build-info` | 构建元信息(git commit 等) |
| `stdio-to-uds` / `uds` | stdio↔Unix domain socket 桥 |
| `process-hardening` | 进程加固(预 main 加固入口) |
| `diagnostics` / `analytics` / `feedback` | 诊断数据 / 遥测分析 / 反馈上报 |
| `terminal-detection` | 终端能力检测 |
| `file-watcher` / `file-system` | 文件监听 / 文件系统抽象 |
| `git-utils` | git 仓库工具 |
| `external-agent-migration` | 从外部 agent(如 Claude Code)迁移会话 |
| `utils/*`(约 25 个) | absolute-path、image、pty、fuzzy-match、stream-parser、template、output-truncation 等小工具库 |
| `codex-experimental-api-macros` | `#[experimental(...)]` 宏(app-server 实验面) |
| `cloud-tasks(-client/-mock-client)` / `cloud-config` | 云端任务与配置 |
| `backend-client` / `codex-backend-openapi-models` | OpenAI 后端 API 客户端与模型 |
| `arg0` 见 [04 §7](./04-entrypoints-and-frontends.md) | 单二进制多身份 |
