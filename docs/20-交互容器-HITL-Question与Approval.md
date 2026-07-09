# HITL Question 与 Approval

本页单独展开 AGW 当前收敛后的 Human-in-the-loop 协议。事实源以 `agent-platform` 最近一轮 HITL 收口后的实现为准：对外交互统一使用 `mode`，当前等待模式为 `question`、`approval`、`form`、`plan`。

通用边界：

- `awaiting.ask`：声明等待态，并把当前交互定义直接发给前端
- `POST /api/submit`：统一提交 `agentKey + runId + awaitingId + params[]`
- `request.submit`：记录前端原始 `params[]`
- `awaiting.answer`：记录服务端归一化后的结构化结果
- `tool.result`：工具型 HITL 完成后，原工具继续在同一条流里产出结果

如果你还没有读主线流程，先看 [交互时序图](40-交互时序-编号化主图与资产引用.md)；如果你在查字段定义，再看 [HTTP API](10-协议数据-HTTP核心运行接口.md) 和 [SSE 事件模型](13-协议数据-SSE传输与请求运行事件.md)。

## 1. Question

![05 AGW Question Sequence](../assets/diagrams/sequences/05-agw-seq-question.svg)

question 用于 `_ask_user_question_` 这类“补信息”交互。当前约定是先发 `awaiting.ask`，再继续把工具参数流完。

事件顺序：

- `tool.start -> awaiting.ask -> tool.args* -> tool.end -> request.submit -> awaiting.answer -> tool.result`

关键点：

- `awaiting.ask.mode = "question"`
- `questions[]` 直接内联在 `awaiting.ask`
- question 默认 `viewportType:"builtin"`、`viewportKey:"question"`
- 当前内置问题类型包括 `text`、`password`、`number`、`select`、`multi-select`、`date`、`datetime`
- 非 `multi-select` 问题提交 `answer`；`multi-select` 问题提交 `answers`，两者不能同时出现
- `date` 和 `datetime` 必须使用单值 `answer`，不能使用 `answers`

`date` / `datetime` 细节：

- `date` 推荐前端提交 `YYYY-MM-DD`，例如 `2026-05-15`
- `datetime` 推荐前端提交 `YYYY-MM-DDTHH:mm`，例如 `2026-05-15T09:30`
- 需要表达时区时可以在字符串中携带时区信息，但协议层仍按字符串传递
- 服务端只要求非空字符串并原样写入 `awaiting.answer.answers[].answer`，不承诺做日期解析、时区转换或强格式校验

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

日期时间型问题示例：

```json
{
  "type": "awaiting.ask",
  "awaitingId": "await_date_001",
  "runId": "run_001",
  "mode": "question",
  "viewportType": "builtin",
  "viewportKey": "question",
  "questions": [
    { "id": "startDate", "question": "开始日期？", "type": "date" },
    { "id": "startAt", "question": "开始时间？", "type": "datetime" }
  ]
}
```

```json
{
  "runId": "run_001",
  "agentKey": "coder",
  "awaitingId": "await_date_001",
  "params": [
    { "id": "startDate", "answer": "2026-05-15" },
    { "id": "startAt", "answer": "2026-05-15T09:30" }
  ]
}
```

## 2. Approval

![06 AGW Approval Sequence](../assets/diagrams/sequences/06-agw-seq-approval.svg)

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
