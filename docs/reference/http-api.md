# HTTP API

本页定义 AGW UI 核心 HTTP 接口。事实源以 `agent-platform` 当前实现为准；记忆、定时任务、归档、Agent 编辑和 Admin Gateway 等平台能力见 [Platform Extensions](platform-extensions.md)。

## 1. 总体约定

- 所有公开 HTTP 接口都以 `/api` 为前缀。
- 普通 JSON 接口返回 `ApiResponse<T>`：`{ "code": number, "msg": string, "data": ... }`，其中 `code = 0` 表示成功。
- `POST /api/query` 与 `GET /api/attach` 是 SSE 接口，成功时返回 `text/event-stream`。如果请求准备阶段失败，也会返回 JSON 失败响应。
- `GET /api/resource` 成功时直接返回文件内容，不包 JSON。
- 如果启用 WebSocket，已注册的 WS route 会用同名 REST 路径作为 `request.type`，payload 与 HTTP body / query 参数语义对齐。
- HTTP 请求语义与 SSE 事件语义不是 1:1 映射：例如 `POST /api/interrupt` 的流内结果是 `run.cancel`，不会产生 `request.interrupt`。

## 2. 对话与运行

### 2.1 `POST /api/query`

发起一次对话运行。成功时直接建立 SSE 事件流。

#### `QueryRequest`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 可选；前端生成，用于幂等、重试和链路追踪；缺省等于最终 `runId` |
| `runId` | `string` | 可选；前端指定运行 ID；缺省由服务端生成 |
| `chatId` | `string` | 可选；缺省由服务端创建 chat |
| `agentKey` | `string` | 可选；缺省按 chat、team、channel 或全局默认智能体推断 |
| `teamId` | `string` | 可选；团队或租户路由信息 |
| `role` | `"user" \| "system" \| "assistant" \| "developer" \| "other"` | 可选；当前消息角色 |
| `message` | `string` | 必填；用户输入正文 |
| `references` | `Reference[]` | 可选；文件、图片、selection、截图等引用 |
| `params` | `object` | 可选；结构化输入或渠道上下文 |
| `scene` | `{url,title}` | 可选；页面上下文 |
| `stream` | `boolean` | 可选；兼容字段，当前实现成功时固定返回 SSE |
| `hidden` | `boolean` | 可选；隐藏本次请求的部分前端展示内容 |

#### 成功响应

| 项 | 说明 |
| --- | --- |
| `Content-Type` | `text/event-stream` |
| 业务帧 | `event: message` + 单行 JSON `data`，字段包含 `seq/type/timestamp` |
| heartbeat | `: heartbeat`，注释帧，不属于业务 JSON |
| 结束帧 | `event: message` + `data: [DONE]` |

边界说明：

- `request.query` 是流内回看事件，不是普通 JSON 响应体。
- `runId/requestId/chatId/agentKey/teamId` 会在服务端准备阶段补齐后进入运行上下文。
- PROXY 模式智能体可能把请求转发给上游，但对前端仍保持同一入口语义。

### 2.2 `GET /api/attach`

观察已有 run 的实时事件，或按 `lastSeq` 补流。成功时返回 SSE。

#### Query 参数

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `runId` | `string` | 必填；要观察的运行 ID |
| `lastSeq` | `number` | 可选；客户端已收到的最后一个事件序号，服务端补发 `seq > lastSeq` 的事件 |

#### 成功响应

与 `POST /api/query` 相同，返回 `text/event-stream`。流结束时发送 `[DONE]`。

#### 常见错误

| HTTP 状态 | `msg` | 说明 |
| --- | --- | --- |
| `400` | `runId is required` / `lastSeq must be a valid integer` | 参数错误 |
| `404` | `run not found` | run 不存在或已不可观察 |
| `409` | `SEQ_EXPIRED` | `lastSeq` 早于可回放窗口，`data` 中包含 `oldestSeq/latestSeq/lastSeq` |

说明：当前实现没有 `/api/run/stream` 或 `/api/run/status` 作为公开 HTTP 入口；观察运行流使用 `/api/attach`。

### 2.3 `POST /api/submit`

提交前端 HITL 交互结果。同步响应只表示接收结果，后续工具结果和模型输出仍在原 run 的事件流中出现。

#### `SubmitRequest`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `runId` | `string` | 必填；当前 run |
| `awaitingId` | `string` | 必填；当前等待态 ID |
| `params` | `SubmitParam[]` | 必填；必须是数组，可为空数组表示整批取消 |

`params[i]` 与当前 `awaiting.ask` 中的 `questions/approvals/forms[i]` 按下标对应。`id` 可选，主要用于审计和日志。

常见形态：

- question：`{"id":"q1","answer":"..."}` 或 `{"id":"q2","answers":[...]}`
- approval：`{"id":"tool_bash","decision":"approve|approve_prefix_run|reject","reason":"..."}`
- form：`{"id":"form-1","payload":{...}}`、`{"id":"form-1","reason":"..."}` 或 `{"id":"form-1"}`

#### `SubmitResponse`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `accepted` | `boolean` | 是否接受 |
| `status` | `string` | 接收状态 |
| `runId` | `string` | 当前 run |
| `awaitingId` | `string` | 当前等待态 |
| `detail` | `string` | 说明文字 |

流内边界：

- 运行流会先记录 `request.submit`，保留前端原始 `params[]`。
- 服务端归一化结果由 `awaiting.answer` 表达。
- WebSocket 形态下 `/api/submit` 返回普通 `response` 帧。

### 2.4 `POST /api/steer`

向进行中的 run 注入新的用户指令。

#### `SteerRequest`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 可选 |
| `chatId` | `string` | 可选 |
| `runId` | `string` | 必填 |
| `steerId` | `string` | 可选；缺省由服务端生成 |
| `agentKey` | `string` | 可选 |
| `teamId` | `string` | 可选 |
| `message` | `string` | 必填；追加用户指令 |
| `planningMode` | `boolean` | 可选 |

#### `SteerResponse`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `accepted` | `boolean` | 是否接受 |
| `status` | `string` | 接收状态 |
| `runId` | `string` | 当前 run |
| `steerId` | `string` | 当前 steer |
| `detail` | `string` | 说明文字 |

流内边界：同步确认来自 HTTP 响应；运行流内回看事件是 `request.steer`。

### 2.5 `POST /api/interrupt`

中断进行中的 run。

#### `InterruptRequest`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 可选 |
| `chatId` | `string` | 可选 |
| `runId` | `string` | 必填 |
| `agentKey` | `string` | 可选 |
| `teamId` | `string` | 可选 |
| `message` | `string` | 可选 |
| `planningMode` | `boolean` | 可选 |

#### `InterruptResponse`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `accepted` | `boolean` | 是否接受 |
| `status` | `string` | 接收状态 |
| `runId` | `string` | 当前 run |
| `detail` | `string` | 说明文字 |

流内边界：当前实现不会发 `request.interrupt`；真实停止以原事件流中的 `run.cancel` 或 WS stream 终止原因为准。

## 3. Catalog 与工具

### 3.1 `GET /api/agents`

获取智能体列表。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `tag` | `string` | 可选；按标签过滤 |
| `channel` | `string` | 可选；按渠道过滤允许的智能体 |

响应 `data` 为 `AgentSummary[]`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `key` | `string` | Agent 唯一键 |
| `name` | `string` | 展示名 |
| `icon` | `any` | 可选 |
| `description` | `string` | 可选 |
| `role` | `string` | 可选 |
| `stats.totalCount` | `number` | 会话总数 |
| `stats.unreadCount` | `number` | 未读会话数 |
| `meta` | `object` | 可选 |

### 3.2 `GET /api/channels`

获取渠道列表。

响应 `data` 为 `ChannelSummary[]`，核心字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | `string` | 渠道 ID |
| `name` | `string` | 展示名 |
| `type` | `string` | 渠道类型 |
| `defaultAgent` | `string` | 可选；默认智能体 |
| `agents` | `string[]` | 允许的智能体 key |
| `connected` | `boolean` | 当前连接状态 |

### 3.3 `GET /api/agent`

获取单个智能体详情。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `agentKey` | `string` | 必填 |

响应 `data` 为 `AgentDetailResponse`，核心字段包括 `key/name/icon/description/role/wonders/model/mode/tools/skills/controls/meta`。如果智能体可编辑，还可能返回 `definition/soulPrompt/agentsPrompt/source`。

### 3.4 `GET /api/teams`

获取团队列表。响应 `data` 为 `TeamSummary[]`，核心字段包括 `teamId/name/icon/agentKeys/meta`。

### 3.5 `GET /api/skills`

获取技能列表。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `tag` | `string` | 可选；按标签过滤 |

响应 `data` 为 `SkillSummary[]`，核心字段包括 `key/name/description/meta`。

### 3.6 `GET /api/tools`

获取工具列表。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `kind` | `string` | 可选；按工具种类过滤 |
| `tag` | `string` | 可选；按标签过滤 |

响应 `data` 为 `ToolSummary[]`，核心字段包括 `key/name/label/description/meta`。

### 3.7 `GET /api/tool`

获取单个工具详情。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `toolName` | `string` | 必填 |

响应 `data` 为 `ToolDetailResponse`，核心字段包括 `key/name/label/description/afterCallHint/parameters/meta`。

## 4. Chat、搜索与反馈

### 4.1 `GET /api/chats`

获取会话列表。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `lastRunId` | `string` | 可选；分页或增量列表游标 |
| `agentKey` | `string` | 可选；按智能体过滤 |

响应 `data` 为 `ChatSummaryResponse[]`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 会话 ID |
| `chatName` | `string` | 会话名称 |
| `agentKey` | `string` | 可选 |
| `teamId` | `string` | 可选 |
| `createdAt` / `updatedAt` | `number` | 毫秒时间戳 |
| `lastRunId` | `string` | 可选 |
| `lastRunContent` | `string` | 可选 |
| `read` | `ChatReadState` | 已读状态 |
| `awaiting` | `Awaiting` | 可选；当前待处理 HITL 状态摘要 |
| `usage` | `ChatUsageData` | 可选；聚合 token 用量 |

### 4.2 `GET /api/chat`

获取单个会话详情。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 必填 |
| `includeRawMessages` | `boolean` | 可选；为 `true` 时返回模型原始消息 |

响应 `data` 为 `ChatDetailResponse`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 会话 ID |
| `chatName` | `string` | 会话名称 |
| `resourceTicket` | `string` | 可选；已鉴权用户访问资源的临时票据 |
| `rawMessages` | `object[]` | 可选；仅 `includeRawMessages=true` |
| `events` | `BaseEvent[]` | 历史事件列表 |
| `runs` | `RunSummary[]` | run 摘要列表 |
| `activeRun` | `ActiveRunInfo` | 可选；仍在进行中的 run，含 `runId/state/lastSeq/oldestSeq/startedAt` |
| `plan` | `object` | 可选；chat 当前计划聚合态 |
| `artifact` | `object` | 可选；chat 当前产物聚合态 |
| `references` | `Reference[]` | 可选；引用池 |
| `usage` | `ChatUsageData` | 可选；聚合 token 用量 |

历史事件折叠规则：

- `start/delta/args/end` 类事件在历史中可能折叠为 `snapshot`。
- `plan.update` 可折叠为 chat 级 `plan`。
- `artifact.publish` 可折叠为 chat 级 `artifact` 聚合态。
- 如果 `activeRun` 存在，当前实现会避免把该 run 的合成 `run.complete` 当作历史完成态返回。

### 4.3 `POST /api/read`

标记单个会话或某个智能体下全部会话为已读。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 可选；指定会话 |
| `runId` | `string` | 可选；指定读到的 run |
| `agentKey` | `string` | 可选；当 `chatId` 为空时，按智能体全部标记已读 |

响应 `data` 为 `MarkChatReadResponse`，包含 `chatId/agentKey/lastRunId/read/agentUnreadCount/updatedCount`。

### 4.4 `POST /api/search`

全局搜索会话内容。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `query` | `string` | 必填 |
| `agentKey` | `string` | 可选 |
| `teamId` | `string` | 可选 |
| `limit` | `number` | 可选；缺省为 20 |

响应 `data` 为 `{ query, count, results }`，`results[]` 包含 `chatId/chatName/agentKey/teamId/runId/kind/role/timestamp/snippet/score`。

### 4.5 `POST /api/session-search`

在单个 chat 内搜索。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 必填 |
| `query` | `string` | 必填 |
| `limit` | `number` | 可选 |

响应 `data` 为 `{ chatId, query, count, results }`，`results[]` 包含 `kind/chatId/runId/stage/role/timestamp/snippet/score/meta`。

### 4.6 `POST /api/feedback`

设置或清除 run 反馈。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 必填 |
| `runId` | `string` | 必填 |
| `type` | `"thumbs_down" \| "clear"` | 必填 |
| `comment` | `string` | 可选 |

响应 `data` 为 `{ chatId, runId, type, setAt }`。

## 5. 资源与上传

### 5.1 `GET /api/viewport`

获取工具或表单对应的视图 payload。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `viewportKey` | `string` | 必填 |

响应 `data` 为视图 payload。本地 HTML viewport 通常包装为 `{ "html": "..." }`。

### 5.2 `GET /api/resource`

读取 chat 资源、附件或产物。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `file` | `string` | 必填；相对 data 根目录的文件路径，兼容 `/data/...` 形式 |
| `t` | `string` | 可选；resource ticket。启用资源票据且没有 Bearer principal 时必填 |

成功时直接返回文件内容。失败时返回 JSON 失败响应。

### 5.3 `POST /api/upload`

上传浏览器本地文件。请求为 `multipart/form-data`。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 可选；缺省由服务端生成 |
| `chatId` | `string` | 可选；缺省由服务端创建 chat |
| `agentKey` | `string` | 可选；创建 chat 时使用 |
| `file` | `multipart file` | 必填；实际实现会接受 multipart 中的第一个文件字段 |

响应 `data` 为 `UploadResponse`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 上传请求 ID |
| `chatId` | `string` | 所属 chat |
| `upload.id` | `string` | chat 内短 ID，如 `r01` |
| `upload.type` | `string` | 当前实现保存为 `"file"` |
| `upload.name` | `string` | 安全化后的文件名 |
| `upload.mimeType` | `string` | MIME 类型 |
| `upload.sizeBytes` | `number` | 文件大小 |
| `upload.url` | `string` | `/api/resource?file=...` |
| `upload.sha256` | `string` | SHA-256 |
| `upload.sandboxPath` | `string` | 沙箱内建议路径，如 `/workspace/name.ext` |

上传完成后，前端可把 `upload` 映射为 `Reference`，并在后续 `query` 中通过 `#{{refid}}` 或 `references[]` 使用。

## 6. HTTP 与 WS 对照

| HTTP 路径 | HTTP 成功形态 | WS 成功形态 |
| --- | --- | --- |
| `POST /api/query` | SSE | `stream` |
| `GET /api/attach` | SSE | `stream` |
| `POST /api/submit` | JSON | `response` |
| `POST /api/steer` | JSON | `response` |
| `POST /api/interrupt` | JSON | `response` |
| catalog/chat/search/feedback/viewport | JSON | 已注册 WS route 返回 `response`；`/api/session-search` 当前仅 HTTP |
| `GET /api/resource` | 文件内容 | `response`，用于网关/资源协商，不是二进制直传 |
| `POST /api/upload` | multipart 上传 + JSON | WS 形态用于网关下载/拉取协商；浏览器文件上传仍优先使用 HTTP |
