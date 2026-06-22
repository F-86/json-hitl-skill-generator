# JSON HITL Skill Generator

> 生成使用 HITL (Human-in-the-Loop) JSON 块协议的 SKILL.md。支持单 skill 生成和 CRUD 批量生成。

## Overview

这是一个 **元技能（meta-skill）**—— 输出是一个完整的、可直接运行的 SKILL.md。它的核心能力：

- **定义 ````hitl` JSON 块协议**：一种结构化的人机交互协议，agent 用可解析的 JSON 块与终端用户交互
- **生成场景 skill**：根据用户需求描述，自动生成包含 HITL 协议的 SKILL.md 文件
- **CRUD 批量生成**：一次输入任务类型 → 输出增删改查 4 个 skill（Create 有最终确认、Query 无最终确认、Update/Delete 有最终确认）

### 工作流程

```
用户需求 → Intake → Analyze → Plan → Generate → Review → Revise → Deliver
                                    ↓
                             ★ 3 个 Checkpoint
                            （自然语言确认，非 JSON 块）
```

## 生成产物特征

生成的 SKILL.md 拥有以下特征：

- 遵循 **Harness 6 维度** 架构设计
- 所有人机交互点通过 ````hitl` JSON 代码块表达，而非自由文本
- JSON 块是**可解析的**——可被前端工具消费，用于自动渲染 UI 或路由决策
- CRUD 模式走 Day 1 原型 → 确认 → Day 2 批量交付

## HITL JSON 块协议

生成的 skill 在运行时向终端用户输出以下格式的块：

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-1",
    "name": "确认",
    "phase": "Phase 2",
    "summary": "需要用户决策的事项",
    "action": "wait",
    "decisions": [...]
  }
}
```

支持四种交互类型：`choice`（多选一）、`confirm`（确认/拒绝）、`review`（审阅）、`input`（信息收集）。

> 完整协议规范见 [`references/hitl-protocol.md`](references/hitl-protocol.md)

## 项目结构

```
json-hitl-skill-generator/
├── SKILL.md                        — 主工作流（本 skill 的数据）
├── references/
│   ├── hitl-protocol.md            — HITL JSON 块协议规范（格式定义 + 校验规则）
│   ├── block-templates.md          — 常见块模板（Choice/Confirm/Review/Input + CRUD 专用）
│   ├── config-reference.md         — 生成配置参考（CP 数推荐、CRUD 差异矩阵）
│   └── crud-pattern.md             — CRUD 批量生成模式参考
└── templates/
    └── generated-skill-template.md — 生成模板骨架
```

## 使用方式

### 作为 Hermes Skill 加载

```bash
skill_view(name='json-hitl-skill-generator')
```

### 标准模式：生成单个 skill

```
用户：写一个 code review skill，每次审查前要确认范围
Agent：→ 走标准模式 → 输出 1 个 SKILL.md
```

### CRUD 模式：批量生成 4 个 skill

```
用户：我的项目需要订单管理的增删改查
Agent：→ 走 CRUD 模式 → 输出 4 个 SKILL.md
```

## 重要边界

- **生成器自己与用户（开发者）用自然语言沟通**——不输出 ````hitl` JSON 块
- **生成的 skill 运行时与终端用户用 ````hitl` JSON 块沟通**
- 不处理：修改现有 skill（待建）、生成非 skill 文档（走 `scribe`）、生成代码实现

## References 索引

| 文件 | 用途 |
|------|------|
| `references/hitl-protocol.md` | HITL JSON 块协议完整规范 |
| `references/block-templates.md` | 常见 ````hitl` 块模板 |
| `references/config-reference.md` | 生成配置参考 |
| `references/crud-pattern.md` | CRUD 批量生成模式参考 |

## License

MIT
