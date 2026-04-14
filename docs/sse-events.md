# SSE 事件模型

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
event: heartbeat
```

| 项 | 说明 |
| --- | --- |
| `wire format` | `event: heartbeat` |
| `data` | 无 |

### 1.3 Done Sentinel

流结束时追加的传输层终止帧，不是业务事件，也不会写入历史 `events`。

```text
data:[DONE]
```

| 项 | 说明 |
| --- | --- |
| `wire format` | `data:[DONE]` |
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

## 3. Request 回看事件

### 3.1 `request.query`

请求动作事件：`request.query`，对应 API `POST /api/query`。

| 字段 | 说明 |
| --- | --- |
| `type` | 固定为 `"request.query"` |
| `requestId` | 前端生成，用于幂等、重试和链路追踪 |
| `chatId` | 聊天会话 ID，可由前端生成或由网关创建后回传 |
| `agentKey` | 可选；如 `auto`、`default`、`agent_researcher` |
| `teamId` | 可选 |
| `role` | `"user" \| "system" \| "assistant" \| "developer" \| "other"` |
| `message` | 用户消息 |
| `references` | `Reference[]`，可选 |
| `params` | 结构化参数，可选 |
| `scene` | `{url,title}`，可选 |
| `stream` | 可选；当前网关仍固定返回 SSE |
| `hidden` | 可选 |

### 3.2 `request.upload`

请求动作事件：`request.upload`，对应 API `POST /api/upload`。这是统一事件记录中的内部对象；公开 HTTP 请求本身是 `multipart/form-data`，不是 JSON `upload` 对象。

| 字段 | 说明 |
| --- | --- |
| `type` | 固定为 `"request.upload"` |
| `requestId` | 前端生成 |
| `chatId` | chat ID，可选；缺省则网关创建 |
| `upload.type` | `"file" \| "image"` |
| `upload.name` | 文件名 |
| `upload.sizeBytes` | 文件大小 |
| `upload.mimeType` | MIME 类型 |
| `upload.sha256` | 可选 |

### 3.3 `request.submit`

请求动作事件：`request.submit`，对应 API `POST /api/submit`。这是运行流里的回看事件，不等于公开 HTTP body。

| 字段 | 说明 |
| --- | --- |
| `type` | 固定为 `"request.submit"` |
| `requestId` | 前端生成 |
| `chatId` | chat ID |
| `runId` | 当前 run |
| `toolId` | 提交给哪个工具 |
| `payload` | 表单值、按钮值、选择结果等 |
| `viewId` | 可选 |

### 3.4 `request.steer`

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

### 3.5 `request.interrupt` 不存在

- 当前流层不会发独立的 `request.interrupt`。
- `POST /api/interrupt` 的结果直接用 `run.cancel` 表达。

## 4. Chat 与 Plan 事件

### 4.1 `chat.start`

开始聊天会话的事件。

| 字段 | 说明 |
| --- | --- |
| `chatId` | 会话唯一标识 |
| `chatName` | 会话名称 |

说明：当前实时 SSE 只发会话开始事件；名称变更不会额外输出一条独立的实时会话更新事件。

### 4.2 `plan.update`

更新任务计划的事件。

| 字段 | 说明 |
| --- | --- |
| `planId` | 计划唯一标识 |
| `chatId` | 关联 chat |
| `plan` | 计划列表，包含多个任务对象 |

说明：当前实时 SSE 统一使用计划更新事件表达首次创建与后续修改，不再拆成单独“创建”事件。

## 5. Run 事件

### 5.1 `run.start`

| 字段 | 说明 |
| --- | --- |
| `runId` | 运行实例 ID |
| `chatId` | 关联 chat |
| `agentKey` | 当前运行的智能体 key |

### 5.2 `run.complete`

| 字段 | 说明 |
| --- | --- |
| `runId` | 已完成的运行实例 ID |
| `finishReason` | 完成原因，缺省为 `end_turn` |

### 5.3 `run.cancel`

运行被取消或中断时的实时事件。常见来源是 `POST /api/interrupt`。

| 字段 | 说明 |
| --- | --- |
| `runId` | 被取消的运行实例 ID |

### 5.4 `run.error`

| 字段 | 说明 |
| --- | --- |
| `runId` | 出错的运行实例 ID |
| `error` | 错误信息 |

## 6. Task 事件

### 6.1 `task.start`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 任务唯一标识 |
| `runId` | 所属运行实例 ID |
| `taskName` | 任务名称 |
| `description` | 任务描述 |

### 6.2 `task.complete`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 完成的任务 ID |

### 6.3 `task.cancel`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 被取消的任务 ID |

### 6.4 `task.fail`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 失败的任务 ID |
| `error` | 错误信息 |

## 7. Reasoning 事件

### 7.1 `reasoning.start`

| 字段 | 说明 |
| --- | --- |
| `reasoningId` | 推理唯一标识 |
| `runId` | 运行实例 ID |
| `taskId` | 可选；任务 ID |

### 7.2 `reasoning.delta`

| 字段 | 说明 |
| --- | --- |
| `reasoningId` | 推理唯一标识 |
| `delta` | 推理增量内容 |

### 7.3 `reasoning.end`

| 字段 | 说明 |
| --- | --- |
| `reasoningId` | 推理唯一标识 |

### 7.4 `reasoning.snapshot`

主要用于历史记录；不是当前 `/api/query` 实时流主路径。

| 字段 | 说明 |
| --- | --- |
| `reasoningId` | 推理唯一标识 |
| `runId` | 运行实例 ID |
| `taskId` | 可选；任务 ID |
| `text` | 推理全文 |

## 8. Content 事件

### 8.1 `content.start`

| 字段 | 说明 |
| --- | --- |
| `contentId` | 正文唯一标识 |
| `runId` | 运行实例 ID |
| `taskId` | 可选；任务 ID |

### 8.2 `content.delta`

| 字段 | 说明 |
| --- | --- |
| `contentId` | 正文唯一标识 |
| `delta` | 正文增量内容 |

### 8.3 `content.end`

| 字段 | 说明 |
| --- | --- |
| `contentId` | 正文唯一标识 |

### 8.4 `content.snapshot`

主要用于历史记录；不是当前 `/api/query` 实时流主路径。

| 字段 | 说明 |
| --- | --- |
| `contentId` | 正文唯一标识 |
| `runId` | 运行实例 ID |
| `taskId` | 可选；任务 ID |
| `text` | 正文全文 |

## 9. Tool 事件

### 9.1 `tool.start`

工具开始事件；前端工具在最终输出中会补充 viewport 相关字段。

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具唯一标识 |
| `runId` | 运行实例 ID |
| `toolName` | 工具名称 |
| `taskId` | 可选；任务 ID |
| `toolType` | 可选；前端工具常见为 `html/qlc/dqlc` |
| `toolLabel` | 可选；展示名 |
| `toolDescription` | 可选；展示描述 |
| `viewportKey` | 仅前端工具会出现 |
| `toolTimeout` | 仅前端工具会出现，单位毫秒 |

### 9.2 `tool.args`

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具唯一标识 |
| `delta` | 工具参数增量 |
| `chunkIndex` | 同一 `toolId` 下的分片序号 |

### 9.3 `tool.end`

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具唯一标识 |

### 9.4 `tool.snapshot`

主要用于历史记录；不是当前 `/api/query` 实时流主路径。

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具唯一标识 |
| `runId` | 运行实例 ID |
| `toolName` | 工具名称 |
| `taskId` | 可选；任务 ID |
| `toolType` | 可选 |
| `toolLabel` | 可选 |
| `toolDescription` | 可选 |
| `viewportKey` | 前端工具可能出现 |
| `toolTimeout` | 前端工具可能出现 |
| `arguments` | 工具参数快照 |

### 9.5 `tool.result`

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具唯一标识 |
| `result` | 工具运行结果 |

### 9.6 提交边界

- 前端交互提交走 `POST /api/submit`。
- 运行流里会记录为 `request.submit`。
- 不会额外发一条独立的“工具参数事件”。

## 10. Artifact 事件

### 10.1 `artifact.publish`

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

## 11. Action 事件

### 11.1 `action.start`

| 字段 | 说明 |
| --- | --- |
| `actionId` | 动作唯一标识 |
| `runId` | 运行实例 ID |
| `actionName` | 动作名称 |
| `taskId` | 可选；任务 ID |
| `description` | 动作描述 |

### 11.2 `action.args`

| 字段 | 说明 |
| --- | --- |
| `actionId` | 动作唯一标识 |
| `delta` | 动作参数增量 |

### 11.3 `action.end`

| 字段 | 说明 |
| --- | --- |
| `actionId` | 动作唯一标识 |

### 11.4 `action.snapshot`

主要用于历史记录；不是当前 `/api/query` 实时流主路径。

| 字段 | 说明 |
| --- | --- |
| `actionId` | 动作唯一标识 |
| `runId` | 运行实例 ID |
| `actionName` | 动作名称 |
| `taskId` | 可选；任务 ID |
| `description` | 动作描述 |
| `arguments` | 动作参数 |

### 11.5 `action.result`

| 字段 | 说明 |
| --- | --- |
| `actionId` | 动作唯一标识 |
| `result` | 动作执行结果 |

## 12. Source 事件

### 12.1 `source.snapshot`

源信息块的快照事件；当前仍未实现实时 SSE 输出。

| 字段 | 说明 |
| --- | --- |
| `icon` | 来源图标 |
| `title` | 来源标题 |
| `url` | 来源地址 |

## 13. 层级关系速览

```text
chat
├── plan
├── run
│   ├── task
│   │   ├── reasoning
│   │   ├── content
│   │   ├── tool
│   │   └── action
│   └── artifact.publish
└── source(snapshot, optional)
```

## 14. 需要重点记住的边界

- `query` 对应 `request.query`
- `submit` 对应 `request.submit`
- `steer` 对应 `request.steer`
- `interrupt` 对应 `run.cancel`
- `heartbeat` 和 `data:[DONE]` 都是传输层概念，不是业务 JSON 事件
