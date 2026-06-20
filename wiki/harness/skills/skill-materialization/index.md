---
title: Skill Materialization
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - skills
---

# Skill Materialization

Skill Materialization 是把 skill catalog 中的能力包落成 agent 可读资源，例如文件夹、`SKILL.md`、脚本、模板、示例或资产。

## 它解决什么问题

有些 skill 不是一段 prompt，而是一组资源：说明文件、脚本、模板、示例、图片、schema、helper 程序。Skill Materialization 让这些资源在运行时有稳定路径、权限和版本，而不是每次临时拼进上下文。

它需要回答：

- skill package 从哪里来：本地目录、plugin、remote registry、repo、artifact store。
- materialized path 怎么命名，如何避免冲突。
- 脚本和 assets 是否可执行/可读取，权限如何设置。
- 更新 skill 时如何清理旧资源和缓存。
- materialization 失败时是否降级为只读说明或禁用 skill。

## 常见做法

常见实现包括：

- install/cache：把 skill package 放到稳定 cache directory。
- manifest：记录 source、version、files、checksums、capabilities。
- activation：把 materialized path 暴露给 Skill Loading 或 tools。
- cleanup：过期版本、损坏 cache、权限不匹配时重新 materialize。
- sandboxing：限制 skill script 的执行环境和资源访问。

## 边界

- [[harness/skills/skill-loading/index|Skill Loading]] 读取 materialized resources；Materialization 负责资源落地。
- [[harness/skills/skill-versioning/index|Skill Versioning]] 决定版本和更新策略。
- [[harness/workspace/artifacts/index|Artifacts]] 是 agent 产出；skill materialization 是 agent 使用的能力资源。

## 目录

- [[harness/skills/skill-materialization/implementations/uxarts-agent|uxarts-agent 实现]]
