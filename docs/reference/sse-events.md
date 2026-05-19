# SSE 事件模型

本页描述 AGW UI 的事件语义和 SSE 线缆格式。WebSocket 复用同一批业务事件，并通过 `stream.event` 承载，连接级差异见 [WebSocket 协议](websocket-protocol.md)。

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

## 7. Reasoning 事件

| 事件 | 关键字段 |
| --- | --- |
| `reasoning.start` | `reasoningId`、`runId`、可选 `taskId`、可选 `reasoningLabel` |
| `reasoning.delta` | `reasoningId`、`delta` |
| `reasoning.end` | `reasoningId` |
| `reasoning.snapshot` | `reasoningId`、`runId`、`text`、可选 `taskId`、可选 `reasoningLabel` |

`reasoning.snapshot` 主要用于历史记录。

## 8. Content 事件

| 事件 | 关键字段 |
| --- | --- |
| `content.start` | `contentId`、`runId`、可选 `taskId` |
| `content.delta` | `contentId`、`delta` |
| `content.end` | `contentId` |
| `content.snapshot` | `contentId`、`runId`、`text`、可选 `taskId` |

`content.snapshot` 主要用于历史记录。

## 9. Awaiting 事件

### 9.1 `awaiting.ask`

进入等待态的事件，直接内联前端需要渲染的 question / approval / form 定义。

| 字段 | 说明 |
| --- | --- |
| `awaitingId` | 当前等待态 ID |
| `mode` | `"question" \| "approval" \| "form"` |
| `viewportType` | 可选；主要用于 form |
| `viewportKey` | 可选；主要用于 form，可配合 `GET /api/viewport` |
| `timeout` | 可选；等待超时秒数 |
| `runId` | 当前 run |
| `questions` | question 模式下出现 |
| `approvals` | approval 模式下出现 |
| `forms` | form 模式下出现 |

说明：

- 当前协议统一使用 `mode`，不以 `kind` 作为对外主字段。
- question / approval / form 的交互定义都直接位于 `awaiting.ask`。
- form 是当前主要保留 viewport 语义的等待态。

### 9.2 `awaiting.answer`

服务端处理提交后的归一化结果事件。

| 字段 | 说明 |
| --- | --- |
| `awaitingId` | 当前等待态 ID |
| `mode` | `"question" \| "approval" \| "form"` |
| `status` | 可选；归一化状态 |
| `answers` | question 模式下出现 |
| `approvals` | approval 模式下出现 |
| `forms` | form 模式下出现 |
| `error` | 可选；处理错误 |

说明：`request.submit` 记录原始 `params[]`，`awaiting.answer` 记录服务端归一化结构。

## 10. Tool 事件

| 事件 | 关键字段 |
| --- | --- |
| `tool.start` | `toolId`、`runId`、可选 `taskId/toolName/toolLabel/toolDescription` |
| `tool.args` | `toolId`、`delta`、`chunkIndex` |
| `tool.end` | `toolId` |
| `tool.snapshot` | `toolId`、`runId`、`toolName`、可选 `taskId/toolLabel/toolDescription/arguments` |
| `tool.result` | `toolId`、`result`、可选 `hitl` |

`tool.snapshot` 主要用于历史记录。视图相关信息应以 `awaiting.ask` 或 `/api/viewport` 为准，不默认挂在 `tool.start` 上。

## 11. Action 事件

| 事件 | 关键字段 |
| --- | --- |
| `action.start` | `actionId`、`runId`、可选 `taskId/actionName/description` |
| `action.args` | `actionId`、`delta` |
| `action.end` | `actionId` |
| `action.snapshot` | `actionId`、`runId`、可选 `actionName/taskId/description/arguments` |
| `action.result` | `actionId`、`result` |

`action.snapshot` 主要用于历史记录。

## 12. Source 事件

### 12.1 `source.publish`

发布来源信息块。

| 字段 | 说明 |
| --- | --- |
| `publishId` | 发布 ID |
| `runId` | 当前 run |
| `taskId` | 可选 |
| `toolId` | 可选 |
| `kind` | 来源类型 |
| `query` | 可选；来源查询 |
| `sourceCount` | 来源数量 |
| `chunkCount` | chunk 数量 |
| `sources` | 来源列表 |

旧文档中的 `source.snapshot` 不属于当前实现的默认实时事件名。

## 13. Artifact 事件

### 13.1 `artifact.publish`

向当前 chat 资产池发布产物。当前实现的事件主形态是批量字段：

| 字段 | 说明 |
| --- | --- |
| `chatId` | 所属 chat |
| `runId` | 来源 run |
| `artifactCount` | 本次发布产物数量 |
| `artifacts` | 产物数组，元素通常含 `type/name/mimeType/sizeBytes/url/sha256` |

历史聚合中，`artifact.publish` 可折叠为 chat 级 `artifact` 聚合态。

## 14. Debug 与 Memory 事件

### 14.1 `debug.preCall` / `debug.postCall`

| 字段 | 说明 |
| --- | --- |
| `runId` | 当前 run |
| `chatId` | 当前 chat |
| `data` | 调试数据 |

这些事件受服务端调试配置影响，不应作为普通前端渲染的必需事件。

### 14.2 `memory.*`

当前实现存在以下记忆相关事件：

```text
memory.write
memory.read
memory.search
memory.update
memory.forget
memory.timeline
memory.promote
memory.consolidate
```

统一字段：

| 字段 | 说明 |
| --- | --- |
| `runId` | 当前 run |
| `chatId` | 当前 chat |
| `data` | 记忆操作数据 |

## 15. 层级关系速览

```text
chat
├── plan
├── run
│   ├── task
│   │   ├── reasoning
│   │   ├── content
│   │   ├── awaiting
│   │   ├── tool
│   │   ├── action
│   │   └── source.publish
│   ├── artifact.publish
│   ├── memory.*
│   └── debug.*
└── history / aggregate snapshots
```

## 16. 重点边界

- `query` 对应 `request.query`，成功响应是 SSE。
- `attach` 只观察已有 run，不创建新 run。
- `submit` 对应 `request.submit`，归一化结果对应 `awaiting.answer`。
- `steer` 对应 `request.steer`。
- `interrupt` 对应 `run.cancel`，没有 `request.interrupt`。
- heartbeat 和 `[DONE]` 是传输层概念，不是业务 JSON 事件。
- snapshot 事件属于历史持久化或回放，不属于 live SSE 必发主路径。
