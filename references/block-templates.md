# HITL JSON Block Templates

常见 ````hitl` JSON 块模板，供生成 skill 时引用。

## Choice（多选一）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-n",
    "name": "标题",
    "phase": "阶段名",
    "summary": "总结描述",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "choice",
      "question": "问题",
      "options": [
        {"value": "a", "label": "✅ 选项A", "desc": "说明"},
        {"value": "b", "label": "✏️ 选项B", "desc": "说明"}
      ],
      "default": "a"
    }]
  }
}
```

**用途：** 多选一决策。适用于范围确认、参数选择、路线决策。

**选项限制：** options ≤ 5 个，太多用户选不动。

---

## Confirm（确认）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-n",
    "name": "确认",
    "phase": "阶段名",
    "summary": "需要确认的事项",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "confirm",
      "question": "是否确认？"
    }]
  }
}
```

**用途：** 简单确认/拒绝。适用于是否覆盖文件、是否执行危险操作。

**注意：** 问题必须能用"是/否"回答。

---

## Review（审阅）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-n",
    "name": "审阅",
    "phase": "阶段名",
    "summary": "请审阅以下产物",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "review",
      "question": "请审阅产物并提出修改意见",
      "artifact_path": ".json-hitl-gen/draft.md"
    }]
  }
}
```

**用途：** 审阅产物质量。必须附带 `artifact_path`。

---

## Input（信息收集）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-n",
    "name": "信息收集",
    "phase": "阶段名",
    "summary": "需要用户提供以下信息",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "input",
      "question": "请提供以下信息：",
      "fields": [
        {"name": "param_1", "type": "string", "label": "参数1", "required": true},
        {"name": "param_2", "type": "choice", "label": "参数2", "options": ["A", "B", "C"]}
      ]
    }]
  }
}
```

**用途：** 收集用户提供的结构化信息。

**限制：** fields ≤ 5 个。

---

## CRUD 专用模板

### Read：参数确认（无最终确认）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-1",
    "name": "查询参数确认",
    "phase": "参数收集",
    "summary": "已识别查询参数，请确认是否正确",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "choice",
      "question": "查询条件是否正确？",
      "options": [
        {"value": "execute", "label": "✅ 执行查询", "desc": "用当前参数查询"},
        {"value": "modify", "label": "✏️ 修改参数", "desc": "调整查询条件"},
        {"value": "cancel", "label": "❌ 取消", "desc": "放弃此次查询"}
      ]
    }]
  }
}
```

### Create/Update：最终确认

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-final",
    "name": "操作确认",
    "phase": "最终确认",
    "summary": "请确认以下操作将执行",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "confirm",
      "question": "确认执行该操作？"
    }]
  }
}
```

### Delete：最终确认（无 default）

```json
{
  "version": "1.0",
  "checkpoint": {
    "id": "cp-final",
    "name": "删除确认",
    "phase": "最终确认",
    "summary": "以下对象将被永久删除，不可恢复",
    "action": "wait",
    "decisions": [{
      "id": "d-1",
      "type": "choice",
      "question": "确认删除？",
      "options": [
        {"value": "confirm_delete", "label": "⚠️ 确认删除", "desc": "永久删除选定对象"},
        {"value": "cancel", "label": "❌ 取消", "desc": "不执行删除"}
      ],
      "default": "cancel"
    }]
  }
}
```

**铁律：** Delete 操作的最终确认 ````hitl` 块不得包含全局 `default` 字段（但 `options` 中可以标记 `"default": "cancel"` 让取消为默认）。
