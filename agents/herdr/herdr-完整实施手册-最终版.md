# herdr 多 AI 编程 Agent 远程运行：完整实施手册（最终版）

> 本手册合并并取代同目录下此前的五份文档：herdr 总结与部署指南、实施指南一（Linux）、实施指南二（两台 Mac）、终端与客户端选型、闭环补遗。补遗中发现的 13 个缺口已直接融入对应步骤。
> 版本基线：herdr v0.9.x、Tailscale 1.9x、Claude Code 2.1.x、Codex CLI、Ghostty 1.2+、macOS 26、Ubuntu 24.04。整理日期 2026-09-24。

## 阅读指引

| 你要做的事 | 读哪里 |
|---|---|
| 先搞懂 herdr 是什么 | 第一部分 |
| 在 Linux 服务器上跑多个 agent | 第二部分总架构 → 第三部分通用组件 → 第四部分 |
| 用日常 Mac 驱动家里/办公室另一台 Mac | 第二部分总架构 → 第三部分通用组件 → 第五部分 |
| 搭好后每天怎么用 | 第六部分 |
| 升级、重启、出问题 | 第七、八部分 |
| 查命令、查配置键 | 附录 A、B |

**命名约定**（全文统一，照抄时替换成你的值）：

| 名称 | 含义 |
|---|---|
| `box` | Linux 服务器主机名（场景一） |
| `dev` | Linux 上运行 agent 的专用用户 |
| `mac-mini` | 常开的宿主 Mac（场景二），宿主用户 `you` |
| `laptop` | 随身日常使用的电脑 |
| `~/src/<repo>` | 仓库主 checkout |
| `~/wt/<repo>/<branch>` 或 `~/.herdr/worktrees/` | 每个任务的 git worktree（二选一坚持用） |
| `box-agents-8f3k2` | ntfy 通知 topic，请换成你自己的难猜字符串 |

---

# 第一部分　herdr 是什么

## 1.1 一句话

herdr 是「AI 编程 agent 的运行时」。一个常驻后台的 herdr-server 真正拥有每个 agent 的终端（PTY）和子进程，你用的 TUI 客户端只是接上去的窗口。关掉客户端、SSH 断线、笔记本合盖，agent 照常运行；从任何机器再执行 `herdr`，现场原样回来。

## 1.2 基本事实

| 项目 | 内容 |
|---|---|
| 实现 | Rust 单二进制约 20 MB，无 Node / Electron 依赖；pane 渲染使用内置的 Ghostty VT 引擎 |
| 许可证 | Apache 2.0 |
| 版本 | v0.9.1（2026-09）；0.9 起一个客户端可同时接入多台机器 |
| 平台 | Linux / macOS（x86_64、aarch64），Windows x86_64 |
| 资源 | 空闲两个 pane：server 约 17 MB，client 约 10 MB |
| 社区 | 约 4 万 star、约 99 万安装、1200+ 插件；已进 Y Combinator 并完成种子轮 |
| 定位 | 不替代 tmux，而是 tmux 之上的一层：tmux 管终端，herdr 管终端里跑的 agent |

## 1.3 核心概念

| 概念 | 含义 |
|---|---|
| Session | 一个 herdr-server 的持久命名空间；默认 session 自动运行，命名 session（`--session lab`）拥有完全独立的 pane、socket 和状态 |
| Workspace | 项目级容器，建议一个 repo 或一个任务一个 |
| Tab | workspace 内的一套布局 |
| Pane | 一个真实终端进程，可左右或上下分割 |
| Agent | pane 里被识别出的 agent 进程，名字须匹配 `[a-z][a-z0-9_-]{0,31}` |
| Machine | 0.9 新增，通过 SSH 接入的远程 herdr-server |

ID 写作 `w1:p2`（workspace 1 的 pane 2），按 server 作用域，两台机器可以各有自己的 `w1:p1`。

## 1.4 Agent 状态

| 状态 | 含义 | 侧边栏颜色 |
|---|---|---|
| `working` | 正在干活 | 黄 |
| `blocked` | 等你批准或回答 | 红 |
| `done` | 完成但你还没看 | 蓝 |
| `idle` | 空闲且已看过 | 绿 |
| `unknown` | 认出是 agent 但无法判断 | 灰 |

状态会向上汇总：pane、tab、workspace、machine，所以侧边栏一眼就能看出谁卡住了。

状态判定有三种来源：
1. **生命周期 hook 或插件**：Pi、OMP、Kimi、OpenCode、Kilo、MastraCode 等由 agent 自己上报。
2. **前台进程加屏幕 manifest**：识别 pane 的前台进程，再对终端底部实时画面做模式匹配，Claude Code、Codex 走这条。`blocked` 判定刻意保守，只在匹配到已知审批提示时才标红。
3. **会话身份集成**：Claude Code、Codex、Copilot、Cursor、Grok、Hermes 等把自己的会话 ID 报给 herdr，用于 server 重启后原生恢复对话。

本地可用 `~/.config/herdr/agent-detection/<agent>.toml` 覆盖检测规则。

开箱识别的 agent：`pi, claude, codex, gemini, cursor, devin, agy, cline, omp, mastracode, opencode, copilot, kimi, kiro, droid, amp, grok, hermes, kilo, qodercli, qwen, letta, maki, muse`。未识别的 CLI 照样能跑，只是显示 `unknown`。

## 1.5 持久化边界（务必分清）

| 场景 | 进程 | 布局 | 屏幕 | Agent 对话 |
|---|---|---|---|---|
| Detach / 断线重连 | 存活 | 保留 | 实时 | 保留 |
| Server 或主机重启 | **不存活** | 从 `session.json` 恢复 | 仅历史（默认关） | 靠原生会话恢复 |
| `herdr update`（无 handoff） | 视情况 | 恢复 | 仅历史 | 原生会话恢复 |
| `herdr update --handoff`（实验） | 尽力保活 | 保留 | 实时 | 保留 |

- Detach 是最强的持久化：`ctrl+b q`、关终端、SSH 掉线都不影响进程。
- 重启只能重建结构。`resume_agents_on_restore = true`（默认开）会用集成上报的会话 ID 重新拉起对话，这是「重建的会话」，不是原进程延续。
- `pane_history = true` 可在重启后回放最近屏幕内容，默认关，因为可能把密钥写进 `session-history.json`。
- 状态文件：`session.json`、`session-backups/`（保留 3 份）、`session-history.json`。

## 1.6 客户端与服务端

- server 拥有 pane 和进程；client 只是 TUI。多个 client 可同时接同一个 server。
- 0.9 把外层 UI 挪到客户端渲染，所以一个客户端能同时接多台机器的 server。
- 通信走本地 Unix socket（Windows 为命名管道），协议是按行分隔的 JSON。
- socket 位置：默认 `~/.config/herdr/herdr.sock`；命名 session 在 `~/.config/herdr/sessions/<name>/herdr.sock`。选择优先级：`--session`、`HERDR_SOCKET_PATH`、`HERDR_SESSION`、默认。
- 多客户端同时接入同一 tab 时，显示区域会缩到最小的那个客户端，手机接入后用完要 detach。

## 1.7 与其他工具的关系

| 能力 | tmux | Zellij | herdr |
|---|---|---|---|
| 真实终端与持久化 | 有 | 有 | 有 |
| Agent 语义状态 | 无 | 无 | 有 |
| 给 agent 用的 API | 脚本 | 无 | read / send / wait / split / attach |
| 一个客户端接多台机器 | 无 | 无 | 0.9 有 |
| 鼠标 | 弱 | 原生 | 原生 |
| 成熟度 | 2007 起 | 2021 起 | 约半年 |

桌面「agent 管理器」（cmux、Warp、Conductor 一类）替你管理窗口，但关掉应用会话就停，也无法从任意 tty 接入。只用 Claude Code 且只在一台机器上时，它们可能更省事；跨机器、多种 agent、要从手机接入时，herdr 这条路线才成立。

**值不值得用**：只跑 1 个 agent 不需要；3 到 4 个开始有价值；4 个以上价值明显。

## 1.8 已知限制

- 没有进程隔离，多个 agent 共用同一 checkout 会互相覆盖文件，必须一任务一 worktree。
- 没有 agent 间共享记忆，也没有任务图，「A 完成后开 B」需要自己用 `agent wait` 编排。
- 0.9 的多机器只在 TUI 层统一，CLI 和 socket API 仍作用于单个 server（可用 `--machine` 指定目标）。
- 插件是本地可执行文件，没有审核，安装前自己读代码。
- 协议仍在演进，客户端与服务端会协商版本，但建议两边同步升级。
- 不要在 herdr pane 里再套 tmux，herdr 会把 tmux 当成前台进程而识别不到里面的 agent。
- Herdr Cloud（免 SSH 打通机器）尚未发布。

---

# 第二部分　总体架构与工具选型

## 2.1 场景一：Linux 服务器

```
 手机（Moshi / Blink / Redock / Termius）         laptop（Ghostty + herdr 客户端）
        │  Tailscale + mosh                               │  Tailscale + ssh
        └──────────────┐                 ┌────────────────┘
                       ▼                 ▼
          ┌──────────── Linux 主机 box（常开）────────────┐
          │ sshd（仅密钥） · tailscaled · mosh-server       │
          │ systemd linger：退出 SSH 后用户进程不被清理      │
          │ herdr-server（dev 用户）                        │
          │   ├─ w1 app          [claude]  [codex 评审]     │
          │   ├─ w2 wt/fix-x     [claude]  [测试]           │
          │   └─ w3 wt/docs      [opencode]                 │
          │ 通知出口：ntfy / moshi-hook                      │
          └────────────────────────────────────────────────┘
```

## 2.2 场景二：两台 Mac

```
        laptop（日常 Mac）                              iPhone（可选）
  ┌───────────────────────────┐                 ┌──────────────────────┐
  │ Ghostty + herdr 客户端     │                 │ Moshi / Redock /     │
  │ machine add / --remote     │                 │ Termius / Blink      │
  │ herdrm.app / Raycast 扩展  │                 │ + moshi-hook 审批卡片 │
  └───────┬─────────┬─────────┘                 └──────────┬───────────┘
   同一局域网         远程                                    │
   .local / mDNS     Tailscale                     Tailscale + mosh
          └────────────┬──────────────────────────────────────┘
                       ▼
        ┌────────── mac-mini（宿主，常开，不睡）───────────┐
        │ 远程登录 · 屏幕共享 · tailscaled · mosh-server     │
        │ Terminal.app 登录项 → herdr-server（GUI 会话）     │
        │   ├─ w1 app      [claude]  [codex]                │
        │   ├─ w2 wt/fix-x [claude]                          │
        │   └─ w3 wt/docs  [opencode]                       │
        │ 集成：herdr integration · moshi-hook               │
        └───────────────────────────────────────────────────┘
```

## 2.3 工具选型总表

| 层 | 工具 | 场景一 | 场景二 | 作用 | 备选 |
|---|---|---|---|---|---|
| 运行时 | herdr | 必选 | 必选（两台都装） | 拥有 agent 终端、状态汇总 | tmux |
| Agent | Claude Code / Codex / OpenCode 等 | 必选 | 只装宿主 | 实际干活 | 任意已识别 agent |
| 终端模拟器 | Ghostty；次选 iTerm2 / WezTerm | laptop | laptop | 真彩、鼠标、Kitty 图形、通知聚焦 | 见 3.1 |
| 远程 shell | OpenSSH / macOS 远程登录 | 必选 | 必选 | 认证与通道 | Tailscale SSH |
| 网络 | Tailscale | 强烈推荐 | 远程必选 | NAT 穿透、MagicDNS、不开端口 | WireGuard / Cloudflare Tunnel |
| 局域网寻址 | Bonjour `.local` | 不需要 | 局域网场景 | 免配置找到宿主 | 固定 IP |
| 弱网 | mosh | 推荐 | 手机推荐 | 换网不断 | 直接 ssh |
| 密钥 | ssh-keygen / 1Password SSH Agent | 必选 | 必选 | 免密登录 | Tailscale SSH |
| 运行时版本 | mise | 推荐 | 推荐 | Node / Python 版本 | nvm / fnm |
| 包管理 | pnpm | 推荐 | 推荐 | 多 worktree 共享依赖 | npm |
| 版本管理 | git worktree + gh | 必选 | 必选 | 一任务一目录、开 PR | 多 clone |
| 主机常驻 | systemd linger | 必选 | 不适用 | 退出 SSH 保进程 | `KillUserProcesses=no` |
| 主机常驻 | pmset / 自动登录 / `fdesetup authrestart` | 不适用 | 必选 | 不睡、重启能回来 | caffeinate |
| 图形界面 | 屏幕共享（VNC） | 不适用 | 推荐 | 点授权弹窗、首次登录 | 到场 |
| 通知 | ntfy / moshi-hook / herdr toast | 推荐 | 推荐 | blocked、done 提醒 | Telegram bot |
| 手机 | Moshi / Redock / Blink / Termius | 可选 | 可选 | 手机上看状态、批审批 | 任意 SSH 客户端 |
| 桌面辅助 | herdrm / Raycast Herdr 扩展 | 可选 | 可选 | 多机侧边栏、菜单栏状态 | 无 |
| 安全 | ufw + fail2ban | 公网暴露时必选 | 不适用 | 端口白名单、封禁爆破 | 云安全组 |
| 隔离 | 专用用户 + worktree；可选 Docker | 推荐 | 推荐 | 限制影响面 | devcontainer |

---

# 第三部分　通用组件（两个场景共用）

第四、五部分的实施步骤会引用本部分的小节编号。

## 3.1 终端模拟器

herdr 是跑在你终端里的 TUI，外层终端决定以下能力：

| 能力 | 影响 |
|---|---|
| 24 位真彩 | herdr 主题与 agent UI 配色；Terminal.app 不支持 |
| 鼠标上报 | herdr 是 mouse-first：点 pane、拖边框、右键菜单、拖选复制 |
| Kitty 图形协议 | pane 内显示图片；herdr 只透传 Kitty 协议，不透传 Sixel 和 iTerm2 协议 |
| 键盘映射 | `ctrl+b` 前缀、`ctrl+alt` 直达键、macOS option 键 |
| 通知聚焦 | 点系统通知能否跳回 herdr；支持 Terminal、iTerm2、Ghostty、WezTerm，不支持 Warp |
| 远程 terminfo | 自定义 TERM 在远端缺 terminfo 会报 `missing or unsuitable terminal` |
| CJK 输入法 | 中文输入法可能吞掉前缀键或光标错位 |

### 3.1.1 推荐

| 角色 | 推荐 | 理由 |
|---|---|---|
| macOS 日常客户端 | **Ghostty**，次选 iTerm2 | herdr 的 VT 引擎源自 Ghostty，社区插件以它为测试基线；iTerm2 最稳但 pane 内不显示图片 |
| Windows 客户端 | **WezTerm**，次选 Windows Terminal | 内置 SSH 客户端、支持全部图片协议 |
| Linux 桌面客户端 | Ghostty / kitty / WezTerm | 都是全功能 |
| 宿主 Mac 上启动 server | **Terminal.app 登录项** | 内置、随登录起、从 GUI 会话启动 |
| 专用远程窗口 | 任何能用 `-e` 起命令的终端 | 见 3.1.3 |
| 编辑器里看一眼 | VS Code / Cursor 内置终端 | 需先解决 `ctrl+b` 冲突 |

不推荐：**Warp**（替代终端并自带 agent 平台，通知聚焦不支持，全屏 TUI 有兼容历史）；**Terminal.app 作日常客户端**（无真彩）；**foot**（只有 Sixel，图片不显示）。

各终端能力对照：

| 终端 | 平台 | 真彩 | Kitty 图形 | SSH 相关 |
|---|---|---|---|---|
| Ghostty | macOS / Linux | 是 | 是 | TERM `xterm-ghostty`，需处理 terminfo |
| iTerm2 | macOS | 是 | 否 | `xterm-256color`，零配置 |
| WezTerm | 全平台 | 是 | 是 | 内置 `wezterm ssh host`，默认 `xterm-256color` |
| kitty | macOS / Linux | 是 | 是 | `kitten ssh host` 自动拷 terminfo |
| Alacritty | 全平台 | 是 | 否 | TERM `alacritty`，需处理 terminfo |
| Terminal.app | macOS | 否 | 否 | 零配置 |
| Windows Terminal | Windows | 是 | 否 | 零配置 |
| VS Code 内置 | 全平台 | 是 | 否 | 可配合 Remote-SSH |

### 3.1.2 远程 terminfo（连接任何远端前做一次）

Ghostty 配置 `~/.config/ghostty/config`：

```
shell-integration-features = ssh-env,ssh-terminfo
macos-option-as-alt = true
```

这两个特性通过包装交互 shell 里的 `ssh` 生效，**不作用于** `herdr --remote`、`herdr machine add`、脚本、mosh、git。所以还要做下面两者之一，它们对 herdr 的远程连接都有效：

```bash
# 方法一：把 terminfo 一次性装到远端（kitty 换成 xterm-kitty，Alacritty 换成 alacritty）
infocmp -x xterm-ghostty | ssh box -- tic -x -
infocmp -x xterm-ghostty | ssh mac-mini -- tic -x -
# macOS Sonoma 及更早本机 infocmp 太旧时，用 /opt/homebrew/opt/ncurses/bin/infocmp
```

```
# 方法二：~/.ssh/config 里回落（OpenSSH 8.7+）
Host box mac-mini
    SetEnv TERM=xterm-256color
```

其他终端的 option 当 alt：iTerm2 在 Profiles → Keys 把 Left Option 设为 Esc+；kitty 设 `macos_option_as_alt yes`。

### 3.1.3 专用远程窗口

若本地终端启动时自动 attach 本地 herdr，再执行 `herdr --remote` 会报 `nested herdr is disabled`；即使打开 `allow_nested`，前缀键也会先被外层吃掉。办法是开一个不跑本地 herdr 的窗口：

```bash
osascript -e 'tell application "Terminal" to do script "herdr --remote mac-mini"'   # Terminal.app
open -na Ghostty.app --args -e herdr --remote mac-mini                                # Ghostty (macOS)
open -na Alacritty --args -e /bin/zsh -lc "herdr --remote mac-mini"                   # Alacritty
wezterm start -- herdr --remote box                                                   # WezTerm
kitty herdr --remote box                                                              # kitty
```

做成 `~/bin/herdr-remote` 绑定到快捷键。使用 0.9 的 `herdr machine add` 时不需要专用窗口。

### 3.1.4 VS Code / Cursor 内置终端

`ctrl+b` 在编辑器里是「切换侧边栏」。二选一：在 herdr 配置里把 `[keys] prefix` 改成别的组合；或在编辑器设置里让 `ctrl+b` 直通终端（`terminal.integrated.commandsToSkipShell` 中加入 `-workbench.action.toggleSidebarVisibility`）。

### 3.1.5 中文输入法

在客户端的 `~/.config/herdr/config.toml`：

```toml
[experimental]
switch_ascii_input_source_in_prefix = true    # 按前缀时临时切英文输入源
reveal_hidden_cursor_for_cjk_ime = true       # 输入法候选框位置不对时再开
```

### 3.1.6 给 agent 粘贴图片

| 场景 | 做法 |
|---|---|
| 本地 pane | 直接 Cmd+V，Claude Code 显示 `[Image #1]` |
| 远程 pane（`--remote` / `machine add`） | 剪贴板在 laptop、进程在远端，曾有静默失败的问题。**可靠做法是先把图片 `scp` 到宿主，再把宿主上的路径贴给 agent**，例如 `pngpaste /tmp/s.png && scp /tmp/s.png mac-mini:~/Exchange/shots/ && echo ~/Exchange/shots/s.png`。herdr-paste-image 插件只在它运行的那台机器上存图并打出本机路径，不负责上传；远程模式下它在哪台机器执行尚未确认，不要依赖它 |
| 在 pane 旁看图 / Mermaid | 插件 herdr-image、herdr-visuals，需 Kitty 协议终端 |

## 3.2 网络：Tailscale 与 mosh

### 3.2.1 安装

**Linux：**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --hostname box
tailscale ip -4 && tailscale status
```

**macOS 三种发行版本**（同一台机器只装一种；切换前删 `Tailscale.app`、清废纸篓、重启）：

| 版本 | 安装 | GUI | Tailscale SSH 服务端 | 无人登录时运行 | 适合 |
|---|---|---|---|---|---|
| Standalone | pkgs.tailscale.com 下载，或 `brew install --cask tailscale-app` | 有 | 有 | 否 | **laptop** |
| App Store | `mas install 1475387142` | 有 | 无 | 否 | 不推荐 |
| 开源 tailscaled | `brew install tailscale` | 无 | 有 | **是** | **宿主 Mac** |

宿主 Mac：

```bash
brew install tailscale
sudo brew services start tailscale
sudo tailscale up --hostname mac-mini
```

**手机**：应用商店安装 Tailscale，登录同一账号。

### 3.2.2 MagicDNS 与连通性检查

2022 年 10 月之后创建的 tailnet 默认开启 MagicDNS，可直接用短名 `box`、`mac-mini`。

```bash
tailscale ping -c 3 box
# "via 192.168.x.y" 或 "via 公网IP:41641" 表示直连；"via DERP(xxx)" 表示走中继，延迟更高
```

走中继时可在路由器开 UPnP 或放行 UDP 41641。

### 3.2.3 可选：Tailscale SSH

```bash
sudo tailscale set --ssh
```

- 服务端只支持 Linux 和 macOS 的开源 tailscaled 版本。
- 认证由 Tailscale 身份完成，不用分发 SSH 公钥。
- 只能用 22 端口，只能从 tailnet 内连。
- 升级或重启 tailscaled 会断开现有 Tailscale SSH 会话；herdr-server 不受影响，重新 attach 即可。

### 3.2.4 关闭宿主节点的密钥过期（无人值守必做）

节点密钥默认 180 天过期，过期后节点掉线、远程直接断。在 Tailscale 管理台 Machines → 选中 `box` 或 `mac-mini` → **Disable key expiry**。laptop 和手机保留过期即可。

### 3.2.5 备选方案

| 方案 | 何时用 | 要点 |
|---|---|---|
| 公网 22 + 加固 | 云主机、不想装 VPN | 仅密钥 + fail2ban（4.1.5） |
| WireGuard 自建 | 不依赖第三方控制面 | 自己处理密钥与 NAT |
| Cloudflare Tunnel | 已用 Cloudflare Zero Trust | 客户端 `ProxyCommand cloudflared access ssh --hostname ssh.example.com`；mosh 不可用 |

### 3.2.6 mosh

```bash
sudo apt install -y mosh      # Linux
brew install mosh             # macOS（两端都装）
mosh box -- herdr             # 弱网下 attach
```

- 需要 UDP 60000 到 61000；走 Tailscale 时天然可达，不要开到公网。
- 需要两端都是 UTF-8 locale。
- mosh 只同步可见画面、没有滚动历史，正好由 herdr 的 scrollback 补上。

## 3.3 SSH 客户端配置与密钥

laptop 上生成并分发公钥：

```bash
ssh-keygen -t ed25519 -C "laptop"            # 已有则跳过
ssh-copy-id -i ~/.ssh/id_ed25519.pub dev@box
ssh-copy-id -i ~/.ssh/id_ed25519.pub you@mac-mini
```

使用 1Password SSH Agent 时私钥文件不存在，必须加 `-f -i ~/.ssh/id_ed25519.pub`，并在 `~/.ssh/config` 顶部写：

```
Host *
    IdentityAgent "~/Library/Group Containers/2BUA8C4S2C.group.com.1password/t/agent.sock"
```

私钥带口令时，herdr 的后台 SSH 连接无法弹口令，先执行：

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519     # macOS
```

`~/.ssh/config` 模板（场景二的局域网自动切换写法见 5.3.3）：

```
Host box
    HostName box                  # MagicDNS 名；未开则写 100.x.y.z
    User dev
    SetEnv TERM=xterm-256color    # 或按 3.1.2 装 terminfo 后删掉
    ServerAliveInterval 30
    ServerAliveCountMax 6
    ControlMaster auto            # 复用连接，machine add 与脚本会频繁开 ssh
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

笔记本换网后复用连接可能僵死，清理一下：

```bash
ssh -O exit box
```

使用 Tailscale SSH 的主机可跳过公钥分发。

## 3.4 Agent 安装、登录与配额

### 3.4.1 运行时与 CLI

```bash
# mise 管理 Node / Python
curl https://mise.run | sh                     # Linux；macOS 用 brew install mise
mise use -g node@lts python@3.12
npm i -g pnpm && pnpm setup

# Claude Code
curl -fsSL https://claude.ai/install.sh | bash
# Codex
npm i -g @openai/codex
# GitHub CLI
sudo apt install -y gh      # Linux；macOS 用 brew install gh
```

### 3.4.2 无头登录

```bash
claude                        # 打印 URL，在 laptop 浏览器完成授权后把授权码贴回
codex login --device-auth     # 设备码登录，适合无浏览器主机
```

也可以只用 API key，写进 3.5 的环境文件。

**macOS 宿主特别注意**：Claude Code 的 OAuth 凭据和 `gh` 的 token 都存在 login Keychain。登录必须在 GUI 会话里完成，也就是在 Terminal.app 启动的 herdr pane 里执行，否则 SSH 里登录成功、pane 里却读不到。原因见 5.5。

### 3.4.3 配额与并发预算

- 同一账号的多个 Claude Code 实例共享订阅用量窗口，同时开 5 个很容易一两小时内触顶，之后全部一起等待。
- 建议先开 2 到 3 个，在 `claude` 里用 `/usage` 观察；长任务错峰；关键 agent 用 API key 按量计费，与订阅隔离。
- Codex 同样注意组织级速率限制。
- 通知里订阅 `quota_auto_resume_fired` 类事件（见 3.10.2），限额恢复时收到提醒。

## 3.5 环境变量与 PATH

**问题**：Debian/Ubuntu 默认 `~/.bashrc` 顶部有交互判断，`ssh box 'herdr agent list'` 这类非交互命令会在那一行返回，拿不到之后才设置的 PATH 和 API key，报 `command not found`。另外 `~/.config/environment.d/` 只对 systemd 拉起的进程生效，herdr-server 不是，不能依赖它。

**Linux 做法**：

```bash
cat > ~/.config/agents.env <<'EOF'
export PATH="$HOME/.local/bin:$HOME/.local/share/mise/shims:$PATH"
export LANG=en_US.UTF-8
export ANTHROPIC_API_KEY=...      # 用 API key 时
export OPENAI_API_KEY=...
export GH_TOKEN=...               # 用细粒度 PAT 时
EOF
chmod 600 ~/.config/agents.env

# 在 .bashrc 第一行（交互判断之前）和 .profile 中都加载
sed -i '1i [ -f "$HOME/.config/agents.env" ] && . "$HOME/.config/agents.env"' ~/.bashrc
echo '[ -f "$HOME/.config/agents.env" ] && . "$HOME/.config/agents.env"' >> ~/.profile
```

**macOS 做法**：同样内容写进 `~/.zshenv`（zsh 在所有模式都会读它），PATH 里加 `/opt/homebrew/bin` 和 `$HOME/.local/share/mise/shims`。不要写在 `~/.zshrc`。

用 mise shims 而不是 `mise activate`，因为 shims 在非交互场景也生效。

**验证**：

```bash
ssh box 'which herdr claude codex; echo ${ANTHROPIC_API_KEY:0:8}'
ssh mac-mini 'which herdr claude codex'
```

## 3.6 herdr 安装、配置与集成

### 3.6.1 安装

```bash
curl -fsSL https://herdr.dev/install.sh | sh    # 通用
brew install herdr                               # macOS
herdr --version
```

其他方式：`mise use -g herdr`、`nix profile install github:herdrdev/herdr/v0.x.y`、`x install herdr`，Windows 用 `powershell -ExecutionPolicy Bypass -c "irm https://herdr.dev/install.ps1 | iex"`。

### 3.6.2 推荐配置

```bash
mkdir -p ~/.config/herdr
herdr --default-config > ~/.config/herdr/config.toml
```

服务端（box 或 mac-mini）需要确认的键：

```toml
[server]
headless_cols = 160            # 无客户端接入时 pane 的虚拟尺寸，纯 CLI 驱动 agent 时很重要
headless_rows = 50

[session]
resume_agents_on_restore = true    # 默认即 true
# pane_history = true              # 需要重启后回放屏幕时再开，注意密钥泄露

[terminal]
new_cwd = "follow"
kitty_graphics = true
```

客户端（laptop）需要确认的键：

```toml
[ui.toast]
delivery = "system"            # macOS 走 terminal-notifier，失败退回 osascript
# 注意：terminal-notifier 是 2017 年的旧版构建，macOS 26 起可能静默丢弃通知。
# 验证：执行一次会 blocked 的操作看是否弹出；不弹就改用下一行，或改用 ntfy 原生客户端（3.10）
# delivery = "terminal"
# delivery = "terminal"        # 通过外层终端发通知，SSH 场景可用

[experimental]
switch_ascii_input_source_in_prefix = true   # 中文输入法用户
```

修改后执行 `herdr server reload-config`，部分启动项需要重启 server。

### 3.6.3 Agent 集成

```bash
herdr integration install claude     # 写 ~/.claude/hooks/herdr-agent-state.sh 并更新 ~/.claude/settings.json
herdr integration install codex      # 写 ~/.codex/hooks.json
herdr integration status
```

agent CLI 升级可能覆盖 settings，升级后再检查一次 `integration status`。

### 3.6.4 插件

`herdr plugin link` 和 `herdr plugin install` 的注册信息是 server 端状态，不随 dotfiles 走，**每台跑 server 的机器都要单独执行**。

```bash
herdr plugin install ddfonseca/herdr-paste-image
herdr plugin list
```

## 3.7 权限策略与仓库规则（远程场景能否转起来的关键）

目标：常规安全操作自动放行，危险操作仍然 blocked 并推送到手机。否则每条 `npm test` 都要人批，远程流程转不动。

### 3.7.1 Claude Code

仓库根 `.claude/settings.json`（可进版本库）：

```json
{
  "permissions": {
    "allow": [
      "Read", "Edit", "Write", "Glob", "Grep",
      "Bash(git status*)", "Bash(git diff*)", "Bash(git log*)",
      "Bash(git add*)", "Bash(git commit*)", "Bash(git push origin HEAD*)",
      "Bash(gh pr create*)", "Bash(gh pr view*)",
      "Bash(npm test*)", "Bash(npm run *)", "Bash(pnpm *)", "Bash(just *)",
      "Bash(pytest*)", "Bash(cargo test*)", "Bash(go test*)"
    ],
    "deny": [
      "Bash(git push --force*)", "Bash(git push * main)", "Bash(git push * master)",
      "Bash(rm -rf *)", "Bash(sudo *)", "Bash(curl * | sh)",
      "Read(./.env)", "Read(./secrets/**)"
    ]
  }
}
```

- 也可用 `claude --permission-mode acceptEdits` 启动：编辑自动放行，Bash 仍逐条询问。
- `--dangerously-skip-permissions` 只在容器或沙箱里用（3.12）。
- 审批提示里「本会话不再询问」慎用，用上面的 allow 列表代替。

### 3.7.2 Codex

```bash
codex --ask-for-approval on-request --sandbox workspace-write    # 编码 agent：工作区内可写，越界或联网才问
codex --ask-for-approval never --sandbox read-only               # 评审 agent：只读，永不 blocked
```

也可写进 `~/.codex/config.toml` 的 `approval_policy` 与 `sandbox_mode`。

### 3.7.3 仓库规则文件

仓库根放 `CLAUDE.md`，Codex 读 `AGENTS.md`，内容可相同：

```markdown
# 无人值守规则
- 只在当前 worktree 的分支上工作。禁止切换到 main，禁止 force push。
- 每次改动后运行 `npm test`，测试不过不要提交。
- 完成后依次执行：`git add -A`、`git commit -m "<类型>: <摘要>"`、`git push origin HEAD`、`gh pr create --fill --draft`。
- 需要新增依赖、删除文件、访问外部服务时，先停下来询问。
- 本地配置来自 .env，由脚本准备好。启动开发服务器必须使用 $PORT。
```

### 3.7.4 从手机批审批的习惯

先看清在批什么，再按键：

```bash
herdr pane read w2:p1 --source visible --lines 20
herdr pane send-keys w2:p1 enter
```

## 3.8 任务生命周期脚本

以下脚本放在跑 server 的机器的 `~/bin/`，并加入 PATH。场景二在 laptop 上用 `herdr-mm`（5.8.3）代替 `herdr` 也能远程执行其中的 herdr 部分，但涉及 git 和文件的部分需要 `ssh mac-mini '~/bin/new-task …'` 在宿主上运行。

脚本中解析 herdr JSON 的字段路径以你机器上 `herdr api schema --json` 的实际输出为准（见附录 C）。

### 3.8.1 new-task：开 worktree、备好环境、起 agent

```bash
#!/usr/bin/env bash
# 用法: new-task <repo> <branch> [agent-kind] [提示词]
set -euo pipefail
repo=$1; branch=$2; kind=${3:-claude}; prompt=${4:-}
src=~/src/$repo

created=$(herdr worktree create --cwd "$src" --branch "$branch" --no-focus)
pane=$(printf '%s\n' "$created" | jq -r '.result.root_pane.pane_id')
wt=$(git -C "$src" worktree list --porcelain \
     | awk -v b="refs/heads/$branch" '$1=="worktree"{p=$2} $1=="branch"&&$2==b{print p}')

# 1. 本地配置：git worktree 不带未跟踪文件
for f in .env .env.local .envrc; do [ -f "$src/$f" ] && cp "$src/$f" "$wt/$f"; done

# 2. 唯一端口：避免多个 worktree 的 dev server 撞车
port=$(( 3000 + $(printf '%s' "$branch" | cksum | cut -d' ' -f1) % 1000 ))
printf '\nPORT=%s\n' "$port" >> "$wt/.env"

# 3. 依赖：优先 pnpm（共享 store，省磁盘）
if   [ -f "$wt/pnpm-lock.yaml" ];    then (cd "$wt" && pnpm install --frozen-lockfile)
elif [ -f "$wt/package-lock.json" ]; then (cd "$wt" && npm ci); fi

# 4. 起 agent
name=$(printf '%s' "$branch" | tr -c 'a-z0-9_-' '-' | cut -c1-30)
herdr agent start "$name" --kind "$kind" --pane "$pane"
[ -n "$prompt" ] && herdr agent prompt "$name" "$prompt"
echo "$name -> $wt (PORT=$port)"
```

仓库里放一份 `.env.example`，并在 CLAUDE.md 写明本地配置从哪来。不认 `PORT` 变量的框架，在 CLAUDE.md 里写明启动命令要带 `--port $PORT`。

### 3.8.2 review：编码完成后自动起评审 agent

```bash
#!/usr/bin/env bash
# 用法: review <编码 agent 名>
set -euo pipefail
coder=$1
herdr agent wait "$coder" --until done --timeout 300000 || true

pane=$(herdr agent list --json | jq -r --arg n "$coder" '.result.agents[] | select(.name==$n) | .pane_id')
split=$(herdr pane split "$pane" --direction right --no-focus)
rp=$(printf '%s\n' "$split" | jq -r '.result.pane.pane_id')

herdr agent start "${coder}-rv" --kind codex --pane "$rp" -- --ask-for-approval never --sandbox read-only
herdr agent prompt "${coder}-rv" "Review the uncommitted diff and the last commits on this branch. List blocking issues first." --wait --timeout 300000
herdr agent read "${coder}-rv" --source recent-unwrapped --lines 200
```

### 3.8.3 done-task：验收后收尾

```bash
#!/usr/bin/env bash
# 用法: done-task <agent 名>
set -euo pipefail
name=$1
pane=$(herdr agent list --json | jq -r --arg n "$name" '.result.agents[] | select(.name==$n) | .pane_id')
ws=${pane%%:*}
wt=$(herdr pane get "$pane" | jq -r '.result.cwd // .result.pane.cwd')
main=$(git -C "$wt" worktree list --porcelain | awk '$1=="worktree"{print $2; exit}')
branch=$(git -C "$wt" branch --show-current)

git -C "$wt" status --short
git -C "$wt" log --oneline -10
read -r -p "PR 已合并，确认删除 $branch 与其 worktree? [y/N] " ok
[ "$ok" = y ] || exit 0

herdr workspace close "$ws"                  # 其中 agent 进程随之结束
git -C "$main" worktree remove "$wt" --force
git -C "$main" branch -d "$branch" || true
git -C "$main" worktree prune
```

每周检查一次遗留：`herdr workspace list`、`git -C ~/src/<repo> worktree list`。

### 3.8.4 命名 session 隔离

```bash
herdr --session teamA
HERDR_SESSION=teamA herdr agent list
herdr session list
```

客户端接入命名 session：`herdr machine add box --remote-session teamA`，或 `herdr --remote box --session teamA`。

### 3.8.5 编排 agent

```bash
npx skills add herdrdev/herdr --skill herdr -g     # 或 herdr --skill 查看内置技能
```

在一个 workspace 里起一个「牧羊人」Claude Code，它的 CLAUDE.md 写明：只通过 `herdr agent start / prompt / wait / read` 调度其他 agent；每个子任务先调用 `new-task`；子 agent `blocked` 时只汇报，不替人批准。技能自带「先确认 `HERDR_ENV=1`」的防护。

## 3.9 git 推送凭据与提交身份

pane 里的进程由 herdr-server 派生，**没有 laptop 的 `SSH_AUTH_SOCK`**，所以 agent 用 SSH 协议 push 会失败。三选一：

```bash
# 方案一（推荐）：gh 的 HTTPS 凭据
gh auth login --git-protocol https
gh auth setup-git

# 方案二：主机专用 deploy key
ssh-keygen -t ed25519 -f ~/.ssh/id_agent -C "agent@box"
# 在 GitHub 仓库 Settings → Deploy keys 添加 ~/.ssh/id_agent.pub，勾选 write
cat >> ~/.ssh/config <<'EOF'
Host github.com
    IdentityFile ~/.ssh/id_agent
    IdentitiesOnly yes
EOF

# 方案三：细粒度 PAT 写进 3.5 的 GH_TOKEN，gh 与 gh 配置过的 git 自动使用
```

提交身份与签名：

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
# 组织要求签名时，用 SSH 签名
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_agent.pub
git config --global commit.gpgsign true
```

macOS 宿主上 `gh` 把 token 存在 Keychain，同样要求 server 在 GUI 会话（5.5）。

## 3.10 通知

| 出口 | 覆盖 | 适合 |
|---|---|---|
| herdr 客户端 toast / 声音 | 客户端接入的所有机器 | 坐在 laptop 前 |
| Claude Code hook → ntfy | Claude Code | 手机推送，信息最准 |
| herdr-watch → ntfy | 任意 agent | 手机推送，覆盖全 |
| moshi-hook → Moshi | Claude Code、Codex、OpenCode 等 12 种 | 手机上直接点批准 |

### 3.10.1 ntfy 准备

手机安装 ntfy 应用，订阅你自己的 topic，例如 `box-agents-8f3k2`。测试：

```bash
curl -H "Title: test" -d "hello" https://ntfy.sh/box-agents-8f3k2
```

对隐私敏感可以自建 ntfy 服务，或在 topic 上加访问控制。

### 3.10.2 Claude Code 的 Notification hook

`~/.claude/hooks/ntfy-notify.sh`：

```bash
#!/usr/bin/env bash
payload=$(cat)
type=$(echo "$payload" | jq -r '.notification_type // "notification"')
msg=$(echo "$payload"  | jq -r '.message // ""')
proj=$(basename "$(echo "$payload" | jq -r '.cwd // "."')")
host=$(hostname -s)
case "$type" in
  permission_prompt|agent_needs_input|elicitation_dialog) prio=high;    tag=rotating_light ;;
  agent_completed|idle_prompt)                             prio=default; tag=white_check_mark ;;
  quota_auto_resume_*)                                     prio=low;     tag=hourglass ;;
  *)                                                       prio=low;     tag=information_source ;;
esac
curl -s -H "Title: $host/$proj: $type" -H "Priority: $prio" -H "Tags: $tag" \
     -d "$msg" https://ntfy.sh/box-agents-8f3k2 >/dev/null
```

```bash
chmod +x ~/.claude/hooks/ntfy-notify.sh
```

合并进 `~/.claude/settings.json`（里面已有 herdr 的 hook，不要覆盖）：

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "permission_prompt|agent_needs_input|agent_completed|idle_prompt|quota_auto_resume_fired",
        "hooks": [ { "type": "command", "command": "~/.claude/hooks/ntfy-notify.sh", "async": true } ]
      }
    ]
  }
}
```

在 Claude Code 里执行 `/hooks` 确认已加载。Codex 有 `~/.codex/hooks.json`，按其文档接入同一脚本。

### 3.10.3 herdr-watch：覆盖所有 agent 种类

```bash
#!/usr/bin/env bash
topic=https://ntfy.sh/box-agents-8f3k2
watch_one() {
  local a=$1
  while true; do
    if herdr agent wait "$a" --until blocked --timeout 300000 >/dev/null 2>&1; then
      curl -s -H "Title: $(hostname -s)/$a blocked" -H "Priority: high" -H "Tags: rotating_light" \
           -d "$(herdr agent read "$a" --source visible --lines 8 2>/dev/null | tail -8)" "$topic" >/dev/null
      herdr agent wait "$a" --until working --timeout 300000 >/dev/null 2>&1 || true
    fi
  done
}
for a in $(herdr agent list --json | jq -r '.result.agents[].name'); do watch_one "$a" & done
wait
```

把它放进 herdr 自己的一个 pane 里运行，随 server 一起存活：`herdr pane run w1:p9 "~/bin/herdr-watch"`。新增 agent 后重启它。

### 3.10.4 moshi-hook

**macOS：**

```bash
brew tap rjyo/moshi && brew trust rjyo/moshi && brew install moshi-hook
moshi-hook pair --token <Moshi → Settings → Agent Hooks 中的 token>
moshi-hook install
brew services start moshi-hook
moshi-hook probe                   # installed / running / gateway 均为 true
```

**Linux：**

```bash
curl -fsSL https://getmoshi.app/install.sh | sh
moshi-hook pair --token <token>
moshi-hook install
systemctl --user enable --now moshi-hook.service    # 或 moshi-hook serve
```

注意事项：
- 配对写 Keychain，macOS 上不能在纯 SSH 里做，见 5.9.2 的两种办法。
- 只转发 approval_required、task_complete、session_started、tool_running、tool_finished 五类事件的摘要，完整对话、diff、源码不经过 Moshi 服务器。
- `moshi-hook install` 会把 `~/.claude/settings.json` 从符号链接改成普通文件，dotfiles 用户要合并回去。
- `moshi-hook host setup` 的前置检查要求 tmux，使用 herdr 时加 `--force`。

## 3.11 手机端

| App | 平台 | 连接 | 特点 |
|---|---|---|---|
| Moshi | iOS / Android | mosh / SSH | 原生识别 herdr，可按 workspace 打开；配 moshi-hook 收审批卡片 |
| Redock | iOS / Android | mosh / SSH | Host / Project / Action 组织，有 herdr Chat View |
| Blink Shell | iOS | mosh / SSH | 键盘体验好 |
| Termius | iOS / Android | SSH | 最通用 |

- 手机装 Tailscale 登录同一账号；主机地址写 MagicDNS 名或 `100.x.y.z`，**不要写 `.local`**。
- 连上后执行 `herdr` 即 attach 到已有 session。
- 用完 `ctrl+b q` detach，否则 laptop 上的画面会被压到手机尺寸。

## 3.12 隔离与安全

| 措施 | 做法 |
|---|---|
| 专用用户 | Linux 上 agent 全在 `dev` 下跑；生产凭据不放这台机器 |
| 一任务一 worktree | 3.8.1；绝不让两个 agent 共用同一 checkout |
| 只从 Tailscale 进入 | 防火墙只放行 tailscale 网卡，公网 22 关闭 |
| 最小凭据 | 细粒度 token；`agents.env` 权限 600；`.env` 与 `secrets/` 在权限策略里禁止读取 |
| 插件审查 | 安装前读 `herdr-plugin.toml` 和脚本，插件以你的权限运行 |
| 审批不外包 | 编排 agent 只汇报 blocked，不替人批准 |
| 更强隔离（可选） | 每个 agent 一个 Docker 或 devcontainer，只挂对应 worktree，网络白名单；只在容器内考虑 `--dangerously-skip-permissions` |

## 3.13 磁盘与日志

```bash
# 查看占用
du -sh ~/src ~/wt ~/.herdr/worktrees ~/.claude ~/.codex ~/.config/herdr 2>/dev/null

# 每周清理（cron 或 launchd）
find ~/wt ~/.herdr/worktrees -maxdepth 3 -name node_modules -mtime +30 -prune -exec rm -rf {} + 2>/dev/null
pnpm store prune
find ~/.config/herdr -name '*.log' -size +100M -delete
```

- 统一用 pnpm，N 个 worktree 共享一个 store，占用接近 1 份。
- `~/.claude/projects/` 里的会话转录是「重启后恢复对话」的依据，不要删最近 30 天的。
- macOS 上 `~/Library/Caches` 也会增长。

---

# 第四部分　场景一：Linux 服务器实施步骤

## 4.0 前置检查

- [ ] 一台长期开机的 Linux（Ubuntu 22.04 / 24.04、Debian 12、Fedora、RHEL 系均可），x86_64 或 aarch64。
- [ ] 2 核 4 GB 起步；每个 agent 约 0.5 到 1 GB 内存，磁盘按「仓库大小 × 并发任务数」加依赖估算。
- [ ] 有 sudo 权限的初始账号，laptop 能 SSH 进去。
- [ ] 各 agent 的账号或 API key。
- [ ] 手机可装 Tailscale 与 SSH 客户端（可选）。

## 4.1 阶段一：主机基础（约 20 分钟）

### 4.1.1 专用用户

```bash
sudo adduser dev
sudo usermod -aG sudo dev
sudo -iu dev          # 之后所有步骤都以 dev 身份执行
```

### 4.1.2 基础包、语言、时区

```bash
sudo apt update && sudo apt install -y git jq curl unzip build-essential mosh locales
sudo locale-gen en_US.UTF-8 zh_CN.UTF-8
sudo timedatectl set-timezone Asia/Shanghai
```

### 4.1.3 退出 SSH 后保住用户进程

```bash
sudo loginctl enable-linger dev
loginctl show-user dev | grep Linger                  # 应为 Linger=yes
grep -i KillUserProcesses /etc/systemd/logind.conf    # 若为 yes，改为 no 后执行下一行
# sudo systemctl restart systemd-logind
```

### 4.1.4 关闭自动重启

Ubuntu 的 unattended-upgrades 若开启自动重启，会在凌晨杀掉所有 agent：

```bash
# 用单独的覆盖文件，不改动发行版自带的 50unattended-upgrades（apt 配置按文件名顺序加载，后者覆盖前者）
echo 'Unattended-Upgrade::Automatic-Reboot "false";' | sudo tee /etc/apt/apt.conf.d/99-no-auto-reboot
apt-config dump | grep Automatic-Reboot      # 应输出 "false"
```

需要重启时手动挑时间，重启后按 7.2 恢复。

### 4.1.5 sshd 加固

先完成 3.3 的公钥分发并验证能免密登录，再执行：

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
AllowUsers dev
ClientAliveInterval 30
ClientAliveCountMax 6
EOF
sudo sshd -t && sudo systemctl restart ssh
```

### 4.1.6 防火墙与爆破封禁

```bash
sudo apt install -y ufw fail2ban
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0
sudo ufw limit 22/tcp          # 过渡期保留公网 SSH；Tailscale 打通后删除此规则
sudo ufw enable
sudo systemctl enable --now fail2ban
```

Tailscale 打通并验证后：

```bash
sudo ufw delete limit 22/tcp
sudo ufw status verbose        # 只剩 tailscale0
```

## 4.2 阶段二：网络与 SSH（约 15 分钟）

1. 按 3.2.1 在 box、laptop、手机安装 Tailscale，box 用 `--hostname box`。
2. 按 3.2.2 执行 `tailscale ping -c 3 box` 确认直连。
3. 按 3.2.4 关闭 box 的密钥过期。
4. 可选按 3.2.3 开启 Tailscale SSH。
5. 按 3.3 配置 laptop 的 `~/.ssh/config` 与密钥，`ssh box 'hostname && id'` 能免密进入。
6. 按 3.1.2 处理终端 terminfo。
7. 回到 4.1.6 删除公网 22 规则。

## 4.3 阶段三：工具链与 Agent（约 20 分钟）

1. 按 3.4.1 安装 mise、Node、pnpm、Claude Code、Codex、gh。
2. 按 3.5 写好 `~/.config/agents.env` 并在 `.bashrc`、`.profile` 中加载，验证非交互 ssh 能找到命令。
3. 按 3.4.2 完成 agent 登录。
4. 按 3.9 配好 git 推送凭据与提交身份。
5. 准备仓库：

```bash
mkdir -p ~/src ~/wt ~/bin
git clone https://github.com/you/app.git ~/src/app
```

6. 在每个仓库里放 3.7.1 的 `.claude/settings.json` 和 3.7.3 的 `CLAUDE.md` / `AGENTS.md`，并提交。

## 4.4 阶段四：herdr 初始化（约 10 分钟）

1. 按 3.6.1 安装 herdr，按 3.6.2 写服务端配置，按 3.6.3 安装集成。
2. 首次启动与验证：

```bash
cd ~/src/app && herdr         # 自动拉起 server，建默认 workspace
# 在 pane 里执行 claude，侧边栏出现状态即成功；ctrl+b q 离开
herdr status server
herdr workspace list
ls ~/.config/herdr/           # 应有 herdr.sock、session.json
```

3. 断开 SSH 再连回，`herdr` 应当回到原样；`ps -u dev -o pid,cmd | grep -E 'claude|codex'` 能看到进程。
4. 把 3.8 的脚本放进 `~/bin/` 并 `chmod +x`。

### 4.4.1 关于开机自启

- 主机重启后 agent 进程本来就不会存活，自启 server 只省一次 `herdr`。
- **不要**写 systemd 单元直接执行 `herdr server`。systemd 托管的进程不是会话组长，herdr 会把它判定为不合格，之后 `herdr machine add` 会拒绝保存这台机器。
- 真要自启，用 cron 让 herdr 走自己的 spawn 路径：

```bash
crontab -e
# @reboot sleep 15 && . $HOME/.config/agents.env && (herdr status server >/dev/null 2>&1 || setsid -f herdr server)
```

- 更简单的做法：让客户端的 `herdr machine add box` 或 `herdr --remote box` 在需要时自动拉起 server。

## 4.5 阶段五：从客户端接入（约 10 分钟）

laptop 也要安装 herdr（3.6.1）。以下方式可并用：

```bash
# A. SSH 进去 attach
ssh box -t herdr

# B. 本地 herdr 远程 attach（单独窗口，见 3.1.3）
herdr --remote box
herdr --remote ssh://dev@box:22 --session teamA

# C. 推荐：把 box 并入本地 herdr 侧边栏
herdr machine add box --label "VPS"
herdr machine list
herdr

# D. 弱网 / 手机
mosh box -- herdr

# E. 纯 CLI，不进 TUI
ssh box 'herdr agent list --json'
ssh box '~/bin/new-task app fix/login-timeout claude "修复登录超时，见 issue #123"'
```

C 方式的说明见 5.8.1，两个场景通用。

## 4.6 阶段六：通知与手机（约 20 分钟）

1. 按 3.10.1 准备 ntfy。
2. 按 3.10.2 配置 Claude Code 的 Notification hook。
3. 可选按 3.10.3 在 herdr 的一个 pane 里运行 herdr-watch。
4. 可选按 3.10.4 安装 moshi-hook（Linux 版）。
5. 按 3.11 配置手机客户端。

## 4.7 阶段七：运维定时任务

```bash
crontab -e
```

```
# 每 5 分钟检查 server 是否存活
*/5 * * * * . $HOME/.config/agents.env; herdr status server >/dev/null 2>&1 || curl -s -H "Priority: urgent" -d "herdr-server on box is down" https://ntfy.sh/box-agents-8f3k2
# 每天备份会话结构
0 3 * * * mkdir -p $HOME/backup && cp $HOME/.config/herdr/session.json $HOME/backup/herdr-session-$(date +\%F).json
# 每周清理磁盘（见 3.13）
0 4 * * 0 find $HOME/wt $HOME/.herdr/worktrees -maxdepth 3 -name node_modules -mtime +30 -prune -exec rm -rf {} + ; $HOME/.local/share/mise/shims/pnpm store prune
```

## 4.8 场景一验收清单

- [ ] `ssh box` 免密秒进；`tailscale ping box` 显示直连。
- [ ] `ssh box 'which herdr claude codex'` 三个都能找到。
- [ ] 在 herdr pane 里运行 `claude`，侧边栏显示状态；detach 后进程仍在。
- [ ] 断开 SSH、重连、`herdr`，pane 与 agent 原样。
- [ ] `herdr integration status` 显示 claude、codex 为 current。
- [ ] `new-task` 能一条命令起一个带 `.env`、依赖、独立端口的 agent。
- [ ] agent 跑 `npm test`、`git commit` 不再询问；`rm -rf` 仍会 blocked。
- [ ] agent 能自己 `git push origin HEAD` 并 `gh pr create --draft`。
- [ ] laptop `herdr machine add box` 成功，侧边栏出现 VPS。
- [ ] 手机能 attach；触发一次审批，手机收到推送。
- [ ] `done-task` 能关闭 workspace 并删除 worktree。
- [ ] `sudo reboot` 后 `herdr` 能重建 workspace 并恢复对话。
- [ ] `ufw status` 只放行 tailscale0；Tailscale 管理台显示 box 密钥不过期；unattended-upgrades 不自动重启。

---

# 第五部分　场景二：两台 Mac 实施步骤

## 5.0 角色与原则

| 角色 | 机器 | 运行什么 |
|---|---|---|
| 宿主 | 家里或办公室常开的 Mac（Mac mini / Mac Studio / 旧 MacBook） | herdr-server、所有 agent、仓库、工具链、登录态 |
| 客户端 | 随身 laptop | herdr 客户端、编辑器、可选 herdrm 与 Raycast |

原则：把 herdr 放在允许一直开着的那台机器上。宿主是唯一事实来源，laptop 只是看进去的窗口。

前置：宿主 macOS 14 或更新（herdrm 需要），常插电，有线网或稳定 Wi-Fi；你知道宿主管理员账号密码；laptop 已装 Homebrew；远程场景需要 Tailscale 账号。

## 5.1 阶段一：到场清单（离开宿主前必须完成）

以下操作会弹图形对话框或需要本机密码，SSH 里做不了。**全部勾完、重启验证一次再离开。**

- [ ] 固定主机名（5.2.1）。
- [ ] 系统设置：远程登录、屏幕共享（5.2.2）；节能与自动重启（5.2.3）；自动登录或 FileVault 策略（5.2.4）。
- [ ] `xcode-select --install`（git 依赖，会弹窗）。
- [ ] 安装 Homebrew（需要 sudo 密码）。
- [ ] 隐私与安全性 → **完全磁盘访问**：打开「远程登录」和「终端」（5.4）。
- [ ] 隐私与安全性 → **自动化**：允许「终端」控制需要的应用（osascript 要用）。
- [ ] 软件更新：关闭「安装 macOS 更新」（5.2.5）。
- [ ] 在 Terminal 里运行一次 `herdr`，并在 pane 里完成 `claude`、`codex`、`gh auth login` 登录（5.5）。
- [ ] Terminal 设置「启动时运行 herdr」，登录项加入 Terminal（5.5.2）。
- [ ] Tailscale 安装、登录、关闭密钥过期（5.3.2）。
- [ ] 重启一次，确认：自动登录 → Terminal 自启 → laptop 执行 `ssh mac-mini 'herdr status server'` 正常。

## 5.2 阶段二：宿主系统设置（约 20 分钟，到场）

### 5.2.1 主机名

```bash
sudo scutil --set ComputerName mac-mini
sudo scutil --set LocalHostName mac-mini      # 决定 mac-mini.local
sudo scutil --set HostName mac-mini
```

局域网场景在路由器里给宿主做 DHCP 地址保留。

### 5.2.2 远程登录与屏幕共享

系统设置 → 通用 → 共享：
- **远程登录**：打开，「允许访问」只选 `you`。
- **屏幕共享**：打开。之后从 laptop 用 `open vnc://mac-mini` 连接，用于点授权弹窗、首次登录、系统更新。

命令行等价：`sudo systemsetup -setremotelogin on`。

### 5.2.3 不睡觉、断电后自启

```bash
sudo pmset -a sleep 0 disksleep 0 womp 1 autorestart 1
sudo pmset -a displaysleep 10          # 显示器可以关
sudo pmset -a disablesleep 1           # 仅当宿主是笔记本、合盖也不能睡时
pmset -g                               # 核对
```

图形界面对应：节能中打开「显示器关闭时防止自动进入睡眠」「唤醒以供网络访问」「断电后自动启动」。临时防睡可用 `caffeinate -s <命令>`。

### 5.2.4 重启后自动回到登录态

系统设置 → 用户与群组 → 自动登录选 `you`。开着 FileVault 时自动登录不可用，二选一：

- 宿主放在可信环境：关闭 FileVault。
- 保留 FileVault：远程重启一律用认证重启，下一次开机自动解锁一次：

```bash
fdesetup supportsauthrestart          # 输出 true 才能用
sudo fdesetup authrestart             # 输入一次密码，立即重启
```

### 5.2.5 关闭 macOS 自动更新重启

系统设置 → 通用 → 软件更新 → 自动更新：关闭「安装 macOS 更新」，保留「安全响应与系统文件」。需要升级时手动执行（7.3）。

### 5.2.6 sshd 只允许密钥

先完成 5.3 与 3.3 的公钥分发并验证，再执行：

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
AllowUsers you
ClientAliveInterval 30
ClientAliveCountMax 6
EOF
sudo sshd -t && sudo launchctl kickstart -k system/com.openssh.sshd
```

### 5.2.7 基础工具

```bash
xcode-select --install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install git jq mosh terminal-notifier herdr mise gh
```

## 5.3 阶段三：网络（两种场景）

### 5.3.1 同一局域网

```bash
ssh you@mac-mini.local 'sw_vers -productVersion'
```

`.local` 依赖 Bonjour，只在同一网段有效。公司网络屏蔽 mDNS 时用 DHCP 保留的固定 IP。

### 5.3.2 远程（宿主在 NAT 后面）

按 3.2.1：宿主装 **开源 tailscaled 版**（`brew install tailscale`，无人登录时也运行，并支持 Tailscale SSH），laptop 装 **Standalone 版**，手机装 App。然后：

```bash
# 宿主
sudo tailscale up --hostname mac-mini
sudo tailscale set --ssh        # 可选
# laptop
tailscale ping -c 3 mac-mini
ssh you@mac-mini 'hostname'
```

按 3.2.4 在管理台关闭 mac-mini 的密钥过期。

### 5.3.3 一个别名覆盖两种场景

两台都常驻 Tailscale 时，同一局域网内 Tailscale 会自动走局域网直连，所以最简单的写法是 HostName 只写 Tailscale 名：

```
Host mac-mini
    HostName mac-mini              # MagicDNS
    User you
    SetEnv TERM=xterm-256color
    ServerAliveInterval 30
    ServerAliveCountMax 6
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

若希望在家时不经过 Tailscale，在上面这段**之前**加一段自动切换（ssh 对同一个键取第一次出现的值）：

```
Match host mac-mini exec "ping -c1 -t1 mac-mini.local >/dev/null 2>&1"
    HostName mac-mini.local
```

### 5.3.4 备选

Cloudflare Tunnel 或端口转发见 3.2.5。

## 5.4 阶段四：macOS 隐私保护（TCC）

**现象**：`ssh mac-mini 'ls ~/Documents'` 报 `Operation not permitted`；agent 读 `~/Downloads/spec.pdf` 失败。macOS 对 Desktop、Documents、Downloads、iCloud、外接卷做了访问保护。

**两条都做**：
1. 仓库放在不受保护的目录：`~/src`、`~/wt`、`~/.herdr/worktrees`。
2. 系统设置 → 隐私与安全性 → 完全磁盘访问：打开「远程登录」和「终端」。Terminal 首次访问受保护目录时也会弹窗，用屏幕共享点允许。

## 5.5 阶段五：宿主上的 agent 与 herdr（硬性前置：GUI 会话）

### 5.5.1 为什么必须从 GUI 会话启动 server

Claude Code 的 OAuth 凭据、`gh` 的 token、1Password CLI、moshi-hook 配对都依赖 login Keychain。SSH 会话拿不到已解锁的 login Keychain，从 SSH 拉起的 herdr-server 及其所有 pane 都会继承这个缺陷，表现为 agent 反复要求登录、`gh` 报未认证、`security` 报 `User interaction is not allowed`。

由 GUI 会话启动的 server，其 pane 继承 GUI 会话权限，这些问题都不存在。它走的是 herdr 自己的 spawn，也满足 `herdr machine add` 的检查。**不要**改成 launchd 直接托管 `herdr server`，那样 `machine add` 会报 `remote server is not ready for saved machines`。

### 5.5.2 设置 Terminal 随登录启动 herdr

1. Terminal → 设置 → 描述文件 → 选默认描述文件 → Shell → 「启动时运行命令」填 `herdr`，勾选「在 Shell 中运行」。
2. 系统设置 → 通用 → 登录项与扩展 → 「登录时打开」加入「终端」。
3. 配合 5.2.4 的自动登录，重启后链路为：自动登录、Terminal 打开、`herdr` 拉起 server、重建 workspace、恢复 agent 对话。

### 5.5.3 在 GUI 会话里完成安装与登录

在 Terminal 启动的 herdr pane 里（到场或屏幕共享）依次执行：

1. 按 3.5 把 PATH 和 API key 写进 `~/.zshenv`。
2. 按 3.4.1 安装 mise、Node、pnpm、Claude Code、Codex。
3. 按 3.4.2 登录 `claude`、`codex`；`gh auth login --git-protocol https && gh auth setup-git`。
4. 按 3.9 配置提交身份。
5. 按 3.6.2 写服务端配置，按 3.6.3 安装集成，按 3.6.4 注册插件。
6. 准备仓库：`mkdir -p ~/src ~/bin && git clone https://github.com/you/app.git ~/src/app`；放入 3.7 的权限文件与规则文件；把 3.8 的脚本放进 `~/bin/`。

### 5.5.4 验证 server 在 GUI 会话

```bash
ssh mac-mini 'herdr status server && herdr workspace list && herdr integration status'
```

在任一 herdr pane 里执行：

```bash
security find-generic-password -s nonexistent-check 2>&1 | head -1
```

输出含 `could not be found` 表示正常；输出 `User interaction is not allowed` 说明 server 在 SSH 会话，需要 `herdr server stop` 后从 Terminal 重新启动。

### 5.5.5 重启后的顺序

宿主重启后，先等自动登录与 Terminal 自启把 server 拉起，再从 laptop 连接。若 laptop 的 `herdr machine add` 抢先连上，它会经 SSH 拉起一个没有 Keychain 权限的 server。连接前先确认：

```bash
ssh mac-mini 'herdr status server'
```

## 5.6 阶段六：终端选择

按 3.1：laptop 用 Ghostty（次选 iTerm2），并完成 3.1.2 的 terminfo 处理和 3.1.5 的输入法设置；宿主上用 Terminal.app 作为 server 启动器；专用远程窗口用 3.1.3。

## 5.7 阶段七：laptop 准备

```bash
brew install herdr mosh terminal-notifier
brew install --cask tailscale-app ghostty
```

按 3.3 分发公钥，按 3.6.2 写客户端配置。

## 5.8 阶段八：laptop 接入宿主的四种方式

### 5.8.1 方式一（推荐）：把宿主并入本地 herdr

```bash
herdr machine add mac-mini --label "Home Mac mini"
herdr machine add mac-mini --label "Home Mac mini" --remote-session agents   # 接宿主的命名 session
herdr machine list
herdr
```

- 侧边栏同时列出本机和 Home Mac mini 的 workspace；点击，或 `prefix+w` 加上下键切换，Enter 进入。输入和窗口尺寸只发给当前选中的机器。
- 断网、合盖、SSH 失败后自动重连，退避最长 2 分钟；连接健康保持一分钟后才恢复快速重试。
- 侧边栏显示 Attention 表示需要交互（首次 host key、密钥口令）。执行 `herdr --remote mac-mini` 走完提示，再重启客户端。
- 多机器时 CLI 用 `herdr --machine "Home Mac mini" agent list` 指定目标。
- 管理：`herdr machine rename|disable|enable|remove <id>`。disable 和 remove 只断开连接，宿主上的 agent 不受影响。
- profile 只保存 ID、标签、SSH 目标、session 名和启用状态，不保存密码或私钥。

### 5.8.2 方式二：专用远程窗口

```bash
osascript -e 'tell application "Terminal" to do script "herdr --remote mac-mini"'
```

或 3.1.3 的其他终端写法。适合全屏只看宿主。

### 5.8.3 方式三：socket 转发，让本地 CLI 与 Raycast 操作宿主

`~/bin/herdr-mm`（隧道断开时自动重建）：

```bash
#!/bin/sh
set -eu
host=mac-mini
sock=/tmp/herdr-${host}.sock
bin=/opt/homebrew/bin/herdr
if [ ! -S "$sock" ] || ! HERDR_SOCKET_PATH="$sock" "$bin" status server >/dev/null 2>&1; then
  rm -f "$sock"
  rhome=$(ssh -o BatchMode=yes "$host" 'printf %s "$HOME"')
  ssh -o BatchMode=yes -fnNT -L "$sock:$rhome/.config/herdr/herdr.sock" "$host"
fi
HERDR_SOCKET_PATH="$sock" exec "$bin" "$@"
```

```bash
chmod +x ~/bin/herdr-mm
herdr-mm workspace list
herdr-mm agent list
```

- herdr 命令通过 socket 作用于宿主；涉及宿主上 git 和文件的脚本仍用 `ssh mac-mini '~/bin/new-task …'`。
- Raycast：安装 Store 中的 **Herdr** 扩展，偏好设置「Herdr Binary」填 `~/bin/herdr-mm`，「Custom Terminal Launcher」填 5.8.2 的脚本。之后菜单栏状态、切换 workspace、创建 worktree、给 agent 发指令都作用于宿主。只能指定一个 Binary，扩展会整体切换到宿主。

### 5.8.4 方式四：原生应用 herdrm

```bash
brew install owo-network/brew/herdrm
```

本机走 Unix socket，宿主填 SSH 目标（支持密钥、Tailscale SSH、Keychain 密码），一个侧边栏列出两台机器的 workspace、agent 和终端，带文件浏览与上传下载。要求 macOS 14+，两台都装 herdr；非商业使用免费。

## 5.9 阶段九：通知与手机

### 5.9.1 通知出口

| 出口 | 配置 |
|---|---|
| laptop 系统通知 | 3.6.2 客户端配置 `delivery = "system"`，不弹时改 `"terminal"`；通过 `machine add` 接入的宿主 agent 一并提醒 |
| ntfy | 3.10.1 至 3.10.3，脚本放在宿主 |
| Moshi 审批卡片 | 3.10.4 加 5.9.2 |

### 5.9.2 moshi-hook 在宿主上的配对

配对要写 Keychain，不能在纯 SSH 里做。二选一：

```bash
# 办法一：借 herdr pane 的 GUI 会话权限
ssh mac-mini 'herdr pane split w1:p1 --direction down --no-focus'
ssh mac-mini 'herdr pane run w1:p2 "moshi-hook pair --token <token>"'
ssh mac-mini 'herdr pane read w1:p2 --source visible'

# 办法二：存文件而不是 Keychain
ssh mac-mini 'moshi-hook pair --token <token> --store file'
```

之后：

```bash
ssh mac-mini 'moshi-hook install && brew services start moshi-hook && moshi-hook probe'
```

从 SSH 执行 `moshi-hook status` 可能显示 unpaired，这是因为 SSH 读不到 Keychain，以 pane 里的结果为准。

### 5.9.3 手机

按 3.11。主机地址写 `mac-mini` 或 `100.x.y.z`，不要写 `.local`。

## 5.10 场景二验收清单

- [ ] 局域网 `ssh mac-mini` 免密；关掉 Wi-Fi 改用手机热点后同一命令仍可进。
- [ ] `ssh mac-mini 'ls ~/src && which herdr claude codex'` 无权限错误、命令都能找到。
- [ ] 宿主 `pmset -g` 显示 sleep 0、womp 1；拔掉显示器一小时后仍能 ssh。
- [ ] pane 内 Keychain 检查通过（5.5.4）；`gh auth status` 正常；`claude` 不要求重新登录。
- [ ] 宿主重启后：自动登录、Terminal 自启、`herdr status server` 正常、workspace 已重建、对话已恢复。
- [ ] `sudo fdesetup authrestart`（如开 FileVault）后无需到场即可恢复。
- [ ] laptop `herdr machine add mac-mini` 成功；合盖再开盖自动重连。
- [ ] `herdr-mm agent list` 能列出宿主 agent；Raycast 菜单栏显示状态。
- [ ] `ssh mac-mini '~/bin/new-task app demo claude "写一行 README"'` 能跑通，并自动 push、开草稿 PR。
- [ ] 手机能 attach；触发一次审批，手机收到卡片或推送并能批准。
- [ ] `open vnc://mac-mini` 能看到宿主桌面。
- [ ] Tailscale 管理台显示 mac-mini 密钥不过期；软件更新不会自动安装 macOS 更新。

---

# 第六部分　日常使用流程

## 6.1 任务生命周期（两个场景相同）

| 步骤 | 做什么 | 命令或位置 |
|---|---|---|
| 1 看状态 | 打开 herdr，扫一眼侧边栏 | `herdr`（场景二侧边栏里切到 Home Mac mini） |
| 2 派任务 | 一条命令开 worktree、备环境、起 agent | `ssh box '~/bin/new-task app fix/x claude "…"'`；场景二把 `box` 换成 `mac-mini` |
| 3 运行 | 常规操作自动放行，不打扰 | 3.7 的权限策略 |
| 4 审批 | 手机收到 blocked 推送，看清再批 | 3.7.4；或 Moshi 卡片 |
| 5 完成 | agent 按规则 commit、push、开草稿 PR | 3.7.3 |
| 6 评审 | 可选起一个只读评审 agent | `review <name>` |
| 7 验收 | 看 PR、跑 CI、必要时 attach 进去追问 | `herdr agent attach <name>` |
| 8 收尾 | 合并后关 workspace、删 worktree | `done-task <name>` |

## 6.2 场景一的一天

1. 早上在 laptop 打开 `herdr`，侧边栏里 VPS 的三个 workspace：一个绿、一个蓝、一个红。
2. 红色的点进去，`pane read` 看是在请求安装新依赖，确认后批准。
3. 蓝色的 PR 已开好，看完合并，执行 `ssh box 'done-task fix-login-timeout'`。
4. 派两个新任务，通勤时合上电脑。
5. 地铁上手机收到 ntfy：一个 agent 要删除文件。用 Moshi 连上 `herdr`，看清后批准，detach。
6. 夜里 server 持续运行；unattended-upgrades 不会重启，磁盘有每周清理。

## 6.3 场景二的一天

1. 早上在家（局域网），laptop 打开 `herdr`，侧边栏同时有本机和 Home Mac mini。
2. 用 Raycast 菜单栏看到宿主有一个 agent 在等待，切过去批准。
3. `ssh mac-mini '~/bin/new-task app feat/y claude "…"'` 再开一个任务。
4. 合盖去办公室。宿主上的 agent 继续干活；手机 Moshi 卡片处理审批。
5. 到办公室开盖，herdr 通过 Tailscale 自动重连，画面延续。
6. 晚上 `herdr-mm agent list` 看 done 的任务，验收合并后 `ssh mac-mini 'done-task …'`。
7. 需要点系统弹窗时 `open vnc://mac-mini`。

## 6.4 每周例行

```bash
# 两个场景：把 host 换成 box 或 mac-mini
ssh host 'herdr update --handoff && herdr status server'
ssh host 'claude update 2>/dev/null; npm update -g @openai/codex'
ssh host 'herdr integration status'           # agent 升级后 hook 可能被覆盖
ssh host 'herdr workspace list'                # 清理遗留 workspace
ssh host 'git -C ~/src/app worktree list'      # 清理遗留 worktree
ssh host 'df -h ~ && du -sh ~/wt ~/.herdr/worktrees 2>/dev/null'
```

---

# 第七部分　运维

## 7.1 升级 herdr

```bash
herdr update --handoff        # 把运行中的 pane 移交给新 server，尽量不杀进程
herdr status server
```

不带 `--handoff` 等于重启 server，agent 进程会结束，对话可由原生会话恢复。Homebrew 安装的用 `brew upgrade herdr`，之后再执行 `herdr update --handoff` 或在方便时重启 server。两端版本尽量同步。

## 7.2 主机重启后恢复

```bash
herdr                         # 重建布局；resume_agents_on_restore 拉回 Claude / Codex 对话
herdr integration status
herdr agent list
```

场景二先确认 server 已由 GUI 会话拉起（5.5.5）。

## 7.3 系统更新与重启

```bash
# Linux
sudo apt update && sudo apt upgrade -y
sudo reboot

# macOS
ssh mac-mini 'sudo softwareupdate -l'
ssh mac-mini 'sudo softwareupdate -i -a'          # 不带 --restart
ssh mac-mini 'sudo fdesetup authrestart'          # 开 FileVault 时
ssh mac-mini 'sudo shutdown -r now'               # 关 FileVault 且有自动登录时
```

macOS 大版本升级建议屏幕共享或到场进行。

## 7.4 日志与诊断

```bash
HERDR_LOG=herdr=debug herdr
ls ~/.config/herdr/                         # herdr.sock、session.json、session-backups/、日志
herdr agent explain w1:p1 --json            # 状态判断依据
herdr server update-agent-manifests         # 拉取最新检测规则
herdr api snapshot                          # 完整状态快照
```

## 7.5 停止一切

```bash
herdr server stop       # 该 session 的所有 agent 一起结束，慎用
```

---

# 第八部分　故障排查

| 现象 | 场景 | 原因 | 处理 |
|---|---|---|---|
| `ssh box 'herdr …'` 报 command not found | 一 | 非交互 shell 在 `.bashrc` 顶部返回 | 3.5 |
| pane 里没有 API key | 一 | 用了 `environment.d` | 3.5 |
| 退出 SSH 后 agent 全没了 | 一 | 未开 linger 或 `KillUserProcesses=yes` | 4.1.3 |
| 半夜 agent 全部消失 | 一 | unattended-upgrades 自动重启 | 4.1.4 |
| 半夜 agent 全部消失 | 二 | macOS 自动更新重启 | 5.2.5 |
| 半夜 ssh 不通，早上又好了 | 二 | 宿主睡眠 | 5.2.3；`pmset -g log \| grep -i sleep` |
| 某天远程突然连不上 | 两者 | Tailscale 节点密钥过期 | 3.2.4 |
| 重启后卡在解锁界面 | 二 | FileVault 且无自动登录 | 5.2.4 |
| `herdr machine add` 报 server not ready | 两者 | server 被 systemd 或 launchd 托管 | 4.4.1；5.5.1 |
| agent 反复要求登录；`gh` 未认证 | 二 | server 从 SSH 会话启动，读不到 Keychain | 5.5.4 后重启 server |
| `Operation not permitted` | 二 | macOS TCC | 5.4 |
| `missing or unsuitable terminal: xterm-ghostty` | 两者 | 远端缺 terminfo | 3.1.2 |
| `--remote` 报 nested；前缀键无效 | 两者 | 在本地 herdr 里再开 herdr | 3.1.3 或改用 `machine add` |
| VS Code 里按 `ctrl+b` 切了侧边栏 | 两者 | 编辑器截获快捷键 | 3.1.4 |
| 中文输入法下前缀键无反应 | 两者 | 输入法吞键 | 3.1.5 |
| 远程 pane 粘贴图片无反应 | 两者 | 剪贴板与进程不在同一台机器 | 3.1.6 |
| agent 一直 `unknown` | 两者 | pane 里套了 tmux 或包装脚本 | 让 agent 直接是前台进程；`herdr agent explain` |
| agent 每一步都 blocked | 两者 | 没有权限策略 | 3.7 |
| agent 第一步跑测试就失败 | 两者 | worktree 缺 `.env` 或依赖 | 3.8.1 |
| 两个 dev server 端口冲突 | 两者 | 端口未隔离 | 3.8.1 |
| agent `git push` 失败 | 两者 | pane 里没有 SSH agent 转发 | 3.9 |
| 所有 agent 同时停下等待 | 两者 | 订阅配额触顶 | 3.4.3 |
| 磁盘满 | 两者 | worktree 依赖堆积 | 3.13 |
| 手机接入后 laptop 画面变矮 | 两者 | 多客户端取最小尺寸 | 手机用完 detach |
| `.local` 解析失败 | 二 | 不在同一网段或 mDNS 被屏蔽 | 用 Tailscale 名或固定 IP |
| Tailscale SSH 起不来 | 二 | 宿主装的是 GUI 版 | 换 `brew install tailscale` |
| 两个 Tailscale 冲突 | 二 | 同机装了 GUI 版和 tailscaled | 只留一个，删 app、清废纸篓、重启 |
| Tailscale 显示走 DERP、很卡 | 两者 | NAT 严格 | 路由器开 UPnP 或放行 UDP 41641 |
| mosh 连不上 | 两者 | UDP 不通或 locale 非 UTF-8 | 走 Tailscale；检查 `locale` |
| 插件不触发 | 两者 | 未在跑 server 的机器上注册 | 3.6.4 |
| 通知没到 | 两者 | hook 未合并或脚本无执行权限 | `/hooks` 检查；`chmod +x` |
| 重启后对话没恢复 | 两者 | 集成未安装或 agent 版本过低 | 3.6.3 |
| 升级 herdr 后 agent 全没了 | 两者 | 没用 `--handoff` | 7.1 |
| 换网后 ssh 卡住 | 两者 | ControlMaster 连接僵死 | `ssh -O exit <host>` |
| moshi-hook 在 SSH 里显示 unpaired | 二 | SSH 读不到 Keychain | 以 pane 里的 `moshi-hook status` 为准 |

---

# 附录 A　CLI 速查

```bash
# 基础
herdr                                   # 启动或接入默认 session
herdr --session lab                     # 命名 session
herdr status / herdr status server
herdr server stop / herdr server reload-config
herdr session list / herdr session attach <name>
herdr update [--handoff] / herdr channel set stable|preview
herdr --default-config / herdr --skill

# 多机器
herdr machine add <ssh目标> --label "名称" [--remote-session <name>]
herdr machine list [--json] / rename / disable / enable / remove <id>
herdr --remote <ssh目标> [--session <name>]
herdr --machine <label或id> agent list

# 布局
herdr workspace create --cwd ~/project --label api --no-focus
herdr workspace list [--json] / herdr workspace close <id>
herdr worktree create --cwd ~/repo --branch feat-x --no-focus
herdr tab create --label logs
herdr pane split w1:p1 --direction right --no-focus
herdr pane get w1:p1

# pane 控制
herdr pane run w1:p2 "npm test"
herdr pane read w1:p2 --source recent --lines 50        # 也可 --source visible
herdr pane send-text w1:p1 "文字"                        # 不回车
herdr pane send-keys w1:p1 enter                         # 也可 ctrl+c、esc
herdr pane wait-output w1:p3 --regex "passed|failed" --timeout 120000

# agent
herdr agent list [--json]
herdr agent start <name> --kind claude --pane w1:p2 [-- <agent 参数>]
herdr agent prompt <name> "提示词" [--wait --timeout 120000]
herdr agent wait <name> --until blocked|done|idle|working --timeout 120000
herdr agent read <name> --source recent-unwrapped --lines 120
herdr agent send-keys <name> esc
herdr agent attach <name>                                # ctrl+b q 离开
herdr agent rename w1:p1 <name>
herdr agent explain w1:p1 --json

# 集成与插件
herdr integration install claude|codex / herdr integration status
herdr plugin install owner/repo[/subdir] / herdr plugin link <path> / herdr plugin list
herdr server update-agent-manifests / reload-agent-manifests

# API
herdr api snapshot / herdr api schema --json
```

`wait` 类命令没有默认超时，必须给 `--timeout`（3000 到 300000 毫秒）。创建类命令返回 JSON，ID 从 `.result` 里取，不要自己推测。

**快捷键**（前缀默认 `ctrl+b`）：

| 动作 | 键 |
|---|---|
| 右分 / 下分 | `prefix+v` / `prefix+-` |
| 新 tab / 下一个 / 上一个 | `prefix+c` / `prefix+n` / `prefix+p` |
| pane 间移动 | `prefix+h/j/k/l` |
| 新 workspace / 切换（含跨机器） | `prefix+shift+n` / `prefix+w` |
| Detach | `prefix+q` |
| 帮助 | `prefix+?` |

鼠标：点击聚焦、拖边框、右键菜单、拖选即复制、ctrl+点击打开链接。

# 附录 B　配置与环境变量速查

配置文件：Linux / macOS 为 `~/.config/herdr/config.toml`，Windows 为 `%APPDATA%\herdr\config.toml`。

| 键 | 默认 | 说明 |
|---|---|---|
| `keys.prefix` | `ctrl+b` | 前缀键 |
| `server.headless_cols` / `headless_rows` | — | 无客户端时 pane 虚拟尺寸 |
| `session.resume_agents_on_restore` | `true` | 重启后恢复 agent 对话 |
| `session.pane_history` | `false` | 重启后回放屏幕，可能泄露密钥 |
| `terminal.default_shell` | `""` | 空则用 `$SHELL` |
| `terminal.shell_mode` | `auto` | `auto` / `login` / `non_login` |
| `terminal.new_cwd` | `follow` | 新 pane 目录策略 |
| `terminal.kitty_graphics` | `true` | 外层终端支持 Kitty 协议时渲染图片 |
| `ui.toast.delivery` | `herdr` | `herdr` / `terminal` / `system` / `off` |
| `ui.toast.herdr.position` | — | 应用内 toast 位置 |
| `ui.sound.*` | — | 提示音文件与按 agent 开关 |
| `theme.name` / `auto_switch` | — | 主题与明暗自动切换 |
| `experimental.allow_nested` | `false` | 允许在 herdr pane 里再开 herdr |
| `experimental.switch_ascii_input_source_in_prefix` | `false` | 前缀时切英文输入源 |
| `experimental.reveal_hidden_cursor_for_cjk_ime` | `false` | 修正输入法光标位置 |

| 环境变量 | 作用 |
|---|---|
| `HERDR_CONFIG_PATH` | 覆盖配置文件路径 |
| `HERDR_SESSION` | 选择命名 session |
| `HERDR_SOCKET_PATH` | 覆盖 socket 路径（socket 转发时使用） |
| `HERDR_LOG` | 日志级别，如 `herdr=debug` |
| `HERDR_DISABLE_SOUND` | 禁用声音 |
| `HERDR_ENV`、`HERDR_PANE_ID`、`HERDR_WORKSPACE_ID`、`HERDR_TAB_ID`、`HERDR_BIN_PATH` | herdr 注入 pane 的变量，供 agent 和脚本识别自身位置 |

# 附录 C　落地前请在你的机器上核对

这些内容依赖具体版本，文档中的写法是按官方文档与社区文章推断的：

1. **脚本中的 JSON 字段路径**：`.result.root_pane.pane_id`、`.result.pane.pane_id`、`.result.agents[].name`、`.result.agents[].pane_id`、`herdr pane get` 输出中的 `cwd`。以 `herdr api schema --json` 和实际命令输出为准，必要时调整 3.8、3.10.3 的脚本。
2. **Claude Code 权限规则语法**：3.7.1 的通配写法以你当前 Claude Code 版本的权限文档为准，可在 Claude Code 里用 `/permissions` 检查。
3. **Codex 命令参数**：`codex login --device-auth`、`--ask-for-approval`、`--sandbox`，以及 `~/.codex/hooks.json` 的通知格式，以 Codex 当前文档为准。
4. **Ghostty `+ssh` CLI**：官方标注在 1.4 起提供；`ssh-env`、`ssh-terminfo` 两个 shell 集成特性从 1.2 起可用。
5. **herdr 远程贴图**：相关 issue 已关闭，但修复版本未确认；herdr 插件在远程模式下在哪台机器执行也未确认。先用 scp 后贴宿主路径的方式（3.1.6）。
6. **Windows 作为被接入的宿主**：0.9 文档写支持 x86_64，但属于最不成熟的路径，本手册未覆盖。

# 附录 D　参考来源

**herdr 官方**
- 官网与文档：herdr.dev、/docs/install、/docs/quick-start、/docs/concepts、/docs/agents、/docs/connecting-machines、/docs/session-state、/docs/configuration、/docs/config-reference、/docs/socket-api、/docs/integrations、/docs/plugins、/docs/keyboard、/compare、/agent-guide.md
- 博客《Connecting the machines》(2026-09-07)
- GitHub：github.com/herdrdev/herdr，issue #205（远程贴图）、#4330（Sixel 不透传）、discussion #820（Warp）

**实战文章**
- DevelopersIO《Make Mac mini a headless server and operate Claude Code from outside》(2026-08-07)
- getmoshi.app《Using Your Mac as a Remote Endless Working Agent with Moshi》与 moshi-hook 文档
- redock.dev《How to Run Codex on Phone with SSH, Tailscale, and Herdr》(2026-08-30)
- jacar.es herdr 评测；Eric Grill《I Replaced tmux and iTerm2 With Herdr and Ghostty》
- 腾讯云开发者社区、vocus、53ai、zheyu.me、x-cmd 介绍文章
- firstmate issue #4642（launchd 托管 herdr server 与 machine add 的冲突）

**生态工具**
- Tailscale：Tailscale SSH、MagicDNS、Linux 安装、macOS 三种版本
- Ghostty：SSH 集成与 terminfo 文档；WezTerm SSH 文档
- mosh.org；ntfy 发布文档；Claude Code hooks 参考
- herdrm（github.com/missuo/herdrm）；Raycast Herdr 扩展；herdr-paste-image 插件
- systemd `loginctl` 手册；macOS `pmset`、`fdesetup authrestart`

**未能读取**：两篇知乎文章需要登录，YouTube 视频无法抓取，其内容已由上述来源交叉覆盖。
