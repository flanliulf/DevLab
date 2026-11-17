# 实施指南二：用日常 Mac 驱动家里 / 办公室另一台 Mac 上的多个 AI 编程 Agent

> 配套阅读：[herdr-总结与部署指南.md](herdr-总结与部署指南.md)（herdr 本体）与 [实施指南-1-Linux服务器多Agent.md](实施指南-1-Linux服务器多Agent.md)（脚本模板、ntfy 通知、编排 agent 部分两边通用）。
> 整理日期 2026-09-22。基于 herdr v0.9.x、macOS 26、Tailscale 1.9x、Claude Code 2.1.x。实战依据：DevelopersIO《Make Mac mini a headless server and operate Claude Code from outside》、getmoshi.app《Using Your Mac as a Remote Endless Working Agent》、redock.dev《Run Codex on Phone with SSH, Tailscale and Herdr》、herdr 官方 docs 与博客《Connecting the machines》。
> 约定：常开的宿主 Mac 叫 `mac-mini`（Mac mini / Mac Studio / 旧 MacBook 均可），宿主用户 `you`；随身日常 Mac 叫 `laptop`。

---

## 0. 目标架构与工具选型

```
        laptop（日常 Mac）                              iPhone（可选）
  ┌───────────────────────────┐                 ┌──────────────────────┐
  │ herdr TUI  ← machine add  │                 │ Moshi / Redock /     │
  │ herdr --remote 专用窗口    │                 │ Termius / Blink      │
  │ herdrm.app / Raycast 扩展  │                 │ + moshi-hook 审批卡片 │
  └───────┬─────────┬─────────┘                 └──────────┬───────────┘
   同一局域网         远程                                    │
   .local/mDNS       Tailscale (WireGuard)          Tailscale + mosh
          └────────────┬──────────────────────────────────────┘
                       ▼
        ┌────────── mac-mini（宿主，常开，不睡）──────────┐
        │ 远程登录(sshd) · tailscaled · mosh-server         │
        │ herdr-server（从 GUI 登录会话启动，能用 Keychain） │
        │   ├─ w1 app      [claude]  [codex]               │
        │   ├─ w2 wt/fix-x [claude]                         │
        │   └─ w3 wt/docs  [opencode]                      │
        │ 集成: herdr integration claude/codex · moshi-hook │
        └──────────────────────────────────────────────────┘
```

| 层 | 工具 | 必要性 | 作用 |
|---|---|---|---|
| 运行时 | **herdr**（两台都装） | 必选 | 宿主跑 server；laptop 跑客户端并把宿主并进侧边栏 |
| Agent | Claude Code / Codex / OpenCode … | 必选 | 只装在宿主 |
| 远程 shell | **macOS 远程登录 (OpenSSH)** | 必选 | 认证与通道 |
| 局域网寻址 | **Bonjour `.local` + DHCP 保留** | 局域网场景 | 免配置找到宿主 |
| 远程网络 | **Tailscale** | 远程场景必选 | NAT 穿透、MagicDNS、不开路由器端口 |
| 弱网 | **mosh** | 手机推荐 | 换网不断 |
| 密钥 | **ssh-keygen / 1Password SSH Agent** | 必选 | 免密登录 |
| 常开 | **pmset / caffeinate / 自动登录 / `fdesetup authrestart`** | 必选 | 宿主不睡、重启后能回来 |
| 偶尔看图形界面 | **屏幕共享（VNC）** | 推荐 | 首次从 GUI 会话起 herdr、处理弹窗 |
| 桌面客户端 | **herdrm**（原生 macOS） | 可选 | 多机 agent 一个侧边栏 |
| 快捷操作 | **Raycast Herdr 扩展** | 可选 | 菜单栏看状态、⌘Space 切 workspace |
| 手机 | **Moshi + moshi-hook / Redock / Termius / Blink** | 可选 | 手机上看状态、批审批 |
| 通知 | **herdr toast (system) + terminal-notifier / ntfy** | 推荐 | blocked/done 提醒 |
| 版本管理 | **git worktree** | 必选 | 一任务一目录 |

---

## 1. 前置检查清单

- [ ] 宿主 Mac 常插电、有线网或稳定 Wi‑Fi，macOS 14+（herdrm 需要）。
- [ ] 宿主上已有一个管理员账号 `you`，你知道密码。
- [ ] laptop 上有 Homebrew。
- [ ] 两台机器都能上网；远程场景需要一个 Tailscale 账号（个人免费）。
- [ ] 各 agent 账号可用。

---

## 2. 阶段一：宿主 Mac 系统设置（约 20 分钟，需要在宿主前操作一次）

### 2.1 主机名固定

```bash
sudo scutil --set ComputerName mac-mini
sudo scutil --set LocalHostName mac-mini      # 决定 mac-mini.local
sudo scutil --set HostName mac-mini
```

路由器里给宿主做 DHCP 地址保留（局域网场景更稳）。

### 2.2 开远程登录（SSH）和屏幕共享

- 系统设置 → 通用 → 共享 → **远程登录** 打开，「允许访问」只选 `you`。
- 同一页 **屏幕共享** 打开（偶尔需要看图形界面、点系统弹窗时用；通过 Tailscale 也能用）。
- 命令行等价：

```bash
sudo systemsetup -setremotelogin on
```

### 2.3 不睡觉、断电后自启

图形界面：系统设置 → 节能（或电池）→ 打开「显示器关闭时防止自动进入睡眠」「唤醒以供网络访问」「断电后自动启动」。

命令行等价（推荐，可复现）：

```bash
sudo pmset -a sleep 0 disksleep 0 womp 1 autorestart 1
sudo pmset -a displaysleep 10          # 显示器可以关
sudo pmset -a disablesleep 1           # 笔记本当宿主、合盖也不睡时加这条
pmset -g                               # 核对
```

临时用法：`caffeinate -s claude` 让某个命令期间不睡。

### 2.4 重启后能自动回到登录态

- 系统设置 → 用户与群组 → **自动登录** 选 `you`。
- 开着 FileVault 时自动登录不可用，重启会卡在解锁界面。两种选择：
  - 宿主放在家里可接受的话，**关闭 FileVault**（设置 → 隐私与安全性 → FileVault）。
  - 保留 FileVault，但远程重启一律用「认证重启」，下一次开机跳过解锁：

```bash
fdesetup supportsauthrestart          # true 才能用
sudo fdesetup authrestart             # 输一次密码，立即重启并自动解锁一次
```

### 2.5 sshd 只允许密钥（把 2.6 的公钥放好、验证通过后再做）

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

### 2.6 Homebrew 与基础工具

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install git jq mosh terminal-notifier herdr
```

---

## 3. 阶段二：网络（两种场景）

### 3.1 场景 A：两台 Mac 在同一局域网

laptop 上：

```bash
ssh you@mac-mini.local 'sw_vers -productVersion'     # 通了就行
```

`~/.ssh/config`：

```
Host mac-mini
    HostName mac-mini.local
    User you
    ServerAliveInterval 30
    ServerAliveCountMax 6
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

`.local` 依赖 Bonjour/mDNS，只在同一网段有效；有的公司网络会屏蔽 mDNS，那就用 DHCP 保留的固定 IP 写 HostName。

### 3.2 场景 B：远程（宿主在家里或办公室的 NAT 后面）

**用 Tailscale。** macOS 上有三种发行版本，宿主和 laptop 选法不同：

| 版本 | 装法 | 有 GUI | 能做 Tailscale SSH 服务端 | 登录前/无人登录也运行 | 适合 |
|---|---|---|---|---|---|
| Standalone | pkgs.tailscale.com 下载，或 `brew install --cask tailscale-app` | 是 | 是 | 否（随用户登录启动） | **laptop** |
| App Store | `mas install 1475387142` | 是 | 否 | 否 | 不推荐 |
| 开源 tailscaled | `brew install tailscale` | 否 | 是 | **是**（系统守护进程） | **宿主** |

三者不要混装在同一台机器上；切换前删掉 `Tailscale.app`、清空废纸篓、重启。

宿主（tailscaled 版）：

```bash
brew install tailscale
sudo brew services start tailscale
sudo tailscale up --hostname mac-mini        # 浏览器完成登录
tailscale ip -4                              # 100.x.y.z
tailscale status
```

可选开 Tailscale SSH（之后 laptop 不再需要分发公钥，认证由 Tailscale 身份完成；只支持 22 端口）：

```bash
sudo tailscale set --ssh
```

laptop（Standalone 版）：装好登录同一账号，然后：

```bash
tailscale ping -c 3 mac-mini        # via 192.168.x.x 表示直连（局域网内也会直连）；via DERP 表示走中继
ssh you@mac-mini 'hostname'         # MagicDNS 短名可用
```

iPhone：App Store 装 Tailscale，登录同一账号。

### 3.3 一个别名两种场景通吃（推荐写法）

两台 Mac 都常驻 Tailscale 时，同一局域网内 Tailscale 也会自动走局域网直连，所以最省事的做法是 **HostName 只写 Tailscale 名**：

```
Host mac-mini
    HostName mac-mini          # MagicDNS
    User you
    ServerAliveInterval 30
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

如果你希望局域网内不经过 Tailscale（例如 laptop 不想常开 Tailscale），用 `Match exec` 自动切换，放在 `Host mac-mini` 之前：

```
Match host mac-mini exec "ping -c1 -t1 mac-mini.local >/dev/null 2>&1"
    HostName mac-mini.local

Host mac-mini
    HostName mac-mini
    User you
```

ssh 对同一个键取「第一次出现」的值，所以局域网可达时 `.local` 优先，否则回落到 Tailscale。

### 3.4 备选

- **Cloudflare Tunnel**：宿主跑 `cloudflared`，laptop 用 `ProxyCommand cloudflared access ssh --hostname ssh.you.example`；不需要两端都装客户端，但 mosh 不可用。
- **路由器端口转发 + DDNS**：不推荐；一定要做就改端口、仅密钥、装 fail2ban 类工具（macOS 上可用 `pf` + 日志脚本，复杂度高）。

---

## 4. 阶段三：SSH 密钥（约 5 分钟）

laptop：

```bash
ssh-keygen -t ed25519 -C "laptop"        # 已有则跳过
ssh-copy-id -i ~/.ssh/id_ed25519.pub mac-mini
ssh mac-mini 'id'                         # 免密
```

用 1Password SSH Agent 的：私钥文件不存在，必须 `ssh-copy-id -f -i ~/.ssh/id_ed25519.pub mac-mini`，并在 `~/.ssh/config` 顶部：

```
Host *
    IdentityAgent "~/Library/Group Containers/2BUA8C4S2C.group.com.1password/t/agent.sock"
```

有口令的普通私钥：herdr 的后台 SSH 连接无法弹口令，先 `ssh-add --apple-use-keychain ~/.ssh/id_ed25519`。

开了 Tailscale SSH 的话本节可跳过。

---

### 4.1 两台 Mac 各用什么终端

详见 [终端与客户端选型.md](终端与客户端选型.md)。要点：

| 机器 | 推荐 | 原因 |
|---|---|---|
| laptop 日常客户端 | Ghostty（次选 iTerm2） | 真彩、鼠标、Kitty 图形、通知可聚焦；herdr 的 VT 引擎源自 Ghostty |
| 宿主上跑 herdr server 的启动器 | Terminal.app 登录项 | 内置、随登录起、从 GUI 会话启动 server，画面好不好看无所谓 |
| 专用远程窗口 | Terminal.app 的 osascript 或 Ghostty `open -na Ghostty.app --args -e herdr --remote mac-mini` | 避开嵌套 herdr 和前缀键被吃 |

Ghostty 用户在 laptop 上加一行，连宿主时自动处理 `xterm-ghostty` terminfo：

```
# ~/.config/ghostty/config
shell-integration-features = ssh-env,ssh-terminfo
```

它只包装交互 shell 里的 `ssh`；`herdr --remote` / `machine add` 走系统 ssh，所以再一次性把 terminfo 装到宿主：

```bash
infocmp -x xterm-ghostty | ssh mac-mini -- tic -x -
```

或在 `~/.ssh/config` 的 `Host mac-mini` 下加 `SetEnv TERM=xterm-256color`。

中文输入法用户在 laptop 的 `~/.config/herdr/config.toml` 里考虑打开：

```toml
[experimental]
switch_ascii_input_source_in_prefix = true    # 按 prefix 时临时切英文输入源，避免 prefix+x 被输入法吃掉
reveal_hidden_cursor_for_cjk_ime = true       # 光标位置跟随问题时再开
```

## 5. 阶段四：宿主上的 Agent 与 herdr（约 20 分钟）

### 5.1 装并登录 agent CLI

```bash
brew install mise && echo 'eval "$(mise activate zsh)"' >> ~/.zshrc && source ~/.zshrc
mise use -g node@lts
curl -fsSL https://claude.ai/install.sh | bash     # Claude Code
npm i -g @openai/codex                              # Codex
brew install gh && gh auth login
claude   # 首次登录；无显示器时在 SSH 里做，浏览器 URL 拿到 laptop 打开
codex
```

### 5.2 herdr 配置与集成

```bash
mkdir -p ~/.config/herdr
herdr --default-config > ~/.config/herdr/config.toml
herdr integration install claude
herdr integration install codex
herdr integration status
```

`~/.config/herdr/config.toml` 关键项：

```toml
[server]
headless_cols = 160
headless_rows = 50

[session]
resume_agents_on_restore = true

[ui.toast]
delivery = "system"       # 宿主本机有人看时用；远程主要靠 laptop 客户端的 toast 和手机推送

[terminal]
new_cwd = "follow"
```

### 5.3 让 herdr-server 从 GUI 登录会话启动（硬性前置，决定 Keychain 是否可用）

原因：macOS 上 Claude Code 的 OAuth 凭据、`gh` 的 token 都存在 login Keychain。SSH 会话里启动的进程读写不了 login Keychain（报 `User interaction is not allowed`），会影响 1Password CLI、gh、moshi-hook 配对、部分 agent 的登录态。从 GUI 会话启动的 herdr-server，它的所有 pane 都继承 GUI 权限。

**做法：让 Terminal.app 随登录启动并自动运行 herdr。**

1. Terminal → 设置 → 描述文件 → 选默认描述文件 → Shell → 「启动时运行命令」填 `herdr`，勾选「在 Shell 中运行」。
2. 系统设置 → 通用 → 登录项与扩展 → 「登录时打开」加入 Terminal。
3. 配合 2.4 的自动登录，宿主重启后：自动登录 → Terminal 打开 → `herdr` 拉起 server → 重建 workspace 并恢复 agent 对话。

这条路径走的是 herdr 自己的 spawn（`setsid`），满足 `herdr machine add` 的检查；**不要**改成 launchd 直接托管 `herdr server`，那样 `machine add` 会拒绝保存（错误：`remote server is not ready for saved machines`）。

第一次可以通过屏幕共享连进去手动开 Terminal 跑 `herdr`，或在 SSH 里用：

```bash
osascript -e 'tell application "Terminal" to do script "herdr"'
```

（需要 SSH 会话对 Terminal 的自动化权限，首次会在宿主屏幕弹授权，用屏幕共享点一次即可。）

### 5.4 验证

```bash
ssh mac-mini 'herdr status server'
ssh mac-mini 'herdr workspace list'
ssh mac-mini 'herdr integration status'
ssh mac-mini 'ps -axo pid,args | grep "[h]erdr server"'
```

### 5.5 插件（如有）

`herdr plugin link` 是 server 端状态，不随 dotfiles 走，宿主上要单独执行：

```bash
ssh mac-mini 'herdr plugin link ~/.config/herdr/plugins/claude-on-worktree && herdr plugin list'
```

---

## 6. 阶段五：laptop 上接入宿主（约 10 分钟）

laptop 也要装 herdr：`brew install herdr`。

### 6.1 方式 1（推荐）：把宿主并进本地 herdr

```bash
herdr machine add mac-mini --label "Home Mac mini"
herdr machine add mac-mini --label "Home Mac mini" --remote-session agents   # 接宿主的命名 session
herdr machine list
herdr                          # 侧边栏同时出现本机与 Home Mac mini 的 workspace
```

- `prefix+w` 上下键跨机器切 workspace，Enter 进入；输入和窗口尺寸只发给当前机器。
- 网络中断、合盖、SSH 失败后自动重连，退避最长 2 分钟。
- 侧边栏「Attention」= 需要交互（host key、口令）：`herdr --remote mac-mini` 走一遍提示，再重启客户端。
- 多机器时用 CLI 指定目标：`herdr --machine "Home Mac mini" agent list`。
- 管理：`herdr machine disable|enable|rename|remove <id>`；disable/remove 只断连接，宿主上的 agent 不受影响。

### 6.2 方式 2：专用远程窗口 `herdr --remote`

想全屏只看宿主时用。坑：laptop 的终端若启动即 attach 本地 herdr，`--remote` 会报 `nested herdr is disabled`，即使在 `config.toml` 打开 `[experimental] allow_nested = true`，prefix 也会先被外层吃掉。解决：开一个不 attach 本地 herdr 的新窗口：

```bash
osascript -e 'tell application "Terminal" to do script "herdr --remote mac-mini"'
```

或做成脚本 `~/bin/herdr-remote`，绑到 Raycast/快捷键。

### 6.3 方式 3：socket 转发，让本地 CLI / Raycast 直接操作宿主

`~/bin/herdr-mm`（来自 DevelopersIO 文章的思路，隧道断了自动重建）：

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

Raycast：装 Store 里的 **Herdr** 扩展，偏好设置「Herdr Binary」填 `~/bin/herdr-mm`，「Custom Terminal Launcher」填 6.2 的脚本。之后菜单栏状态、切 workspace、创建 worktree、给 agent 发指令全部作用在宿主上。注意只能指定一个 Binary，扩展会整体变成「远程模式」。

### 6.4 方式 4：原生 macOS 应用 herdrm

```bash
brew install owo-network/brew/herdrm
```

打开后添加设备：本机走 Unix socket，宿主填 SSH 目标（支持密钥、Tailscale SSH、Keychain 密码），侧边栏统一列出两台机器的 workspace/agent/终端，带文件浏览与上传下载。非商业免费。

---

## 7. 阶段六：手机端（可选，约 15 分钟）

### 7.1 客户端

| App | 连接 | 特点 |
|---|---|---|
| Moshi | mosh/SSH | 原生识别 herdr，能按 workspace 打开；配 moshi-hook 收审批卡片 |
| Redock | mosh/SSH | Host/Project/Action 组织，herdr Chat View |
| Blink Shell | mosh/SSH | 键盘和主题好 |
| Termius | SSH | 最通用 |

iPhone 装 Tailscale 登录，App 里主机写 `mac-mini`（MagicDNS）或 `100.x.y.z`，**不要写 `.local`**。连上后运行 `herdr` 即 attach 到宿主 session。用完 `ctrl+b q` detach，否则宿主所有客户端的画面会缩到手机尺寸。

### 7.2 moshi-hook：把审批推到手机上直接点

宿主上：

```bash
brew tap rjyo/moshi && brew trust rjyo/moshi && brew install moshi-hook
```

配对写 Keychain，**不能在纯 SSH 里做**。两种办法：

```bash
# 办法 A：通过 herdr 的 pane（继承 GUI 会话权限）
ssh mac-mini 'herdr pane split w1:p1 --direction down --no-focus'
ssh mac-mini 'herdr pane run w1:p2 "moshi-hook pair --token <Moshi→Settings→Agent Hooks 的 token>"'
ssh mac-mini 'herdr pane read w1:p2 --source visible'

# 办法 B：存文件而不是 Keychain
ssh mac-mini 'moshi-hook pair --token <token> --store file'
```

然后：

```bash
ssh mac-mini 'moshi-hook install'             # 写入 ~/.claude/settings.json、~/.codex/hooks.json 等；会与 herdr 的 hook 合并
ssh mac-mini 'brew services start moshi-hook'
ssh mac-mini 'moshi-hook probe'               # installed/running/gateway 三个 true
```

注意：`moshi-hook install` 会把 `~/.claude/settings.json` 从符号链接改成普通文件，dotfiles 用户要把结果合并回去。`moshi-hook host setup` 的前置检查要求 tmux，用 herdr 的加 `--force`。

---

## 8. 阶段七：通知

| 出口 | 配置 | 覆盖 |
|---|---|---|
| laptop 本地 toast | laptop 的 `config.toml` 里 `[ui.toast] delivery = "system"`，`brew install terminal-notifier` 效果更好 | 通过 `machine add` 接入的远程 agent 一并提示（0.9 的外层 UI 在客户端渲染） |
| 手机审批卡片 | 7.2 moshi-hook | Claude Code、Codex、OpenCode 等 12 种 |
| ntfy 推送 | 见指南一 §9.1，脚本原样放到宿主 | 任意 agent |
| 声音 | `[ui.sound]` 在客户端播放 | 本地 |

---

## 9. 日常工作流示例

1. **早上在家（局域网）**：laptop 开 `herdr`，侧边栏看到 Home Mac mini 下 w1/w2/w3；对红色 blocked 的 agent 按一下 Enter 批准；用 `herdr-mm worktree create --cwd ~/src/app --branch feat-y --no-focus` 再开一个任务。
2. **通勤**：合盖。宿主上的 agent 继续跑。手机 Moshi 收到审批卡片就点掉。
3. **到办公室（远程）**：开盖，herdr 自动通过 Tailscale 重连，画面延续。Raycast 菜单栏显示宿主 agent 状态。
4. **晚上**：`herdr-mm agent list` 看一眼 done 的，attach 进去验收，`git diff` / 跑测试，提交。
5. **每周**：`ssh mac-mini 'herdr update --handoff'`，`brew upgrade` agent CLI，再 `herdr integration status` 确认 hook 未被覆盖。

---

## 10. 运维

```bash
# 远程重启（FileVault 开着时）
ssh mac-mini 'sudo fdesetup authrestart'
# 关着 FileVault 且开了自动登录：
ssh mac-mini 'sudo shutdown -r now'

# 系统更新（大版本建议屏幕共享或到场做）
ssh mac-mini 'sudo softwareupdate -l'

# herdr 健康
ssh mac-mini 'herdr status server && herdr agent list'
ssh mac-mini 'HERDR_LOG=herdr=debug herdr status'
ssh mac-mini 'ls ~/.config/herdr/'          # session.json、session-backups/、日志

# 断网后 laptop 侧
herdr machine list                           # 看连接状态
herdr --remote mac-mini                      # 处理 Attention
```

---

## 10.5 闭环补遗（必读）

按一天的实际使用推演后补的缺口，都在 [闭环补遗-实际使用中的缺口与修补.md](闭环补遗-实际使用中的缺口与修补.md)。Mac 场景特别注意：离开宿主前必须到场完成的清单（I）、macOS TCC 对 Documents/Desktop/Downloads 的限制与完全磁盘访问（J）、Claude Code 与 gh 凭据在 Keychain 因而 server 必须从 GUI 会话启动（K）、屏幕共享 `open vnc://mac-mini`（L）、关闭 macOS 自动更新重启（M）；通用项 C/D/E/F/H 同样适用。

## 11. 验收清单

- [ ] 局域网：`ssh mac-mini` 免密进；远程（关掉 Wi‑Fi 用手机热点）：同一条命令仍能进，`tailscale ping` 显示直连或 DERP。
- [ ] 宿主 `pmset -g` 显示 sleep 0、womp 1；拔显示器一小时后仍能 ssh。
- [ ] 宿主重启后，自动登录 → Terminal 自启 → `ssh mac-mini 'herdr status server'` 正常，workspace 已重建，Claude 对话已恢复。
- [ ] 在 herdr pane 里跑 `security find-generic-password -s test 2>&1 | head -1` 不报 `User interaction is not allowed`（说明 server 在 GUI 会话）。
- [ ] laptop `herdr machine add mac-mini` 成功，侧边栏出现宿主 workspace，合盖再开盖自动重连。
- [ ] `herdr-mm agent list` 能列出宿主 agent；Raycast 菜单栏显示状态。
- [ ] 手机 Moshi 能 attach；触发一次 Claude 审批，手机收到卡片并能批准。
- [ ] `sudo fdesetup authrestart` 后无需到场即恢复。

---

## 12. 故障排查

| 现象 | 原因 | 处理 |
|---|---|---|
| 半夜 ssh 不通，早上又好了 | 宿主睡了 | 2.3；`pmset -g log | grep -i sleep` 查原因 |
| 重启后卡在解锁界面 | FileVault + 无自动登录 | 关 FileVault 或每次 `fdesetup authrestart` |
| `herdr machine add` 报 server not ready | server 被 launchd 托管，或宿主还没起 server | 5.3；先在宿主 GUI 起一次 `herdr` |
| pane 里 `gh` / `op` / moshi-hook 报 Keychain 错误 | server 从 SSH 会话启动 | 5.3 重新从 GUI 启动 server（`herdr server stop` 后在 Terminal 里 `herdr`） |
| `--remote` 报 nested | laptop 终端自动 attach 了本地 herdr | 6.2 专用窗口 |
| prefix 键不生效（远程窗口） | 外层本地 herdr 先吃掉 | 同上，别在本地 herdr 里再开远程 |
| 手机连上后 laptop 画面变矮 | 多客户端取最小尺寸 | 手机 detach |
| `.local` 解析失败 | 不在同一网段 / mDNS 被屏蔽 | 用 Tailscale 名或固定 IP |
| Tailscale SSH 起不来 | 装的是 GUI 版 | 宿主换 `brew install tailscale`（tailscaled），先卸载 GUI 版 |
| 两个 Tailscale 打架 | 同机装了 GUI + tailscaled | 只留一个，删 app、清废纸篓、重启 |
| 插件不触发 | 未在宿主 `plugin link` | 5.5 |
| agent 状态 unknown | pane 里套 tmux / 包装脚本 | 让 agent 直接是前台进程；`herdr agent explain` |
| 升级 herdr 后 agent 全没了 | 没用 `--handoff` | 下次 `herdr update --handoff`；对话可由 resume 恢复 |
