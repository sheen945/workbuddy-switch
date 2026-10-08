# workbuddy-switch

WorkBuddy / CodeBuddy CLI / CodeBuddy CN IDE 多账号切换桌面应用（Tauri 2 + Rust + React），内置积分到期监控、自动签到与 CodeBuddy CLI 自动轮换。

## 简介

workbuddy-switch 是一款面向 WorkBuddy 重度用户的桌面工具，解决"多账号共享一台机器"的实际痛点：

- WorkBuddy 桌面端同一时间只能登录一个账号，手动切换需要替换认证文件并重启，繁琐且易出错；
- 多账号的积分资源各自独立到期，不盯紧就会过期浪费；
- CodeBuddy CLI / CodeBuddy CN IDE 与 WorkBuddy 桌面端账号体系相互独立，需要分别维护。

本应用把这三件事统一到一个界面：多账号共享登录态（`workbuddy-desktop.info`）、一键切换 WorkBuddy 登录账号（可顺带把当前账号的会话复制给目标账号）、积分到期高亮与排序、自动签到与 Token 保活，并能把"积分最紧迫"的账号自动设为 CodeBuddy CLI 后续会话的默认账号。

适合人群：同时拥有多个 WorkBuddy / CodeBuddy 账号、希望最大化利用各账号积分额度的 macOS / Windows / Linux 用户。

应用提供两种形态：

- **桌面 App**：Tauri 2 原生应用（推荐日常使用），从 GitHub Releases 下载安装包
- **npm / webui**：`npm i -g workbuddy-switch` 后在浏览器中使用同一套账号管理能力

## 功能

| 模块 | 说明 |
| --- | --- |
| 账号管理 | OAuth 扫码登录（device flow）、从本机导入、手动添加 token、删除账号 |
| 账号切换 | 备份认证文件 → 关闭 WorkBuddy → 写入目标账号 → 重启，切换过程实时进度反馈 |
| 会话复制 | 将当前账号勾选的会话以新 id 复制给目标账号（jsonl 正文 + `workbuddy.db` 索引 + edge-sync 注册，云端归属目标账号） |
| 自动签到 | 默认开启；启动时立即检查，运行期间每 30 分钟自动补签；支持一键全部签到；保留 30 天签到日志 |
| Token 保活 | 惰性刷新（操作前剩余不足阈值则刷新）+ 每日保活（默认每天无条件刷新一次；阈值 >0 时仅刷新剩余不足该天数的账号），避免 refresh token 过期 |
| 积分到期查询 | 自动查询每个账号的积分资源、剩余量和到期时间；7 天内到期高亮，并按到期紧迫度排序，最前者标记"建议优先使用" |
| 积分统计 | 汇总 WorkBuddy 官方请求用量：每日趋势、模型分布、账号消耗与请求明细；官方数据不可用时明确回退到本地余额快照观察 |
| Token 统计 | 分别查看 WorkBuddy、CodeBuddy CLI 与 CodeBuddy IDE 的 Token 总览；输入/输出/缓存读写按 K/M/B 展示，趋势图合并呈现每日构成与调用次数，另提供构成占比、活跃热力图、项目/模型 Top 10 与会话排行 |
| CodeBuddy CLI 集成 | 与 WorkBuddy 复用同一账号库但默认账号独立；macOS/Linux 经 `apiKeyHelper`、Windows 经 `settings.json` 的 `env.CODEBUDDY_AUTH_TOKEN` 设置后续会话使用的账号；任何平台都不会修改正在运行的当前会话 |
| CodeBuddy CN IDE 集成 | 向 CodeBuddy CN 桌面客户端（www.codebuddy.cn）注入 Safe Storage 凭证（`state.vscdb` / `planning-genie.new.accessTokencn`）并重启 IDE；与 CodeBuddy CLI、国际版 CodeBuddy 相互独立 |
| 自动轮换 | 后台定时把 CodeBuddy CLI 的后续启动账号设为积分最紧迫（最早到期且仍有剩余）的账号，防止积分过期浪费 |
| 自动更新 | 配置 GitHub Releases 源检查新版本；整包更新经 tauri-updater 签名校验 |
| 权限检测 | macOS 授权引导（App 管理 / 完全磁盘访问拖拽授权 + 自动检测） |

## 工作原理与技术栈

### 技术栈

- **后端**：Rust workspace（Cargo），Tauri 2（含 tray-icon）
  - `crates/wb-switch-core`：全部业务逻辑，按模块划分（account、auth_file、checkin、codebuddy_cli、codebuddy_cn_ide、config、credit_usage、credits、export_import、oauth、official_usage、process、refresh、rotate、session、switch、token_stats、travel、update、vscode_cn_inject 等 20 个模块）
  - `crates/wb-switch-server`：axum HTTP 服务 + rust-embed 内嵌前端产物，为 npm/webui 形态提供与桌面端一致的能力
  - `src-tauri`：薄命令封装层，将 core 模块暴露为 Tauri commands；集成 tauri-plugin-opener / updater / notification / dialog / autostart
- **前端**：React 19 + TypeScript + Vite 7，Tailwind CSS 4 + Radix UI 组件，zustand 状态管理，recharts 图表；`src/lib/api.ts` 为双通道封装——桌面端走 Tauri `invoke`，webui 走 HTTP `fetch`，页面代码完全复用
- **npm 分发**：主包 `workbuddy-switch` 通过 `optionalDependencies` 依赖 4 个平台二进制包（`workbuddy-switch-darwin-arm64` / `-darwin-x64` / `-win32-x64` / `-linux-x64`），postinstall 脚本按平台选择可执行文件
- **更新与签名**：Tauri updater 公钥校验（私钥 `TAURI_SIGNING_PRIVATE_KEY` 在发布时注入），更新清单由 `scripts/gen-update-json.sh` 与 `scripts/merge-update-manifests.py` 生成

### 关键机制

- **共享登录态**：所有账号保存在本地账号库，切换 = 备份当前认证文件 → 退出 WorkBuddy 进程 → 写入目标账号认证 → 重启应用
- **自动轮换策略**：每次检查按 7 条规则顺序决策（见下文「自动轮换策略」），只更新后续会话的默认账号，绝不热切换正在运行的会话
- **CodeBuddy CLI 接入**：macOS/Linux 写 `apiKeyHelper`（CLI 每次启动新会话时调用取 token），Windows 写 `~/.codebuddy/settings.json` 的 `env.CODEBUDDY_AUTH_TOKEN`（保留其他配置，不依赖 `.cmd` 跳板）

## 安装与使用

### 方式一：桌面 App（推荐）

前往 [GitHub Releases](https://github.com/sheen945/workbuddy-switch/releases/latest) 下载对应平台安装包：

| 平台 | 安装包 | 安装方式 |
| --- | --- | --- |
| macOS Apple Silicon（arm64） | `workbuddy-switch_<版本>_aarch64.dmg` | 打开 DMG，将 `workbuddy-switch.app` 拖入「应用程序」 |
| macOS Intel（x86_64） | `workbuddy-switch_<版本>_x86_64.dmg` | 同上 |
| Windows x64 | `workbuddy-switch_<版本>_x64-setup.exe` | 运行安装程序并按提示完成安装 |
| Linux x64 | `workbuddy-switch_<版本>_amd64.deb` / `_amd64.AppImage` | Debian/Ubuntu 安装 `.deb`；其他发行版给 AppImage 加执行权限后直接运行 |

macOS 首次启动若提示无法验证开发者：在 Finder 中按住 Control 点击应用选「打开」，或前往「系统设置 → 隐私与安全性」选择「仍要打开」。仅当安装包来自官方 Releases 且系统仍提示「已损坏」时，再执行：

```bash
xattr -rd com.apple.quarantine "/Applications/workbuddy-switch.app"
```

### 方式二：npm / webui

```bash
npm i -g workbuddy-switch
workbuddy-switch              # 启动本地服务并自动打开浏览器
workbuddy-switch status       # 终端查看当前账号
```

webui 界面与桌面 App 一致：账号切换、积分到期监控、自动签到、会话复制、Token 统计与 token 保活均可使用。

### 日常使用流程

1. **添加账号**：账号页 →「扫码登录」（OAuth device flow）或「从本机导入」「手动添加」
2. **切换账号**：账号卡片 →「切换」，可勾选复制当前会话
3. **自动签到**：账号页可直接开关；设置页可调整保活参数、立即签到并查看日志
4. **积分到期**：账号页自动查询各账号积分资源；「刷新积分」手动更新；临期资源高亮并按最近到期排序
5. **积分统计**：侧栏「积分统计」查看总览、近 30 天趋势、模型分类、账号消耗与请求明细；筛选不重复请求官方接口，点「刷新统计」才重新采集
6. **Token 统计**：侧栏「Token 统计」选择来源（WorkBuddy / CodeBuddy CLI / CodeBuddy IDE）查看
7. **CodeBuddy CN IDE**：账号卡片一键切换国内版桌面客户端；切换会关闭并重启 CodeBuddy CN，把所选账号写入 `~/Library/Application Support/CodeBuddy CN` 的登录态；首次使用前请先手动打开并登录一次以生成 Keychain Safe Storage
8. **CodeBuddy CLI**：账号页一键接入/更新认证。「切换 CodeBuddy」只更新后续加载会话使用的默认账号；请由 ACP 重新加载会话或重启 CLI 后生效。普通 CLI 在同一进程内执行 `/resume` 不保证重新读取认证配置
9. **自动轮换**：设置 → CodeBuddy CLI 自动轮换，开启后后台按间隔检查
10. **更新**：应用自动检查 GitHub Releases；发现新版本后可在左下角直接升级，或从设置页打开 Release 页面手动下载

### 自动轮换策略

目标是防止积分过期浪费：后台定时查询所有账号的积分到期情况，把 CodeBuddy CLI 后续会话的默认账号设为「最紧迫」的账号。为避免默认账号频繁变化，每次检查按以下顺序决策：

1. **有效账号**：查询成功、未过期、有剩余积分的账号才可被选为目标
2. **紧迫度检查**：最紧迫账号剩余时间超过 `min_urgency_hours`（默认 72 小时）→ 不切
3. **已是目标**：CLI 默认账号就是最紧迫账号 → 不切
4. **冷却期**：切换后 `cooldown_minutes`（默认 120）内不重复切
5. **活跃保护**：最近 `active_guard_minutes`（默认 30）内 CLI 会话有写入（正在对话）→ 不切
6. **价值过滤**：目标账号剩余积分低于 `min_remaining_credits` → 不切（默认 0 关闭；每次检查会把各账号剩余积分写入日志，可据此调整）
7. **防抖动**：目标比当前早到期但差异小于 `min_gap_hours`（默认 24）→ 不切

> **生效边界**：轮换只更新后续 restore/load 使用的默认账号，不会热切换当前会话。macOS/Linux 下一次 helper 执行读取最新账号；Windows 会把最新 Token 写入 settings。正在运行的会话继续使用启动时取得的账号。

配置项：`check_interval_minutes`（默认 5）、`cooldown_minutes`、`min_urgency_hours`、`active_guard_minutes`、`min_remaining_credits`、`min_gap_hours`。可在设置页调整，或直接编辑 `~/.wb-switch/auto_rotate_config.json`。

## 从源码构建

环境要求：Node.js ≥ 20、Rust stable 工具链；macOS / Windows / Linux 均可。

```bash
git clone https://github.com/sheen945/workbuddy-switch.git
cd workbuddy-switch
npm install

# 开发模式（前端热更新 + Tauri 窗口）
npm run tauri dev

# 仅构建前端产物
npm run build

# 打包 macOS .app（debug，签名私钥默认读取 ~/.wb-switch/wb-switch-updater.key）
npm run build:app

# 发布构建（release）
npm run build:app:release
```

发布流程、更新清单生成（`scripts/gen-update-json.sh`、`scripts/merge-update-manifests.py`）、npm webui 发布（`cd npm && npm publish`）等细节见 [docs/DEVELOPMENT.md](./docs/DEVELOPMENT.md)。

> 注意：`src-tauri` 默认启用 `custom-protocol` feature；若手工 `cargo build --release` 而未启用该 feature，产物会是加载 `http://localhost:1420` 的开发模式可执行文件，并非独立应用。请使用上面的 npm 脚本构建。

## 项目结构

```
workbuddy-switch/
├── src-tauri/            # Tauri 壳：命令封装、插件、应用配置（tauri.conf.json）
├── crates/
│   ├── wb-switch-core/   # 核心业务库：20 个功能模块（账号/切换/签到/轮换/统计等）
│   └── wb-switch-server/ # axum HTTP 服务 + rust-embed，npm/webui 形态的后端
├── src/                  # React 前端
│   ├── pages/            # 账号、积分统计、Token 统计、设置四个页面
│   ├── components/       # 账号卡片、OAuth 登录、切换/导入/导出/更新对话框、ui 组件
│   ├── lib/              # api.ts（Tauri/HTTP 双通道）、demo-mode、theme、types、update
│   └── stores/           # zustand 状态（accounts 等）
├── npm/                  # npm 包：bin 入口、postinstall、平台二进制 optionalDependencies
├── scripts/              # 构建与发布脚本（make-dmg、gen-update-json、fix-app、版本 bump 等）
├── docs/                 # DEVELOPMENT.md 与界面截图
├── public/               # 图标等静态资源
├── Cargo.toml            # Rust workspace 定义
└── LICENSE               # MIT
```

## 注意事项

- **macOS 权限**：切换账号需写入 WorkBuddy 认证文件，macOS 要求授权「App 管理」（或「完全磁盘访问」）。首次切换报「无权限」时按引导打开系统设置授权，授权后重启本应用生效；设置页「权限检测」可随时验证。webui 模式由启动服务的终端进程权限决定。
- **会话切换边界**：CodeBuddy CLI / 自动轮换只影响后续会话，当前运行中的会话不会被切换，需重新加载会话或重启 CLI。
- **CodeBuddy CN IDE 前置条件**：首次注入前需手动打开并登录一次 CodeBuddy CN，以生成 Keychain Safe Storage。
- **隐私**：账号库、密钥、token 文件均在本机（`~/.wb-switch/` 等），发布前仓库已配置 gitignore 并有 token 扫描流程；请勿把 `accounts.json`、密钥文件提交到任何公开位置。
- **在线演示**：[GitHub Pages 演示](https://sheen945.github.io/workbuddy-switch/)为只读模式，账号、积分与请求记录均为虚构数据，所有业务操作已禁用。

## 致谢

感谢 [Linux.do](https://linux.do) 社区。

## 许可

[MIT](./LICENSE)

## 作者

sheen945
