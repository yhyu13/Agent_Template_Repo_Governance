---
name: prompt-pro
description: |
  Rewrite the user's rough task request into an executable brief, then execute the brief
  instead of the raw request. Load this skill at the START of any non-trivial task request —
  building, fixing, auditing, writing, deploying, refactoring, investigating. Also trigger on
  explicit asks like "优化 prompt" / "prompt Pro".
  Do NOT use it for: pure Q&A with no state change, single trivial actions, or when the user
  already supplied a precise spec with checkable acceptance criteria.
---

# Prompt Pro

把粗请求补成可执行提纲，再执行提纲。提纲是**输入**，不是交付物。

## Procedure

1. **判定要不要重写**。一句话能做完且不改状态、或用户已给可判定验收标准 → 跳过直接做。
   拿不准就重写——多花几行字，比猜错方向便宜。

2. **先看真实系统，再写提纲**。提纲写在假设上就是一个错的前提。先读源码/配置/现有工具，
   再落笔。grounding 的成本高于提纲本身，这是正常的；提纲是压缩，不是取代 grounding。

3. **补齐缺口**。只用 5 个槽位，只补用户没说的，一个都不发明：

   | 槽位 | 判据 |
   |---|---|
   | 目标 | 一句话说清交付什么。动词开头。 |
   | 验收 | 可判定：退出码、字节数、逐位相等、勾选项 |
   | 范围 | 动什么 + **不动什么** |
   | 约束 | 禁止项、硬红线、必须走的通路 |
   | 输出 | 交付形态：文件 / 提交 / 消息 |

   审计类任务额外加一条：**该说的没说**——文档写得比事实更保守，也算错。

4. **交出去执行**。执行的是提纲，不是原始请求。过程中拿到新信息就更新提纲。

5. **收尾时交代**：补了哪几条、跳过了哪条、有什么假设。不确定的地方标出来。

## Failure handling

- 缺的信息会改变结果、且无法从上下文推断 → 直接问，不脑补。
- 用户否决提纲 → 按用户的改，不争辩。
- 提纲在执行中显得多余 → 说明它多余，然后照做，不为走流程硬塞步骤。
