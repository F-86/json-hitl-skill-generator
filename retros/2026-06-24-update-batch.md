# 回顾：2026-06-24 product-update 批量改名 409 防护调试记录

> 本文件记录当天对 `product-update` skill 的批量改名场景调试全部修改方向，用于反哺 `json-hitl-skill-generator`。
> 与 `2026-06-24-query.md`（product-query 会话）互补，那次聚焦 Read，这次聚焦 Update/Batch。

## 今天的 Git 提交

| Commit | 说明 |
|--------|------|
| `75b07f2` | feat: 批量改名 409 防护——后端拦截 + 前端硬护栏 + skill 加 B0 铁律 |

提交涉及 5 个文件，762 insertions(+), 93 deletions(-)：
- `backend/main.py`（+332）
- `frontend/src/components/AIChat.js`（+1）
- `frontend/src/components/ApiCallResult.js`（+97）
- `frontend/src/components/HITLWidget.js`（+15）
- `skills/product-update/SKILL.md`（+410）

> 注：本会话实际只触发了 SKILL.md 两处护栏文案、ApiCallResult.js 的 ErrorBlock、AIChat.js 传 prevFailed、HITLWidget.js 过滤危险按钮；backend/main.py 与 SKILL.md 主体（批量分支、dry_run 流程、铁律 B1/B2/B3）是会话前已有的累积改动，一并随提交入库。

---

## 修改方向分类

### 方向 1：批量 Update 的 dry_run → 确认 → 正式执行两步流

**背景：** 原 Update skill 只有单条 `PUT /api/products/{id}`。批量改名/改价是高频场景，需要先预览影响范围再执行。

**最终流程（写入 SKILL.md）：**
1. **Step 1 dry_run 预览**：`POST /api/products/bulk_update` 带 `dry_run: true`，后端返回 `{matched, items, preview:[{id, before, after}]}`，前端 UI 展示匹配数和 before/after 对比
2. **CP2a 批量预览确认**：紧跟 dry_run apicall **同一回复**输出 hitl choice 块（approve/refine/cancel），让用户看完 UI 列表后决策
3. **Step 2 正式执行**：用户选 approve 后，输出 `dry_run: false` 的 apicall，**必须带 `expected_count`**（取自 dry_run 的 matched）

**铁律 B1/B2/B3：**
- B1：批量正式执行必须带 `expected_count`（防误改双保险）
- B2：Step 1 和 Step 2 的 `filters` 和 `update` 必须**完全一致**，不能偷偷改条件
- B3：`expected_count` 取自 dry_run 响应中的 matched

**核心教训：**
- LLM 看不到 apicall 结果，靠**前端 UI** 展示 dry_run 结果给用户，LLM 只负责输出 apicall + 后续 hitl
- `expected_count` 是并发安全双保险：若 dry_run 后数据被并发改动导致匹配数变化，后端用 `count_mismatch` 409 拒绝

### 方向 2：name / price 表达式（批量场景）

**背景：** 批量改 name 不能直接 set 同一字面值（N 条同名），需要表达式：
- `name={"suffix":"·统一饮料"}` 后缀
- `name={"prefix":"新-"}` 前缀
- `name={"replace":["旧","新"]}` 替换
- `price={"multiply":1.1}` 涨价 10%（注意不是 1.0 也不是 0.1）
- `price={"add":-0.5}` 降 0.5 元

**核心教训：** price 表达式方向容易写反（涨价是 multiply 1.1 不是 1.0），必须在 SKILL.md 用对照表明确。

### 方向 3：后端结构化 409 错误 + 前端渲染崩溃

**问题：** 后端用 `HTTPException(detail={...})` 返回结构化错误对象：
- `name_collision_in_batch` → `{error, message, duplicates:[...]}`
- `name_collision_with_existing` → `{error, message, conflicts:[{id, name}]}`
- `count_mismatch` → `{error, message, matched, expected}`

前端 `AIChat.js` 把 `data.detail`（对象）塞进 `result.error`，`ApiCallResult.js` 原来直接 `{result.error}` 渲染对象 → **React crash：Objects are not valid as a React child**。

**修复（ApiCallResult.js）：** 新增 `ErrorBlock` 组件：
- `typeof error === 'string'` → 直接显示
- `typeof error === 'object'` → 展示 `message` + `code`，并针对字段渲染：
  - `duplicates` → 批内重复名称
  - `conflicts` → 与库内冲突（带 `#id name` 形式）
  - `matched/expected` → 匹配数 vs 预期数
- 结尾追加 **"⚠️ 操作已中止，数据未变更。请调整参数后重试。"**

### 方向 4：跨层字段名契约漂移

**问题：** 前端 ErrorBlock 第一版用了 `existing_names` / `actual`，但后端实际返回的是 `conflicts` / `matched`。前后端字段名不一致导致冲突信息渲染不出来。

**修复：** 对齐字段名为后端实际返回值（`conflicts`、`matched`、`expected`）。

**教训：** 错误响应的字段名是前后端契约的一部分，但 SKILL.md 里通常只描述成功路径，**没有声明错误响应的 shape**，导致前端实现时只能靠猜。

### 方向 5：软约束（SKILL.md 铁律）不可靠 → 前端硬护栏

**这是本次最核心的架构教训。**

**现象：** SKILL.md 已写入"铁律 B0：dry_run 失败时禁止继续输出 CP2a 与 Step 2"，但 LLM **仍然违规**——dry_run 返回 409 后，LLM 还说"已确认执行批量改名操作"并输出了 CP2a 的"全部执行"按钮。用户若点了"全部执行"就会触发 Step 2 再次 409。

**根因：** Prompt 层约束是**软约束**，无法 100% 保证 LLM 行为。LLM 会忽略或误判铁律。

**修复（前端硬护栏，AIChat.js + HITLWidget.js）：**
- `AIChat.js` 给 `HITLWidget` 传 `prevFailed={!!(msg.apicall && msg.apicallResult && msg.apicallResult.error !== undefined)}`
- `HITLWidget.js` 接收 `prevFailed`，当为 true 时：
  - choice 类决策过滤掉 `approve` / `confirm` / `execute` 这些会触发后续 apicall 的"危险按钮"
  - confirm 类型直接隐藏"确认"按钮
  - 仅保留 `refine` / `modify` / `cancel` 安全选项
  - 底部追加红色提示："⚠️ 上一步操作失败，已隐藏执行类按钮。请选择\"调整条件\"重新提需求，或取消。"

**教训：** Prompt 约束 + 前端硬护栏是**互补的两层防线**。不能只靠 SKILL.md 写铁律，必须在运行时（前端/HITL 消费层）对"前置 apicall 失败"做硬性拦截。

### 方向 6：Phase 1 业务不变式预检（不等后端报错才反应）

**问题：** 用户说"所有饮料改名为统一饮料"，"统一饮料"是固定字面值，N 条商品改成同一名字必然批内重名。LLM 却盲发 dry_run → 看到 409 → 还继续往下推 Step 2。

**修复（SKILL.md Phase 1 新增识别规则）：** 识别"所有 X 改名为 Y（Y 是固定字面值）"模式 → 必须**先 HITL 询问澄清**，建议改用 `{"suffix":"·Y"}` / `{"replace":["X","Y"]}` 等表达式，或走单条路径。**不要先盲发 dry_run 看后端报错才反应。**

**教训：** 后端 409 是最后一道防线，但 LLM 应在 Phase 1 就识别明显的业务不变式违反（批量同名、唯一约束冲突等），提前 HITL 澄清，减少无效请求和糟糕体验。

### 方向 7：dry_run 本身也可能失败（前置 apicall 阻断）

**问题：** 之前隐含假设 dry_run 总是成功（只是预览）。实际上 dry_run 也可能 4xx/409 失败（批内重名、与库内冲突、0 条命中）。

**修复（SKILL.md 铁律 B0）：** 明确 dry_run 失败时：
- 前端会在 UI 上显式展示红色错误卡片
- **绝对不要**继续输出 CP2a 的"批量预览确认" hitl 块，也不要输出 Step 2 的正式 apicall
- 正确做法：在 dry_run apicall 之后输出一条**询问类 hitl**（input 块），告知报错并请用户调整 name 表达式 / filters / 改走单条路径

**教训：** 任何"预览/前置" apicall 都是可能失败的步骤，失败时必须阻断后续确认和执行步骤。这是多步流程的状态机完整性问题。

---

## 与 json-hitl-skill-generator 的关联

上述 7 个方向暴露了生成器在以下方面的不足（详见 `2026-06-24-update-batch-improvements.md`）：

1. **批量 Update 模板缺失**（方向 1、2）：Update 仍是单条 PUT，无 dry_run→确认→执行两步流、无表达式、无 expected_count
2. **错误响应契约缺失**（方向 3、4）：生成器不指导生成的 skill 声明错误响应 shape，也无前端 ErrorBlock 渲染指南
3. **软约束 vs 硬护栏认知缺失**（方向 5）：consumer pattern 没有"前置 apicall 失败时屏蔽危险按钮"的硬护栏模式
4. **Phase 1 业务不变式预检缺失**（方向 6）：模板没有"识别业务约束违反并提前 HITL 澄清"的段落
5. **前置 apicall 失败阻断缺失**（方向 7）：多步流程没有"中间步骤失败必须 halt"的状态机规则
