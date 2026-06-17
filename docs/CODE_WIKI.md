# Codex++ Code Wiki

> **项目原名**: trae-agent
> **GitHub**: https://github.com/bytedance/trae-agent
> **当前版本**: 1.2.9
> **技术栈**: Rust (后端) + Tauri + React (前端) + Python (部分脚本)

## 项目概述

Codex++ 是面向 Codex App 的外部增强启动器和管理工具。它不修改 Codex App 原始安装文件，而是通过外部 launcher 启动 Codex，并使用 Chromium DevTools Protocol (CDP) 注入增强脚本。

**核心特性**:
- 中转注入模式：支持多个中转配置，写入 `CodexPlusPlus` provider
- 传统增强模式：插件入口解锁、会话删除、Markdown 导出、项目移动、Timeline 等
- 用户脚本管理：可在启动时注入自定义脚本
- Provider 同步：启动前同步本地会话 metadata
- Zed 远程开发集成
- GitHub Release 自动更新

---

## 整体架构

### 工作区结构

```
/workspace
├── Cargo.toml                    # Rust workspace 根配置
├── Cargo.lock
├── apps/
│   ├── codex-plus-launcher/      # 静默启动入口 (CLI)
│   └── codex-plus-manager/        # Tauri 管理工具 (GUI)
├── crates/
│   ├── codex-plus-core/           # 核心逻辑库
│   └── codex-plus-data/           # 数据处理库
├── assets/
│   └── inject/
│       └── renderer-inject.js    # 注入到 Codex 渲染端的增强脚本
├── scripts/installer/            # 安装包制作脚本
└── docs/                         # 设计文档和发布说明
```

### 模块依赖关系

```
┌─────────────────────────────────────────────────────────────────┐
│                        apps/codex-plus-launcher                  │
│  (静默启动器 - 直接基于 codex-plus-core 实现完整启动流程)         │
└─────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                        codex-plus-core                           │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │   launcher   │  │   bridge     │  │   routes              │  │
│  │   (启动)     │  │   (CDP桥接)  │  │   (HTTP请求处理)      │  │
│  └──────────────┘  └──────────────┘  └───────────────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │   settings   │  │   cdp        │  │   protocol_proxy      │  │
│  │   (配置)     │  │   (调试协议) │  │   (协议代理)          │  │
│  └──────────────┘  └──────────────┘  └───────────────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │   install    │  │   user_scripts│ │   upstream_worktree │  │
│  │   (安装)     │  │   (用户脚本)  │  │   (Git worktree)     │  │
│  └──────────────┘  └──────────────┘  └───────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                        codex-plus-data                           │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │   storage    │  │   provider_sync│ │   markdown           │  │
│  │   (会话存储) │  │   (供应商同步) │ │   (Markdown导出)     │  │
│  └──────────────┘  └──────────────┘  └───────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 主要模块职责

### 1. apps/codex-plus-launcher

**职责**: 静默启动入口，不显示管理界面，只负责启动 Codex 并注入增强功能。

**入口文件**: `apps/codex-plus-launcher/src/main.rs`

**关键类型**:

| 类型 | 说明 |
|------|------|
| `LauncherHooks` | 实现 `LaunchHooks` trait，协调 core 逻辑、运行时服务和数据服务 |
| `LauncherDataService` | 实现 `BridgeDataService`，提供会话删除、导出、移动等数据操作 |
| `LauncherRuntimeService` | 实现 `BridgeRuntimeService`，提供用户脚本、DevTools、管理工具等运行时功能 |

**启动流程**:
1. 获取单实例锁 (`acquire_single_instance_guard`)
2. 检查更新 (`notify_manager_when_update_available`)
3. 调用 `launch_and_inject_with_hooks` 启动 Codex 并注入
4. 等待 Codex 退出

### 2. apps/codex-plus-manager

**职责**: Tauri 控制面板，用于启动、检查、修复、更新、配置中转注入、管理增强功能和用户脚本。

**技术栈**: Tauri 2.x + React + TypeScript

**主要功能**:
- 后端状态监控
- 中转注入配置
- 增强功能开关
- 用户脚本管理
- 更新检查

### 3. crates/codex-plus-core

**职责**: 核心逻辑库，包含启动、注入、配置、更新、桥接等核心功能。

#### 3.1 launcher 模块

**文件**: `crates/codex-plus-core/src/launcher.rs`

**核心函数**:

```rust
// 主启动函数
pub async fn launch_and_inject(options: LaunchOptions) -> anyhow::Result<LaunchHandle>
pub async fn launch_and_inject_with_hooks<H>(options: LaunchOptions, hooks: H) -> anyhow::Result<LaunchHandle>

// Codex 启动命令构建
pub fn build_codex_command(app_dir: &Path, debug_port: u16, extra_args: &[String]) -> Vec<String>
pub fn build_macos_open_command(app_dir: &Path, debug_port: u16, extra_args: &[String]) -> Vec<String>

// 注入相关
async fn retry_injection(debug_port: u16, helper_port: u16) -> anyhow::Result<()>
pub async fn check_and_reinject_bridge(debug_port: u16, helper_port: u16) -> bool
```

**关键类型**:

| 类型 | 说明 |
|------|------|
| `LaunchOptions` | 启动选项，包含 app_dir、debug_port、helper_port、status_store |
| `LaunchHandle` | 启动句柄，包含端口、目录、启动信息等 |
| `CodexLaunch` | Codex 启动方式 (`Process` 或 `PackagedActivation`) |
| `ProcessWaitStrategy` | 进程等待策略 (`TrackedChild` 或 `ExternalWaitCommand`) |
| `DefaultLaunchHooks` | 默认启动钩子实现 |

#### 3.2 bridge 模块

**文件**: `crates/codex-plus-core/src/bridge.rs`

**职责**: 通过 WebSocket 连接 Codex 的 CDP (Chromium DevTools Protocol)，在渲染进程中安装桥接脚本，实现前后端通信。

**核心函数**:

```rust
// 桥接脚本构建
pub fn build_bridge_script(binding_name: &str) -> String
pub fn bridge_health_check_script() -> &'static str

// CDP 命令执行
pub async fn evaluate_script(websocket_url: &str, script: &str) -> anyhow::Result<Value>
pub async fn evaluate_script_with_await_promise(websocket_url: &str, script: &str, await_promise: bool) -> anyhow::Result<Value>

// 桥接安装
pub async fn install_bridge(
    websocket_url: &str,
    binding_name: &str,
    handler: BridgeHandler,
    new_document_scripts: &[String],
) -> anyhow::Result<()>
```

**桥接协议**:
- 绑定名称: `codexSessionDeleteV2`
- 通过 `Runtime.addBinding` 在渲染进程注册 JS 绑定
- 渲染进程通过 `window.{binding_name}()` 调用 Rust 后端
- 后端通过 `Runtime.evaluate` 执行 JS 回调

#### 3.3 routes 模块

**文件**: `crates/codex-plus-core/src/routes.rs`

**职责**: 处理通过 bridge 收到的 HTTP 请求，将请求路由到对应的服务。

**核心 trait**:

```rust
pub trait BridgeSettingsService: Send + Sync {
    async fn get_settings(&self) -> anyhow::Result<BackendSettings>;
    async fn set_settings(&self, payload: Value) -> anyhow::Result<BackendSettings>;
}

pub trait BridgeRuntimeService: Send + Sync {
    async fn user_script_inventory(&self) -> anyhow::Result<Value>;
    async fn backend_status(&self) -> anyhow::Result<Value>;
    async fn zed_remote_status(&self) -> anyhow::Result<Value>;
    async fn upstream_worktree_status(&self) -> anyhow::Result<Value>;
    // ... 更多方法
}

pub trait BridgeDataService: Send + Sync {
    async fn delete(&self, session: SessionRef) -> anyhow::Result<DeleteResult>;
    async fn export_markdown(&self, session: SessionRef) -> anyhow::Result<ExportResult>;
    async fn thread_usage_history(&self, session: SessionRef) -> anyhow::Result<Value>;
    // ... 更多方法
}
```

**路由路径**:

| 路径 | 服务 | 说明 |
|------|------|------|
| `/settings/get` | BridgeSettingsService | 获取设置 |
| `/settings/set` | BridgeSettingsService | 更新设置 |
| `/backend/status` | BridgeRuntimeService | 后端状态 |
| `/delete` | BridgeDataService | 删除会话 |
| `/export-markdown` | BridgeDataService | 导出 Markdown |
| `/zed-remote/*` | BridgeRuntimeService | Zed 远程集成 |
| `/upstream-worktree/*` | BridgeRuntimeService | Git worktree 管理 |

#### 3.4 settings 模块

**文件**: `crates/codex-plus-core/src/settings.rs`

**职责**: 配置管理，读取和保存用户设置。

**关键类型**:

```rust
pub struct BackendSettings {
    pub codex_app_path: String,
    pub enhancements_enabled: bool,
    pub relay_profiles_enabled: bool,
    pub relay_profiles: Vec<RelayProfile>,
    pub active_relay_id: String,
    pub computer_use_guard_enabled: bool,
    // ... 大量功能开关
}

pub struct RelayProfile {
    pub id: String,
    pub name: String,
    pub base_url: String,
    pub api_key: String,
    pub protocol: RelayProtocol,  // Responses 或 ChatCompletions
    pub relay_mode: RelayMode,    // Official、MixedApi、PureApi
    pub config_contents: String,
    pub auth_contents: String,
    // ...
}

pub struct SettingsStore {
    path: PathBuf,
}
```

#### 3.5 cdp 模块

**文件**: `crates/codex-plus-core/src/cdp.rs`

**职责**: CDP 协议封装，列出目标、选择可注入页面等。

**核心函数**:

```rust
pub async fn list_targets(debug_port: u16) -> anyhow::Result<Vec<CdpTarget>>
pub fn pick_injectable_codex_page_target(targets: &[CdpTarget]) -> anyhow::Result<CdpTarget>

pub struct CdpTarget {
    pub id: String,
    pub target_type: String,
    pub title: String,
    pub url: String,
    pub web_socket_debugger_url: Option<String>,
}
```

#### 3.6 protocol_proxy 模块

**文件**: `crates/codex-plus-core/src/protocol_proxy.rs`

**职责**: 协议代理，将 Chat Completions 请求转换为 Responses 协议，或透传。

**支持的代理路径**:
- `/v1/chat/completions` → 上游
- `/v1/responses` → 上游
- `/v1/models` → 上游

#### 3.7 其他核心模块

| 模块 | 文件 | 职责 |
|------|------|------|
| `ads` | `ads.rs` | 远程广告列表获取 |
| `app_paths` | `app_paths.rs` | Codex 应用路径解析 |
| `assets` | `assets.rs` | 注入脚本资源 |
| `cli_wrapper` | `cli_wrapper.rs` | CLI 包装器 |
| `codex_sqlite` | `codex_sqlite.rs` | Codex SQLite 数据库操作 |
| `computer_use_guard` | `computer_use_guard.rs` | Computer Use 配置守护 |
| `diagnostic_log` | `diagnostic_log.rs` | 诊断日志 |
| `install` | `install/mod.rs` | 安装相关 |
| `model_catalog` | `model_catalog.rs` | 模型目录 |
| `models` | `models.rs` | 共享数据模型 |
| `paths` | `paths.rs` | 路径工具 |
| `ports` | `ports.rs` | 端口管理 |
| `proxy` | `proxy.rs` | 代理基础 |
| `relay_config` | `relay_config.rs` | 中转配置管理 |
| `relay_switch` | `relay_switch.rs` | 中转切换 |
| `script_market` | `script_market.rs` | 脚本市场 |
| `status` | `status.rs` | 状态存储 |
| `update` | `update.rs` | 更新检查 |
| `upstream_worktree` | `upstream_worktree.rs` | Git worktree 管理 |
| `user_scripts` | `user_scripts.rs` | 用户脚本管理 |
| `version` | `version.rs` | 版本信息 |
| `watcher` | `watcher.rs` | 进程监控 |
| `zed_remote` | `zed_remote.rs` | Zed 远程集成 |

### 4. crates/codex-plus-data

**职责**: 会话数据处理、导出、Provider 同步。

#### 4.1 storage 模块

**文件**: `crates/codex-plus-data/src/storage.rs`

**职责**: SQLite 会话存储适配器，支持会话删除、撤销、移动、排序等。

**关键类型**:

```rust
pub struct SQLiteStorageAdapter {
    db_path: PathBuf,
    backup_store: BackupStore,
}

pub struct LocalSession {
    pub id: String,
    pub title: String,
    pub cwd: String,
    pub model_provider: String,
    pub archived: bool,
    pub updated_at_ms: Option<i64>,
    pub rollout_path: String,
    pub db_path: String,
}
```

**关键函数**:

```rust
pub fn delete_local_from_paths(...) -> DeleteResult
pub fn delete_local(&self, session: &SessionRef) -> DeleteResult
pub fn undo(&self, token: &str) -> DeleteResult
pub fn list_local_sessions(&self) -> anyhow::Result<Vec<LocalSession>>
pub fn find_archived_thread_by_title(&self, title: &str) -> Option<SessionRef>
pub fn move_codex_thread_workspace(&self, session: &SessionRef, target_cwd: &str) -> Value
pub fn codex_thread_sort_keys(&self, sessions: &[SessionRef]) -> Value
pub fn codex_thread_usage_history(&self, session: &SessionRef) -> Value
```

**SchemaKind 枚举**:

| 变体 | 说明 |
|------|------|
| `GenericSessions` | 通用会话模式 |
| `CodexThreads` | Codex 线程模式 |
| `CodexAutomationRuns` | Codex 自动化运行模式 |

#### 4.2 provider_sync 模块

**文件**: `crates/codex-plus-data/src/provider_sync.rs`

**职责**: 在切换供应商时同步本地会话元数据，确保旧会话在切换供应商后仍可见。

**核心函数**:

```rust
pub fn run_provider_sync(codex_home: Option<&Path>) -> ProviderSyncResult
pub fn run_provider_sync_with_target(
    codex_home: Option<&Path>,
    explicit_target_provider: Option<&str>,
) -> ProviderSyncResult
pub fn load_provider_sync_targets(codex_home: Option<&Path>) -> ProviderSyncTargetList
```

**同步内容**:
1. Rollout 文件中的 `session_meta` 记录
2. SQLite 数据库中的 `threads` 表
3. `.codex-global-state.json` 中的工作区根目录

#### 4.3 markdown 模块

**文件**: `crates/codex-plus-data/src/markdown.rs`

**职责**: 将 Codex 会话导出为 Markdown 格式。

```rust
pub struct MarkdownExportService {
    db_path: Option<PathBuf>,
}

pub fn export(&self, session: &SessionRef) -> anyhow::Result<ExportResult>
```

#### 4.4 backup 模块

**文件**: `crates/codex-plus-data/src/backup.rs`

**职责**: 会话删除前的备份存储，支持撤销操作。

```rust
pub struct BackupStore {
    backup_dir: PathBuf,
}

pub fn write_backup(&self, session_id: &str, db_path: &Path, data: Value) -> anyhow::Result<String>
pub fn read_backup(&self, token: &str) -> anyhow::Result<Value>
```

---

## 关键数据模型

### models 模块 (`crates/codex-plus-core/src/models.rs`)

```rust
pub struct SessionRef {
    pub session_id: String,
    pub title: String,
}

pub struct DeleteResult {
    pub status: DeleteStatus,
    pub session_id: String,
    pub message: String,
    pub undo_token: Option<String>,
    pub backup_path: Option<String>,
}

pub enum DeleteStatus {
    LocalDeleted,
    Undone,
    Failed,
}

pub struct ExportResult {
    pub status: ExportStatus,
    pub session_id: String,
    pub message: String,
    pub filename: Option<String>,
    pub markdown: Option<String>,
}
```

---

## 通信协议

### Helper HTTP 服务

在 `helper_port` (默认 57321) 上启动 HTTP 服务器，处理以下路径：

| 路径 | 方法 | 说明 |
|------|------|------|
| `/backend/status` | GET/POST | 后端状态检查 |
| `/backend/repair` | GET/POST | 后端修复 |
| `/diagnostics/log` | POST | 诊断日志 |
| `/overlay/image` | GET | 图片覆盖层 |
| `/v1/chat/completions` | POST | Chat Completions 代理 |
| `/v1/responses` | POST | Responses 协议代理 |
| `/v1/models` | GET | 模型列表代理 |

### Bridge 通信

```
渲染进程 (renderer-inject.js)
       │
       │ window.codexSessionDeleteV2(JSON.stringify({id, path, payload}))
       ▼
Rust 后端 (bridge.rs)
       │
       │ handle_bridge_request()
       ▼
routes.rs 路由分发
       │
       ├──► BridgeSettingsService (设置)
       ├──► BridgeRuntimeService (运行时)
       └──► BridgeDataService (数据)
```

---

## 配置文件

### Codex++ 设置存储

- **路径**: `~/.config/Codex++/settings.json` (Linux/macOS) 或 `%APPDATA%/Codex++/settings.json` (Windows)
- **格式**: JSON

### Relay 配置

```json
{
  "relayProfiles": [{
    "id": "default",
    "name": "默认中转",
    "baseUrl": "https://example.com/v1",
    "apiKey": "sk-...",
    "protocol": "responses",
    "relayMode": "mixedApi"
  }],
  "activeRelayId": "default"
}
```

### Codex 原生配置

- **config.toml**: `~/.codex/config.toml`
- **auth.json**: `~/.codex/auth.json`
- **SQLite**: `~/.codex/sqlite/*.db`

---

## 运行方式

### 开发环境检查

```bash
# 前端检查
cd apps/codex-plus-manager
npm install
npm run check
npm run vite:build

# Rust 检查
cargo fmt --check
cargo test
cargo build --release
```

### 启动参数

**Launcher 支持的参数**:

| 参数 | 说明 |
|------|------|
| `--app-path <path>` | 指定 Codex 应用目录 |
| `--debug-port <port>` | CDP 调试端口 (默认 9229) |
| `--helper-port <port>` | Helper 服务端口 (默认 57321) |

### 单实例机制

Launcher 使用端口锁定实现单实例：
1. 尝试在 `LAUNCHER_GUARD_PORT` 绑定端口
2. 如果失败，说明已有实例运行
3. 激活已有 Codex 窗口而非启动新实例

---

## 依赖关系

### Workspace 依赖

```toml
[workspace.dependencies]
anyhow = "1"
base64 = "0.22"
directories = "6"
reqwest = { version = "0.12", features = ["json", "stream", "rustls-tls", "system-proxy"] }
rusqlite = { version = "0.32", features = ["bundled"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["macros", "process", "rt-multi-thread", "time"] }
tokio-tungstenite = { version = "0.26", features = ["rustls-tls-webpki-roots"] }
toml = "0.8"
toml_edit = "0.22"
uuid = { version = "1", features = ["v4"] }
```

### 模块间依赖

```
codex-plus-launcher
  ├── codex-plus-core (启动、注入、桥接)
  └── codex-plus-data (Provider同步、数据操作)

codex-plus-core
  └── (无内部依赖)

codex-plus-data
  └── codex-plus-core (数据库路径、Provider同步目标)
```

---

## 注入脚本 (renderer-inject.js)

**位置**: `assets/inject/renderer-inject.js`

**功能**:
1. 解锁插件入口
2. 添加会话删除按钮
3. 支持 Markdown 导出
4. 会话移动功能
5. Timeline 显示
6. Zed Remote 打开入口
7. Upstream Worktree 创建

**注入时机**:
- 通过 `Page.addScriptToEvaluateOnNewDocument` 在每个新文档加载时执行
- 通过 `Runtime.evaluate` 在现有页面执行

---

## Windows 特定实现

### 特性

- `windows_subsystem = "windows"` 隐藏控制台窗口
- 使用 `Win32 API` 激活已安装的应用 (`IApplicationActivationManager`)
- `CREATE_NO_WINDOW` 标志防止产生控制台窗口
- 单实例通过 `LoopbackPortGuard` 实现

### 注册表路径

用于查找已安装的 Codex App:
- `HKEY_CURRENT_USER\Software\Classes\Local Settings\Software\Microsoft\Windows\CurrentVersion\AppModel\Repository\Packages`
- `HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall`

---

## macOS 特定实现

### 特性

- 使用 `open -W -a` 命令启动 `.app` 应用
- 通过 `osascript` 检测应用是否已运行
- 通过 `osascript` 退出已启动的应用
- Dock 图标隐藏 (静默入口)

### 应用包检测

检查 `/Applications/Codex.app` 是否存在

---

## 测试

### 单元测试

```bash
cargo test
```

### 集成测试

测试文件位于各 crate 的 `tests/` 目录:
- `crates/codex-plus-core/tests/`
- `crates/codex-plus-data/tests/`

### 覆盖的测试场景

- CDP 目标选择
- 桥接健康检查
- 设置加载/保存
- Provider 同步
- Markdown 导出
- 文件监控
- 中转配置
- 上游 Worktree

---

## 贡献指南

详见 [CONTRIBUTING.md](../../CONTRIBUTING.md)

### 开发设置

```bash
# 克隆仓库
git clone https://github.com/bytedance/trae-agent
cd trae-agent

# 安装 Rust 工具链 (1.85+)
rustup update

# 前端依赖
cd apps/codex-plus-manager
npm install

# 运行测试
cargo test

# 构建发布版本
cargo build --release
```

### 代码风格

- Rust: `cargo fmt` + `cargo clippy`
- TypeScript: ESLint + Prettier
- 提交前运行 pre-commit 钩子

---

## 常见问题

### Q: 如何确认注入成功？
A: 检查 Codex++ 菜单是否出现，或访问 `http://127.0.0.1:57321/backend/status`

### Q: 插件显示后端连不上？
A: 重启 Codex++，或在管理工具的"诊断"页面查看 `renderer.script_loaded`、`bridge.request` 日志

### Q: Provider 同步的作用？
A: 确保切换供应商后，旧会话仍可见且可续聊

---

## 版本历史

- **v1.2.9**: 最新版本
- 支持多 Provider 配置
- 增强的 Provider 同步
- Zed Remote 集成改进
- 上游 Worktree 管理
