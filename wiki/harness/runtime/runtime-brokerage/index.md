---
title: Runtime Brokerage
type: concept
created: 2026-06-23
updated: 2026-06-23
tags:
  - agent-harness
  - runtime
  - runtime-brokerage
---

# Runtime Brokerage

Runtime Brokerage 处理“同一个 agent/task 应该交给哪个 external runtime 执行，以及执行失败后如何 fallback、resume、重建上下文和保留治理边界”。

它关注的不是单个 runtime 内部的 Agent Loop，而是多个 runtimes 之间的选择、切换和连续性策略。典型对象包括 Claude Code、Codex、OpenClaw、Hermes 这类外部 coding agent CLI，或同一产品内的 local/remote/runtime-profile 组合。

## 它解决什么问题

当系统不只接一个模型 API，而是接多个 agent runtime 时，会出现一组新的 runtime-level 问题：

- 哪些 runtimes 当前在线、可执行、已授权。
- 同一个 task 应该路由到哪个 runtime。
- provider-native session 能不能 resume，不能时如何 cold rebuild。
- runtime 切换后，哪些上下文必须由平台补回，而不能假设 provider 会话还在。
- 工具、skills、runtime apps、permissions 应该按哪一层计算和注入。
- 失败应该归因为 provider、runtime、configuration、tool 还是 policy。

没有这层，系统通常只能把“换 runtime”当成人工操作，或者把 provider session 当成唯一连续性来源，一旦 session 丢失就无法稳定恢复。

## 常见做法

常见实现会把 Runtime Brokerage 做成一层独立于 agent loop 的 runtime adapter / router：

- runtime catalog：维护每个 runtime 的 provider、版本、健康状态、可执行路径和能力。
- launch adapter：把统一的 task request 转成各家 CLI 的启动参数、环境变量和输出解析规则。
- continuity split：区分 platform session / router session 与 provider-native session。
- failure normalization：把不同 runtime 的 stderr / event / exit code 映射成统一错误码。
- capability gating：在真正启动前，先按 runtime 已安装 app、工具诊断和权限状态裁剪能力。
- fallback policy：resume 失败、provider 失效或 runtime 切换时，改为 cold rebuild 并显式补回平台上下文。

## 关键边界

- [[harness/runtime/agent-loop/index|Agent Loop]] 处理单个 runtime 内的一次 run 如何推进；Runtime Brokerage 处理 run 该交给哪个 runtime，以及跨 runtime 时如何保持连续性。
- [[harness/tooling/tool-availability/index|Tool Availability]] 决定模型看到哪些 tools；Runtime Brokerage 决定某个 runtime 物理上有哪些真实可执行能力。
- [[harness/lifecycle/pending-resume/index|Pending & Resume]] 关注挂起与续跑语义；Runtime Brokerage 关注 provider-native session 丢失时如何继续。
- [[harness/context/context-injection/index|Context Injection]] 负责把 router transcript、fallback reason、runtime app inventory 等平台上下文补回模型。

## 目录

- [[harness/runtime/runtime-brokerage/implementations/agentspace|AgentSpace 实现]]

## 后续收集

如果外部项目显式实现了 runtime router、provider adapter、cross-runtime resume/fallback、runtime health normalization 或 capability inventory，可以归到本目录。
