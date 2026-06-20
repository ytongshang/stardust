---
title: Skill Versioning
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - skills
---

# Skill Versioning

Skill Versioning 处理 skill 的版本、更新、兼容性和缓存失效。长期来看，skill 像依赖包一样需要版本管理。

## 它解决什么问题

Skill 会变：instructions 更新、script 修复、model adapter 改动、资源路径变化。如果没有 Skill Versioning，同一个任务在不同时间可能加载到不同能力，replay/eval 也很难解释差异。

它需要控制：

- run 使用的是哪个 skill version。
- skill 更新是否立即生效，还是按 project/run pin 住。
- 新版本和旧 harness/model/tool 是否兼容。
- cache 什么时候失效，materialized files 什么时候重建。
- 出问题时能否回滚到旧版本。

## 常见做法

常见策略包括：

- semantic version 或 content hash。
- lockfile / capability snapshot 记录 run 使用版本。
- compatibility metadata：requires model、requires tool、requires runtime feature。
- cache busting：manifest 变化时重新 materialize。
- rollout：灰度启用、手动 pin、按 workspace 或 user opt-in。

## 边界

- [[harness/skills/skill-materialization/index|Skill Materialization]] 处理文件落地；Skill Versioning 决定落地哪个版本。
- [[harness/observability/replay/index|Replay]] 需要版本信息才能复现旧 run。
- [[harness/tooling/extension-registry/index|Extension Registry]] 可以管理 skill source 和 update state。

## 目录

- [[harness/skills/skill-versioning/implementations/uxarts-agent|uxarts-agent 实现]]
