# 以 pi 为编排器、Claude Code 和 Codex 为 worker：接入层调研

> 调研日期：2026-10-12。依据：DSH 和 pi 的仓库源码（pi 取 2026-10-12 HEAD，v1.1.0），ACP TS SDK 1.8 的类型声明，以及各家官方文档。

## 0. 结论

- **主干用 ACP**：给 pi 写一个 `delegate` 扩展，让 pi 作为 ACP client，接入以下几种 worker：
  - `claude-agent-acp`（0.89.x）
  - `codex-acp`（2.2.x）
  - `gemini --acp`
  - `pi-acp`（pi 自己也可以当 worker）
- **按需加原生后端**：
  - 需要 `turn/steer` 或 `review/start` 时，接 Codex app-server。
  - 需要 hooks、结构化输出或进程内 MCP 时，接 Claude Agent SDK。
- **DSH 的做法**：Claude Code 和 Codex 各走官方原生接口，ACP 用作通用后端。三种都是一次性执行，只回传最终文本，不经人工审批。个人工作台可以做得更多：支持多轮对话、轨迹可视化和人工审批。
- **pi 核心不含任何 ACP 代码**，ACP 支持全部来自社区包（见 §3）。

## 1. DSH 子代理是怎么实现的

**共同骨架**

- `dsh-subagent` 提供 `ctx.subagents` provider 注册表；`dsh-tool-subagent` 把每个 provider 绑定成一个工具，如 `subagent_claude_code`、`subagent_codex`。
- 工具调用后立即返回 `started subagent <childId>`。子代理跑完后，结果以 user-role 通知的形式注入父会话，并可以唤醒一个新 turn。
- 限额：
  - `maxActiveSubagents` 默认 8，由整棵子代理树共享。
  - `maxDepth` 默认 1。外部 provider 的深度由 provider 自己管理。
  - 外部 provider 不支持 outputSchema、persona、toolFilter 等选项，请求这些选项会被直接拒绝。
- 双向消息（`send_message`、`interrupt_agent`、`list_agents`）只对进程内子代理有效。外部后端都是一次性执行，**不能追问**。
- 只回传最终文本，外加不超过 4096 字节的安全诊断信息。reasoning、工具流、diff 一律不进父会话，外部子代理也没有轨迹视图。
- 子进程启动前，会先清掉形似凭据的环境变量。

**三个外部 provider**

| 插件 | 接入方式 | 权限处理 | 要点 |
|---|---|---|---|
| subagent-claude-code | `@anthropic-ai/claude-agent-sdk@0.3.263`，每次调用都是全新的 `query()` | `permissionMode` 默认 dontAsk；`canUseTool` 一律拒绝；禁用 `AskUserQuestion` | `persistSession:false`；用 SDK 自带的 CLI，不退回宿主机的 `claude`；不指定 `settingSources`，因此宿主的 settings 和登录态会生效 |
| subagent-codex | `codex app-server --stdio`（JSON-RPC） | 默认 `never`；可选 `approve-for-me` 或 `dangerously-bypass` | 调用顺序：`initialize` → `thread/start{ephemeral:true}` → `turn/start` → 等待 `turn/completed`；结果取最后一条 `phase:"final_answer"`；审批请求自动应答，遇到未知请求直接失败；取消用 `turn/interrupt` |
| subagent-acp | `@agentclientprotocol/sdk@1.4.0`，每次运行新起一个进程 | 默认拒绝 | `clientCapabilities:{}`（不提供 fs 和 terminal）；只收集 `agent_message_chunk`；取消顺序：cancel → 关 stdin → SIGTERM → SIGKILL |

DSH 的设计记录明确否决了 `codex exec` 和自己手写 Claude CLI 协议这两条路，理由是它们绕开了官方集成接口，配置、审批和进程回收都没法验证。

## 2. 各家 agent 的程序化驱动方式

| Agent | 路径 | 优点 | 缺点 |
|---|---|---|---|
| Claude Code | Agent SDK（TS/Py） | 能力最全：streaming input 支持多轮、`interrupt()`、`canUseTool`、hooks、resume/fork、进程内 MCP | 平台二进制约 300MB；只能用 Claude |
| | `claude -p --output-format stream-json` | 零依赖；支持 `--resume`；可用 `--permission-prompt-tool` 处理审批 | 审批流程绕 |
| | claude-agent-acp | 标准 ACP，权限、终端、session load 都有 | 多一层转换 |
| Codex | `codex exec --json` | 一次性任务最简单 | 没有审批回调 |
| | Codex SDK | `startThread`、`runStreamed`、`resumeThread` | 底层就是封装 exec |
| | app-server（stdio JSONL / WS） | 能力最全：`turn/steer`、`turn/interrupt`、审批请求、`thread/fork`、`review/start`；官方推荐给富客户端 | 协议面大、变化快，需要锁定版本 |
| | codex-acp | 底层启动 app-server；把 ChatGPT 登录和 API key 暴露为 ACP 认证方式 | 多一层转换 |
| Gemini CLI | `gemini --acp` | 原生支持，不需要适配器 | 旧参数 `--experimental-acp` 已弃用 |
| 任意 TUI | coder/agentapi | — | **已于 2026-09 归档**，不推荐 |

**ACP 要点（TS SDK 1.8）**

- 生命周期：`initialize` → `authenticate` → `session/new|load|resume|list|fork|close` → `session/prompt` → `session/cancel`。
- `session/update` 的事件类型：agent_message_chunk、agent_thought_chunk、tool_call、tool_call_update、plan、available_commands_update、current_mode_update。
- 需要由 client 实现的请求：`session/request_permission`、`fs/read_text_file`、`fs/write_text_file`、`terminal/*`、elicitation。
- 新版写法：`acp.client({name}).onRequest/onNotification(...).connect(stream)`。

**认证与服务条款（重要）**

- **Anthropic**
  - 订阅账号的 OAuth 只能用于 Claude Code 和 Anthropic 官方应用。开发者做的产品必须用 API key，也不能代理或存储订阅凭据。
  - 但官方不禁止终端用户用自己的订阅登录**未修改的** Claude Code 二进制。
  - 因此，在本机 spawn 官方 Claude Code（无论通过 SDK、`-p` 还是 claude-agent-acp），并使用自己 `claude login` 的账号，属于相对安全的灰区。如果要分发给别人用，就必须改用 API key。
  - 自 10/7 起，Max/Team 套餐附带每月 API credits，但只有使用已关联 Console 组织的 API key 时才能抵扣。
  - ⚠️ pi-ai 仍然内置 Anthropic Pro/Max OAuth。让 pi 直接拿订阅 token 调 API，正是 2026-02 被明确禁止的用法，**有封号风险**。
- **OpenAI**
  - Codex 的各条路径共用 `~/.codex/auth.json`，都支持 ChatGPT 登录。
  - 2026-09-29 推出 "Sign in with ChatGPT"，正式允许第三方使用订阅额度，pi 是首批合作方。因此 pi 编排器本身可以直接走 ChatGPT 订阅。
- **其他**：DSH 的 e2e 测试把 `ANTHROPIC_BASE_URL` 指向 DeepSeek。这说明 Claude Code 这类 worker 的底层模型可以替换。

## 3. pi 侧现状

**ACP 相关**

- pi 作为 ACP server：
  - `pi-acp`（svkozak，已进入 Zed 注册表，内部桥接 `pi --mode rpc`）
  - Rust 实现 `pi-acpinator`
- pi 作为 ACP client：
  - `pi-acp-agents`（提供 10 个 acp_* 工具）
  - `@junghanacs/pi-shell-acp`（继任项目叫 entwurf），把 claude-agent-acp 和 codex-acp 当作 pi 的 provider，不做 OAuth 代理，也不读屏

**非 ACP 的委派包**

- `pi-harness-delegate`：支持 claude、codex（exec --json）、opencode、amp、devin；权限统一为 readonly / edit / danger 三档；支持 fan-out 和 resume。
- `@danchamorro/pi-subagents`：在后台运行 pi、Claude Code、Codex，用 Agent SDK 实现，并发上限默认 5。
- 另有 `pi-codex`、`pi-flow-external`、`claude-agent-sdk-pi`。

**官方示例 `examples/extensions/subagent`**

- 每个任务都 spawn 一个 `pi --mode json -p --no-session`。
- 支持三种模式：single、parallel（最多 8 个任务，并发 4）、chain。
- 通过 `onUpdate` 流式更新；用 `renderCall`/`renderResult` 渲染，Ctrl+O 展开。

**与 delegate 相关的扩展 API**

- `pi.registerTool`：TypeBox 定义参数，`execute(toolCallId, params, signal, onUpdate, ctx)`，可配合 `renderCall`/`renderResult` 使用。
- `pi.sendMessage(msg, {triggerTurn, deliverAs:"steer"|"followUp"|"nextTurn"})`：用于后台任务完成后的通知。
- `pi.appendEntry`：持久化不进入上下文的数据，比如 worker 的 sessionId 映射。
- UI：`registerMessageRenderer`、`ctx.ui.select/confirm/custom({overlay:true})`、`setWidget`、`setStatus`。
- 用 `session_shutdown` 事件做清理；不要在扩展的 factory 函数里启动进程。
- RPC 模式（`pi --mode rpc`）可以转发对话框，但不能转发自定义 TUI 组件。

**pi-durable（实验性）**

- 核心概念：Harness、Conversation、Entry、原子 Commit、Task。Task 是一个每步都存 checkpoint 的状态机，崩溃后可以续跑。
- `{background:true}` 的 task 构成一个边界：父任务 abort 时，它拥有的工作不受影响。
- 示例 23 演示了 spawn、message、wait、stop、list，并用 requestId 去重，几乎就是现成的后台子代理写法。
- `taskGraph`/`watchTaskGraph` 可用来做任务面板。
- chord 提供跨进程对象、handle、状态复制和热重载；pi-env 提供基于 SSH 的远程执行环境。

## 4. 推荐架构

1. **编排层**：pi coding-agent 加一个 `delegate` 扩展，提供以下工具：
   - `delegate`：前台执行，流式输出
   - `delegate_bg`：立即返回 id，完成后通过 `sendMessage({triggerTurn, deliverAs:"followUp"})` 通知
   - `delegate_send`：在同一个 ACP session 上再发一轮 prompt
   - `delegate_cancel`、`delegate_list`
2. **Worker 抽象**：接口为 `{start, prompt, cancel, load?, steer?}`。默认实现走 ACP，另有可选的原生实现（Codex app-server、Claude Agent SDK）。
3. **Supervisor**：
   - 同时活跃的 worker 控制在 4–8 个。
   - 用 `PI_DELEGATE_DEPTH` 限制委派深度不超过 1。
   - 多个 worker 并行写代码时，每个 worker 分配一个独立的 worktree。
   - 权限转交：有 UI 时，把 ACP 的 `request_permission` 转给 `ctx.ui.select` 让人确认；无 UI 时按策略自动应答。
4. **结果处理**：给模型的只有截断后的最终文本，附上完整输出的路径；完整轨迹放进 `details`，供界面渲染。
5. **持久化**：用 `appendEntry` 或 pi-durable task 记录 `{agent, sessionId, cwd}`，重启后通过 `session/load` 接回。

**三种接入方式的取舍**

| 方式 | 优点 | 缺点 |
|---|---|---|
| ACP 优先 | 一个协议覆盖各家；自带权限和工具流事件；适配器维护活跃 | 多一层转换，可能有版本漂移；没有回合中途 steer；npx 冷启动慢，应改为全局安装 |
| 各家原生接口（DSH 的路线） | 保真度最高 | 每接一家就要写一份适配 |
| CLI JSONL（`-p` / `exec --json`） | 做 MVP 最快 | 没有审批回调 |

## 5. 示意代码：基于 ACP 的 `delegate` 扩展

> API 名称已对照 ACP SDK 1.8 的 `.d.ts` 和 pi 的 `types.ts` 核对过，但**代码没有编译或运行过**。

```ts
// ~/.pi/agent/extensions/delegate/index.ts   (npm i @agentclientprotocol/sdk)
import { spawn, type ChildProcess } from "node:child_process";
import { Readable, Writable } from "node:stream";
import * as acp from "@agentclientprotocol/sdk";
import { StringEnum } from "@earendil-works/pi-ai";
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

const AGENTS = {
  claude: ["claude-agent-acp"],   // npm i -g @agentclientprotocol/claude-agent-acp（复用本机 claude 登录）
  codex: ["codex-acp"],           // npm i -g @agentclientprotocol/codex-acp（ChatGPT 或 API key）
  gemini: ["gemini", "--acp"],
  pi: ["pi-acp"],                 // 递归：pi 自身作为 worker
} as const;
type AgentName = keyof typeof AGENTS;
const MAX_LIVE = 4;
const DEPTH = Number(process.env.PI_DELEGATE_DEPTH ?? 0);

type Ask = (p: acp.RequestPermissionRequest) => Promise<acp.RequestPermissionResponse>;
interface Worker {
  sessionId: string; agent: AgentName; proc: ChildProcess; conn: acp.ClientConnection;
  busy: boolean; onUpdate?: (u: acp.SessionUpdate) => void; ask?: Ask;
}
const rejectAll: Ask = async (p) => {
  const r = p.options.find((o) => o.kind === "reject_once");
  return { outcome: r ? { outcome: "selected", optionId: r.optionId } : { outcome: "cancelled" } };
};

export default function (pi: ExtensionAPI) {
  if (DEPTH >= 1) return;                       // 深度上限 1：子 pi 不再暴露 delegate
  const workers = new Map<string, Worker>();    // key = ACP sessionId

  async function spawnWorker(agent: AgentName, cwd: string): Promise<Worker> {
    if (workers.size >= MAX_LIVE) throw new Error(`已达 ${MAX_LIVE} 个活跃 worker，请先 delegate_close`);
    const [cmd, ...args] = AGENTS[agent];
    const proc = spawn(cmd, args, { cwd, stdio: ["pipe", "pipe", "ignore"],
      env: { ...process.env, PI_DELEGATE_DEPTH: String(DEPTH + 1) } });
    const conn = acp.client({ name: "pi-delegate" })
      .onNotification(acp.methods.client.session.update, async ({ params }) => {
        workers.get(params.sessionId)?.onUpdate?.(params.update);
      })
      .onRequest(acp.methods.client.session.requestPermission, ({ params }) =>
        (workers.get(params.sessionId)?.ask ?? rejectAll)(params))
      .connect(acp.ndJsonStream(
        Writable.toWeb(proc.stdin!) as WritableStream<Uint8Array>,
        Readable.toWeb(proc.stdout!) as ReadableStream<Uint8Array>));
    try {
      // 不声明 fs/terminal：worker 在自己进程里读写、跑命令。
      // 若 initialize 返回 authMethods 且未登录，需先 authenticate。
      await conn.agent.request(acp.methods.agent.initialize,
        { protocolVersion: acp.PROTOCOL_VERSION, clientCapabilities: {} });
      const { sessionId } = await conn.agent.request(acp.methods.agent.session.new, { cwd, mcpServers: [] });
      const w: Worker = { sessionId, agent, proc, conn, busy: false };
      workers.set(sessionId, w);
      proc.once("exit", () => workers.delete(sessionId));
      return w;
    } catch (e) { conn.close(); proc.kill("SIGTERM"); throw e; }
  }

  pi.registerTool({
    name: "delegate",
    label: "Delegate",
    description: "把自包含任务交给外部编码 agent（claude/codex/gemini/pi），返回最终回答和 sessionId；传 sessionId 可在同一 worker 会话中追问。",
    parameters: Type.Object({
      agent: StringEnum(["claude", "codex", "gemini", "pi"] as const),
      task: Type.String({ description: "完整、自包含的任务说明" }),
      sessionId: Type.Optional(Type.String()),
      cwd: Type.Optional(Type.String({ description: "并行写入时建议指向独立 git worktree" })),
    }),
    async execute(_id, p, signal, onUpdate, ctx) {
      const w = p.sessionId ? workers.get(p.sessionId) : await spawnWorker(p.agent, p.cwd ?? ctx.cwd);
      if (!w) throw new Error(`worker ${p.sessionId} 不存在或已退出`);
      if (w.busy) throw new Error("该 worker 正在执行上一轮");
      w.busy = true;
      let text = "";
      const tools = new Map<string, string>();
      const details = () => ({ agent: w.agent, sessionId: w.sessionId, tools: [...tools.values()] });
      w.onUpdate = (u) => {
        if (u.sessionUpdate === "agent_message_chunk" && u.content.type === "text") text += u.content.text;
        else if (u.sessionUpdate === "tool_call") tools.set(u.toolCallId, `${u.title} [${u.status ?? "pending"}]`);
        else if (u.sessionUpdate === "tool_call_update" && tools.has(u.toolCallId))
          tools.set(u.toolCallId, tools.get(u.toolCallId)!.replace(/\[[^\]]*\]$/, `[${u.status ?? "…"}]`));
        onUpdate?.({ content: [{ type: "text", text: text.slice(-2000) }], details: details() });
      };
      w.ask = ctx.hasUI
        ? async (req) => {                        // 人在环：把子 agent 的权限请求弹给用户
            const labels = req.options.map((o) => `${o.name} (${o.kind})`);
            const pick = await ctx.ui.select(`[${w.agent}] ${req.toolCall.title ?? "请求授权"}`, labels);
            const opt = req.options[labels.indexOf(pick ?? "")];
            return opt ? { outcome: { outcome: "selected", optionId: opt.optionId } } : rejectAll(req);
          }
        : rejectAll;
      const onAbort = () => void w.conn.agent.notify(acp.methods.agent.session.cancel, { sessionId: w.sessionId });
      signal?.addEventListener("abort", onAbort, { once: true });
      try {
        const res = await w.conn.agent.request(acp.methods.agent.session.prompt,
          { sessionId: w.sessionId, prompt: [{ type: "text", text: p.task }] });
        const out = text.length > 50_000 ? `…[截断]\n${text.slice(-50_000)}` : text;
        return {
          content: [{ type: "text", text: `[${w.agent} session=${w.sessionId} stop=${res.stopReason}]\n${out || "(无文本输出)"}` }],
          details: { ...details(), stopReason: res.stopReason },
        };
      } finally {
        w.busy = false; w.onUpdate = undefined; w.ask = undefined;
        signal?.removeEventListener("abort", onAbort);
      }
    },
  });

  pi.registerTool({
    name: "delegate_close", label: "Close worker", description: "关闭一个 worker 会话",
    parameters: Type.Object({ sessionId: Type.String() }),
    async execute(_id, { sessionId }) {
      const w = workers.get(sessionId);
      w?.conn.close(); w?.proc.kill("SIGTERM"); workers.delete(sessionId);
      return { content: [{ type: "text", text: w ? `closed ${sessionId}` : "not found" }], details: undefined };
    },
  });

  pi.on("session_shutdown", async () => {          // 幂等清理
    for (const w of workers.values()) { w.conn.close(); w.proc.kill("SIGTERM"); }
    workers.clear();
  });
}
```

后续可以扩展的方向：

- 后台模式：`execute` 立即返回，任务完成后通过 `pi.sendMessage(..., {triggerTurn:true, deliverAs:"followUp"})` 通知。
- 用 overlay 做一个 worker 面板。
- 用 `appendEntry` 记录 sessionId，重启后通过 `session/load` 恢复。
- SIGTERM 之后补一个 SIGKILL 兜底。
- 接 Codex app-server 后端，实现真正的 `turn/steer`。

## 6. 风险与待验证事项

- 适配器发版频繁，需要锁定 claude-agent-acp、codex-acp 和 ACP SDK 的版本。
- ACP 原生的子代理会话（`clientCapabilities.subagents`）还处于草案阶段。
- codex-acp 首次运行可能需要先走 `authenticate`。
- pi-durable 是实验性项目，API 可能变动。

## 来源

- https://github.com/deepseek-ai/deepseek-harness（packages/subagent/*、docs/subsystems/subagent.md、.agents/notes/archived/feature/2026-08-04-claude-code-and-codex-subagent-backends.md）
- https://github.com/earendil-works/pi（coding-agent/docs/{extensions,rpc,sdk,providers}.md、examples/extensions/subagent、packages/durable、packages/chord、packages/env）
- https://github.com/agentclientprotocol/typescript-sdk 、/claude-agent-acp 、/codex-acp
- https://learn.chatgpt.com/docs/app-server
- https://code.claude.com/docs/en/agent-sdk/overview 、https://code.claude.com/docs/en/legal-and-compliance
- https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan
- https://support.claude.com/en/articles/17154008-monthly-api-credits-for-max-and-team-plans
- https://thenewstack.io/sign-in-with-chatgpt/
- https://geminicli.com/docs/cli/acp-mode/
- https://github.com/svkozak/pi-acp 、https://pi.dev/packages/pi-acp-agents 、https://pi.dev/packages/@junghanacs/pi-shell-acp
- https://www.npmjs.com/package/pi-harness-delegate 、https://pi.dev/packages/@danchamorro/pi-subagents
