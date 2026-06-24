# Changelog

## v1.4.0 (2026-06-24)

基于 demo-ai-crud product-query skill 的调试经验反哺：

- 新增 `text_input` decision 类型（查询场景自由文本输入，如 id、名称）
- 查询操作工作流改为 CP-1a 内嵌 apicall 模板，前端提交时直接执行（去掉独立的参数确认 choice 块）
- 新增时间表达式解析规则模板（"今年"→01-01~12-31 等）
- 新增 LLM 行为约束：禁止裸 JSON、CP-1a 只出一次、已提取参数用 default 预填
- 后端消费指南新增：max_tokens ≥ 2048、system prompt 注入当前日期、裸 JSON 兜底解析
- 前端消费指南新增：IME 回车处理、filters 合并（baseFilters + formFilters）
- 参数清单表格增加"收集方式"列（查询操作）
- 更新校验规则：新增 text_input 类型检查（第 17 条）
- 更新 Pitfalls（新增 8 条）、Verification Checklist、Draft 自查清单
- 更新 CRUD 差异矩阵：Read 改为"CP-1a 内嵌 apicall"

## v1.2.0 (2026-06-22)

- Full rewrite following scribe pattern: core protocol in SKILL.md, details in references/
- Add CRUD batch mode support (generate 4 skills: create/query/update/delete)
- Fix: generator communicates in natural language, only generated skills use ````hitl` JSON blocks
- Add references/: hitl-protocol.md, block-templates.md, config-reference.md, crud-pattern.md
- Add templates/: generated-skill-template.md

## v1.0.0 (2026-06-22)

- Initial version: single skill generation with HITL JSON block protocol
