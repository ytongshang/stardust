# Tooling

Tooling 层记录工具定义、执行、结果、错误恢复、中断、可用性、权限、远程工具和扩展注册。

它关注 model tool use 到真实执行之间的 contract：tool 如何被描述、暴露、授权、运行、失败恢复和回传结果。这里不把某个 tool 的 application-specific logic 当作概念，只记录 harness 对 tools 的通用管理方式。

- [[harness/tooling/tool-definition/index|Tool Definition]]
- [[harness/tooling/tool-execution/index|Tool Execution]]
- [[harness/tooling/tool-results/index|Tool Results]]
- [[harness/tooling/tool-error-recovery/index|Tool Error Recovery]]
- [[harness/tooling/tool-interruption/index|Tool Interruption]]
- [[harness/tooling/tool-availability/index|Tool Availability]]
- [[harness/tooling/tool-permissions/index|Tool Permissions]]
- [[harness/tooling/remote-tools/index|Remote Tools]]
- [[harness/tooling/extension-registry/index|Extension Registry]]
