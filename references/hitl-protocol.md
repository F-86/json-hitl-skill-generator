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
| `type` | string | 是 | 交互类型：`choice` / `confirm` / `review` / `input` / `combobox` / `number_range` / `datetime_range` |
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
  "type": "string | choice | number | number_range | datetime_range",
  "label": "<string>",
  "required": true,
  "default": "<any>",
  "options": ["<string>", ...]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 字段名，用于 agent 读取用户输入 |
| `type` | string | 是 | 字段类型：`string` / `choice` / `number` / `number_range` / `datetime_range` |
| `label` | string | 是 | 字段说明，展示给用户 |
| `required` | boolean | 否 | 是否必填（默认 false） |
| `default` | any | 否 | 默认值 |
| `options` | array | type=choice 时 | 可选项列表 |

### 范围字段类型（number_range / datetime_range）

前端将范围字段渲染为两个输入框（下界 / 上界），两端均可选填。agent 收到用户输入后，将非空端序列化为 `gte`/`lte` 算符。

| 类型 | 前端渲染 | 序列化格式 |
|------|---------|-----------|
| `number_range` | 两个数字输入（最小值 / 最大值） | `{"gte": <number>, "lte": <number>}` |
| `datetime_range` | 两个日期时间选择器（开始 / 结束） | `{"gte": "<ISO8601>", "lte": "<ISO8601>"}` |

- 只填一端时，filters 中只出现对应算符（如只填最小值 → `{"gte": x}`）
- 两端均未填 → 该字段不进入 filters
- 可用算符：`gte`（≥）、`lte`（≤）；若需精确等于，使用 `number` 类型配合值列表

**filters 混合示例：**

```json
{
  "filters": {
    "user_id": ["u-1", "u-2"],
    "age": {"gte": 18, "lte": 30},
    "created_at": {"gte": "2024-01-01T00:00:00Z", "lte": "2024-12-31T23:59:59Z"}
  }
}
```

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

**使用条件：** 仅限 Create / Update 场景，用户需要填写新数据（如商品名称、价格等）。**禁止**用于查询条件收集——查询字段的可选值应由前端通过 `combobox` 的 `options_from` 拉取。

### combobox — 前端拉取选项后用户选择

LLM 输出包含 `options_from` apicall 的 decision，前端执行该 apicall 获取选项列表后渲染 combobox，用户从中选择（支持多选）。

**使用条件：** 查询条件中，可选值存在于系统数据中（如分类、状态、用户等），需要由前端去拉取再展示，**不**由 LLM 直接枚举。

**decision 结构：**

```text
{
  "id": "d-n",
  "type": "combobox",
  "question": "<向用户展示的问题>",
  "field": "<filters 中的 key>",
  "label": "<字段名称>",
  "multiple": true,
  "options_from": {
    "method": "GET",
    "endpoint": "<拉取选项的 API 路径>",
    "label_field": "<选项展示字段名>",
    "value_field": "<选项值字段名>"
  }
}
```

**decision 专用字段：**

| 字段 | 必填 | 说明 |
|------|------|------|
| `field` | 是 | 对应 filters 的 key |
| `label` | 是 | 展示给用户的字段名称 |
| `multiple` | 否 | 是否允许多选，默认 `true` |
| `options_from.method` | 是 | 拉取选项的 HTTP 方法，通常为 `GET` |
| `options_from.endpoint` | 是 | 拉取选项的 API 路径 |
| `options_from.label_field` | 是 | 选项对象中用于展示的字段名 |
| `options_from.value_field` | 是 | 选项对象中用于提交的字段名 |

前端收到用户选择后，将所选值（单个或多个）作为数组放入 filters：`{"field": ["v1", "v2"]}`。

### number_range — 数值范围

用户填写单个数值字段的范围条件（最小值 / 最大值）。

**使用条件：** 需要对数值字段做大于/小于/区间筛选，如年龄、数量、金额。

**decision 专用字段：**

| 字段 | 必填 | 说明 |
|------|------|------|
| `field` | 是 | 对应资源的字段名，agent 用于构建 filters key |
| `label` | 是 | 展示给用户的字段名称 |
| `unit` | 否 | 数值单位（如"岁"、"件"、"元"），前端展示用 |

前端渲染为两个数字输入框（最小值 / 最大值）。序列化规则：只填最小值 → `{"gte": x}`；只填最大值 → `{"lte": y}`；两端都填 → `{"gte": x, "lte": y}`；两端均空 → 不进入 filters。

### datetime_range — 时间范围

用户填写单个时间字段的范围条件（开始时间 / 结束时间）。

**使用条件：** 需要对时间字段做范围筛选，如创建时间、修改时间、过期时间。

**decision 专用字段：**

| 字段 | 必填 | 说明 |
|------|------|------|
| `field` | 是 | 对应资源的字段名，agent 用于构建 filters key |
| `label` | 是 | 展示给用户的字段名称 |

前端渲染为两个日期时间选择器（开始 / 结束）。序列化为 ISO 8601：只填开始 → `{"gte": "...Z"}`；只填结束 → `{"lte": "...Z"}`；两端都填 → `{"gte": "...Z", "lte": "...Z"}`；两端均空 → 不进入 filters。

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
| 10 | `type` 为 `choice`/`confirm`/`review`/`input`/`combobox`/`number_range`/`datetime_range` | 🔴 必须 |
| 11 | `type=choice` 时 `options` 为非空数组且 ≤ 5 项 | 🟡 建议 |
| 12 | 每个 option 有 `value`、`label` | 🔴 必须 |
| 13 | 每个 option 有 `desc`（建议有） | 🟢 可选 |
| 14 | `type=input` 时 `fields` 为非空数组且 ≤ 5 项 | 🟡 建议 |
| 15 | `type=combobox` 时 `field`、`options_from.endpoint`、`options_from.label_field`、`options_from.value_field` 均存在 | 🔴 必须 |
| 16 | `type=number_range` 或 `type=datetime_range` 时 `field`、`label` 均存在 | 🔴 必须 |

---

## apicall Block

````apicall` 块与 ````hitl` 块平级，表示 agent **执行** API 调用的步骤（机器执行节点，不等待人类决策）。前端负责执行 apicall 并渲染结果；agent 不编造数据、不描述结果。

### 两种用法

**独立块**：参数齐全后直接输出，前端执行后渲染结果。

```text
```apicall
{"method": "GET", "endpoint": "/api/orders?status=pending"}
```
```

**内嵌在 hitl checkpoint 中**：将 `apicall` 字段放在 `checkpoint` 顶层，用户确认后前端直接执行。适用于 Delete 等需要最终人类确认再执行的操作。

```text
```hitl
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-delete",
    "name": "删除确认",
    "phase": "Phase 2",
    "summary": "即将永久删除对象 <id>",
    "action": "wait",
    "apicall": {"method": "DELETE", "endpoint": "/api/orders/<id>"},
    "decisions": [...]
  }
}
```
```

### 字段说明（独立块 / 内嵌 apicall 对象）

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `method` | string | 是 | HTTP 方法：`GET` / `POST` / `PUT` / `PATCH` / `DELETE` |
| `endpoint` | string | 是 | API 路径，GET 参数拼入 query string，如 `/api/orders?status=pending` |
| `body` | object | 否 | POST / PUT / PATCH 的请求体；GET 和 DELETE 不需要 |

> **只包含用户实际给出的参数**，未提及的字段不拼入 URL 或 body。

### 放置规则

| 操作 | apicall 形式 | 位置 | endpoint 格式 |
|------|------------|------|--------------|
| Read/Query | 独立块 | 参数确认 `hitl` 之后 | `POST /api/<resource>/query`，body 为 `{"filters": {<field>: [<values>]}}` |
| Create | 独立块 | 最终确认 `hitl` 之后（或参数齐全时直接输出） | `POST /api/<resource>`，body 为资源字段 |
| Update | 独立块 | 最终确认 `hitl` 之后 | `PUT /api/<resource>/<id>`，body 为要修改的字段 |
| Delete | 内嵌在最终确认 `hitl` 中 | `checkpoint.apicall` 字段 | `DELETE /api/<resource>/<id>` |

> **铁律：** CUD 操作的写操作（POST/PUT/PATCH/DELETE）必须在最终确认 `hitl` 之后（或内嵌其中）才能执行，不能在用户确认之前发出写请求。

### apicall 校验规则

| # | 规则 | 严重性 |
|---|------|--------|
| 1 | JSON 可解析 | 🔴 必须 |
| 2 | `method` 为合法 HTTP 方法 | 🔴 必须 |
| 3 | `endpoint` 为非空字符串 | 🔴 必须 |
| 4 | GET/DELETE 块不含 `body` 字段 | 🟡 建议 |
| 5 | CUD 操作：apicall 前有对应的最终确认 `hitl` 块 | 🔴 必须 |

---

## 变更记录

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0 | 初始 | 协议定义 |
| 1.1 | — | 新增 `apicall` 块类型（独立块 + 内嵌形式） |
