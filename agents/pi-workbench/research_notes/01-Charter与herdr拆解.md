# Charter 与 herdr 源码级拆解

> 调研日期：2026-10-12。方法：两个仓库都 shallow clone 下来读了源码。

**关键发现：Charter 本身就是基于 pi 的多 agent 工作台。** 它的 npm 包名是 `pi-ide`，依赖 `@earendil-works/pi-coding-agent@0.82.1`。它还直接移植了 herdr 的状态检测规则（`builtin-agent-lifecycle-manifests.json` 里写着 `"sourceRevision": "herdr-ddffb6e1"`），仓库里另有一份 1059 行的 `docs/HERDR_FEATURE_ADOPTION_PLAN.md`。

两者的分工可以概括为：herdr 提供运行时和状态感知，Charter 提供任务、证据和编排语义。

---

## 一、Charter（longyunfeigu/Charter）

**基本事实**

- 定位：本地优先的 Electron 桌面 IDE，口号 "Observable agent work, from prompt to proof"。
- 技术栈：TS + Electron 43 + React 19，MIT 许可。
- 规模和活跃度：10 star、3 fork。2026-07-14 创建，最后 push 在 2026-08-20，已停更约 7 周。85 个 commit（作者 77 个，claude 8 个）。
- 最新版本 v1.0.0-beta.6，未签名，仍是开发预览。
- 文档很完整：59 篇 ADR、产品规格，以及 e2e、安全、性能三类测试门禁。

**架构（六层平面）**

- Renderer 跑在沙箱里，经 Preload 和带版本、经 schema 校验的 IPC 进入 Main。
- Main 持有：SQLite 证据账本、Task Engine、Tool Gateway、权限引擎，以及 Git、搜索、终端、验证等服务。
- 独立的 Agent Worker 进程只跑模型回路。
- 终端由一个 daemon 持有 PTY，Main 受控重启后可以回放屏幕。
- monorepo 包括：`agent-runtime-pi`、`tool-gateway`、`orchestration-domain`、`terminal-service`、`ssh-service`、`persistence`。

**控制 agent 的三条路径**

1. **自带的 Charter Agent（基于 pi SDK）**
   - 用 `createAgentSession({noTools:'all', customTools:[...]})` 关掉 pi 内置的 read、bash、edit、write，只注入网关代理工具。
   - 工具调用经 MessagePort 回到 Main，按 R0–R4 风险等级判断后执行，并写入账本（ADR-0001/0002）。
2. **外部 CLI 走原生 PTY（默认，ADR-0049）**
   - 运行用户自己装的真实可执行文件，不做同名包装。
   - 每个 agent 有一份 manifest，内容包括：
     - 可执行文件的发现路径
     - 启动关卡的识别文字（如信任目录、更新提示），并自动按键通过
     - 是否需要 bracketed paste
     - 会话参数：`--session-id` 预分配、`--resume`、`--continue`
     - 退出序列和能力声明
   - 第三方 agent 可在 `~/.charter/agents/*.json` 里声明。
3. **ACP**
   - 原本是默认路径，后来降级成 `PI_IDE_ACP=1` 才开启的兼容层。
   - 降级原因：token 和工具事件全部走持久化加 IPC，性能太差。

**状态检测与编排**

- 前台进程约每 700ms 轮询一次，判断进入或退出 agent。
- 进入 agent 时，用 git 临时索引执行 `write-tree` 拍一张入口快照；配合文件 watcher 记录逐文件 baseline。这样外部 CLI 做的改动也能逐字节回滚。
- 状态（working / blocked / idle）用的是移植过来的 herdr 规则，并区分来源：structured、observed、unknown。
- **数据面与控制面分离**：
  - 数据面：PTY 字节直接送进 xterm，带 ACK 流控，不进数据库。
  - 控制面：本地 socket 上的 `charter orchestration ...` CLI，共 22 条命令（delegate、message、request、join、park、continue、complete、retry、reassign 等），另配一个 Skill 教模型怎么用。
- 消息只用门铃通知，到 turn 边界才注入。`park` 加 continuation 可以持久化等待条件，等待期间不占用 agent turn（ADR-0050）。
- 编排深度上限为 2。每个 PTY 有独立令牌 `CHARTER_CTL_TOKEN`，用来审计"谁指挥了谁"。
- 默认限额：每个协调者最多 5 个 worker、最多 30 次 send/分钟。用户在 worker 终端里按键即视为接管。
- **完成要靠证据**：必须显式调用 `complete` 并附带证据，状态变为空闲不算完成。

**任务模型与周边**

- Session = 项目 + agent + worktree + 对话 + 计划 + 证据 + review + replay + memory。
- 执行层之上有 Work 看板（Inbox / In progress / Waiting / Review / Done），可导入 GitHub issue。
- 隔离：每个任务执行 `git worktree add -b charter/<taskId>`。Accept 时做文件级 merge-back 并检查冲突；回滚就是丢弃 worktree。
- 其他能力：SSH 远程 worker、Preview（只认任务进程树里的 loopback 端口，支持选取页面元素回传给 agent）、把 review 反馈沉淀成 `.charter/rules.md` 并投射到 CLAUDE.md / AGENTS.md。

**评价**

- 优点：
  - 证据和审计模型最完整。
  - 数据面与控制面分离讲得最清楚。
  - 用 manifest 抽象 agent。
  - 是"pi 加收窄后的工具面"的现成参考实现。
- 缺点：
  - 太重（Electron，1284 个文件）。
  - 只有一个人维护，已停更约 7 周。
  - 编排依赖模型遵守 Skill 的指引。
  - Electron Main 退出后，任务不再后台运行。

**值得借鉴的设计（按优先级）**

1. agent manifest：启动关卡、resume 参数、退出序列。
2. PTY 字节流和控制记录分成两个平面。
3. turn 边界门铃加 park/continuation 式的持久等待。
4. 每个 PTY 一个身份令牌，加深度上限和调用限额。
5. 用临时索引快照，让外部 CLI 的改动也可回滚。
6. pi 用 `noTools` 关掉内置工具，再注入网关工具。
7. 状态不等于完成证据。

---

## 二、herdr（herdrdev/herdr）

**基本事实**

- 上游已从 ogulcancelik/herdr 迁到 herdrdev/herdr，官网 herdr.dev。
- Rust 单二进制，Apache-2.0。
- 43.6k star、3.4k fork、334 个 open issue。
- 2026-03-27 创建，最新稳定版 v0.9.3（2026-09-29），预览版每天发布。项目所属公司是 YC 公司。
- 本地手册写的是 v0.9.1、"约 4 万 star"，需要更新。

**架构**

- 常驻的 herdr-server 持有所有 PTY 和子进程；TUI 客户端只负责显示，可以同时连多个。
- 通信走 Unix socket（Windows 上是命名管道），协议是按行分隔的 JSON。
- 0.9 起改为客户端渲染，一个窗口可以接入多台 SSH 机器（`herdr machine add`、`herdr --remote host`）。
- 层级：Session → Workspace → Tab → Pane。agent 是对 pane 当前占用进程的识别结果，不是一个容器。

**状态检测（按优先级）**

1. **集成上报**
   - 用 `pane report-agent --state working|blocked|idle --seq` 上报，seq 单调递增，防止乱序。
   - 走这条路的有 pi、OMP、OpenCode、Kimi、Kilo、MastraCode。
   - pi 集成是一个扩展，放在 `~/.pi/agent/extensions/herdr-agent-state.ts`，订阅 `session_start`、`agent_start`、`agent_settled` 和自定义的 `herdr:blocked`。
   - Codex 只有在 `--no-daemon` 模式下才能通过 hooks 上报 turn 的起止。
2. **前台进程识别加屏幕规则**
   - 用 `tcgetpgrp` / `proc_pidinfo` 判断前台进程，轮询间隔约 100–800ms。
   - 屏幕规则是 TOML，带优先级。匹配区域包括 `osc_title`、`bottom_non_empty_lines(n)`、`after_last_horizontal_rule`；匹配方式有 contains、regex、any、not。
   - Claude Code 的状态完全靠这一路判断，它的集成只用 `SessionStart` hook 上报 session id。
   - `blocked` 判定刻意保守，拿不准时退回 idle。
   - 规则会从 herdr.dev 热更新；本地可在 `~/.config/herdr/agent-detection/<agent>.toml` 覆盖。
   - `herdr agent explain` 可以解释某个状态是怎么判出来的。
3. **状态集合**：working / blocked / done（已完成但还没看）/ idle / unknown。状态逐级汇总到 pane、tab、workspace、machine。

**持久化**

- detach 后进程照常运行，这是最强的一档。
- server 重启时从 `session.json` 恢复布局和 cwd，保留 3 份损坏备份和 48 份布局快照。
- 屏幕回放默认关闭，因为可能泄露密钥。
- 原生会话恢复：集成会上报恢复命令，如 `claude --resume <id>`、`pi --session <id>`。
- `update --handoff`（实验性）尝试在升级 server 时把进程活着交接过去。

**CLI / API**

- 资源管理：workspace、tab、pane；`worktree` 会以带 git 信息的 workspace 形式创建或打开；layout 可导出和应用。
- pane 操作：`pane run`、`send-text`、`send-keys`、`read`（可读 visible、recent、recent-unwrapped、detection），以及 `wait-output`。
- agent 操作：
  - `agent start --kind codex`：等 agent 就绪后才返回。
  - `agent prompt --wait`：如果 agent 处于 blocked，拒绝发送。
  - `agent wait --until blocked|done`：由 server 事件驱动，等待期间锁定 pane 的占用进程。
  - 另有 `agent read`、`agent explain`、`agent attach --takeover`。
- `events.subscribe`：订阅 `pane.agent_status_changed`、`pane.output_matched` 等事件。
- `herdr api schema --json` 可导出完整 schema。
- 插件就是任意可执行文件加一份 `herdr-plugin.toml`，可调用整个 CLI。没有审核，也没有沙箱。

**通知**

- 通过 `[ui.toast] delivery = herdr | terminal | system | off` 配置。选 terminal 时由外层终端发通知，SSH 下也能用。
- 声音分"完成"和"需要输入"两类，可以按 agent 单独开关。

**局限**

- 屏幕规则天生脆弱，agent 改一次 UI 就得追着改规则。
- 没有任务图，也没有共享记忆。
- worktree 只是被打开成一个 workspace，没有隔离保障，也没有 review 或回滚。
- 多机统一只做到 TUI 层；CLI 和 API 每次只能针对一台 server。
- 在 pane 里再套 tmux 会让识别失效。
- 主机重启后进程丢失。

**值得借鉴的设计**

1. 由 server 持有 PTY，客户端可以随时断开。
2. 两级状态源：结构化上报优先，屏幕规则兜底，并提供 explain。
3. 把 done 和"已查看"分开，状态向上汇总。
4. `agent prompt/wait` 的语义细节：bracketed paste、blocked 时拒发、超时不等于没送达、等待时锁定占用进程。
5. 上报 resume argv，让会话可以恢复。
6. 注入 `HERDR_PANE_ID`、`HERDR_SOCKET_PATH`，让 agent 能反过来驱动工作台。
7. 上报带 seq，防止乱序。

## 来源

- https://github.com/longyunfeigu/Charter（docs/adr/ADR-0001、0002、0009、0017、0044、0049、0050、0051、0058；docs/design/mission-fabric-v3.md；docs/HERDR_FEATURE_ADOPTION_PLAN.md）
- https://github.com/herdrdev/herdr（src/detect/manifests/*.toml；src/integration/assets/pi/herdr-agent-state.ts）
- https://herdr.dev/docs/agents/ 、/session-state/ 、/add-herdr-support/ 、/agent-automation/ 、/socket-api/ 、/integrations/ 、/connecting-machines/
