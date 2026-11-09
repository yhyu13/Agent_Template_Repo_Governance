# Git Workflow — issue → 切片 → worktree → PR → 独立评审 → 评级 → 合并

多 agent 并行改一个仓的操作手册。命令为 bash；Windows 上用 Git Bash，PowerShell 差异见文末注。

## 0. Bootstrap（一次性，owner 做）

```bash
git status --short && git log --oneline -1 && git remote -v   # 三条都必须正常
gh auth status                                                  # 必须有 repo scope
python scripts/check-refs.py --all                             # 必须 0 处不可达引用
```

## 1. 模型

- 一个 issue = 一个目标，切成 **2 个及以上切片**；每切片 = 一个 worktree + 一个分支 + 一个 PR。
- Trunk-based：main 受保护、始终可发布；分支短命（目标 < 2 天）。
- Worktree 位于 `wt/<slice>`，分支 `feat/<slice>`，两者用同一个 `<slice>`。
- **并行不靠自觉，靠登记**：每个切片的独占范围写进 issue 切片表（机读格式），
  `ownership-check` 逐文件核对，越界即红。

## 2. 角色与交接物

| 角色 | 负责 | 绝不做 |
|---|---|---|
| Coordinator | issue 拆解、独占范围表、集成 | 在 worker 的 worktree 里实现 |
| Worker | 恰好一个 worktree 一个切片：实现+证明+PR | 动别人的路径、审自己、无独立批准就合 |
| Reviewer | 独立上下文对抗式审查，出 `GRADE=` 块 | 是作者、与作者共享 worktree/会话 |

交接靠文件不靠聊天：issue 契约 → 切片表 → 分支 → PR 正文（模板见 `.github/PULL_REQUEST_TEMPLATE.md`）→ 验证日志。

## 3. Issue 契约（coordinator 逐字写入 issue）

标题动词开头；验收标准每条可机械判定（命令退出码 / 字节数 / 截图）；写明 In scope / Out of scope
路径、验证命令（reviewer 重跑的那一条）、阻塞边（#N 先合）。判不出来的验收标准改到能判为止。

**独占范围登记**（机读格式，逐字照写，见 issue 模板）：

```
- `<slice>` → `<独占目录1>`、`<独占目录2>`
```

## 4. Worktree 生命周期

```bash
bash scripts/new-slice.sh <slice-name> [base]     # 创建 worktree + 分支（唯一写操作是 git worktree add）
cd wt/<slice> && git status --short              # 确认隔离
# ...实现、跑 issue 里的验证命令...
git add <只加自己拥有的路径>
git commit -m "feat(<slice>): <主题>" -m "<交付内容、验证命令、验证结论>" -m "Refs: #<issue>"
git push -u origin feat/<slice>
gh pr create --base main --head feat/<slice> --title "..." --body-file /tmp/pr-<issue>.md
```

提交信息格式见 AGENTS.md「提交信息格式」：`<type>(<slice>): <主题>` + 正文 + `Refs: #<编号>`，
禁止 `Closes`。

## 5. 合并前检查（R1，不可跳过）

```bash
bash scripts/pre-merge-check.sh origin/feat/<slice> main
```

输出 `SAFE_TO_MERGE`（含 `CHECK_TIMESTAMP` / `MAIN_HEAD`）或 `BLOCKED`。
**完整输出**贴进 PR 正文第 5 小节；TTL 10 分钟，超时重跑；合并前再核对一次
`git status --porcelain` 与 `git rev-parse main`。

> 同步基线时用**合规 merge message**，否则 commit-lint 必红、PR 永远失去绿级资格：
> `git merge origin/main -m "chore(<slice>): 同步 main 基线" -m "Refs: #<issue>"`

## 6. 独立评审与评级（R5）

```bash
# 评审者（独立上下文 subagent）跑：机判 + 内容级复核
bash scripts/pr-grade.sh --pr <N>
```

评审者把 `GRADE=` 块原样贴进 PR 评论，并逐条回复自己的 FINDINGS。

- **绿**：机判 8 项全过且无遗留意见 → agent 自动合并。
- **黄**：需 user 批准；user 批准后由 agent 代执行，**批准引用写进 PR 评论**。
- **红**：仅 admin 账号执行合并；agent 禁止执行。

评级后 PR 出现任何新 push → 评级作废，重新评级。

## 7. 并行规则

- 单机并发 ≤ 3–4 个 worker；共享契约（接口/schema/文件布局）先在 main 稳定再拆分。
- 一个分支一个 worktree 一个写者，按独占范围切，不按任务名切。
- 永不 `git stash`；端口/构建目录从 `WORKTREE_PATH` 派生，不复用固定值。
- 共享正本默认只读；要写先登记为独占（见 R4）。

## 8. 冲突与失败协议

1. 请求 review 前：worktree 内 `git fetch origin && git merge origin/main`（用合规 merge message）。
2. 冲突在 worktree 内解决，重跑验证命令，只对自己分支推送。
3. 验证失败 → PR 不合并，evidence 块如实记录失败命令与退出码；不合并「基本绿」。
4. 合并后集成失败 → main 上新 PR `git revert <sha>`，不改写历史。
5. 被别的 issue 阻塞 > 1 天 → coordinator 重新切分，不悄悄扩大范围。

## 9. 文档引用检查（§1.5）

```bash
python scripts/check-refs.py --all        # 全仓
python scripts/check-refs.py --pr <N>     # 只扫该 PR 触及的文件
```

- 路径 / md 锚点 / `file:line` 越界 → **机器必查**，退出码非 0 即失败（已接入 pr-gate 第 5 闸门）。
- 引用指向的文件存在、但内容/列名/版本号对不上 → **只能独立 reviewer 判**。
- 两份正本说法相反 → 独立评审 + 状态标记 + 正本单一化。
- 确认属于外部仓路径或已登记待产出的物料 → 写进 `.refcheck-ignore`，并在 PR 正文说明理由。
- 整篇在讲「还缺什么」的文档 → 文件头写 `refcheck:skip-file` 声明；产物入库后移除。

## 10. 合并政策

- PR 正文必须有 `Refs: #N`（非用户批准不写 Closes）、pre-merge 三件套、风险/回滚。
- 每个 PR 独立评审；reviewer 亲跑验证命令，APPROVED/GRADE 评论记录命令、结果、reviewed SHA。
- review 后 head 有任何新 commit / merge / force-push → 批准作废，重新审。
- 合并闸门清单见 `AGENTS.md`「R1 合并前的冲突与基线检查」与「R5 合并分级与授权」。
- PowerShell 5.1 注：无 `&&`；`gh pr create` 用 `--body-file` 不用多行 `--body`；
  Windows 上 `python3` 可能是 Store 空壳存根，用 `python`（`scripts/pr-grade.sh` 里的 `pick_python` 会自动挑）。
