<!--
6 个小节是模板的一部分，不要删除标题。机判（scripts/pr-grade.sh --check-template）逐条查：
6 小节齐全 + 正文里出现 SAFE_TO_MERGE / CHECK_TIMESTAMP / MAIN_HEAD 三个词。
-->

## 1. 关联 issue（同一 issue 可多 PR）

- 关联 issue：#（填 issue 编号）
- 本 PR 对应的切片名：（填 slice，与分支 `feat/<slice>`、工作树 `wt/<slice>` 一致）
- 同 issue 的其他 PR：（列出编号，或写 0 个）
- 引用写法：使用 `Refs: #<编号>` 或 `#<编号>` 链接；禁止写 `Closes #<编号>`（关闭 issue 需 user 许可，见 R3）

## 2. 本切片交付什么

- 交付内容：（逐条列出可验证的产物，1 条对应 1 个文件或 1 个能力）
- 切片独占范围：（列出本切片在 issue 切片表登记的独占路径，2 个切片不得重叠，见 R4）
- 对应 Spec Kit 阶段：（constitution / specify / plan / tasks / implement 中选 1 个）

## 3. 不做什么

- 明确排除的范围：（列出本切片不碰的目录、不实现的能力、不修改的共享文档）
- 留给其他切片的项：（列出切片名与 PR 编号，或写 0 个）

## 4. 验收方式

- 复现命令：（逐条写出可复制执行的命令，例如 `bash scripts/pr-grade.sh --pr <N>`）
- 预期结果：（写出可判定的结论，例如输出 0 个脏文件、0 处冲突、0 处不可达引用）
- 依赖条件：（写出需要的主线状态、环境、数据）

## 5. pre-merge 检查结果（贴脚本完整输出）

- 执行命令：`bash scripts/pre-merge-check.sh origin/<本 PR 分支> main`
- 粘贴下方脚本完整输出，必须同时含 `SAFE_TO_MERGE`、`CHECK_TIMESTAMP`、`MAIN_HEAD` 三项；
  缺任一项、或仅粘贴 `BLOCKED`，不得合并：

```
（在此粘贴脚本完整输出，保留 CHECK_TIMESTAMP= 与 MAIN_HEAD= 两行原文）
```

- 合并前复核的 status 与 HEAD 一致性结论：（填写 `git status --porcelain`、`git rev-parse main`
  是否与检查时状态及 `MAIN_HEAD` 一致；不一致不得合并。TTL <10 分钟，超时须重跑）
- 提交前自检：
  - [ ] 脚本输出三项齐全：`SAFE_TO_MERGE`、`CHECK_TIMESTAMP`、`MAIN_HEAD`
  - [ ] 主线含 staged、unstaged、untracked 共 0 个脏文件
  - [ ] merge-tree 试算 0 处冲突
  - [ ] 本地 main 与远端 main 一致
  - [ ] PR 分支已包含 main 最新提交
  - [ ] 每个 PR 合并前单独重跑本检查，未复用其他 PR 的结论
  - [ ] TTL <u>10 分钟</u>内执行合并，超时已重跑完整检查
  - [ ] 检查时未使用 `--no-verify`，未执行 checkout --、reset、stash、clean、强推

## 6. 测试与截图

- 测试命令与结果：（命令、通过数、失败数；失败数大于 0 个时不得勾选完成）
- 引用可达性检查：`python scripts/check-refs.py --pr <N>` → （N 个文件，N 处不可达引用）
- 截图 / 录屏：（贴链接或相对路径，或写 0 张并说明原因）
- 验证人 / 验证时间：（填写）
