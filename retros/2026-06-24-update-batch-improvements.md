# 改进计划：json-hitl-skill-generator 的不足与修改清单（Update/Batch 会话）

> 基于 `2026-06-24-update-batch.md` 总结的 7 个修改方向，对照 `json-hitl-skill-generator` 当前内容，列出需要改进的地方。
> 与 `2026-06-24-query-improvements.md`（Query 会话）互补。

---

## 不足 1：Update 仍是单条 PUT 模板，缺批量（dry_run → 确认 → 执行）两步流

**当前状态：**
- `templates/generated-skill-template.md` 的 Update 部分只有 `PUT /api/<resource>/<id>` 单条更新 + 最终 confirm hitl + 独立 apicall
- `references/crud-pattern.md` 的 Update 差异点只有"修改预览 + 最终确认 CP"，没有批量/dry_run 概念
- `references/config-reference.md` 的 CRUD 差异矩阵 Update 行只写"参数收集 + 修改预览 + 确认"
- `references/block-templates.md` 的 Update apicall 模板只有 `PUT /api/<resource>/<id>`

**实际问题（从调试中发现）：**
- 批量改名/改价是高频场景，需要先 dry_run 预览影响范围（matched 数 + before/after 对比）再执行
- 批量流程是三段：Step 1 dry_run apicall → CP2a 预览确认 hitl → Step 2 正式 apicall（带 expected_count）
- 正式执行必须带 `expected_count`（取自 dry_run 的 matched）作为并发安全双保险
- Step 1 与 Step 2 的 `filters` / `update` 必须完全一致，不能偷偷改条件

**需要修改：**
- `templates/generated-skill-template.md`：Update 段拆成「单条路径」+「批量路径」两个子段；批量路径写明 dry_run→CP2a→Step2 三步 + 铁律 B1/B2/B3（expected_count、条件一致、expected_count 来源）
- `references/crud-pattern.md`：Update 差异点新增"批量场景：dry_run 预览 + expected_count 双保险"
- `references/config-reference.md`：CRUD 差异矩阵 Update 行补充"批量需 dry_run + expected_count"
- `references/block-templates.md`：新增 Update 批量 dry_run apicall 模板 + 批量预览确认 hitl 模板 + 批量正式执行 apicall 模板

---

## 不足 2：批量场景缺 name/price 表达式（suffix/prefix/replace/multiply/add）

**当前状态：**
- 生成器完全没有提到批量更新时 name/price 等字段需要支持表达式（而非直接 set 字面值）
- 没有提到"批量 name set 为同一字面值会导致批内重名"这个业务不变式

**实际问题：**
- 批量改 name 必须用表达式：`{"suffix":"..."}` / `{"prefix":"..."}` / `{"replace":["旧","新"]}`
- 批量改 price 必须用表达式：`{"multiply":1.1}`（涨价 10%）/ `{"add":-0.5}`（降 0.5 元）
- price 表达式方向容易写反（涨价是 1.1 不是 1.0 也不是 0.1）

**需要修改：**
- `templates/generated-skill-template.md`：批量 Update 子段新增"name/price 表达式对照表"
- `references/block-templates.md`：批量 Update apicall 模板示例用表达式

---

## 不足 3：缺错误响应契约——生成 skill 不声明错误响应 shape，前端无法对齐渲染

**当前状态：**
- 生成器只在 hitl-protocol.md 的 apicall 校验规则提到"CUD 操作 apicall 前有最终确认 hitl"，完全没有错误响应的描述
- consumer pattern 只讲成功路径（hitl 渲染、apicall 执行），不讲失败路径
- 没有"前端渲染 apicall 结果时如何处理 error 是对象而非字符串"的指南

**实际问题（从调试中发现）：**
- 后端用 `HTTPException(detail={...})` 返回结构化错误对象（如 `{error, message, duplicates}`）
- 前端把 `data.detail`（对象）塞进 `result.error`，直接 `{result.error}` 渲染对象 → React crash
- 前后端字段名漂移：前端写 `existing_names`/`actual`，后端返回 `conflicts`/`matched`，冲突信息渲染不出来

**需要修改：**
- `templates/generated-skill-template.md`：新增"错误响应契约"段，要求生成的 skill 声明每个写操作的错误码 + detail shape（字段名 + 类型）
- `SKILL.md`：consumer pattern 的前端消费流程新增"apicall 结果渲染——error 可能是字符串或对象，必须用 ErrorBlock 之类组件兜底处理，禁止直接渲染原始 error"
- `references/hitl-protocol.md`：apicall 校验规则新增"建议生成的 skill 声明错误响应 shape"

---

## 不足 4：缺"软约束 vs 硬护栏"认知——prompt 铁律不可靠，需运行时硬护栏兜底

**这是本次最核心的架构教训，生成器当前完全没有这个概念。**

**当前状态：**
- 生成器把所有 LLM 行为约束都写成 SKILL.md 里的"铁律"（如"CP-1a 只出一次"、"禁止裸 JSON"）
- consumer pattern 只讲"前端解析 hitl + 渲染控件"，没有任何"当前置条件不满足时前端该如何拦截"的机制

**实际问题（从调试中发现）：**
- SKILL.md 写了"铁律 B0：dry_run 失败禁止继续输出 CP2a"，LLM 仍然违规输出"全部执行"按钮
- Prompt 层约束是软约束，LLM 会忽略/误判
- 单靠 SKILL.md 写铁律无法保证流程正确性

**需要修改：**
- `SKILL.md`：consumer pattern 新增"软约束 vs 硬护栏"段——明确 SKILL.md 铁律是软约束，关键流程正确性必须靠前端/消费层硬护栏兜底
- `SKILL.md`：前端消费流程新增"硬护栏模式：当前置 apicall 失败时，过滤掉同消息内后续 hitl 的危险决策按钮（approve/confirm/execute），仅保留 refine/modify/cancel"
- `templates/generated-skill-template.md`：Common Pitfalls 新增"相信 SKILL.md 铁律能 100% 约束 LLM——错误，关键拦截必须靠前端硬护栏"

---

## 不足 5：缺 Phase 1 业务不变式预检（识别明显违规并提前 HITL 澄清）

**当前状态：**
- 生成器的 Phase 1 只讲"参数提取 + 缺失/歧义触发 HITL"
- 没有"识别业务约束违反（如批量 set 同名、唯一冲突）并提前 HITL 澄清，而非盲发请求等后端报错"的模式

**实际问题（从调试中发现）：**
- 用户说"所有饮料改名为统一饮料"，LLM 盲发 dry_run → 后端 409 → LLM 还继续推 Step 2
- LLM 应在 Phase 1 就识别"批量 name set 为同一字面值"必然重名，提前 HITL 问是否用表达式

**需要修改：**
- `templates/generated-skill-template.md`：Update 批量子段的 Phase 1 新增"业务不变式预检"规则——识别批量 set 同名、唯一约束冲突等模式时必须先 HITL 澄清，禁止盲发 dry_run

---

## 不足 6：缺多步流程的"前置 apicall 失败阻断"状态机规则

**当前状态：**
- 生成器的 apicall 放置规则只区分"独立块 / 内嵌块"，没有"前置 apicall 失败时如何阻断后续步骤"的概念
- 没有"dry_run/预览类 apicall 也可能失败"的认知

**实际问题（从调试中发现）：**
- 隐含假设 dry_run 总成功（只是预览），实际 dry_run 也可能 4xx/409（批内重名、0 条命中）
- dry_run 失败后 LLM 仍输出 CP2a 确认块 + Step 2 执行块，导致用户能点"全部执行"触发二次失败

**需要修改：**
- `references/hitl-protocol.md`：apicall 放置规则新增"多步流程状态机"——前置 apicall（dry_run/预览）失败时，禁止后续 hitl 确认块和执行 apicall 输出
- `templates/generated-skill-template.md`：Update 批量子段新增铁律 B0——dry_run 失败时改输出询问类 hitl（input 块）让用户调整，而非继续 CP2a

---

## 不足 7：HITL 协议无法标记"危险决策选项"，前端无法通用化硬护栏

**当前状态：**
- HITL 协议的 option 对象只有 `value` / `label` / `desc`，没有"此选项会触发后续写操作 / 是危险选项"的标记
- 前端硬护栏只能靠硬编码 value 名单（approve/confirm/execute）过滤，无法通用

**实际问题：**
- 不同 skill 的危险选项 value 可能不同（如 `confirm_delete`、`bulk_approve`）
- 硬编码 value 名单不可维护

**需要修改（协议扩展，可选但推荐）：**
- `references/hitl-protocol.md`：option 对象新增可选字段 `risk: "safe" | "dangerous"`，dangerous 表示会触发后续写操作
- 前端硬护栏可通用化为"前置 apicall 失败时，过滤 risk=dangerous 的选项"
- 这是协议演进，需同步更新校验规则

---

## 不足 8：Draft 自查清单 / Verification Checklist 未覆盖批量与错误场景

**当前状态：**
- `SKILL.md` 的 Draft 自查清单 10 条全是 Query/单条 CUD 相关，无批量、无错误响应、无硬护栏
- Verification Checklist 同样

**需要修改：**
- `SKILL.md`：Draft 自查清单新增：批量 Update 有 dry_run→CP2a→Step2？expected_count 存在？Step1/Step2 条件一致？错误响应 shape 已声明？
- `SKILL.md`：Verification Checklist 新增对应检查项

---

## 不足 9：CHANGELOG 未记录本次反哺

**需要修改：**
- `CHANGELOG.md`：新增 v1.5.0 条目，记录本次 Update/Batch 会话的反哺改动
