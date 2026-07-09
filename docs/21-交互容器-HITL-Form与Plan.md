# HITL Form 与 Plan

本页拆出 HTML 表单交互与 CODER planning confirmation。Question 与 Approval 见 [HITL Question 与 Approval](20-交互容器-HITL-Question与Approval.md)。

## 1. Form

![08 AGW Form Sequence](../assets/diagrams/sequences/08-agw-seq-form.svg)

form 用于 HTML 表单型交互。它默认使用 `viewportType:"html"`，`viewportKey` 来自 HITL rule 或 frontend tool metadata，可配合 `GET /api/viewport?viewportKey=...` 获取 HTML payload。

事件顺序：

- `tool.start -> tool.args* -> tool.end -> awaiting.ask -> request.submit -> awaiting.answer -> tool.result`

关键点：

- `awaiting.ask.mode = "form"`
- `forms[]` 直接内联在 `awaiting.ask`
- form 默认 `viewportType:"html"`；有表单视图时保留 `viewportKey`
- `forms[]` item 主字段是 `id`、可选 `title`、可选 `toolName`、可选 `command`、可选初始 `form` 对象
- approve 必须提交 `decision:"approve"` + `form:{...}`
- reject 提交 `decision:"reject"`，可带 `reason` 和可选 `form`
- `params[]` 按下标对应 `forms[]`；`id` 只用于审计和日志，不用于分发
- `awaiting.answer.forms[]` 会归一化为 `id`、可选 `command`、`decision`、可选 `form`、可选 `reason`
- 兼容型 generic frontend tool 可接受 `payload`、`value`、`answer` 作为提交表单对象来源；对外协议推荐统一使用 `form`

典型形态：

```json
{
  "type": "awaiting.ask",
  "awaitingId": "await_003",
  "runId": "run_003",
  "mode": "form",
  "viewportType": "html",
  "viewportKey": "leave_form",
  "forms": [
    {
      "id": "form-1",
      "title": "请假申请",
      "command": "submit_leave_request",
      "form": {
        "employeeName": "Lin",
        "days": 1,
        "reason": ""
      }
    }
  ]
}
```

```json
{
  "runId": "run_003",
  "agentKey": "coder",
  "awaitingId": "await_003",
  "params": [
    {
      "id": "form-1",
      "decision": "approve",
      "form": {
        "employeeName": "Lin",
        "days": 2,
        "reason": "Conference"
      }
    }
  ]
}
```

reject 可带原因，也可带用户修改后的表单快照：

```json
{
  "runId": "run_003",
  "agentKey": "coder",
  "awaitingId": "await_003",
  "params": [
    {
      "id": "form-1",
      "decision": "reject",
      "reason": "请先把天数改成 1 天",
      "form": {
        "employeeName": "Lin",
        "days": 1,
        "reason": "Conference"
      }
    }
  ]
}
```

## 2. Plan

plan 用于 CODER planning confirmation。它不是工具审批，不放进 `approvals[]`；`awaiting.ask.plan` 是单个对象。

事件顺序：

- `planning.start -> planning.delta* -> planning.end -> awaiting.ask -> request.submit -> awaiting.answer`

关键点：

- `awaiting.ask.mode = "plan"`
- `plan` 直接内联在 `awaiting.ask`，且是单个对象
- plan 默认 `viewportType:"builtin"`、`viewportKey:"plan"`
- `params` 固定只接受 1 项，对应 `awaiting.ask.plan`
- `decision` 只能是 `approve` 或 `reject`；reject 可带 `reason`
- approve 后执行最新 plan；reject 后当前 plan 永不执行，模型可生成下一版 plan 或直接结束

典型形态：

```json
{
  "type": "awaiting.ask",
  "awaitingId": "run_004_coder_plan_confirm_1",
  "runId": "run_004",
  "mode": "plan",
  "viewportType": "builtin",
  "viewportKey": "plan",
  "plan": {
    "id": "run_004_coder_plan_confirm_1",
    "planningId": "run_004_planning_1",
    "title": "更新协议文档"
  }
}
```

```json
{
  "runId": "run_004",
  "agentKey": "coder",
  "awaitingId": "run_004_coder_plan_confirm_1",
  "params": [
    {
      "id": "run_004_coder_plan_confirm_1",
      "decision": "reject",
      "reason": "请把 viewportKey 边界写得更明确"
    }
  ]
}
```

## 3. 需要记住的边界

- question / approval / form / plan 都不再使用分离式 payload 事件
- 四态统一使用 `mode`，不再用 `kind` 作为对外主字段
- `request.submit` 是原始输入回看事件，`awaiting.answer` 才是归一化结果
- `params[]` 顶层始终是数组，按顺序对应当前批次 item；`mode=plan` 固定只接受 1 项
- question / approval / plan 默认是 builtin viewport；form 默认是 html viewport
- `viewportKey` 是前端视图载荷的检索键，不应从 `tool.start` 推断
