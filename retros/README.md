# Retros — 调试经验回顾与改进记录

本文件夹存放对 `json-hitl-skill-generator` 生成 skill 的调试经验回顾，用于反哺 generator 本身。

## 命名规范

文件按 `YYYY-MM-DD-<topic>.md` 格式命名，同一会话的回顾与改进清单配对：

| 文件 | 类型 | 来源会话 |
|------|------|---------|
| `2026-06-24-query.md` | 回顾 | product-query 调试（Read/Query 工作流演进） |
| `2026-06-24-query-improvements.md` | 改进清单 | 同上——暴露的 generator 不足 |
| `2026-06-24-update-batch.md` | 回顾 | product-update 批量改名 409 防护 |
| `2026-06-24-update-batch-improvements.md` | 改进清单 | 同上——暴露的 generator 不足 |
| `2026-06-24-delete-batch-hitl-ui.md` | 回顾 | product-delete 批量删除与 HITL 消费语义调试 |
| `2026-06-24-delete-batch-hitl-ui-improvements.md` | 改进清单 | 同上——暴露的 generator 不足 |
| `2026-06-24-delete-batch-hitl-ui-implementation-log.md` | 落地日志 | 同上——修改 skill 时的同步记录 |

- **回顾文件**（`-<topic>.md`）：记录当天 git 提交与对话涉及的修改方向分类
- **改进清单**（`-<topic>-improvements.md`）：对照 generator 当前内容，列出具体不足与修改项
- **落地日志**（`-<topic>-implementation-log.md`）：记录本轮已落地到哪些文件、改了什么、是否已完成检查

## 回顾 → 改进 → 落地流程

1. 在真实项目中调试生成的 skill（如 demo-ai-crud 的 product-* 系列）
2. 调试完成后，在本文件夹新增回顾文件 + 改进清单（配对）
3. 将改进项落地到 `SKILL.md` / `references/` / `templates/`，并在 `CHANGELOG.md` 记录版本
4. 已落地的改进项保留在清单中作为追溯，不删除
