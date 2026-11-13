# AGENTS.md — <项目名>

<!-- TODO(agent): 复制本模板后，把所有 <占位符> 换成真实内容；不需要的小节整节删除，不要留空壳。 -->

<一句话：这个仓库是什么、给谁用、当前处于什么阶段。>

本仓是**多 agent 并行协作**的仓：多个 agent 同时改不同的切片，靠 issue 契约、文件所有权、
机器闸门与独立评审保证不打架。下面的规则与脚本都是可执行的，不是口号。

## 占位标记约定

- `TODO(agent)`：需要 agent 按上下文填写的占位，填完删除标记。
- `fill later on demand`：文件先给骨架，出现真实实践再填。
- `DELETED-ON-PURPOSE`：故意不提供的文件/小节。在对应位置留一行说明为什么不要，防止后续会话误以为缺失而补错。

## 事实优先级

文档冲突时：**当前代码 > 本文件 > docs/ 下旧长文**。
产品规格冲突以 docs/spec.md 为准。本文件与角色契约冲突时以本文件为准。

## Read first

| 文件 | 内容 |
|---|---|
| docs/GIT_WORKFLOW.md | issue → 切片 → worktree → PR → 独立评审 → 评级 → 合并 全流程与命令 |
| docs/REVIEW.md | reviewer 检查清单、APPROVED 评论格式、绿黄红分级与授权 |
| docs/TESTING.md | 验证分级（L1 静态 / L2 行为 / L3 构建 / L4 真实环境） |
| .agent-gate.conf | 项目相关的判定参数（治理面清单、默认 issue、分支与工作树前缀） |

## 硬红线

1. **不经用户明确指令不 commit / push**；禁止直推 main，禁止 force-push main，禁止 `git stash`（跨 worktree 串扰）。
2. **先有 issue 号，再开分支**；一个 issue = 一个切片 = 一个 worktree = 一个分支 = 一个 PR。
3. **PR 写 `Refs: #N`，禁止 `Closes/Fixes/Resolves`**；issue 只由用户关。合并不等于完成。
4. **证据先于宣布完成**：完成必须有命令输出证明（退出码 / 测试摘要 / 文件清单），否则明确说未验证。
5. **不在 main 上直接改**；动手前 `git branch --show-current` 确认，不对就停下来问。
6. **禁止入库**：密钥、用户数据、构建产物、机器特定绝对路径（用相对路径和环境变量）。
7. **一个文件一个写者**；并行 agent 按 issue 切片表的独占登记切分，越界由 `ownership-check` 拦。

## R1 合并前的冲突与基线检查

任何 PR 合并进主线前（脚本、网页界面、人工代执行都算），按顺序完成 4 步，缺一不可：

```bash
bash scripts/pre-merge-check.sh origin/<pr-branch> main
```

1. **主线脏检查**：`git -C <main-worktree> status --porcelain`，staged / unstaged / untracked 三类分别计数。
   脏文件 ≥ 1 个即判 `BLOCKED` —— `merge-tree` 只能比较已提交的树，看不见未提交改动，
   所以「主线有未提交改动」不能被「试算无冲突」抵消。
2. **非破坏性冲突试算**：`git merge-tree --write-tree --messages <base> <pr-branch>`。
3. **基线包含检查**：`git merge-base --is-ancestor <base> <pr-branch>`。
4. **本地与远端同步检查**：`git fetch` 后比对本地 main 与 origin/main。

四项全过、冲突 0 处、脏文件 0 个 → `SAFE_TO_MERGE`；任一不满足 → `BLOCKED` 并停止合并。

### R1.1 结论的时效性与真实性

`SAFE_TO_MERGE` 是**某一时刻的快照**，不是长期通行证：

- 结论 MUST 使用脚本标准输出原文，不得手工转录或凭记忆粘贴，不得把 `BLOCKED` 改写成 `SAFE_TO_MERGE`。
- 脚本 MUST 同时输出 `CHECK_TIMESTAMP`（ISO 8601 UTC）与 `MAIN_HEAD`（完整 SHA），两者同源。
- **TTL = 10 分钟**：由调用方按 `CHECK_TIMESTAMP` 与当前时间之差判定，超过即失效并重跑完整检查。
- 合并前 MUST 再核对一次 `git status --porcelain` 与 `git rev-parse <base>` 是否与检查时一致。
- 以下任一事件发生，结论**立即失效**并须重跑：主线出现新提交、工作树变脏、PR 有新 push、远端 main 有新提交、超过 TTL。
- 若用 GitHub 网页界面合并，点击前 MUST 用本地脚本重新验证一次；界面显示的「No conflicts」不替代 R1。
- 违反 R1.1 等同违反 R1，按 `BLOCKED` 处理。

### R1.2 明确禁止的动作

`--no-verify`；`git checkout --`；`git restore .`；`git reset --hard`；`git stash`；`git clean`；
强推（`--force` / `--force-with-lease`）；先合再补检查；丢弃或回滚任何未提交改动。

### R1.3 主工作区被他人占用的协调路径

多 agent 并行时，「主线 0 脏」常常不是自己能决定的：别的会话正在主工作区改文件，R1 于是永久 `BLOCKED`。
这时**默认动作是停下，不是清场**。按顺序处理：

1. **先确认是谁**：`git status --porcelain` + `git worktree list`，判断脏文件属于哪个会话/切片。
2. **不要碰别人的文件**：R1.2 的禁止项全部适用。为了让检查变绿而 stash / restore / 删除他人改动，
   等同于丢弃现场，按 `BLOCKED` 处理并记入 issue。
3. **通知写者**：在对应 issue 发进度评论，写明「主线被 X 文件阻塞，PR #N 等待合并」，把等待摆到台面上。
4. **等待上限 1 个工作日**：超时仍无人认领，由 coordinator（或 user）决定：由写者先提交 WIP 到一个分支，
   或明确授权把该路径登记为「冻结路径」后单独处理。
5. **根因规避**：所有切片一律在独立 worktree 里开工（R4），主工作区只做集成与检查。
   主工作区不承担开发，就不会长期脏。

## R2 默认分支与停止条件

开始写任何文件之前，MUST 执行 `git branch --show-current` 与 `git rev-parse --show-toplevel`。

**停止条件**（同时成立时 MUST 立即停止一切写操作）：当前 checkout 不是默认分支；且 user 本次请求
没有明确指出 branch 或 worktree。命中时唯一允许动作：停止写操作，向 user 说明当前分支、worktree 路径、
命中 R2 的判定，并请 user 补充目标。不得自行切换分支，不得「先写着回头再改」。

## R3 Issue 纪律

**关闭 issue**：MUST NOT 自行关闭，除非 user 在当前对话中明确给出关闭许可。

**进度更新**：以下 5 类事件每触发 1 类，至少发 1 条 issue 进度评论，不得攒到最后一次性发：

1. 开工（目标、切片拆分、worktree 与分支已建立）
2. 每完成 1 个可验证切片（交付内容、验证命令、验证结论）
3. 遇到阻塞（原因、已尝试的动作、需要谁解决）
4. 提交 PR（PR 编号、关联 issue、本切片范围、pre-merge 检查结论）
5. 测试结论（测试命令、通过数、失败数、结论；失败数 > 0 时不得报完成）

**多 PR**：1 个 issue 可拆多个切片，每切片 1 个 PR，都关联同一个 issue；不因为「已开过 1 个 PR」就停止拆分。

## R4 单 issue 多切片 多 worktree

1 个 issue = 1 个目标，MUST 拆成 2 个及以上独立可交付切片，每个切片满足：
可独立运行验证、可独立合并、有明确的文件所有权边界。

**一一对应**：每切片 = 1 次 `git worktree add` + 1 个分支 + 1 个 PR。
禁止 2 个切片共用 worktree、分支，或在 1 个 worktree 里做完多个切片再拆 PR。

**文件所有权登记**（机读格式，机判按此解析，务必逐字照写）：

```
- `<slice>` → `<独占目录1>`、`<独占目录2>`
```

2 个切片的独占范围不得重叠。共享正本默认只读；要写必须先在 issue 切片表里登记为本切片独占。

```bash
bash scripts/new-slice.sh <slice-name> [base]   # 唯一写操作是 git worktree add
```

## R5 合并分级与授权

任何 PR 合并前，除 R1 外还要完成一次分级评定：

| 等级 | 判定 | 批准主体 | merge 执行 |
|---|---|---|---|
| 绿 GREEN | 机判 8 项全过且独立 reviewer 无异议 | reviewer subagent 即可 | **agent 自动合并** |
| 黄 YELLOW | 命中任一黄级条目 | user（人） | user 批准后由 user 或 agent 代执行（须在 PR 评论留批准引用） |
| 红 RED | 命中任一红级条目 | admin 账号 | **仅 admin 执行；agent 禁止执行合并** |

**黄级条目**：触碰共享正本但已登记为本切片独占；reviewer 提出非阻塞意见；
存在需人工目验的验收命令（已在 PR 正文声明）；新类型改动未被机判覆盖。

**红级条目**：修改治理面文件（`.agent-gate.conf` 的 `GOV_PATHS_RE` 内，含 `AGENTS.md`、
`scripts/pre-merge-check.sh`、`scripts/new-slice.sh`、`.github/**`、`.agent-gate.conf` 自身）；
文件所有权越界 ≥ 1 处；force push / `--no-verify` / 篡改检查记录；diff 命中密钥凭据；
删除或重命名正本；reviewer 定性判定存在安全或正确性风险。

**硬约束**：R5 任何等级都不豁免 R1~R4；等级**只升不降**（降级唯一路径是 user/admin 在 PR 评论写明理由改判）；
评级后 PR 出现新 push，评级立即失效，必须重新评级；CI 全绿是绿级必要条件，但不替代本地 R1。

评级记录写进 PR 评论，格式见 `scripts/pr-grade.sh` 的 `GRADE=` 块。

## R6 自动化通道（机器提交）

有定时任务的仓（submodule 指针同步、抓取、报表产出）不能每个周期都开一个 PR。允许**唯一一种**
绕过 R1~R5 的写入方式，称为自动化通道。它必须同时满足下面全部条件，缺一条就必须走正常 PR：

**准入**

- 提交者是一个被明确登记的自动化任务（写进 `AGENTS.md` 的通道表：任务名 + 频率 + 允许提交的路径）。
- 提交内容**只包含机器生成的、可追溯到上游哈希的指针/产物**（例如 gitlink、lock 文件、抓取快照）。
  源码、文档、治理面文件一律不得走这条通道。
- 未在通道表登记的自动化，一律视为普通 agent，不得直推。

**护栏**

1. 只 `git add <登记路径>`；禁止 `git add -A`、禁止 `git commit -a`。
2. push 前 MUST `git fetch origin`；落后则 `git pull --rebase`；出现冲突 MUST `git rebase --abort` 后停下报告，
   不得手工解冲突。
3. 禁止 `--force` / `--force-with-lease`；禁止改 `.gitmodules`、分支、标签、治理面文件。
4. 提交信息 MUST 使用登记的前缀（如 `chore(submodule):`），便于把机器提交与人工提交分开审计。
5. 自动化任务的提交不获得 R5 的任何授权：它不评审、不合 PR、不关 issue。

**审计**

- 每条通道在 `AGENTS.md` 用一张表登记，人和审计脚本都能一眼看出「谁在写 main」。
- 通道新增或扩大路径范围，属于治理面改动（红级），须 user/admin 批准。

## §1.5 文档引用的检查归属（分级声明）

文档引用失准分三类，检查手段不同，**不要指望 agent 每次自己记得查**：

| 类别 | 例子 | 谁来查 | 判据 |
|---|---|---|---|
| **可达性** | 引用指向不存在的文件 / 断链 / `file:line` 越界 / md 锚点不存在 | **机器必查**：`scripts/check-refs.py`（`pr-grade.sh --check-refs`，已接入 pr-gate 第 5 闸门） | 退出码非 0 即失败 |
| **内容相符** | 引用指向的文件存在，但列名 / 措辞 / 版本号与被引文件对不上 | **独立 reviewer subagent**：机判读不出语义 | reviewer 提出即按黄级处理 |
| **跨正本口径** | 两份正本对同一件事说法相反 | **独立评审 + 状态标记 + 正本单一化** | 评审回路抓；正本唯一 |

- 「机器查不到的」不等于「可以不管」：可达性由机器兜底，**内容相符与跨正本口径必须由独立 reviewer 兜底**——
  实现者不得给自己的 PR 出评级。
- `.refcheck-ignore` 只登记两类：确认属于外部仓的路径、已登记为「待产出」的物料。
  新增忽略项必须在 PR 正文说明理由；**不得用它掩盖真实漂移**。
- 有意指向尚不存在物料的文档，在文件头写 `refcheck:skip-file` 声明并说明原因；产物入库后移除声明。

## 独立评审（FR-608）

- reviewer 必须是**独立上下文**的 subagent（fresh context），不复用实现者的会话；实现者不得给自己的 PR 出评级。
- 给 reviewer **可检查的标准**（「AC-13 必须是真实表格行」），不要给「好不好」。
- 每轮只修阻塞项；reviewer 的评论原样贴进 PR，处置逐条写明。
- reviewer 产出可被脚本解析的 `GRADE=` 块，写进 PR 评论 → 「合不合格」有留痕、可回溯。

## 验证基线

<!-- TODO(agent): 把下面四条换成真实命令。 -->
- 局部变更：<test / lint / typecheck 中最相关的最小命令>
- 跨模块或交付前：<完整 ci 命令>
- 主流程变更：<e2e 命令>
- 发布前必须本地跑通 <build 命令>：*CI 绿不等于能构建*。

该做到哪一级按「改动是否依赖真实的时间、网络、设备」定，分级规则见 docs/TESTING.md。

**合并前必跑**：`bash scripts/pre-merge-check.sh origin/<pr-branch> main`（R1），
并把完整输出（含 `SAFE_TO_MERGE` / `CHECK_TIMESTAMP` / `MAIN_HEAD`）贴进 PR 正文第 5 小节。

## 上下文纪律

- 不整读未变的文件；重读用 offset/limit。
- 测试/构建全量输出写进 git-ignored log，只看 tail（≤40 行）+ 立即捕获退出码。
- 一次性产物（抓取回显、探针渲染、临时 txt、patch 草稿）一律写 `scratch/`（已 ignore）；
  仓根不得新增临时文件——否则 `git status` 被淹，R1 的脏文件检查也就失去意义。
- 先 `git diff --stat`，只展开在审文件的全量 diff。
- 列目录用 `git ls-files` / `-Depth 2`；定位用 `grep -l`，找到再读。
- 输出预计 >200 行就写临时文件再按需读片段。

## 项目特定约束

<!-- TODO(agent): 本项目的结构性事实（技术栈入口、软删除过滤、公开路由白名单等），
     每条带文件路径。超过 ~15 条时移出入本文件，由人审后归入 memory/ 或删除。 -->

## 沉淀纪律

- **禁止 agent 直接编辑本文件与 memory/core/**（bootstrap 时填占位除外）。学到的坑、约定、
  事实 → 写入 `.agent/inbox/<YYYY-MM-DD>-<短名>.md`：
  `type: pitfall|convention|fact` / `confidence: verified|suspected` /
  `suggested-home: AGENTS.md#<节> | memory/core/<篇>` / `source: "<证据>"`
  + 一句话结论、为什么、怎么防。
- inbox 由人定期审阅合入；suspected 且无证据的直接删。

## 语言

文档、代码、注释、commit message 用英文；与用户交流可用中文。
