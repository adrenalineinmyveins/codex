# 07 · 构建、测试与运行

## 1. 环境准备

- Rust 工具链:按 `codex-rs/rust-toolchain.toml`(CI 用 Bazel 内置 1.95.0);
- 系统要求(用户安装):macOS 12+ / Ubuntu 20.04+ / Win11(WSL2),4–8GB RAM;
- 推荐安装:`cargo install just dotslash cargo-nextest cargo-insta`(缺什么装什么);
- `just install` = 确认 rustup 工具链 + `cargo fetch`。

## 2. 本地构建与运行(Cargo 轨)

```bash
cd codex-rs
cargo build                          # 或 cargo build -p codex-cli
cargo run --bin codex -- "..."       # 从源码运行主 CLI

# 更常用(仓库根目录):
just codex <args>                    # = cargo run --bin codex
just exec <args>                     # = codex exec
just c                               # just codex 的别名
```

其他 recipe:`just file-search`、`just code-mode-host`、`just tui-with-exec-server`(先起 exec-server 再跑 TUI)、`just app-server-test-client`。

## 3. justfile 关键 recipe(默认工作目录 codex-rs)

| 分类 | recipe | 说明 |
|---|---|---|
| 格式化 | `just fmt` / `just fmt-check` | 统一格式化 justfile/Rust/Starlark/Python(调 `scripts/format.py`);**改完代码必须 `just fmt`** |
| Lint | `just clippy` / `just fix [-p <crate>]` | `cargo clippy --tests`;`fix` 自动修复(改动大前先 `-p` 限定范围) |
| 测试 | `just test [-p codex-<crate>]` | **强制 cargo-nextest**(`NEXTEST_PROFILE=local`,--no-fail-fast);不要直接 `cargo test` |
| 基准 | `just bench` / `bench-smoke` / `bench-e2e` | divan 基准 / 单迭代冒烟 / Bazel 端到端宏基准 |
| Bazel | `just bazel-codex` / `bazel-test` / `bazel-clippy` / `bazel-lock-update` | 见 §5 |
| Schema | `just write-config-schema` | 改 `ConfigToml` 后重新生成 `core/config.schema.json` |
| | `just write-app-server-schema [--experimental]` | 改 app-server API 后重新生成协议 fixtures |
| | `just write-hooks-schema` | hooks schema |
| 打包 | `just assemble-codex-package` | 本地组装 Codex 包(`scripts/build_codex_package.py`) |

## 4. 测试体系

### 运行方式
```bash
just test -p codex-tui        # 改了哪个 crate 就测哪个
just test -p codex-core
just test                     # 共享 crate(common/core/protocol)改动后才跑全量(先征得同意)
```
- nextest 配置:`codex-rs/.config/nextest.toml`(slow-timeout 30s、retries=1、按资源分 test-group,如 app_server_integration 串行);
- `RUST_MIN_STACK=8388608`(8MiB)由 just 自动设置。

### 测试金字塔
| 层 | 位置 | 工具/模式 |
|---|---|---|
| core 集成测试 | `codex-rs/core/tests/suite/`(~120 个主题模块) | `TestCodexBuilder::build_with_auto_env()` + wiremock `mount_sse_once` + `ResponseMock` 请求断言(见 [03 §5](./03-core-engine.md)) |
| app-server 集成测试 | `codex-rs/app-server/tests/suite/v2/`(约 100 文件) | `TestAppServer::builder().build()` + 公共 JSON-RPC API 驱动 |
| insta 快照 | `core/tests/suite/snapshots/`、`tui/**/snapshots/` 等 | UI/输出变更必须带快照;更新流程:`just test -p codex-tui` → `cargo insta pending-snapshots` → 审阅 `*.snap.new` → `cargo insta accept -p codex-tui` |
| 单元测试 | 各 crate `#[cfg(test)]`(新模块用 `#[path = "*_tests.rs"]` 独立文件) | `pretty_assertions::assert_eq`,整对象深比较 |
| Python | `.github/scripts/test_*.py`、`scripts/codex_package/test_*.py` | `just test-github-scripts` / unittest |
| TS SDK | `sdk/typescript/tests/*.test.ts` | jest + ts-jest;CI 注入 `CODEX_EXEC_PATH` |
| Python SDK | `sdk/python/tests/` | pytest;含真实 app-server 集成测试 |

### 测试编写要点(AGENTS.md)
- agent 逻辑变更 **必须**加 `core/suite` 集成测试;
- 跨平台支持 Linux/macOS/Windows;Wine 变体由 Bazel 生成;
- 二进制定位用 `codex_utils_cargo_bin::cargo_bin`;资源定位用 `find_resource!`;
- 沙箱环境感知:`CODEX_SANDBOX=seatbelt` / `CODEX_SANDBOX_NETWORK_DISABLED=1` 下相关测试自动跳过。

## 5. Bazel 轨(封闭构建 / CI / 发布)

```bash
just bazel-codex <args>            # bazel run //codex-rs/cli:codex
just bazel-test                    # bazel test //...(排除 v8/third_party)
just bazel-clippy                  # 三平台 clippy(--config=clippy)
just bazel-lock-update             # 改 Cargo.toml/Cargo.lock 后必须执行
just build-for-release             # //codex-rs/cli:release_binaries(6 平台)
```

要点:
- `MODULE.bazel`:hermetic LLVM 工具链、macOS SDK 拉取、MSVC/Windows SDK、`crate.from_cargo`(从 Cargo.lock 导入全部外部依赖,9 个平台 triple)、V8/rusty_v8 体系;
- `defs.bzl` 的 `codex_rust_crate` 宏:每个 crate 一行 BUILD.bazel 即生成与 Cargo 对齐的 library/binaries/unit-tests/integration-tests 目标(统一 `workspace_root_test` 启动器处理 cwd、INSTA、runfiles);
- `multiplatform_binaries`(6 平台)汇成 `release_binaries`;
- `.bazelrc`:BuildBuddy 缓存/RBE 配置、`ci-linux`/`ci-macos`/`ci-windows-cross`、clippy deny 列表与 Cargo lints 同步。

## 6. 配置与认证

- **配置文件**:`~/.codex/config.toml`(权威 schema:`codex-rs/core/config.schema.json`);支持 profile、分层合并、`-c key=value` 覆盖、cloud config bundle;
- **认证**(三种):`codex login` 浏览器 ChatGPT 登录(推荐)/ 设备码 / API key;凭据存 OS keyring 或 `~/.codex/auth.json`;
- **日志**:`RUST_LOG` + `-c log_dir=./.codex-log`(文件);`codex exec` 默认 `RUST_LOG=error`;也可 `just log` 读取 SQLite 状态库日志。

## 7. CI 工作流概览

| 工作流 | 触发 | 内容 |
|---|---|---|
| `blocking-ci.yml` | PR + main push | 聚合门:bazel / blob-size / cargo-deny / codespell / repo-checks / rust-ci / sdk |
| `rust-ci.yml` | PR(按路径) | cargo fmt --check、bench-smoke、cargo-shear、argument-comment-lint(三平台预编译) |
| `bazel.yml` | PR + main | **主验证路径**:macOS×2、Linux gnu+musl 的 `bazel test //...`;Windows 4 分片(gnullvm 交叉);三平台 clippy;release 构建验证 |
| `postmerge-ci.yml` | main push | 全量 cargo clippy 矩阵、分平台 nextest、Windows ARM64、remote-env 测试、v8-canary |
| `rust-release.yml` | tag `rust-v*.*.*` | 多目标构建 → 签名公证(macOS AKV/Linux cosign)→ codex-package tar.gz → GitHub Release → dotslash/R2/npm/winget → Python runtime wheel |
| `sdk.yml` | PR | Bazel 构建 codex 二进制 → TS SDK build/lint/test |

## 8. SDK 开发

### Python(`sdk/python/`,包 `openai-codex`)
```bash
cd sdk/python
uv sync --group dev
uv run pytest            # ruff format+lint / pytest
```
- 手写门面层(`src/openai_codex/`)+ 生成的协议模型(`generated/`,datamodel-code-generator,`test_contract_generation.py` 防漂移);
- 运行时依赖二进制 wheel `openai-codex-cli-bin`(与 CLI release 同步,由 rust-release 工作流产出)。

### TypeScript(`sdk/typescript/`,包 `@openai/codex-sdk`)
```bash
cd sdk/typescript
pnpm install && pnpm build && pnpm test    # jest;CODEX_EXEC_PATH 指向本地 codex 可跳过下载
```
- 机制:spawn `codex` 子进程 + stdin/stdout JSONL 事件协议(与 Python 走 app-server 不同)。

## 9. 常见开发流程速查

| 场景 | 步骤 |
|---|---|
| 改 core 逻辑 | 改码 → `just fmt` → `just test -p codex-core`(必要时加 suite 集成测试)→ `just fix -p codex-core` |
| 改 TUI | 改码 → `just test -p codex-tui` → 快照审阅/accept → `just fmt` |
| 改 app-server API | v2 加类型(camelCase/`#[ts(export_to = "v2/")]`)→ `just write-app-server-schema` → 更新 `app-server/README.md` → `just test -p codex-app-server-protocol` |
| 改配置 | 改 `ConfigToml` → `just write-config-schema` |
| 改依赖 | 改 Cargo.toml → `just bazel-lock-update`(锁文件随同提交) |
| 调试沙箱 | `codex sandbox`(profile 运行)、`codex debug sandbox`、`just codex doctor` |
