---
name: json-hitl-skill-generator
description: >-
  生成使用 HITL (Human-in-the-Loop) JSON 块协议的 SKILL.md。支持单 skill 生成
  和 CRUD 批量生成（一次输入任务类型 → 输出增删改查 4 个 skill）。
  关键词：HITL、skill 生成、CRUD 批量生成、checkpoint 协议、人机协作、元技能
version: 1.2.0
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
| **CRUD 单个模式** | 1 个 SKILL.md | 同一任务类型下只生成指定操作（如只生成 order-query） |
| **CRUD 批量模式** | 4 个 SKILL.md（增-查-改-删） | 同一任务类型的全部操作场景 |

生成的 skill 拥有以下特征：

- 遵循 **Harness 6 维度** 架构设计
- 所有人机交互点通过 ````hitl` JSON 代码块表达，而非自由文本
- JSON 块是**可解析的**——可被前端工具消费，用于自动渲染 UI 或路由决策
- CRUD 批量模式走 Day 1 原型 → 确认 → Day 2 批量交付；CRUD 单个模式直接生成 1 个

---

## HITL JSON 块协议

本 skill 定义 ````hitl` JSON 代码块的协议格式。它生成的所有 SKILL.md 在运行时使用该协议与终端用户交互——生成器本身与 Jane 用自然语言沟通。协议是 skill 的核心定义。

### 块结构

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
          {"value": "approve", "label": "✅ 批准", "desc": "进入生成阶段"},
          {"value": "modify", "label": "✏️ 修改", "desc": "用户提出调整"},
          {"value": "reject", "label": "❌ 拒绝", "desc": "重新规划"}
        ]
      }
    ]
  }
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `version` | string | 是 | 协议版本号，当前为 `"1.0"` |
| `checkpoint.id` | string | 是 | 全局唯一标识，如 `cp-1` |
| `checkpoint.name` | string | 是 | 人类可读的名称，如 `"Plan 确认"` |
| `checkpoint.phase` | string | 是 | 所属阶段名称，如 `"Phase 2"` |
| `checkpoint.summary` | string | 是 | 一句话说明决策原因 |
| `checkpoint.action` | string | 是 | `"wait"`（等用户）或 `"notify"`（仅通知） |
| `checkpoint.decisions` | array | 是 | 决策项数组，至少 1 项 |

### Decision 对象

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 决策标识，如 `d-1` |
| `type` | string | 是 | `choice` / `confirm` / `review` / `input` |
| `question` | string | 是 | 向用户展示的问题 |
| `options` | array | choice 时 | 选项数组（≤ 5 项） |
| `fields` | array | input 时 | 字段数组（≤ 5 项） |
| `default` | string | 否 | 默认选项的 `value`，agent 推荐但不会自动执行 |

### 交互类型

| 类型 | 用途 | 条件 |
|------|------|------|
| `choice` | 多选一决策 | 选项已知且有限（≤ 5） |
| `confirm` | 二元确认/拒绝 | 只需"是/否"回答 |
| `review` | 审阅产物 | 有 artifact 需人工判断 |
| `input` | 收集结构化信息 | agent 无法自行获取 |

> 完整协议规范（含校验规则 14 条）见 `references/hitl-protocol.md`。
> 常见块模板（含 CRUD 专用模板）见 `references/block-templates.md`。

---

## 边界

- **进入条件**：用户明确要生成/创建一个新的 SKILL.md
- **特别检查**：如果用户说"增删改查"、"多种操作场景"，优先进入 CRUD 模式
- **不处理**：修改现有 skill（待建）、生成非 skill 文档（走 `scribe`）、生成代码实现

---

## Harness 视角

| 维度 | 本 Skill 的设计 | 实现手段 |
|------|----------------|---------|
| **CONTEXT** | 渐进加载 | 标准模式只看边界 + 协议；CRUD 模式额外加载 `references/crud-pattern.md` |
| **TOOLS** | 输入输出 | 用户需求描述 / 文件路径读取；输出为 1 个或 4 个 SKILL.md |
| **ORCHESTRATION** | Phase 管道 | 6 Phase 线性管道 + 3 个 Checkpoint（CRUD 模式拆分 Day1/Day2） |
| **MEMORY** | 文件系统 | `.json-hitl-gen/` 工作区：按模式划分子目录 |
| **EVALUATION** | 分层质检 | 内联自查 + 结构化 HITL JSON 校验 |
| **RECOVERY** | 约束恢复 | Checkpoint 确认制 + 最小切片修复 + 不覆盖现有文件 |

---

## 各阶段文件读取指南

| 阶段 | 必读 | 按需查 |
|------|------|--------|
| Phase 0 Intake | 本 SKILL.md（边界 + 工作流） | — |
| Phase 1 Analyze | `references/hitl-protocol.md`（仅 CRUD 模式加 `references/crud-pattern.md`） | — |
| Phase 2 Plan | `references/block-templates.md` | `references/config-reference.md` |
| Phase 3 Generate | `templates/generated-skill-template.md` | `references/block-templates.md` |
| Phase 4 Review | `references/hitl-protocol.md`（校验规则段） | — |
| Phase 5 Revise | — | — |
| Phase 6 Deliver | — | — |

---

**三种生成模式的工作流：**

### 标准模式

```
Phase 0  Intake         判断模式 + 收集需求
   ▼
Phase 1  Analyze        分析 HITL 触点 → 设计 JSON 块结构
   ▼
Phase 2  Plan           输出规划到 .json-hitl-gen/plan.md
         ★ Checkpoint 1 — 必须停
   ▼
Phase 3  Generate       生成 1 个 SKILL.md
         └ 内联 checklist 自查
   ▼
Phase 4  Review         质检：HITL JSON 合法性校验 + 结构审查
         ★ Checkpoint 2 — 必须停
   ▼
Phase 5  Revise         最小切片修复
   ▼
Phase 6  Deliver        写入目标路径
         ★ Checkpoint 3 — 最终确认
```

### CRUD 单个模式

```
Phase 0  Intake         判断模式 + 收集任务类型 + 指定操作
   ▼
Phase 1  Analyze        设计 HITL 触点（单操作分析）
   ▼
Phase 2  Plan           输出规划
         ★ Checkpoint 1 — 必须停
   ▼
Phase 3  Generate       直接生成 1 个 SKILL.md
         └ 内联 checklist 自查（含 CRUD 规则）
   ▼
Phase 4  Review         质检
         ★ Checkpoint 2 — 必须停
   ▼
Phase 5  Revise         修复
   ▼
Phase 6  Deliver        写入
         ★ Checkpoint 3 — 最终确认
```

### CRUD 批量模式

```
Phase 0  Intake         判断模式 + 收集任务类型信息
   ▼
Phase 1  Analyze        设计 HITL 触点（含 CRUD 差异分析）
   ▼
Phase 2  Plan           输出规划，含 4 个 skill 的 Outline
         ★ Checkpoint 1 — 必须停
   ▼
Phase 3  Generate (Day 1)
         生成第一个 skill 作为原型（建议先做 Read 或 Create）
         └ 内联自查
   ▼
Phase 4  Review (Day 1) 用户审阅原型
         ★ Checkpoint 2 — 必须停
   ▼
Phase 5  Generate (Batch)
         批量生成剩余 3 个 skill，一次交付
         └ 最终内联自查
   ▼
Phase 6  Deliver        写入目标路径（4 个文件）
         ★ Checkpoint 3 — 最终确认
```

---

## Phase 0 — Intake

### 判断工作模式

| 信号 | 模式 |
|------|------|
| "生成一个 skill"、"写个 skill"、"我要一个 X skill" | 标准模式 |
| "我的项目有订单管理"、"帮我生成订单的增删改查"、"任务管理 CRUD" | **先问清楚：批量还是单个？** |
| "增删改查"、"CRUD"、"全部操作"、"增删改查都要" | **CRUD 批量模式** |
| "只先做个查询"、"帮我的订单生成查询 skill"、"只生成删除" | **CRUD 单个模式**（指定操作） |
| "修改"、"改一下"、"优化"、"加个 HITL 点" | 修改模式（待建） |

> **CRUD 模式处理流程：** 当发现用户说的是任务类型相关操作时，先问清楚：是做全部 4 个操作（批量）还是只要 1 个操作（单个）？不要默认全做。

### CRUD 模式：收集需求

**第一轮（必问）：**
1. **任务类型名称**：这个任务是做什么的？（如"订单查询"、"用户管理"）
2. **任务类型英文标识**：用于 skill 命名（如 `order`、`user`）
3. **CRUD 操作说明**：每种操作具体做什么？
   - Create：创建什么？需要什么参数？
   - Read/Query：查询条件？支持哪些筛选？
   - Update：哪些字段可改？
   - Delete：什么条件下能删？
4. **查询参数的详细结构**：哪些参数 LLM 可能不确定（这是 HITL 触发的关键点）

**第二轮（按需追问）：**
5. 确认强度、输出形式、目标 agent

### 标准模式：收集需求

1. **Skill 用途**：一句话描述
2. **触发条件**：用户说什么时加载
3. **关键阶段**：哪些需要人类介入
4. 输出形式、输入来源、质检要求、目标 agent

> 追问原则：用户能回答就细化，说不清就用默认推荐。

---

## Phase 1 — Analyze

### 标准模式分析

从用户描述中提取 HITL 触点，设计 JSON 块结构。

通用 HITL 触点模板（按 skill 复杂度调整）：

| 触点 | 阶段 | 类型 | 说明 |
|------|------|------|------|
| CP-1 | Plan 之后 | `choice` | 确认规划范围和大纲 |
| CP-2 | Draft 之后 | `review` | 审阅初稿质量 |
| CP-3 | 交付前 | `confirm` | 确认最终交付 |

### CRUD 模式分析

加载 `references/crud-pattern.md`。关键设计决策：

1. **4 个 skill 的公共结构**：触发条件、工作区命名、HITL 协议定义段
2. **CRUD 差异矩阵**（详见 `references/config-reference.md`）：
   - Create/Update/Delete：需要最终确认
   - Read/Query：不需要最终确认
3. **HITL 触点差异**：所有 4 个 skill 在参数收集阶段都有 HITL；增改删多一个最终确认 CP

### 决策收集规则（适用于所有生成 skill 的铁律）

- 每个 ````hitl` 块的 `decisions` 至少有 1 个条目
- Agent **可以推荐**默认选项（用 `default` 字段标记），但**不能静默替用户执行**
- 收到用户响应后，agent 必须按用户选择行动
- 如果用户的选择超出 options 范围，agent 必须适配而非拒绝
- **Delete 操作的最终确认，不得设置全局 `default` 字段**

---

## Phase 2 — Plan

输出 `.json-hitl-gen/plan.md`。

### 标准模式规划内容

1. **Skill 概述**：名称、描述、触发条件
2. **Harness 6 维度设计表**
3. **HITL 触点地图**：位置、类型、JSON 块示例
4. **Outline**：章节列表
5. **特殊注意事项**

### CRUD 模式规划内容

1. **任务类型概述**：名称、英文标识、业务描述
2. **4 个 skill 清单**：命名方案、触发条件
3. **CRUD 差异表**
4. **HITL 触点地图 × 4**
5. **原型选择**：推荐从 Read（最简单）或 Create（最常见）开始

### Plan 自查清单

1. **模式识别正确？** 有没有遗漏操作类型？
2. **CRUD 差异处理正确？** 增改删有最终确认吗？Read 没有吗？
3. **HITL 触点完整但不冗余？**
4. **JSON 块格式正确？** 符合 `references/hitl-protocol.md`。
5. **用户能看懂？** CRUD 模式下能一眼看出 4 个 skill 的差异。

### ★ Checkpoint 1 — 必须停

向用户展示规划方案的核心内容（Outline + HITL 触点数 + 差异对比），等待用户回复。

**CRUD 模式展示策略：** 展示整体方案 + 4 个 skill 的差异对比表。不要一次展示 4 个完整 Outline。

> CRUD 模式话术示例："共 4 个 skill：order-create / order-query / order-update / order-delete。增改删有最终确认，查没有。建议先做 order-query 原型。请确认以上方案是否合理？"

---

## Phase 3 — Generate

### 标准模式

基于 `plan.md` + `templates/generated-skill-template.md` 生成 1 个 SKILL.md。

### CRUD 模式——分两个子阶段

#### 子阶段 A：原型生成（Day 1）

1. 按 CP-1 确认的首选操作生成第一个 skill
2. 加载 `templates/generated-skill-template.md`
3. 按用户信息填充 CRUD 特定内容
4. 命名规则：`<task-type>-<operation>`

#### 子阶段 B：批量生成（Day 2）

仅当 CP-2 确认原型后：
1. 基于原型骨架，生成剩余 3 个 skill
2. 差异点调整：
   - **Read/Query**：去掉最终确认 CP，增加结果展示阶段
   - **Create**：增加参数预览 + 最终确认 CP
   - **Update**：增加修改预览 + 最终确认 CP
   - **Delete**：增加删除对象确认 + 最终确认 CP（无全局 default）
3. 一次写入 `.json-hitl-gen/` 下

### 强制包含的内容

每个生成的 skill 必须包含以下部分：

- **HITL 协议定义段**：说明 ````hitl` JSON 块格式，指向 `references/hitl-protocol.md`
- **决策收集铁律**：每个 Checkpoint 处写明
  - Delete 操作额外铁律：最终确认不得包含全局 `default`
- **Harness 架构表**：6 维度

### 生成质量硬性规则（防止返工的关键）

以下规则是**必须遵守**的，不是建议。违反这些规则是导致每次生成后需要返工的根本原因。

#### 规则 1：HITL 协议示例块的 fence 风格

**✅ 正确做法：** 用 ````text` 展示协议示例，不使用嵌套 fence。

```text
```text
{
  "version": "1.0",
  "checkpoint": {
    ...
  }
}
```
```

**❌ 错误做法 1：** 使用 ````markdown` 包裹 ````hitl` 嵌套
```text
```markdown
```hitl        ← 错误！嵌套 fence 导致解析器误匹配
{...}
```
```
```

**❌ 错误做法 2：** 示例中存在 `"..."`、`[...]`、`<Decision>` 等非法占位符

→ 示例块中所有字段值必须是**合法的 JSON 值**（字符串、数字、对象、数组），不能出现 `...` 等非法记号。

#### 规则 2：对话示例不能混合 fence

**✅ 正确做法：** 对话中的 ````hitl` 块用独立代码块展示，外面用块引用 `>` 包裹整个对话。

```text
> 用户：帮我查查
> Agent：输出 hitl 块
> ```text
> {"checkpoint":{...}}
> ```
```

**❌ 错误做法：** 在 ```` 围栏内嵌入 ````hitl`/````json` 块

```text
```
用户：帮我查查
Agent：```hitl       ← 错误！混合 fence 导致匹配错位
{...}
```
```
```

#### 规则 3：JSON 块内禁止非法占位符

所有 ````hitl` 和 ````json` 块中的内容必须是**合法可解析的 JSON**。

合法示例（占位内容用合规 JSON 值填充）：`{"id": "cp-n", "name": "示例名称"}`

非法示例：`{"id": "...", "name": "..."}` / `{"options": [...]}` / `{"fields": [<Field>, ...]}`

### Draft 自查清单

1. **所有 JSON 块都是合法 JSON？** 逐个检查，没有 `...`、`[...]`、`<...>` 等非法记号。
2. **HITL 协议示例块没有嵌套 fence？** 用 ````text` 而非 ````markdown`+````hitl`。
3. **对话示例没有混合 fence？** 所有 ```hitl/```json 块要么是独立代码块，要么在块引用内。
4. **CRUD 差异正确？** Read 没有 final CP，Delete 的 final CP 没有全局 default。
5. **决策收集铁律已写入每个 CP？**
6. **任务类型特定参数完整？**

---

## Phase 4 — Review

### HITL JSON 合法性校验

对所有生成的 SKILL.md 中的 ````hitl` 块校验（校验规则见 `references/hitl-protocol.md` 校验规则段）：

- [ ] JSON 可解析
- [ ] 含 `version`（`"1.0"`）、`checkpoint.id`、`name`、`phase`、`summary`
- [ ] `action` 为 `"wait"` 或 `"notify"`
- [ ] `decisions` 为非空数组
- [ ] 每个 decision 有 `id`、`type`、`question`
- [ ] `type` 合法（choice / confirm / review / input）
- [ ] choice 类型包含 `options`
- [ ] 每个 option 有 `value`、`label`
- [ ] **CRUD 专项**：增改删有 final CP，Read 没有
- [ ] **CRUD 专项**：Delete 的 final CP 没有全局 `default`

### 结构审查

- [ ] Frontmatter 合法
- [ ] Harness 6 维度表完整
- [ ] 有工作流总览图
- [ ] 有边界条件
- [ ] 有 Common Pitfalls
- [ ] 有 Verification Checklist
- [ ] CRUD 模式：4 个 skill 命名一致（相同前缀）

### ★ Checkpoint 2 — 必须停

**标准模式：** 向用户展示校验结果（JSON 块数量 + 通过数 + issues），请用户审阅生成的 SKILL.md。

话术示例："校验完成：3 个 ````hitl` 块全部合法。结构审查通过。请审阅生成的 skill 内容。"

**CRUD 模式（原型确认）：** 展示原型路径 + 校验结论 + 询问是否按此模板继续。

---

## Phase 5 — Revise

**最小切片修复，禁止整篇重写。**

- 每个 issue 单独定位修复
- CRUD 模式：问题影响所有 4 个 skill 时一并修复
- 修改直接在 `.json-hitl-gen/` 下的文件上做
- 有修复则追加 `.json-hitl-gen/review.md`

---

## Phase 6 — Deliver

### ★ Checkpoint 3 — 最终确认

向用户展示目标路径列表，确认写入。自然语言询问：

- 确认将生成的 skill(s) 写入目标路径？
- 目标目录是哪里？
- 如果路径已有文件，询问覆盖还是备份

**写入规则：**
1. 目标路径已有文件 → 询问覆盖还是备份
2. CRUD 模式：逐个检查 4 个路径，已存在的列表告知用户
3. 写入后建议创建 symlink 注册到 Hermes

```bash
for name in <task-type>-create <task-type>-query <task-type>-update <task-type>-delete; do
  mkdir -p ~/.hermes/skills/software-development/$name
  ln -sf /root/code/skill/$name/SKILL.md ~/.hermes/skills/software-development/$name/SKILL.md
done
```

---

## 常见 Pitfalls

1. **JSON 块中使用非法占位符**：`"..."`、`[...]`、`<Decision>` 等不是合法 JSON。所有示例块中必须用合规 JSON 值填充。

2. **HITL 协议示例用嵌套 fence**：````markdown` 包裹 ````hitl` 会导致 Markdown 解析器误匹配。必须用 ````text` 展示协议示例。

3. **对话示例混合 fence**：```` 围栏内嵌入 ````hitl` 块 → 外层围栏被内层 ```` 意外关闭。改用块引用 `>` + 独立 ````hitl` 块。

4. **没有引导语就扔出 JSON 块**：必须附带自然语言说明。

5. **Checkpoint 过多**：超过 4 个 CP 会让用户烦躁。Read 只有 2 个 CP。

6. **CRUD 模式：把 Read 也加了 final confirm**：查是安全的，不需要。

7. **CRUD 模式：Delete 的 final confirm 设了全局 default**：禁止，必须让用户主动选择。

8. **决策选项语义模糊**：`"你觉得怎么样？"` 不是有效选项。

9. **忘记生成决策收集铁律**：每个 CP 处必须写明。

10. **生成的 skill 没有边界条件**：必须包含判断段。

11. **CRUD 命名不一致**：必须统一 `<task-type>-<operation>`。

12. **CRUD 批量模式跳过原型环节**：必须走 Day 1 原型 → 确认 → Day 2 批量。

---

## references/ 目录索引

| 文件 | 用途 |
|------|------|
| `references/hitl-protocol.md` | HITL JSON 块协议完整规范（格式定义、字段说明、校验规则） |
| `references/block-templates.md` | 常见 ````hitl` 块模板（Choice / Confirm / Review / Input + CRUD 专用） |
| `references/config-reference.md` | 生成配置参考（CP 数推荐、CRUD 差异矩阵、命名规则） |
| `references/crud-pattern.md` | CRUD 批量生成模式参考（识别信号、需求收集、生成流程、交付路径） |

## Verification Checklist

- [ ] 模式判断正确（标准 / CRUD 单个 / CRUD 批量）
- [ ] HITL JSON 块全部合法（语法 + 结构校验通过，无 `...`/`[...]` 占位符）
- [ ] HITL 协议示例块使用 ````text` 而非嵌套 fence
- [ ] 对话示例没有混合 fence
- [ ] HITL JSON 块全部合法（语法 + 结构校验通过）
- [ ] Frontmatter 合法（name + description ≤ 1024 chars）
- [ ] 生成的 SKILL.md 包含 HITL 协议定义段
- [ ] 每个 Checkpoint 附带了决策收集铁律
- [ ] CRUD 模式：Read 没有最终确认 CP
- [ ] CRUD 模式：Delete 的最终确认 CP 没有全局 `default`
- [ ] CRUD 模式：4 个 skill 命名统一前缀
- [ ] Harness 6 维度表已填写
- [ ] 有边界条件段
- [ ] 包含 Common Pitfalls + Verification Checklist
- [ ] ````hitl` 围栏语法正确
- [ ] 已写入目标路径（不覆盖已存在文件，除非用户确认）
