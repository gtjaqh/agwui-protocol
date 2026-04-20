# HTTP API

## 1. 总体约定

- 所有公开接口均以 `/api` 为前缀。
- 动作类请求建议使用 `POST`，资源类获取建议使用 `GET`。
- `query` 是特殊入口：`POST /api/query` 直接返回 `text/event-stream`，不会返回独立 JSON 响应体。
- 除 `POST /api/query` 与 `GET /api/resource` 外，当前接口默认返回 JSON 包装：`{ "code": number, "msg": string, "data": ... }`。
- `viewport` 是历史 `ViewRequest / ViewResponse` 的统一收敛命名，当前规范使用 `GET /api/viewport`。
- HTTP API 与 SSE 事件层不是同一层协议。`request.*` 是流内回看事件，不等于公开 HTTP body。
- 如果服务端启用 WebSocket，建议直接沿用原始 REST 路径作为 WS `type`；HTTP 仍是公开请求语义的基线定义。

## 2. 动作类接口

### 2.1 `POST /api/query`

发起一次对话请求；当前实现固定返回 SSE 实时事件流。

#### `QueryRequest`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 前端生成，用于幂等、重试和链路追踪；前端不填时会等于 `runId` |
| `chatId` | `string(UUID)` | 可选；缺省则服务端创建 chat |
| `agentKey` | `string` | 可选；如 `auto`、`default`、`agent_researcher` |
| `teamId` | `string` | 可选；团队或租户路由信息 |
| `role` | `"user" \| "system" \| "assistant" \| "developer" \| "other"` | 当前消息角色 |
| `message` | `string` | 必填；用户输入正文 |
| `references` | `Reference[]` | 可选；本次请求携带的文件、图片、selection、截图等引用 |
| `params` | `object` | 可选；结构化输入，如 `input/select/datepicker/switch` |
| `scene` | `object` | 可选；页面上下文，通常以 `url` 为主 |
| `scene.url` | `string` | 建议提供 |
| `scene.title` | `string` | 可选 |
| `stream` | `boolean` | 可选；保留作兼容字段，当前实现固定返回 SSE |
| `hidden` | `boolean` | 可选；隐藏本次请求的部分前端展示内容 |

#### `QueryResponse`

当前实现不会返回独立 JSON 响应体；`POST /api/query` 直接返回 `text/event-stream`。

| 项 | 说明 |
| --- | --- |
| `Content-Type` | `text/event-stream` |
| `business frames` | 业务事件统一使用 `event: message`，`data` 为单行 JSON，包含 `seq/type/timestamp` |
| `heartbeat` | 传输层保活时会出现 `: heartbeat`，注释帧，不带 JSON `data` |
| `done sentinel` | 流结束时发送 `event: message` + `data: [DONE]`；它不是业务事件，不会进入历史 `events` |

#### 边界说明

- `request.query` 是 SSE 流里的回看事件，不是公开响应对象。
- 当前前端应把 `query` 理解为“建立执行流”的入口，而不是“拿一份 JSON 结果”的接口。
- `reasoning.snapshot`、`content.snapshot`、`tool.snapshot`、`action.snapshot` 不走 live SSE，只进入历史事件持久化。
- 如果改走 WebSocket，请求 payload 仍与 `QueryRequest` 对齐，但 live 输出改由 `stream` 帧承载，见 [WebSocket 协议](websocket-protocol.md)。

### 2.2 `POST /api/upload`

浏览器本地文件一步上传；上传完成后，前端可把返回的 `upload` 映射为 `Reference` 参与后续 `query`。

#### `UploadRequest`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 前端生成，用于幂等和重试 |
| `chatId` | `string(UUID)` | 可选；缺省则服务端创建 chat |
| `file` | `multipart file` | 必填 |
| `sha256` | `string` | 可选 |

#### `UploadResponse`

外层通常仍为统一 JSON 包装，核心数据为 `upload`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 对应上传请求 ID |
| `chatId` | `string(UUID)` | 所属 chat |
| `upload.id` | `string` | 必填；chat 内短 ID，如 `r01` |
| `upload.type` | `"file" \| "image"` | 上传类型 |
| `upload.name` | `string` | 文件名 |
| `upload.mimeType` | `string` | MIME 类型 |
| `upload.sizeBytes` | `number` | 文件大小 |
| `upload.url` | `string` | 必填；上传后的资源地址 |
| `upload.sha256` | `string` | 可选；校验或去重用 |

#### 边界说明

- 上传成功后，前端可以把 `upload` 映射到本地引用池。
- 后续 `query` 可通过 `#{{refid}}` 或 `references[]` 使用该上传资源。

### 2.3 `POST /api/submit`

提交前端工具交互结果。注意：这是公开 HTTP body，不等于流内的 `request.submit` 事件结构。

#### `SubmitRequest`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `runId` | `string` | 当前运行 ID |
| `awaitingId` | `string` | 必填；当前交互关联的 awaiting / tool |
| `params` | `SubmitParam[]` | 必填；顶层始终是数组，按顺序对应 `awaiting.ask` 当前批次里的 item |

#### `SubmitParam`

`params[i]` 与 `awaiting.ask.questions|approvals|forms[i]` 固定按下标对应。`id` 可选，但仅作审计/日志用途，服务端不会据此路由。

常见形态：

- `question`：`{"id":"q1","answer":"..."}` 或 `{"id":"q2","answers":[...]}`，两者必须二选一
- `approval`：`{"id":"tool_bash","decision":"approve|approve_prefix_run|reject","reason":"..."}`
- `form`：`{"id":"form-1","payload":{...}}`、`{"id":"form-1","reason":"..."}` 或 `{"id":"form-1"}`

补充：

- question / approval / form 都允许 `params: []` 表示整批取消
- form 是唯一与 `viewportType:"html"`、`viewportKey`、`viewportPayload` 配套出现的 HITL 形态

#### `SubmitResponse`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `accepted` | `boolean` | 是否接受 |
| `status` | `string` | 接收状态 |
| `runId` | `string` | 当前 run |
| `awaitingId` | `string` | 当前 awaiting |
| `detail` | `string` | 说明文字 |

#### 边界说明

- `submit` 是同步确认接口。
- 服务端按 `awaitingId` 反查当前等待态，不要求前端重复提交 `mode`。
- 工具执行结果与最终回答仍走原 SSE 流，常见表现是 `tool.result` 或后续 `content.*`。
- 运行流里会先记录 `request.submit`，透传前端原始 `params[]`；随后再通过 `awaiting.answer` 记录服务端归一化后的结构化结果。
- WebSocket 形态下，`/api/submit` 也返回普通 `response` 帧，而不是 `stream`。

### 2.4 `POST /api/steer`

向进行中的 run 注入新的用户指令。注意：这是公开 HTTP body；流里对应的回看事件是 `request.steer`，两者不应混用。

#### `SteerRequest`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 可选 |
| `chatId` | `string(UUID)` | 可选 |
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

#### 边界说明

- `steer` 的同步确认来自 HTTP 响应。
- 后续处理结果继续走原 SSE 流。
- 流层记录为 `request.steer`，这是“回看事件”，不是公开 HTTP schema 的镜像对象。
- WebSocket 形态下，`/api/steer` 对应普通 `response` 帧。

### 2.5 `POST /api/interrupt`

中断进行中的 run。注意：公开 API 名称是 `interrupt`，但流层不会产生同名的 `request.*` 事件；中断结果由 `run.cancel` 表达。

#### `InterruptRequest`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 可选 |
| `chatId` | `string(UUID)` | 可选 |
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

#### 边界说明

- `interrupt` 的同步确认来自 HTTP 响应。
- 实际停止以原 SSE 流中的 `run.cancel` 为准。
- 当前流层不会发独立的 `request.interrupt`。
- WebSocket 形态下，`/api/interrupt` 对应普通 `response` 帧。

## 3. 资源类接口

说明：上传与资源下载继续保留在 HTTP，不迁移到 WebSocket。

### 3.1 `GET /api/viewport`

获取工具或渲染对应的视图（表单、UI、HTML 等）；`viewportKey` 通常来自 `awaiting.ask` 或其他业务约定。

#### `ViewportRequest`

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `viewportKey` | `string` | 必填 |

#### `ViewportResponse`

返回统一 `ApiResponse<Object>`，其中 `data` 即视图 payload。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `code` | `number` | 状态码 |
| `msg` | `string` | 消息 |
| `data` | `any` | 视图载荷 |

#### 边界说明

- 本地 HTML viewport 会包装为 `{ "html": "..." }`。
- 其他类型通常直接透传 payload。
- 历史文档中使用 `ViewRequest / ViewResponse`，当前统一收敛为 `viewport`。

### 3.2 `GET /api/resource`

获取资源（含附件、图片、产物）。当前接口成功时直接返回文件内容，而不是 JSON 包装。

#### `ResourceRequest`

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `file` | `string` | 必填；相对 data 根目录的文件路径，兼容 `/data/...` 形式 |
| `download` | `boolean` | 可选；为 `true` 时强制下载 |
| `t` | `string` | 可选；resource ticket |

#### `ResourceResponse`

| 项 | 说明 |
| --- | --- |
| `Content-Type` | 由文件名推断 MIME |
| `Content-Disposition` | 图片默认 `inline`，其余默认 `attachment` |
| `Cache-Control` | `private, no-store` |

失败时通常返回统一 `ApiResponse.failure`。

### 3.3 `GET /api/agents`

获取智能体列表，可用于前端选择或展示。

#### `AgentsRequest`

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `includeHidden` | `boolean` | 可选；暂未实现 |
| `tag` | `string` | 可选；按标签过滤 |

#### `AgentsResponse`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `agents` | `object[]` | 必填；智能体列表 |
| `agents[].key` | `string` | 如 `agent_researcher` |
| `agents[].name` | `string` | 展示名 |
| `agents[].description` | `string` | 可选 |
| `agents[].capabilities` | `string[]` | 可选；暂未实现 |
| `agents[].meta` | `object` | 可选；暂未实现 |

### 3.4 `GET /api/agent`

获取某个智能体的内容或详情（配置、说明、能力等）。

#### `AgentRequest`

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `agentKey` | `string` | 必填 |

#### `AgentResponse`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `agent` | `object` | 必填；单个智能体详情 |
| `agent.key` | `string` | Agent 唯一键 |
| `agent.name` | `string` | 展示名 |
| `agent.description` | `string` | 可选 |
| `agent.instructions` | `string` | 可选 |
| `agent.capabilities` | `string[]` | 可选；暂未实现 |
| `agent.meta` | `object` | 可选；暂未实现 |

### 3.5 `GET /api/chats`

获取聊天列表（历史会话列表）。

#### `ChatsRequest`

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `cursor` | `string` | 可选；分页游标 |
| `limit` | `number` | 可选 |
| `q` | `string` | 可选；按名称或内容检索 |

#### `ChatsResponse`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `items` | `object[]` | 必填；聊天列表 |
| `items[].chatId` | `string(UUID)` | 会话 ID |
| `items[].chatName` | `string` | 会话名称，可选 |
| `items[].updatedAt` | `number(timestamp)` | 更新时间，可选 |
| `nextCursor` | `string` | 下一页游标，可选 |

### 3.6 `GET /api/chat`

获取聊天内容（会话详情、历史消息、关联 run 等）。

#### `ChatRequest`

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string(UUID)` | 必填 |
| `includeEvents` | `boolean` | 可选；是否返回事件流快照 |
| `cursor` | `string` | 可选；分页消息 |
| `limit` | `number` | 可选 |

#### `ChatResponse`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string(UUID)` | 会话 ID |
| `chatName` | `string` | 会话名称，可选 |
| `chatImageToken` | `string` | 可选；访问当前 chat 作用域内图片资源的 token |
| `rawMessages` | `object[]` | 可选；若 `includeRawMessages=true` 可返回模型原始消息 |
| `plan` | `object` | 可选；chat 当前最新计划状态，形如 `{planId,tasks}` |
| `artifact` | `object` | 可选；chat 当前最新产物聚合状态，形如 `{items:[...]}` |
| `references` | `object[]` | 可选；query 引用与 chat 资产聚合后的引用池 |
| `events` | `BaseEvent[]` | 历史事件列表 |

#### 历史事件折叠规则

- 历史 `events` 中，`start/delta/args/end` 通常会折叠为 `snapshot`，并保留 `params/result`。
- `plan.update` 不再出现在历史 `events` 中，而是折叠为 chat 级 `plan`。
- `artifact.publish` 不再出现在历史 `events` 中，而是折叠为 chat 级 `artifact` 聚合态。

## 4. 重点边界

- `query -> SSE`：`POST /api/query` 直接返回事件流。
- `steer -> request.steer`：`POST /api/steer` 的运行回看事件是 `request.steer`。
- `interrupt -> run.cancel`：`POST /api/interrupt` 的流层结果是 `run.cancel`，而不是 `request.interrupt`。
