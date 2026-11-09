# Review — 独立评审与分级

reviewer 不是「看一眼说不错」，是**独立上下文**下按可检查的标准逐条核，并把结论写成机器可解析的块。

## 0. 硬要求

- **独立上下文**：reviewer 必须是 fresh-context 的 subagent，不复用实现者的会话；实现者不得给自己的 PR 出评级。
- **只读**：reviewer 只读 diff / issue / 规格 / 规则，不写文件、不提交、不合并、不评论 GitHub。
- **给可检查的标准**，不要给「好不好」：「AC-13 必须是真实表格行」可检查；「文档写得不错」不可检查。
- **每轮只修阻塞项**；非阻塞意见也记下来，但不要为它开新一轮。

## 1. 评审前必跑

```bash
bash scripts/pr-grade.sh --pr <N>        # 机判（FR-603/604/605 的机器可判项）
bash scripts/pre-merge-check.sh origin/<branch> main   # R1 快照（确认 reviewer 评的是当前 head）
python scripts/check-refs.py --pr <N>     # 引用可达性（若该 PR 触及文档）
```

## 2. 八项机判（FR-603，绿级条件）

| # | 项 | 怎么查 |
|---|---|---|
| 1 | CI required checks 全绿 | `statusCheckRollup` 逐个 check 的 conclusion |
| 2 | H1 `SAFE_TO_MERGE` 且未超 TTL | 重跑 `pre-merge-check.sh`，比对 `CHECK_TIMESTAMP` 与 `MAIN_HEAD` |
| 3 | 越界 0 处、契约漂移 0 处 | `--check-ownership` |
| 4 | diff 全落本切片独占目录 | 同上 |
| 5 | 不碰治理面/正本 | `GOV_PATHS_RE`（见 `.agent-gate.conf`） |
| 6 | 无风险面命中 | 删除/重命名、二进制、force-push、凭据、新增依赖 |
| 7 | 测试失败数 = 0 | **自己重跑**，不要只读 PR 正文 |
| 8 | 模板 6 小节 + 三件套原文 | `--check-template` |

## 3. 机判查不出来的（reviewer 的真正战场）

| 类别 | 典型问题 | 怎么查 |
|---|---|---|
| **内容相符** | 引用指向的文件存在，但列名 / 措辞 / 版本号对不上 | 打开被引文件**逐字比对**，不靠记忆 |
| **跨正本口径** | 两份正本对同一件事说法相反 | 把相关正本并排读，找口径冲突 |
| **自证失效** | 举的「证据」其实指不到位置（例子：把「同一份文档 2.2 vs 5.1」写成立，但 5.1 里根本没那句话） | 评审给出的位置**亲自去看一遍**，指不到就是问题 |
| **未验证写成完成** | 「0 处不可达引用」其实靠 ignore 掩盖 | 把 ignore 置空重跑，看真实数字 |
| **越界改动** | 改了别的切片/共享正本，且语义还写错了 | 读 issue 的独占登记，逐文件核对；顺带确认**改对了没有** |
| **工具自身的缺陷** | 解析逻辑有假阴性/假阳性 | 造反例实跑（浅/深目录、`../` 越界、首段拼错、裸文件名…） |

> 经验：交付者最容易在**自己的交付里**出错，而机判恰好看不见那些。
> 评审的价值不在重复机判，在抓「机判只看得见结构」的那一半。

## 4. 输出格式

贴进 PR 评论：

```
GRADE=GREEN|YELLOW|RED
INPUTS: ci=<pass|fail> h1=<pass|fail|stale> h2_cross=<n> h2_drift=<n> d2_hit=<n> d3_hit=<n> test_fail=<n>
REASONS: <逐维 1 行；绿级写 8 项判定结果，黄/红写命中条目编号>
BLOCKERS: <黄/红必填；绿级写 0 条>
REVIEWER: <subagent 标识 + fresh context 声明>
CHECK_TIMESTAMP: <ISO 8601 UTC>
FINDINGS:
- [级别 minor|major/critical] <位置：文件 + 引用原文> — <问题> — <为什么是问题> — <建议>
```

FINDINGS 每条必须给出**位置**与**原文引用**。核对了什么、没核对什么，写清楚。

## 5. 分级与授权

| 等级 | 判定 | 批准主体 | merge 执行 |
|---|---|---|---|
| 绿 | 机判 8 项全过且无遗留意见 | reviewer 即可 | agent 自动合并 |
| 黄 | 命中任一黄级条目 | user | user 批准后 user 或 agent 代执行（PR 评论留批准引用） |
| 红 | 命中任一红级条目 | admin | 仅 admin 执行；agent 禁止执行 |

- **只升不降**：黄/红不能被 agent 或 reviewer 降级；降级只能由 user/admin 在 PR 评论写明原等级、新等级、理由、改判人。
- **评级时效**：评级后 PR 出现任何新 push → 评级立即失效，重新评级。
- **红级触发器**：改治理面（`GOV_PATHS_RE` 内，含 `.agent-gate.conf` 自身）、所有权越界 ≥1 处、
  force push / `--no-verify` / 篡改检查记录、凭据命中、删除正本。

## 6. APPROVED 评论（仓库用 `Refs` 风格）

```text
APPROVED · PR #<N> · reviewed SHA <full-sha>
验证命令：<命令> → <退出码 + 关键输出，≤10 行>
判定：<逐条结论>
遗留：<非阻塞意见，或 无>
```

reviewer 亲跑验证命令并记录**命令、结果、reviewed SHA**。只贴「看起来没问题」不算批准。
