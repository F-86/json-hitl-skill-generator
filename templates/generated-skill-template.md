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
>
> **铁律 4：````hitl` 块必须用代码块包裹（```` ```hitl ````），禁止直接输出裸 JSON，否则前端无法渲染。**
>
> **铁律 5：[查询操作] CP-1a 一轮查询只出一次，禁止分多轮询问（先问价格、再问分类是错误的）。所有参数在同一个 CP-1a 中一次性收集。**

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

**[查询操作]**

| 参数 | 必填 | 类型 | 收集方式 | 默认值 | 说明 |
|------|------|------|----------|--------|------|
| `<param_1>` | 否 | number[] | `text_input` | `<—>` | `<说明>` |
| `<param_2>` | 否 | string/string[] | `text_input` | `<—>` | `<说明>` |
| `<param_3>` | 否 | string[] | `combobox` | `<—>` | `<说明>` |
| `<param_4>` | 否 | {gte,lte} | `number_range` | `<—>` | `<说明>` |
| `<param_5>` | 否 | {gte,lte} | `datetime_range` | `<—>` | `<说明>` |

> 查询参数全部可选，用户不填的字段不参与筛选。

**[增/改/删操作]**

| 参数 | 必填 | 类型 | 默认值 | 说明 |
|------|------|------|--------|------|
| `<param_1>` | 是/否 | string/number/choice | `<默认值或 —>` | `<说明>` |
| `<param_2>` | 是/否 | string/number/choice | `<默认值或 —>` | `<说明>` |

### 执行逻辑

**[查询操作]** 查询条件通过 `hitl` 块收集，所有字段均非必填：

1. **直接输出** `hitl` 块，将所有可筛选字段一次性展示给用户，不先询问意图
   - 自由文本字段（id、名称等）→ `text_input`
   - 可枚举字段（分类、状态等）→ `combobox`（前端拉取选项渲染 combobox）
   - 数值区间字段（年龄、数量等）→ `number_range`
   - 时间区间字段（创建时间等）→ `datetime_range`
   - **禁止**在查询场景使用 `input` 类型
2. **LLM 从用户输入提取的参数用 `default` 字段预填**到对应 decision，用户可修改/跳过
3. checkpoint 内嵌 `apicall` 模板（`filters: {}`），前端提交时注入 filters 直接执行，不再经过 LLM
4. **CP-1a 一轮查询只出一次**——禁止分多轮询问

**时间表达式解析规则：** 当用户输入包含相对时间表达式时，按以下规则转换为具体日期（`{YYYY}` 为系统注入的当前年份）：

| 表达式 | gte（起始） | lte（结束） |
|--------|------------|------------|
| "今年" / "本年" | `{YYYY}-01-01` | `{YYYY}-12-31` |
| "去年" | `{YYYY-1}-01-01` | `{YYYY-1}-12-31` |
| "本月" | `{YYYY}-{MM}-01` | 本月最后一天 |
| "YYYY年" | `YYYY-01-01` | `YYYY-12-31` |
| "YYYY年到今年" | `YYYY-01-01` | `{YYYY}-12-31` |

> **结束时间取所在区间的最后一天（年底、月末），而非今天。**

**[增/改/删操作]** 从用户输入中提取参数：

1. 从用户输入中提取参数
2. **已提取到的参数不要重复问**。只输出缺失字段的 ````hitl` `input` 块
   - **🔴 铁律：必须输出 ````hitl` 块，禁止用自然语言询问参数。** 即使只有一个字段缺失，也要输出 ````hitl` 块
   - 枚举字段（category、status 等）用 `fields[].type: "choice"` + `options` 数组，渲染为下拉框
   - 如果用户明确说了枚举值（如"服装"、"数码"），直接使用该值，不要质疑。只有模糊描述（"吃的"、"穿的"）才需要 HITL
3. 如果全部参数已齐全（如用户一句话说了所有字段），跳过 CP-1，直接进入参数确认
4. 用户填写后，展示参数摘要，进入下一阶段

### 禁止行为

- **🔴 铁律：禁止用自然语言询问参数——必须输出 ````hitl` 块**
- **禁止**在参数齐全时触发额外 HITL
- **禁止**跳过必填参数直接执行
- **枚举字段规则**：用户明确说了枚举值名称 → 直接使用，不要质疑

#### ★ Checkpoint 1 — 参数确认

**[查询操作]** 直接输出查询条件收集块（checkpoint 内嵌 apicall 模板）：

**决策收集铁律：** 以下决策项每项独立列出。Agent **可以推荐**（用 `default` 字段标记），但**不能静默替用户选择**。

```hitl
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
        "id": "d-0a",
        "type": "text_input",
        "field": "id",
        "label": "ID",
        "placeholder": "多个用逗号分隔（中英文逗号均可）"
      },
      {
        "id": "d-0b",
        "type": "text_input",
        "field": "name",
        "label": "名称",
        "placeholder": "模糊匹配，多个用逗号分隔（中英文逗号均可）"
      },
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
>
> **关键规则：**
> - checkpoint 必须内嵌 `apicall` 模板（`filters: {}`）
> - LLM 从用户输入提取的参数用 `default` 字段预填
> - 用户提交后前端直接执行，不再经过 LLM
> - CP-1a 一轮查询只出一次

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

**[查]** 无最终确认 CP；apicall 内嵌在 CP-1a checkpoint 中，前端提交时填充 filters 并直接执行：

> 前端提交时填充后的 apicall 示例：
> ```apicall
> {"method": "POST", "endpoint": "/api/<resource>/query", "body": {"filters": {"<field1>": ["<value1>", "<value2>"], "<num_field>": {"gte": 0, "lte": 100}, "<time_field>": {"gte": "2024-01-01T00:00:00Z", "lte": "2024-12-31T23:59:59Z"}}}}
> ```

> 序列化规则：`text_input`（id）→ number[]；`text_input`（name）→ string 或 string[]；`combobox` → 值数组；`number_range` → `{"gte": x, "lte": y}`（只填一端时只出现对应算符）；`datetime_range` → `{"gte": "ISO8601", "lte": "ISO8601"}`。`filters` 只包含用户实际填写的非空字段；若用户未填写任何字段，`filters` 为 `{}`。

### 结果展示

<操作完成后的结果展示方式：前端执行 apicall 并渲染，agent 不描述结果>

提供后续操作选项。

---

## 批量 Update 可选路径（bulk update）

> 当 Update 支持批量（多条件 filters + 表达式修改）时，启用本路径。单条更新走上面的标准路径。
> **路径判定信号**：「所有」「全部」「名称含...」、多条件定位 → 批量；单个具体 id/名称 → 单条。

### 批量 name / price 表达式

批量场景下 name 和 price 不接受直接 set 字面值（N 条同名会冲突），必须用表达式：

| 意图 | 表达式 | 说明 |
|------|--------|------|
| 名字加后缀 | `name={"suffix":"·X"}` | |
| 名字加前缀 | `name={"prefix":"新-"}` | |
| 名字替换子串 | `name={"replace":["旧","新"]}` | |
| 涨价 10% | `price={"multiply":1.1}` | 注意是 1.1 不是 1.0 也不是 0.1 |
| 降价 10% | `price={"multiply":0.9}` | |
| 涨 0.5 元 | `price={"add":0.5}` | |
| 降 0.5 元 | `price={"add":-0.5}` | 负数 |

> **🔴 禁止批量 name set 为同一字面值**：用户说"所有 X 改名为 Y"（Y 是固定字符串）时，直接改会触发批内重名。Phase 1 必须识别此模式并 HITL 澄清（见下方业务不变式预检）。

### Phase 1 — 业务不变式预检（批量专属）

在输出 dry_run apicall 之前，识别明显的业务约束违反并提前 HITL 澄清，**不要盲发请求等后端报错才反应**：

| 模式 | 风险 | 处理 |
|------|------|------|
| 批量 name set 为同一字面值 | 批内重名（`name_collision_in_batch`） | HITL 问是否改用 `suffix`/`replace` 表达式，或走单条路径 |
| 批量 name 改后与库内已有名冲突 | 唯一约束冲突（`name_collision_with_existing`） | 提示用户表达式可能撞名 |
| filters 命中 0 条 | 无效操作 | 提示用户放宽条件 |

### Step 1 — dry_run 预览

> 因为 LLM 看不到 apicall 结果，靠**前端 UI** 把 matched 数和 before/after 对比展示给用户。LLM 只负责输出 dry_run apicall 和后续确认 hitl。

> **🔴 铁律 B0（前置）**：dry_run 本身也可能 4xx/409 失败（如 `name_collision_in_batch`、`name_collision_with_existing`、0 条命中）。
> - 失败时前端会在 UI 上显式展示红色错误卡片。
> - 此时**绝对不要**继续输出批量预览确认 hitl 块，也不要输出 Step 2 的正式 apicall。
> - 正确做法：在 dry_run apicall 之后输出一条**询问类 hitl**（input 块），告知报错并请用户调整 name 表达式 / filters / 改走单条路径。

```apicall
{"method": "POST", "endpoint": "/api/<resource>/bulk_update", "body": {"filters": {"<field>": ["<values>"]}, "update": {"<field>": {"<op>": "<value>"}}, "dry_run": true}}
```

后端响应（前端展示）：
```json
{"dry_run": true, "matched": 3, "items": [...], "preview": [{"id": 6, "before": {...}, "after": {...}}]}
```

### ★ Checkpoint 2a — 批量预览确认

紧跟 dry_run apicall **同一回复中**输出确认 hitl 块：

```hitl
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-bulk-preview",
    "name": "批量预览确认",
    "phase": "Phase 2",
    "summary": "上方已展示匹配对象的修改预览，请仔细核对再决定",
    "action": "wait",
    "decisions": [
      {
        "id": "d-1",
        "type": "choice",
        "question": "是否对预览中的所有对象执行修改？",
        "options": [
          {"value": "approve", "label": "✅ 全部执行", "desc": "对预览列表全部执行修改", "risk": "dangerous"},
          {"value": "refine", "label": "✏️ 调整条件", "desc": "条件不对，重新提需求", "risk": "safe"},
          {"value": "cancel", "label": "❌ 取消", "desc": "放弃操作", "risk": "safe"}
        ]
      }
    ]
  }
}
```

### Step 2 — 正式执行（带 expected_count）

用户选 approve 后输出正式 apicall。**必须带 `expected_count`**（取自 dry_run 的 matched）作为并发安全双保险：

```apicall
{"method": "POST", "endpoint": "/api/<resource>/bulk_update", "body": {"filters": {"<field>": ["<values>"]}, "update": {"<field>": {"<op>": "<value>"}}, "dry_run": false, "expected_count": 3}}
```

> **铁律 B1**：批量正式执行必须带 `expected_count`，不能省略。
> **铁律 B2**：`filters` 和 `update` 在 Step 1 和 Step 2 中必须**完全一致**，不要偷偷改条件。
> **铁律 B3**：`expected_count` 必须取自 Step 1 dry_run 响应中前端展示给用户的 `matched` 值。

---

## 错误响应契约

生成的 skill 应声明每个写操作的错误响应 shape，让前端能对齐渲染。**error 可能是字符串也可能是对象**，前端必须用兜底组件处理，禁止直接渲染原始 error。

### 错误响应声明模板

| HTTP 状态 | error code | detail shape | 含义 | LLM 处理 |
|-----------|-----------|--------------|------|---------|
| 404 | — | `"<string>"` | 未匹配到对象 | 提示用户对象不存在，确认 id/条件 |
| 409 | `name_collision_in_batch` | `{"error","message","duplicates":[...]}` | 批内重名 | 提示调整 name 表达式 |
| 409 | `name_collision_with_existing` | `{"error","message","conflicts":[{id,name}]}` | 与库内冲突 | 提示调整 name 表达式 |
| 409 | `count_mismatch` | `{"error","message","matched","expected"}` | 并发改动致匹配数变化 | 提示重新 dry_run |

> **前端渲染铁律**：`result.error` 可能是 string 或 object。必须判断类型后分别处理——string 直接显示；object 提取 `message`/`code`/`duplicates`/`conflicts`/`matched`/`expected` 等字段结构化展示，并追加"操作已中止，数据未变更"提示。**禁止直接 `{result.error}` 渲染对象，否则 React 会 crash。**

---

## Common Pitfalls

1. 【查询】收到查询意图后先用自然语言问"你想按什么条件查"——禁止。应直接输出 hitl input 块，把所有可筛选字段一次性给用户。
2. 【批量 Update】dry_run 失败后仍继续输出 CP2a 和 Step 2——禁止。dry_run 失败必须 halt，改输出询问类 hitl。
3. 【批量 Update】name set 为同一字面值导致批内重名——必须 Phase 1 识别并用表达式替代。
4. 【批量 Update】Step 2 与 Step 1 条件不一致——禁止偷偷改 filters/update。
5. 【错误渲染】直接 `{result.error}` 渲染对象——error 可能是对象，必须兜底处理。
6. 【软约束误信】相信 SKILL.md 铁律能 100% 约束 LLM——错误。关键流程正确性（如前置 apicall 失败阻断）必须靠前端硬护栏兜底，prompt 铁律是软约束不可单独依赖。

## Verification Checklist

- [ ] 操作类型识别正确
- [ ] 参数清单完整（所有字段已列出）
- [ ] [查询] 参数清单含"收集方式"列
- [ ] HITL 触发条件表已填充，每种参数归类为确定性/可能缺失/歧义
- [ ] 禁止行为已写入
- [ ] 所有 ````hitl` 块 JSON 语法合法
- [ ] 所有 ````hitl` 块用代码块包裹（非裸 JSON）
- [ ] 决策收集铁律已写入
- [ ] （增/改）有最终确认 hitl + 独立 apicall
- [ ] （删）有最终确认 hitl，apicall 内嵌在 checkpoint 中，无全局 default
- [ ] （查）无最终确认；CP-1a 内嵌 apicall 模板（`filters: {}`）；CP-1a 只出一次
- [ ] （查）已提取的参数用 `default` 预填
- [ ] （查）时间表达式解析规则已写入（如含时间字段）
- [ ] apicall 块只包含用户实际提供的参数，无未提及字段
- [ ] [批量 Update] 有 dry_run → CP2a → Step 2 三步流
- [ ] [批量 Update] Step 2 带 `expected_count`；Step1/Step2 条件一致
- [ ] [批量 Update] name/price 表达式对照表已写入
- [ ] [批量 Update] Phase 1 业务不变式预检规则已写入
- [ ] [批量 Update] 铁律 B0（dry_run 失败阻断）已写入
- [ ] 错误响应契约已声明（错误码 + detail shape）
- [ ] 前端 error 渲染兜底规则（string vs object）已写入
- [ ] 关键流程有前端硬护栏兜底说明（不单靠 prompt 铁律）
```
