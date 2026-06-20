# Context

Context 层记录输入如何进入模型、prompt 如何组装、上下文如何注入、预算如何控制，以及历史如何压缩恢复。

它关注“模型这一轮看到了什么、为什么看到、以什么结构看到”。这里不写 application-specific prompt 文案，而是记录 Input Transform、Prompt Assembly、Context Injection、Budgeting 和 Compaction 这些可迁移的机制。

- [[harness/context/input-transform/index|Input Transform]]
- [[harness/context/prompt-assembly/index|Prompt Assembly]]
- [[harness/context/context-injection/index|Context Injection]]
- [[harness/context/context-budgeting/index|Context Budgeting]]
- [[harness/context/compaction/index|Compaction]]
