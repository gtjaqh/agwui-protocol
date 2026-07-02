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

进入等待态的事件，直接内联前端需要渲染的 question / approval / form / plan 定义。

| 字段 | 说明 |
| --- | --- |
| `awaitingId` | 当前等待态 ID |
| `mode` | `"question" \| "approval" \| "form" \| "plan"` |
| `viewportType` | 可选；question / approval / plan 默认 `"builtin"`，form 默认 `"html"` |
| `viewportKey` | 可选；question / approval / plan 默认同名 key，form 可由输入提供，可配合 `GET /api/viewport` |
| `timeout` | 可选；等待超时秒数 |
| `runId` | 当前 run |
| `questions` | question 模式下出现；item 可声明 `type`，内置类型包括 `text/password/number/select/multi-select/date/datetime` |
| `approvals` | approval 模式下出现 |
| `forms` | form 模式下出现；item 主字段是 `id`、可选 `title/toolName/command/form`，其中 `form` 是初始表单对象 |
| `plan` | plan 模式下出现；单个对象 |

说明：

- 当前协议统一使用 `mode`，不以 `kind` 作为对外主字段。
- question / approval / form / plan 的交互定义都直接位于 `awaiting.ask`。
- question 中除 `multi-select` 外均提交单值 `answer`；`date` 推荐 `YYYY-MM-DD`，`datetime` 推荐 `YYYY-MM-DDTHH:mm`，服务端按非空字符串透传。
- `approvals[]` 只服务工具/HITL 审批；CODER planning 确认使用 `mode="plan"` 和单个 `plan` 对象。
- `viewportKey` 是视图 payload 的检索键，视图相关信息应从 `awaiting.ask` 或 `/api/viewport` 获取。

### 9.2 `awaiting.answer`

服务端处理提交后的归一化结果事件。

| 字段 | 说明 |
| --- | --- |
| `awaitingId` | 当前等待态 ID |
| `mode` | `"question" \| "approval" \| "form" \| "plan"` |
| `status` | 可选；归一化状态 |
| `answers` | question 模式下出现；item 含 `id/question/header/answer`，`date/datetime` 的 `answer` 为提交的非空字符串 |
| `approvals` | approval 模式下出现 |
| `forms` | form 模式下出现；item 含 `id`、可选 `command`、`decision`、可选 `form/reason` |
| `plan` | plan 模式下出现；含 `decision`、可选 `id/planningId/reason` |
| `error` | 可选；处理错误 |

说明：`request.submit` 记录原始 `params[]`，`awaiting.answer` 记录服务端归一化结构。form 推荐提交 `form` 字段；兼容型 generic frontend tool 可接受 `payload/value/answer` 作为表单对象来源。`mode=plan` 固定只接受 1 个提交项，归一化结果写入 `awaiting.answer.plan`，`decision` 只能是 `approve` 或 `reject`。

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

发布来源信息块，用于把引用、检索命中或可展示来源卡片独立于正文和 `tool.result` 下发。客户端可按 `toolId` 把来源信息展示在对应工具结果附近，也可按 `runId` 汇总展示。

| 字段 | 说明 |
| --- | --- |
| `publishId` | 发布 ID |
| `runId` | 当前 run |
| `taskId` | 可选；所属 task |
| `toolId` | 可选；关联工具调用 ID |
| `kind` | 来源类型，如 `kbase`、`websearch`、`ragflow`、`local` 等 |
| `query` | 可选；产生来源的查询文本 |
| `sourceCount` | source 数量，通常等于 `sources.length` |
| `chunkCount` | chunk 总数，通常等于所有 `sources[].chunks.length` 之和 |
| `sources` | 来源列表；空结果时为 `[]` |

`sources[]` 表示来源聚合单元。

| `sources[]` 字段 | 说明 |
| --- | --- |
| `id` | 来源 ID |
| `name` | 展示名称 |
| `title` | 可选；标题、路径或更完整的展示名称 |
| `icon` | 可选；来源图标标识 |
| `url` | 可选；可打开 URL |
| `link` | 可选；兼容型链接字段 |
| `collectionId` | 可选；集合 ID |
| `collectionName` | 可选；集合名称 |
| `chunkIndexes` | 当前 source 下 chunk 的 `index` 列表 |
| `minIndex` | 当前 source 下最小 chunk `index` |
| `chunks` | 命中片段列表 |

`chunks[]` 描述可展示和可定位的命中片段。

| `chunks[]` 字段 | 说明 |
| --- | --- |
| `chunkId` | chunk ID |
| `index` | 本次发布内的 chunk 序号 |
| `content` | 片段内容 |
| `score` | 可选；相关性分数 |
| `timestamp` | 可选；片段时间戳 |
| `path` | 可选；文件或资源路径 |
| `heading` | 可选；标题或章节 |
| `startLine` / `endLine` | 可选；文本行号范围 |
| `pageStart` / `pageEnd` | 可选；页码范围 |
| `slideStart` / `slideEnd` | 可选；幻灯片页范围 |
| `sourceType` | 可选；来源文件或资源类型 |
| `matchType` | 可选；匹配类型 |

示例：

```text
event: message
data: {"seq":14,"type":"source.publish","publishId":"src-a1b2c3","runId":"run_1","toolId":"call_1","kind":"kbase","query":"报销流程","sourceCount":1,"chunkCount":2,"sources":[{"id":"kbase:docs/policy.md","name":"policy.md","title":"docs/policy.md","icon":"kbase","collectionName":"KBASE","chunkIndexes":[1,2],"minIndex":1,"chunks":[{"chunkId":"chunk_1","index":1,"content":"报销申请需要提交发票。","path":"docs/policy.md","startLine":12,"endLine":14,"sourceType":"markdown"},{"chunkId":"chunk_2","index":2,"content":"审批通过后进入付款流程。","score":0.82,"path":"docs/policy.md","startLine":30,"endLine":33,"sourceType":"markdown","matchType":"semantic"}]}],"timestamp":1707000005000}
```

事件顺序建议：

- 与工具调用有关的来源信息，建议在对应 `tool.result` 之后发送。
- 与整体回答有关的来源信息，可以仅关联 `runId`。

本协议不定义 `source.snapshot` 作为默认实时事件名。

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
│   │   ├── planning
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
