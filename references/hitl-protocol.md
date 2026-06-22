# HITL JSON Block Protocol — 规范 v1.0

## 概述

HITL（Human-in-the-Loop）JSON 块是一种结构化人机交互协议。agent 在需要人类介入时，输出一个 ````hitl` 代码块，包含可解析的 JSON 数据，定义：

- 为什么需要人类介入（summary）
- 需要人类做什么决策（decisions）
- 决策的选项或输入字段（options / fields）

---

## 格式定义

### 顶层结构

```text
{
  "version": "<string>",
  "checkpoint": {
    "id": "<string>",
    "name": "<string>",
    "phase": "<string>",
    "summary": "<string>",
    "action": "<string>",
    "decisions": [<Decision>, ...]
  }
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `version` | string | 是 | 协议版本号，当前为 `"1.0"` |
| `checkpoint.id` | string | 是 | 全局唯一标识，格式 `cp-{n}`，如 `cp-1` |
| `checkpoint.name` | string | 是 | 人类可读的名称，如 `"Plan 确认"` |
| `checkpoint.phase` | string | 是 | 所属阶段名称，如 `"Phase 2"` 或 `"规划完成"` |
| `checkpoint.summary` | string | 是 | 一句话说明当前状态和需要人类决策的原因 |
| `checkpoint.action` | string | 是 | 动作类型：`"wait"`（必须等用户响应）或 `"notify"`（仅通知，不等回复） |
| `checkpoint.decisions` | array | 是 | 决策项数组，至少 1 项 |

### Decision 对象

```text
{
  "id": "<string>",
  "type": "<string>",
  "question": "<string>",
  "options": [<Option>, ...],
  "fields": [<Field>, ...],
  "default": "<string>"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 决策项标识，格式 `d-{n}` |
| `type` | string | 是 | 交互类型：`choice` / `confirm` / `review` / `input` |
| `question` | string | 是 | 向用户展示的问题，必须清晰可用"是/否"或选项回答 |
| `options` | array | 按 type | `choice` 类型必填；`confirm` 可选；其他类型忽略 |
| `fields` | array | 按 type | `input` 类型必填；其他类型忽略 |
| `default` | string | 否 | 默认选项的 `value`，agent 推荐但不会默认执行 |

### Option 对象

```text
{
  "value": "<string>",
  "label": "<string>",
  "desc": "<string>"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `value` | string | 是 | 选项的内部值，agent 用于路由决策 |
| `label` | string | 是 | 展示给用户的可读标签，建议含表情符号（✅ ✏️ ❌ 等） |
| `desc` | string | 否 | 选项的简短说明（选此项会怎样） |

### Field 对象

```text
{
  "name": "<string>",
  "type": "string | choice | number",
  "label": "<string>",
  "required": true,
  "default": "<any>",
  "options": ["<string>", ...]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 字段名，用于 agent 读取用户输入 |
| `type` | string | 是 | 字段类型：`string` / `choice` / `number` |
| `label` | string | 是 | 字段说明，展示给用户 |
| `required` | boolean | 否 | 是否必填（默认 false） |
| `default` | any | 否 | 默认值 |
| `options` | array | type=choice 时 | 可选项列表 |

---

## 类型说明

### choice — 多选一

用户从预定义选项中选择一个。

**使用条件：** 用户的选择空间是已知的、有限的（≤ 5 个选项）。

### confirm — 确认/拒绝

用户回答"是/否"。

**使用条件：** 只需要二元决策（是否覆盖？是否交付？）。

### review — 审阅产物

用户审阅一个 artifact 并提供反馈。

**使用条件：** 生成了需要人工判断质量的产物。

**特殊行为：** 用户响应后，agent 应追问具体修改意见，而非仅接受"好/不好"。

### input — 信息收集

用户提供结构化信息（文本、路径、偏好等）。

**使用条件：** 需要收集 agent 无法自行获取的信息。

---

## 嵌入规则（在 SKILL.md 中的写法）

生成的 skill 中，每个 checkpoint 的写法格式为：

```text
#### ★ Checkpoint N — 名称

自然语言说明，让用户知道当前状态和需要做什么。

**决策收集铁律：** 以下决策项每项独立列出。Agent **可以推荐**（用 `default` 字段标记），但**不能静默替用户选择**。

```text
{
  "version": "1.0",
  "checkpoint": {
    ...
  }
}
```
```

### 转义规则

在 SKILL.md 文件中展示 ````hitl` 块时，外层围栏用 ```` ````（4 个反引号），内部代码块示例用 ``` ````（3 个反引号）：

``````text
```text
{...}
```
``````

如果 SKILL.md 中需要展示"如何在 SKILL.md 中嵌入示例"（嵌套两层），外层用 ```` ```` ```` ````（5 个反引号）。

---

## 校验规则

| # | 规则 | 严重性 |
|---|------|--------|
| 1 | JSON 可解析（`json.loads` 无异常） | 🔴 必须 |
| 2 | `version` 存在且为 `"1.0"` | 🔴 必须 |
| 3 | `checkpoint` 对象存在 | 🔴 必须 |
| 4 | `checkpoint.id` 为非空字符串 | 🔴 必须 |
| 5 | `checkpoint.name` 为非空字符串 | 🔴 必须 |
| 6 | `checkpoint.summary` 不为空 | 🔴 必须 |
| 7 | `action` 为 `wait` 或 `notify` | 🔴 必须 |
| 8 | `decisions` 为非空数组 | 🔴 必须 |
| 9 | 每个 decision 有 `id`、`type`、`question` | 🔴 必须 |
| 10 | `type` 为 `choice`/`confirm`/`review`/`input` | 🔴 必须 |
| 11 | `type=choice` 时 `options` 为非空数组且 ≤ 5 项 | 🟡 建议 |
| 12 | 每个 option 有 `value`、`label` | 🔴 必须 |
| 13 | 每个 option 有 `desc`（建议有） | 🟢 可选 |
| 14 | `type=input` 时 `fields` 为非空数组且 ≤ 5 项 | 🟡 建议 |

---

## 变更记录

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0 | 初始 | 协议定义 |
