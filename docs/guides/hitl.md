# HITL 交互指南

本页单独展开 AGW 当前收敛后的 Human-in-the-loop 协议。事实源以 `agent-platform` 最近一轮 HITL 收口后的实现为准：对外交互统一使用 `mode`，当前等待模式为 `question`、`approval`、`form`、`plan`。

通用边界：

- `awaiting.ask`：声明等待态，并把当前交互定义直接发给前端
- `POST /api/submit`：统一提交 `agentKey + runId + awaitingId + params[]`
- `request.submit`：记录前端原始 `params[]`
- `awaiting.answer`：记录服务端归一化后的结构化结果
- `tool.result`：工具型 HITL 完成后，原工具继续在同一条流里产出结果

如果你还没有读主线流程，先看 [交互时序图](interaction-sequences.md)；如果你在查字段定义，再看 [HTTP API](../reference/http-api.md) 和 [SSE 事件模型](../reference/sse-events.md)。

## 1. Question

![05 AGW Question Sequence](../../assets/diagrams/sequences/05-agw-seq-question.svg)

question 用于 `_ask_user_question_` 这类“补信息”交互。当前约定是先发 `awaiting.ask`，再继续把工具参数流完。

事件顺序：

- `tool.start -> awaiting.ask -> tool.args* -> tool.end -> request.submit -> awaiting.answer -> tool.result`

关键点：

- `awaiting.ask.mode = "question"`
- `questions[]` 直接内联在 `awaiting.ask`
- question 默认 `viewportType:"builtin"`、`viewportKey:"question"`
- `params[i]` 使用 `answer` 或 `answers`，两者必须二选一

典型形态：

```json
{
  "type": "awaiting.ask",
  "awaitingId": "await_001",
  "runId": "run_001",
  "mode": "question",
  "viewportType": "builtin",
  "viewportKey": "question",
  "questions": [
    { "id": "q1", "question": "目标读者是谁？" },
    { "id": "q2", "question": "更偏实现还是更偏架构？" }
  ]
}
```

```json
{
  "runId": "run_001",
  "agentKey": "coder",
  "awaitingId": "await_001",
  "params": [
    { "id": "q1", "answer": "前端工程师" },
    { "id": "q2", "answer": "偏实现" }
  ]
}
```

## 2. Approval

![06 AGW Approval Sequence](../../assets/diagrams/sequences/06-agw-seq-approval.svg)

approval 主要对应 Bash HITL builtin confirm 或文件工具越权路径审批。旧 `_ask_user_approval_` 工具路径已退役，对外只保留 `awaiting.ask` 审批协议。

事件顺序：

- `tool.start -> tool.args* -> tool.end -> awaiting.ask -> request.submit -> awaiting.answer -> tool.result`

关键点：

- `awaiting.ask.mode = "approval"`
- `approvals[]` 直接内联在 `awaiting.ask`
- approval 默认 `viewportType:"builtin"`、`viewportKey:"approval"`
- `params[i]` 使用 `decision` / `reason`
- `decision` 的常见值是 `approve`、`approve_rule_run`、`reject`

典型形态：

```json
{
  "type": "awaiting.ask",
  "awaitingId": "await_002",
  "runId": "run_002",
  "mode": "approval",
  "viewportType": "builtin",
  "viewportKey": "approval",
  "approvals": [
    {
      "id": "tool_bash",
      "command": "chmod 777 ~/a.sh",
      "description": "放开脚本权限"
    }
  ]
}
```

```json
{
  "runId": "run_002",
  "agentKey": "coder",
  "awaitingId": "await_002",
  "params": [
    {
      "id": "tool_bash",
      "decision": "approve_rule_run",
      "reason": "同一规则本轮一并放行"
    }
  ]
}
```

## 3. Form

![08 AGW Form Sequence](../../assets/diagrams/sequences/08-agw-seq-form.svg)

form 用于 HTML 表单型交互。它默认使用 `viewportType:"html"`，`viewportKey` 由等待态输入提供，可配合 `GET /api/viewport` 获取视图 payload。

事件顺序：

- `tool.start -> tool.args* -> tool.end -> awaiting.ask -> request.submit -> awaiting.answer -> tool.result`

关键点：

- `awaiting.ask.mode = "form"`
- `forms[]` 直接内联在 `awaiting.ask`
- form 默认 `viewportType:"html"`；有表单视图时保留 `viewportKey`
- approve 必须提交 `decision:"approve"` + `form:{...}`
- reject 提交 `decision:"reject"`，可带 `reason` 和可选 `form`
- `awaiting.answer.forms[]` 会归一化为 `decision`、可选 `form`、可选 `reason`

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
    { "id": "form-1", "command": "submit_leave_request" }
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

## 4. Plan

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

## 5. 需要记住的边界

- question / approval / form / plan 都不再使用分离式 payload 事件
- 四态统一使用 `mode`，不再用 `kind` 作为对外主字段
- `request.submit` 是原始输入回看事件，`awaiting.answer` 才是归一化结果
- `params[]` 顶层始终是数组，按顺序对应当前批次 item；`mode=plan` 固定只接受 1 项
- question / approval / plan 默认是 builtin viewport；form 默认是 html viewport
- `viewportKey` 是前端视图载荷的检索键，不应从 `tool.start` 推断
