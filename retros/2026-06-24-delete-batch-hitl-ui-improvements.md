# 改进计划：json-hitl-skill-generator 在 Delete 批量流程与 HITL 消费上的不足（2026-06-24）

> 基于 `2026-06-24-delete-batch-hitl-ui.md` 的真实调试结果，对照 `json-hitl-skill-generator` 当前内容，列出需要补强的地方。

---

## 不足 1：Delete 仍主要被建模为单条确认，缺少“批量 Delete 双路径”抽象

**当前状态：**

- `templates/generated-skill-template.md` 对 Delete 的描述仍以单条 `DELETE /api/<resource>/<id>` 为主
- `references/block-templates.md` 只有单条 Delete 最终确认模板
- `references/crud-pattern.md` 与 `references/config-reference.md` 对 Delete 的差异描述中，没有批量 Delete 的明确位置

**实际问题：**

- 真实项目里 Delete 已经不是只有“按 ID 删除”这一种路径
- 批量条件删除是常见场景，且风险不低于批量 Update
- 如果生成器不抽象双路径，生成出来的 delete skill 会天然偏向单条删除

**需要修改：**

- `templates/generated-skill-template.md`：Delete 段补充“单条路径 + 批量路径”的判定与工作流
- `references/block-templates.md`：新增批量 Delete 的 dry_run / 预览确认 / 最终确认模板
- `references/crud-pattern.md`：Delete 差异点补充“条件删除 + dry_run 预览 + 最终确认”
- `references/config-reference.md`：CRUD 差异矩阵 Delete 行补充批量删除特殊规则

---

## 不足 2：`dry_run → 预览确认 → 正式执行` 仍被写成 Update 专属，未升级为“批量写操作通用模式”

**当前状态：**

- 生成器已完整支持批量 Update 的三段流
- 但 Delete 侧还没有同等级抽象

**实际问题：**

- 批量 Delete 与批量 Update 一样，都有“先看范围再执行”的强需求
- 如果只在 Update 中强调三段流，会导致 Delete 生成出来仍是“一步确认后直接删”

**需要修改：**

- `SKILL.md`：把 `dry_run → 预览确认 → 正式执行` 升级为“批量写操作通用风控模式”
- `templates/generated-skill-template.md`：Delete 增加与 Update 对称的批量三段流子章节
- `references/config-reference.md`：把 Delete 的 CP/Phase 说明补齐到批量场景

---

## 不足 3：`expected_count` 双保险仍被写成 Update 专属，没有抽象成“预览后执行类写操作”的通用要求

**当前状态：**

- 现在文档里 `expected_count` 主要围绕批量 Update 展开
- Delete 场景没有同步要求

**实际问题：**

- 批量 Delete 正式执行前同样应该验证“本次正式命中数 == 预览时 matched”
- 否则 Delete 的并发安全性和风控完整性比 Update 弱一截

**需要修改：**

- `SKILL.md`：明确 `expected_count` 适用于所有“预览后正式执行”的批量写操作，而非只限 Update
- `references/block-templates.md`：新增 Delete 正式执行模板，带 `expected_count`
- `templates/generated-skill-template.md`：Delete 批量路径写入 `expected_count` 铁律

---

## 不足 4：缺少“结构化 HITL 回执”这一消费侧契约说明

**当前状态：**

- 生成器已定义输出的 `hitl` / `apicall` 块协议
- 但对“前端如何把用户点击回传给后端/LLM”写得不够

**实际问题：**

- 批量 Delete 的最终确认依赖 `dry_run.matched`
- 前端在“继续确认”时需要把 `approve` 归一化为：
  - 纯动作值，或
  - 结构化 JSON 回执，如 `{"action":"approve","expected_count":3}`
- 如果没有这层契约，生成出来的 skill 往往只会假设用户回的是字符串 `approve`

**需要修改：**

- `SKILL.md`：在 consumer pattern 中新增“结构化回执归一化”说明
- `references/hitl-protocol.md`：补一段“多步流程的用户回执归一化建议”
- `templates/generated-skill-template.md`：批量 Delete / Update 场景明确写“收到结构化 JSON 回执时优先识别并复用其中的 expected_count”

---

## 不足 5：对 ` ```text ` 与 ` ```hitl ` / ` ```apicall ` 的职责区分还不够强，容易误导运行时输出

**当前状态：**

- 生成器已经有“协议示例块用 `text`，避免嵌套 fence”规则
- 但运行时与文档示例的边界还不够醒目

**实际问题：**

- 真实项目里模型把 checkpoint JSON 包在了 ` ```text ` 里
- 后端原先只认 ` ```hitl `，导致前端不渲染
- 这说明生成器必须更强地强调：**示例展示语法 ≠ 运行时输出语法**

**需要修改：**

- `SKILL.md`：补充“文档示例 fence / 运行时 fence”双层规则
- `references/hitl-protocol.md`：补充“运行时必须输出 `hitl` / `apicall`；`text` 仅用于文档示例”的说明
- `templates/generated-skill-template.md`：Delete/Update 多步模板中显式加入这条运行时铁律

---

## 不足 6：缺少消费侧对 ` ```text ` 误包裹 JSON 的兼容建议

**当前状态：**

- 当前生成器强调了裸 JSON 兜底
- 但没有提到“模型偶尔会把 runtime JSON 放进 ` ```text `”这种现实问题

**实际问题：**

- 仅防裸 JSON 不够
- 实际上还需要消费侧能兼容 ` ```text ` 包裹的 checkpoint/apicall JSON

**需要修改：**

- `SKILL.md`：后端消费流程新增“兼容解析 ` ```text ` 包裹的 runtime JSON”建议
- `references/hitl-protocol.md`：放进消费侧兼容建议，而非协议本体规则

---

## 不足 7：前端消费指南没有强调“自然语言用户消息 ≠ HITL 内部回执”

**当前状态：**

- 当前 consumer pattern 更关注：解析 hitl、执行 apicall、过滤危险按钮
- 还没覆盖“消息展示语义”这一层

**实际问题：**

- 若前端把上一条 AI HITL 后的下一条 user 消息统一当成回执，会吞掉用户自然语言气泡
- 这直接破坏对话可理解性，也会让用户误以为自己消息消失

**需要修改：**

- `SKILL.md`：前端消费流程新增“内部回执与自然语言消息分流”规则
- 建议：
  - 结构化回执 / 内部动作值（如 `approve` / JSON）可按内部协议处理
  - 自然语言回复必须保留为独立用户气泡

---

## 不足 8：前端消费指南还缺少“批量 Delete 预览态 / 完成态结果卡片”意识

**当前状态：**

- 当前说明里强调了 apicall 执行与错误卡片
- 但对批量删除成功/预览的 UI 状态没有成体系说明

**实际问题：**

- 批量 Delete 不是只有“执行成功/失败”两态
- 至少有：
  - dry_run 预览态
  - 正式删除完成态
- 如果没有结果卡片约定，生成 skill 时就会缺少对消费侧 UI 的指导

**需要修改：**

- `SKILL.md`：前端消费模式中补充“批量预览结果 / 正式执行结果”的展示建议
- `references/block-templates.md`：可补充 delete 预览与 delete 完成态的结果结构说明

---

## 不足 9：Verification Checklist / Draft 自查清单尚未覆盖本轮新增风险点

**当前状态：**

- 目前 checklist 已覆盖 query、batch update、error block、硬护栏
- 但还没覆盖 Delete 批量化和消息语义问题

**需要补的检查项：**

- [ ] Delete 是否支持“单条 + 批量”双路径
- [ ] 批量 Delete 是否走 dry_run → 预览确认 → 最终确认
- [ ] 批量 Delete 正式执行是否带 `expected_count`
- [ ] 运行时是否明确要求用 ` ```hitl ` / ` ```apicall ` 而不是 ` ```text `
- [ ] consumer 是否说明了 ` ```text ` runtime JSON 的兼容解析
- [ ] consumer 是否说明了“内部回执与自然语言用户消息分流”

---

## 不足 10：README / retros 索引还未纳入本轮回顾与改进文档

**需要修改：**

- `README.md`：项目结构中的 retros 列表加入本轮 2~3 个新文件
- `retros/README.md`：索引表加入本轮回顾 / 改进 / 落地日志
- `CHANGELOG.md`：新增版本记录，说明 Delete 批量流程与 HITL 消费语义修正
