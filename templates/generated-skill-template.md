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
Phase 1  <操作阶段>     参数收集与确认
         ★ Checkpoint 1 — 参数确认
   ▼
Phase 2  <执行阶段>     执行操作 + 展示结果
         ★ Checkpoint 2 — 最终确认（仅增/改/删）
```

## HITL 触发条件

本 skill 的 HITL 触发是**精确约束的**。只在以下明确条件下才允许输出 ````hitl` 块。

### 参数触发矩阵

| 条件 | 触发 HITL | 块类型 | 说明 |
|------|----------|--------|------|
| `<参数名>` 缺失且必填 | ✅ | `input` | 收集缺少的必填参数 |
| `<参数名>` 存在歧义（如"最近的"、"大概"） | ✅ | `choice` | 让用户明确选择 |
| 全部参数明确且齐全 | ❌ | — | 直接执行，不打断 |
| 用户已明确说"直接执行"/"不用确认" | ❌ | — | 跳过 HITL，直接进行 |

> **铁律 1：不在上表中的情况，一律不触发 HITL。禁止 LLM 自由发挥添加额外确认点。**
>
> **铁律 2：触发 HITL 时必须附带自然语言说明，让用户知道为什么需要介入、当前是什么状态。**
>
> **铁律 3：每个 ````hitl` 块的 `decisions` 至少 1 项，每项独立。Agent 可推荐默认值（用 `default` 字段），但不能静默替用户选择。**

### [仅增/改/删] 最终确认触发条件

| 条件 | 触发 HITL | 块类型 | 说明 |
|------|----------|--------|------|
| 参数已齐全、操作即将执行 | ✅ | `confirm`（增/改） / `choice`（删） | 展示将要执行的操作预览，等待用户最终确认 |

> **Delete 铁律：删除最终确认的 `choice` 块，不得设置全局 `default` 字段。用户可设置 option 级别的 `default` 将"取消"设为默认。**

### [仅查] 不触发最终确认

查询操作在参数确认后直接执行并展示结果，无最终确认环节。

## Phase 0 — Intake

### 判断操作意图

从用户描述中判断操作类型，提取已提供的参数。

## Phase 1 — <操作名称>（如"查询参数确认"、"创建参数收集"）

### 参数清单

| 参数 | 必填 | 类型 | 说明 |
|------|------|------|------|
| `<param_1>` | 是/否 | string/number/choice | `<说明>` |
| `<param_2>` | 是/否 | string/number/choice | `<说明>` |

### 执行逻辑

1. 从用户输入中提取参数
2. 检查必填参数是否齐全
   - 缺失 → 触发 HITL（见 ## HITL 触发条件）
   - 歧义 → 触发 HITL（见 ## HITL 触发条件）
3. 参数齐全 → 展示参数摘要，进入下一阶段

### 禁止行为

- **禁止**在参数齐全时触发额外 HITL
- **禁止**猜测歧义参数的含义
- **禁止**跳过必填参数直接执行

#### ★ Checkpoint 1 — 参数确认

**决策收集铁律：** 以下决策项每项独立列出。Agent **可以推荐**（用 `default` 字段标记），但**不能静默替用户选择**。

```hitl
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-1",
    "name": "参数确认",
    "phase": "Phase 1",
    "summary": "已完成参数收集",
    "action": "wait",
    "decisions": [
      {
        "id": "d-1",
        "type": "choice",
        "question": "是否确认以上参数？",
        "options": [
          {"value": "approve", "label": "✅ 确认", "desc": "进入执行阶段"},
          {"value": "modify", "label": "✏️ 修改", "desc": "调整参数"},
          {"value": "cancel", "label": "❌ 取消", "desc": "放弃操作"}
        ]
      }
    ]
  }
}
```

## Phase 2 — <执行阶段名称>

### 执行

<操作说明：创建/查询/更新/删除的具体行为>

#### ★ Checkpoint 2 — <操作名称确认>（仅增/改/删）

```hitl
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-2",
    "name": "操作确认",
    "phase": "Phase 2",
    "summary": "请确认执行操作",
    "action": "wait",
    "decisions": [
      {"id": "d-1", "type": "confirm", "question": "确认执行？"}
    ]
  }
}
```

### 结果展示

<操作完成后的结果展示方式>

提供后续操作选项。

---

## Common Pitfalls

1. ...
2. ...

## Verification Checklist

- [ ] 操作类型识别正确
- [ ] 参数清单完整（所有字段已列出）
- [ ] HITL 触发条件表已填充，每种参数归类为确定性/可能缺失/歧义
- [ ] 禁止行为已写入
- [ ] 所有 ````hitl` 块 JSON 语法合法
- [ ] 决策收集铁律已写入
- [ ] （增/改/删）有最终确认
- [ ] （查）无最终确认
- [ ] （删）最终确认无全局 default
```
