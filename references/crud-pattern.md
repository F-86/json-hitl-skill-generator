# CRUD 批量生成模式

本文件定义了 CRUD 批量生成时的模式识别和参考信息。

## 模式识别

| 信号 | 模式 |
|------|------|
| "生成一个 skill"、"写个 skill"、"我要一个 X skill" | 标准模式 |
| "增删改查"、"CRUD"、"多种操作"、"我的项目有查任务/增任务/改任务/删任务" | **CRUD 批量模式** |
| "修改"、"改一下"、"优化"、"加个 HITL 点" | 修改模式（待建） |

## CRUD 需求收集模板

**第一轮（必问）：**

1. **任务类型名称**：这个任务是做什么的？（如"订单查询"、"用户管理"、"数据导入"）
2. **任务类型英文标识**：用于 skill 命名（如 `order`、`user`、`task`）
3. **CRUD 操作说明**：每种操作具体做什么？
   - Create：创建什么？需要什么参数？
   - Read/Query：查询什么？支持哪些筛选条件？
   - Update：更新什么？哪些字段可改？
   - Delete：删除什么？什么条件下能删？
4. **查询参数的详细结构**：Read 操作需要哪些参数、哪些参数 LLM 可能不确定（这些是 HITL 触发的关键点）

**第二轮（按需追问）：**

5. **确认强度**：Create/Update/Delete 是否都需要最终确认？
6. **Read 是否需要 HITL**：还是只需收集参数、展示结果，不需要用户确认？
7. **输出形式**：文件？消息？网络请求？
8. **目标 agent**：Hermes / Claude Code / Codex / 通用？

## CRUD 生成流程

```
子阶段 A: 原型生成 (Day 1)
  1. 按确认的首选操作生成第一个 skill（推荐 Read 或 Create）
  2. 加载 generated-skill-template.md 作为起点
  3. 按用户提供的信息填充 CRUD 特定内容
  4. 命名: <task-type>-<operation>

子阶段 B: 批量生成 (Day 2) — 仅当原型确认后
  1. 基于原型骨架，生成剩余 3 个 skill
  2. 差异点调整:
     - Read/Query: 去掉最终确认 checkpoint，增加结果展示阶段
     - Create: 增加参数预览 + 最终确认 checkpoint
     - Update: 增加修改预览 + 最终确认 checkpoint
     - Delete: 增加删除对象确认 + 最终确认 checkpoint（无全局 default）
```

## CRUD 交付路径

| 文件 | 目标路径 |
|------|---------|
| Create | `<target>/<task-type>-create/SKILL.md` |
| Read/Query | `<target>/<task-type>-query/SKILL.md` |
| Update | `<target>/<task-type>-update/SKILL.md` |
| Delete | `<target>/<task-type>-delete/SKILL.md` |
