# Runtime

Runtime 层记录 agent 运行时的核心循环、运行事件和执行主体切换。

它关注 run 如何被推进、记录、转交，以及当系统接入多个 external runtimes 时如何路由、恢复和 fallback。一个外部项目如果提出新的 loop boundary、event model、handoff semantics、runtime adapter 或可恢复执行方式，通常先从这里归类。

- [[harness/runtime/agent-loop/index|Agent Loop]]
- [[harness/runtime/event-log/index|Event Log]]
- [[harness/runtime/handoff-transfer/index|Handoff & Transfer]]
- [[harness/runtime/runtime-brokerage/index|Runtime Brokerage]]
