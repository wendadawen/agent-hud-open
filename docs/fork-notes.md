# 本 fork 说明

基于上游 [`jazzenchen/agent-hud-open`](https://github.com/jazzenchen/agent-hud-open) 的 **v0.6.9**（2026-11-20），只做一件事：让 Antigravity 的额度在 Agent HUD 里自动刷新。

## 为什么改

上游的 Antigravity 额度完全依赖 `agy` 交互式 CLI 的 **status-line 回调**：CLI 在刷新状态栏时把账号和额度推给 HUD。CLI 不活跃时，卡片上的读数会一直停在旧值（超过 5 分钟标 stale）。而 Antigravity IDE 2.34 的 language server 不再在命令行暴露 `--csrf_token`，HUD 的另一条本地服务发现路径也认不出它。

`agy -p /usage --output-format json` 是官方支持的只读命令：不开回合、不消耗额度、不留会话，能拿到完整的 5h / 周窗口。上游其实已经实现了它的解析（`src/crates/agents/src/antigravity/cli.rs`），但只把结果当作「完全没有其他读数时的一次性匿名兜底」，且明确禁用自动更新。

## 改了什么

- `src/crates/agents/src/antigravity/account.rs`
  - 每次账号读取（后台 job 已经在跑，不需要新建定时器或常驻进程）在用户授权允许时，调用一次 `agy -p /usage --output-format json`。
  - CLI 可执行文件优先取 `ANTIGRAVITY_CLI_PATH`，否则用官方安装脚本默认的 `~/.local/bin/agy`；文件不存在时零开销。
  - 运行失败不影响原有读数，旧的 status-line / language server 结果照常显示。
- `src/crates/agents/src/antigravity/cli.rs`
  - 新增 `identified()`：把 `/usage` 的匿名读数按已登录邮箱重新编号，使其与官方 status-line 报告**合并到同一张卡片**，而不是多出一个匿名账号。账号行仍由官方报告提供（它才知道套餐）。
- 测试：新增 4 个单元/回归测试（贴回账号、每轮刷新、关闭授权时不启动、CLI 缺失时零影响）。
- `.github/workflows/macos.yml`：macOS arm64 自动构建。

## 开关（无需新增设置项）

复用项目已有的「读取额度」授权（`quotaReading`，界面：设置 → 智能体 → Antigravity → 读取额度）：

| 级别 | 行为 |
|---|---|
| 全部 `full`（默认） | 轮询开启 ✅ |
| 官方 `official` | 轮询开启（命令行本来就是 CLI 自己的接口） |
| 关闭 `off` | 完全不读额度，也不启动 CLI |

本机默认是 `full`，所以**构建完就是开启状态**，不用额外配置。要临时关闭就把该下拉改成「关闭」。

## 构建（GitHub Actions）

`.github/workflows/macos.yml` 在 `macos-14`（Apple Silicon）上：

1. `pnpm install --frozen-lockfile` → `pnpm run build`
2. `cargo clippy --workspace --tests --locked -- -D warnings` 和 `cargo test --workspace --locked --no-fail-fast`
3. `pnpm run build:macos -- --bundle`
4. 上传 `agent-hud-open-macos-arm64` artifact（dmg / .app / provenance.json / SHA256SUMS），保留 30 天
5. 打 `v*` tag 时自动把 dmg 挂到 Release

触发方式：push 到 `main` / `fix/**`、PR、手动 `workflow_dispatch`、或推 tag。

## 本地开发

需要 Node.js 24+、pnpm（仓库 `packageManager` 已固定版本）、Rust 1.90+、Xcode Command Line Tools。

```bash
pnpm install --frozen-lockfile
pnpm run build
cargo test -p agent-hud-agents --lib antigravity --locked
pnpm run build:macos -- --bundle
```

只验证 Agents crate 可以跑 `cargo check -p agent-hud-agents --lib --locked`，十几秒。

## 安装注意

- 本 fork 与官方版共用 `CFBundleIdentifier = app.agenthud.mac` 和数据目录，所以设置、额度历史不会丢；但**不能同时运行两个副本**。
- 本地构建是 ad-hoc 签名，首次打开需要「右键 → 打开」。
- 版本号改为 `0.6.9-fork.1`（`package.json` 与 `tauri.conf.json`），避免显示成源码默认的 `0.1.0`。
- **bundle id 保持 `app.agenthud.mac` 不改**：它同时是数据目录名、Cursor 会话日志路径、`legacy_preferences` 的偏好域和 `data_folder.rs` 的测试断言；改了会丢设置、额度历史和会话统计，与「其他功能不变」冲突。隔离靠**别把官方版安装到同一路径**（或把本 fork 装到 `~/Applications`）。
- 开源构建本身**没有更新器**：`UpdatesPane.tsx` 写着 "Open builds have no update feed"，设置页显示 Unavailable，`capabilities.json` 把 updater feeds 列为未实现（planned）。所以不会被官方自动更新覆盖；需要注意的是**别手动安装官方 dmg / Homebrew 版**去覆盖 `/Applications/Agent HUD.app`。
- 以后同步上游：`git fetch upstream && git rebase upstream/main`，冲突集中在 `antigravity/account.rs` 与 `cli.rs`。

## 许可

Apache-2.0，与上游一致。改动仅在本 fork 内使用。
