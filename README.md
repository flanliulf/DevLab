# DevLab

个人开发工作台：放日常 coding 提效工具的方案和配置，以及技术调研、积累、讨论的笔记。

## 目录

| 一级目录 | 收录范围 | 现有内容 |
|---|---|---|
| [agents/](agents/) | AI coding agent：选型、多 agent 运行与编排、agent 工作台 | [herdr 实施手册](agents/herdr/herdr-完整实施手册-最终版.md)、[基于 pi 的多 agent 工作台方案](agents/pi-workbench/README.md) |
| [remote-dev/](remote-dev/) | 远程开发与多机协作：SSH/网络、远程桌面、跨机文件和剪贴板 | [一台 Mac 掌控另一台 Mac](remote-dev/一台Mac掌控另一台Mac/一台Mac掌控另一台Mac.md) |

下面几个一级目录先不建，等有内容时再按这个划分加：

| 一级目录 | 收录范围 |
|---|---|
| `toolchain/` | 终端、shell、编辑器、CLI 工具的选型与配置 |
| `workflow/` | 日常提效脚本、自动化、Git 工作流 |
| `languages/` | 语言、框架、运行时的专题调研 |

## 约定

- **一个专题一个目录**，放在对应的一级目录下。专题目录内：
  - 主文档：`README.md` 或语义化文件名，结论写在最前面。
  - `research_notes/`：调研原始笔记，主文档会引用。
  - `历史版本/`：被合并或取代的旧文档，只作对照。
- 文档开头写清**调研日期**。时效性强的数据（版本号、star 数、价格）以当时为准，复用前先核对。
- 跨专题引用用相对路径。

## 近期状态

- herdr 上游已迁到 `herdrdev/herdr`，最新稳定版 v0.9.3（2026-09-29）。手册里写的还是 v0.9.1，落地前先核对。
- `agents/pi-workbench` 目前是方案阶段，下一步见其中第 5 节 P0。
