# ops/ —— 仓库治理（agent 子树入口）

> 本目录只放**仓库治理**脚本：守卫与声明，防的是仓库因 agent 而变形——路径泄漏、密钥入库、
> 文档膨胀、文档与事实脱钩。板端业务代码在 src/（见 [../src/AGENTS.md](../src/AGENTS.md)），两者分治。
> 红线全表在仓库根 AGENTS.md（你的来路，纯文字提及不作链接）；行数上限口径在 [../docs/AGENTS.md](../docs/AGENTS.md)。

## 脚本（三件套，`<域><动作>.mjs`；Guard 族挡提交，报错自带修法）

| 脚本 | 拦什么 |
|---|---|
| [docIndex](scripts/docIndex.mjs) | 文档内路径引用的目标缺失 = check 模式 notice（不拦截）；md 链接向上（祖先）或越出仓库 = 拦；被引用文件变了而引用文档没动 = 漂移提醒（不拦截） |
| [repoGuard](scripts/repoGuard.mjs) | 绝对路径（三形态，只扫文档/配置扩展）与 API key（八类，扫全部文本源）= 拦；密钥命中不回显；`<占位符>` 豁免 |
| [docGuard](scripts/docGuard.mjs) | Markdown 行数超上限 = 拦（入口类硬上限 80、其余 300；上限表见其头注，动笔前先登记） |

## 门控链（pre-commit；clone 后一次性激活 `git config core.hooksPath .githooks`）

`set -e` 串联：docIndex hook → repoGuard → docGuard，任一失败即中止提交。`--no-verify` 是
人类的绕行特权，agent 不用——门控为纪律而设，不是安全机制。提交身份用 git 自带的
（user.name/email），不设 identity 门控（决策见 [.agents/notes/](../.agents/notes/README.md)）。
体检：各脚本 `--report` / `check`。

## 路径红线（repoGuard 的口径依据）

- 所有路径一律仓库相对：命令示例从仓库根运行；指代仓库根写 `<仓库根>` 占位。
- 引用仓库外文件属越权动作：改写为指认来源的相对引用，或删除。
- 密钥检出 = 立即移出并轮换；占位符（`<二级令牌，可空>` 这类）是文档示例的合法形态。
