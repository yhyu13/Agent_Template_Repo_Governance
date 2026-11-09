# .agent/inbox — 候选沉淀

agent 不直接编辑 AGENTS.md / memory；学到的坑、约定、事实写到这里，一条一文件，人审后合入。

文件名：`<YYYY-MM-DD>-<短名>.md`

```markdown
---
type: pitfall | convention | fact | idea
confidence: verified | suspected
suggested-home: AGENTS.md#<节> | memory/core/<篇> | 新 skill | 无
source: "<跑通的命令 / 文件路径 / 错误码>"
---

一句话结论。

为什么（带 source 指向的证据）。

怎么防再犯（可断言的那一步）。
```

审阅规则：confidence 为 suspected 且补不出证据 → 直接删。已合入或已否决 → 删文件。
不长期积压。
