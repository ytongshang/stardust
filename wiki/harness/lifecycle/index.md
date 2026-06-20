# Lifecycle

Lifecycle 层记录 agent run 在产品系统里的排队、心跳、挂起、恢复、完成、取消和耐久执行。

它关注 run 在真实系统里如何活下去：进入队列、拿到 lease、执行中 heartbeat、等待外部输入、失败后 retry、完成后收尾。这里和 Runtime 的区别是，Runtime 关心 run 内部怎么推进，Lifecycle 关心 run 在系统调度和持久化里的状态。

- [[harness/lifecycle/queueing/index|Queueing]]
- [[harness/lifecycle/heartbeat-retry/index|Heartbeat & Retry]]
- [[harness/lifecycle/pending-resume/index|Pending & Resume]]
- [[harness/lifecycle/completion/index|Completion]]
- [[harness/lifecycle/cancellation/index|Cancellation]]
- [[harness/lifecycle/durable-execution/index|Durable Execution]]
