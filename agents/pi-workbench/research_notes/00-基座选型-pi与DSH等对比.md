# 基座选型：pi 与 DSH 等 harness 的对比

> 调研日期：2026-10-12。结论被 [../README.md](../README.md) 的方案设计采纳。

## 结论

pi 适合作为个人 agent 工作台的基座，前提是愿意用 TypeScript 写扩展、自己补上上层能力。DSH 架构最激进，但目前还是 alpha，更适合作为设计参考。

## pi

- **归属**：Mario Zechner 开发，2026-04 被 Earendil（Armin Ronacher 的公司）收购。
- **版本**：2026-10-02 发布 1.0，2026-10-12 HEAD 为 v1.1.0。
- **许可**：MIT。
- **分层**：
  - `pi-ai`：多厂商 LLM 统一接口
  - `pi-agent-core`：agent 循环
  - `pi-coding-agent`：CLI 和 SDK，入口是 `createAgentSession()`
  - `pi-tui`：终端 UI
  - `pi-durable`（实验性）：长时运行、checkpoint、后台 task
  - `chord`：跨进程对象和 RPC
- **优点**：
  - 可按层取用。
  - 已有大型产品验证：OpenClaw 在进程内嵌 pi SDK。
  - 系统提示词不到 1k token。
  - 扩展在进程内运行，有 25+ 个钩子。
  - 1.0 起支持 MCP（codemode）、虚拟模型、工具按需加载、缓存预热。
- **短板**：
  - 没有内置权限和沙箱。
  - 不内置子 agent 和 plan mode。
  - 只支持 Node/Bun。
  - 扩展有完整的进程内权限，引入第三方扩展需要审代码。

## DeepSeek Harness（dsh）

- **基本情况**：2026-08-13 发布，MIT，最新版本 v0.2.1-alpha，官方明确提示会有破坏性变更。
- **架构**：基于 Cordis 插件框架，一切皆插件，agent 循环和 UI 都可以替换。
- **自带功能**：
  - Web UI 和桌面端
  - Trajectory 视图
  - 子 agent：可以委派给 Claude Code、Codex 或任意 ACP agent
  - agent 团队、定时任务
  - Creator mode：用对话描述需求，生成插件
- **模型**：DeepSeek、Anthropic、OpenAI、Bedrock、Vertex、Azure，以及任何 OpenAI 兼容接口。
- **风险**：
  - 会话日志默认上传到 DeepSeek。
  - 沙箱不限制文件读取和网络访问。
  - GitHub issue 区已关闭。
  - 替换核心插件后，每次上游升级都要维护自己的补丁。

## 其他选项

| | 定位 | 适合场景 | 当作基座的问题 |
|---|---|---|---|
| OpenCode | 功能齐全的编码 agent 成品，client/server 架构 | 想直接用、少折腾 | 更像产品而非库，深度定制要写插件或 fork |
| Hermes Agent（Nous） | 带自学习技能和多级记忆的个人助理，Python | 主要用在聊天渠道里的常驻助理 | 设计取向强，是成品而非内核 |
| OpenClaw | 消息网关 + 个人助理，内核就是 pi | 多渠道助理 | 侧重聊天渠道 |
| Claude Agent SDK / Codex | 厂商官方 harness | 只用一家模型 | 绑定厂商 |

## 来源

- https://github.com/earendil-works/pi
- https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/sdk.md
- https://www.trendingtopics.eu/pi-1-0-earendil-en/
- https://rywalker.com/research/pi
- https://docs.openclaw.ai/pi
- https://github.com/deepseek-ai/deepseek-harness
- https://www.eesel.ai/blog/deepseek-harness
- https://opper.ai/apps/deepseek-harness
- https://composio.dev/content/pi-vs-opencode
- https://insights.reinventing.ai/ai-agent-harnesses
- https://hermes-agent.nousresearch.com/docs
