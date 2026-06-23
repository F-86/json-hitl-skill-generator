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
| 全部参数明确且齐全 | ❌ | — | 不触发参数收集 HITL，进入 ★ Checkpoint 确认 |
| 用户已明确说"直接执行"/"不用确认" | ❌ | — | 跳过 HITL，直接进行 |

> **铁律 1：不在上表中的情况，一律不触发参数收集阶段的 HITL。禁止在参数收集过程中自由发挥添加额外确认点。各 Phase 末尾标有 ★ 的 Checkpoint 是流程必经点，不在此表约束范围内——无论参数是否齐全都必须触发。**
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

**[查询操作] 禁止行为：**
- **禁止**用自然语言询问"你想查全部还是按条件筛选？"、"需要按什么条件查？"等问题
- 收到查询意图后，**直接进入 Phase 1，输出 hitl input 块**，把所有可筛选字段一次性展示给用户
- 用户留空的字段 = 不按该字段筛选；所有字段留空 = 查询全部

## Phase 1 — <操作名称>（如"查询条件收集"、"创建参数收集"）

### 参数清单

| 参数 | 必填 | 类型 | 默认值 | 说明 |
|------|------|------|--------|------|
| `<param_1>` | 是/否 | string/number/choice | `<默认值或 —>` | `<说明>` |
| `<param_2>` | 是/否 | string/number/choice | `<默认值或 —>` | `<说明>` |

### 执行逻辑

**[查询操作]** 查询条件通过 `hitl` 块收集，所有字段均非必填：

1. **直接输出** `hitl` 块，将所有可筛选字段一次性展示给用户，不先询问意图
   - 可枚举字段（分类、状态等）→ `combobox`（前端拉取选项渲染 combobox）
   - 数值区间字段（年龄、数量等）→ `number_range`
   - 时间区间字段（创建时间等）→ `datetime_range`
   - **禁止**在查询场景使用 `input` 类型
2. 前端执行 `combobox.options_from` 拉取选项，用户操作完成后将结果构建为 filters 对象
3. 输出 `apicall`

**[增/改/删操作]** 从用户输入中提取参数：

1. 从用户输入中提取参数
2. 检查必填参数是否齐全
   - 缺失但有默认值 → 使用默认值，不触发 HITL
   - 缺失且无默认值 → 触发 HITL（见 ## HITL 触发条件）
   - 歧义 → 触发 HITL（见 ## HITL 触发条件）
3. 参数齐全 → 展示参数摘要，进入下一阶段

### 禁止行为

- **禁止**在参数齐全时触发额外 HITL
- **禁止**猜测歧义参数的含义
- **禁止**跳过必填参数直接执行

#### ★ Checkpoint 1 — 参数确认

**[查询操作]** 直接输出查询条件块（combobox + 范围字段混合）：

**决策收集铁律：** 以下决策项每项独立列出。Agent **可以推荐**（用 `default` 字段标记），但**不能静默替用户选择**。

```hitl
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-1",
    "name": "查询条件",
    "phase": "Phase 1",
    "summary": "请选择筛选条件，留空的字段不参与筛选",
    "action": "wait",
    "decisions": [
      {
        "id": "d-1",
        "type": "combobox",
        "question": "请选择<字段1>",
        "field": "<field1>",
        "label": "<字段1>",
        "multiple": true,
        "options_from": {
          "method": "GET",
          "endpoint": "/api/<options-endpoint>",
          "label_field": "<name>",
          "value_field": "<id>"
        }
      },
      {
        "id": "d-2",
        "type": "number_range",
        "question": "请填写<数字字段>范围",
        "field": "<num_field>",
        "label": "<数字字段>",
        "unit": "<单位>"
      },
      {
        "id": "d-3",
        "type": "datetime_range",
        "question": "请填写<时间字段>范围",
        "field": "<time_field>",
        "label": "<时间字段>"
      }
    ]
  }
}
```

> 根据资源实际字段，只保留适用的 decision 条目；不适用的类型直接删除。

**[增/改/删操作]** 标准参数确认块：

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

**[增/改]** 最终确认后输出独立 apicall：

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

```apicall
{"method": "POST", "endpoint": "/api/<resource>", "body": {"field1": "{{param1}}", "field2": "{{param2}}"}}
```

**[删]** apicall 内嵌在最终确认 hitl 中，用户确认后前端直接执行：

```hitl
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-delete",
    "name": "删除确认",
    "phase": "Phase 2",
    "summary": "即将永久删除 <对象描述>，不可恢复",
    "action": "wait",
    "apicall": {"method": "DELETE", "endpoint": "/api/<resource>/{{id}}"},
    "decisions": [
      {
        "id": "d-1",
        "type": "choice",
        "question": "确认删除？",
        "options": [
          {"value": "confirm", "label": "⚠️ 确认删除", "desc": "永久删除"},
          {"value": "cancel", "label": "❌ 取消", "desc": "不执行"}
        ]
      }
    ]
  }
}
```

**[查]** 无最终确认 CP；参数确认 hitl 之后直接输出 apicall：

```apicall
{"method": "POST", "endpoint": "/api/<resource>/query", "body": {"filters": {"<field1>": ["<value1>", "<value2>"], "<num_field>": {"gte": 0, "lte": 100}, "<time_field>": {"gte": "2024-01-01T00:00:00Z", "lte": "2024-12-31T23:59:59Z"}}}}
```

> 序列化规则：`string` 字段 → 值数组；`number_range` → `{"gte": x, "lte": y}`（只填一端时只出现对应算符）；`datetime_range` → `{"gte": "ISO8601", "lte": "ISO8601"}`。`filters` 只包含用户实际填写的非空字段；若用户未填写任何字段，`filters` 为 `{}`。

### 结果展示

<操作完成后的结果展示方式：前端执行 apicall 并渲染，agent 不描述结果>

提供后续操作选项。

---

## Common Pitfalls

1. 【查询】收到查询意图后先用自然语言问"你想按什么条件查"——禁止。应直接输出 hitl input 块，把所有可筛选字段一次性给用户。
2. ...

## Verification Checklist

- [ ] 操作类型识别正确
- [ ] 参数清单完整（所有字段已列出）
- [ ] HITL 触发条件表已填充，每种参数归类为确定性/可能缺失/歧义
- [ ] 禁止行为已写入
- [ ] 所有 ````hitl` 块 JSON 语法合法
- [ ] 决策收集铁律已写入
- [ ] （增/改）有最终确认 hitl + 独立 apicall
- [ ] （删）有最终确认 hitl，apicall 内嵌在 checkpoint 中，无全局 default
- [ ] （查）无最终确认；参数确认 hitl 后紧跟独立 apicall
- [ ] apicall 块只包含用户实际提供的参数，无未提及字段
```
