# SSE 事件模型

本页描述的是“事件语义 + SSE 线缆格式”。如果服务端启用 WebSocket，这些业务事件也可以通过 `stream.event` 承载；WebSocket 特有的连接、结束和错误语义请见 [WebSocket 协议](websocket-protocol.md)。

## 1. 传输约定

### 1.1 业务事件

业务事件统一使用：

```text
event: message
data: {"seq":3,"type":"run.start","timestamp":1707000002,"runId":"run_001","chatId":"chat_123","agentKey":"auto"}
```

说明：

- `data` 必须是一行 JSON。
- 实时业务事件统一包含 `seq`、`type`、`timestamp`。

### 1.2 Heartbeat

SSE 传输层保活事件；由注释帧标准化而来，不属于业务 JSON 事件。

```text
: heartbeat
```

| 项 | 说明 |
| --- | --- |
| `wire format` | `: heartbeat` |
| `data` | 无 |

### 1.3 Done Sentinel

流结束时追加的传输层终止帧，不是业务事件，也不会写入历史 `events`。

```text
event: message
data: [DONE]
```

| 项 | 说明 |
| --- | --- |
| `wire format` | `event: message` + `data: [DONE]` |
| `data` | 非 JSON 文本 |

## 2. Base Event

所有事件类共享的基础字段：

| 字段 | 说明 |
| --- | --- |
| `seq` | 唯一序列号；实时业务事件统一包含 |
| `type` | 事件类型 |
| `timestamp` | 事件创建时间戳 |
| `rawEvent` | 调试用途原始事件对象，可选 |

这里列的都是会真实出现在 SSE 流里的事件；公开 HTTP API 不要求与流事件形成 1:1 命名映射。

## 3. 事件分层

当前实现需要区分三类内容：

- live SSE 事件：会通过 `POST /api/query` 实时发给客户端。
- snapshot / persisted 事件：只进入历史持久化，不走 live SSE。
- 传输层帧：heartbeat 与 `[DONE]`，不属于业务 JSON 事件。

补充：

- 这里定义的 `request.* / run.* / content.* / tool.*` 属于业务事件语义
- SSE 特有的是 `event: message`、`: heartbeat`、`data: [DONE]`
- 对应的颜色与类别预览可见 [SSE Event Color Preview](../visuals/sse-event-color-preview.html)

## 4. Request 回看事件

### 4.1 `request.query`

请求动作事件：`request.query`，对应 API `POST /api/query`。

| 字段 | 说明 |
| --- | --- |
| `type` | 固定为 `"request.query"` |
| `requestId` | 前端生成，用于幂等、重试和链路追踪 |
| `chatId` | 聊天会话 ID，可由前端生成或由服务端创建后回传 |
| `agentKey` | 可选；如 `auto`、`default`、`agent_researcher` |
| `teamId` | 可选 |
| `role` | `"user" \| "system" \| "assistant" \| "developer" \| "other"` |
| `message` | 用户消息 |
| `references` | `Reference[]`，可选 |
| `params` | 结构化参数，可选 |
| `scene` | `{url,title}`，可选 |
| `stream` | 可选；当前实现固定返回 SSE |
| `hidden` | 可选 |

说明：当前事件 payload 不应等同于 HTTP request body；只有实现实际写入流中的字段才属于 live SSE 协议。

### 4.3 `request.submit`

请求动作事件：`request.submit`，对应 API `POST /api/submit`。这是运行流里的回看事件，不等于公开 HTTP body。

| 字段 | 说明 |
| --- | --- |
| `type` | 固定为 `"request.submit"` |
| `requestId` | 前端生成 |
| `chatId` | chat ID |
| `runId` | 当前 run |
| `awaitingId` | 当前提交关联的等待态 |
| `params` | 前端原始提交数组；顺序与当前 `awaiting.ask` item 一一对应 |

说明：

- `request.submit` 保留前端原始输入，便于审计与回放。
- 归一化后的结构化结果由 `awaiting.answer` 表达。

### 4.4 `request.steer`

请求动作事件：`request.steer`，对应 API `POST /api/steer`。这是运行流里的回看事件，用于记录运行中注入的追加指令；它不是 `/api/steer` 的完整 HTTP body schema。

| 字段 | 说明 |
| --- | --- |
| `type` | 固定为 `"request.steer"` |
| `requestId` | 可选 |
| `chatId` | chat ID |
| `runId` | 当前 run |
| `steerId` | 当前 steer |
| `message` | 追加指令 |
| `role` | 固定为 `"user"` |

### 4.5 `request.interrupt` 不存在

- 当前流层不会发独立的 `request.interrupt`。
- `POST /api/interrupt` 的结果直接用 `run.cancel` 表达。

## 5. Chat 与 Plan 事件

### 5.1 `chat.start`

开始聊天会话的事件。

| 字段 | 说明 |
| --- | --- |
| `chatId` | 会话唯一标识 |
| `chatName` | 会话名称 |

说明：当前实时 SSE 只发会话开始事件；名称变更不会额外输出一条独立的实时会话更新事件。

### 5.2 `plan.update`

更新任务计划的事件。

| 字段 | 说明 |
| --- | --- |
| `planId` | 计划唯一标识 |
| `chatId` | 关联 chat |
| `plan` | 计划列表，包含多个任务对象 |

说明：当前实时 SSE 统一使用计划更新事件表达首次创建与后续修改，不再拆成单独“创建”事件。

## 6. Run 事件

### 6.1 `run.start`

| 字段 | 说明 |
| --- | --- |
| `runId` | 运行实例 ID |
| `chatId` | 关联 chat |
| `agentKey` | 当前运行的智能体 key |

### 6.2 `run.complete`

| 字段 | 说明 |
| --- | --- |
| `runId` | 已完成的运行实例 ID |
| `finishReason` | 完成原因；取值由当前运行流决定 |

### 6.3 `run.cancel`

运行被取消或中断时的实时事件。常见来源是 `POST /api/interrupt`。

| 字段 | 说明 |
| --- | --- |
| `runId` | 被取消的运行实例 ID |

### 6.4 `run.error`

| 字段 | 说明 |
| --- | --- |
| `runId` | 出错的运行实例 ID |
| `error` | 错误信息 |

## 7. Task 事件

### 7.1 `task.start`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 任务唯一标识 |
| `runId` | 所属运行实例 ID |
| `taskName` | 任务名称 |
| `description` | 任务描述 |

### 7.2 `task.complete`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 完成的任务 ID |

### 7.3 `task.cancel`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 被取消的任务 ID |

### 7.4 `task.fail`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 失败的任务 ID |
| `error` | 错误信息 |

## 8. Reasoning 事件

### 8.1 `reasoning.start`

| 字段 | 说明 |
| --- | --- |
| `reasoningId` | 推理唯一标识 |
| `runId` | 运行实例 ID |
| `taskId` | 可选；任务 ID |

### 8.2 `reasoning.delta`

| 字段 | 说明 |
| --- | --- |
| `reasoningId` | 推理唯一标识 |
| `delta` | 推理增量内容 |

### 8.3 `reasoning.end`

| 字段 | 说明 |
| --- | --- |
| `reasoningId` | 推理唯一标识 |

### 8.4 `reasoning.snapshot`

主要用于历史记录；不是当前 `/api/query` 实时流主路径。

| 字段 | 说明 |
| --- | --- |
| `reasoningId` | 推理唯一标识 |
| `runId` | 运行实例 ID |
| `taskId` | 可选；任务 ID |
| `text` | 推理全文 |

## 9. Content 事件

### 9.1 `content.start`

| 字段 | 说明 |
| --- | --- |
| `contentId` | 正文唯一标识 |
| `runId` | 运行实例 ID |
| `taskId` | 可选；任务 ID |

### 9.2 `content.delta`

| 字段 | 说明 |
| --- | --- |
| `contentId` | 正文唯一标识 |
| `delta` | 正文增量内容 |

### 9.3 `content.end`

| 字段 | 说明 |
| --- | --- |
| `contentId` | 正文唯一标识 |

### 9.4 `content.snapshot`

主要用于历史记录；不是当前 `/api/query` 实时流主路径。

| 字段 | 说明 |
| --- | --- |
| `contentId` | 正文唯一标识 |
| `runId` | 运行实例 ID |
| `taskId` | 可选；任务 ID |
| `text` | 正文全文 |

## 10. Awaiting 事件

### 10.1 `awaiting.ask`

进入等待态的事件；用于把当前 question / approval / form 交互定义直接发给前端。

| 字段 | 说明 |
| --- | --- |
| `awaitingId` | 当前等待态唯一标识 |
| `runId` | 当前运行实例 ID |
| `mode` | `"question" \| "approval" \| "form"` |
| `timeout` | 可选；等待超时秒数 |
| `questions` | question 模式下出现；问题定义直接内联 |
| `approvals` | approval 模式下出现；审批项直接内联 |
| `forms` | form 模式下出现；表单定义直接内联 |
| `viewportType` | 仅 form 模式保留；当前常见为 `"html"` |
| `viewportKey` | 仅 form 模式保留；前端可据此取视图 |
| `viewportPayload` | 可选；form 相关视图载荷 |

说明：

- 当前协议统一使用 `mode`，不再以 `kind` 作为对外主字段。
- 旧版分离式 payload 事件已移除；question / approval / form 的交互定义都直接位于 `awaiting.ask`。
- `question` 常见顺序是 `tool.start -> awaiting.ask -> tool.args* -> tool.end`。
- `approval / form` 常见顺序是 `tool.start -> tool.args* -> tool.end -> awaiting.ask`。
- question / approval 默认不要求 `viewportType`、`viewportKey`；form 是唯一保留 viewport 语义的形态。

### 10.2 `awaiting.answer`

服务端在处理完提交后写回的归一化结果事件；与 `request.submit` 配套出现。

| 字段 | 说明 |
| --- | --- |
| `awaitingId` | 当前等待态唯一标识 |
| `runId` | 当前运行实例 ID |
| `mode` | `"question" \| "approval" \| "form"` |
| `answers` | question 模式下出现；归一化答案数组 |
| `approvals` | approval 模式下出现；归一化审批结果数组 |
| `forms` | form 模式下出现；归一化表单结果数组 |
| `cancelled` | 可选；整批取消时为 `true` |
| `reason` | 可选；取消或拒绝原因 |

说明：

- `request.submit` 记录原始 `params[]`，`awaiting.answer` 记录服务端归一化后的结构化结果。
- form 的每一项通常会归一化成 `action: "submit" | "reject" | "cancel"`。

## 11. Tool 事件

### 11.1 `tool.start`

工具开始事件；这里只列当前 live SSE 主路径里稳定出现的字段。

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具唯一标识 |
| `runId` | 运行实例 ID |
| `toolName` | 工具名称 |
| `taskId` | 可选；任务 ID |
| `toolLabel` | 可选；展示名 |
| `toolDescription` | 可选；展示描述 |

说明：

- 当前实现的 live `tool.start` 不应默认暴露 `toolType`、`viewportKey`、`toolTimeout`。
- 需要视图相关信息时，应以 `awaiting.ask` 或 `GET /api/viewport` 的约定为准，而不是把这些字段视为所有 `tool.start` 的默认 schema。

### 11.2 `tool.args`

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具唯一标识 |
| `delta` | 工具参数增量 |
| `chunkIndex` | 同一 `toolId` 下的分片序号 |

### 11.3 `tool.end`

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具唯一标识 |

### 11.4 `tool.snapshot`

主要用于历史记录；不是当前 `/api/query` 实时流主路径。

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具唯一标识 |
| `runId` | 运行实例 ID |
| `toolName` | 工具名称 |
| `taskId` | 可选；任务 ID |
| `toolLabel` | 可选 |
| `toolDescription` | 可选 |
| `arguments` | 工具参数快照 |

说明：

- 历史 `tool.snapshot` 也不应被文档默认定义为包含 `toolType`、`viewportKey`、`toolTimeout`。
- 如某些兼容实现或视图相关上下文带出额外字段，应作为具体实现扩展说明，不应提升为当前协议默认字段。

### 11.5 `tool.result`

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具唯一标识 |
| `result` | 工具运行结果 |

### 11.6 提交边界

- 前端交互提交走 `POST /api/submit`。
- HTTP body 使用 `runId + awaitingId + params[]` 标识当前等待态与提交内容。
- 运行流里会依次出现 `request.submit` 与 `awaiting.answer`。
- 不会额外发一条独立的“工具参数事件”。

## 12. Artifact 事件

### 12.1 `artifact.publish`

向当前 chat 资产池发布产物的事件；每个产物对应一条 `artifact.publish`。

| 字段 | 说明 |
| --- | --- |
| `artifactId` | 产物唯一标识 |
| `chatId` | 所属 chat；产物进入该 chat 的资产集合 |
| `runId` | 来源 run；用于溯源，不表示归属层级 |
| `artifact` | 精简产物对象，仅包含 `type/name/mimeType/sizeBytes/url/sha256` |

说明：

- 一次调用 `_artifact_publish_` 时，`artifacts[]` 里有几个产物，就会出现几条 `artifact.publish`。
- 常见顺序是 `tool.start -> tool.args -> tool.end -> tool.result -> artifact.publish`。
- 如果一次调用里有多个产物，`tool.result` 后会连续出现多条 `artifact.publish`。

## 13. Action 事件

### 13.1 `action.start`

| 字段 | 说明 |
| --- | --- |
| `actionId` | 动作唯一标识 |
| `runId` | 运行实例 ID |
| `actionName` | 动作名称 |
| `taskId` | 可选；任务 ID |
| `description` | 动作描述 |

### 13.2 `action.args`

| 字段 | 说明 |
| --- | --- |
| `actionId` | 动作唯一标识 |
| `delta` | 动作参数增量 |

### 13.3 `action.end`

| 字段 | 说明 |
| --- | --- |
| `actionId` | 动作唯一标识 |

### 13.4 `action.snapshot`

主要用于历史记录；不是当前 `/api/query` 实时流主路径。

| 字段 | 说明 |
| --- | --- |
| `actionId` | 动作唯一标识 |
| `runId` | 运行实例 ID |
| `actionName` | 动作名称 |
| `taskId` | 可选；任务 ID |
| `description` | 动作描述 |
| `arguments` | 动作参数 |

### 13.5 `action.result`

| 字段 | 说明 |
| --- | --- |
| `actionId` | 动作唯一标识 |
| `result` | 动作执行结果 |

## 14. Source 事件

### 14.1 `source.snapshot`

源信息块的快照事件；当前仍未实现实时 SSE 输出。

| 字段 | 说明 |
| --- | --- |
| `icon` | 来源图标 |
| `title` | 来源标题 |
| `url` | 来源地址 |

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
│   │   └── action
│   └── artifact.publish
└── source(snapshot, optional)
```

## 16. 需要重点记住的边界

- `query` 对应 `request.query`
- `submit` 对应 `request.submit`
- 提交归一化结果对应 `awaiting.answer`
- `steer` 对应 `request.steer`
- `interrupt` 对应 `run.cancel`
- `heartbeat` 和 `event: message
data: [DONE]` 都是传输层概念，不是业务 JSON 事件
- snapshot 事件属于历史持久化，不属于 live SSE
