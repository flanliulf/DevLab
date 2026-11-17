# 浏览器与 Web 服务访问：在日常 Mac（笔记本）上用本地浏览器访问宿主 Mac 上的 dev server、Web UI 和 Agent 浏览器（不用远程桌面）

> 场景：宿主 Mac（Apple silicon，macOS 26）上用 herdr 跑很多 Claude Code / Codex agent，每个任务一个 git worktree，每个 worktree 一个 dev server 端口（如 3000–3999）。笔记本经 Tailscale（MagicDNS 名 `mac-mini`）或同一局域网连接宿主。
> 调研日期：2026-09-24。npm registry 当日最新版本（用 `registry.npmjs.org/<pkg>/latest` 查到）：vite 8.3.0、next 16.3.6、portless 0.15.6、chrome-devtools-mcp 1.10.1、@playwright/mcp 0.0.82 — [npm registry](https://registry.npmjs.org/vite/latest)
> 记法：「Cited Findings」下的每一条都附来源；「Inferences」是我根据引用材料推出来的结论，还没有逐条实测。

---

## Q1. 直接访问 `http://mac-mini:PORT` / `http://100.x.y.z:PORT`（Tailscale 或 LAN）

### Takeaway
这是最省事的办法，但有两个前提：dev server 必须监听非 loopback 地址，页面也会变成非安全上下文（非 localhost 的 HTTP）。另外，Vite/Next.js 在 2025–2026 年都加了 Host/Origin 检查，用 `mac-mini` 这类主机名访问时，默认会被拦下或导致 HMR 失效。在 macOS 15+ 上，访问 LAN 地址还会受「本地网络」隐私权限影响，Tailscale 的 100.x 地址不受这个限制。

### Cited Findings
- **Vite 默认只监听 `localhost`**：`server.host` 默认是 `'localhost'`，要监听 LAN/公网地址得设成 `0.0.0.0` 或 `true`，或者加 CLI 参数 `--host`。— [Vite server options](https://vite.dev/config/server-options)
- **Vite `server.allowedHosts`（Host 头白名单）**：默认值 `[]`，但 "`localhost` and domains under `.localhost` and all IP addresses are allowed by default"；写成 `.example.com` 表示允许该域名及其所有子域；"When using HTTPS, this check is skipped"（只在 Vite 自己跑 HTTPS 时跳过）。— [Vite docs 源文件 (main)](https://github.com/vitejs/vite/blob/main/docs/config/server-options.md)
- **不要设 `allowedHosts: true`**：官方原话是这样会 "allows any website to send requests to your dev server through DNS rebinding attacks, allowing them to download your source code"，并建议始终用显式列表（见 GHSA-vg6x-rcgg-rjx6）。也不要把 `.com` 这种顶级域加进去，只加解析结果由你控制的域名。— [Vite docs 源文件](https://github.com/vitejs/vite/blob/main/docs/config/server-options.md)
- 这个 Host 检查是从 Vite 6.0.9 左右开始的，社区里有大量 "Blocked request. This host (...) is not allowed" 报错。— [vite issue #19411](https://github.com/vitejs/vite/issues/19411)、[vite discussion #19426](https://github.com/vitejs/vite/discussions/19426)
- **Vite `server.cors` 默认值**只放行 `localhost`、`127.0.0.1`、`[::1]` 这几个 origin（正则 `^https?://(?:(?:[^:]+\.)?localhost|127\.0\.0\.1|\[::1\])(?::\d+)?$`）；改成 `true` 同样有 DNS rebinding 风险。— [Vite docs 源文件](https://github.com/vitejs/vite/blob/main/docs/config/server-options.md)
- **Next.js `allowedDevOrigins`**："Next.js blocks cross-origin requests to dev-only assets and endpoints during development by default"。默认只放行 `localhost` 及其子域，以及 dev server 初始化时使用的 hostname。通配符 `*` 匹配一个 label，`**` 匹配一个或多个 label（只能放在开头），不支持 `team-*.x` 这种部分通配。只比对 `Origin` 头里的 hostname，端口和 scheme 都忽略；no-cors 请求（例如 `<script>`）改用 `Referer` 的 hostname。示例：`allowedDevOrigins: ['local-origin.dev', '*.local-origin.dev']`。— [Next.js docs 源文件](https://github.com/vercel/next.js/blob/canary/docs/01-app/03-api-reference/05-config/01-next-config-js/allowedDevOrigins.mdx)（对应 [nextjs.org 页面](https://nextjs.org/docs/app/api-reference/config/next-config-js/allowedDevOrigins)）
- **Next.js 16 默认直接拦截**：通过 LAN IP 访问时 `/_next/webpack-hmr` 会被拒，HMR 失效，要把 IP 加进 `allowedDevOrigins`（例如 `['192.168.1.9']`）。— [Pramuditya 博客](https://www.pramuditya.com/en/blog/nextjs-16-hmr-breaks-over-lan-ip-alloweddevorigins)。有 PR 记录日志文案是 "Blocked cross-origin request to Next.js dev resource /_next/hmr from [origin]"，还指出 `127.0.0.1` 和 `localhost` 算不同 origin — [Noye PR #16](https://github.com/Kouen-Park/Noye/pull/16)（只看了搜索摘要）。15.2.x 起就有相关问题：[next.js discussion #77429](https://github.com/vercel/next.js/discussions/77429)
- Next.js 16 起 `next dev` 默认用 Turbopack。— [Next.js v16 升级指南源文件](https://github.com/vercel/next.js/blob/canary/docs/01-app/02-guides/upgrading/version-16.mdx)
- **安全上下文（secure context）**：只有 `https`/`wss`/`file`、`127.0.0.0/8`、`::1`、`localhost`、`*.localhost` 算 potentially trustworthy；`http://example.com` 这类普通 HTTP 主机不算。Service Worker 等特性要求 `window.isSecureContext` 为真。— [MDN Secure Contexts](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts)
- **macOS 15 (Sequoia) 起的 Local Network Privacy**：被拒的进程连同网段 LAN 对端时拿到的是 `EHOSTUNREACH`（"No route to host"），不会有任何「权限」字样的提示。TCC 按 responsible process 归属权限：tmux server 里启动的 node/python 算在 tmux 头上，而 Apple 签名的 `sshd` 是豁免的。Tailnet 地址（100.x，走 `utun`）不受影响。报告里只测了**出站**连接。— [Spexcode issue #73](https://github.com/shuxueshuxue/Spexcode/issues/73)、[vdaluz 博客](https://vdaluz.com/blog/macos-local-network-permissions-no-route-to-host)、[Homebrew discussion #6076](https://github.com/orgs/Homebrew/discussions/6076)
- 修复方法：在 System Settings → Privacy & Security → Local Network 里给对应 App 开权限，或关掉再打开一次。— [vdaluz 博客](https://vdaluz.com/blog/macos-local-network-permissions-no-route-to-host)、[iachieved.it](https://dev.iachieved.it/iachievedit/macos-sequoia-no-route-to-host/)

### Inferences
- 用 `http://mac-mini:5173` 访问 Vite 时，`mac-mini` 既不是 IP 也不是 `.localhost`，会被 Host 检查拦下。可以改用 `http://100.x.y.z:5173`（IP 默认放行），或者设 `server.allowedHosts: ['mac-mini', '.tailXXXX.ts.net']`（换成你自己的 tailnet 名）。**不要写 `.ts.net`**，因为 `ts.net` 是公共后缀（见 Q5），不是你能控制的域名。
- Next.js 16 要写 `allowedDevOrigins: ['mac-mini', 'mac-mini.tailXXXX.ts.net', '100.x.y.z']`。官方通配语义按 DNS label 切分，文档没说 IP 能不能用 `*` 匹配，所以最稳的是逐个写出具体主机名和 IP。
- 直接用 HTTP 主机名访问会丢掉安全上下文：Service Worker/PWA、`crypto.subtle`、`getUserMedia`、Clipboard 等 API 不可用或行为不同。OAuth 回调也不再是 `localhost`（见 Q5）。所以这条路只适合「看一眼 UI」。
- 让 dev server 绑 `0.0.0.0` 等于把它暴露给宿主所在的整个 LAN。更收敛的做法是只绑 Tailscale IP（Vite `--host 100.x.y.z`），或者继续绑 127.0.0.1，再用 Tailscale Serve 或 SSH 转发出去（Q2/Q3）。
- 在家同一个 LAN 里，仍然建议用 MagicDNS 或 100.x 访问，而不是 192.168.x.x：Tailscale 走 utun，绕开了 macOS Local Network Privacy 那类「No route to host」怪问题。这对宿主上 herdr 内 agent 的**出站** LAN 访问尤其要紧，因为 herdr 和 tmux 一样是 responsible process。至于笔记本浏览器访问宿主 LAN IP 这种入站场景，报告只测了出站，没有直接覆盖。

### Gaps
- macOS 应用防火墙（Application Firewall）对 node/bun 这类未签名或 ad-hoc 签名二进制的入站弹窗行为（"accept incoming network connections?"），以及 `socketfilterfw` 的命令行用法，本轮没取到一手来源。
- `next dev` 在 16.x 的默认监听地址（`-H/--hostname` 默认值）没从官方文档核实到。
- Chrome 的 Local Network Access（LNA，2025 年起逐步收紧「公网页面访问私网」）会不会影响 `100.64.0.0/10` 这类 CGNAT 地址，没找到权威说明。

---

## Q2. Tailscale Serve / Funnel / Services

### Takeaway
Tailscale Serve 是 2026 年从笔记本或手机访问宿主 dev server 的首选：只在 tailnet 内可见，自动签发 `*.ts.net` 的 Let's Encrypt 证书，所以页面是 HTTPS 安全上下文；`--bg` 配置重启后保留；多个 worktree 可以用不同的 `--https=PORT` 并存。Funnel 会把服务暴露到公网，dev server 永远不要用。Tailscale Services 已于 2026-02-19 GA，但要求宿主用 tag 身份，免费额度是每用户 10 个，适合 code-server 这类长期服务，不适合几十个临时 worktree。

### Cited Findings
- **Serve 基本用法**：`tailscale serve 3000` 把本地服务代理到 tailnet。前提是在 tailnet 里启用 HTTPS 证书，CLI 会引导你同意。前台模式下会显示正在服务的内容和访问 URL。— [Tailscale Serve KB 1312](https://tailscale.com/kb/1312/serve)（页面标注 last validated 2026-01-20）
- Serve 的 CLI 在 **1.52** 版做过改动（之前的博客或教程语法可能过时）；app capabilities 转发需要 v1.92+。— [KB 1312](https://tailscale.com/kb/1312/serve)
- **身份头**：Serve 会给反代请求加 `Tailscale-User-Login`、`Tailscale-User-Name`、`Tailscale-User-Profile-Pic`；**来自 tagged 设备的流量不带这些头**。文档建议只代理到 localhost，这是安全最佳实践。— [KB 1312](https://tailscale.com/kb/1312/serve)
- **CLI 参考**：`tailscale serve [flags] <target> [off]`。主要参数：`--https=<port>`（默认）、`--http=<port>`、`--tcp=<port>`、`--tls-terminated-tcp=<port>`、`--set-path=<path>`、`--service=...`、`--accept-app-caps`、`--proxy-protocol=1|2`、`--tun`、`--bg`（后台运行，重启后仍在）、`--yes`。子命令：`status [--json]`、`reset`、`get-config/set-config`、`drain`、`advertise`。目标写法：`tailscale serve localhost:3000`、`tailscale serve --http=80 localhost:3000`、`text:"Hello"`、文件路径；后端是 HTTPS 时写 `https://localhost:8443`，后端是自签证书时写 `https+insecure://localhost:8443`。关闭单条规则：`tailscale serve --https=443 --set-path=/foo off`。**在 macOS 上，文件服务只有开源版 tailscaled 支持。**— [tailscale serve CLI KB 1242](https://tailscale.com/kb/1242/tailscale-serve)
- **多端口实践**：portless 的 `--tailscale` 模式给每个 app 单独分配一个 Tailscale HTTPS 端口（第一个 443，之后 8443、8444……）并挂在根路径，"so no framework `basePath` configuration is needed"。这间接说明 Serve 可以在 443/8443/10000 以外的任意端口上开 HTTPS。— [portless README](https://github.com/vercel-labs/portless)
- **Serve 会透传 ts.net 作为 Host**：用 Tailscale 共享 Vite 时报 "Blocked request. This host ('<device>.<tailnet>.ts.net') is not allowed"，解决办法是 `server.allowedHosts: ['.tail21f530.ts.net']`。— [portless issue #310](https://github.com/vercel-labs/portless/issues/310)
- **HTTPS 证书**：先开 MagicDNS，再在 admin console 的 DNS 页点 "Enable HTTPS"。证书由 Let's Encrypt 签发，DNS-01 校验，私钥留在本机。证书名是 `{machine-name}.{tailnet-dns-name}`，tailnet 名形如 `tailNNNN.ts.net`，也可以换成随机词组名。证书 90 天过期，用 `tailscale cert` 导出文件时要自己续期。**机器名会写进公开的 Certificate Transparency 日志**，"Do not enable the HTTPS feature if any of your machine names contain sensitive information"。— [Enabling HTTPS KB 1153](https://tailscale.com/kb/1153/enabling-https)
- **Funnel（公网）**：只能用 443、8443、10000 三个端口；要求 v1.38.3+、MagicDNS、HTTPS 证书，并且策略文件里有 `funnel` node attribute。有不可配置的带宽上限，relay 不解密流量；公网 DNS 最多要 10 分钟才生效；反复申请证书可能撞上 Let's Encrypt 限流，被锁 34 小时。只想在 tailnet 内共享时应该用 Serve。— [Funnel KB 1223](https://tailscale.com/kb/1223/funnel)
- **Tailscale Services**：把资源做成有稳定 MagicDNS 名和 TailVIP 的服务，名字格式 `svc:<name>`，访问地址 `https://web-server.<tailnet>.ts.net`。宿主端命令：`tailscale serve --service=svc:web-server --https=443 127.0.0.1:8080`。要求 v1.86.0+；**服务宿主必须用 tag 身份，不能是用户账号登录**；只有 Owner/Admin/Network admin 能定义服务；只支持 TCP；Layer 3（`--tun`）只支持 Linux。grants 示例：`{"src":["autogroup:member"],"dst":["svc:web-server"],"ip":["443"]}`。— [Tailscale Services KB 1552](https://tailscale.com/kb/1552/tailscale-services)
- **Services 在 2026-02-19 GA**：所有套餐（含 Personal）都能用，每个用户最多 10 个服务；自动分配 VIP 要客户端 1.94.1+。— [Tailscale blog: Services GA](https://tailscale.com/blog/services-ga)
- 用 Tailscale 跑 Services 时，`tailscale serve` 默认就在后台运行。— [KB 1242](https://tailscale.com/kb/1242/tailscale-serve)

### Inferences
- **推荐的多 worktree 方案（Serve 端口制）**：dev server 一律绑 127.0.0.1，每个 worktree 执行 `tailscale serve --bg --https=<8400+N> localhost:<3000+N>`，笔记本访问 `https://mac-mini.tailXXXX.ts.net:<8400+N>`。用 `tailscale serve status` 查看，worktree 删掉时执行 `tailscale serve --https=<port> off` 清理，`tailscale serve reset` 清空全部。这件事可以写进 herdr 或 worktree 的创建和销毁 hook。
- 不推荐 `--set-path=/wt-a` 这种路径制：多数 dev server（Vite、Next）要改 `base`/`basePath` 才能正常工作。portless 选择「每个 app 一个端口、根路径挂载」正是为了避开这个问题。
- ts.net 证书只签发给机器名本身（`{machine-name}.{tailnet}`），**拿不到 `wt-a.mac-mini.tailXXXX.ts.net` 这样的通配子域证书**。想按子域区分 worktree，要么用 Services（每个服务一个名字，但需要 tag 宿主，而且上限 10 个），要么用自有域名配 DNS-01 通配证书（见 Q4）。
- 宿主加 tag 以后变成「tagged device」，Serve 不再带用户身份头，设备也不再归属个人账号。对一台个人宿主 Mac 来说，为了 Services 去打 tag 代价偏大，建议只给 code-server 这类少量长期服务考虑。
- 页面走 HTTPS 以后是安全上下文，Service Worker、摄像头、剪贴板都能用；Vite HMR 的 WebSocket 会按页面地址走 `wss://mac-mini…ts.net:<port>`，依赖 Serve 代理 WebSocket（见 Q5 引用的 Vite 说明）。
- 在同一 LAN 里访问 ts.net 名字，流量仍然走 Tailscale，一般能建立点对点直连，不用切换配置。这一点本轮没取到一手来源。

### Gaps
- Serve 对 WebSocket 的支持、HTTPS 可用端口范围，没在官方 KB 正文里找到明确表述（portless 的用法和社区的 code-server 实践间接支持）。`tailscale.com/docs/features/tailscale-serve` 和 KB 1313（serve examples）几次抓取都失败了。
- macOS 上 App Store 版、Standalone 版和开源 tailscaled 三种形态对反代 Serve 的差异，只核实到「文件服务只有开源版支持」这一点。
- Serve 转发时是否附带 `X-Forwarded-Host`/`X-Forwarded-Proto` 等头，没核实。

---

## Q3. SSH 本地端口转发（`-L`）、动态 SOCKS（`-D`）、ControlMaster、VS Code Remote-SSH、sshuttle

### Takeaway
`ssh -L 3000:localhost:3000 mac-mini` 让浏览器看到的 origin 仍然是 `http://localhost:3000`。这是唯一不用改任何 dev server 配置就能同时满足安全上下文、OAuth `localhost` 回调、Vite/Next 默认 Host 白名单的方案。代价是每个端口都要单独转发。配合 ControlMaster 用 `ssh -O forward` 可以在已有连接上随时增减转发，适合按 worktree 动态管理。

### Cited Findings
- `-L`："connections to the given TCP port or Unix socket on the local (client) host are to be forwarded to the given host and port ... on the remote side"。bind_address 写 `localhost` 表示只供本机使用，留空或写 `*` 表示所有网卡都能访问；默认按 `GatewayPorts` 设置决定。— [OpenBSD ssh(1)](https://man.openbsd.org/ssh.1)
- `-D`：本地动态转发，ssh 充当 SOCKS4/SOCKS5 服务器，按应用协议决定从远端连到哪里。— [OpenBSD ssh(1)](https://man.openbsd.org/ssh.1)
- `-O ctl_cmd` 控制已有的 master 连接：`check`、`forward`（"Request forwardings without command execution"）、`cancel`（"Cancel forwardings"）、`exit`、`stop`。`-M`/`-S` 用于 ControlMaster 连接复用；`-N` 不执行远程命令，适合只做转发；`-f` 转到后台。— [OpenBSD ssh(1)](https://man.openbsd.org/ssh.1)
- **VS Code Remote-SSH**：可以用 "Forward a Port" 命令或 Ports 视图里的 Add Port 临时转发。**本地端口被占用时会自动换一个**，例如远端 3000 可能映射成 localhost:4123，可以右键选 "Change Local Address Port" 改回来。持久转发建议在 ssh config 里写 `LocalForward 127.0.0.1:3000 127.0.0.1:3000`；`"remote.restoreForwardedPorts": true` 会在会话之间记住转发。— [VS Code Remote-SSH docs](https://code.visualstudio.com/docs/remote/ssh)
- Apple 签名的 `/usr/bin/ssh`、`sshd` 对 Local Network Privacy 是豁免的；Homebrew 版 OpenSSH 在 Sequoia 上会遇到「No route to host」。— [Spexcode issue #73](https://github.com/shuxueshuxue/Spexcode/issues/73)、[Homebrew discussion #6076](https://github.com/orgs/Homebrew/discussions/6076)
- Google OAuth 规则：redirect URI 必须是 HTTPS，"Localhost URIs (including localhost IP address URIs) are exempt"；host 不能是裸 IP（localhost IP 除外）；不允许通配符。— [Google OAuth 2.0 Web Server](https://developers.google.com/identity/protocols/oauth2/web-server)

### Inferences
- **按 worktree 动态转发的做法**：
  1. `~/.ssh/config` 里给 `mac-mini` 配 `ControlMaster auto`、`ControlPath ~/.ssh/cm-%C`、`ControlPersist 10m`。
  2. `ssh -fN mac-mini` 建立 master 连接。
  3. worktree 启动时 `ssh -O forward -L 3007:localhost:3007 mac-mini`，结束时 `ssh -O cancel -L 3007:localhost:3007 mac-mini`。
  整个过程不需要新的认证和握手，也可以用脚本读宿主的端口登记表批量处理。建议加 `ExitOnForwardFailure yes`，这样本地端口被占用时直接报错，不会默默失败。
- 一定要让本地和远端端口号一致（例如 3007→3007）。OAuth 回调、`NEXTAUTH_URL`/`APP_URL` 这类环境变量、cookie 都绑定到 `localhost:3007`，VS Code 自动换成 4123 这种情况会让 OAuth 回调不匹配。
- 默认 bind 就是 localhost，**不要写 `-L 0.0.0.0:3000:...` 或 `-L *:3000:...`**，否则会把宿主的 dev server 再暴露给笔记本所在的咖啡馆 LAN。
- `-D` SOCKS 方案：一个隧道就能访问宿主能访问的所有端口，比较适合给某个专用浏览器 profile 配 `socks5://localhost:1080`。但通过代理访问的目标要写成 `http://mac-mini:3007` 或 `http://localhost:3007`，而 Chrome 默认对 loopback 地址不走代理（需要 `--proxy-bypass-list="<-loopback>"`），这样 origin 又不是 localhost 了，失去了 `-L` 的主要优点。有 Tailscale 以后 `-D` 的价值不大。这条推论里 Chrome 的 bypass 行为本轮没有来源核实。
- sshuttle（把 SSH 当成简易 VPN）在已经有 Tailscale 三层互通的场景里是多余的，本轮没有深入调研。
- LAN 和 Tailscale 下 `-L` 的行为完全一样，只是 `mac-mini` 解析到的地址不同。在 Tailscale 上还可以用 Tailscale SSH，免去密钥管理（属于相邻主题，其他笔记覆盖）。

### Gaps
- VS Code 的 `remote.autoForwardPorts` 和 `remote.autoForwardPortsSource`（process/output/hybrid）细节在抓到的页面里没有，需要看 VS Code settings 参考。Cursor、Zed 等编辑器的自动转发行为也没核实。
- ssh_config 没有「端口区间」转发语法，这是我的记忆，未引用。
- sshuttle 在 macOS 26 上的现状没有调研。

---

## Q4. 宿主上的反向代理 + 每个 worktree 一个子域名（Caddy、`*.localhost`、portless 等）

### Takeaway
portless（vercel-labs，2026 年仍在 1.0 前，npm 0.15.6）在「每个 worktree 自动一个名字」这件事上做得最完整：自动分配端口，linked worktree 自动用 `<branch>.<app>.localhost`，本地 CA 提供 HTTPS，还有 `--tailscale` 模式为每个 app 分配 ts.net 端口。缺点是子域名方案只在宿主本机的浏览器上真正成立，因为 `*.localhost` 只解析到本机。笔记本要按名字访问，需要 SSH 转发 443，或者自有域名加通配 DNS 和证书。Caddy 能自动从本机 tailscaled 取 `*.ts.net` 证书，适合做统一入口。

### Cited Findings
- **portless 的机制**：本地 HTTPS 代理（默认 443，`--no-tls` 时用 80），首次运行会生成本地 CA 并加入系统信任库，支持 HTTP/2，WebSocket 在 HTTP/1.1 和 HTTP/2 上都能用。app 端口从 4000–4999 随机分配并通过 `PORT` 传入；对不读 `PORT` 的框架（Vite、Astro、Angular）会自动注入 `--port`，必要时再注入 `--host`。常用命令：`portless run`、`portless <name> <cmd>`、`portless proxy start`、`portless list`、`portless alias <name> <port>`。— [portless README](https://github.com/vercel-labs/portless)
- **worktree 支持**："In a linked worktree, the branch name is prepended as a subdomain"，例如 `fix-ui.myapp.localhost`。— [portless README](https://github.com/vercel-labs/portless)
- **网络模式**：默认只绑 `127.0.0.1` 和 `::`；`--lan` 绑 `0.0.0.0`/`::` 并用 mDNS 发布 `.local` 名字（Next.js 需要把 `.local` 名字加进 `allowedDevOrigins`）；`--tailscale` 输出 `https://devbox.yourteam.ts.net` 这样的地址，每个 app 挂在独立的 Tailscale HTTPS 端口上（8443、8444……）；`--funnel` 和 `--ngrok` 会暴露到公网。其他注意点：绑 443 需要 sudo（会自动提权）；Safari 可能要执行 `portless hosts sync` 同步 `/etc/hosts`；需要 Node 24+；版本在 1.0 前，可能有破坏性变更。— [portless README](https://github.com/vercel-labs/portless)
- **portless 已知问题**：`PORTLESS_TAILSCALE=1` 时只往 Vite allowedHosts 注入了 `.localhost`，没注入 ts.net 主机名，结果 Vite 返回 "Blocked request"，需要手动加 `.tailXXXX.ts.net`。— [portless issue #310](https://github.com/vercel-labs/portless/issues/310)
- **`*.localhost` 在浏览器里是安全上下文**（host 以 `.localhost` 结尾就算）。— [MDN Secure Contexts](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts)。Vite 默认也放行 `.localhost` 子域 — [Vite docs](https://github.com/vitejs/vite/blob/main/docs/config/server-options.md)；Next.js 默认放行 `localhost` 的子域 — [Next.js allowedDevOrigins](https://github.com/vercel/next.js/blob/canary/docs/01-app/03-api-reference/05-config/01-next-config-js/allowedDevOrigins.mdx)
- **Caddy + Tailscale**：以 `.ts.net` 结尾的站点不需要任何 TLS 配置，Caddy（2.5+）会在握手时从本机 Tailscale 守护进程取证书。条件是 tailnet 已开 HTTPS，并且 Caddy 以 root 运行，或者 tailscaled 已授权 Caddy 所在用户取证书。— [Caddy Automatic HTTPS](https://caddyserver.com/docs/automatic-https)、[Tailscale blog: Caddy](https://tailscale.com/blog/caddy)
- Caddy 的 `tls internal` 用本地 CA 给内部域名签证书。— [Caddy tls directive](https://caddyserver.com/docs/caddyfile/directives/tls)
- **Cookie 不按端口隔离**："Cookies do not provide isolation by port. If a cookie is readable by a service running on one port, the cookie is also readable by a service running on another port of the same server." — [RFC 6265 §8.5](https://www.rfc-editor.org/rfc/rfc6265#section-8.5)

### Inferences
- **子域方案真正的好处是 cookie 隔离**。`localhost:3001` 和 `localhost:3002`、`mac-mini…ts.net:8443` 和 `:8444` 之间 cookie 是共享的，多个 worktree 同时登录同一个 app 时 session cookie 会互相覆盖（同名 `session`/`next-auth.session-token`）。`wt-a.app.localhost` 和 `wt-b.app.localhost` 则天然隔离。
- **笔记本怎么用上宿主的 `*.localhost` 名字**：
  - 方案 A：`ssh -L 443:localhost:443 mac-mini`（本地绑 443 需要 root），笔记本浏览器访问 `https://fix-ui.myapp.localhost`，笔记本本机把 `*.localhost` 解析到 127.0.0.1，经隧道到宿主上的 portless 或 Caddy。代理按 SNI/Host 路由。笔记本还要信任宿主 portless 或 Caddy 的本地 CA 根证书（手动导入钥匙串）。
  - 方案 B：用非特权本地端口，例如 `-L 8443:localhost:443`，访问 `https://fix-ui.myapp.localhost:8443`（代理必须按 Host 路由，不能依赖端口）。
  - 这样 origin 是 `*.localhost`，属于安全上下文，Vite 和 Next 默认放行，cookie 按 worktree 隔离。OAuth 提供方是否接受 `*.localhost` 回调因 provider 而异：Google 只对 localhost 豁免 HTTPS 要求，对子域名的处理没核实。
- **方案 C（自有域名）**：给自有域名设 `*.dev.example.com` A 记录指向宿主的 Tailscale 100.x IP（只有 tailnet 内能连通），Caddy 通过 DNS-01 申请通配证书，按子域 `reverse_proxy localhost:<port>`。笔记本和手机都能直接用，但需要 Caddy 的 DNS provider 插件，这个没核实。
- **方案 D**：portless `--tailscale`，也就是每个 app 一个 ts.net 端口。本质就是 Q2 的「Serve 端口制」加上自动管理，但有 cookie 按端口共享的问题。
- 旧工具 hotel、puma-dev（`.test`/`.dev` 本地 DNS）在 2026 年基本被 `*.localhost` 方案取代，因为 `.localhost` 由浏览器原生解析，不需要本地 DNS。这个判断没核实各项目的维护状态。

### Gaps
- portless `--tailscale` 模式具体是不是调用 `tailscale serve`，README 没说。
- hotel、puma-dev、Caddy 通配 DNS-01 插件在 2026 年的维护状态没核实。
- Safari 对 `*.localhost` 的解析行为只知道 portless 提到的 `hosts sync` 变通办法，没有 WebKit 一手来源。

---

## Q5. 跨方案的「origin 问题」：OAuth 回调、安全上下文、cookie、CORS、HMR WebSocket

### Takeaway
选方案时最关键的是浏览器地址栏里的 origin 是什么：
- `http://localhost:PORT`（SSH `-L`）：兼容性最好。
- `https://<name>.localhost`（portless/Caddy + 隧道）：兼容性好，而且 cookie 隔离。
- `https://mac-mini.<tailnet>.ts.net:PORT`（Serve）：是 HTTPS 安全上下文，手机也能用，但要改 Vite/Next 白名单，还要登记额外的 OAuth 回调。
- `http://mac-mini:PORT`（直连）：最差。

### Cited Findings
- `ts.net` 在公共后缀列表（PSL）的 Tailscale 私有段里，同时还有 `*.c.ts.net`。— [publicsuffix.org list](https://publicsuffix.org/list/public_suffix_list.dat)（本轮直接 grep 原文件确认）
- Google OAuth redirect URI：必须是 HTTPS（localhost 例外）；不能是裸 IP（localhost IP 例外）；顶级域必须在 PSL 里；不能有通配符。— [Google OAuth 2.0](https://developers.google.com/identity/protocols/oauth2/web-server)
- Cookie 不按端口隔离，也不按 scheme 隔离。— [RFC 6265 §8.5](https://www.rfc-editor.org/rfc/rfc6265#section-8.5)
- **Vite HMR 在反代后面**：默认配置下，前面的反代需要能代理 WebSocket。如果 HMR 客户端连不上，会退回到绕过反代直连 Vite HMR 端口，并打印 "Direct websocket connection fallback"。规避办法有三个：让反代也代理 WS；设 `server.strictPort = true` 并让 `server.ws.clientPort` 等于 `server.port`；或者设不同的 `server.ws.port`。— [Vite docs 源文件](https://github.com/vitejs/vite/blob/main/docs/config/server-options.md)
- **2026 年 Vite 文档变化（旧文档已过时）**：`server.hmr` 里跟 WebSocket 相关的选项（`protocol`、`host`、`port`、`path`、`clientPort`、`timeout`、`server`）已标记 deprecated，改用 `server.ws.*`，旧配置会自动同步所以还能用；`server.hmr` 现在只剩 `overlay`。— [Vite docs 源文件 (main)](https://github.com/vitejs/vite/blob/main/docs/config/server-options.md)（注意：这是 main 分支文档，对应的发布版本号没核实；npm 最新是 8.3.0）
- Vite 的 `server.ws.server` 可以把 WS 挂到已有 server 上，"helpful when ... you want to expose Vite over a network on a single port"。— [Vite docs 源文件](https://github.com/vitejs/vite/blob/main/docs/config/server-options.md)
- Next.js 的 `allowedDevOrigins` 只比对 Origin 的 hostname（忽略端口）。— [Next.js docs 源文件](https://github.com/vercel/next.js/blob/canary/docs/01-app/03-api-reference/05-config/01-next-config-js/allowedDevOrigins.mdx)

### Inferences
- **OAuth**：
  - 回调已经注册成 `http://localhost:3000/api/auth/callback/...` 的项目，用 SSH `-L` 什么都不用改。
  - 走 Serve 的话，要在 provider 那里额外登记 `https://mac-mini.tailXXXX.ts.net:<port>/...`（Google 接受，因为是 HTTPS，而 `.net` 在 PSL 里），**每个 worktree 端口都要登记一条**（不允许通配）。另外要把 app 的 `NEXTAUTH_URL`/`AUTH_URL`/`APP_URL` 按 worktree 设成对应的 ts.net 地址。
  - 回调用 `http://mac-mini:3000` 会被 Google 拒绝（非 HTTPS、非 localhost）。
- **Cookie**：`ts.net` 在 PSL 里，所以 `Domain=.ts.net` 的 cookie 会被浏览器拒收，eTLD+1 是 `tailXXXX.ts.net`。也就是说，同一 tailnet 内所有机器的 ts.net 名字在 cookie 意义上属于同一个 site，可以用 `Domain=.tailXXXX.ts.net` 共享。同一台机器的不同端口之间 cookie 本来就共享。
- **`Secure` cookie 和 `__Host-` 前缀**：在 `http://mac-mini:3000` 上 `Secure` cookie 无法设置（Chrome 只对 localhost 有特例），很多 auth 库在生产模式下会默认加 `Secure`；走 Serve 的 HTTPS 就没这个问题。这条是常识推论，没引用。
- **CORS**：前后端分开两个端口（例如 web 3000、api 4000）时，换成 ts.net 或 mac-mini origin 后，后端的 CORS 白名单也要跟着加；Vite 的 `server.cors` 默认只认 localhost。
- **HMR**：
  - SSH `-L`：页面和 WS 同端口，自然能通。如果 `server.ws.port` 单独设了端口，那个端口也要转发。
  - Tailscale Serve：页面是 `https://…:8443`，Vite 客户端按页面地址连 `wss://…:8443`，经 Serve 反代到 localhost。如果出现 "Direct websocket connection fallback"，说明 WS 没穿过代理；因为 Vite 只绑 127.0.0.1，fallback 也会失败，表现为 HMR 不生效但页面能开。
  - Next.js 16：HMR 端点 `/_next/webpack-hmr`/`/_next/hmr` 会被 cross-origin 检查拦截，要加 `allowedDevOrigins: ['mac-mini.tailXXXX.ts.net']`（或 `'*.tailXXXX.ts.net'`）。

### Gaps
- GitHub、Auth0、Clerk、Supabase 等 OAuth 提供方对 `*.localhost`、`ts.net` 回调的具体规则没逐一核实。
- Chrome 对 `http://localhost` 上 `Secure` cookie 的特例行为缺一手来源。

---

## Q6. 远程浏览器调试与 Agent 驱动的浏览器（CDP、Chrome DevTools MCP、Playwright MCP、trace viewer）

### Takeaway
宿主上的 agent 应该用独立 profile 的 Chrome（或 Chrome for Testing），或者 Playwright 自带的浏览器，不要连用户的日常 Chrome。Chrome 136 起，`--remote-debugging-port` 必须配非默认的 `--user-data-dir`。笔记本想「看」或「接管」agent 的浏览器，走 `ssh -L 9222:localhost:9222` 加 `chrome://inspect`，不要把 CDP 端口暴露到网络上，因为 CDP 没有任何认证。异步复盘用 Playwright trace、截图、视频，不需要远程桌面。

### Cited Findings
- **Chrome 136 起的限制**：`--remote-debugging-port`/`--remote-debugging-pipe` 在默认用户数据目录上不再生效，必须配非标准的 `--user-data-dir`。原因是信息窃取木马利用远程调试绕过 App-Bound Encryption 偷 cookie。自动化场景推荐用 Chrome for Testing，它保持旧行为。— [Chrome blog: remote-debugging-port](https://developer.chrome.com/blog/remote-debugging-port)
- **chrome://inspect 连远程 Chrome**：目标 Chrome 用 `--remote-debugging-port=PORT` 启动；本地打开 `chrome://inspect/#devices`，勾选 "Discover network targets"，点 Configure，填 `localhost:PORT`。— [Chrome DevTools: Access local servers and Chrome instances](https://developer.chrome.com/docs/devtools/remote-debugging/local-server)
- 调试端口 "does not provide any authentication or encryption"，应该通过 SSH 隧道或 VPN 访问。— [Igalia: trace-chrome](https://blogs.igalia.com/dape/2025/03/25/trace-chrome-easy-remote-tracing-of-chromium/)（搜索摘要，属于二手表述）
- **Chrome DevTools MCP**（npm 1.10.1）：启动命令 `npx -y chrome-devtools-mcp@latest`。参数包括 `--headless`、`--isolated`（每个会话独立 context）、`--user-data-dir`（文档称默认在系统临时目录）、`--channel`、`--slim`；连接已有浏览器用 `--browser-url`/`--browserUrl`、`--wsEndpoint`、`--autoConnect`。能力覆盖截图、性能 trace、网络请求、console（带 source map）。免责声明：它会把浏览器内容暴露给 MCP 客户端。usage statistics 默认开启，可用 `--no-usage-statistics` 或 `CHROME_DEVTOOLS_MCP_NO_USAGE_STATISTICS` 关闭。— [chrome-devtools-mcp GitHub](https://github.com/ChromeDevTools/chrome-devtools-mcp)
- **`--autoConnect`（Chrome 144+）**：先在 `chrome://inspect/#remote-debugging` 打开远程调试，MCP 配置里加 `--autoConnect`；每次连接 Chrome 都会弹窗要求点 Allow，受控期间显示 "controlled by automated test software" 横幅；默认连 Stable，Beta/Canary 用 channel 指定；MCP server 是本地进程，不向 Google 上传浏览器数据。— [Chrome docs: auto-connect](https://developer.chrome.com/docs/devtools/agents/use-cases/auto-connect)、[Chrome blog: debug your browser session](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- **Playwright MCP**（npm 0.0.82）：启动命令 `npx @playwright/mcp@latest`。参数：`--host`（默认 localhost，`0.0.0.0` 表示所有网卡）、`--port`（独立 HTTP 模式，客户端配置 `"url": "http://localhost:8931/mcp"`）、`--allowed-hosts`、`--cdp-endpoint`、`--extension`（通过扩展连接已打开的 Chrome/Edge 标签页）、`--isolated`、`--user-data-dir`、`--output-dir`、`--save-session`、`--shared-browser-context`、`--caps`（vision/pdf/devtools）。README 说独立 `--port` 模式适合 "headed browsers on systems without display or IDE worker processes"，并强调 "Playwright MCP is **not** a security boundary"。编码 agent 可以改用更省 token 的 Playwright CLI（以 skills 形式提供）。— [playwright-mcp GitHub](https://github.com/microsoft/playwright-mcp)
- **Playwright 复盘工具**：`npx playwright show-trace path/to/trace.zip` 打开 trace viewer — [Playwright Trace viewer](https://playwright.dev/docs/trace-viewer)；UI Mode 可以用 `npx playwright test --ui-port=8080 --ui-host=0.0.0.0` 从别的机器访问 — [Playwright UI Mode](https://playwright.dev/docs/test-ui-mode)；`npx playwright show-report` 默认在 9323 端口起 HTML 报告服务 — [TestMu 学习中心](https://www.testmuai.com/learning-hub/playwright-reporting/)（二手来源）

### Inferences
- **推荐的宿主侧配置（多 agent）**：
  - 每个 agent 或 worktree 用 Playwright MCP 或 DevTools MCP 的 `--isolated`/`--headless`，各自有独立 profile，避免多个 agent 抢同一个浏览器。输出目录（`--output-dir`，放截图、trace、视频）放在 worktree 内，笔记本可以通过 Syncthing 或 `tailscale serve` 静态文件查看（macOS 上文件服务需要开源版 tailscaled），或者直接 `scp` 下来。
  - 想实时围观某个 agent：给它用 headed Chrome for Testing，加 `--remote-debugging-port=92xx --user-data-dir=/tmp/agent-xx`，MCP 用 `--browser-url=http://127.0.0.1:92xx` 连上去。笔记本执行 `ssh -N -L 92xx:localhost:92xx mac-mini`，然后在 `chrome://inspect` 里 Configure `localhost:92xx`，就能用 DevTools（含页面 screencast）看它，必要时手动接管。这等于用 DevTools 代替远程桌面。
- `--autoConnect` 会在**宿主屏幕**上弹 Allow 对话框，并且连的是宿主用户的日常 Chrome（带全部登录态），不适合无人值守的宿主，也有凭据风险。笔记本端 agent 想操作笔记本自己的浏览器时才用它。
- `--host 0.0.0.0` 暴露 Playwright MCP，或者 UI Mode 用 `--ui-host=0.0.0.0`，都会把浏览器控制权交给整个 LAN。应该保持 127.0.0.1，需要从笔记本访问时再用 `tailscale serve --https=<port> localhost:<port>` 或 SSH `-L`。
- Chrome 的 `--remote-debugging-port` 默认只监听 127.0.0.1（经验说法），所以通过 SSH 隧道访问是标准做法。

### Gaps
- Chrome `--remote-debugging-address` 在 2026 年是否仍只在 headless 下生效，以及 CDP 默认监听地址，没取到官方来源。
- `show-trace` 的默认端口、`--host` 参数没从 playwright.dev 一手核实。
- 宿主上 headed 浏览器通过 macOS 屏幕共享观看属于远程桌面主题，由其他笔记覆盖。
- Claude Code、Codex 自带的浏览器集成（如 Claude in Chrome）在宿主无人值守时的表现没调研。

---

## Q7. 用 Tailscale Serve 安全访问宿主上的 Web UI（code-server、Syncthing GUI 8384、Jupyter 等）

### Takeaway
通用模式：服务只绑 127.0.0.1，再用 `tailscale serve --bg --https=<端口> localhost:<服务端口>` 暴露到 tailnet，同时保留服务自己的认证（密码、token）。需要注意带 Host 检查的服务（Syncthing、Vite），因为 Serve 会把 ts.net 主机名透传给后端。

### Cited Findings
- Syncthing GUI 放在反代后面、GUI 又只绑 localhost 时，会报 "Host check error"。办法是在 `conf.xml` 的 `<gui>` 段里设 `<insecureSkipHostcheck>true</insecureSkipHostcheck>`，或者由反代把 Host 头改写成 localhost/127.0.0.1；并建议反代到用户之间只允许 TLS。— [Syncthing FAQ](https://docs.syncthing.net/users/faq.html)、[syncthing/docs issue #401](https://github.com/syncthing/docs/issues/401)、[syncthing/docs PR #402](https://github.com/syncthing/docs/pull/402)
- Serve 透传 ts.net Host（Vite 被拦截的证据）。— [portless issue #310](https://github.com/vercel-labs/portless/issues/310)
- Serve 文件和静态文本：`tailscale serve /path/index.html`、`text:"..."`（macOS 上文件服务只有开源版支持）；后端自签 HTTPS 用 `https+insecure://localhost:8443`。— [KB 1242](https://tailscale.com/kb/1242/tailscale-serve)
- 身份头 `Tailscale-User-Login` 等可以让后端知道访问者是谁（tagged 设备来源不带）。— [KB 1312](https://tailscale.com/kb/1312/serve)
- Services 适合长期服务，名字稳定（`https://<svc>.<tailnet>.ts.net`），但宿主需要 tag。— [KB 1552](https://tailscale.com/kb/1552/tailscale-services)

### Inferences
- **Syncthing**：GUI 保持 `127.0.0.1:8384`，执行 `tailscale serve --bg --https=8384 localhost:8384`（或其他空闲端口），并设 `insecureSkipHostcheck=true` 且保留 GUI 用户名和密码。因为 GUI 只绑 loopback，关闭 Host 检查主要剩下宿主本机浏览器被 DNS rebinding 的风险，这时 GUI 密码是最后一道防线。
- **code-server**：绑 `127.0.0.1:8080`，执行 `tailscale serve --bg --https=8080 localhost:8080`；需要 WebSocket。
- **Jupyter**：绑 127.0.0.1，保留 token，经 Serve 暴露。
- 这几项的具体参数本轮没取到一手文档（见 Gaps）。
- 可以把「服务 → Serve 端口」的映射写进一个脚本，并用 `tailscale serve status --json` 做自检。worktree 的 dev server 用一段端口区间（如 84xx），长期 UI 用另一段，避免冲突。

### Gaps
- code-server（`--bind-addr`、`--auth`、反代要求）、Jupyter（`ServerApp.allow_remote_access`、`allow_origin`、XSRF）在 Tailscale Serve 后面的具体配置，没抓到一手文档。
- Serve 能否改写 Host 头（像 nginx 的 `proxy_set_header Host`），没找到对应选项。

---

## Q8. 手机预览（经 Tailscale）

### Takeaway
手机装 Tailscale 客户端加入同一 tailnet 后，直接打开 `https://mac-mini.<tailnet>.ts.net:<port>`（Serve）即可。页面是 HTTPS 安全上下文，PWA/Service Worker/摄像头等都能测。手机不方便用 SSH `-L`，所以 Serve 是手机预览的主力方案。

### Cited Findings
- Serve 用于 "share local services only with other devices in your tailnet"，要公网访问才用 Funnel。— [Funnel KB 1223](https://tailscale.com/kb/1223/funnel)
- ts.net 证书由 Let's Encrypt 签发，受浏览器信任，不需要在手机上装 CA。— [Enabling HTTPS KB 1153](https://tailscale.com/kb/1153/enabling-https)
- portless `--tailscale` 能给每个 app 一个 ts.net HTTPS 地址（8443、8444……），手机可以直接访问。— [portless README](https://github.com/vercel-labs/portless)
- 用 HTTP 访问非 localhost 主机时不是安全上下文。— [MDN Secure Contexts](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts)

### Inferences
- 相比 portless/Caddy 的本地 CA 方案（手机要装并信任根证书，iOS 还要在「证书信任设置」里手动开启），ts.net 的公有证书对手机最省事。
- 不想让手机常驻 tailnet 的话，只能用 Funnel 或 ngrok 走公网，这违背「不公开暴露」的原则，不推荐。
- 手机上测 OAuth 登录时，回调要登记 ts.net 地址（见 Q5）。

### Gaps
- 没核实 iOS/Android Tailscale 客户端在 2026 年的 MagicDNS 短名（`http://mac-mini:3000`）解析表现；FQDN 应该总是可用。

---

## Q9. 安全底线与 2026 年现状、过时信息汇总

### Takeaway
原则只有一条：dev server 和各种 UI 只绑 127.0.0.1，经 Tailscale Serve（tailnet 内、HTTPS、带身份头）或 SSH `-L` 访问；永远不要对 dev server 用 Funnel、ngrok、`allowedHosts: true` 或把 CDP 端口暴露到网络。旧教程里过时的点主要有：Tailscale Serve 1.52 之前的语法、Vite 的 `server.hmr.clientPort`、Chrome 136 之前的远程调试用法、Next.js 16 之前的宽松跨域默认值。

### Cited Findings
- 开启 Tailscale HTTPS 会把机器名写进公开的 CT 日志，机器名不要含敏感信息。— [KB 1153](https://tailscale.com/kb/1153/enabling-https)
- Funnel 是公网暴露，另有带宽限制和证书限流风险。— [KB 1223](https://tailscale.com/kb/1223/funnel)
- Vite 的 `allowedHosts: true`、`cors: true` 都有 DNS rebinding 风险。— [Vite docs](https://github.com/vitejs/vite/blob/main/docs/config/server-options.md)
- CDP 远程调试被信息窃取木马用来偷 cookie（Chrome 136 因此加了限制）。— [Chrome blog](https://developer.chrome.com/blog/remote-debugging-port)
- Playwright MCP "not a security boundary"；DevTools MCP 会把浏览器内容暴露给 MCP 客户端。— [playwright-mcp](https://github.com/microsoft/playwright-mcp)、[chrome-devtools-mcp](https://github.com/ChromeDevTools/chrome-devtools-mcp)
- ssh `-L` 默认 bind 由 `GatewayPorts` 决定；写 `*` 或空地址表示所有网卡。— [ssh(1)](https://man.openbsd.org/ssh.1)
- 2026 年关键版本和事件：Tailscale Services GA（2026-02-19）— [blog](https://tailscale.com/blog/services-ga)；Serve KB 最后验证于 2026-01-20，CLI 在 1.52 改过语法 — [KB 1312](https://tailscale.com/kb/1312/serve)；Chrome 144+ 支持 DevTools MCP autoConnect — [Chrome docs](https://developer.chrome.com/docs/devtools/agents/use-cases/auto-connect)；Vite 文档里 `server.hmr` 的 WS 选项已迁到 `server.ws` — [Vite docs](https://github.com/vitejs/vite/blob/main/docs/config/server-options.md)；Next.js 16 默认拦截跨 origin 的 dev 资源 — [Pramuditya](https://www.pramuditya.com/en/blog/nextjs-16-hmr-breaks-over-lan-ip-alloweddevorigins)；portless 仍在 1.0 前 — [README](https://github.com/vercel-labs/portless)

### Inferences
- **各方案对照（给报告作者）**：

| 方案 | 浏览器 origin | 安全上下文 | 改 dev 配置 | 多 worktree | 手机 | cookie 隔离 | 公开风险 |
|---|---|---|---|---|---|---|---|
| 直连 `http://mac-mini:PORT` | mac-mini:PORT | 否 | 要（绑 0.0.0.0、Vite allowedHosts、Next allowedDevOrigins） | 端口天然区分 | 可 | 否 | LAN 可见 |
| Tailscale Serve `--https=PORT` | `mac-mini.tailX.ts.net:PORT` | 是 | 要（allowedHosts/allowedDevOrigins 加 ts.net） | 每个 worktree 一条 serve | 最佳 | 否（端口共享） | 仅 tailnet |
| SSH `-L` 同端口 | `localhost:PORT` | 是 | 不用 | 用 `ssh -O forward/cancel` 动态增减 | 差 | 否 | 仅本机 |
| portless/Caddy 子域 + SSH 443 隧道 | `wt.app.localhost` | 是 | 基本不用 | 自动（按分支名） | 差（需要 CA） | 是 | 仅本机 |
| portless `--tailscale` | ts.net:8443+ | 是 | Vite 要手动加 ts.net（issue #310） | 自动分配端口 | 好 | 否 | 仅 tailnet |
| Funnel/ngrok | 公网 | 是 | 要 | — | 好 | — | **公开，禁止用于 dev** |

- 建议组合：
  - 笔记本日常：SSH ControlMaster 加脚本化的 `-O forward`，保证 localhost 语义（OAuth 和 cookie 省心）。
  - 手机或临时从别的设备看：Tailscale Serve 端口制。
  - agent 自己的浏览器：headless 加 isolated profile，出 trace 和截图；需要时用 CDP 隧道配 chrome://inspect 实时围观。
  - 长期 UI（Syncthing、code-server）：固定的 Serve 端口。

### Gaps
- 没找到 herdr 自身对端口登记或转发 hook 的支持文档，这不在本主题范围内。
- Tailscale ACL/grants 里怎么把宿主的 Serve 端口限制到只有本人设备能访问（例如 `autogroup:self`），没取到具体策略示例。
