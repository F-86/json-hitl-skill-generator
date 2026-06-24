# Changelog

## v1.5.0 (2026-06-24)

基于 demo-ai-crud product-update 批量改名 409 防护调试经验反哺：

- **批量 Update 路径**：模板新增 dry_run → CP2a 预览确认 → Step 2 正式执行（带 `expected_count`）三步流 + 铁律 B0/B1/B2/B3
- **name/price 表达式**：新增 suffix/prefix/replace/multiply/add 对照表，禁止批量 name set 同一字面值
- **Phase 1 业务不变式预检**：批量同名、唯一冲突等模式必须提前 HITL 澄清，不盲发请求
- **错误响应契约**：模板新增段，要求生成的 skill 声明错误码 + detail shape，前后端字段名对齐
- **软约束 vs 硬护栏**：SKILL.md 新增关键架构原则段——prompt 铁律是软约束，关键流程正确性必须靠消费侧硬护栏兜底
- **前端硬护栏模式（prevFailed）**：前置 apicall 失败时过滤 `risk: "dangerous"` 选项
- **前端 ErrorBlock 渲染**：error 可能是 string 或 object，禁止直接渲染对象（React crash）
- **HITL 协议扩展**：option 新增可选 `risk: "safe" | "dangerous"` 字段；新增多步流程状态机规则（前置 apicall 失败阻断）；校验规则新增第 18 条；协议版本 1.2
- block-templates.md 新增批量 dry_run/预览确认/正式执行三件套模板 + 前端错误渲染参考
- crud-pattern.md / config-reference.md：Update 差异点补充批量 dry_run + expected_count
- Draft 自查清单新增 4 条（批量/错误/硬护栏）；Pitfalls 新增 4 条；Verification Checklist 新增 3 项

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
