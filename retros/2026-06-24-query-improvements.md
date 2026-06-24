# 改进计划：json-hitl-skill-generator 的不足与修改清单

> 基于 `2026-06-24-query.md` 中总结的修改方向，对照 json-hitl-skill-generator 当前内容，列出需要改进的地方。

---

## 不足 1：查询 skill 的工作流模板不匹配实际最佳实践

**当前状态：**
- `templates/generated-skill-template.md` 中查询操作的流程是：CP-1 参数确认（choice 类型，让用户选 execute/modify/cancel）→ 独立 apicall
- `references/block-templates.md` 中 Read 的参数确认用的是 `choice` 类型（execute/modify/cancel）
- `references/hitl-protocol.md` 的放置规则表写的是"Read/Query: 独立块，参数确认 hitl 之后"

**实际问题：**
- 查询场景不需要 choice 类型的"参数确认"——用户填完表单就应该直接执行
- 正确做法是：CP-1a 的 checkpoint 内嵌 `apicall` 模板（`filters: {}`），前端提交时注入 filters 直接执行
- 不需要独立的 apicall 块跟在 hitl 后面——apicall 已经内嵌在 hitl checkpoint 中

**需要修改：**
- `templates/generated-skill-template.md`：查询操作的 Phase 1 改为"CP-1a 内嵌 apicall 模板，前端提交时直接执行"
- `references/block-templates.md`：Read 的参数确认块改为"内嵌 apicall 的条件收集块"
- `references/hitl-protocol.md`：放置规则表中 Read/Query 改为"apicall 内嵌在 hitl checkpoint 中"

---

## 不足 2：缺少 `text_input` decision 类型

**当前状态：**
- `references/hitl-protocol.md` 的 decision 类型只有：choice / confirm / review / input / combobox / number_range / datetime_range
- `references/block-templates.md` 没有 text_input 模板
- SKILL.md 的控件映射表没有 text_input
- 校验规则第 10 条的 type 枚举没有 text_input

**实际问题：**
- 查询场景中 id 和 name 等字段需要自由文本输入，但不适合用 `input`（input 是 Create/Update 专用的结构化表单）
- 不适合用 `combobox`（选项不是从后端拉取的枚举）
- 需要一个轻量的单行文本输入类型

**需要修改：**
- `references/hitl-protocol.md`：新增 `text_input` 类型定义、字段说明、校验规则
- `references/block-templates.md`：新增 text_input 模板
- `SKILL.md`：控件映射表新增 text_input
- 校验规则第 10 条的 type 枚举加入 text_input

---

## 不足 3：缺少时间表达式解析规则

**当前状态：**
- 生成器完全没有提到 LLM 如何处理"今年"、"去年"、"本月"等相对时间表达式
- 没有提到 system prompt 需要注入当前日期

**实际问题：**
- LLM 的训练数据有截止日期，不知道"今天"是哪天
- LLM 对"今年"的结束时间可能取今天而非年底

**需要修改：**
- `templates/generated-skill-template.md`：查询操作模板中增加"时间表达式解析规则"段
- `SKILL.md`：在 HITL Consumer Pattern 的后端消费流程中，增加"system prompt 注入当前日期"的说明

---

## 不足 4：缺少 LLM 行为约束规则

**当前状态：**
- SKILL.md 的 Pitfalls 第 6 条提到了"Phase 指令强度不足"，但没有以下具体约束

**实际问题（从调试中发现）：**
- LLM 有时输出裸 JSON（不加 ```hitl 包裹），前端无法渲染
- LLM 有时把同一个 CP 输出两次（分多轮询问）
- LLM 有时不用 default 预填已提取的参数
- LLM 有时在收到表单提交后还输出自然语言询问

**需要修改：**
- `SKILL.md`：Pitfalls 新增 4 条
- `templates/generated-skill-template.md`：查询模板的禁止行为段新增对应规则

---

## 不足 5：缺少后端消费指南中的关键配置

**当前状态：**
- SKILL.md 的后端消费流程提到了路由、system prompt、history，但没有提到 max_tokens 和裸 JSON 兜底

**实际问题：**
- max_tokens=512 会导致 hitl JSON 被截断
- LLM 有时输出裸 JSON，后端需要兜底解析

**需要修改：**
- `SKILL.md`：后端消费流程新增 max_tokens 建议和裸 JSON 兜底说明

---

## 不足 6：缺少前端消费指南中的关键处理

**当前状态：**
- SKILL.md 的前端消费流程提到了 hitl 解析、apicall 执行，但没有提到 IME 处理和 filters 合并

**实际问题：**
- 中文输入法下回车会误触发消息发送
- 前端提交表单时需要合并 LLM 预填的 filters 和表单填写的字段

**需要修改：**
- `SKILL.md`：前端消费流程新增 IME 处理和 filters 合并说明

---

## 不足 7：参数规范表格缺少"收集方式"列

**当前状态：**
- `templates/generated-skill-template.md` 的参数清单表格只有：参数 / 必填 / 类型 / 默认值 / 说明

**实际问题：**
- 查询场景中，每个参数的"收集方式"（text_input / combobox / number_range / datetime_range / 直接提取）是关键信息
- 没有这一列，生成的 skill 容易在 decision 类型选择上出错

**需要修改：**
- `templates/generated-skill-template.md`：查询操作的参数清单表格增加"收集方式"列

---

## 不足 8：`input` 类型的使用范围说明需更新

**当前状态：**
- `references/hitl-protocol.md` 和 `references/block-templates.md` 中 `input` 类型标注"仅限 Create/Update 场景"
- 但没有说明查询场景中需要自由文本输入时应该用什么类型

**需要修改：**
- `references/hitl-protocol.md`：input 类型说明中补充"查询场景的自由文本输入用 `text_input`"
- `references/block-templates.md`：input 模板的用途说明同步更新

---

## 不足 9：CRUD 差异矩阵中 Read 的描述需更新

**当前状态：**
- `references/config-reference.md` 的 CRUD 差异矩阵：Read "不需要最终确认"，但没有说明 apicall 内嵌方式
- `references/crud-pattern.md`：Read "去掉最终确认 CP，增加结果展示阶段"

**实际问题：**
- Read 的正确做法是 CP-1a 内嵌 apicall，前端提交时直接执行，不需要独立 apicall 块

**需要修改：**
- `references/config-reference.md`：Read 行的"特殊规则"补充"apicall 内嵌在 CP-1a checkpoint 中"
- `references/crud-pattern.md`：Read 的差异点调整说明更新

---

## 不足 10：校验规则和 Verification Checklist 需同步更新

**当前状态：**
- 校验规则第 10 条的 type 枚举没有 text_input
- Verification Checklist 没有检查"查询 CP 是否内嵌 apicall"

**需要修改：**
- `references/hitl-protocol.md`：校验规则更新
- `SKILL.md`：Verification Checklist 新增查询相关检查项
