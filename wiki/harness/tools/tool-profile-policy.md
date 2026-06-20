---
title: Tool Profile & Policy
type: concept
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/prototype_tools_builder.py
  - uxarts-agent/uxarts/web/modules/getui/agent/biz/sub_designer/sub_agent_tools_builder.py
tags:
  - agent-harness
  - policy
  - tools
---

# Tool Profile & Policy

## 这个概念是什么

Tool Profile 决定“这一轮有哪些工具”。Tool Policy 决定“这些工具能做什么、哪些写入被允许、哪些能力需要被禁用”。

业务层可以说 stage1/stage4/consult/sub-agent；runtime 层应该只看到 profile 和 policy。

## uxarts-agent 现在做什么

prototype 主 agent：

- stage1 使用精简工具。
- consult 使用 readonly filesystem。
- remix 使用完整工具但去掉 present choices。
- stage2/stage3/stage4 按业务阶段裁剪工具。
- plan mode 只允许写 `internal/plans/plan.md`。
- architecture planning 未确认时只允许写 README。
- plugin enabled 后合并 remote manifest tools。
- skills materialize 到 `.superun/skills`。

sub-agent：

- 独立 `sub_agent_tools_builder`。
- 删除 main-only tools。
- 增加 `AskMainAgent`。
- stage4 才保留 Supabase data tools。
- plugin 状态查 main session。
- 防 background 递归。

## uxarts-agent 怎么做

工具构建流程：

```text
BusinessRoute(mode/stage/sub_agent_type)
  -> create_prototype_tools_func / sub_agent_tools_builder
  -> build_full_tools / build_simple_tools / get_sub_agent_tools
  -> dynamic remote plugin tools
  -> editor write allowlist / readonly policy
```

硬权限主要在 editor 层：

- plan mode 设置 write allowlist。
- architecture planning 设置 README-only。
- 每轮开始先 reset allowlist，避免上一轮泄漏。

## 不能丢的细节

- prompt 里的提醒只是软约束，写文件必须有硬拦截。
- sub-agent 删除 main-only 工具不是为了省 token，而是权限边界。
- remote plugin tools 可能和静态工具重名，需要 shadow check。
- Supabase tools 按 stage 和 agent type 双重裁剪。
- skill 不一定是工具，更多是 progressive disclosure 的文件包。

## 建议抽象

```python
@dataclass
class ToolProfile:
    base_tools: list[ToolRef]
    dynamic_sources: list[ToolSource]
    exclusions: list[ToolExclusion]
    policies: list[ToolPolicy]
```

先让现有 builder 产出 profile，再由 profile 生成工具，行为不变。

## Related Implementations

- 外部实现如果只是“根据 agent 配工具”，不算新。
- 值得记录的是统一 permission metadata、policy engine、tool capability discovery、tool profile eval。
