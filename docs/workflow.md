# 开工流程（给模型）

读者：在本仓库工作的模型。本文件假设你已按根 AGENTS.md 的子树地图到达这里；各步规则链接到相应子树入口。它把一次会话串成操作序列——拿不准任何一步 = 停下来问人。

## 开工

1. `git pull --ff-only`（无远端时跳过）——禁止在过期工作区上动手。
2. 读根 AGENTS.md 的子树地图，只读本次要动的子树的 AGENTS.md。
3. 查 [.agents/notes/](../.agents/notes/README.md)：有没有覆盖本次工作的决策（写新笔记前先查旧笔记）。
4. 动硬件/固件相关代码前，先查 [research/INDEX.md](research/INDEX.md) 与其指向的调研文档。

## 工作中

- CanMV 代码：遵守 [../src/AGENTS.md](../src/AGENTS.md)。
- 写文档：先想归属与级别（[AGENTS.md](AGENTS.md)）；改动触碰某文档声明跟踪的事实时，同步更新该文档。
- 需要拍板的决策：写 proposed 笔记或直接问人——不替人决定。
- 装依赖、动门控、改目录与类别闭集：先提 proposed 决策笔记征得同意。

## 收工

1. 体检（手动可跑）：`node ops/scripts/docGuard.mjs --report`、`node ops/scripts/repoGuard.mjs --report`、`node ops/scripts/docIndex.mjs check`。
2. 提交（必经门控）：守卫在 pre-commit 自动跑，报错按修法改；`--no-verify` 不用。
3. **push 归人**：提交后报告即可。

## 红线

红线全文在根 AGENTS.md（你的来路），门控与越权细则在 [../ops/AGENTS.md](../ops/AGENTS.md)——不在本文件重复。
