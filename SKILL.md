---
name: json-hitl-skill-generator
description: >-
  生成使用 HITL (Human-in-the-Loop) JSON 块协议的 SKILL.md。支持单 skill 生成
  和 CRUD 批量生成（一次输入任务类型 → 输出增删改查 4 个 skill）。
  关键词：HITL、skill 生成、CRUD 批量生成、checkpoint 协议、人机协作、元技能
version: 1.3.0
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
4. `data.apicall` → 前端自动执行 API 调用并渲染结果

### 控件映射

| decision.type | HITLWidget 渲染 | 说明 |
|--------------|----------------|------|
| choice | 选项按钮列表 | 点击触发操作 |
| confirm | 确认/取消按钮 | 二元决策 |
| input | 文本输入框 | 收集自由文本 |
| combobox | 标签选择器 | 从后端 API 拉取选项 |
| number_range | 最小/最大值输入 | 数字区间 |
| datetime_range | 日期选择器 | 使用 DateRangePicker |

### 关键约束

- ````hitl` 块必须附带自然语言引导语，不能裸扔 JSON
- LLM system prompt = SKILL.md 完整内容（含 Phase 指令、触发条件）
- 前端 API_BASE 不能硬编码 localhost，使用相对路径或环境变量
- history LIMIT 和 LLM context slice 的 N 值保持一致

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

## Phase 3 — Generate

### CRUD 模式——分两个子阶段

子阶段 A（Day 1）：生成原型 skill → 用户确认
子阶段 B（Day 2）：按差异矩阵生成剩余 skill

### 生成质量硬性规则

规则 1：HITL 协议示例块用 ```text，不用嵌套 fence
规则 2：对话示例用 > 块引用 + 独立 hitl 块
规则 3：JSON 块内禁止非法占位符

### Draft 自查清单

1. 所有 JSON 块合法？无 ... / [...] / <...>
2. 增/改/删 Phase 1 有"必须输出 hitl 块"铁律 + "已提取字段不重复问"规则？
3. 枚举参数有"精确值直接使用"规则？
4. 增/改/删 CP-1 用 `input` + `fields[].type: "choice"` 而非独立 `choice`？
5. ````apicall` 块用 `"<参数名>"` 而非 `{{param}}`？

## Phase 4 — Review

- JSON 合法性校验（14 条规则）
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

8. 没有引导语就扔出 JSON 块
9. Checkpoint 过多
10. CRUD 模式：把 Read 也加了 final confirm
11. CRUD 模式：Delete 设了全局 default
12. 决策选项语义模糊
13. 忘记决策收集铁律
14. 生成的 skill 没有边界条件

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
- [ ] Delete 无全局 default
- [ ] Harness 表完整
- [ ] 有边界条件段
- [ ] 有 Pitfalls + Checklist
- [ ] 增/改/删 Phase 1 有"必须输出 hitl 块"铁律
- [ ] 增/改/删 Phase 1 有"已提取字段不重复问"规则
- [ ] 枚举参数有"精确值直接使用"规则
- [ ] 增/改/删 CP-1 用 `input` + `fields[].type: "choice"` 而非独立 `choice`（前端兼容性）
