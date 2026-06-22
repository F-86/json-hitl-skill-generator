# Generated Skill Template

这是一个生成 SKILL.md 的模板。基于 Harness 6 维度架构 + HITL JSON 块协议。

使用 `json-hitl-skill-generator` 生成时，按以下骨架生成，按实际需求增删章节。

---

```markdown
---
name: <skill-name>
description: >-
  <description>
version: 1.0.0
author: <user>
license: MIT
metadata:
  hermes:
    tags: [<tag1>, <tag2>]
    related_skills: [<related>]
---

# <Skill Title>

## Overview

简短描述。

## 边界（先判断要不要进入本 Skill）

- **进入条件**：...
- **不处理**：...

## Harness 视角

| 维度 | 设计 | 实现手段 |
|------|------|---------|
| **CONTEXT** | 渐进加载 | ... |
| **TOOLS** | 输入输出 | ... |
| **ORCHESTRATION** | Phase 管道 | ... |
| **MEMORY** | 文件系统 | ... |
| **EVALUATION** | 分层质检 | ... |
| **RECOVERY** | 约束恢复 | ... |

## HITL 交互协议

本 skill 使用 ````hitl` JSON 代码块作为标准人机交互协议。每个需要人类介入的节点，agent 输出以下格式的 block：

```text
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-n",
    "name": "名称",
    "phase": "阶段",
    "summary": "当前状态描述",
    "action": "wait",
    "decisions": [
      {"id": "d-1", "type": "choice", "question": "问题"}
    ]
  }
}
```

协议完整规范见 `references/hitl-protocol.md`。

## 工作流总览

```
Phase 0  Intake         判断模式 + 收集输入
   ▼
Phase 1  Context        收集上下文
   ▼
Phase 2  Plan           输出规划
         ★ Checkpoint 1 — 必须停
   ▼
Phase 3  Build          生成产物
   ▼
Phase 4  Review         质检
         ★ Checkpoint 2 — 必须停
   ▼
Phase 5  Revise         修复
   ▼
Phase 6  Deliver        交付
```

## Phase 0 — Intake

### 判断模式

...

## Phase 1 — Context

渐进加载：

| Phase | 必读 | 按需查 |
|-------|------|--------|
| Phase 0 | 本 SKILL.md | — |
| Phase 1 | ... | ... |

## Phase 2 — Plan

...

#### ★ Checkpoint 1 — 名称

**决策收集铁律：** 以下决策项每项独立列出。Agent **可以推荐**（用 `default` 字段标记），但**不能静默替用户选择**。

```hitl
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-1",
    "name": "名称",
    "phase": "Phase 2",
    "summary": "已完成规划，等待用户确认",
    "action": "wait",
    "decisions": [
      {
        "id": "d-1",
        "type": "choice",
        "question": "是否同意该方案？",
        "options": [
          {"value": "approve", "label": "✅ 批准", "desc": "进入下一阶段"},
          {"value": "modify", "label": "✏️ 修改", "desc": "提出调整"},
          {"value": "reject", "label": "❌ 重做", "desc": "重新规划"}
        ]
      }
    ]
  }
}
```

## Phase 3 — Build

...

## Phase 4 — Review

...

#### ★ Checkpoint 2 — 名称

...

## Phase 5 — Revise

...

## Phase 6 — Deliver

#### ★ Checkpoint 3 — 最终确认

...

---

## Common Pitfalls

1. ...

## Verification Checklist

- [ ] ...
```
