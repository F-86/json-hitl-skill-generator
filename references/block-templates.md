# HITL JSON Block Templates

常见 ````hitl` JSON 块模板，供生成 skill 时引用。

## Choice（多选一）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-n",
    "name": "标题",
    "phase": "阶段名",
    "summary": "总结描述",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "choice",
      "question": "问题",
      "options": [
        {"value": "a", "label": "✅ 选项A", "desc": "说明"},
        {"value": "b", "label": "✏️ 选项B", "desc": "说明"}
      ],
      "default": "a"
    }]
  }
}
```

**用途：** 多选一决策。适用于范围确认、参数选择、路线决策。

**选项限制：** options ≤ 5 个，太多用户选不动。

---

## Confirm（确认）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-n",
    "name": "确认",
    "phase": "阶段名",
    "summary": "需要确认的事项",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "confirm",
      "question": "是否确认？"
    }]
  }
}
```

**用途：** 简单确认/拒绝。适用于是否覆盖文件、是否执行危险操作。

**注意：** 问题必须能用"是/否"回答。

---

## Review（审阅）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-n",
    "name": "审阅",
    "phase": "阶段名",
    "summary": "请审阅以下产物",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "review",
      "question": "请审阅产物并提出修改意见",
      "artifact_path": ".json-hitl-gen/draft.md"
    }]
  }
}
```

**用途：** 审阅产物质量。必须附带 `artifact_path`。

---

## Input（信息收集）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-n",
    "name": "信息收集",
    "phase": "阶段名",
    "summary": "需要用户提供以下信息",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "input",
      "question": "请提供以下信息：",
      "fields": [
        {"name": "param_1", "type": "string", "label": "参数1", "required": true},
        {"name": "param_2", "type": "choice", "label": "参数2", "options": ["A", "B", "C"]}
      ]
    }]
  }
}
```

**用途：** 收集用户提供的结构化信息。**仅限 Create / Update 场景**（用户填写新数据）。查询条件中的字段选择禁止使用 `input`——枚举字段用 `combobox`，自由文本字段（如 id、名称）用 `text_input`。

**限制：** fields ≤ 5 个。

---

## Combobox（前端拉取选项后用户选择）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-n",
    "name": "查询条件",
    "phase": "阶段名",
    "summary": "请选择筛选条件",
    "action": "wait",
    "decisions": [
      {
        "id": "d-1",
        "type": "combobox",
        "question": "请选择分类",
        "field": "category",
        "label": "分类",
        "multiple": true,
        "options_from": {
          "method": "GET",
          "endpoint": "/api/categories",
          "label_field": "name",
          "value_field": "id"
        }
      },
      {
        "id": "d-2",
        "type": "combobox",
        "question": "请选择状态",
        "field": "status",
        "label": "状态",
        "multiple": true,
        "options_from": {
          "method": "GET",
          "endpoint": "/api/statuses",
          "label_field": "label",
          "value_field": "value"
        }
      }
    ]
  }
}
```

**用途：** 查询条件中可枚举的字段。LLM 输出该块后，前端执行 `options_from` apicall 拉取选项列表并渲染 combobox，用户选择后前端将结果写入 filters。

**规则：**
- 一个 checkpoint 可以包含多个 `combobox` decision（每个字段一项）
- 可与 `number_range` / `datetime_range` decision 混排在同一 checkpoint 中
- `multiple: true` 时前端渲染多选 combobox，用户结果序列化为数组；`multiple: false` 时单选
- 用户未选择的字段不进入 filters

---

## Text Input（单行文本输入）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-n",
    "name": "查询条件",
    "phase": "阶段名",
    "summary": "请填写筛选条件",
    "action": "wait",
    "apicall": {"method": "POST", "endpoint": "/api/<resource>/query", "body": {"filters": {}}},
    "decisions": [
      {
        "id": "d-1",
        "type": "text_input",
        "field": "id",
        "label": "ID",
        "placeholder": "多个ID用逗号分隔（中英文逗号均可），如 1, 3, 5"
      },
      {
        "id": "d-2",
        "type": "text_input",
        "field": "name",
        "label": "名称",
        "placeholder": "模糊匹配，多个名称用逗号分隔（中英文逗号均可）"
      }
    ]
  }
}
```

**用途：** 查询条件中需要用户手动输入的自由文本字段（如 id 列表、名称关键词）。适用于字段值不由后端枚举、也不是数值/时间范围的场景。

**规则：**
- `placeholder` 应说明多值分隔符（推荐"中英文逗号均可"）
- LLM 从用户输入提取的值用 `default` 字段预填
- 前端负责将逗号分隔的字符串解析为数组（id → number[]，name → string[] 或单个 string）
- checkpoint 应内嵌 `apicall` 模板（`filters: {}`），前端提交时填充

---

## Number Range（数值范围）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-n",
    "name": "数值范围",
    "phase": "阶段名",
    "summary": "请填写数值范围条件",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "number_range",
      "question": "请填写范围（两端均可选填，留空表示不限）",
      "field": "age",
      "label": "年龄",
      "unit": "岁"
    }]
  }
}
```

**用途：** 收集单个数值字段的范围条件（大于/小于/区间）。适用于年龄、数量、金额、评分等。

**字段说明：**

| 字段 | 必填 | 说明 |
|------|------|------|
| `field` | 是 | 对应资源的字段名，agent 用于构建 filters key |
| `label` | 是 | 展示给用户的字段名称 |
| `unit` | 否 | 数值单位，前端展示用（如"岁"、"件"、"元"） |

前端渲染为两个数字输入框（最小值 / 最大值）。agent 收到后序列化：只填最小值 → `{"gte": x}`，只填最大值 → `{"lte": y}`，两端都填 → `{"gte": x, "lte": y}`，两端均空 → 该字段不进入 filters。

---

## Datetime Range（时间范围）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-n",
    "name": "时间范围",
    "phase": "阶段名",
    "summary": "请填写时间范围条件",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "datetime_range",
      "question": "请填写时间范围（两端均可选填，留空表示不限）",
      "field": "created_at",
      "label": "创建时间"
    }]
  }
}
```

**用途：** 收集单个时间字段的范围条件。适用于创建时间、修改时间、过期时间等。

**字段说明：**

| 字段 | 必填 | 说明 |
|------|------|------|
| `field` | 是 | 对应资源的字段名，agent 用于构建 filters key |
| `label` | 是 | 展示给用户的字段名称 |

前端渲染为两个日期时间选择器（开始 / 结束）。agent 收到后序列化为 ISO 8601：只填开始 → `{"gte": "...Z"}`，只填结束 → `{"lte": "...Z"}`，两端都填 → `{"gte": "...Z", "lte": "...Z"}`，两端均空 → 该字段不进入 filters。

---

### Read：条件收集（无最终确认，apicall 内嵌）

查询操作不需要最终确认。CP-1a 的 checkpoint 内嵌 `apicall` 模板（`filters: {}`），用户提交表单后前端直接填充 filters 并执行，不再经过 LLM。

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-1a",
    "name": "查询条件收集",
    "phase": "Phase 1",
    "summary": "请选择筛选条件，不需要的直接跳过，所有条件均为可选",
    "action": "wait",
    "apicall": {"method": "POST", "endpoint": "/api/<resource>/query", "body": {"filters": {}}},
    "decisions": [
      {
        "id": "d-1",
        "type": "text_input",
        "field": "id",
        "label": "ID",
        "placeholder": "多个用逗号分隔（中英文逗号均可）"
      },
      {
        "id": "d-2",
        "type": "combobox",
        "question": "请选择分类",
        "field": "category",
        "label": "分类",
        "multiple": true,
        "options_from": {
          "method": "GET",
          "endpoint": "/api/<resource>/categories",
          "label_field": "label",
          "value_field": "value"
        }
      },
      {
        "id": "d-3",
        "type": "number_range",
        "question": "请填写价格范围",
        "field": "price",
        "label": "价格",
        "unit": "元"
      }
    ]
  }
}
```

**关键规则：**
- checkpoint 必须内嵌 `apicall` 模板，`filters` 留空 `{}`
- 前端提交时把表单值注入 `apicall.body.filters` 并直接执行，不经过 LLM
- LLM 从用户输入提取的参数用 `default` 字段预填到对应 decision
- 一轮查询只输出一次 CP-1a，禁止分多轮询问

### Create/Update：最终确认

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-final",
    "name": "操作确认",
    "phase": "最终确认",
    "summary": "请确认以下操作将执行",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "confirm",
      "question": "确认执行该操作？"
    }]
  }
}
```

### Delete：最终确认（无 default）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-final",
    "name": "删除确认",
    "phase": "最终确认",
    "summary": "以下对象将被永久删除，不可恢复",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "choice",
      "question": "确认删除？",
      "options": [
        {"value": "confirm_delete", "label": "⚠️ 确认删除", "desc": "永久删除选定对象"},
        {"value": "cancel", "label": "❌ 取消", "desc": "不执行删除"}
      ],
      "default": "cancel"
    }]
  }
}
```

---

## CRUD apicall 模板

### Read：查询（独立块）

参数确认 `hitl` 之后输出。只拼入用户实际给出的参数。

```text
```apicall
{"method": "GET", "endpoint": "/api/<resource>?<param>=<value>"}
```
```

### Create：创建（独立块）

最终确认 `hitl` 之后输出，或参数齐全时直接输出。

```text
```apicall
{"method": "POST", "endpoint": "/api/<resource>", "body": {"field1": "value1", "field2": "value2"}}
```
```

### Update：更新（独立块）

最终确认 `hitl` 之后输出。body 中只包含用户实际要修改的字段。

```text
```apicall
{"method": "PUT", "endpoint": "/api/<resource>/<id>", "body": {"field": "value"}}
```
```

### Delete：删除（内嵌在最终确认 hitl 中）

将 `apicall` 字段嵌入 `checkpoint` 顶层，用户点确认后前端直接执行。不得含全局 `default`。

```text
```hitl
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-delete",
    "name": "删除确认",
    "phase": "Phase 2",
    "summary": "即将永久删除 <对象描述>，不可恢复",
    "action": "wait",
    "apicall": {"method": "DELETE", "endpoint": "/api/<resource>/<id>"},
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
```

**铁律：** Delete 最终确认块不得含全局 `default` 字段（option 级别可以标记 `"default": "cancel"`）。

---

## 多值组合筛选查询（Read/Query 专用）

适用于支持多字段、每字段多值组合条件的查询操作。LLM 直接输出包含所有可筛选字段的 `hitl` 块；前端对 `combobox` 字段执行 `options_from` 拉取选项，对范围字段渲染输入框，用户操作后构建 filters 对象。

**字段类型选择：**

| 字段特征 | 使用类型 | 说明 |
|---------|---------|------|
| 可枚举选项（分类、状态、用户等） | `combobox` | 前端拉取选项渲染 combobox |
| 数值区间（年龄、数量、金额等） | `number_range` | 前端渲染最小/最大值输入框 |
| 时间区间（创建时间、修改时间等） | `datetime_range` | 前端渲染日期时间选择器 |
| 自由文本（id 列表、名称关键词等） | `text_input` | 前端渲染单行文本输入框 |
| 禁止 | `input` | `input` 仅限 Create/Update 场景 |

### 查询条件 hitl 块（混合示例）

```json
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
        "question": "请选择分类",
        "field": "category",
        "label": "分类",
        "multiple": true,
        "options_from": {
          "method": "GET",
          "endpoint": "/api/categories",
          "label_field": "name",
          "value_field": "id"
        }
      },
      {
        "id": "d-2",
        "type": "number_range",
        "question": "请填写年龄范围",
        "field": "age",
        "label": "年龄",
        "unit": "岁"
      },
      {
        "id": "d-3",
        "type": "datetime_range",
        "question": "请填写创建时间范围",
        "field": "created_at",
        "label": "创建时间"
      }
    ]
  }
}
```

### apicall 块（内嵌在 hitl checkpoint 中）

查询操作的 `apicall` 不作为独立块输出，而是内嵌在 CP-1a 的 `checkpoint.apicall` 字段中，`filters` 留空 `{}`。前端在用户提交表单时填充 filters 并直接执行。

```json
{
  "checkpoint": {
    "apicall": {"method": "POST", "endpoint": "/api/<resource>/query", "body": {"filters": {}}},
    ...
  }
}
```

前端提交时填充后的 apicall 示例：

```json
{"method": "POST", "endpoint": "/api/<resource>/query", "body": {"filters": {"category": ["electronics", "clothing"], "age": {"gte": 18, "lte": 30}, "created_at": {"gte": "2024-01-01T00:00:00Z", "lte": "2024-12-31T23:59:59Z"}}}}
```

**序列化规则：**

| 字段类型 | filters 中的格式 |
|---------|----------------|
| `text_input`（id） | `[<number>, ...]`（逗号分隔字符串 → number[]） |
| `text_input`（name） | `"<string>"` 或 `["<string>", ...]`（单个用字符串，多个用数组） |
| `combobox` | `[<selected_values>]` |
| `number_range` | `{"gte": x, "lte": y}`（只填一端时只出现对应算符） |
| `datetime_range` | `{"gte": "<ISO8601>", "lte": "<ISO8601>"}` |

`filters` 只包含用户实际操作的非空字段；若用户未操作任何字段，`filters` 为 `{}`，代表查询全部。
