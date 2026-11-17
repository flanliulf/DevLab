# 六层分工，让笔记本接管宿主 Mac

从笔记本掌控家里或办公室那台跑着多个 agent 的 Mac，不该去找一个"万能远控"，而应按层拆分，让日常操作尽量以**共享状态**而非**共享像素**完成。终端交给已有的 herdr。编辑器用 VS Code Remote-SSH 或 Zed 远程直接读写宿主磁盘。浏览器用 SSH `-L` 保住 `localhost` 语义，手机和 HTTPS 场景交给 Tailscale Serve。文件以 git 为唯一真相，Syncthing 交换目录与 scp 只搬截图、spec 和产物。只有点授权弹窗、首次登录、看模拟器这类偶发需求，才打开 Apple 屏幕共享的**高性能模式**：两端都是 Apple silicon 加 macOS 14+ 即可免费使用，最多两块 4K 虚拟屏，每块 4K 建议 **75 Mbps**。通用控制、Deskflow 这类键鼠共享只传输入、不传画面。通用控制还依赖蓝牙加点对点 Wi‑Fi 的近场链路、不走 IP 路由，所以它只在宿主显示器就摆在笔记本旁边时才有用，Tailscale 帮不上忙。一旦离开那张桌子，笔记本自己的键盘本来就是唯一的输入设备，"切换键鼠"不再是问题，真正要优化的是**少开整屏共享**。每一层都会碰到同一个陷阱：macOS 的会话边界。SSH 登录不在 GUI 会话里，Keychain、TCC 授权、截图、剪贴板和通知都会静默失效。可靠的做法是把 GUI 敏感命令交给由 Terminal 拉起的 herdr pane，或交给已拿到授权的常驻应用去执行。macOS 27 Golden Gate 已于 9 月 14 日发布，Deskflow 1.26 在 27 上会崩溃，多数第三方工具还没有 27 支持声明，宿主暂时留在 macOS 26 更稳。另外，macOS 原生应用的"单窗口转发"在 2026 年仍没有成熟方案，Xcode 和模拟器应改走无头构建加截图。

> 与既有手册的衔接：本报告用 `§x.y` 指代 [herdr 完整实施手册](../../agents/herdr/herdr-完整实施手册-最终版.md) 的小节，那里已写过的步骤（Tailscale 安装、`herdr machine add`、pmset、TCC、Terminal 登录项拉起 herdr-server 等）不再重复。宿主主机名沿用手册约定的 `mac-mini`。

## 三种控制方式，只有"共享状态"走得出房间

"用笔记本操作宿主"其实分三种机制，适用范围差别很大。**共享输入**把笔记本的键鼠事件送到宿主，画面仍然留在宿主自己的显示器上。Deskflow 对自己的定位就是 "software KVM (but without the video)"（[Deskflow](https://github.com/deskflow/deskflow)），通用控制、ShareMouse、硬件 KVM 也属于这一类。**共享像素**把宿主整块屏幕编码成视频送过来，再把输入传回去。它什么都能做，但对带宽和延迟的要求最高。**共享状态**是 herdr、Remote-SSH、端口转发和 git 走的路：界面在笔记本本地渲染，网络上只传终端字节、文件事件和 HTTP 请求。用户的原始诉求是用笔记本**自己的屏幕**和键鼠来操作。共享输入只满足了一半，因为画面还在宿主屏上。能把宿主的东西放到笔记本屏幕上的，只有共享像素和共享状态。结论是：共享输入只在同桌时适用，共享像素是万能兜底但要少用，日常工作交给共享状态。

对延迟的敏感程度也按这个顺序递减。整屏共享最敏感。在 herdr 里用 Neovim 次之，因为每次按键都要走一次网络往返。VS Code、Zed 这类远程编辑器在本地渲染、本地回显，只有补全、跳转、保存受网络影响。GitHub PR 网页审阅与连接质量完全无关。Tailscale 的连接都先经 DERP 中继建立，再尝试升级为直连（[Tailscale 文档](https://tailscale.com/docs/reference/connection-types)）。同一大洲内直连一般在 5–30 ms，中继在 50–200 ms（二手经验值）（[SSD Nodes](https://www.ssdnodes.com/learn/tailscale-slow-direct-vs-relayed)）。所以远程走中继时，最先变得难用的是整屏共享和终端内编辑，远程编辑器几乎不受影响。这就是要分层的根本原因。

### SSH 进不了 GUI 会话，这条边界贯穿每一层

SSH 登录落在 launchd 的 Background 域（`user/<uid>`），GUI 终端则在 Aqua 域（`gui/<uid>`），可以用 `launchctl managername` 查看当前属于哪个域（[the KodeLab](https://thekodelab.com/en/posts/macos-gui-terminal-vs-ssh-session/)）。这条边界在每一层都会造成问题。在 Tahoe 上，从 SSH 直接执行 app 包里的二进制，窗口会出现，但**收不到键盘事件**，应该改用 `open` 交给 LaunchServices 启动（[Apple 开发者论坛](https://developer.apple.com/forums/thread/803553)）。没有"屏幕录制"授权时，经 SSH 执行 `screencapture` 只能得到黑帧，而在无头会话里，修复它所需的授权弹窗永远不会出现（[Petrenko](https://jonnyzzz.com/blog/2026/07/16/a-real-macos-desktop-for-a-linux-agent/)）。login Keychain 在 SSH 里处于锁定状态，Claude Code 会反复要求登录（[phoenixtrap](https://phoenixtrap.com/2025/10/26/claude-code-cli-over-ssh-on-macos-fixing-keychain-access/)），VS Code Remote-SSH 里的 Claude Code 扩展也有同样问题（[claude-code #44089](https://github.com/anthropics/claude-code/issues/44089)）。手册 §5.5 已经为 herdr-server 本身解决了这个问题，办法是由 Terminal 登录项拉起它。本报告在每一层都沿用同一思路：需要 GUI 会话的动作，要么交给那个 GUI pane 执行，要么交给已拿到 TCC 授权的常驻应用，要么干脆在笔记本本地完成。

### macOS 27 发布十天后的兼容性快照

macOS 27 已于 2026-09-14 正式发布（[MacRumors](https://www.macrumors.com/2026/09/10/macos-27-golden-gate-release-date/)）。调研时，笔记本上的 `sw_vers` 已经显示 27.0，宿主仍在 26。两台版本不同，对 Apple 原生能力没有影响：高性能屏幕共享只要求两端都是 macOS 14+（[Apple ARD 指南](https://support.apple.com/guide/remote-desktop/use-high-performance-screen-sharing-apdf8e09f5a9/mac)），Apple 的屏幕共享类型页也已覆盖 27（[Apple 使用手册](https://support.apple.com/en-lamr/guide/mac-help/mchl1883115d/mac)）。风险集中在第三方工具上。下表只列有来源的条目；"未见报告"表示本轮没有找到相关资料，不代表确认可用。

| 工具 | macOS 26 Tahoe | macOS 27 Golden Gate | 来源 |
|---|---|---|---|
| 通用控制 | 可用；个别机器"图标闪一下就断"，退出再登录 Apple 账户可修 | 新功能清单里没有 UC 变化 | [Apple Community](https://discussions.apple.com/thread/256222654)、[MacRumors](https://www.macrumors.com/2026/09/17/new-features-changes-macos-27/) |
| 屏幕共享高性能模式 | 可用；26.2 在无人登录时能否共享有争议 | Apple 指南已覆盖 | [Apple Community](https://discussions.apple.com/thread/256211596)、[Apple](https://support.apple.com/en-lamr/guide/mac-help/mchl1883115d/mac) |
| Deskflow 1.26 | 可用；另有 26.3.1 server 偶发崩溃等未修问题 | **会崩溃**，要等基于 Qt 6.11.3/6.12 构建的 1.27 | [#10068](https://github.com/deskflow/deskflow/issues/10068)、[#10086](https://github.com/deskflow/deskflow/issues/10086) |
| Synergy 3.6.3 | 修复 Tahoe 上的重连循环 | 无声明 | [Symless](https://symless.com/synergy/download/release-notes/synergy3-personal/v3.6.3) |
| BetterDisplay 5 | 需要 26.3+ | 支持 | [GitHub](https://github.com/waydabber/BetterDisplay) |
| Cursor Remote-SSH | 每次启动都反复提示安装扩展 | 未见报告 | [Cursor 论坛](https://forum.cursor.com/t/macos-26-remote-ssh-extension-prompt-loop-on-every-startup-anysphere-remote-ssh-installed-but-ms-vscode-remote-remote-ssh-id-checked/160607) |
| macFUSE FSKit 后端 | 26.6.2 上 fskitd 不报告模块 | 27.0 beta 上同样出现 | [macFUSE #1192](https://github.com/macfuse/macfuse/issues/1192) |
| herdr 的 OSC 52 | 超过约 196 KB 时静默丢弃 | 在 27.0 + Ghostty 上复现，官方不修 | [herdr #4194](https://github.com/herdrdev/herdr/issues/4194) |
| terminal-notifier | 不带 `-sender` 参数时通知被静默丢弃 | 未见单独报告 | [Apprise #1705](https://github.com/caronc/apprise/issues/1705) |
| 经 SSH 解锁 FileVault | 26 新增，仅限 Apple silicon | "Tahoe 或更新"均适用 | [Apple](https://support.apple.com/en-us/124963)、[Der Flounder](https://derflounder.wordpress.com/2025/10/11/unlocking-filevault-via-ssh-on-macos-tahoe/) |
| Sunshine（macOS 作主机） | 实验性，M4 + 26.5.2 上光标明显滞后 | 未见报告 | [Sunshine #5441](https://github.com/LizardByte/Sunshine/issues/5441) |
| Mutagen 0.18.1 | 2025-02 之后没有新版本，未验证 | 未验证 | [GitHub](https://github.com/mutagen-io/mutagen/releases) |

结论很直接：宿主暂时留在 26。笔记本已经在 27 上，就先不装 Deskflow 1.26，同桌场景只用通用控制。

## 方向一：键鼠共享只在两块屏幕都在眼前时成立

### 通用控制无需安装，但它根本不走 IP

通用控制的官方要求是：macOS 12.4+；两台登录同一个 Apple 账户并开启双重认证；蓝牙、Wi‑Fi 和接力都要打开；设备相距 10 米以内；一台主 Mac 最多带两台设备；Mac 不能开"互联网共享"。它原生支持跨设备拖放文件和 ⌘C/⌘V（[Apple 支持](https://support.apple.com/en-us/102459)）。它和 AirDrop 用的是同一套近场机制：先用低功耗蓝牙发现设备，再建立**不经过接入点、也不经过互联网**的点对点 Wi‑Fi 连接（[Apple 平台安全](https://support.apple.com/guide/security/airdrop-security-sec2261183f4/web)）。由此可以推出三点。第一，Tailscale 的 100.x 虚拟网卡根本不在这条路径上，宿主在别处时怎么配都用不了。第二，两台连着不同 Wi‑Fi 网络（比如笔记本连访客网络）很可能照样能用，但本轮没有核实。第三，宿主只插网线、关掉 Wi‑Fi 时一定会失败，所以 Mac mini 当宿主时要保持 Wi‑Fi 无线电开着，哪怕不连任何网络。Apple 的排障说明还专门提到要检查 VPN 和安全软件。两台 Mac 都装着 Tailscale 时，如果通用控制不稳，可以先暂停 Tailscale 排除干扰。

在安全和延迟上，通用控制是这一层最好的：由 Apple 账户认证，不用给第三方授予辅助功能权限，也不开任何监听端口。可靠性方面，Tahoe 26.2 上有人报告图标出现约一秒就消失，重置蓝牙也没用，最后的解法是在 Tahoe 那台上退出 Apple 账户再登录，然后重启（[Apple Community](https://discussions.apple.com/thread/256222654)）。macOS 27 的新功能清单里没有提到通用控制，Continuity 方面只说 Sidecar 支持了触控（[MacRumors](https://www.macrumors.com/2026/09/17/new-features-changes-macos-27/)）。它的根本限制在于，通用控制是把指针推到对方的**物理屏幕**上。宿主不接显示器时，你在笔记本上什么也看不到，这时应该改用方向二的屏幕共享。屏幕共享窗口本身就会转发键鼠，通用控制也就用不上了。基于同样的理由，另外三种"把 Mac 当显示器"的思路也可以排除：Sidecar 只能让 iPad 当 Mac 的副屏（[Apple 支持](https://support.apple.com/en-us/102597)）；目标显示器模式只有 2009–2014 年的 iMac 才有（[Apple 支持](https://support.apple.com/en-us/105126)）；AirPlay 接收器只传画面，不回传输入。

### Deskflow 是软件 KVM 的首选，但目前在 macOS 27 上用不了

2026 年开源软件 KVM 的首选是 Deskflow。它采用 GPL-2.0 许可，开发很活跃，**默认开启 TLS 1.3**，支持剪贴板，提供 arm64 安装包，并由 Synergy 官方出资赞助（[Deskflow](https://github.com/deskflow/deskflow)）。安装时有三个坑。官方安装包没有经过公证，需要执行 `xattr -c Deskflow.app`，或者改用 `brew tap deskflow/tap && brew install deskflow` 安装。辅助功能权限要同时授予 `Deskflow` 应用和 `deskflow` 进程。每次升级后都要先删掉旧的辅助功能条目，再重新授权（同上）。v1.26.0（2026-02-16）允许客户端填写一组服务器地址并依次尝试，还修复了 RIME 输入法发错键的问题（[Release](https://github.com/deskflow/deskflow/releases/tag/v1.26.0)）。问题在于，**1.26 在 macOS 27 上会因为 Qt 版本太旧而崩溃**。1.27 要等 Qt 6.11.3（计划 9 月 24 日发布）或 Qt 6.12（计划 9 月 30 日发布）（[#10086](https://github.com/deskflow/deskflow/issues/10086)）。软件 KVM 中，接着物理键鼠的那台机器是 server，在本场景里就是笔记本当 server、宿主当 client。笔记本已经升到 27 的话，Deskflow 目前根本跑不起来。

另外几个已知问题正好落在本场景上。宿主只让显示器休眠时，把指针移过去不会唤醒屏幕，必须按一下键盘（[#10020](https://github.com/deskflow/deskflow/issues/10020)）。Tahoe 26.3.1 上的 Apple silicon server 偶尔会抛出未捕获的异常并崩溃，Barrier 也能复现（[#10068](https://github.com/deskflow/deskflow/issues/10068)）。Mac 当 server 并开着简体拼音输入法时，client 上打不出逗号和句号（[#9465](https://github.com/deskflow/deskflow/issues/9465)）。两台都是 Mac 时，任意一台睡眠后连接就会断开（[#8263](https://github.com/deskflow/deskflow/issues/8263)）。中文用户可以这样规避：server 端保持英文输入源，需要中文时在宿主端切换输入法。还有一个本轮没有直接核实、但出现概率不低的坑：笔记本上开着终端的"安全键盘输入"或正在输入密码时，Deskflow 截获不到按键，就会出现鼠标能移过去、键盘却不起作用的情况。

其他选项都有明显短板。Input Leap 的 GitHub 仓库已经归档，最后一个版本是 2025-06-13 的 v3.0.3（[Input Leap](https://github.com/input-leap/input-leap)）。Barrier 自 2021 年发布 2.4.0 之后就再没有新版本（[Barrier](https://github.com/debauchee/barrier/releases/tag/v2.4.0)）。Synergy 3 是一次性买断：Lite $15，没有 TLS；**Plus $19，才有 TLS 和热键**；Max $29（[Symless](https://symless.com/synergy/purchase)）。它的 3.6.3（2026-06-12）修复了 Tahoe 上的间歇性重启和重连循环（[Release notes](https://symless.com/synergy/download/release-notes/synergy3-personal/v3.6.3)），但还没有 macOS 27 的支持声明，而且和 Deskflow 在网络上不兼容。ShareMouse 官方明确说明不能跨互联网使用，也不支持 VPN 环境（[ShareMouse FAQ](https://www.sharemouse.com/faq/)）；它的免费版没有剪贴板和文件拖放，也不许商用（[ShareMouse](https://www.sharemouse.com/shop/)）。Lan Mouse 直到 2026 年 6 月的 v0.11.0 才加入 DTLS 加密（[Release](https://github.com/feschber/lan-mouse/releases/tag/v0.11.0)），至今仍不支持剪贴板（[README](https://github.com/feschber/lan-mouse)）。

那么软件 KVM 能不能走 Tailscale？Deskflow 用的是 TCP 24800 端口的 client/server 结构，只要两边 IP 互通就能运行。维护者提到，他们做过的中转实验在 ping 低于 20 ms 时效果"pretty well"（[#7439](https://github.com/deskflow/deskflow/issues/7439)）。但人在远处时看不到宿主画面，这项能力就没有意义。它唯一的实际用途是"同桌但不在同一网段"，比如笔记本连着访客 Wi‑Fi、公司做了 VLAN 隔离，或者开着全局 VPN。这时可以在 client 的服务器地址里同时填上局域网 IP 和 MagicDNS 名，再用 Tailscale ACL 只放行 TCP 24800。这样 TLS 再加上 WireGuard，就是两层加密。所有软件 KVM 都依赖登录会话里的辅助功能权限，所以宿主停在登录窗口或 FileVault 解锁界面时，没法通过它们输入。

### 硬件 KVM 切不了笔记本自带的键盘和触控板

硬件 KVM 不需要任何软件，也没有延迟，在登录窗口、FileVault 解锁界面和恢复模式下都能正常工作。但它切换的是**外接**显示器和**外接**键鼠。笔记本内置的键盘和触控板不走 USB 口，所以它满足不了"用笔记本自己的键鼠"这个要求。只有笔记本合盖接外接屏、桌上又只有一块显示器时，才值得买。Mac 用 KVM 还有两个特有的坑。一是 macOS 不支持 DisplayPort MST，接双屏要两个通道，最好选带 EDID 模拟的型号，免得切换时窗口被重新排列（[TESmart](https://www.tesmart.com/blogs/news/facts-to-consider-before-buying-a-kvm-switch-for-your-macbook)，厂商博客）。二是带 Touch ID 的妙控键盘接在 KVM 的 HID 口上时，键盘和指纹都完全不能用（[Level1Techs](https://forum.level1techs.com/t/does-apple-keyboard-with-touchid-work-through-kvm-hid-port/249462)）。Logitech Flow 要求所有电脑在同一网络，不支持跨机拖放文件，在登录界面也不能用（[Logitech](https://support.logi.com/hc/en-us/articles/1500005634742-What-is-Logitech-Flow-and-how-do-I-set-it-up-and-troubleshoot-it)），在两台 Mac 之间用，比通用控制没有优势。反过来，如果想用一套外接妙控配件控制两台 Mac，通用控制本身就是最好的"切换器"：配件只和笔记本配对，由通用控制转发给宿主。

| 方案 | 原理 | 同桌/局域网 | 经 Tailscale | 剪贴板与文件 | 安全 | 26 / 27 | 定位 |
|---|---|---|---|---|---|---|---|
| 通用控制 | BLE 发现 + 点对点 Wi‑Fi | 最佳 | 不可用 | 通用剪贴板 + 拖放 | Apple 账户，无第三方授权 | 可用 / 未见变化 | 首选 |
| Deskflow | TCP 24800 + TLS 1.3 | 好 | 技术上可行，远程无意义 | 剪贴板 | 默认 TLS | 可用 / 1.26 崩溃 | 备选，等 1.27 |
| Synergy 3 Plus | Deskflow 的商业下游 | 好 | 同上 | 剪贴板 | Plus 起才有 TLS | 已修 Tahoe 问题 / 无声明 | 需要商业支持时 |
| ShareMouse | 局域网广播发现 | 好 | 官方不支持 | Standard 版起才有 | AES | [第三方页面](https://sharemouse.macupdate.com/)称支持 26–27 | 可选 |
| Lan Mouse | UDP 4242 + DTLS | 可用 | 技术上可行 | 无剪贴板 | DTLS | 未见报告 | 暂不适合当主力 |
| 硬件 KVM | 物理切换外接设备 | 仅合盖模式 | 不适用 | 无 | 物理隔离 | 与系统版本无关 | 合盖且只有一块屏时 |

## 方向二：Apple 高性能屏幕共享免费拿下 Mac 对 Mac 的最佳画质

### 原生高性能模式要的是带宽，不花钱

两台都是 Apple silicon、都在 macOS 14+ 时，系统自带"屏幕共享"的高性能模式就是 Mac 对 Mac 的最佳选择。它支持 30/60 fps、HDR、4:4:4 色度采样和立体声；Standard 模式则会根据网络状况自动降低分辨率和帧率（[Apple 使用手册](https://support.apple.com/en-lamr/guide/mac-help/mchl1883115d/mac)）。高性能模式可以开 1 或 2 块虚拟显示器，每块最高 4K（3840×2160，或 1920×1080 HiDPI）。Apple 建议用有线网络，单块 4K 需要 **75 Mbit/s** 带宽，双方要能互通 **UDP 5900–5902**。每台 Mac 同一时间只能有一个高性能会话（[Apple ARD 指南](https://support.apple.com/guide/remote-desktop/use-high-performance-screen-sharing-apdf8e09f5a9/mac)）。会话中可以开启"动态分辨率"，让虚拟屏跟着本地屏幕的原生分辨率走（[Six Colors](https://sixcolors.com/post/2026/06/high-performance-mode-allows-sharing-another-macs-display-as-if-your-own/)；[9to5Mac](https://9to5mac.com/2026/02/05/how-to-use-macos-high-performance-screen-sharing-for-lower-latency-and-better-color-video/)）。如果用宿主当前登录用户的身份认证，宿主的物理显示器会被遮黑，本地也无法操作（同 ARD 指南），效果相当于幕帘模式。剪贴板方面有"使用共享剪贴板"选项，但**会话期间通用剪贴板不可用**；文件可以直接拖进共享窗口传输（[Apple 支持](https://support.apple.com/en-au/guide/mac-help/mh14066)）。

它能在 Tailscale 上运行。有用户在 sing-box 的 Tailscale 实现上只能建立 Standard 连接，换成官方 Tailscale 客户端后，高性能模式"完美工作"（[sing-box #4155](https://github.com/SagerNet/sing-box/issues/4155)）。本轮没有找到经 Tailscale 跨城市使用高性能模式的实测延迟和带宽数据，下面是根据协议做的推断。HiDPI 的 1920×1080 实际渲染的是 3840×2160 像素，传输量要按 4K 算。家里上行带宽只有几十 Mbps 时，远程高性能模式会不稳定。`tailscale ping` 显示直连时，它跑在 WireGuard 隧道里，体验接近局域网，只是多了一段公网往返时间；回落到 DERP 中继时，应改用 Standard 模式或 Jump Desktop。安全上，应优先用 macOS 账户认证，只在第三方 VNC 客户端确实需要时才打开"VNC 显示程序可以使用密码控制屏幕"，因为传统 VNC 密码的长度和强度都有限（推断）。版本方面，Apple 社区对 26.2 在没有用户登录时能否屏幕共享有争议，变通办法是开自动登录或插 HDMI 假负载（[Apple Community](https://discussions.apple.com/thread/256211596)）。手册 §5.2.2 已经开启了屏幕共享，`open vnc://mac-mini` 连上后，在类型面板里选 High Performance 即可。

### Jump Desktop 弥补弱网和隐私需求，其余方案各有硬伤

Jump Desktop 是 Mac 对 Mac 最成熟的第三方方案。它使用 Fluid 2.0 协议，支持 4:4:4 10 bit 色彩，并做了端到端加密（[Jump Desktop](https://jumpdesktop.com/)）。厂商称 Fluid 能以 60fps 共享屏幕，所需带宽只有 RDP/VNC 的十分之一（[Jump 支持](https://support.jumpdesktop.com/hc/en-us/articles/216423983-General-Fluid-Remote-Desktop)）。隐私模式会遮黑宿主屏幕、屏蔽本地键鼠，并在断开后自动锁屏（[Jump 支持](https://support.jumpdesktop.com/hc/en-us/articles/18250305826573-Privacy-Mode-for-Fluid)）。会话期间，它可以为无头宿主按需创建虚拟显示器（[Jump 支持](https://support.jumpdesktop.com/hc/en-us/articles/45768818796813-General-Using-the-Display-menu-and-virtual-displays-in-Fluid-sessions)），并默认优先走能到达宿主的 tailnet 网卡（[Jump 社区](https://support.jumpdesktop.com/hc/en-us/community/posts/43160868665741-Tailscale)）。Mac 客户端一次性售价约 $29.99–34.99，不同来源说法不一（[SelectHub](https://www.selecthub.com/p/remote-desktop-software/jump-desktop/)）。值得买它的情况很具体：远程上行带不动高性能模式；需要从 iPad 或 iPhone 接入；需要断开连接就自动锁屏。添加连接条目时手动填 100.x 地址或 MagicDNS 名，不依赖 Jump 的云中转。

其余方案用在本场景都有硬伤。Parsec 早就正式支持 macOS 作主机（[Parsec](https://parsec.app/blog/macos-launch)），但虚拟显示器、多屏和隐私模式都要订阅 Warp（$9.99/月），4:4:4 只支持 Windows 主机，而且必须用 Parsec 账户并经它的云端信令连接（[Parsec Warp](https://parsec.app/warp)）。Sunshine 的文档自己承认 "on macOS is experimental"：手柄不可用，Moonlight 不转发 Command 键（[Sunshine 文档](https://docs.lizardbyte.dev/projects/sunshine/latest/md_docs_2getting__started.html)）。在 M4 + macOS 26.5.2 上，它的光标延迟明显比 Parsec 甚至 TeamViewer 差（[Sunshine #5441](https://github.com/LizardByte/Sunshine/issues/5441)）；它的分支 Apollo 的虚拟显示器只支持 Windows（[Apollo](https://github.com/ClassicOldSong/Apollo)）。RustDesk 打开"直接 IP 访问"后可以按 100.x 地址直连，完全不需要 RustDesk 服务器（[Tailscale 文档](https://tailscale.com/docs/solutions/access-remote-desktops-with-rustdesk)），但 macOS 作主机时可能只能用 VP8/VP9/AV1 软件编码（[RustDesk #9236](https://github.com/rustdesk/rustdesk/discussions/9236)），适合在 Apple 屏幕共享和 Jump 都出问题时兜底。NoMachine 从 2026-07-30 发布的 v10 起取消了免费版，Personal Edition 每台服务器每年 $24.50（[NoMachine](https://www.nomachine.com/what-s-new-in-nomachine)；[Cendio](https://www.cendio.com/blog/nomachine-10-free-edition-linux/)），旧文章里"个人免费"的说法已经过时。Chrome Remote Desktop 在 Big Sur 之后不再支持幕帘模式（[Google](https://support.google.com/chrome/a/answer/2799701?hl=en)）。Apple Remote Desktop 售价 $79.99（[App Store](https://apps.apple.com/us/app/apple-remote-desktop/id409907375)），3.10 版加入了 Tahoe 兼容（[Wikipedia](https://en.wikipedia.org/wiki/Apple_Remote_Desktop)），它的价值在于传统幕帘模式和批量管理，个人只有一台宿主时用不上。向日葵、ToDesk 本轮没有调研。按常见理解，它们和 TeamViewer、AnyDesk 一样，靠厂商的云中继加账户体系工作，这与"只走自己的 tailnet"的思路相反，所以只作对照。

### 无头宿主要么交给会话虚拟屏，要么补一块假显示器

Apple silicon 的 Mac mini 不接显示器时会自动生成一块虚拟显示器，但默认是 1920×1080 非 Retina（厂商博客）（[Astropad](https://astropad.com/blog/dummy-plug-headless-mac-mini/)）。所以用 Standard 模式或第三方 VNC 连上去，看到的是发糊的 1080p 画面。高性能模式、Jump 和付费版 Parsec 都会为会话单独创建虚拟屏，基本不用额外处理。如果希望 Standard 模式和 RustDesk 也清晰，可以用 BetterDisplay Pro（$21.99 买断）建一块常驻的 HiDPI 虚拟屏。注意 **BetterDisplay 5 要求 macOS 26.3+ 或 27**（[BetterDisplay](https://github.com/waydabber/BetterDisplay)）。不想装软件的话，可以插一个 HDMI 假负载。`displayplacer` 可以写进登录脚本，用来固定分辨率和缩放（[displayplacer](https://github.com/jakehilborn/displayplacer)）。常驻虚拟屏和会话虚拟屏同时存在时，窗口可能跑到看不见的屏幕上（推断），所以开高性能会话前可以先关掉常驻虚拟屏。

| 方案 | 画质 | 带宽与延迟 | 无头时的显示 | 遮黑宿主屏 | 按 100.x 直连 | 账户 | 价格 | 定位 |
|---|---|---|---|---|---|---|---|---|
| Apple 高性能 | 2×4K、60fps、HDR、4:4:4 | 单块 4K 约 75 Mbps，UDP 5900–5902 | 会话级虚拟屏 1–2 块 | 以当前登录用户认证时遮黑 | 可以 | 不需要 | 免费 | 首选 |
| Apple Standard | 自适应 | 弱网可用 | 用宿主当前的屏幕 | 否 | 可以 | 不需要 | 免费 | 弱网时回退 |
| Jump Desktop | 4:4:4 10bit、60fps | 厂商称约为 RDP/VNC 的十分之一 | 会话级虚拟屏 | 遮黑并在断开时锁屏 | 可以 | 通常需要 | 约 $30–35 买断 | 弱网或用 iPad 时 |
| RustDesk | 中等（可能只有软件编码） | CPU 占用高 | 否 | 未知 | 可以 | 不需要 | 免费 | 兜底 |
| Parsec | Mac 主机不支持 4:4:4 | 低延迟 | 需订阅 Warp | 需订阅 Warp | 依赖云端信令 | 必须 | 免费 / $9.99 每月 | 不推荐 |
| Sunshine + Moonlight | 取决于宿主屏幕 | 光标滞后 | 无 | 无 | 可以 | 不需要 | 免费 | 不推荐 |
| NoMachine 10 | 中等（软件编码 H.264） | — | 未知 | — | 可以 | 不需要 | $24.50/年 | 不推荐 |
| 向日葵 / ToDesk | 本轮未调研 | 厂商云中继 | — | — | 否（推断） | 必须 | — | 仅作对照 |

## 方向三：远程编辑器让 agent 的改动无需同步就出现在笔记本上

### VS Code Remote-SSH 最成熟，Zed 最能扛网络抖动

VS Code Remote-SSH 会在宿主上安装 VS Code Server。大多数扩展（包括语言服务）在宿主上运行，主题之类只影响界面的扩展留在本地，终端也自动开在远端。官方支持开启了"远程登录"的 macOS 10.14+ 宿主（[VS Code 文档](https://code.visualstudio.com/docs/remote/ssh)）。对"agent 在宿主上写代码、人在笔记本上审阅"的用法来说，它最大的优势是**没有同步层**：文件监听、git 和搜索都在宿主本地执行，agent 一写盘，编辑器里立刻就能看到。VS Code Server 会用交互式 login shell 来解析环境变量，所以 `~/.zprofile` 里配置的 Homebrew PATH 能生效（[VS Code 排障](https://code.visualstudio.com/docs/remote/troubleshooting)），手册 §3.5 的 PATH 设置可以直接沿用。有两个坑和多 agent 直接相关。第一，自动转发的端口超过 20 个时，VS Code 会弹出提示，并把设置切换为 `hybrid`，之后自动转发可能失效（[vscode-remote-release #10926](https://github.com/microsoft/vscode-remote-release/issues/10926)）。宿主上多个 worktree 各自起 dev server 时很容易触发这个问题，建议关掉自动转发，改用方向六的显式转发。第二，Claude Code 的 VS Code 扩展在 Remote-SSH 下因为 Keychain 被锁，每次都要重新登录（[claude-code #44089](https://github.com/anthropics/claude-code/issues/44089)）。所以 agent 继续放在 herdr 里跑，编辑器只负责编辑和审阅。Remote Tunnels 经微软 dev tunnels 中继，要求两端登录同一个 GitHub 或微软账号，每个账号最多 10 条隧道（[VS Code 文档](https://code.visualstudio.com/docs/remote/tunnels)）。已经有 Tailscale 的话，没必要再用它。微软的 Remote 扩展只允许在官方 VS Code 中使用，code-server 和 VSCodium 都用不了（[code-server FAQ](https://coder.com/docs/code-server/FAQ)）。

Cursor 因为用不了微软的扩展，自研了 `anysphere.remote-ssh`。架构和 VS Code 相同，但 2026 年多次出现回归。5 月 22 日自动更新到 1.0.51 后，"Connect to Host"完全失效（[Cursor 论坛](https://forum.cursor.com/t/bug-report-remote-ssh-extension-v1-0-51-breaks-connect-to-host/161341)）。在 macOS 26 上，每次启动都会反复提示安装扩展（[Cursor 论坛](https://forum.cursor.com/t/macos-26-remote-ssh-extension-prompt-loop-on-every-startup-anysphere-remote-ssh-installed-but-ms-vscode-remote-remote-ssh-id-checked/160607)）。多个 agent 同时做大范围代码搜索时，另一个 agent 的 shell 工具会收不到命令完成的信号（[Cursor 论坛](https://forum.cursor.com/t/remote-ssh-agent-shell-tool-loses-completion-under-concurrent-code-search-load/160918)）。Cursor 能用，但要锁定扩展版本，需要长时间运行的 agent 仍交给 herdr。JetBrains 通过 Gateway 连接时，后端只支持 Linux；要以 macOS 为宿主，必须用 **Toolbox App 2.6+** 的远程开发功能（[JetBrains 博客](https://blog.jetbrains.com/toolbox-app/2025/04/toolbox-app-2-6-is-here-with-remote-development-support/)）。它的后端是一个完整的 IDE 进程，和多个 agent 跑在同一台机器上时要预留足够内存。

Zed 的分工方式对不稳定的网络最友好。界面、Tree-sitter 语法解析、模型调用和**未保存的修改**都在本地，源码、语言服务、任务和终端在宿主上。官方支持 Catalina 以上的 macOS 宿主。在命令行执行 `zed ssh://mac-mini/~/src/app` 就能打开项目。`port_forwards` 默认只绑定 localhost。目录里的文件超过 10 万个时性能很差，应该只打开具体的 worktree（[Zed 文档](https://zed.dev/docs/remote-development)）。通过 ACP 接入的外部 agent 在 SSH 断开后会丢失会话线程（[zed #60213](https://github.com/zed-industries/zed/issues/60213)），这也说明 agent 应该留在 herdr 里。浏览器方案 code-server 只能用 Open VSX 上的扩展，webview 要求 HTTPS 安全上下文，在 macOS 上还要给 Node 授予完全磁盘访问权限（[code-server FAQ](https://coder.com/docs/code-server/FAQ)）。让它只绑定 127.0.0.1，再用 `tailscale serve --bg` 暴露出去，正好满足安全上下文的要求（[Tailscale Serve](https://tailscale.com/kb/1312/serve)）。它唯一不可替代的价值，是 iPad 或临时设备不用装客户端就能用。最轻量的做法是直接在 herdr pane 里跑 Neovim、Helix、lazygit 和 delta，不需要任何额外服务，已有 herdr 加 Mac mini 的实践就是这样用的（[DevelopersIO](https://dev.classmethod.jp/en/articles/reona-herdr-remote-mac-mini/)）。代价是每次按键都要走一次网络往返，经中继连接时回显会明显滞后。

所有远程编辑器都是 sshd 的子进程，都受 TCC 限制。手册 §5.4 已经建议把仓库放在 `~/src`，并给"远程登录"开启完全磁盘访问。Apple 的"允许远程用户完全磁盘访问"开关一般只能由坐在宿主前的用户打开（[Apple 支持](https://support.apple.com/en-bn/guide/mac-help/allow-a-remote-computer-to-access-your-mac-mchlp1066/26/mac/26)），属于到场时就要完成的事项。编辑器和 agent 共用宿主磁盘时，有三个冲突点需要预防：编辑器里有未保存的内容，agent 又同时改了同一个文件；编辑器的 git 自动 fetch 与 agent 的 git 操作争抢 `.git/index.lock`；以及前面提到的端口自动转发。对策是每个 agent 一个 worktree，编辑器只打开要审阅的那个 worktree，并关闭 git 自动 fetch。

### 单独转发 macOS 原生应用窗口，2026 年仍没有可用方案

想把宿主上某一个原生 Cocoa 窗口单独、无缝地转到笔记本上，目前没有成熟方案。Xpra 的无缝单窗口模式只对 X11 程序有效，在 macOS 上只提到转发整个桌面（[Xpra](https://xpra.org/index.html)），而且在 macOS 上不支持经 SSH 启动 shadow 服务器（[Xpra 手册](https://xpra.org/manual)）。`ssh -X` 加 XQuartz 只能显示 X11 程序，而原生应用通过 WindowServer 绘制，根本不发 X11 请求。Apple 屏幕共享只能共享整个屏幕（[MacRumors 论坛](https://forums.macrumors.com/threads/mac-screen-sharing-limited-to-specific-app-window.1644755/)）。Jump Desktop 只能按显示器拆分窗口，"只共享单个应用窗口"还停留在社区功能请求阶段（[Jump 更新日志](https://changelog.jumpdesktop.com/)）。微软的 RemoteApp 只支持 Windows 宿主。底层技术其实是有的，ScreenCaptureKit 就能只采集单个窗口（[Apple Developer](https://developer.apple.com/documentation/screencapturekit)），但网络传输和输入注入都要自己实现，目前没有人做成产品。对最常见的 Xcode 和模拟器需求，现实的做法是走无头路线。xcode-remote 在宿主上执行 `xcodebuild`，把 `.app` 经 SSH 送到"你正在看的那台机器"上，再用 `open`、`simctl` 或 `devicectl` 启动，并以 JSON 返回构建状态、日志和崩溃报告（[xcode-remote](https://edc.me/posts/introducing-xcode-remote/)）。agent 也可以通过 MCP 在宿主上启动模拟器、截图和点按（[mcp-ios-simulator](https://github.com/mgcrea/mcp-ios-simulator)）。实在需要看界面时，再开高性能屏幕共享。

| 方案 | 界面在哪 | 宿主要求 | 抗网络抖动 | 与 herdr agent 共存 | 26/27 状态 | 定位 |
|---|---|---|---|---|---|---|
| VS Code Remote-SSH | 本地 | macOS 10.14+，开启远程登录 | 好 | 好；Claude 扩展要反复登录 | 未见 26 的专门问题 | 首选 |
| Zed 远程 | 本地（含未保存的修改） | Catalina+ | 最好 | 好；SSH 断线会丢 ACP 线程 | 1.x 活跃 | 弱网时首选 |
| Cursor | 本地 | 自研 SSH 扩展 | 中 | 多个 agent 并发时丢完成信号 | 26 上反复弹提示 | 锁定版本后可用 |
| JetBrains（Toolbox 2.6） | 瘦客户端 | 支持 macOS 宿主 | 中 | 后端占内存多 | 未核实 | 已是 JetBrains 用户时 |
| code-server + Serve | 浏览器 | Node 需要完全磁盘访问 | 中 | 好 | 未核实 | iPad 或临时设备 |
| herdr 里的 nvim/Helix | 宿主 | 无 | 差（每次按键都要往返） | 最好 | 无额外依赖 | 零配置的小改动 |
| 单窗口转发 | — | — | — | — | 没有成熟方案 | 放弃，改走无头构建 |

## 方向四：文件以 git 为准，其他通道只搬零碎文件

### 近场通道出不了房间，iCloud 不能放仓库

通用剪贴板要求两台用同一个 Apple 账户、打开蓝牙和 Wi‑Fi，并且设备"彼此靠近"，复制的内容只会保留很短时间（[Apple 支持](https://support.apple.com/en-us/102430)）。AirDrop 用 BLE 发现设备，用点对点 Wi‑Fi 传输，全程 TLS 加密，不经过接入点（[Apple 平台安全](https://support.apple.com/guide/security/airdrop-security-sec2261183f4/web)）。两者都不走 Tailscale。同桌时它们是零配置的剪贴板和文件通道；隔着几个房间时时好时坏；出门就用不了了，所以都不能当主通道。如果宿主为 agent 单独建了一个 Apple 账户，或者没有登录 Apple 账户，它们同样用不了。iCloud Drive 在哪里都能用，但开了"优化 Mac 存储空间"后，磁盘紧张时会把本地文件（包括 `.git`）悄悄替换成占位文件（[DEV Community](https://dev.to/ombak/many-mac-developers-misunderstand-does-icloud-secretly-ignore-git-files-the-reality-is-much-3970)），所以只适合放截图、PDF 这类非代码文件。如果宿主开了"桌面与文稿文件夹"同步，放在 `~/Documents` 下的仓库就可能被这样处理。本报告所在的 `~/Documents/herdr` 正好在这个范围内。

### Tailscale 自带的两条文件通道都还是 alpha

Taildrop（`tailscale file cp shot.png mac-mini:`）走加密的点对点连接。截至 2026 年 9 月它仍是 public alpha，只能在自己名下的个人设备之间传文件，**打了 tag 的节点不能用**。macOS 收到的文件放在 `~/Downloads`，并且 macOS 作为接收端时**不支持断点续传**（[Tailscale KB 1106](https://tailscale.com/kb/1106/taildrop)）。接收端可以用 `tailscale file get --wait --loop --conflict=rename <dir>` 持续收件（[Tailscale CLI](https://tailscale.com/kb/1080/cli)）。手册只关闭了宿主的密钥过期（§3.2.4），没有给宿主打 tag，所以 Taildrop 可以用。但 `~/Downloads` 是受 TCC 保护的目录，agent 直接读它会遇到 §5.4 的权限问题，收件目录最好改到 `~/Exchange`。Taildrive 在 `100.100.100.100:8080` 上提供 WebDAV 服务，同样是 alpha。在 macOS 上要先执行一条 `defaults write` 才会显示共享选项，还要在 Tailscale 策略里配置 `drive:share`/`drive:access` 和 `tailscale.com/cap/drive` 授权（[Tailscale KB 1369](https://tailscale.com/kb/1369/taildrive)）。它适合用 Finder 取文件，不适合在上面跑 git 或 npm。

### 同步和挂载只用于交换目录

Syncthing 的官方 macOS 应用从 2026-02-23 起以 v2 为受支持版本，用 `brew install --cask syncthing-app` 安装，原生支持 Apple silicon（[syncthing-macos](https://github.com/syncthing/syncthing-macos/blob/v2/README.md)）。它会把检测到的文件变化**累积 10 秒**再处理，删除操作还要再多等 1 分钟。发生冲突时，它把较旧的一份改名为 `.sync-conflict-…` 副本，并同步到所有设备（[Syncthing 文档](https://raw.githubusercontent.com/syncthing/docs/main/users/syncing.rst)）。传一张截图过去，这个速度够用；但如果两边同时编辑同一个 git 工作树，就会在里面留下一堆冲突副本。所以 Syncthing 只用来同步 `~/Exchange/{shots,specs,logs,artifacts}` 这类交换目录。远程时在设备地址里填 `tcp://100.x.y.z:22000`，可以不依赖公共中继（推断）。Mutagen 有 `two-way-safe`、`two-way-resolved`、`one-way-replica` 等明确的冲突处理模式（[Mutagen 文档](https://mutagen.io/documentation/synchronization)），适合"只在笔记本上编辑、在宿主上构建"的流程。但它从 2025-02-24 的 v0.18.1 之后就没有新版本了（[Mutagen releases](https://github.com/mutagen-io/mutagen/releases)），在 26/27 上使用前要自己测试。何况本场景的代码以宿主为准，基本用不上它。

挂载的原则是只用来浏览，不在上面干活。macOS 自带的 SMB 文件共享可以勾选"仅允许 SMB 加密连接"（[Apple 使用手册](https://support.apple.com/guide/mac-help/set-up-file-sharing-on-mac-mh17131/mac)），只共享 `~/Exchange` 和产物目录即可。git 的 fsmonitor 默认拒绝在网络挂载上工作（[git 文档](https://git-scm.com/docs/git-fsmonitor--daemon)），在挂载盘上跑 `git status`、`npm install` 或让编辑器建索引，都只会带来麻烦。2026 年用 SSHFS 有两条不需要内核扩展的路。一条是 FUSE-T，支持 NFS、SMB，以及 macOS 26 起的原生 FSKit 后端，个人免费使用（[FUSE-T](https://www.fuse-t.org/)）。另一条是 macFUSE 的 FSKit 后端，但只能挂载在 `/Volumes` 下，性能不如 kext 后端（[macFUSE wiki](https://github.com/macfuse/macfuse/wiki/FUSE-Backends)），而且在 26.6.2 和 27.0 beta 上有一个模块无法被识别的未修问题（[macFUSE #1192](https://github.com/macfuse/macfuse/issues/1192)）。rclone 在 macOS 上官方推荐用 `rclone nfsmount`（[rclone](https://rclone.org/commands/rclone_mount/)）。一次性传文件仍然是 `scp` 最可靠。要注意，从 Sequoia 起，系统自带的 `rsync` 换成了只支持部分参数的 openrsync（[Der Flounder](https://derflounder.wordpress.com/2025/04/06/rsync-replaced-with-openrsync-on-macos-sequoia/)），所以两端都要装 Homebrew 的 rsync 3.x，并用 `--rsync-path=/opt/homebrew/bin/rsync` 指定。手册 §5.8.4 提到的 herdrm 自带文件浏览和上传下载，也是一个现成的图形界面入口。

### 剪贴板要按复制方向分别处理

从宿主复制文本到笔记本，标准做法是 OSC 52：pane 里的程序发出一段转义序列，笔记本上的终端收到后写入本机剪贴板。iTerm2 需要勾选 "Applications in terminal may access clipboard"（[iTerm2 文档](https://iterm2.com/documentation-one-page.html)），Ghostty 用 `clipboard-write` 选项控制。经过 herdr 时有一条硬限制：**超过约 196 KB 的 OSC 52 内容会被悄悄丢弃**，没有任何提示和日志。这个问题已在 macOS 27.0 + Ghostty 上复现，官方标为"not planned"关闭（[herdr #4194](https://github.com/herdrdev/herdr/issues/4194)）。所以整段日志或对话记录应该改用 `ssh mac-mini cat file | pbcopy` 或者走文件通道。反方向的"读取剪贴板"要格外小心。herdr 的 `advanced.osc52_paste` 默认关闭（[herdr 配置](https://herdr.dev/docs/configuration/)），应该保持关闭；Ghostty 的 `clipboard-read` 要设为询问或拒绝。否则宿主上的任何程序，包括被提示注入的 agent，都能读走笔记本剪贴板里的密码。从笔记本往宿主传文本，直接在终端里 ⌘V 就行。

图片只能走"先传文件、再贴路径"的方式。这里要修正手册 §3.1.6 的一处建议。herdr-paste-image 插件的做法是：把剪贴板里的图片存到**本机**的 `~/.cache/herdr-paste-image/`，再用 `herdr pane send-text` 把这个**本机路径**输入到 pane 里，中间没有任何上传步骤（[herdr-paste-image](https://github.com/ddfonseca/herdr-paste-image)）。在远程 pane 里，宿主上的 agent 拿到的是一个笔记本上的路径，根本读不到文件。插件在 `--remote` 模式下到底在哪台机器上执行，文档没有说明，需要实测。稳妥的做法是自己加一步 `scp`，并把路径改成宿主上的路径；也可以用 Raycast 的 SSH Image Drop 扩展，一个热键就能经 SSH 把剪贴板里的图片发过去（[Raycast Store](https://www.raycast.com/wjsong/ssh-image-drop)）。在 SSH 里调用 `pbcopy`/`pbpaste`，操作的是宿主自己的剪贴板。Mac 对 Mac 经 SSH 使用时，已经出现过"程序提示已复制，剪贴板实际没变"的情况（[opencode #16962](https://github.com/anomalyco/opencode/issues/16962)），不要依赖它。如果确实需要跨网络的常驻剪贴板同步，有两个选择：ClipCascade 可自建，端到端加密，支持文本、图片和文件（[ClipCascade](https://github.com/Sathvik-Rao/ClipCascade)）；UniClipboard 是点对点的，无需账号，可以走 Tailscale（[UniClipboard](https://github.com/uniclipboard/uniclipboard)）。但它们会把**每一次**复制的内容（包括密码）都推到一台跑着 agent 的机器上，默认不建议安装。

### 通知应在笔记本本地弹出

herdr 客户端的 `ui.toast.delivery` 有四个选项：`herdr`、`terminal`（经外层终端发通知，SSH 下可用）、`system` 和 `off`（[herdr 配置](https://herdr.dev/docs/configuration/)）。手册 §3.6.2 和 §5.9.1 选的是 `system`，这个选项在 macOS 上是通过 terminal-notifier 发通知的。问题在于，在 macOS 26 上，terminal-notifier **不带 `-sender` 参数时通知会被悄悄丢弃**（[Apprise #1705](https://github.com/caronc/apprise/issues/1705)）。笔记本已经在 27 上了，如果发现宿主 agent 的提醒没有弹出来，先怀疑这一点。解决办法是改用 `delivery = "terminal"`，让 Ghostty 来弹通知；或者装一个原生 ntfy 客户端（比如 ntfy-mac），订阅手册 §3.10 里的 topic（[ntfy-mac](https://github.com/jkrumm/ntfy-mac)）。还有一个常见误区：在宿主上调用 `terminal-notifier` 或 `osascript display notification`。这样通知会弹在宿主的屏幕上，而且在 SSH 会话里可能根本不生效。

| 需求 | 同桌/局域网 | 远程经 Tailscale | 首选 | 坑 |
|---|---|---|---|---|
| 代码 | git + 宿主上的 worktree | 同左 | git，不同步 `.git` | iCloud 会驱逐文件；网络盘上 fsmonitor 不工作 |
| 一次性文件 | AirDrop / scp | scp / Taildrop | scp | Taildrop 不能用于带 tag 的节点，macOS 不支持续传 |
| 持续交换 | Syncthing | Syncthing（填 100.x 地址） | Syncthing 同步 `~/Exchange` | 10 秒延迟；冲突时生成副本 |
| 用 Finder 浏览 | SMB（仅加密连接） | SMB / Taildrive | SMB 只共享 Exchange | Taildrive 还是 alpha |
| 文本：宿主 → 笔记本 | 通用剪贴板 / OSC 52 | OSC 52 | OSC 52 | 经 herdr 时上限约 196 KB |
| 文本：笔记本 → 宿主 | 终端里直接粘贴 | 同左 | 终端粘贴 | 不要开 OSC 52 读取 |
| 图片 | 拖放 / 传文件后贴路径 | 传文件后贴路径 | scp 后贴宿主上的路径 | paste-image 插件不会上传 |
| 通知 | herdr toast | herdr toast + ntfy | toast + ntfy | terminal-notifier 在 26 上会丢通知 |

## 方向五：自动化的难点都在"命令在哪个会话里执行"

### 把 GUI 动作交给已授权的会话执行

综合各来源判断，在 SSH 里可以直接用的命令分三类。第一类基本不依赖 GUI：`pmset`、`caffeinate`、`shutdown`、`fdesetup`、`launchctl kickstart` 和 herdr CLI。第二类是 `open -a` 和 `open url`，它们走 LaunchServices，前提是宿主上有人已经登录了 GUI；Tahoe 论坛帖也推荐用它们代替直接执行 app 包里的二进制。第三类需要 GUI 会话或 TCC 授权，在 SSH 里容易失败，或者弹出没人去点的对话框：控制应用的 `osascript`、System Events 界面脚本、`screencapture`、`cliclick`、用 `security` 读 Keychain，以及 `shortcuts run`。`launchctl asuser <uid> <cmd>` 可以把命令送回 GUI 会话执行，但它不会建立 shell 环境，PATH 里的命令必须写绝对路径（[Scripting OS X](https://scriptingosx.com/2018/04/demystifying-root-on-macos-part-3-root-and-scripting/)）；没有人登录 GUI 时它会直接失败。如果坚持在 SSH 里直接截图，就要把"屏幕录制"授权给 SSH 链路上负责的那个进程。但这个进程在不同系统版本上不一样：过去是 `sshd-keygen-wrapper`，Sequoia 起有时变成 `com.apple.sshd-session`。而且从 macOS 15 起，即使授权过，系统仍会周期性弹出重新确认的对话框（[Petrenko](https://jonnyzzz.com/blog/2026/07/16/a-real-macos-desktop-for-a-linux-agent/)）。

所以最干净的办法，是把手册 §5.5 为 Keychain 设计的思路套用到 TCC 上。在 Terminal 登录项拉起的 herdr 里常驻一个名为 `gui-ops` 的 pane。找一次屏幕共享的机会，在这个 pane 里把 `screencapture`、用 `osascript` 控制 System Events、`cliclick` 各跑一遍，把"终端"的辅助功能、屏幕录制和自动化授权一次性点掉。之后在笔记本上用手册 §5.8.3 的 `herdr-mm pane run <gui-ops> "screencapture -x /tmp/s.png"` 发送命令，再用 `scp` 把截图取回来。这条通道本身就带有 SSH 密钥认证和 Tailscale ACL 保护。代价是，任何能访问 herdr socket 的人都等于完全控制了宿主的 GUI 会话，所以笔记本上转发出来的 socket 文件权限必须保持 0600（OpenSSH 的 `StreamLocalBindMask` 默认值是 0177）。这套做法是根据 TCC 按"负责进程"归属授权的规则推断出来的，本轮没有在 Tahoe 上实测，落地前先验证一次。

远程应用脚本（`eppc://`）在 2026 年仍然存在，Apple 的帮助页已经覆盖到 macOS 27（[Apple 支持](https://support.apple.com/en-gb/guide/mac-help/mchlp1398/mac)）。它监听 TCP 3031 端口，用账号和密码认证。Cisco Talos 在 2026 年 4 月把它列为 macOS 原生的横向移动手段（[Talos](https://blog.talosintelligence.com/bad-apples-weaponizing-native-macos-primitives-for-movement-and-execution/)）。`ssh mac-mini osascript -e '…'` 能做到同样的事，还沿用了密钥认证，所以没必要打开远程应用脚本。

### 常驻应用配合笔记本热键，入口只对 tailnet 开放

第二种做法是在宿主上常驻一个已拿到 TCC 授权的应用，开一个只有 tailnet 内能访问的触发入口，再由笔记本上的 Raycast 热键来调用。Hammerspoon 的 `hs.httpserver` 可以用 `setInterface` 绑定到指定的 IP 或网卡，用 `setPassword` 设置 Basic Auth，也可以开启自签名 HTTPS；它默认会做 Bonjour 广播，需要手动关掉（[Hammerspoon](https://www.hammerspoon.org/docs/hs.httpserver.html)）。路由里只写白名单里的动作，比如打开某个应用、聚焦某个窗口、截图、点"允许"，绝不能提供"执行任意 Lua 或 shell"的入口。BetterTouchTool 的 webserver 默认关闭，开启后默认只绑定 127.0.0.1，可以设置一个 shared secret 并通过请求头传递（[BTT 文档](https://docs.folivora.ai/docs/1104_webserver.html)），保持只绑本机、再用 `ssh -L` 转发即可。Keyboard Maestro 要小心：它的 Remote trigger 经 Stairways 的公网服务器中转，而且**没有认证**（[KM Wiki](https://wiki.keyboardmaestro.com/trigger/Remote)）；Public Web trigger（端口 4490/4491）同样不需要密码（[KM Wiki](https://wiki.keyboardmaestro.com/trigger/Public_Web)）。这两个入口只能用来触发"重启某个服务"这类无害动作。更安全的做法是经 SSH 调用：`ssh mac-mini 'osascript -e "tell application \"Keyboard Maestro Engine\" to do script \"<UUID>\""'`（[KM Wiki](https://wiki.keyboardmaestro.com/manual/Scripting)）。笔记本上的 Raycast Script Command 可以绑定全局热键（[Raycast 手册](https://manual.raycast.com/script-commands)）。要聚焦窗口，最干净的是 AeroSpace。它的命令行工具通过 `/tmp` 下的 Unix socket 与应用通信（[AeroSpace](https://nikitabobko.github.io/AeroSpace/guide)），因此 `ssh mac-mini aerospace focus …` 很可能行得通（推断），而且不像 yabai 的部分功能那样需要关闭 SIP。`cliclick` 需要辅助功能授权，不能操作登录窗口（[cliclick](https://github.com/BlueM/cliclick)）。实测它有修饰键卡住不松开的坑，曾因此误触发注销对话框；在 HiDPI 屏幕上，截图里的坐标还要除以 2 才能用来点击（[Petrenko](https://jonnyzzz.com/blog/2026/07/16/a-real-macos-desktop-for-a-linux-agent/)）。能按辅助功能元素的名字点击时，就不要用坐标。

### 让宿主上的 agent 自己看屏幕、点按钮

2026 年最省事的"不开桌面也能做 GUI 操作"的办法，是让 agent 代劳。Anthropic 官方的 computer use 在 Claude Code Desktop 里是 macOS 研究预览功能，需要 Pro 或 Max 订阅，要授予辅助功能和屏幕录制权限，每个应用第一次使用时要单独批准。对浏览器它只能看，对终端和 IDE 只能点击。它的支持矩阵里列出了 SSH 会话（[Claude Code 文档](https://code.claude.com/docs/en/desktop)）。从 2026-09-02 起，它在 Mac 上可以在后台运行，不会抢用户的鼠标指针（[9to5Mac](https://9to5mac.com/2026/09/02/anthropic-upgrades-claude-codes-computer-use-to-run-in-the-background-on-mac/)）。按支持矩阵推断，在 SSH 会话模式下，被控制的是运行 Desktop 的那台机器的屏幕，而不是 SSH 连接的目标机器。所以要控制宿主的桌面，就得在宿主上运行 Desktop，这一点需要实测确认。开源的选择是 Peekaboo。它是一个 CLI 加 MCP 服务器，能截图、分析界面元素、点击、输入、操作菜单和对话框，要求 macOS 15+。它的 Bridge Host 可以让 CLI 借用 Mac App 已有的 TCC 权限（[Peekaboo](https://github.com/openclaw/Peekaboo)）。也可以用能执行 AppleScript 和 JXA 的 macos-automator-mcp（[GitHub](https://github.com/steipete/macos-automator-mcp)）。把这些 MCP 挂到 `gui-ops` 这类由 Terminal 拉起的 herdr pane 里的 agent 上，TCC 权限仍然记在 Terminal 名下。这样笔记本只需要发一句"点掉 Xcode 的权限弹窗，截图给我"。风险也很明确：屏幕内容会被发给模型，也可能带来提示注入。官方明确说明，绕过权限确认的模式只应在容器或虚拟机里使用（[Claude Code 文档](https://code.claude.com/docs/en/desktop)）。让 agent 自动点授权弹窗，等于让它替你做 TCC 决定，所以必须限定在白名单里的应用和按钮上。

### FileVault 远程解锁在 macOS 26 上已经可用，但还有两个问题没解决

手册 §5.2.3–§5.2.4 已经讲了防睡眠、断电后自动开机和 `fdesetup authrestart`，这里补充两个新事实。第一，在 Apple silicon 上，authrestart 由安全隔区签发一个**一次性**令牌，只对这一次重启有效；断电后重新开机，仍然要手动完成 FileVault 登录（[macuser.org.uk](https://macuser.org.uk/2026/07/22/how-to-remotely-reboot-a-filevault-protected-mac-without-entering-a-password/)；[Macworld](https://www.macworld.com/article/2568036/how-to-manage-filevault-to-maintain-remote-access-to-your-mac.html)）。第二，**macOS 26 新增了"FileVault 解锁前也能接受 SSH 登录"**。开启远程登录后，Apple silicon Mac 重启后在数据卷解锁之前就能接受 SSH 密码认证，借此远程解锁（[Apple 企业新功能](https://support.apple.com/en-us/124963)）。由于 OpenSSH 的配置文件放在加密的数据卷上，这时**只能用密码，不能用公钥**。解锁后 SSH 会被短暂断开，等相关服务启动后再重新连接即可（[apple_ssh_and_filevault(7)](https://keith.github.io/xcode-man-pages/apple_ssh_and_filevault.7.html)；[Der Flounder](https://derflounder.wordpress.com/2025/10/11/unlocking-filevault-via-ssh-on-macos-tahoe/)）。有使用者报告，通过 Wi‑Fi 连接时这个功能不可用，改接网线后才稳定（[Jeff Geerling](https://www.jeffgeerling.com/blog/2025/you-can-finally-manage-macs-filevault-remotely-tahoe/)）。

还有两个问题没有答案。一是开源版 `tailscaled` 在数据卷解锁前是否已经运行，官方没有说明。根据它的状态文件存放位置推断，大概率没有运行。这意味着这种远程解锁很可能只能在局域网内完成。二是解锁后会不会自动登录 GUI，也没有权威说明。而手册里 herdr 能自动启动，恰恰依赖 GUI 登录后拉起 Terminal。所以意外重启后，很可能还是需要开一次屏幕共享来登录。

唤醒方面，Wake-on-LAN 的魔术包是二层广播，Tailscale 这种三层网络转发不了。需要在局域网里放一台常开的设备代发，比如装了 etherwake 的树莓派，或者部署 UpSnap 面板（[Tailscale 博客](https://tailscale.com/blog/wake-on-lan-tailscale-upsnap)）。这台设备还可以兼作 FileVault SSH 解锁的局域网跳板。如果只是想点亮休眠的显示器，执行 `ssh mac-mini caffeinate -u -t 5` 就够了（[SS64](https://ss64.com/mac/caffeinate.html)）。

## 方向六：浏览器地址栏里的 origin 决定用哪种方案

### 直连最省事，也最容易出问题

直接访问 `http://mac-mini:5173` 会遇到三个障碍。第一，Vite 默认只监听 `localhost`，要加 `--host` 才能从外部访问。它的 `server.allowedHosts` 默认只放行 `localhost`、`.localhost` 子域和所有 IP 地址。所以 `http://100.x.y.z:5173` 能打开，`http://mac-mini:5173` 却会被"Blocked request"拦下。官方明确警告不要设置 `allowedHosts: true`，否则任何网站都能通过 DNS rebinding 攻击下载你的源码（[Vite 文档](https://github.com/vitejs/vite/blob/main/docs/config/server-options.md)）。第二，Next.js 16 默认拦截跨 origin 访问开发资源。用局域网 IP 访问时，HMR 端点会被拒绝，热更新随之失效，必须把地址加进 `allowedDevOrigins`（[Pramuditya](https://www.pramuditya.com/en/blog/nextjs-16-hmr-breaks-over-lan-ip-alloweddevorigins)；[Next.js 文档](https://nextjs.org/docs/app/api-reference/config/next-config-js/allowedDevOrigins)）。第三，非 localhost 的 HTTP 页面不属于安全上下文，Service Worker 等 API 用不了（[MDN](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts)）。

macOS 15 起还多了一个隐蔽的坑：本地网络隐私权限被拒绝时，进程连接同网段地址只会得到"No route to host"，没有任何权限相关的提示。而 tailnet 的 100.x 地址走 utun 网卡，不受影响；Apple 签名的 `sshd` 也不受这项限制（[vdaluz](https://vdaluz.com/blog/macos-local-network-permissions-no-route-to-host)；[Spexcode #73](https://github.com/shuxueshuxue/Spexcode/issues/73)）。这给手册 §5.3.3 里"在家时走 `.local`"的写法加了一个注脚：系统自带的 `/usr/bin/ssh` 没问题，但宿主上 herdr pane 里的 node、浏览器等访问 192.168.x 地址时可能莫名失败。所以在家也用 MagicDNS 名更省心。

### SSH -L 保住 localhost，Tailscale Serve 服务手机和 HTTPS

`ssh -L 3007:localhost:3007 mac-mini` 让浏览器看到的 origin 仍然是 `http://localhost:3007`。这是唯一一种不改 dev server 任何配置，就能同时满足安全上下文、Vite/Next 默认白名单和 OAuth 回调要求的方案。比如 Google OAuth 要求回调地址必须是 HTTPS，只有 localhost 例外（[Google](https://developers.google.com/identity/protocols/oauth2/web-server)）。配合手册 §3.3 已经配好的 ControlMaster，可以用 `ssh -O forward -L …` 和 `ssh -O cancel -L …` 在现有连接上随时增减转发，不用重新建立连接（[ssh(1)](https://man.openbsd.org/ssh.1)）。

有一点要注意：手册 §3.8.1 的 `new-task` 是在宿主上运行的，而转发只能由笔记本发起。所以正确的接法是由笔记本上的包装脚本从 `new-task` 的输出里解析出 `PORT=`，再执行转发。两端的端口号必须一致。VS Code 发现本地端口被占用时会自动换一个端口（[VS Code 文档](https://code.visualstudio.com/docs/remote/ssh)），一旦 3000 被换成 4123，OAuth 回调和 cookie 就全部对不上了。绑定地址保持默认的 localhost，不要写 `0.0.0.0` 或 `*`，否则宿主的 dev server 会通过你的笔记本暴露给咖啡馆里的整个局域网。

```sh
#!/bin/sh
# ~/bin/pf（笔记本）——用法：pf add 3007 / pf rm 3007，本地与宿主同端口
set -eu
op=$1; p=$2
ssh -O check mac-mini 2>/dev/null || ssh -fN mac-mini     # 沿用 §3.3 的 ControlMaster
case "$op" in
  add) ssh -O forward -L "127.0.0.1:$p:localhost:$p" mac-mini ;;
  rm)  ssh -O cancel  -L "127.0.0.1:$p:localhost:$p" mac-mini ;;
esac
```

Tailscale Serve 是手机访问和需要 HTTPS 时的主力方案。它只在 tailnet 内可访问，自动签发受信任的 `*.ts.net` Let's Encrypt 证书，还会给后端请求加上 `Tailscale-User-Login` 等身份头；不过从打了 tag 的设备发来的请求不带这些头（[Tailscale KB 1312](https://tailscale.com/kb/1312/serve)）。加 `--bg` 参数后配置在重启后仍然保留，`--https=<port>` 可以给每个 worktree 分配一个端口（[Tailscale KB 1242](https://tailscale.com/kb/1242/tailscale-serve)）。这些操作在宿主上执行，可以直接写进 `new-task` 和 `done-task`，例如 `tailscale serve --bg --https=$((port+5000)) localhost:$port` 和 `tailscale serve --https=$((port+5000)) off`。

它的代价有三点。第一，Serve 会把 ts.net 主机名原样传给后端，所以 Vite 的 `allowedHosts` 里要加上 `.tailXXXX.ts.net`（[portless #310](https://github.com/vercel-labs/portless/issues/310)），但不能直接写 `.ts.net`，因为它在公共后缀列表里（[PSL](https://publicsuffix.org/list/public_suffix_list.dat)）；Next 的 `allowedDevOrigins` 也要加上 `mac-mini.tailXXXX.ts.net`。第二，每个 worktree 的端口都要去 OAuth 提供方那里单独登记一条回调地址。第三，开启 HTTPS 后，**机器名会被写进公开的证书透明度日志**（[Tailscale KB 1153](https://tailscale.com/kb/1153/enabling-https)）。

Funnel 会把服务暴露到公网，而且只能用 443、8443、10000 三个端口（[Tailscale KB 1223](https://tailscale.com/kb/1223/funnel)），dev server 永远不要用它。2026-02-19 正式发布的 Tailscale Services 能给服务一个固定的名字，免费版每个用户可以建 10 个，但**宿主必须使用 tag 身份**（[Tailscale 博客](https://tailscale.com/blog/services-ga)；[KB 1552](https://tailscale.com/kb/1552/tailscale-services)）。给宿主打 tag 会让 Taildrop 失效，宿主也不再归属于你的个人账号，对个人宿主来说不划算。

### 按 worktree 分配子域名，换来 cookie 隔离

按端口区分 worktree 有一个常被忽略的缺陷：cookie 不按端口隔离（[RFC 6265 §8.5](https://www.rfc-editor.org/rfc/rfc6265#section-8.5)）。`localhost:3001` 和 `localhost:3002` 同时登录同一个应用时，同名的 session cookie 会互相覆盖。子域名方案天然没有这个问题。portless（npm 版本 0.15.6，尚未到 1.0）会自动分配端口。在 linked worktree 里，它把分支名放在前面，生成 `fix-ui.myapp.localhost` 这样的地址，并用本地 CA 提供 HTTPS；`--tailscale` 模式还能给每个应用分配一个 ts.net 端口（[portless](https://github.com/vercel-labs/portless)）。`*.localhost` 属于安全上下文（[MDN](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts)），Vite 和 Next 也默认放行。难点在于 `*.localhost` 只会解析到本机。笔记本要访问宿主上的子域名，得先执行 `ssh -L 8443:localhost:443 mac-mini`，再打开 `https://fix-ui.myapp.localhost:8443`，还要在笔记本的钥匙串里信任宿主上 portless 的根证书（推断，本轮未实测）。ts.net 证书只签给机器名本身，拿不到 `wt-a.mac-mini.tailXXXX.ts.net` 这种子域名的证书。如果想让手机也能按子域名访问，只能用自己的域名：配置通配 DNS 指向宿主的 100.x 地址，再由 Caddy 申请通配证书。对于以 `.ts.net` 结尾的站点，Caddy 可以直接从本机的 tailscaled 拿证书（[Caddy](https://caddyserver.com/docs/automatic-https)）。

### 通过隧道查看 agent 的浏览器，不必开远程桌面

宿主上的 agent 应该用独立 profile 的浏览器。从 Chrome 136 起，`--remote-debugging-port` 在默认用户数据目录上不再生效，必须同时指定一个非默认的 `--user-data-dir`；自动化场景推荐使用 Chrome for Testing（[Chrome 博客](https://developer.chrome.com/blog/remote-debugging-port)）。Playwright MCP 和 Chrome DevTools MCP 都支持 `--isolated`、`--headless` 和 `--user-data-dir`。Playwright MCP 自己声明"not a security boundary"，它的 `--host` 默认只绑定 localhost（[playwright-mcp](https://github.com/microsoft/playwright-mcp)；[chrome-devtools-mcp](https://github.com/ChromeDevTools/chrome-devtools-mcp)）。想实时查看某个 agent 在浏览器里做什么，可以给它一个带 `--remote-debugging-port=92xx` 的独立 Chrome；笔记本上执行 `ssh -N -L 92xx:localhost:92xx mac-mini`，再在 `chrome://inspect` 的 Configure 里填 `localhost:92xx`（[Chrome DevTools](https://developer.chrome.com/docs/devtools/remote-debugging/local-server)）。这样就能用 DevTools 看到页面画面，必要时还能直接接管，相当于用 DevTools 代替了远程桌面。DevTools MCP 的 `--autoConnect`（Chrome 144+）每次连接都要在**宿主屏幕**上点"允许"，而且连的是宿主用户日常使用的 Chrome（[Chrome 文档](https://developer.chrome.com/docs/devtools/agents/use-cases/auto-connect)），不适合无人值守的宿主。事后复盘可以用 Playwright trace，执行 `npx playwright show-trace trace.zip` 查看（[Playwright](https://playwright.dev/docs/trace-viewer)）。trace 和截图放在 worktree 里，再通过 Syncthing 或 scp 取回本地查看。

| 方案 | 浏览器看到的 origin | 安全上下文 | 需要改 dev 配置 | 多个 worktree | 手机访问 | cookie 隔离 | 暴露范围 |
|---|---|---|---|---|---|---|---|
| 直连 `http://mac-mini:PORT` | `mac-mini:PORT` | 否 | 要（绑定 0.0.0.0、改白名单） | 靠端口区分 | 可以 | 否 | 整个局域网 |
| SSH `-L`，两端同端口 | `localhost:PORT` | 是 | 不用 | 用 `pf add/rm` 动态增减 | 差 | 否 | 仅本机 |
| Tailscale Serve `--https=PORT` | `mac-mini.tailX.ts.net:PORT` | 是 | 要（白名单加 ts.net） | 每个 worktree 一条规则 | 最佳 | 否 | 仅 tailnet |
| portless 子域名 + SSH 443 隧道 | `wt.app.localhost` | 是 | 基本不用 | 按分支名自动生成 | 差（要装 CA） | 是 | 仅本机 |
| portless `--tailscale` | ts.net:8443 起 | 是 | Vite 要手动加 ts.net | 自动分配端口 | 好 | 否 | 仅 tailnet |
| Funnel / ngrok | 公网地址 | 是 | 要 | — | 好 | — | **公网，dev server 禁用** |

## 分层组合：六层各有首选、备选和何时往下一层走

把六个方向合在一起看，是一条"上层能解决就不往下走"的栈。每往下走一层，占用的带宽、注意力和安全面都会增加。能写成命令的事，就留在终端层。要看跨文件的 diff、在多处修改或调试时，用编辑器层。要看页面渲染效果、测试 OAuth 时，用浏览器层。只有必须在某个 macOS 窗口里点一下时，才考虑完整桌面层，而且先试试 `gui-ops` pane 或让 agent 代点。同桌键鼠层和其他层是另一个维度的东西：宿主的显示器就在眼前时，它可以替代完整桌面层，因为直接在宿主屏上操作比开屏幕共享更直接。

| 层 | 首选 | 备选 | 什么时候用这一层 | 衔接 |
|---|---|---|---|---|
| 1 终端层 | herdr `machine add` | 专用远程窗口、socket 转发加 Raycast、herdrm、弱网用 mosh | 派任务、看状态、批审批、读 agent 输出，以及一切能用命令完成的事 | §5.8.1–§5.8.4、§3.2.6 |
| 2 编辑器层 | VS Code Remote-SSH | 弱网用 Zed 远程；herdr pane 里的 nvim/lazygit；iPad 用 code-server + Serve；正式审阅用 GitHub PR 网页 | 审阅 diff、跨文件修改、断点调试 | §3.5（PATH）、§5.4（TCC） |
| 3 文件层 | git（以宿主为准）加 scp | Syncthing 同步 `~/Exchange`、Taildrop、SMB 只读浏览、herdrm 上传下载；剪贴板用 OSC 52，通知用 toast 加 ntfy | 截图、spec、日志、构建产物；给 agent 发图片 | §3.1.6（需修正）、§3.6.2、§3.10 |
| 4 浏览器/预览层 | SSH `-L` 两端同端口 + ControlMaster | 手机和 HTTPS 用 Tailscale Serve 按端口分配；需要 cookie 隔离时用 portless 子域名；查看 agent 浏览器用 CDP 隧道；事后复盘用 Playwright trace | 看 dev server、测 OAuth 回调、查看 agent 的浏览器 | §3.8.1、§3.8.3 |
| 5 完整桌面层 | Apple 屏幕共享高性能模式（1 块虚拟屏） | 先试不开桌面的办法（`gui-ops` pane、Hammerspoon、Peekaboo agent）；弱网或用 iPad 时用 Jump Desktop；RustDesk 直连 IP 兜底；Standard 模式 | 处理 TCC 授权弹窗、首次登录、系统更新、查看模拟器和原生应用界面、意外重启后登录 | §5.2.2、§5.5.5、§7.3 |
| 6 同桌键鼠层 | 通用控制 | Deskflow（macOS 27 上等 1.27）、Synergy 3 Plus；合盖只接一块屏时用硬件 KVM | 宿主显示器就在旁边，想直接在宿主屏上操作 | — |

安全底线也可以用一句话概括：所有入口只绑定 localhost 或 tailnet 地址，身份一律复用 SSH 密钥和 Tailscale。按这条原则，dev server 不用 Funnel 或 ngrok，Vite 不设 `allowedHosts: true`，不开远程应用脚本，不用 Keyboard Maestro 的 Remote trigger 做敏感动作，CDP 和 Playwright MCP 只经隧道访问，OSC 52 只允许写、不允许读。

## 三种情形的推荐组合与落地清单

### 同桌：通用控制加 herdr，几乎不用配置

宿主接着自己的显示器，摆在笔记本旁边。笔记本屏幕上仍然用 herdr 和远程编辑器处理日常工作。需要在宿主上点点按按时，把指针直接推到宿主屏幕上。宿主如果没接显示器，即使就在同一张桌上，也请按下一节的"不同房间"处理。

| 步骤 | 做什么 | 命令或位置 | 依据 |
|---|---|---|---|
| 1 | 两台登录同一个 Apple 账户（开启双重认证），都打开蓝牙、Wi‑Fi 和接力。宿主即使走网线，也要保持 Wi‑Fi 无线电开着；不要开"互联网共享" | 系统设置 → 通用 → 隔空投送与接力 | 方向一 |
| 2 | 在笔记本的"显示器"设置里启用通用控制，按实际摆放位置排列宿主屏幕；拖一个文件、做一次 ⌘C/⌘V 来验证 | 系统设置 → 显示器 | 方向一 |
| 3 | 如果连上又断开：在 Tahoe 那台上退出再登录 Apple 账户，然后重启；确认"辅助功能 › 缩放"里没有开启"放大时停用通用控制"；还不稳定就临时暂停 Tailscale | — | 方向一 |
| 4 | 终端和编辑器照常用状态层：herdr `machine add`、VS Code Remote-SSH。同桌时 Tailscale 会自动走局域网直连 | §5.8.1 | 方向三 |
| 5 | 文件和剪贴板优先用近场功能：跨屏拖放、通用剪贴板、AirDrop。给 agent 的图片仍要放到宿主上的路径 | — | 方向四 |
| 6 | 只有需要热键切换、或两台不在同一网段时，才装 Deskflow：笔记本当 server，宿主当 client，服务器地址填局域网 IP 和 `mac-mini`。笔记本已在 27 上时，等 1.27 发布或改用 Synergy 3 Plus | `brew tap deskflow/tap && brew install deskflow` | 方向一 |
| 7 | 宿主显示器休眠时，先唤醒再移指针过去 | `ssh mac-mini caffeinate -u -t 5` | 方向五 |
| 8 | 只有笔记本合盖、桌上只有一块外接屏时，才考虑带 EDID 模拟的 USB‑C KVM；Touch ID 键盘不要接 KVM 的 HID 口 | — | 方向一 |

### 同一局域网不同房间：用屏幕共享代替键鼠共享

宿主的画面不在眼前，键鼠共享也就没有用武之地。带宽充足，是高性能模式发挥最好的场景。

| 步骤 | 做什么 | 命令或位置 | 依据 |
|---|---|---|---|
| 1 | SSH 和 herdr 统一用 MagicDNS 名。`tailscale ping mac-mini` 应显示经 192.168.x 直连；在家也不必切换到 `.local` | §5.3.3、§3.2.2 | 方向六 |
| 2 | 编辑器：用 VS Code Remote-SSH 打开宿主上的 worktree，关掉端口自动转发和 git 自动 fetch；或者用 `zed ssh://mac-mini/~/src/app` | VS Code 设置 | 方向三 |
| 3 | 浏览器：在笔记本上放好 `~/bin/pf`，从 `new-task` 的输出里取出 `PORT` 后执行 `pf add`；收尾时执行 `pf rm` | §3.8.1、§3.8.3 | 方向六 |
| 4 | 完整桌面：`open vnc://mac-mini` → High Performance → 1 块虚拟屏。宿主尽量走网线；用登录用户的身份认证时，宿主屏幕会被遮黑 | §5.2.2 | 方向二 |
| 5 | 第一次屏幕共享时，在 Terminal 拉起的 herdr 里建一个 `gui-ops` pane，把"终端"的辅助功能、屏幕录制、自动化授权一次性点掉 | §5.5、§5.8.3 | 方向五 |
| 6 | 两端都装 Syncthing，只同步 `~/Exchange/{shots,specs,logs,artifacts}`；把笔记本的截图保存位置改到交换目录 | `brew install --cask syncthing-app`；`defaults write com.apple.screencapture location ~/Exchange/shots` | 方向四 |
| 7 | 剪贴板：Ghostty 允许 `clipboard-write`，`clipboard-read` 设为询问；herdr 的 `osc52_paste` 保持关闭；贴图改成先 scp、再贴宿主路径 | §3.1.6（修正） | 方向四 |
| 8 | 通知：确认 toast 能弹出来；如果不弹，改成 `delivery = "terminal"`，或者装 ntfy-mac | §3.6.2 | 方向四 |
| 9 | 可选：用 Raycast 热键调用宿主上的 Hammerspoon（只绑定 tailnet 地址并设密码），完成打开应用、聚焦窗口、截图等操作 | — | 方向五 |

### 远程经 Tailscale：先确认直连，再决定要不要开桌面

远程时，编辑器层和终端层几乎感觉不到差别。受影响最大的是完整桌面层和电源管理。

| 步骤 | 做什么 | 命令或位置 | 依据 |
|---|---|---|---|
| 1 | 出门前确认 `tailscale ping mac-mini` 能建立直连。如果走的是 DERP 中继，回家后在路由器上开 UPnP，或放行 UDP 41641 | §3.2.2、第八部分 | 方向二、三 |
| 2 | 终端层不变，网络差时用 mosh；编辑器优先用 Zed 或 VS Code，少在 herdr 里用 Neovim 做大段编辑 | §3.2.6 | 方向三 |
| 3 | 笔记本上预览照旧用 `pf`。手机预览时，在宿主上执行 `tailscale serve --bg --https=$((PORT+5000)) localhost:$PORT`；Vite 的 `allowedHosts` 加 `.tailXXXX.ts.net`，Next 的 `allowedDevOrigins` 加 `mac-mini.tailXXXX.ts.net`；OAuth 回调按端口逐个登记 | 宿主 `new-task`/`done-task` | 方向六 |
| 4 | 完整桌面：连接是直连、上行带宽也够时，仍然用高性能模式（1 块 HiDPI 屏按 4K 的 75 Mbps 估算）；不够时改用 Standard 模式，或 Jump Desktop（条目填 100.x 地址或 MagicDNS 名） | — | 方向二 |
| 5 | 文件：Syncthing 的设备地址填 `tcp://100.x.y.z:22000`；一次性文件用 scp，或者 `tailscale file cp`（前提是宿主没打 tag） | — | 方向四 |
| 6 | 通知：沿用 §3.10 的 ntfy，或者在 tailnet 里自建 ntfy；笔记本用原生客户端订阅 | §3.10 | 方向四 |
| 7 | 计划内重启一律用 `sudo fdesetup authrestart`。为意外断电准备一台局域网常开设备（比如树莓派装 Tailscale），用来发 WoL 和用密码 SSH 解锁 FileVault，之后再屏幕共享登录一次，拉起 Terminal 和 herdr。这套流程要先在家演练一遍 | §5.2.4、§7.3 | 方向五 |
| 8 | 安全底线：不用 Funnel 或 ngrok 暴露 dev server；不开远程应用脚本；不用 Keyboard Maestro Remote trigger 做敏感动作；CDP 和 Playwright MCP 只经隧道访问 | — | 方向五、六 |

## 结论

这次调研之后，问题本身变了。"不频繁切换键鼠"只有在同桌时才是真问题，而通用控制几乎已经完美解决了它。离开那张桌子后，真正缺的是"不开整屏共享也能完成 GUI 操作"的能力。答案不是某款远控软件，而是 herdr 本身。由 Terminal 拉起的 herdr-server 原本是为了绕开 Keychain 限制而设计的，顺带也成了宿主 GUI 会话里一个已获授权的执行代理。再配上 Peekaboo 这类能让 agent 看屏幕、点按钮的 MCP，屏幕共享就可以降级为只在首次授权和意外重启时才用的维修通道。手册让宿主装开源版 `tailscaled`，这正是 Tailscale SSH 和 Serve 文件服务在 macOS 上的前提；宿主没打 tag，又是 Taildrop 的前提。以后要避免为了用 Tailscale Services 去给宿主打 tag，否则 Taildrop 这条文件通道就没了。

对现有手册来说，有两处需要修订，还有五个问题需要实测。需要修订的是：herdr-paste-image 在远程 pane 里只会贴出笔记本上的路径，并不会上传图片；`delivery = "system"` 依赖的 terminal-notifier 从 macOS 26 起可能悄悄丢掉通知。需要实测的是：FileVault 经 SSH 解锁后会不会自动登录 GUI，以及 `tailscaled` 在解锁前是否已经在线；高性能屏幕共享走 DERP 中继时能不能用；插件在 `--remote` 模式下到底在哪台机器上执行；在 SSH 里执行 `pbpaste`，有没有 GUI 登录时的行为分别是什么；以及让 `gui-ops` pane 复用 Terminal TCC 授权的做法在 Tahoe 上是否成立。在 Deskflow 1.27 发布、各第三方工具给出 macOS 27 支持声明之前，风险最低的组合是：宿主留在 26，已经在 27 上的笔记本只用 Apple 原生能力。
