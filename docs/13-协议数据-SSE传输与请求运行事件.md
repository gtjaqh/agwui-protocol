# SSE 传输与请求运行事件

本页描述 AGW UI 的事件语义和 SSE 线缆格式。WebSocket 复用同一批业务事件，并通过 `stream.event` 承载，连接级差异见 [WebSocket 协议](15-协议数据-WebSocket传输协议.md)。

## 1. 传输约定

### 1.1 业务事件

业务事件统一使用：

```text
event: message
data: {"seq":3,"type":"run.start","runId":"run_001","chatId":"chat_123","agentKey":"auto","timestamp":1707000002000}
```

说明：

- `data` 是单行 JSON。
- 实时业务事件统一包含 `seq`、`type`、`timestamp`。
- `timestamp` 使用毫秒时间戳。

### 1.2 Heartbeat

SSE 保活事件是注释帧，不属于业务 JSON 事件。

```text
: heartbeat
```

### 1.3 Done Sentinel

流结束时追加传输层终止帧，不进入历史事件。

```text
event: message
data: [DONE]
```

## 2. 事件分层

- live SSE 事件：通过 `POST /api/query` 或 `GET /api/attach` 实时输出。
- snapshot / persisted 事件：用于历史回放或聚合态，不一定出现在 live SSE 主路径。
- 传输层帧：heartbeat 与 `[DONE]`，不是业务事件。

所有业务事件共享基础字段：

| 字段 | 说明 |
| --- | --- |
| `seq` | 事件序号 |
| `type` | 事件类型 |
| `timestamp` | 事件创建毫秒时间戳 |
| `rawEvent` | 可选；调试用途原始事件对象 |

## 3. Request 回看事件

### 3.1 `request.query`

对应 `POST /api/query` 的运行回看事件。

| 字段 | 说明 |
| --- | --- |
| `requestId` | 请求 ID；缺省时等于 `runId` |
| `runId` | 运行 ID |
| `chatId` | 聊天会话 ID |
| `role` | 当前消息角色 |
| `message` | 用户消息 |
| `agentKey` | 可选；空字符串会省略 |
| `teamId` | 可选；空字符串会省略 |
| `references` | 可选；`Reference[]` |
| `params` | 可选；结构化参数 |
| `scene` | 可选；页面上下文 |
| `stream` | 可选；兼容字段 |
| `hidden` | 可选 |

### 3.2 `request.submit`

对应 `POST /api/submit` 的运行回看事件，保留前端原始提交。

| 字段 | 说明 |
| --- | --- |
| `requestId` | 请求 ID |
| `chatId` | chat ID |
| `runId` | 当前 run |
| `awaitingId` | 当前等待态 |
| `params` | 前端原始提交数组 |

归一化结果由 `awaiting.answer` 表达。

### 3.3 `request.steer`

对应 `POST /api/steer` 的运行回看事件。

| 字段 | 说明 |
| --- | --- |
| `requestId` | 可选 |
| `chatId` | chat ID |
| `runId` | 当前 run |
| `steerId` | 当前追加指令 ID |
| `message` | 追加指令 |
| `role` | 固定为 `"user"` |

### 3.4 `request.interrupt` 不存在

当前流层不会发独立的 `request.interrupt`。`POST /api/interrupt` 的运行结果由 `run.cancel` 表达。

## 4. Chat、Plan 与 Stage 事件

### 4.1 `chat.start`

| 字段 | 说明 |
| --- | --- |
| `chatId` | 会话唯一标识 |
| `chatName` | 会话名称，可为空并被省略 |

### 4.2 `plan.create` / `plan.update`

| 字段 | 说明 |
| --- | --- |
| `planId` | 计划唯一标识 |
| `chatId` | 关联 chat |
| `plan` | 任务计划列表或聚合对象 |

当前实时主路径通常使用 `plan.update` 表达创建与后续修改；历史中可折叠为 chat 级 `plan`。

### 4.3 `stage.marker`

| 字段 | 说明 |
| --- | --- |
| `runId` | 当前 run |
| `chatId` | 当前 chat |
| `stage` | 阶段标记 |

### 4.4 `planning.*`

CODER planning 模式通过 `planning_write` 产出可确认的执行计划。`planning_write` 的内部工具参数仍可叫 `markdown`，但对外 live / replay 事件中的计划正文统一使用 `text`。

| 事件 | 关键字段 |
| --- | --- |
| `planning.start` | `planningId`、可选 `planningFile/chatId/runId/title/updatedAt` |
| `planning.delta` | `planningId`、`delta` |
| `planning.snapshot` | `planningId`、可选 `planningFile/chatId/runId/title/text/updatedAt` |
| `planning.end` | `planningId` |

`planning.start/delta/end` 是 planning 能力事件，不是 `planning_write` 工具 UI 事件。即使 `planning_write.yml` 配置 `clientVisible:false`，也只隐藏对应 `tool.*` 展示，不隐藏 `planning.*` 生命周期事件。

## 5. Run 事件

### 5.1 `run.start`

| 字段 | 说明 |
| --- | --- |
| `runId` | 运行实例 ID |
| `chatId` | 关联 chat |
| `agentKey` | 当前智能体 key |

### 5.2 `run.complete`

| 字段 | 说明 |
| --- | --- |
| `runId` | 已完成 run |
| `finishReason` | 完成原因 |
| `usage` | 可选；token 用量 |

### 5.3 `run.cancel`

| 字段 | 说明 |
| --- | --- |
| `runId` | 被取消或中断的 run |
| `usage` | 可选；token 用量 |

### 5.4 `run.error`

| 字段 | 说明 |
| --- | --- |
| `runId` | 出错 run |
| `error` | 错误信息 |
| `usage` | 可选；token 用量 |

### 5.5 `run.expired`

| 字段 | 说明 |
| --- | --- |
| `runId` | 过期 run |

## 6. Task 事件

### 6.1 `task.start`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 任务 ID |
| `runId` | 所属 run |
| `taskName` | 任务名称 |
| `description` | 可选；任务描述 |
| `subAgentKey` | 可选；子智能体 key |
| `toolId` | 可选；关联工具 ID |

### 6.2 `task.complete` / `task.cancel`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 任务 ID |
| `status` | 可选；状态 |

### 6.3 `task.fail`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 任务 ID |
| `status` | 可选；状态 |
| `error` | 错误信息 |
