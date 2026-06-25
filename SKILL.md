---
name: json-hitl-skill-generator
description: >-
  生成使用 HITL (Human-in-the-Loop) JSON 块协议的 SKILL.md。支持单 skill 生成
  和 CRUD 批量生成（一次输入任务类型 → 输出增删改查 4 个 skill）。
  关键词：HITL、skill 生成、CRUD 批量生成、checkpoint 协议、人机协作、元技能
version: 1.6.0
author: Jane
license: MIT
metadata:
  hermes:
    tags: [skill-generator, hitl, human-in-the-loop, meta-skill, checkpoint-protocol, crud]
    related_skills: [hermes-agent-skill-authoring, harness-framework]
---

# JSON HITL Skill Generator

> 生成一个使用 ````hitl` JSON 块作为人机交互协议的 SKILL.md。

## Overview

这是一个**元技能（meta-skill）**—— 输出是一个完整的、可直接运行的 SKILL.md。

**三种生成模式：**

| 模式 | 输出 | 适用场景 |
|------|------|---------|
| **标准模式** | 1 个 SKILL.md | 单个独立的 skill（非 CRUD） |
| **CRUD 单个模式** | 1 个 SKILL.md | 同一任务类型下只生成指定操作 |
| **CRUD 批量模式** | 4 个 SKILL.md（增-查-改-删） | 同一任务类型的全部操作场景 |

生成的 skill 拥有以下特征：

- 遵循 **Harness 6 维度** 架构设计
- 所有人机交互点通过 ````hitl` JSON 代码块表达
- JSON 块可被前端工具消费，用于自动渲染 UI 或路由决策
- CRUD 批量模式走 Day 1 原型 → 确认 → Day 2 批量交付；CRUD 单个模式直接生成 1 个

## HITL JSON 块协议

本 skill 定义 ````hitl` JSON 代码块的协议格式。它生成的所有 SKILL.md 在运行时使用该协议与终端用户交互——生成器本身用自然语言沟通。

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-1",
    "name": "Plan 确认",
    "phase": "Phase 2",
    "summary": "已完成规划，等待用户决策",
    "action": "wait",
    "decisions": [
      {
        "id": "d-1",
        "type": "choice",
        "question": "你是否同意这个大纲？",
        "options": [
          {"value": "approve", "label": "批准", "desc": "进入生成阶段"},
          {"value": "modify", "label": "修改", "desc": "用户提出调整"},
          {"value": "reject", "label": "拒绝", "desc": "重新规划"}
        ]
      }
    ]
  }
}
```

## HITL Consumer Pattern

本 skill 生成的 SKILL.md 需要在应用中**消费**：后端加载 SKILL.md 作为 LLM system prompt，LLM 按 Phase 指令输出 ````hitl` JSON 块，前端解析并渲染为交互控件。

完整参考实现：`/root/code/project/demo-ai-crud/`
详细说明见 `references/consumer-pattern.md`（若不存在则参考 demo 项目）。

### 后端消费流程

1. 启动时 `SkillRegistry` 扫描 `skills/<name>/SKILL.md` 注册所有 skill
2. 用户消息 → `_route_skill()` → LLM 判断意图属于哪个 skill
3. `execute_skill()` 取 SKILL.md 完整内容作 system prompt
4. 最近 N 轮对话历史 + 用户消息 → 调 LLM → LLM 按 Phase 指令输出 reply

### 前端消费流程

1. `AIChat.js` 检测 `data.hitl` → 解析 ````hitl` JSON → 传给 `HITLWidget`
2. `HITLWidget` 按 `decision.type` 映射为不同 UI 组件
3. 多个 decision 展示为**轮换图**（每个 decision 一屏，圆点导航）
4. `data.apicall` → 前端自动执行 API 调用，并区分**预览态**（如 dry_run）与**正式执行完成态**两类结果卡片
5. **多步流程回执归一化**：当下一步需要复用前一步 apicall 结果（如 `matched → expected_count`）时，前端可把按钮动作归一化为结构化 JSON 回执，例如 `{"action":"approve","expected_count":3}`
6. **消息语义分流**：内部协议回执（如 `approve` / `confirm` / 结构化 JSON）可按 HITL 内部回执处理；**自然语言用户消息必须保留为独立用户气泡**，不要折叠成 AI 卡片的附属回执

### 控件映射

| decision.type | HITLWidget 渲染 | 说明 |
|--------------|----------------|------|
| choice | 选项按钮列表 | 点击触发操作 |
| confirm | 确认/取消按钮 | 二元决策 |
| input | 结构化表单（fields 数组） | Create/Update 收集；查询场景禁用 |
| text_input | 单行文本输入框 | 查询场景自由文本（id、名称等） |
| combobox | 标签选择器 | 从后端 API 拉取选项 |
| number_range | 最小/最大值输入 | 数字区间 |
| datetime_range | 日期选择器 | 使用 DateRangePicker |

### 关键约束

- ````hitl` 块必须附带自然语言引导语，不能裸扔 JSON
- **文档示例块**可用 ```` ```text ```` 展示协议与对话示例；**真实运行输出**必须使用 ```` ```hitl ```` / ```` ```apicall ````，禁止把运行时 checkpoint JSON 包在 ```` ```text ```` 里
- ````hitl` / ````apicall` 块必须用代码块包裹，禁止裸 JSON 输出
- LLM system prompt = SKILL.md 完整内容（含 Phase 指令、触发条件）
- **system prompt 应注入当前日期**（如"今天的日期是 2026-06-24"），否则 LLM 不知道"今年"是哪年
- **max_tokens 建议 ≥ 2048**——HITL JSON 块较大，512 会导致截断
- **后端应增加双重兜底解析**——既要兼容裸 JSON（`{` 开头且含 `"checkpoint"` / `"method"`），也建议兼容 LLM 偶发把运行时 JSON 包进 ```` ```text ```` 的情况
- 前端 API_BASE 不能硬编码 localhost，使用相对路径或环境变量
- 前端输入框需处理 IME（中文输入法）——`handleKeyDown` 中检查 `!e.nativeEvent.isComposing`，避免输入法选字时回车误触发发送
- 多步批量写操作（如批量 Update / Delete）若下一步依赖前一步 `matched`，消费侧应支持把按钮动作归一化为结构化 JSON 回执（如 `{"action":"approve","expected_count":3}`）
- 前端展示历史消息时，应区分**内部协议回执**与**自然语言用户消息**；自然语言消息必须保留为独立用户气泡，不能被折叠成 AI 的 HITL 附属回复
- history LIMIT 和 LLM context slice 的 N 值保持一致

### 软约束 vs 硬护栏（关键架构原则）

SKILL.md 中写给 LLM 的"铁律"是**软约束**——LLM 会忽略或误判，无法 100% 保证行为正确。关键流程正确性必须靠**消费侧硬护栏**兜底。

| 风险类型 | 软约束（SKILL.md 铁律） | 硬护栏（消费侧运行时） |
|---------|----------------------|---------------------|
| dry_run 失败后继续执行 | 铁律 B0：禁止继续输出 CP2a/Step2 | 前端：前置 apicall 失败时过滤 `risk: "dangerous"` 选项 |
| 裸 JSON 输出 | 禁止裸 JSON | 后端：检测 `{` 开头且含 `"checkpoint"` 直接解析 |
| CP 重复输出 | CP-1a 只出一次 | 前端：同一 CP id 只渲染一次 |

> **铁律：** 凡是"前置步骤失败必须阻断后续"的流程，都不能只靠 SKILL.md 写铁律，必须在消费侧实现硬护栏。生成的 skill 应在「错误响应契约」段声明错误 shape，让前端能判断前置 apicall 是否失败。

### 前端硬护栏模式（prevFailed）

当一条 AI 消息内同时包含前置 apicall 和后续 hitl 块时，前端应检测前置 apicall 结果：

- 若 `apicallResult.error !== undefined`（前置失败），给 HITLWidget 传 `prevFailed` 标志
- `prevFailed=true` 时：
  - 过滤 `risk: "dangerous"` 的选项（或硬编码过滤 approve/confirm/execute 类 value）
  - confirm 类型直接隐藏"确认"按钮
  - 仅保留 refine/modify/cancel 等安全选项
  - 底部追加提示："⚠️ 上一步操作失败，已隐藏执行类按钮"

### 前端 apicall 结果渲染（ErrorBlock）

`apicall` 执行结果的 `error` 字段**可能是字符串也可能是对象**（后端 `HTTPException(detail={...})` 返回结构化错误时，detail 是对象）。**禁止**直接 `{result.error}` 渲染对象（React 会 crash）。必须判断类型后分别处理：

- string → 直接显示
- object → 提取 `message`/`code`/`duplicates`/`conflicts`/`matched`/`expected` 等字段结构化展示
- 追加"操作已中止，数据未变更"提示，避免用户误以为成功

详见 `references/block-templates.md` 的「前端错误渲染参考」段。

## 边界

- **进入条件**：用户明确要生成/创建一个新的 SKILL.md
- **特别检查**：如果用户说"增删改查"、"多种操作场景"，优先进入 CRUD 模式
- **不处理**：修改现有 skill（待建）、生成非 skill 文档（走 scribe）、生成代码实现

## Harness 视角

| 维度 | 本 Skill 的设计 | 实现手段 |
|------|----------------|---------|
| CONTEXT | 渐进加载 | 标准模式只看边界+协议；CRUD 模式额外加载 crud-pattern.md |
| TOOLS | 输入输出 | 用户需求描述/文件读取；输出 1 或 4 个 SKILL.md |
| ORCHESTRATION | Phase 管道 | 6 Phase 线性管道 + 3 Checkpoint |
| MEMORY | 文件系统 | .json-hitl-gen/ 工作区 |
| EVALUATION | 分层质检 | 内联自查 + 结构化 HITL JSON 校验 |
| RECOVERY | 约束恢复 | Checkpoint 确认制 + 最小切片修复 |

## 各阶段文件读取指南

| 阶段 | 必读 | 按需查 |
|------|------|--------|
| Phase 0 | 本 SKILL.md（边界+工作流） | — |
| Phase 1 | references/hitl-protocol.md（CRUD 加 crud-pattern.md） | — |
| Phase 2 | references/block-templates.md | references/config-reference.md |
| Phase 3 | templates/generated-skill-template.md | references/block-templates.md |
| Phase 4 | references/hitl-protocol.md（校验规则段） | — |

## Phase 0 — Intake

### 判断工作模式

| 信号 | 模式 |
|------|------|
| "生成一个 skill"、"写个 skill" | 标准模式 |
| "我的项目有订单管理"、"增删改查" | 问清批量还是单个 |
| "增删改查都要"、"全部操作" | CRUD 批量模式 |
| "只先做个查询"、"只生成删除" | CRUD 单个模式 |

### CRUD 模式：收集需求

第一轮（必问）：
1. 任务类型名称
2. 任务类型英文标识
3. CRUD 操作说明
4. 操作参数详细结构（逐参数标注确定性/可能缺失/歧义）

第二轮（按需追问）：确认强度、输出形式、目标 agent

### 标准模式：收集需求

1. Skill 用途、触发条件、关键阶段、输出形式

## Phase 1 — Analyze

### CRUD 模式分析

加载 references/crud-pattern.md。关键决策：
1. 4 个 skill 公共结构
2. CRUD 差异矩阵（增改删有 final CP，Read 无）
3. HITL 触发条件提取（三步：分类→映射→写入模板）
4. **批量写操作共性**：Update / Delete 一旦涉及按条件批量执行，应优先抽象为 `dry_run → 预览确认 → 正式执行`，并评估是否需要 `expected_count`、结构化回执归一化、前端预览结果卡片与自然语言消息分流

## Phase 3 — Generate

### CRUD 模式——分两个子阶段

子阶段 A（Day 1）：生成原型 skill → 用户确认
子阶段 B（Day 2）：按差异矩阵生成剩余 skill

### 生成质量硬性规则

规则 1：HITL 协议示例块用 ```text，不用嵌套 fence
规则 2：对话示例用 > 块引用 + 独立 hitl 块
规则 3：JSON 块内禁止非法占位符
规则 4：必须明确区分**文档示例 fence**与**运行时输出 fence**——示例可用 ```text，运行时必须要求输出 ```hitl / ```apicall
规则 5：涉及预览后正式执行的批量写操作时，必须显式说明是否需要 `expected_count` 与结构化回执归一化

### Draft 自查清单

1. 所有 JSON 块合法？无 ... / [...] / <...>
2. 增/改/删 Phase 1 有"必须输出 hitl 块"铁律 + "已提取字段不重复问"规则？
3. 枚举参数有"精确值直接使用"规则？
4. 增/改/删 CP-1 用 `input` + `fields[].type: "choice"` 而非独立 `choice`？
5. ````apicall` 块用 `"<参数名>"` 而非 `{{param}}`？
6. 查询 CP-1a 内嵌 apicall 模板（`filters: {}`）？
7. 查询 CP-1a 只出一次 + 已提取参数用 default 预填？
8. 查询场景禁止 `input`，用 `text_input` / `combobox` / `number_range` / `datetime_range`？
9. 含时间字段时有时间表达式解析规则？
10. HITL 块用 ```` ```hitl ```` 包裹，禁止裸 JSON？
11. [批量 Update] 有 dry_run → CP2a → Step 2 三步流？Step 2 带 `expected_count`？Step1/Step2 条件一致？
12. [批量 Update] name/price 表达式对照表已写入？Phase 1 业务不变式预检已写入？铁律 B0 已写入？
13. 写操作的错误响应契约已声明（错误码 + detail shape）？
14. 前端 error 渲染兜底规则（string vs object）+ 硬护栏模式已说明？
15. [批量 Delete] 是否支持单条/批量双路径，且批量路径走 `dry_run → CP2a → CP2b`？
16. [批量 Delete] 正式删除是否带 `expected_count`，且 Step1/Step2 的 filters 完全一致？
17. 是否明确区分了文档示例 fence 与运行时 fence（运行时禁止输出 ```text 包裹的 checkpoint JSON）？
18. consumer 是否说明了结构化回执归一化，以及“内部协议回执 vs 自然语言用户消息”的分流规则？

## Phase 4 — Review

- JSON 合法性校验（18 条规则）
## 常见 Pitfalls

1. **JSON 块中使用非法占位符**：`"..."`、`[...]`、`<Decision>` 等不是合法 JSON。所有示例块中必须用合规 JSON 值填充。

2. **HITL 协议示例用嵌套 fence**：````markdown` 包裹 ````hitl` 会导致 Markdown 解析器误匹配。必须用 ````text` 展示协议示例。

3. **对话示例混合 fence**：```` 围栏内嵌入 ````hitl` 块 → 外层围栏被内层 ```` 意外关闭。改用块引用 `>` + 独立 ````text` 块。

4. **````apicall` 块使用 `{{param}}` 模板语法**：`{{param}}` 不是合法 JSON。应使用 `"<参数名>"` 占位符。

5. **HITL decision type 与前端渲染能力不匹配**：生成 skill 前需确认目标前端 HITLWidget 支持哪些 `decision.type`。例如 demo 前端的 HITLWidget **不支持独立的 `choice` 类型**（带内联 `options` 的 `choice` 不渲染任何控件）。替代方案：
   - 枚举选择 → 用 `type: "input"` + `fields[].type: "choice"` + `options` 数组，渲染为下拉框
   - 多选标签 → 用 `type: "combobox"` + `options_from.endpoint` 指向后端 API
   - 确认 → 用 `type: "confirm"`（仅当前端支持）

6. **Phase 指令强度不足导致 LLM 用自然语言替代 HITL 块**：LLM 默认用自然语言对话。如果 Phase 1 指令说"缺失时触发 HITL"，LLM 会解释为"用自然语言问用户"。**必须写"必须输出 ````hitl` 块，禁止用自然语言询问"**，且放在禁止行为段第一条作为 🔴 铁律。

7. **枚举参数（category 等）缺少"精确值直接使用"规则**：如果用户明确说了枚举枚举值（如"服装"、"数码"），SKILL.md 必须写明：直接使用该值，不要质疑。只有模糊描述（"吃的"、"穿的"）才需要 HITL 让用户选择。不加这条，LLM 会连明确值也质疑。

8. **LLM 输出裸 JSON（不加代码块包裹）**：LLM 有时直接输出以 `{` 开头的 JSON 文本，不加 ```` ```hitl ```` 包裹，导致前端无法解析。SKILL.md 必须写明"HITL 块必须用 ```` ```hitl ```` 包裹，禁止裸 JSON 输出"。后端也应增加兜底：检测到 `{` 开头且含 `"checkpoint"` 的回复直接解析。

9. **查询 CP 出现两次**：LLM 有时分多轮询问（先问价格、再问分类），而非一次性收集。SKILL.md 必须写明"CP-1a 一轮查询只出一次，禁止分多轮询问"。

10. **已提取的参数忘记用 default 预填**：LLM 从用户输入提取到参数（如价格 10~100）后，应填入对应 decision 的 `default` 字段，而不是只在自然语言里提到。不预填的话用户需要重新输入。

11. **查询操作不需要最终确认**：与 CUD 不同，查询操作不需要 choice 类型的"参数确认"步骤。正确做法是 CP-1a 内嵌 apicall 模板，前端提交时直接执行。

12. **max_tokens 过小导致 JSON 截断**：HITL JSON 块较大（含多个 decision），max_tokens=512 会导致输出被截断、JSON 不完整。建议 ≥ 2048。

13. **system prompt 未注入当前日期**：LLM 训练数据有截止日期，不知道"今天"是哪天。system prompt 应注入"今天的日期是 YYYY-MM-DD"，否则"今年"等时间表达式会解析错误。

14. **时间表达式结束时间取了今天而非区间最后一天**：用户说"今年"，LLM 可能把 lte 设成今天而非 12-31。SKILL.md 必须包含时间表达式解析规则表，明确"结束时间取区间最后一天"。

15. **前端 IME 回车误触发**：中文输入法下按回车确认选字，不应触发消息发送。前端 `handleKeyDown` 需检查 `!e.nativeEvent.isComposing`。

16. **前端提交时 filters 覆盖问题**：当 LLM 预填了部分 filters（如 id），前端提交表单时不能直接覆盖 `apicall.body.filters`，而应合并：`{ ...baseFilters, ...formFilters }`。

17. 没有引导语就扔出 JSON 块
18. Checkpoint 过多
19. CRUD 模式：Delete 设了全局 default
20. 决策选项语义模糊
21. 忘记决策收集铁律
22. 生成的 skill 没有边界条件
23. **相信 SKILL.md 铁律能 100% 约束 LLM**：prompt 铁律是软约束，LLM 会忽略/误判（如 dry_run 失败后仍输出 CP2a）。关键流程正确性必须靠前端硬护栏兜底——当前置 apicall 失败时过滤 `risk: "dangerous"` 选项。
24. **前端直接 `{result.error}` 渲染对象**：后端 `HTTPException(detail={...})` 返回结构化错误时，error 是对象，直接渲染会导致 React crash。必须用 ErrorBlock 兜底处理 string vs object。
25. **批量 Update name set 为同一字面值**：N 条同名必触发批内重名 409。Phase 1 必须识别并用表达式（suffix/replace）替代，不要盲发 dry_run 等后端报错。
26. **跨层字段名漂移**：前端 ErrorBlock 字段名与后端实际返回不一致（如前端写 `existing_names`，后端返回 `conflicts`）。生成的 skill 必须在「错误响应契约」段声明字段名。
27. **把文档示例 ` ```text ` 写法误当成运行时输出格式**：协议示例为了避免嵌套 fence 才使用 `text`；真实运行时 checkpoint / apicall 必须输出 ` ```hitl ` / ` ```apicall `。
28. **多步流程只假设用户回 `approve` 字符串**：当 Step 2 需要复用 Step 1 的 `matched` 时，消费侧可能回传结构化 JSON（如 `{"action":"approve","expected_count":3}`）。生成的 skill 必须明确“优先识别结构化回执”。
29. **把自然语言用户消息折叠成 HITL 内部回执**：前端若把上一条 AI HITL 后的下一条 user 消息一律当内部回执，会吞掉真实用户气泡。consumer 指南必须写清：内部动作值/结构化 JSON 与自然语言消息要分流。
30. **批量 Delete 仍按单条 Delete 模板生成**：Delete 一旦支持按条件删除，就必须像批量 Update 一样补上 dry_run 预览、预览确认、最终确认与 `expected_count` 双保险，而不是只复用单条 DELETE 确认块。

## references/ 目录索引

| 文件 | 用途 |
|------|------|
| references/hitl-protocol.md | JSON 块协议完整规范 |
| references/block-templates.md | 常见 hitl 块模板 |
| references/config-reference.md | 生成配置参考 |
| references/crud-pattern.md | CRUD 批量生成模式参考 |
| (project) demo-ai-crud/ | HITL consumer 参考实现（前后端架构/消费流程） |

## Verification Checklist

- [ ] 模式判断正确
- [ ] JSON 块全部合法
- [ ] 协议示例块用 ```text
- [ ] 对话示例无混合 fence
- [ ] Frontmatter 合法
- [ ] 每个 Checkpoint 附决策收集铁律
- [ ] Read 无最终确认
- [ ] Read 的 CP-1a 内嵌 apicall 模板（`filters: {}`）
- [ ] Read 的 CP-1a 只出一次（禁止分多轮询问）
- [ ] Read 已提取参数用 `default` 预填
- [ ] Read 含时间字段时有时间表达式解析规则
- [ ] Delete 无全局 default
- [ ] Harness 表完整
- [ ] 有边界条件段
- [ ] 有 Pitfalls + Checklist
- [ ] 增/改/删 Phase 1 有"必须输出 hitl 块"铁律
- [ ] 增/改/删 Phase 1 有"已提取字段不重复问"规则
- [ ] 枚举参数有"精确值直接使用"规则
- [ ] 增/改/删 CP-1 用 `input` + `fields[].type: "choice"` 而非独立 `choice`（前端兼容性）
- [ ] 查询场景用 `text_input` / `combobox` / `number_range` / `datetime_range`，禁止 `input`
- [ ] HITL 块用 ```` ```hitl ```` 包裹，禁止裸 JSON
- [ ] 后端 max_tokens ≥ 2048
- [ ] system prompt 注入当前日期
- [ ] [批量 Update] dry_run → CP2a → Step 2 三步流 + expected_count + 条件一致
- [ ] [批量 Update] name/price 表达式对照表 + Phase 1 业务不变式预检 + 铁律 B0
- [ ] [批量 Delete] 支持单条/批量双路径；批量路径有 `dry_run → CP2a → CP2b`
- [ ] [批量 Delete] 正式删除带 `expected_count`，且 Step1/Step2 filters 一致
- [ ] 运行时已明确要求用 ` ```hitl ` / ` ```apicall `，没有把 ` ```text ` 当成运行时格式
- [ ] consumer 已说明结构化回执归一化（如 `{"action":"approve","expected_count":N}`）
- [ ] consumer 已说明“内部协议回执 vs 自然语言用户消息”分流
- [ ] 写操作的错误响应契约已声明（错误码 + detail shape）
- [ ] 前端 error 渲染兜底（string vs object）+ 硬护栏模式已说明
