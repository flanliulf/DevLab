# 基于 pi 的个人多 agent 工作台：方案设计

> 状态：方案（2026-10-13）。依据见 `research_notes/` 下的五份笔记：
>
> - [00 基座选型](research_notes/00-基座选型-pi与DSH等对比.md)
> - [01 Charter 与 herdr 拆解](research_notes/01-Charter与herdr拆解.md)
> - [02 开源全景](research_notes/02-多agent工作台开源全景.md)
> - [03 接入层](research_notes/03-agent控制协议与pi接入层.md)
> - [04 pi-harness 拆解](research_notes/04-pi-harness拆解.md)（2026-10-13 补充）

## 1. 诉求

| # | 诉求 | 参照对象 |
|---|---|---|
| R1 | 同时开多个 Claude Code、Codex、pi 会话，统一看状态，随时切换或接管 | herdr |
| R2 | 一个主 agent 把子任务派给 Claude Code、Codex 等，拿回结构化结果 | DSH 子代理 |
| R3 | 任务级编排：worktree 隔离、多个 agent 同题比较、交叉 review、完成后附证据 | pi-harness、Charter，以及 Orca、Codeg、Agent Orchestrator |
| R4 | 以 pi 为核心，保持轻量、可升级、可以自己改 | — |

## 2. 关键判断

1. **市面上已经有人做过"pi + herdr + 编排"，就是 Charter。** 它的 npm 包名是 `pi-ide`，移植了 herdr 的状态规则。但它把 PTY 管理、窗口、IDE 全部用 Electron 重写了一遍，结果体量太重，一个人维护不动，目前只有 10 star，已停更约 7 周。**结论：参考它的语义设计，不要重走它重写运行时的路。**
2. **R1 不需要自己写。** herdr 有 43.6k star，YC 公司在维护，几乎每天发版。它已经提供了：
   - server 托管 PTY
   - 读屏规则加结构化上报的两级状态检测
   - `agent start/prompt/wait` 和事件订阅等完整的 socket/CLI API
   - 现成的 pi 集成

   你的[实施手册](../herdr/herdr-完整实施手册-最终版.md)也已经写好了。用 Node 重写一个 PTY daemon 加 TUI 复用器，投入产出很差。
3. **"控制外部 agent"有两种互补的方式，两种都要：**

   | | **可见 worker**（PTY，走 herdr） | **headless worker**（结构化协议，走 ACP / 原生协议） |
   |---|---|---|
   | 体验 | 原生 TUI，随时可以人工接管 | 看不到原生界面，由 pi 渲染轨迹 |
   | 结果 | 只能读屏或看 git diff | 有消息、工具调用、stopReason 等事件 |
   | 权限审批 | 在 agent 自己的界面里 | 转交给 pi 的 UI，或按策略自动处理 |
   | 订阅计费 | 跑的是未修改的官方 CLI，最稳妥 | 底层仍是官方二进制，个人本机使用属于灰区，相对安全 |
   | 适合 | 长任务、需要人盯、探索性工作 | 短而明确的子任务、扇出比较、交叉 review |

   herdr 只能做左边这一列，DSH 只能做右边这一列。Charter 两列都做了，但做得很重。**我们的做法：pi 做中枢，左边交给 herdr，右边用 ACP。**
4. **pi 内部的编排和治理也有人做过，就是 pi-harness。** 它用 pi SDK 在进程内跑多个 pi 会话，实现了任务 DAG、确定性调度、评审角色、policy、checkpoint、run 对比，还能把 Claude Code、Codex 的本地历史交接给 pi 继续。但它不驱动任何外部 agent，和 Charter 正好互补：
   - Charter 擅长驱动外部 CLI；
   - pi-harness 擅长 pi 内部的编排与治理。

   两者都是一个人维护的 Electron 应用，这再次说明：**用扩展加现成运行时组合起来，比自己做一个大应用更可持续。** 另外，pi-harness 使用 AGPL 许可，只借鉴设计，不复制代码。
5. **所有能力都以 pi 扩展的形式实现，不改 pi 核心。** 打包成一个个人用的 pi package，这样 pi 升级时自己的代码不受影响。

## 3. 总体架构

```
          ┌───────────────────────── 你（终端 / herdr TUI / 手机 SSH）─────────────────────────┐
          │                                                                                    │
┌─────────▼───────────────────────────────────────┐        ┌────────────────────────────────────▼──────┐
│  pi（编排者，自己一个 herdr pane）                 │        │  herdr server（PTY 托管、窗口、状态、通知） │
│  ─ 个人 pi package「workbench」                   │ socket │   ├─ pane: claude（worktree A）            │
│    ├─ ext/herdr-bridge   可见 worker 控制  ───────┼───────►│   ├─ pane: codex （worktree B）            │
│    ├─ ext/delegate       headless worker（ACP）   │  CLI   │   ├─ pane: pi    （本编排者）               │
│    ├─ ext/tasks          任务账本 + complete 证据  │◄───────┤   └─ events: agent_status_changed …       │
│    ├─ ext/arena          扇出比较 / 交叉 review    │ events └────────────────────────────────────────────┘
│    ├─ ext/handoff        跨 agent 历史交接         │
│    ├─ ext/guard          policy + checkpoint       │
│    └─ ext/panel          worker 面板 overlay       │
└───────┬─────────────────────────────────────────┘
        │ ACP (stdio JSON-RPC)
        ├── claude-agent-acp ──► 官方 Claude Code 二进制（本机 claude login）
        ├── codex-acp ─────────► codex app-server（ChatGPT 登录）
        ├── gemini --acp
        └── pi-acp（pi 递归当 worker）
```

**职责划分**

| 层 | 归谁 | 说明 |
|---|---|---|
| PTY 托管、窗口布局、detach/重连、多机 | **herdr** | 不自己实现 |
| agent 状态（working/blocked/done/idle） | **herdr** | pi 的状态由 herdr 自带的 pi 扩展上报；Claude 和 Codex 的状态靠 herdr 读屏规则判断 |
| 通知 | **herdr**（toast/声音），pi 侧只补充任务级通知 | |
| worktree 创建 | **herdr `worktree`** 或 git 脚本 | 每个任务一个，命名 `wb/<taskId>-<agent>` |
| 派活、等待、收结果 | **pi 扩展** | 分可见和 headless 两条路径 |
| 任务账本、证据、比较、review | **pi 扩展** | 借鉴 Charter 的语义和 pi-harness 的确定性调度 |
| pi 自身的权限与恢复 | **pi 扩展** | 借鉴 pi-harness 的 policy guard 和 checkpoint |
| 编排者模型 | **pi + pi-ai** | 推荐 ChatGPT 订阅（官方已允许第三方使用）或 API key。**不要用 pi-ai 的 Anthropic 订阅 OAuth**（有封号风险） |

## 4. 各扩展设计

### 4.1 `herdr-bridge`：可见 worker（对应 R1 和 R2 的可见路径）

用 herdr 的 CLI 实现（第一版直接 `execFile("herdr", [...])`，以后再换成 socket 加 `events.subscribe`）。

| 工具 | 实现 | 语义要点（借鉴 herdr 和 Charter） |
|---|---|---|
| `worker_spawn{agent, task?, worktree?}` | `herdr worktree` → `herdr agent start --kind <agent>` | 等 agent 就绪再返回；新 pane 自动获得 `HERDR_PANE_ID` |
| `worker_prompt{pane, text, wait?}` | `herdr agent prompt [--wait]` | 目标处于 blocked 时拒绝发送；超时不代表没送达 |
| `worker_wait{pane, until: done\|blocked}` | `herdr agent wait --until` | 事件驱动，不轮询；等待时锁定目标 pane |
| `worker_read{pane, source}` | `herdr pane read --source recent-unwrapped` | 截断后返回给模型，完整内容写入文件 |
| `worker_list` | `herdr` workspace/pane 列表 + 状态 | 供面板和模型使用 |

**防护规则（照搬 Charter）**

- 深度上限为 1。
- 每个编排者最多 5 个可见 worker。
- 每分钟最多发 30 条消息。
- 用户在 worker pane 里按任意键，就视为人工接管，编排者暂停向它发送。

### 4.2 `delegate`：headless worker（对应 R2，即 DSH 那种能力）

- 以 [03 笔记 §5](research_notes/03-agent控制协议与pi接入层.md) 的 ACP 示意代码为起点。
- 工具：`delegate` / `delegate_bg` / `delegate_send` / `delegate_cancel` / `delegate_list`。
- **比 DSH 多做三件事：**
  - 支持多轮追问：同一个 ACP session 可以多次 `session/prompt`。
  - 完整轨迹放进 `details`，由面板渲染。
  - 子 agent 的权限请求转交给 `ctx.ui.select`，由人决定。
- 后台模式：任务完成后用 `pi.sendMessage(..., {triggerTurn:true, deliverAs:"followUp"})` 唤醒编排者。这就是 Charter 那种"门铃"：结果在 turn 边界注入，不打断编排者当前的回合。
- 锁定适配器版本（claude-agent-acp、codex-acp、ACP SDK），并改为全局安装，避免每次 npx 冷启动。
- 只有确实需要 `turn/steer` 或 `review/start` 时，才额外实现 Codex app-server 原生后端。

### 4.3 `tasks`：任务账本与完成证据（对应 R3）

- 存储：每个仓库下的 `.workbench/tasks.jsonl`，只追加。初期不引入 SQLite。
- 任务记录：`{id, title, agent, mode: visible|headless, worktree, paneId|sessionId, status, evidence[]}`。
- `task_complete{id, evidence}`：**要标记完成，必须给出证据**（测试命令和输出、diff 摘要、PR 链接）。agent 变成 idle 或 done 不算完成，这条是 Charter 的核心原则。
- 任务之间可以声明依赖，调度照搬 pi-harness 的三种确定性策略：manual、sequential、dependency-based。不让 AI 去规划调度。
- 收尾时借鉴 pi-harness 的"声明抽取"：从 worker 输出里抽出它声称完成了什么，再和 git diff、测试结果对照，对不上就不允许 complete。
- `/tasks` 命令列出任务；任务带上 resume argv（`claude --resume <id>`、`pi --session <id>`），重启后可以接回。

### 4.4 `arena`：扇出比较与交叉 review（对应 R3）

- `/arena <task> --agents claude,codex`：同一个任务分别派给多个 agent，每个 agent 一个 worktree。全部完成后，由编排者汇总各自的 diff 和测试结果，给出对比和推荐，由**你**决定合并哪一份。参考 Orca 和 Parallel Code 的 Arena。
- `/review <taskId> --by codex`：一个 agent 写、另一个 agent 审，审阅意见回送给作者 agent 修改（参考 Codeg 和 Agent Orchestrator）。
- 有 headless 和可见两种跑法：小任务默认用 headless，大任务用可见 pane。

### 4.5 `panel`：worker 面板

- 用 `ctx.ui.custom({overlay:true})` 实现，表格列出全部 worker：agent、模式、状态、所属任务、最近一次工具调用。
- 可见 worker 可以一键跳转到对应的 herdr pane；headless worker 可以展开轨迹。这就是 DSH Trajectory 视图的精简版。

### 4.6 `handoff`：跨 agent 历史交接（借鉴 pi-harness 的 universal 模块）

- 只读扫描 `~/.claude/projects`、`~/.codex/sessions` 和 pi 自己的会话目录，按项目和时间列出历史会话。
- `/handoff <会话>`：
  1. 压缩该会话的上下文（设字符上限）；
  2. 读取原项目当前的 git 状态；
  3. 抽取"已声称完成"的事项；
  4. 生成一份交接提示，确认后在原项目目录里由 pi 接着做，或者派给另一个 agent。
- 用处：Claude Code 额度用完、某个会话卡住、想换个 agent 接手时，**不需要实时控制外部 agent 就能接力**。这是成本最低的一种跨 agent 协作，和 `delegate` 互补。

### 4.7 `guard`：policy 与 checkpoint（借鉴 pi-harness 的 harness/policy、harness/checkpoint）

- pi 自身没有权限系统。在 `tool_call` 钩子里实现：
  - 危险命令（`rm -rf`、`git reset --hard`、`git push --force` 等）需要确认；
  - 识别访问网络的命令；
  - 设置单任务的预算上限。
- checkpoint 只记录会话 leaf、git HEAD 和未提交改动状态，恢复交给 pi 的 fork 和 navigate，不承诺回滚整个工作区。外部 CLI 的改动由 worktree 兜底。
- 这一块开发量小、风险收益高，放在 P1 和 `herdr-bridge` 一起做。

## 5. 分阶段落地

| 阶段 | 目标 | 内容 | 验收标准 |
|---|---|---|---|
| **P0 验证**（1–2 天，不写代码） | 确认工作流是否成立 | 按手册部署 herdr（先升级到 v0.9.3+），安装 herdr 的 pi 集成；试装 `pi-harness-delegate` 或 `pi-acp-agents` 体验 headless 委派 | 能在 herdr 里同时跑 claude、codex、pi 三个 pane 并正确识别状态；pi 能成功委派一次 codex |
| **P1 可见编排** | R1 + R2 可见路径 | `herdr-bridge`、`guard`，加最简单的 `worker_list` 面板 | pi 能开出一个 Claude worktree pane、发任务、等到 done、读回结果；中途可以人工接管 |
| **P2 headless 委派与交接** | R2 | `delegate`（含后台模式和权限转交）、`handoff` | 能并行委派 3 个 headless 子任务，结果自动唤醒编排者；轨迹可以展开查看；能把一个 Claude Code 历史会话交接给 pi 继续 |
| **P3 编排语义** | R3 | `tasks`（含依赖调度、声明对照）加 `arena` | 跑通一次 claude 对 codex 的同题比较，以及一次交叉 review，完成证据记入账本 |
| **P4 按需再做** | 长时运行和多端 | pi-durable 后台任务；CI 失败回送；手机端 | 根据实际使用情况决定 |

**提前想好退路：** 如果做到 P1 发现自己大部分时间其实在 GUI 里看 diff、做 review，就不要自己做 GUI，直接换用现成的 C 型结构化客户端：Paseo、Codeg（全 ACP，支持飞书和微信）或 Orca。pi 扩展部分仍然可以通过 `pi-acp` 作为其中一个 agent 接入。如果只需要 pi 原生的多 agent 编排 GUI，而不需要驱动外部 agent，直接用 pi-harness 也可以。

## 6. 明确不做

- 不自己实现 PTY daemon、终端复用器或 Electron IDE。这是 Charter 停滞的原因。
- 不 fork 或修改 pi 核心，全部用扩展和 pi package 实现。
- 不用 pi-ai 的 Anthropic 订阅 OAuth 直接调 API（违反服务条款）。Claude 只通过官方 Claude Code 二进制使用。
- 不引入 Gas Town 或 Multica 那种重流程、团队化的编排概念。
- 不复制 pi-harness 的代码（AGPL），只借鉴设计。
- 不把 DSH 当作运行时依赖，它还是 alpha，并且会话日志默认上传。

## 7. 待验证

- herdr 的 Claude 状态完全依赖读屏规则。Claude Code 升级后，需要验证 `agent wait --until done` 的可靠性。
- 用 `herdr agent prompt` 往 Claude/Codex 发多行任务时，bracketed paste 的表现。
- claude-agent-acp 和 codex-acp 在订阅登录下的认证流程，以及用量计入哪种额度。
- pi 1.x 扩展 API 的稳定性。ACP 示意代码还没有编译过。
