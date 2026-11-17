# 实施指南一：在 Linux 服务器上部署 herdr 管理多个 AI 编程 Agent

> 配套阅读：[herdr-总结与部署指南.md](herdr-总结与部署指南.md)（herdr 本体的概念、命令和限制）。
> 本文只讲「怎么把整套东西搭起来并用起来」，按阶段给出可直接执行的命令。整理日期 2026-09-22，基于 herdr v0.9.x、Tailscale 1.9x、Claude Code 2.1.x、Codex CLI。
> 约定：服务器主机名 `box`，运行 agent 的 Linux 用户 `dev`，你的笔记本叫 `laptop`。

---

## 0. 目标架构与工具选型

```
 手机 (Termius / Blink / Moshi / Redock)          笔记本 (herdr TUI / ssh)
        │  Tailscale (WireGuard)                          │  Tailscale
        └──────────────┐                 ┌────────────────┘
                       ▼                 ▼
                 ┌──────────── Linux 主机 box ────────────┐
                 │ sshd (仅密钥)  ← mosh-server (UDP)     │
                 │   └─ herdr-server (dev 用户, 常驻)     │
                 │        ├─ w1 repo-a  [claude] [codex]  │
                 │        ├─ w2 repo-b  [claude] [tests]  │
                 │        └─ w3 worktree-x [opencode]     │
                 │ 通知出口: herdr toast / ntfy / moshi-hook│
                 └────────────────────────────────────────┘
```

| 层 | 工具 | 必要性 | 作用 | 备选 |
|---|---|---|---|---|
| 运行时 | **herdr** | 必选 | 拥有 agent 终端，状态汇总，detach 不死 | tmux（无 agent 状态） |
| Agent | **Claude Code / Codex / OpenCode …** | 必选 | 真正干活的 CLI | 任意 22 种已识别 agent |
| 网络 | **Tailscale** | 强烈推荐 | 不开端口、不暴露公网、MagicDNS、可选 Tailscale SSH | WireGuard 手搓 / Cloudflare Tunnel / 公网 22 + 加固 |
| 远程 shell | **OpenSSH** | 必选 | 认证与通道 | Tailscale SSH（替代密钥分发） |
| 弱网 | **mosh** | 推荐 | 手机 / 地铁 / 换网不断 | 直接 ssh |
| 版本管理 | **git worktree** | 必选 | 一任务一目录，agent 不互踩 | 多 clone |
| 通知 | **ntfy** | 推荐 | agent blocked/done 推到手机 | moshi-hook / Telegram bot |
| 手机端 | **Termius / Blink Shell / Moshi / Redock** | 可选 | 手机上 attach herdr | 任意 SSH 客户端 |
| 隔离 | **专用用户 + worktree**；可选 Docker sandbox | 推荐 | 限制 agent 权限与影响面 | devcontainer |
| 运行时工具 | **mise**（Node/Python 版本）、jq、gh | 推荐 | agent CLI 依赖 Node；脚本要 jq | nvm / fnm |
| 主机守护 | **systemd-logind linger** | 必选 | 退出 SSH 后用户进程不被杀 | `KillUserProcesses=no` |
| 安全 | **ufw + fail2ban** | 公网暴露时必选 | 端口白名单、爆破封禁 | 云厂商安全组 |

---

## 1. 前置检查清单

- [ ] 一台长期开机的 Linux（Ubuntu 22.04/24.04、Debian 12、Fedora、RHEL 系均可），x86_64 或 aarch64，2 核 4 GB 起步；agent 多时按每个 agent 约 0.5–1 GB 内存估算（Node 进程）。
- [ ] 有 sudo 权限的初始账号。
- [ ] 笔记本能 `ssh` 到它（公网 IP、内网或云厂商控制台）。
- [ ] 各 agent 的账号/API Key 已准备好（Claude、OpenAI 等）。
- [ ] 手机上可装 Tailscale 和一个 SSH 客户端（可选）。

---

## 2. 阶段一：主机基础（约 15 分钟）

### 2.1 建专用用户并给 sudo（可选但推荐）

```bash
sudo adduser dev
sudo usermod -aG sudo dev
sudo -iu dev
```

所有后续步骤都以 `dev` 身份执行。agent 以这个用户的权限跑，凭据只给它够用的。

### 2.2 语言、时区、基础包

```bash
sudo apt update && sudo apt install -y git jq curl unzip build-essential mosh locales
sudo locale-gen en_US.UTF-8 zh_CN.UTF-8
echo 'export LANG=en_US.UTF-8' >> ~/.bashrc      # mosh 和 TUI 都要求 UTF-8
sudo timedatectl set-timezone Asia/Shanghai
```

### 2.3 退出 SSH 后保住用户进程（关键）

systemd 默认在用户所有会话退出后终止 `user@.service`，某些发行版还会开 `KillUserProcesses=yes`。开启 linger 后用户管理器在开机时启动、注销后也保留：

```bash
sudo loginctl enable-linger dev
loginctl show-user dev | grep Linger        # Linger=yes
grep -i KillUserProcesses /etc/systemd/logind.conf   # 若为 yes，改成 no 并 sudo systemctl restart systemd-logind
```

### 2.4 sshd 加固（无论是否走 Tailscale 都建议做）

先把笔记本公钥放上去（见 4.1），确认密钥登录成功后再改：

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
ClientAliveInterval 30
ClientAliveCountMax 6
AllowUsers dev
EOF
sudo sshd -t && sudo systemctl restart ssh
```

### 2.5 防火墙与爆破封禁（有公网 IP 时必做）

```bash
sudo apt install -y ufw fail2ban
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0            # 只从 Tailscale 网卡放行一切
# 如果暂时还要公网 SSH：
# sudo ufw limit 22/tcp
# sudo ufw allow 60000:61000/udp           # 仅当要在公网用 mosh
sudo ufw enable
sudo systemctl enable --now fail2ban
```

目标状态：Tailscale 打通后，把公网 22 从 ufw 里删掉，只留 `tailscale0`。

---

## 3. 阶段二：网络接入（约 10 分钟）

### 3.1 Tailscale（推荐路径）

服务器：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --hostname box          # 浏览器完成登录
tailscale ip -4                           # 100.x.y.z
tailscale status
```

笔记本 / 手机：装 Tailscale 客户端，登录同一账号。MagicDNS 在 2022-10 之后创建的 tailnet 默认开启，此后 `ssh dev@box` 直接可用（短名自动补全为 `box.<tailnet>.ts.net`）。

验证是直连还是中继（中继会明显变慢）：

```bash
tailscale ping -c 3 box       # "via 1.2.3.4:41641" 直连；"via DERP(xxx)" 走中继
```

### 3.2 可选：Tailscale SSH（省掉密钥分发）

```bash
sudo tailscale set --ssh
```

- 服务端仅支持 Linux（和 macOS 的开源 tailscaled 版本）。
- 认证由 Tailscale 身份完成，OpenSSH 密钥不再需要；`ssh dev@box` 直接进。
- 需在 tailnet 的 ACL 里有 `ssh` 规则；个人 tailnet 默认策略已允许自己连自己的机器。
- 注意：升级/重启 `tailscaled` 会断掉所有 Tailscale SSH 会话（herdr-server 不受影响，重新 attach 即可）。
- 只占 22 端口，不可改端口；且只能从 tailnet 内连。

### 3.3 备选路径

| 方案 | 何时用 | 要点 |
|---|---|---|
| 公网 22 + 加固 | 云主机、不想装 VPN | 必做 2.4、2.5；改端口意义不大，重点是仅密钥 + fail2ban |
| WireGuard 自建 | 不想依赖 Tailscale 控制面 | 自己管密钥和端点，NAT 穿透要自己处理 |
| Cloudflare Tunnel | 已在用 Cloudflare Zero Trust | 服务器 `cloudflared tunnel`，客户端 `ProxyCommand cloudflared access ssh --hostname ssh.example.com`；TCP 走 HTTPS 隧道，mosh 不可用 |

---

## 4. 阶段三：客户端 SSH 配置（约 5 分钟）

### 4.1 笔记本上

```bash
ssh-keygen -t ed25519 -C "laptop"          # 已有则跳过
ssh-copy-id -i ~/.ssh/id_ed25519.pub dev@box
```

用 1Password SSH agent 的话，私钥文件不存在，`ssh-copy-id` 要加 `-f -i ~/.ssh/id_ed25519.pub`，并在 `~/.ssh/config` 顶部加：

```
Host *
    IdentityAgent "~/Library/Group Containers/2BUA8C4S2C.group.com.1password/t/agent.sock"
```

`~/.ssh/config` 增加：

```
Host box
    HostName box                # MagicDNS 名；未开 MagicDNS 则写 100.x.y.z
    User dev
    ServerAliveInterval 30
    ServerAliveCountMax 6
    ControlMaster auto          # 复用连接，herdr machine add / 脚本会频繁开 ssh
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

验证：

```bash
ssh box 'hostname && id'
```

### 4.2 笔记本终端模拟器（连服务器前做一次）

详见 [终端与客户端选型.md](终端与客户端选型.md)。要点：

- macOS 客户端首选 Ghostty，次选 iTerm2；Windows 客户端首选 WezTerm；不要用 Warp、日常不要用 Terminal.app（无真彩）。
- Ghostty / kitty / Alacritty 的自定义 TERM 在服务器上缺 terminfo 会报 `missing or unsuitable terminal`。二选一：

```bash
infocmp -x xterm-ghostty | ssh box -- tic -x -      # 把 terminfo 装到服务器（kitty 换 xterm-kitty，Alacritty 换 alacritty）
```

或在 `~/.ssh/config` 的 `Host box` 下加 `SetEnv TERM=xterm-256color`。`herdr --remote` 与 `herdr machine add` 走系统 ssh，这两种方式都对它们生效。

- VS Code / Cursor 内置终端里跑 herdr 要先解决 `ctrl+b` 被编辑器截获的问题（改 herdr 前缀或调整 `terminal.integrated.commandsToSkipShell`）。

### 4.3 手机上

| 客户端 | 平台 | 特点 |
|---|---|---|
| Termius | iOS/Android | 通用 SSH，简单稳定 |
| Blink Shell | iOS | 支持 mosh，键盘体验好 |
| Moshi | iOS/Android | mosh + 原生识别 herdr/tmux，配 moshi-hook 可在手机上点审批 |
| Redock | iOS/Android | SSH/Mosh，按 Host/Project/Action 组织，有 herdr Chat View |

手机上装 Tailscale 登录后，用 `100.x.y.z` 或 MagicDNS 名加主机即可。

---

## 5. 阶段四：工具链与 Agent（约 15 分钟）

### 5.1 用 mise 统一管理运行时

```bash
curl https://mise.run | sh
echo 'eval "$(~/.local/bin/mise activate bash)"' >> ~/.bashrc && source ~/.bashrc
mise use -g node@lts python@3.12
node -v
```

### 5.2 安装并登录 agent CLI

```bash
# Claude Code（官方原生安装器）
curl -fsSL https://claude.ai/install.sh | bash
claude --version
claude            # 首次登录（浏览器 OAuth 或 API Key）；服务器无浏览器时用 URL 在笔记本上完成

# Codex
npm i -g @openai/codex
codex --version && codex

# 其他：opencode / gemini / pi 按各自文档
# GitHub CLI（agent 开 PR 用）
sudo apt install -y gh && gh auth login
```

无头登录技巧：Claude Code 的 OAuth 会打印 URL，在笔记本浏览器打开完成后把回调码贴回终端；Codex 用 `codex login --device-auth`。API key 与 PATH 统一写进 `~/.config/agents.env`，并在 `~/.bashrc` 的交互判断之前和 `~/.profile` 里 source 它，这样 herdr pane 里的 shell 和 `ssh box '…'` 非交互命令都能拿到。注意 `~/.config/environment.d/` 只对 systemd 拉起的进程生效，herdr-server 不是，所以不要指望它。完整做法见 [闭环补遗 §B](闭环补遗-实际使用中的缺口与修补.md)。

### 5.3 仓库与 worktree 目录规范

```bash
mkdir -p ~/src ~/wt
git clone git@github.com:you/app.git ~/src/app
```

约定：`~/src/<repo>` 是主 checkout，`~/wt/<repo>/<branch>` 是每个任务的 worktree（herdr 自己的默认位置是 `~/.herdr/worktrees/`，两者选一个坚持用）。

---

## 6. 阶段五：安装并初始化 herdr（约 10 分钟）

### 6.1 安装

```bash
curl -fsSL https://herdr.dev/install.sh | sh
herdr --version
```

### 6.2 配置文件

```bash
mkdir -p ~/.config/herdr
herdr --default-config > ~/.config/herdr/config.toml    # 先拿一份默认配置再改
```

在 `~/.config/herdr/config.toml` 里确认或修改这些键：

```toml
[server]
headless_cols = 160       # 没有客户端接入时 pane 的虚拟尺寸；纯 CLI 驱动 agent 时很重要
headless_rows = 50

[session]
resume_agents_on_restore = true   # server 重启后用 claude --resume / codex resume 拉回对话
# pane_history = true             # 想在重启后看到最近屏幕再开；可能把密钥写进 session-history.json

[ui.toast]
delivery = "terminal"     # 通过外层终端通知，SSH 场景可用；本地桌面用 "system"

[terminal]
new_cwd = "follow"        # 新 pane 跟随当前 pane 的目录
```

### 6.3 安装 agent 集成（状态更准 + 重启可恢复）

```bash
herdr integration install claude     # 写 ~/.claude/hooks/herdr-agent-state.sh 并改 ~/.claude/settings.json
herdr integration install codex      # 写 ~/.codex/hooks.json
herdr integration status
```

### 6.4 第一次启动与验证

```bash
cd ~/src/app
herdr                     # 自动拉起后台 server，建默认 workspace
# 在 pane 里敲 claude，侧边栏出现 working/idle 即成功
# ctrl+b q detach
herdr status
herdr status server
herdr workspace list
ls ~/.config/herdr/       # 应有 herdr.sock、session.json
```

退出 SSH 再进来，`herdr` 应当回到原样，agent 进程仍在（`ps -u dev -o pid,cmd | grep -E 'claude|codex'`）。

### 6.5 关于开机自启（读完再决定）

- 主机重启后 agent 进程本来就活不了，所以「开机自启 server」的收益只是省一次 `herdr`。
- **不要**写 systemd 单元直接 `ExecStart=herdr server`：systemd 托管的进程不是会话组长，herdr 会把它标为 `detached_server_daemon=false`，后续 `herdr machine add` 会拒绝把这台机器存为远程机器。
- 真要自启，用 cron 让 herdr 走自己的 spawn 路径：

```bash
crontab -e
# @reboot sleep 15 && /home/dev/.local/bin/herdr status server >/dev/null 2>&1 || /usr/bin/setsid -f /home/dev/.local/bin/herdr server
```

更稳妥的做法是让客户端的 `herdr machine add box` 或 `herdr --remote box` 在需要时自动拉起 server。

---

## 7. 阶段六：从客户端接入（约 5 分钟）

### 7.1 笔记本：三选一或并用

```bash
# A. 最简单：SSH 进去 attach
ssh box -t herdr

# B. 本地 herdr 一键远程 attach（本地也要装 herdr）
herdr --remote box
herdr --remote ssh://dev@box:22 --session agents

# C. 0.9 多机器：把 box 并入本地 herdr 的侧边栏
herdr machine add box --label "VPS"
herdr machine add box --label "VPS" --remote-session agents   # 接命名 session
herdr machine list
```

C 方式下本地 `herdr` 同时显示本机和 box 的 workspace，`prefix+w` 上下键切换；侧边栏出现「Attention」说明要走一次交互（host key / 密钥口令），跑 `herdr --remote box` 处理后重启客户端。私钥有口令的先 `ssh-add`。

### 7.2 弱网 / 手机：mosh 包一层

```bash
mosh box -- herdr           # 本地也要 brew install mosh
```

mosh 需要 UDP 60000–61000；走 Tailscale 时 UDP 直接通，无需开公网端口。mosh 只同步可见画面、没有滚动历史，但 herdr 自己有 scrollback，所以组合正好互补。**不要**在 herdr pane 里再开 tmux。

### 7.3 纯 CLI 驱动（脚本、CI、另一个 agent）

herdr CLI 走 socket、不需要 TTY，所以：

```bash
ssh box 'herdr agent list --json'
ssh box 'herdr workspace list'
```

或把远端 socket 转发到本地，让本地工具直接操作：

```bash
ssh -fnNT -L /tmp/herdr-box.sock:/home/dev/.config/herdr/herdr.sock box
HERDR_SOCKET_PATH=/tmp/herdr-box.sock herdr agent list
```

---

## 8. 阶段七：把工作流固化成脚本（约 30 分钟）

在服务器上建 `~/bin/`，加入 PATH。

### 8.1 `new-task`：一条命令开 worktree + workspace + agent

```bash
#!/usr/bin/env bash
# 用法: new-task <repo> <branch> [agent-kind] [提示词]
set -euo pipefail
repo=$1; branch=$2; kind=${3:-claude}; prompt=${4:-}
src=~/src/$repo

created=$(herdr worktree create --cwd "$src" --branch "$branch" --no-focus)
pane=$(printf '%s\n' "$created" | jq -r '.result.root_pane.pane_id')
name=$(echo "$branch" | tr -c 'a-z0-9_-\n' '-' | cut -c1-30)

herdr agent start "$name" --kind "$kind" --pane "$pane"
[ -n "$prompt" ] && herdr agent prompt "$name" "$prompt"
echo "started $kind as '$name' in $pane ($branch)"
```

> `herdr worktree create` 的返回 JSON 字段以 `herdr api schema --json` 为准；若 `root_pane` 路径不同，用 `herdr workspace list --json` 查 pane ID。

### 8.2 `review-pipeline`：编码 agent 完成后自动起评审 agent

```bash
#!/usr/bin/env bash
set -euo pipefail
coder=$1                      # 已存在的 agent 名
herdr agent wait "$coder" --until done --timeout 300000 || true

ws=$(herdr agent list --json | jq -r --arg n "$coder" '.result.agents[] | select(.name==$n) | .workspace_id')
split=$(herdr pane split "$ws:p1" --direction right --no-focus)
rp=$(printf '%s\n' "$split" | jq -r '.result.pane.pane_id')

herdr agent start "${coder}-review" --kind codex --pane "$rp"
herdr agent prompt "${coder}-review" "Review the uncommitted diff in this worktree. List blocking issues first." --wait --timeout 300000
herdr agent read "${coder}-review" --source recent-unwrapped --lines 200
```

### 8.3 命名 session 按团队 / 项目隔离

```bash
herdr --session teamA
HERDR_SESSION=teamA herdr agent list
herdr session list
```

socket 在 `~/.config/herdr/sessions/teamA/herdr.sock`。客户端用 `herdr machine add box --remote-session teamA` 或 `herdr --remote box --session teamA`。

### 8.4 编排 agent（agent 管 agent）

```bash
npx skills add herdrdev/herdr --skill herdr -g     # 给 Claude Code 装 herdr 技能
```

在一个 workspace 里起一个「牧羊人」Claude Code，给它的 CLAUDE.md 写清：只允许通过 `herdr agent start/prompt/wait/read` 调度其他 agent；每个子任务先 `herdr worktree create`；子 agent `blocked` 时汇报而不是替人批。技能自带「先确认 `HERDR_ENV=1`」的防护。

---

## 9. 阶段八：通知到手机（约 15 分钟）

### 9.1 ntfy（最通用）

手机装 ntfy 应用，订阅一个难猜的 topic，例如 `box-agents-8f3k2`。

**方式 A：Claude Code 自己的 Notification hook（最准确，只覆盖 Claude Code）**

`~/.claude/hooks/ntfy-notify.sh`：

```bash
#!/usr/bin/env bash
payload=$(cat)
type=$(echo "$payload" | jq -r '.notification_type // "notification"')
msg=$(echo "$payload"  | jq -r '.message // ""')
cwd=$(echo "$payload"  | jq -r '.cwd // ""' | xargs basename)
case "$type" in
  permission_prompt|agent_needs_input|elicitation_dialog) prio=high; tag=rotating_light ;;
  agent_completed|idle_prompt)                           prio=default; tag=white_check_mark ;;
  *)                                                     prio=low; tag=information_source ;;
esac
curl -s -H "Title: claude@$cwd: $type" -H "Priority: $prio" -H "Tags: $tag" \
     -d "$msg" https://ntfy.sh/box-agents-8f3k2 >/dev/null
```

`~/.claude/settings.json` 里合并（herdr 的 hook 已在里面，注意不要覆盖）：

```json
{
  "hooks": {
    "Notification": [
      { "matcher": "permission_prompt|agent_needs_input|agent_completed|idle_prompt",
        "hooks": [ { "type": "command", "command": "~/.claude/hooks/ntfy-notify.sh", "async": true } ] }
    ]
  }
}
```

`chmod +x ~/.claude/hooks/ntfy-notify.sh`。Codex 有类似的 `~/.codex/hooks.json`，按其文档加同样的脚本。

**方式 B：herdr 侧统一监听（覆盖所有 agent 种类）**

`~/bin/herdr-watch`（用 `agent wait` 阻塞式等待，每个 agent 一个后台循环）：

```bash
#!/usr/bin/env bash
# 对每个 agent 循环等待 blocked/done，变化时推 ntfy
topic=https://ntfy.sh/box-agents-8f3k2
watch_one() {
  local a=$1
  while true; do
    if herdr agent wait "$a" --until blocked --timeout 300000 >/dev/null 2>&1; then
      curl -s -H "Title: $a blocked" -H "Priority: high" -H "Tags: rotating_light" \
           -d "$(herdr agent read "$a" --source visible --lines 8 2>/dev/null | tail -8)" "$topic" >/dev/null
      herdr agent wait "$a" --until working --timeout 300000 >/dev/null 2>&1 || true
    fi
  done
}
for a in $(herdr agent list --json | jq -r '.result.agents[].name'); do
  watch_one "$a" &
done
wait
```

把它丢进 herdr 自己的一个 pane 里跑（`herdr pane run w1:p9 "~/bin/herdr-watch"`），这样它随 server 一起活。JSON 字段名以 `herdr agent list --json` 实际输出为准。

### 9.2 moshi-hook（Linux 也支持，配 Moshi 手机端可直接在手机上批审批）

```bash
curl -fsSL https://getmoshi.app/install.sh | sh
moshi-hook pair --token <Moshi 应用 → Settings → Agent Hooks 里的 token>
moshi-hook install               # 写入 Claude Code / Codex / OpenCode 等的 hook
moshi-hook serve                 # 或 systemctl --user enable --now moshi-hook.service
moshi-hook status
```

只转发 approval_required / task_complete / session_started / tool_* 五类事件的摘要，完整对话不经过 Moshi 服务器。

---

## 10. 阶段九：隔离与安全

| 措施 | 做法 |
|---|---|
| 专用用户 | agent 全部在 `dev` 下跑；生产凭据不放这台机器 |
| 一任务一 worktree | `herdr worktree create`；绝不让两个 agent 共享同一 checkout |
| 只从 Tailscale 进 | ufw 只放行 `tailscale0`；公网 22 关掉 |
| 密钥最小化 | `gh auth login` 用细粒度 token；API Key 放 `~/.config/environment.d/`，权限 600 |
| 插件审查 | `herdr plugin install` 前读 `herdr-plugin.toml` 和脚本；插件以你的权限运行 |
| 更强隔离（可选） | 每个 agent 一个 Docker/devcontainer，只挂对应 worktree，网络白名单；在容器内才考虑 `claude --dangerously-skip-permissions` |
| 审批不外包给 agent | 编排 agent 只汇报 blocked，不替人按 y |

---

## 11. 阶段十：运维

```bash
# 升级（尽量不杀 agent 进程）
herdr update --handoff
herdr status server

# 主机重启后
herdr                       # 重建布局；resume_agents_on_restore 会拉回 Claude/Codex 对话
herdr integration status    # 确认 hook 仍在（agent CLI 升级可能覆盖 settings）

# 日志与排错
HERDR_LOG=herdr=debug herdr
ls ~/.config/herdr/                    # herdr.sock session.json session-backups/ *.log
herdr agent explain w1:p1 --json       # 状态判断依据
herdr server update-agent-manifests    # 拉最新的检测规则

# 备份会话结构（可进 cron）
cp ~/.config/herdr/session.json ~/backup/herdr-session-$(date +%F).json

# 停掉一切（慎用，所有 agent 一起退出）
herdr server stop
```

健康巡检脚本（cron 每 5 分钟）：

```bash
#!/usr/bin/env bash
herdr status server >/dev/null 2>&1 || curl -s -H "Priority: urgent" -d "herdr-server on box is down" https://ntfy.sh/box-agents-8f3k2
```

---

## 11.5 闭环补遗（必读）

按一天的实际使用推演后补的缺口，都在 [闭环补遗-实际使用中的缺口与修补.md](闭环补遗-实际使用中的缺口与修补.md)：Codex 无头登录与配额预算（A）、非交互 ssh 的 PATH 与环境变量（B）、worktree 的 .env / 依赖 / 端口（C）、减少 blocked 的权限策略与仓库规则（D）、pane 内 git push 凭据与提交身份（E）、任务收尾脚本（F）、unattended-upgrades 自动重启与 Tailscale 密钥过期（G）、磁盘与日志（H）。

## 12. 验收清单

- [ ] `ssh box` 免密秒进；`tailscale ping box` 显示直连。
- [ ] `herdr` 起来后在 pane 里跑 `claude`，侧边栏显示状态；`ctrl+b q` 后 `ps` 仍能看到 claude 进程。
- [ ] 断开 SSH、重连、`herdr`，pane 与 agent 原样。
- [ ] `herdr integration status` 显示 claude/codex current。
- [ ] 笔记本 `herdr machine add box` 成功，侧边栏出现 VPS 的 workspace。
- [ ] 手机 Tailscale + SSH 客户端能 attach；mosh 可用。
- [ ] 触发一次 Claude Code 审批，手机收到 ntfy / Moshi 推送。
- [ ] `sudo reboot` 后 `herdr` 能重建 workspace 并恢复对话。
- [ ] 公网 22 已关闭，`ufw status` 只放行 tailscale0。

---

## 13. 故障排查

| 现象 | 原因 | 处理 |
|---|---|---|
| 退出 SSH 后 agent 全没了 | 没开 linger 或 `KillUserProcesses=yes` | 2.3 |
| `herdr machine add` 报 server not ready | server 被 systemd 托管 | 6.5，改成让 herdr 自己 spawn |
| agent 一直 `unknown` | pane 里套了 tmux / 被包装脚本挡住 | 让 agent 直接是 pane 前台进程；`herdr agent explain` |
| 手机 attach 后画面变窄 | 多客户端取最小尺寸 | 手机用完 detach |
| mosh 连不上 | UDP 60000–61000 不通或 locale 非 UTF-8 | 走 Tailscale；`locale` 检查 |
| `claude: command not found`（脚本里） | 非登录 shell 没有 mise / PATH | 脚本开头 `source ~/.bashrc` 或用绝对路径 |
| 重启后对话没恢复 | 集成没装 / agent 版本过低 | `herdr integration install claude` 后再试 |
| 通知没到 | hook 未合并进 settings.json / 脚本无执行权限 | `claude /hooks` 查看；`chmod +x` |
| Tailscale 走 DERP 很慢 | NAT 类型严格 | 路由器开 UPnP 或放行 41641/udp；实在不行接受中继 |
