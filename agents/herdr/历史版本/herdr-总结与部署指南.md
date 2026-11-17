# herdr 总体整理 + 两种部署场景说明

> 资料来源：herdr.dev 官网与 docs（install / quick-start / concepts / agents / connecting-machines / session-state / configuration / socket-api / integrations / plugins / compare）、GitHub 仓库 herdrdev/herdr、官方博客《Connecting the machines》(2026-09-07)、x-cmd 安装页、腾讯云开发者社区文章、vocus 文章、jacar.es 评测、53ai 文章，以及两篇实战文章：DevelopersIO《Make Mac mini a headless server and operate Claude Code from outside》(2026-08-07) 和 redock.dev《Run Codex on Phone with SSH, Tailscale and Herdr》(2026-08-30)。
> 两篇知乎文章需要登录才能阅读、YouTube 视频无法抓取，其内容已用官方文档和其他文章交叉覆盖。
> 整理日期：2026-09-22，对应 herdr v0.9.1。

---

## 一、herdr 是什么

**一句话**：herdr 是「AI 编程 agent 的运行时（runtime）」。它是一个常驻后台的服务器进程（herdr-server），真正拥有每个 agent 的终端（PTY）和子进程；你用的 TUI 客户端只是「接上去看一眼」的窗口。客户端关掉、SSH 断掉、笔记本合盖，agent 照常跑；从任何机器再 `herdr` 一下，现场原样回来。

官网原话：*Herdr is where your coding agents live... Walk away and they keep working. Come back from any machine and they're where you left them.*

### 1.1 基本事实

| 项目 | 内容 |
|---|---|
| 实现 | Rust 单二进制，约 20 MB，无 Node / Electron / 运行时依赖 |
| 许可证 | Apache 2.0（作者称「运行时保持开源」） |
| 版本 | v0.9.1（2026-09），0.9 引入多机器统一客户端 |
| 平台 | Linux / macOS（x86_64、aarch64）、Windows x86_64 |
| 社区 | 约 4 万 GitHub star，约 99 万安装，约 1200+ 插件；已进 Y Combinator，2026-09 完成 600 万美元种子轮 |
| 资源占用 | 空闲两个 pane：server 约 17 MB RSS，client 约 10 MB（jacar 实测 0.8.2） |
| 定位 | 不是 tmux 替代品，而是「tmux 之上的一层」：tmux 管终端，herdr 管终端里跑的 agent |

### 1.2 核心概念（五层结构）

| 概念 | 含义 |
|---|---|
| **Session** | 一个 herdr-server 的持久命名空间。默认 session 自动运行；命名 session（`--session lab`）有完全独立的 pane、socket 和状态 |
| **Workspace** | 项目级容器，建议「一个 repo / 一个任务 / 一次排查 = 一个 workspace」 |
| **Tab** | workspace 内的一套布局（如 agents / logs / servers） |
| **Pane** | 真正的终端进程（PTY），可左右/上下分割 |
| **Agent** | pane 里被识别出来的 agent 进程，有名字（`[a-z][a-z0-9_-]{0,31}`）和状态 |
| **Machine**（0.9 新增） | 通过 SSH 接入的远程 herdr-server，其 workspace / agent 与本地并列显示 |

ID 形式：`w1:p2` 表示 workspace 1 的 pane 2。ID 按 server 作用域，两台机器都可以有自己的 `w1:p1`。

### 1.3 Agent 状态（这是 herdr 最核心的价值）

| 状态 | 含义 | 侧边栏颜色（社区文章描述） |
|---|---|---|
| `working` | 正在干活 | 黄 |
| `blocked` | 等你批准 / 回答（如「是否允许删除文件」） | 红 |
| `done` | 干完了但你还没看 | 蓝 |
| `idle` | 空闲、已被看过 | 绿 |
| `unknown` | 认出是 agent 但无法判断状态 | 灰 |

状态会「向上冒泡」：pane → tab → workspace → machine，所以不用逐个 pane 巡视，侧边栏一眼看出谁卡住了。

**状态判定的三种权威来源**（按优先级）：
1. **生命周期 hook / 插件集成**：Pi、OMP、Kimi、OpenCode、Kilo、MastraCode 等，由 agent 自己上报 idle/working/blocked。
2. **屏幕 manifest + 前台进程**：识别 pane 的前台进程，再对终端底部实时画面做模式匹配。Claude Code、Codex 等走这条路。`blocked` 判定刻意保守，只在底部画面匹配到已知的审批提示时才标红。
3. **Session identity 集成**：Claude Code、Codex、Copilot、Cursor、Grok、Hermes 等，集成脚本把 agent 自己的会话 ID 报给 herdr，用于 server 重启后 `claude --resume` / `codex resume` 式的原生恢复。

manifest 随 herdr 内置并可远程自动更新；本地可用 `~/.config/herdr/agent-detection/<agent>.toml` 覆盖。

### 1.4 支持的 agent

官方列出的 agent kind：`pi, claude, codex, gemini, cursor, devin, agy(antigravity), cline, omp, mastracode, opencode, copilot, kimi, kiro, droid, amp, grok, hermes, kilo, qodercli, qwen, letta, maki, muse`，共 22+ 种开箱识别。不在列表里的 CLI 也能跑，只是标 `unknown`，或者你自己用 socket API 上报状态。

### 1.5 三条持久化路径（一定要分清）

| 场景 | 进程 | 布局 | 屏幕 | Agent |
|---|---|---|---|---|
| Detach / 重连 | 存活 | 保留 | 实时终端 | 保留 |
| Server 重启 / 主机重启 | **不存活** | 恢复 | 仅历史（默认关） | 靠原生会话恢复 |
| `herdr update`（无 handoff） | 视情况 | 恢复 | 仅历史 | 原生会话恢复 |
| `herdr update --handoff`（实验） | 尽力保活 | 保留 | 实时 | 保留 |

- **Detach 是最强的持久化**：`ctrl+b q` 或直接关终端 / SSH 掉线，server 和所有进程都在。
- **Server 重启只恢复结构**：workspace/tab/pane/cwd/布局/焦点从 `session.json` 重建，pane 变成新 shell。`resume_agents_on_restore = true`（默认开）会用集成上报的会话 ID 重新拉起 agent 对话，这是「重建的会话」而不是原进程延续。
- `pane_history = true` 可以把最近屏幕内容回放到重启后的 pane（默认关，因为可能泄露密钥）。
- 状态文件：`session.json`、`session-backups/`（保留 3 份）、`session-history.json`。

### 1.6 客户端 / 服务端架构

- **server 拥有 pane 和进程状态；client 只是接上去的 TUI**。多个 client 可同时接同一个 server，各看各的 workspace。
- 0.9 把「外层 UI」搬到了 client 渲染，server 只负责会话和终端画面，所以一个 client 可以同时接多台机器的 server（见博客《Connecting the machines》）。
- 通信走本地 Unix socket（Windows 是命名管道），协议是按行分隔的 JSON：`{"id":"req_1","method":"ping","params":{}}` → `{"id":"req_1","result":{"type":"pong"}}`。
- socket 路径：默认 `~/.config/herdr/herdr.sock`；命名 session 在 `~/.config/herdr/sessions/<name>/herdr.sock`。选择顺序：`--session` > `HERDR_SOCKET_PATH` > `HERDR_SESSION` > 默认。
- 多客户端同时 attach 时，显示区域会**收缩到最小的那个客户端**（和 tmux 一样）。手机接上来会把 94 行压到 15 行，用完记得让手机 detach。

### 1.7 安装与升级

```bash
# Linux / macOS
curl -fsSL https://herdr.dev/install.sh | sh
# 或
brew install herdr
mise use -g herdr
nix profile install github:herdrdev/herdr/v0.x.y
x install herdr            # x-cmd

# Windows
powershell -ExecutionPolicy Bypass -c "irm https://herdr.dev/install.ps1 | iex"

# 手动
chmod +x herdr-linux-x86_64 && mv herdr-linux-x86_64 ~/.local/bin/herdr

# 升级
herdr update                # 脚本安装时
herdr update --handoff      # 实验：升级时把运行中的 pane 移交给新 server，不杀进程
herdr channel set stable|preview
```

### 1.8 日常快捷键（prefix 默认 `ctrl+b`，与 tmux 一致）

| 动作 | 键 |
|---|---|
| 右分割 / 下分割 | `prefix+v` / `prefix+-` |
| 新 tab / 上下个 tab | `prefix+c` / `prefix+n` `prefix+p` |
| 在 pane 间移动 | `prefix+h/j/k/l` |
| 新 workspace / 切 workspace | `prefix+shift+n` / `prefix+w` 然后上下箭头 |
| Detach | `prefix+q` |
| 帮助 | `prefix+?` |

鼠标原生支持：点击聚焦、拖边框调大小、右键菜单、拖选即复制、ctrl+点击打开链接。

### 1.9 常用 CLI 速查

```bash
herdr                                  # 启动或接入默认 session
herdr --session lab                    # 命名 session
herdr status / herdr status server     # 运行摘要
herdr server stop                      # 停掉后台 server（所有 agent 一起停）
herdr server reload-config
herdr session list / herdr session attach <name>

herdr workspace create --cwd ~/project --label api --no-focus
herdr workspace list
herdr worktree create --cwd ~/repo --branch feat-x --no-focus   # 每个任务一个 git worktree

herdr pane split w1:p1 --direction right --no-focus
herdr pane run w1:p2 "npm test"
herdr pane read w1:p2 --source recent --lines 50
herdr pane send-text w1:p1 "请修复登录超时"      # 不回车
herdr pane send-keys w1:p1 enter                 # 回车
herdr pane wait-output w1:p3 --regex "passed|failed" --timeout 120000

herdr agent list [--json]
herdr agent start reviewer --kind codex --pane w1:p2 -- -m gpt-5.4
herdr agent prompt reviewer "Review the current diff" --wait --timeout 120000
herdr agent wait reviewer --until blocked --timeout 120000
herdr agent read reviewer --source recent-unwrapped --lines 120
herdr agent send-keys reviewer esc
herdr agent attach reviewer            # 直接钻进某个 agent 的终端
herdr agent explain w1:p1 --json       # 解释为什么判成这个状态
herdr agent rename w1:p1 reviewer

herdr integration install claude       # 写 ~/.claude/hooks/herdr-agent-state.sh + settings.json
herdr integration install codex
herdr integration status

herdr api snapshot / herdr api schema --json
herdr plugin install owner/repo/subdir / herdr plugin link /path / herdr plugin list
herdr --skill                          # 打印内置的 Claude Code skill（含 HERDR_ENV=1 防护说明）
```

`wait` 类命令没有默认超时，必须显式给 `--timeout`（3000–300000 ms）。创建类命令返回 JSON，ID 要从 `.result` 里取，不要自己猜。

### 1.10 Agent 驱动 herdr（agent-native 自动化）

pane 里的进程会自动拿到 `HERDR_ENV`、`HERDR_SOCKET_PATH`、`HERDR_WORKSPACE_ID`、`HERDR_TAB_ID`、`HERDR_PANE_ID`、`HERDR_BIN_PATH`。agent 可以：开 pane、拉起别的 agent、给它发 prompt、`agent wait --until blocked/done` 真正等到它需要人时再介入、读它的输出、订阅事件（`events.subscribe`）而不是轮询。

典型「牧羊人」模式（zheyu.me 文章）：开一个编排 agent + 后端 agent + 前端 agent + 评审 agent + 部署 agent，人只跟编排 agent 说话。给 Claude Code 装技能：

```bash
npx skills add herdrdev/herdr --skill herdr -g
```

### 1.11 与 tmux / Zellij / 桌面管理器的关系

| 能力 | tmux | Zellij | herdr |
|---|---|---|---|
| 真实终端 + 持久化 | 有 | 有 | 有 |
| Agent 语义状态 | 无 | 无 | blocked/working/done/idle |
| Agent 用的 API | 脚本 | 无 | read/send/wait/split/attach |
| 一个客户端接多台机器 | 无 | 无 | 0.9 有 |
| 鼠标 | 弱 | 原生 | 原生 |
| 成熟度 | 2007 | 2021 | 约半年 |

结论：只跑 1 个 agent 时不需要它；3–4 个并发 agent 开始有价值；4 个以上「注意力队列」价值明显。纯看日志的服务器继续用 tmux 就好。**不要在 herdr pane 里再套 tmux**：herdr 只看前台进程，会把 tmux 当成 agent 而识别不到里面的 Claude/Codex。

### 1.12 已知限制

- 没有进程隔离，多个 agent 共享同一工作区会互相踩文件，请用 `herdr worktree` 一任务一 worktree。
- 没有 agent 间共享记忆，没有任务图（「A 完成后自动开 B」要自己用 `agent wait` 编排）。
- **0.9 的多机器只在 TUI 层统一**；CLI / socket API 仍只作用于单个 server（可用 `--machine <label>` 指定目标机器，但跨机器的 agent 协作官方还在做）。
- 插件是本地可执行文件，1200+ 插件零审核，安装前自己看代码。
- 协议仍在演进（socket protocol rev 20），客户端与服务端做版本协商，不必严格同版本，但建议同步升级。
- Windows 做客户端没问题；做被远程接入的主机是最不成熟的路径（0.8 时明确不支持，0.9 文档写支持 x86_64）。
- Herdr Cloud（免 SSH 打通机器）尚未发布，只有 waitlist。

---

## 二、场景一：Linux 服务器上部署 herdr 管理多个 AI 编程 agent

### 2.1 适用形态

- 一台 VPS / 公司内网开发机 / 家里的 Linux 盒子长期开机；
- 代码、工具链、凭据都放在这台机器上；
- 你从笔记本、办公室电脑、手机随时接进来看 agent，看完就走。

路径：`你 → (Tailscale 或公网) → SSH → Linux 主机 → herdr-server → Claude Code / Codex / ...`

### 2.2 服务器侧一次性准备

```bash
# 1. 安装 herdr（普通用户，不要 root）
curl -fsSL https://herdr.dev/install.sh | sh
herdr --version

# 2. 安装并登录你要用的 agent CLI（示例）
npm i -g @anthropic-ai/claude-code && claude        # 首次登录
npm i -g @openai/codex && codex

# 3. 装状态/会话集成（写 hook，server 重启后能 --resume 原会话）
herdr integration install claude
herdr integration install codex
herdr integration status

# 4. 第一次启动（会自动拉起后台 server 并建默认 workspace）
cd ~/projects/app && herdr
```

**关于后台 server 的启动方式（重点）**：
- 不需要也不建议自己写 systemd 单元去跑 `herdr server`。herdr 自己 fork 的 server 会调用 `setsid()` 成为会话组长；被 systemd/launchd 直接托管的进程不满足这个「detached_server_daemon」检查，`herdr machine add` 会拒绝把它存成远程机器（firstmate 项目 issue #4642 有完整分析，解决办法是包一层 fork + setsid 再 exec `herdr server`）。
- 正确做法：让 `herdr`（attach）或客户端的 `herdr machine add` 自动拉起 server。
- systemd 系统上执行 `loginctl enable-linger $USER`，保证你退出 SSH 后用户进程不会被清理。
- 主机重启后进程本来就不会活，第一次 `herdr` 会按 `session.json` 重建布局并用原生会话恢复 agent 对话；真的要开机自启，用 `@reboot setsid -f herdr server` 这类方式而不是 systemd 直托管。

推荐的 `~/.config/herdr/config.toml` 补充：

```toml
[server]
headless_cols = 160     # 没有客户端接入时 pane 的虚拟尺寸，给纯 CLI 驱动的 agent 用
headless_rows = 50

[session]
resume_agents_on_restore = true   # 默认即 true
# pane_history = true             # 想在重启后看到最近屏幕内容再开，注意密钥泄露
```

### 2.3 常见用法 A：交互式「一进多出」

```bash
ssh box
herdr                      # 接入默认 session
# prefix+shift+n 新建 workspace，一个 repo 一个
# prefix+v / prefix+- 分 pane，每个 pane 起一个 agent
claude                     # pane 1
codex                      # pane 2
opencode                   # pane 3
# prefix+q detach，或直接关掉 SSH
```

侧边栏按 workspace 汇总状态；红色（blocked）就点进去回答，蓝色（done）就点进去验收。想只盯一个 agent 时 `herdr agent attach <name>`。

### 2.4 常见用法 B：按项目 / 团队拆命名 session

```bash
herdr --session teamA           # 独立 socket、独立状态
HERDR_SESSION=teamA herdr agent list
```

适合一台服务器被多个人或多组任务共用；不同 session 之间完全隔离。

### 2.5 常见用法 C：一任务一 worktree，避免 agent 互踩

```bash
herdr worktree create --cwd ~/src/app --branch fix/session-expiry --no-focus
herdr worktree create --cwd ~/src/app --branch review/rate-limit --no-focus
herdr workspace list
# w2 fix/session-expiry   agent: idle   panes: 1
# w3 review/rate-limit    agent: idle   panes: 1
```

可以配一个 `worktree.created` 事件插件，让新建 worktree 时自动在首个 pane 拉起 Claude Code（DevelopersIO 文章的 `claude-on-worktree` 插件）。注意 `herdr plugin link` 是 server 端状态，每台机器都要执行一次。

### 2.6 常见用法 D：非交互式 / 脚本化驱动（CI、cron、另一个 agent）

herdr CLI 走 socket，不需要 TTY，所以可以直接 `ssh box '...'`：

```bash
ssh box 'herdr agent list --json'
ssh box 'herdr pane send-text w3:p1 "请给 remote-demo-proof.md 写一行内容"'
ssh box 'herdr pane send-keys w3:p1 enter'
ssh box 'herdr agent wait w3:p1 --until blocked --timeout 120000'
ssh box 'herdr pane read w3:p1 --source visible'
ssh box 'herdr pane send-keys w3:p1 enter'      # 批准
```

一个完整的「评审流水线」脚本（来自官方 agent-automation 文档）：

```bash
created=$(herdr workspace create --cwd ~/project --label api --no-focus)
pane_id=$(printf '%s\n' "$created" | jq -r '.result.root_pane.pane_id')
split=$(herdr pane split "$pane_id" --direction right --no-focus)
review_pane=$(printf '%s\n' "$split" | jq -r '.result.pane.pane_id')

herdr agent start reviewer --kind codex --pane "$review_pane" -- -m gpt-5.4
herdr agent prompt reviewer "Review the current diff" --wait --timeout 120000
herdr agent read reviewer --source recent-unwrapped --lines 120

herdr pane run w1:p3 "just test --watch"
herdr pane wait-output w1:p3 --regex "passed|failed" --timeout 120000
```

### 2.7 常见用法 E：编排 agent（agent 管 agent）

在服务器上开一个「牧羊人」Claude Code，装上 herdr skill（`herdr --skill` 或 `npx skills add herdrdev/herdr --skill herdr -g`），让它用 `workspace create / agent start / agent prompt / agent wait` 去调度后端、前端、测试、评审等 agent；人只跟牧羊人说话。skill 内置了「先确认 `HERDR_ENV=1` 再操作」的防护。

### 2.8 从外面接进来

| 方式 | 命令 / 说明 |
|---|---|
| 直接 SSH 再 attach | `ssh box` 然后 `herdr` |
| 本地 herdr 一键远程 attach | `herdr --remote box` 或 `herdr --remote ssh://you@server:2222`（远程 Linux 上会自动装/起 server） |
| 0.9 多机器统一视图 | `herdr machine add box --label "VPS"`，之后本地 `herdr` 里同时看到本地和 VPS 的 workspace |
| 手机 | Tailscale + 任意 SSH 客户端（Termius / Redock / Moshi）→ `herdr`；网络差用 mosh 包一层 |
| 转发 socket 给本地工具 | `ssh -fnNT -L /tmp/herdr-remote.sock:/home/you/.config/herdr/herdr.sock box` 然后 `HERDR_SOCKET_PATH=/tmp/herdr-remote.sock herdr agent list` |

### 2.9 运维与安全要点

- 升级用 `herdr update --handoff`，否则升级等于重启 server，agent 进程会死（对话可恢复，进程不可）。
- `herdr server stop` 会停掉该 session 的所有 agent，慎用。
- 日志在 `~/.config/herdr/`（命名 session 在 `sessions/<name>/`），排查用 `HERDR_LOG=herdr=debug`。
- 不要把 22 端口暴露公网；用 Tailscale / WireGuard，或至少改端口 + 仅密钥登录。
- agent 以你的用户权限运行，能碰到你能碰到的所有凭据。建议单独建一个用户跑 agent，凭据按最小范围给。
- 插件 = 本地可执行文件，安装前审查 `herdr-plugin.toml` 和脚本。
- 多个客户端同时 attach 会把屏幕压缩到最小客户端，手机看完记得 detach。

---

## 三、场景二：两台 Mac 之间部署 herdr

### 3.1 角色划分

| 角色 | 机器 | 跑什么 |
|---|---|---|
| **Host（宿主）** | 放在家里 / 办公室、常开的 Mac（Mac mini / Mac Studio / 旧 MacBook） | herdr-server、所有 agent、代码仓库、工具链、登录态 |
| **Client（日常）** | 随身 MacBook | herdr TUI（`machine add` 或 `--remote`）、编辑器、可选 herdrm / Raycast 扩展 |

原则（官方博客原话大意）：**把 herdr 放在允许一直开着的那台机器上**。宿主是「单一事实源」，随身 Mac 只是看进去的窗口。

### 3.2 宿主 Mac 一次性准备（两种网络场景通用）

1. **开 SSH**：系统设置 → 通用 → 共享 → 远程登录，只允许你的账号。
2. **别睡**：系统设置 → 电池/节能 → 显示器关闭时防止自动进入睡眠；勾选「唤醒以供网络访问」。或者：

   ```bash
   sudo pmset -a sleep 0 disksleep 0 womp 1
   sudo pmset -a disablesleep 1      # 合盖也不睡（笔记本当宿主时）
   ```
3. **重启后能自己回来**：开启自动登录（有 FileVault 时重启后会停在解锁界面，要么关 FileVault，要么接受「重启后需要人到场输一次密码」）。
4. **让 herdr-server 从 GUI 登录会话里启动**：在宿主本机的 Terminal（或 VNC/屏幕共享进去）跑一次 `herdr`，这样 server 及其 pane 里的进程都继承 GUI 会话权限，能读写 login Keychain。纯 SSH 会话拉起来的进程写不了 Keychain（`User interaction is not allowed`），会影响 1Password / gh / 某些 CLI 登录。DevelopersIO 文章的经验：需要 Keychain 的命令，通过 `herdr pane run` 丢进 herdr 的 pane 里执行即可。
5. **不要用 launchd 直接托管 `herdr server`**：launchd 起的进程不是会话组长，`herdr machine add` 会报「remote server is not ready for saved machines」。让 herdr 自己 spawn。
6. **装 herdr + agent + 集成**：

   ```bash
   brew install herdr
   herdr integration install claude
   herdr integration install codex
   herdr integration status
   ```
7. 有 herdr 插件的话，在宿主上再 `herdr plugin link ...` 一次（插件注册是 server 端状态，不随 dotfiles 走）。

### 3.3 网络场景 A：两台 Mac 在同一局域网

最简单，走 mDNS：

```bash
# 日常 Mac 上
ssh you@mac-mini.local            # 先验证能通
```

固定下来写进 `~/.ssh/config`（显式写 User，避免本地和远程用户名不同）：

```
Host mac-mini
    HostName mac-mini.local        # 或路由器上做 DHCP 保留后的固定 IP
    User you
```

```bash
ssh-copy-id -f -i ~/.ssh/id_ed25519.pub mac-mini   # 用 1Password 之类的 agent 管钥匙时必须 -f -i
ssh mac-mini 'sw_vers -productVersion'             # 免密通了
```

`.local` 只在同一网段可解析，出了局域网就没用了，这就是场景 B 要解决的问题。

### 3.4 网络场景 B：远程（家里 / 办公室的 Mac 在 NAT 后面）

**推荐 Tailscale**（两篇实战文章都是这个方案）：不用开路由器端口、不暴露 SSH 到公网、WireGuard 加密、UDP 可穿透（mosh 也能用）。

```bash
# 两台 Mac 都装并登录同一账号
brew install tailscale        # 或 App Store 版
sudo tailscale up
tailscale ip -4               # 记下宿主的 100.x.y.z
tailscale ping -c 3 mac-mini  # "via 192.168.x.y" 表示直连，"via DERP" 表示走中继
```

`~/.ssh/config`：

```
Host mac-mini
    HostName mac-mini            # 开了 MagicDNS 直接用机器名；否则写 100.x.y.z
    User you
```

可选：Tailscale 1.98+ 的 Tailscale SSH 可以完全省掉密钥分发，herdr 和 herdrm 都能识别。

其他可选路径：ZeroTier / Cloudflare Tunnel（同类效果）；路由器端口转发 + 动态 DNS（不推荐，除非改端口 + 仅密钥 + fail2ban）。

**一个别名同时覆盖两种场景的技巧**：两台 Mac 都常驻 Tailscale 时，同一局域网内 Tailscale 会自动走局域网直连（`tailscale ping` 会显示 `via 192.168.x.y`），所以 `Host mac-mini` 只指向 Tailscale 名字即可，在家和在外无需切换配置。不想在局域网也走 Tailscale，可以用两个别名 `mac-mini-lan` / `mac-mini-ts`，或 ssh 的 `Match` 按当前网络选择。

### 3.5 日常 Mac 上接入宿主的四种方式

**方式 1（推荐，0.9 多机器）：把宿主加进本地 herdr**

```bash
herdr machine add mac-mini --label "Home Mac mini"
# 需要接宿主的命名 session 时：
herdr machine add mac-mini --label "Home Mac mini" --remote-session agents

herdr machine list [--json]
herdr machine disable|enable|rename|remove <profile-id>
```

`machine add` 会检查宿主上的 herdr 二进制和 server，没有就走审批式安装并把后台 server 拉起来，然后保存 profile（只存 ID、标签、SSH 目标、session 名、启用状态，不存密钥密码）。之后本地 `herdr` 里，侧边栏同时列出本机和 Mac mini 的 workspace / agent；点选或 `prefix+w` 上下键切换，输入和窗口尺寸只发给当前选中的机器，其他机器在后台继续刷新状态。断网 / 合盖 / SSH 掉线后自动重连，退避最长 2 分钟。

- 侧边栏显示「Attention」= 需要人机交互（首次 host key、密钥口令），跑一次 `herdr --remote mac-mini` 走完提示再重启客户端。
- 私钥有口令的先 `ssh-add`。
- 多机器时 CLI 用 `herdr --machine "Home Mac mini" agent list` 指定目标。

**方式 2：单独开一个远程窗口 `herdr --remote mac-mini`**

适合只想全屏看宿主、不想混在一起。坑：如果你的终端启动时自动 attach 本地 herdr，`--remote` 会因「嵌套 herdr」被拒；即便在 `[experimental] allow_nested = true` 后放行，prefix 键也会先被外层本地 herdr 吃掉。解决：开一个不自动 attach herdr 的终端窗口专门跑它，例如：

```sh
#!/bin/sh
exec open -na Alacritty --args -e /bin/zsh -lc "herdr --remote mac-mini"
```

**方式 3：纯 CLI / 脚本，不进 TUI**

```bash
ssh mac-mini 'herdr workspace list'
ssh mac-mini 'herdr worktree create --cwd ~/src/app --branch remote-demo --no-focus'
ssh mac-mini 'herdr pane send-text w3:p1 "请实现 X"'
ssh mac-mini 'herdr pane send-keys w3:p1 enter'
ssh mac-mini 'herdr agent wait w3:p1 --until blocked --timeout 120000'
ssh mac-mini 'herdr pane read w3:p1 --source visible'
```

或把宿主的 socket 转发到本地，让本地的 herdr / Raycast 扩展 / 自写脚本「以为」在操作本机：

```bash
ssh -fnNT -L /tmp/herdr-remote.sock:/Users/you/.config/herdr/herdr.sock mac-mini
HERDR_SOCKET_PATH=/tmp/herdr-remote.sock herdr workspace list
```

DevelopersIO 文章给了一个包装脚本：socket 不存在或不响应时自动重建隧道，再 `exec` 真正的 herdr，并把 Raycast 的「Herdr Binary」指向它，于是菜单栏状态、workspace 切换、worktree 创建全都作用到 Mac mini。

**方式 4：原生 macOS 客户端 herdrm**

第三方 SwiftUI 应用（`brew install owo-network/brew/herdrm`，非商业免费），本地走 Unix socket，远程走 SSH 隧道（密钥 / Tailscale SSH / Keychain 密码），把多台机器的 agent 和终端放进一个侧边栏，带文件浏览、上传下载、⌘K 搜索。要求 macOS 14+，两边都装 herdr。

### 3.6 推荐的日常工作流

1. 宿主上一 repo 一 workspace、一任务一 worktree；每个 pane 一个 agent。
2. 日常 Mac 打开 `herdr`，本地 workspace 和 Mac mini 的 workspace 并列；红色 blocked 就切过去批一下，蓝色 done 就 attach 进去验收。
3. 合盖出门，agent 继续跑；到办公室再开 `herdr`，自动重连，现场还在。
4. 手机上（Moshi / Redock / Termius）通过 Tailscale SSH 进宿主跑 `herdr` 批审批；配合 moshi-hook 可以把 Claude Code 的审批推到 iPhone 上直接点。
5. 宿主升级 herdr 用 `herdr update --handoff`；宿主重启后第一次 `herdr` 会重建 workspace 并用 `claude --resume` 拉回对话。

### 3.7 两台 Mac 场景的坑清单

| 坑 | 表现 | 处理 |
|---|---|---|
| 宿主睡了 | Attention / 重连失败 | pmset 关睡眠，勾选网络唤醒；笔记本当宿主要 `disablesleep 1` |
| FileVault + 重启 | 重启后 SSH 不通 | 自动登录 + 关 FileVault，或接受需人到场 |
| Keychain | SSH 里跑的命令写不了钥匙串 | 让 server 从 GUI 会话启动；需要钥匙串的命令用 `herdr pane run` 丢进 pane |
| launchd 托管 server | `machine add` 拒绝保存 | 不要用 launchd 直托管；让 herdr 自己 spawn |
| `.local` 域名 | 出了局域网解析不到 | 远程用 Tailscale IP / MagicDNS |
| 嵌套 herdr | `--remote` 报 nested；prefix 被本地吃掉 | 用不自动 attach 的窗口跑 `--remote`，或改用 `machine add` |
| 屏幕变小 | 手机 attach 后宿主 pane 只剩 15 行 | 手机看完 detach |
| 插件不生效 | `herdr plugin list` 为空 | 每台机器各 `plugin link` 一次 |
| herdr pane 里套 tmux | agent 状态一直 unknown | 不要套 tmux，让 Claude/Codex 直接是 pane 前台进程 |
| 两边版本差太多 | 协议协商失败 | 两边都 `herdr update`，宿主用 `--handoff` |
| 多 agent 同一 checkout | 互相覆盖文件 | `herdr worktree create` 一任务一分支 |

---

## 四、一页速记

- **本质**：后台 server 拥有 agent 的终端；客户端只是窗口。Detach 不死，SSH 掉线不死，主机重启只能恢复布局和对话，不能恢复进程。
- **最大价值**：blocked / working / done / idle 状态汇总，4 个以上并发 agent 时最明显。
- **Linux 服务器**：`curl | sh` 装好，`herdr integration install claude/codex`，第一次 `herdr` 自动起 server；从外面 `ssh box && herdr`、`herdr --remote box`、或 0.9 的 `herdr machine add box`；脚本化全靠 `ssh box 'herdr ...'` 或 socket 转发；别用 systemd 直托管 server。
- **两台 Mac**：常开的那台当宿主，开远程登录、关睡眠、从 GUI 会话起一次 herdr；局域网用 `.local`，远程用 Tailscale，一个 ssh 别名指向 Tailscale 名字两边通吃；日常 Mac 上 `herdr machine add mac-mini` 把宿主并进本地 TUI，或者 `--remote` 单开窗口，或者 socket 转发给 CLI / Raycast，或者用 herdrm 原生应用。
