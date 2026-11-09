# Agent Template Repo — 多 Agent 协作治理版

<!-- TODO(agent): 本 README 是骨架。复制本模板到新项目后，由 agent 补齐：
     1) 项目一句话介绍（是什么、给谁用、当前阶段）；
     2) Quickstart（clone 后到能跑起来的最小命令序列）；
     3) 指向 AGENTS.md / docs/ 的导航表；
     4) License 名称（与 LICENSE 文件一致）。
     补完删除本 TODO 块。在模板仓库自身中使用时保持骨架不动。 -->

多 agent 并行改一个 GitHub 仓的模板：issue → 切片 → worktree → 独立评审 → 绿黄红分级 → 证据合并，
外加**文档引用可达性**这道常驻闸门。

## 装了什么（与基础模板的差别）

| feature | 解决什么 |
|---|---|
| **R1 合并前冲突与基线检查** | 合并前跑一次只读脚本：主线脏检查（staged/unstaged/untracked 三类计数）、`merge-tree` 非破坏性试算、基线包含检查、本地与远端同步；输出 `SAFE_TO_MERGE` + `CHECK_TIMESTAMP` + `MAIN_HEAD`，**TTL 10 分钟** |
| **R2 默认分支与停止条件** | 不在默认分支、user 又没说清楚时，agent 必须停下来问，而不是"先写着回头再改" |
| **R3 issue 纪律** | 5 类事件各发一条进度评论；issue 只能由 user 关 |
| **R4 单 issue 多切片多 worktree** | 每切片 = 1 worktree + 1 分支 + 1 PR；独占范围在 issue 里**机读登记**，越界由 `ownership-check` 拦 |
| **R5 绿黄红分级与授权** | 绿=agent 自动合、黄=要 user 批准、红=只有 admin 执行；**独立上下文 reviewer** 出 `GRADE=` 块；只升不降、评级随 push 失效 |
| **pr-gate 5 道机器闸门** | commit-lint / ownership-check / syntax / template-check / **refs-check**——每个都**真跑命令**，不从 PR 正文里 grep 结论 |
| **文档引用可达性检查** | `scripts/check-refs.py`：路径、md 锚点、`file:line` 越界；`../` 按**仓内相对层级**解析（换 checkout 深度结论不变）；首段拼错会报；`.refcheck-ignore` 只登记外部仓路径与已登记待产出的物料 |
| **引用检查归属分级** | 可达性=机器查；内容相符=独立 reviewer 判；跨正本口径=评审回路 + 正本单一化 |
| **4 个常驻 skill** | truth-seeking-research（求真）、prompt-pro（提纲）、skill-distill（沉淀）、hackathon-pitch-pipeline（黑客松交付流水线） |
| **沉淀 inbox** | agent 学到的坑写 `.agent/inbox/`，人审后合入 AGENTS.md；agent 不直接改 AGENTS.md |

## 快速开始

```bash
git clone <this-repo> my-project && cd my-project
git checkout -b feat/<slice>                      # 或用 scripts/new-slice.sh
bash scripts/new-slice.sh <slice-name> main        # 建 worktree + 分支
# 改完后：
python scripts/check-refs.py --all                 # 引用可达性
bash scripts/pr-grade.sh --pr <N>                  # 机判
bash scripts/pre-merge-check.sh origin/feat/<slice> main   # R1
```

CI 需要 GitHub Actions；`dev-sync.yml` 维护一个 `dev` 稳定分支，不需要就删掉。

## 复制后必改清单

| 位置 | 动作 |
|---|---|
| `AGENTS.md` 所有 `<占位符>` | TODO(agent): 按项目实际填写；不需要的小节整节删 |
| `.agent-gate.conf` | TODO(agent): 填本项目的治理面清单、默认 issue、分支与工作树前缀 |
| `docs/GIT_WORKFLOW.md` 的 `<...>` 段 | TODO(agent): 换成真实命令与脚本名 |
| `README.md` 本块 | TODO(agent): 补齐后删除 |
| `LICENSE` | TODO(agent): 确定许可证后替换占位（见文件内说明） |
| `.github/CODEOWNERS` | TODO(agent): 填入真实文件所有权表 |
| `.github/workflows/pr-gate.yml` | TODO(agent): 保留 5 道闸门；按项目补 lint/test job |
| `docs/REVIEW.md` / `docs/TESTING.md` | fill later on demand：有真实 review/test 实践后再填 |

## 结构

- `AGENTS.md` — agent 行为规范（R1~R5、硬红线、合并闸门、引用检查归属、沉淀纪律）。`CLAUDE.md` 只做 `@AGENTS.md` 导入。
- `docs/GIT_WORKFLOW.md` — 全流程操作手册；`docs/REVIEW.md`（reviewer 清单与分级）、`docs/TESTING.md`（验证分级）按需填。
- `scripts/` — `pre-merge-check.sh`（R1）、`new-slice.sh`（切片/worktree）、`pr-grade.sh`（R5 机判 + 5 个子检查）、`check-refs.py`（引用可达性）。
- `.github/workflows/` — `pr-gate.yml`（每 PR 5 道闸门）、`dev-sync.yml`（main → dev 自动同步）。
- `.github/` — PR 模板（6 小节 + pre-merge 三件套）、issue 契约表单（含切片表与机读独占登记）、CODEOWNERS。
- `.agent-gate.conf` — 项目相关判定参数（治理面清单可配置，但**自保护**：它自己也在治理面里）。
- `.refcheck-ignore` — 引用忽略清单（外部仓路径 / 已登记待产出的物料）。
- `.claude/skills/` — 四个常驻 skill。
- `.agent/inbox/` — agent 产出的候选沉淀，人审后合入；agent 不直接改 AGENTS.md。

## 占位标记约定（全仓库通用）

- `TODO(agent)`：留给 agent 按上下文填，填完删标记。
- `fill later on demand`：文件先给骨架，出现真实实践再填。
- `DELETED-ON-PURPOSE`：故意不提供的内容，就地留一行说明，防止后续会话误补。
