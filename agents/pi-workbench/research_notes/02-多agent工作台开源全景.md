# 多 agent 工作台开源全景（不含 herdr 和 Charter）

> 调研日期：2026-10-12。star 数、许可证、最后推送时间和语言来自当天的 `gh api` 快照。每个项目控制 agent 的方式，是通过 package.json 依赖和代码搜索核实的。

## 一、近期状态变化

- **Vibe Kanban**：所属公司 bloop 于 2026-04 宣布关闭，项目转为社区维护、纯本地运行。README 标注为 sunsetting。
- **Crystal**：已停更，继任项目是 Nimbalyst。
- **opcode/claudia**：基本停更，并且只支持 Claude。
- **coder/mux**：因商标问题更名为 **Xum**。它自己实现了 agent loop，不托管外部 CLI。
- **coder/agentapi**：已归档并弃用。它是"抓终端屏幕、包装成 HTTP API"这一做法的开创者。
- **Omnara**：已转型为可自托管的 agent 平台。
- **uzi**：已无人维护。**Toad**：最后提交在 2026-05，节奏放缓。

## 二、对比表

原型（A–E）的含义见第三节。

| 项目 | ★ / 许可 / 最后推送 / 语言 | 形态 | 控制 agent 的方式 | 隔离 | 状态检测 | 特色 | 原型 |
|---|---|---|---|---|---|---|---|
| claude-squad | 8.6k / AGPL / 08-20 / Go | TUI | tmux 读屏（capture-pane） | worktree | 屏幕变化 | 预览/diff 标签、自动确认 | A |
| agent-deck | 1.1k / MIT / 10-11 / Go | TUI+Web | tmux 读屏 + hooks | worktree | 轮询 + hooks | 原生 fork 会话；Conductor（监督 agent 可升级到手机/Telegram） | A+E |
| agent-of-empires | 3.3k / MIT / 10-12 / Rust | TUI+Web/PWA+HTTP API | tmux，另有 ACP 结构化视图 | worktree + Docker/Podman/Apple 容器 | 识别 agent 类型 | 断开 SSH 后会话照跑；手机可用 | A（部分 C） |
| ccmanager | 1.3k / MIT / 10-12 / TS | TUI | 自管 PTY，不依赖 tmux | worktree + devcontainer | 每种 agent 单独策略 | 跨 worktree 复制 Claude 会话；`.worktreeinclude` | A |
| workmux | 2.8k / MIT / 10-11 / Rust | tmux/zellij/kitty | 复用器窗口 | worktree | hooks 写入窗口名 | `/worktree` skill 让 agent 自己派活 | A |
| dmux | 1.8k / MIT / 08-16 | TUI | tmux pane | worktree | — | 一个 prompt 可选多个 agent | A |
| AWS CLI Agent Orchestrator | 1.4k / Apache / 10-12 / Py | CLI+Web | 每个 agent 一个 tmux 会话 | worktree | — | supervisor 与 worker 之间走 MCP | A+E |
| Gas Town | 18.3k / MIT / 10-08 / Go | CLI/TUI/Web | tmux | worktree | 看门狗层级 | 用 beads（git 承载的任务单）保存持久状态，逐级升级 | E |
| mngr（Imbue） | 421 / 10-12 / Py | CLI | SSH + tmux + git | 本地/Docker/Modal | 空闲自动关停 | "agent 版 git" | A |
| cmux | 28k / GPL / 10-12 / Swift | macOS 原生终端（Ghostty） | 真实 TTY + socket | 无 | hooks 通知环 | 内置可脚本化浏览器 | B |
| Emdash | 6.0k / Apache / 10-10 / TS | Electron | node-pty 跑原生 CLI | worktree + SSH | 注入生命周期 hooks | 10 种 issue 源；PR/CI/合并 | B |
| Superset | 15k / Elastic-2.0 / 10-12 / TS | Electron + iOS | node-pty | worktree | hooks | 同题两解择优；CLI/SDK/MCP；定时任务 | B |
| Orca | 90k / MIT / 10-12 / TS | Electron + 移动端 | node-pty + Claude Agent SDK + Codex app-server | worktree + SSH | 通知/未读 | **一个 prompt 扇出给 5 个 agent，比较后合并最佳结果**；diff 批注回传；agent 可反向调用 `orca` CLI | B+C |
| Parallel Code | 1.0k / MIT / 10-12 / TS | 桌面 | node-pty + Agent SDK | worktree | — | **AI Arena：多个 agent 同题比拼** | B+C |
| Conductor（闭源） | — | macOS | Claude Code / Codex | worktree | — | 业界参照，已推出 Cloud 版 | B |
| Vibe Kanban | 28k / Apache / 09-19 / Rust | Web | headless：Claude stream-json、Codex app-server、其余走 ACP | worktree | 结构化 | 看板；diff 评论发回 agent | C |
| T3 Code | 27k / MIT / 10-12 / TS | Web/桌面/手机 | Codex app-server + Agent SDK + OpenCode SDK | worktree | 结构化 | 常驻控制面服务 | C |
| Paseo | 21k / 源码可见 / 10-12 / TS | daemon + 多端 | app-server + Agent SDK + OpenCode SDK + ACP + PTY | worktree | 结构化 | **agent 之间可跨厂商互相拉起和对话**；端到端加密中继 | C |
| Nimbalyst | 1.9k / MIT / 10-11 / TS | Electron + iOS | Agent SDK + Codex SDK | worktree | 结构化 | 在渲染后的文档里逐处接受或拒绝改动 | C |
| Codeg | 3.9k / Apache / 10-12 / Rust | 桌面/自托管/手机 | **15 种 agent 全部走 ACP** | worktree | 结构化 | `@` 多个 agent 并行（如 Claude 写、Codex 审）；fork 与 steer；可接飞书/微信 | C |
| Toad | 3.5k / AGPL / 05-26 / Py | TUI | 纯 ACP 客户端 | 无 | 结构化 | agent 应用商店 | C |
| Happy | 24k / MIT / 10-11 / TS | 手机/Web/桌面 | CLI 包装（SDK + ACP）+ 端到端加密中继 | 无 | 结构化 | 多人同控一个会话；语音 | C→D |
| Sculptor | 238 / MIT / 10-12 / Py | 桌面 | Claude stream-json | worktree | 结构化 | 研究预览阶段 | C |
| Agor | 1.4k / BSL / 10-12 / TS | Web 多人画布 | 各家 CLI 与 SDK | 分支 + 沙箱 | 结构化 | 自身作为 MCP 暴露，agent 可派生会话 | C+E |
| pi-harness | 202 / AGPL / 10-11 / TS | Electron | 进程内 pi SDK，只编排多个 pi 会话 | worktree | 结构化 | 任务 DAG + 确定性调度、policy、checkpoint、run 对比；跨 7 家 agent 历史交接（详见 [04](04-pi-harness拆解.md)） | C+E |
| Xum（原 Coder Mux） | 2.1k / AGPL / 10-12 / TS | 桌面 + 移动 Web | **自研 agent loop** | 本地/worktree/SSH | 原生 | Plan/Exec 两种模式 | D |
| Agent Orchestrator | 13k / Apache / 10-12 / Go | 桌面 + daemon | 多种 harness | worktree | daemon 监视 agent、PR、CI | **CI 失败、review 意见、冲突自动回送给负责的 agent** | E |
| Multica | 53k / 改版 Apache / 10-12 / Go | Web + 本地 daemon | 调用本机 agent CLI | 本机 | issue 状态 | agent 作为队友领取 issue，完成后进入 review 关卡 | E |
| CCPM | 8.4k / MIT / 03-18 / Shell | Claude Code skill | Claude 子 agent | worktree | GitHub Issues | 从 PRD 到 Epic 再到 Issue 并行推进 | E |

## 三、五种架构原型

**A. 终端复用器 + 读屏**（tmux/PTY，用 capture-pane 或正则判断状态）
- 优点：
  - 任何 CLI 都能零适配接入，保留原生 TUI 的全部能力。
  - 会话能脱离 UI 常驻。
  - 实现轻量。
- 缺点：
  - 状态检测脆弱。
  - 拿不到结构化的消息、工具调用和 token 数据。
  - "多个 agent 同题比较"最多只能比到 diff 这一层。

**B. GUI 内嵌真实终端 + 用 hooks 补状态**
- 优点：
  - 保留 CLI 的原生体验和订阅计费。
  - 能围绕 diff、PR、CI、预览做完整的工作流。
  - 用 hooks 判断状态比读屏可靠。
- 缺点：
  - hooks 会写进用户的全局配置，有侵入性。
  - 结构化程度仍然有限。
  - Electron 应用偏重。

**C. 结构化协议客户端**（stream-json / Codex app-server / Agent SDK / ACP，自己渲染界面）
- 优点：
  - 状态、权限请求、工具调用、成本全部以事件形式提供。
  - 天然适合调度、比较、交接、交叉 review，以及移动端访问。
- 缺点：
  - 每家协议都要单独维护，版本变化快。
  - 会丢掉一部分原生 TUI 能力。
  - Claude 和 Codex 的 ACP 支持依赖第三方适配器。

**D. 自研 harness**（自己写 agent loop，直接调模型 API）
- 优点：完全可控。
- 缺点：用不上官方 harness 的能力和订阅额度。

**E. 编排和控制面层**（任务板、监督 agent、PR/CI 闭环），叠在 A–C 之上
- 优点：
  - 直接回答"活派给谁、谁来 review、失败回给谁"。
  - 用持久化任务跨越单个会话的寿命。
- 缺点：
  - 流程重，概念多。
  - 对个人使用来说容易过度设计。

## 四、值得借鉴的做法

1. **扇出比较**：Orca（同一 prompt 发给 5 个 agent，比较后合并）、Parallel Code（AI Arena）、Superset（同题两解择优）。
2. **交叉 review**：Codeg（Claude 写、Codex 审）、Agent Orchestrator（把 CI 和 review 结果回送给负责的 agent）。
3. **会话 fork**：agent-deck、ccmanager、Codeg。
4. **监督 agent**：agent-deck 的 Conductor、Gas Town 的 Mayor。
5. **反向可编程**（agent 能反过来操作工作台）：Orca 的 CLI、cmux 的 socket、Agor 和 Superset 的 MCP。
6. **常驻 daemon + 多端客户端 + 端到端加密中继**：Paseo、T3 Code、Happy。
7. **worktree 配套**：`.worktreeinclude`、用 symlink 共享 node_modules、post-create hooks。

## 来源

- GitHub（`https://github.com/<名称>`）：
  - 终端/TUI 类：smtg-ai/claude-squad、asheshgoplani/agent-deck、agent-of-empires/agent-of-empires、kbwo/ccmanager、raine/workmux、standardagents/dmux、awslabs/cli-agent-orchestrator、gastownhall/gastown、imbue-ai/mngr、manaflow-ai/cmux
  - 桌面/GUI 类：generalaction/emdash、superset-sh/superset、stablyai/orca、johannesjo/parallel-code、nimbalyst/nimbalyst、imbue-ai/sculptor
  - 结构化客户端类：BloopAI/vibe-kanban、pingdotgg/t3code、getpaseo/paseo、spacering-net/codeg、batrachianai/toad、slopus/happy、preset-io/agor
  - 编排类：OrchestratorInc/agent-orchestrator、multica-ai/multica、automazeio/ccpm
  - 其他：coder/mux、coder/agentapi
- https://github.com/1shiharat/awesome-agent-clients
- https://vibekanban.com/blog/shutdown
- https://www.conductor.build
- https://github.com/agentclientprotocol/agent-client-protocol

> 注：部分 star 数（Orca 90k、Multica 53k）明显偏高，但都是 API 原样返回的数字。Superset、Paseo、Multica 在 API 中的许可证为 NOASSERTION，表中写的许可证来自 awesome-agent-clients 清单。
