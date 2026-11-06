# AGENTS.md — 模型纪律（基于 K230 的极简相机）

本文件只做**向下索引**：每个子树有自己的 AGENTS.md，动哪个子树先读哪份——子树细节不在本文件承载。人类入口是 README.md。项目文档的优先级高于任何技能包。

## 子树地图

| 子树 | 域入口 | 域规则要点 |
|---|---|---|
| `src/` | [src/AGENTS.md](src/AGENTS.md) | CanMV 应用代码、板端事实归属 |
| `docs/` | [docs/AGENTS.md](docs/AGENTS.md) | 项目文档、硬件调研资料、行数口径 |
| `ops/` | [ops/AGENTS.md](ops/AGENTS.md) | 仓库门控、守卫链、路径红线 |
| `.agents/notes/` | [.agents/notes/README.md](.agents/notes/README.md) | 决策笔记机制（路径即元数据） |

## 全局红线（所有子树生效）

- 通用纪律：[generate-swe skill](.agents/skills/generate-swe/SKILL.md)（随库冻结，仓库版为准；开发者本机用户级同名技能会遮蔽本目录，以仓库版为权威）。
- **开工先拉**（`git pull --ff-only`，无远端时跳过），**收工就提交**（必经门控，见 [ops/AGENTS.md](ops/AGENTS.md)）。push、远端与版本号管理归人。
- **路径一律仓库相对**：命令示例从仓库根运行；指代仓库根写 `<仓库根>` 占位；引用仓库外文件属越权动作。
- 提交必过守卫链（pre-commit 自动跑，报错自带修法）；`--no-verify` 是人类的绕行特权，agent 不用。提交身份用 git 自带（user.name/email），不设额外溯源门控。
- **装依赖先问人**：agent 不得未经用户同意安装任何东西——要加就提一篇 proposed 决策笔记（写清用途与替代方案）征求同意。
- 影响项目的决定写 [.agents/notes/](.agents/notes/README.md)（写前先查旧笔记）；表格与状态文件只记事实，理由与放弃项进笔记。
- 删除即移动 `trash/`（随库忽略，可恢复）；不在文档写日期；文档与事实不符必须主动报告。
