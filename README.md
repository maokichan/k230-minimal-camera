# 基于 K230 实现的极简相机

立创·庐山派 K230-CanMV 开发板上的极简相机：摄像头取图 → JPEG 编码 → 存 TF 卡。板端固件走 CanMV（MicroPython），仓库治理脚本走 Node（零依赖 ESM）。

## 硬件

- 主控：嘉楠 Kendryte K230——玄铁 C908 双核 RISC-V（大核 1.6GHz 带 RVV 1.0，小核 800MHz），KPU ≈6 TOPS（INT8）
- 板卡：立创·庐山派 K230-CanMV（1GB 内存版），GC2093 摄像头（MIPI CSI-2），TF 卡启动固件
- 开发环境：CanMV 固件 + CanMV IDE（桌面/Web）

## 目录结构

- `src/` — CanMV 应用代码（相机主程序）；域规则见 [src/AGENTS.md](src/AGENTS.md)
- `docs/` — 项目文档与硬件调研资料（索引：[docs/research/INDEX.md](docs/research/INDEX.md)）；域规则见 [docs/AGENTS.md](docs/AGENTS.md)
- `ops/` — 仓库治理脚本（docIndex / repoGuard / docGuard 三守卫）；域规则见 [ops/AGENTS.md](ops/AGENTS.md)
- `.agents/` — 随库 AI 配置：[notes/](.agents/notes/README.md)（决策笔记）与 `skills/`（generate-swe 纪律）
- `trash/` — 删除的文件移到这里（git 忽略，保留可恢复性）

## 快速上手

1. 激活门控（clone 后一次性）：`git config core.hooksPath .githooks`
2. 体检：`node ops/scripts/docGuard.mjs --report`、`node ops/scripts/repoGuard.mjs --report`、`node ops/scripts/docIndex.mjs check`
3. 板子到手先查 [docs/research/INDEX.md](docs/research/INDEX.md)：固件烧录、相机 API、CPU 注意点都在调研文档里

## 协作

模型侧纪律见 [AGENTS.md](AGENTS.md)（子树地图与全局红线）；开工手册见 [docs/workflow.md](docs/workflow.md)；机制为什么这样设计见 [docs/rationale.md](docs/rationale.md)。
