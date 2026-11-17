# pi-harness 源码级拆解（Source Review）

> 调研日期：2026-10-13。方法：浅克隆 `wangmiaozero/pi-harness` 并阅读源码，未运行。

## 一、基本事实（Facts）

| 项 | 值 |
|---|---|
| 仓库 | https://github.com/wangmiaozero/pi-harness |
| 定位 | 自称 "Pi Coding Agent 的 Operational Superset"：保留原生 pi 运行时，在其上增加观测、控制、恢复、评估和多 agent 编排 |
| 形态 | Electron + Vue 3 桌面应用，支持 macOS、Windows、Linux |
| 许可 | **AGPL-3.0-only**（见第五节） |
| 热度 | 202 star、12 fork |
| 活跃度 | 2026-08-20 创建，最新版 v1.9.0（2026-10-11），近期几乎每天都有提交 |
| 维护者 | 一人（wangmiao） |
| pi 依赖 | `@earendil-works/pi-coding-agent` 等包，锁定在 0.87.1 |
| 规模 | 约 845 个文件，其中 `src/main/harness` 约 1.46 万行，配有 e2e 和单元测试 |

## 二、架构（Architecture）

**pi 的接入方式**

- 在进程内直接使用 pi SDK：先 `createAgentSessionServices()`，再 `createAgentSessionFromServices()`。
- agent 循环、会话、上下文压缩都用 pi 原生实现，pi-harness 没有自己重写这些部分。

**`src/main` 各模块（按代码量）**

| 模块 | 约行数 | 职责 |
|---|---|---|
| `harness/` | 14.6k | runs、trace、replay、compare、diagnostics、evaluation、regression、artifacts、policy、checkpoint、orchestrator |
| `services/` | 5.6k | 通用服务 |
| `universal/` | 4.4k | 读取多家 agent 的本地历史，并支持交接 |
| `packages/`、`pi/`、`ipc/`、`agent/` | 各 2–3k | pi 包管理、pi 环境检测与安装、IPC、运行时服务 |
| `git/`、`environment/`、`capabilities/`、`skills/` | 各约 2k | 工程工作区功能 |

**编排（`harness/orchestrator`）**

- 一个 agent 就是一个 **pi 会话**，各自配置不同的模型、system prompt、工具集、skills 和预算。
- 用 `isReviewer` 标出评审角色。
- 每个 agent 可以选择共享工作区，或使用独立的 git worktree。
- 任务之间是依赖图（DAG）。调度器是**确定性的**，不靠 AI 规划，有三种策略：manual、sequential、dependency-based。并发受 `maxConcurrentAgents` 和 `maxConcurrentRuns` 限制。
- 另有 handoff-service、conflict-service（处理冲突）、team-service。
- **不支持驱动外部 agent**：代码里找不到任何调用 Claude Code、Codex、ACP 或 PTY 的地方，所谓多 agent 只是多个 pi 会话。

**控制（`harness/policy`、`harness/checkpoint`）**

- Policy 是声明式配置，在 harness 的工具边界上拦截工具调用（`policy-tool-guard.ts`），不改 pi 自身的配置。
  - 危险命令（`rm -rf`、`sudo`、`git reset --hard`、`git push --force` 等）会弹出交互确认。
  - 访问网络的命令会被识别出来（curl、ssh、rsync 等）。
  - 可以设置预算上限（费用和 token）。
- Checkpoint 只记录真正可恢复的锚点：会话树的 leaf、git 的 HEAD 和未提交改动状态、上下文快照。恢复靠 pi 原生的 navigate 和 fork，代码注释写明"不假装做了完整的工作区回滚"。
- 上限：总共 500 个 checkpoint，每个会话 100 个。

**跨 agent 历史与交接（`universal/`，v1.9.0 新增）**

- 只读扫描以下 7 家 agent 的本地会话：
  - Claude Code：`~/.claude/projects`
  - Codex：`~/.codex/sessions`、`archived_sessions`
  - Cursor、Cursor Agent
  - Gemini CLI
  - OpenCode
  - pi
- 扫描结果统一规范化，存进 SQLite，可按来源、项目、日期搜索和筛选。
- **交接（handoff）流程**：
  1. 压缩历史上下文，默认上限 48k 字符。
  2. 检查原项目的 git 状态。
  3. 抽取"声明"（claims），即历史里 agent 声称完成了什么。
  4. 生成一份交接提示，用户确认后，在**原项目目录**里用当前模型新开一个 pi 对话继续。
- 浏览历史和预览交接不消耗 token；只有点"AI 整理"才会调用模型。原始会话始终只读。

**其他**

- 能力中心可安装 pi 包和 skills（可选接入 Superpowers、Odai 两套方法论）。
- 内置轻量文本编辑器（明确说明不是 IDE）、Git 面板、Provider 与模型管理。
- 大量投入在外观上：主题、皮肤、吉祥物、启动动画。

## 三、与诉求对照（Fit）

| 诉求 | pi-harness 的覆盖情况 |
|---|---|
| R1 多个 Claude Code、Codex 窗口并行，状态统一 | ❌ 不支持。不托管外部 CLI |
| R2 主 agent 委派给 Claude Code、Codex | ❌ 不支持。只能委派给其他 pi 会话 |
| R3 任务编排、worktree、review、证据 | ✅ 覆盖最完整：DAG、确定性调度、评审角色、worktree、冲突处理、run 对比、评估与回归 |
| R4 以 pi 为核心、轻量、可改 | ⚠️ 以 pi 为核心，但 Electron 应用体量大，且使用 AGPL 许可 |
| 额外：跨 agent 历史交接 | ✅ 独有能力：把 Claude Code、Codex 的历史会话交接给 pi 继续 |

**结论**：在"pi 原生多 agent 编排 + 治理"这一块，pi-harness 是目前做得最完整的开源实现，是 R3 的首要参考对象。它和 Charter 刚好互补：Charter 擅长驱动外部 CLI，pi-harness 擅长 pi 内部的编排与治理。但它和 Charter 一样是单人维护的 Electron 应用，并且完全不涉及外部 agent，**不适合当作基座**。

## 四、值得借鉴的设计（Takeaways）

1. **跨 agent 历史交接**：扫描 `~/.claude/projects`、`~/.codex/sessions` 等目录，压缩上下文、检查 git 状态、抽取声明后生成交接提示。这条路成本最低：不需要运行任何外部 agent，就能让 pi 接手 Claude Code 或 Codex 没做完的活，可以和 `delegate` 互补。
2. **工具边界上的 policy guard**：pi 本身没有权限系统，最便宜的补法是在扩展里用 `tool_call` 钩子做危险命令确认、网络命令识别和预算上限。
3. **诚实的 checkpoint**：只记录会话 leaf、git 状态和上下文，恢复交给 pi 的 fork 和 navigate，不承诺回滚整个工作区。
4. **确定性调度**：任务 DAG 加 manual / sequential / dependency 三种策略，不依赖 AI 规划，行为可预期、可测试。
5. **Run 对比与回归**：同一任务的多次运行可以并排比较，可直接作为 `arena` 结果对比的参考。
6. **声明抽取**：从 agent 的输出里抽出"声称完成了什么"，再拿去和 git 状态、测试结果对照，和 Charter 的"完成要有证据"是同一个思路。

## 五、许可提醒（License）

- pi-harness 使用 **AGPL-3.0-only**。
- 借鉴思路、自己从头实现，不受影响。
- 如果把它的代码直接复制进自己的扩展，整个扩展就必须也采用 AGPL。个人本机使用影响不大；一旦对外分发或以网络服务形式提供，就要遵守 AGPL。
- 建议：**只参考设计，不复制代码**。
