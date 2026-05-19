# WebSocket 协议

WebSocket 是 AGW 的可选传输层扩展，用于在一条长连接上复用普通请求、运行流观察和服务端主动推送。它不替代 HTTP API 和 SSE 事件语义；业务事件仍以 [SSE 事件模型](sse-events.md) 中的 `EventData` 为准。

## 1. 连接入口

```http
GET /ws
```

认证方式：

- 推荐：`Sec-WebSocket-Protocol: bearer.<JWT>`
- 兼容：`GET /ws?token=<JWT>`

说明：

- 服务端启用鉴权时，握手阶段校验 token；失败返回 `401`，不会升级连接。
- 服务端未启用鉴权时，当前实现允许空 token 通过。
- 如果使用 `bearer.<JWT>` 子协议，服务端会在握手响应中回显同一 `Sec-WebSocket-Protocol`。

## 2. 帧模型

所有业务消息均为 JSON 文本帧，使用 `frame` 区分类型：

```json
{ "frame": "request" | "response" | "stream" | "push" | "error" }
```

方向约定：

- 客户端只发送 `request`。
- 服务端发送 `response`、`stream`、`push`、`error`。
- 非 `request` 客户端帧会被拒绝；反向网关 silent 连接是内部兼容模式，不作为普通前端协议。

### 2.1 `request`

客户端向服务端发起业务请求：

```json
{
  "frame": "request",
  "type": "/api/query",
  "id": "req_001",
  "payload": {
    "message": "请总结当前协议"
  }
}
```

| 字段 | 说明 |
| --- | --- |
| `frame` | 固定为 `"request"` |
| `type` | 复用 REST 路径，如 `/api/query`、`/api/attach`、`/api/submit` |
| `id` | 必填；同一连接内 in-flight 请求唯一 |
| `payload` | 可选；与对应 HTTP body / query 参数语义一致 |

### 2.2 `response`

普通请求的同步响应：

```json
{
  "frame": "response",
  "type": "/api/submit",
  "id": "req_003",
  "code": 0,
  "msg": "success",
  "data": {
    "accepted": true,
    "status": "accepted",
    "runId": "run_001",
    "awaitingId": "await_001"
  }
}
```

`code/msg/data` 与 HTTP `ApiResponse<T>` 语义保持一致。

### 2.3 `stream`

运行流事件帧：

```json
{
  "frame": "stream",
  "id": "req_001",
  "streamId": "s_1",
  "event": {
    "seq": 12,
    "type": "content.delta",
    "contentId": "c_1",
    "delta": "hello",
    "timestamp": 1707000002000
  }
}
```

| 字段 | 说明 |
| --- | --- |
| `id` | 对应发起 stream 的 request id |
| `streamId` | 连接内传输层流 ID，不等于业务 `runId` |
| `event` | `EventData`；字段与 SSE 业务事件一致 |

### 2.4 `stream` 终止帧

```json
{
  "frame": "stream",
  "id": "req_001",
  "streamId": "s_1",
  "reason": "done",
  "lastSeq": 127
}
```

终止原因：

| `reason` | 说明 |
| --- | --- |
| `done` | 收到 `run.complete` 后正常结束 |
| `error` | 收到 `run.error` 后结束 |
| `cancelled` | 收到 `run.cancel` 后结束 |
| `expired` | 收到 `run.expired` 后结束 |
| `detached` | 观察者被分离或连接结束 |

WebSocket 下不使用 SSE 的 `[DONE]`。

### 2.5 `push`

服务端主动推送连接级或广播级通知：

```json
{
  "frame": "push",
  "type": "heartbeat",
  "data": {
    "timestamp": 1707000002000
  }
}
```

当前实现中常见 `push.type`：

| 类型 | 说明 |
| --- | --- |
| `connected` | 连接建立后发送，`data.sessionId` 为连接会话 ID |
| `heartbeat` | 周期保活，`data.timestamp` 为毫秒时间戳 |
| `auth.expiring` | token 即将过期，`data.expiresAt` 为毫秒时间戳 |
| `chat.created` | chat 创建 |
| `chat.updated` | chat 更新 |
| `chat.read` | 单个 chat 已读状态变化 |
| `chat.unread` | chat 未读状态变化 |
| `chat.read_all` | 某智能体下会话批量已读 |
| `chat.deleted` | chat 删除 |
| `run.started` | run 启动 |
| `run.finished` | run 完成或结束 |

### 2.6 `error`

```json
{
  "frame": "error",
  "type": "invalid_request",
  "id": "req_009",
  "code": 400,
  "msg": "unknown type: /api/missing",
  "data": {}
}
```

常见错误类型：

| `type` | `code` | 说明 |
| --- | --- | --- |
| `invalid_request` | `400` | JSON、frame、id、type 或 payload 错误 |
| `unauthorized` | `401` | token 无效或刷新失败 |
| `not_found` / `run_not_found` | `404` | 资源或 run 不存在 |
| `duplicate_id` | `409` | 同连接内 request id 正在处理 |
| `duplicate_observe` | `409` | 同连接重复观察同一个 run |
| `SEQ_EXPIRED` | `409` | `lastSeq` 超出可回放窗口 |
| `active_run_conflict` | `409` | chat 存在多个或冲突的活跃 run |
| `too_many_streams` | `429` | 超过单连接最大观察流数量 |
| `internal_error` | `500` | 服务端内部错误 |

## 3. 核心请求映射

| `request.type` | payload | 成功返回 | 说明 |
| --- | --- | --- | --- |
| `/api/query` | `QueryRequest` | `stream` | 创建 run 并转发 live 事件 |
| `/api/attach` | `{ "runId": "...", "lastSeq": 0 }` | `stream` | 观察已有 run，按 `lastSeq` 补流 |
| `/api/submit` | `SubmitRequest` | `response` | 同步确认提交；后续结果仍在原 stream 中出现 |
| `/api/steer` | `SteerRequest` | `response` | 同步确认追加指令；后续结果仍在原 stream 中出现 |
| `/api/interrupt` | `InterruptRequest` | `response` | 同步确认中断；真实停止以 `run.cancel` 或 stream 终止帧为准 |
| `/api/agents` | `{tag?,channel?}` | `response` | Agent 列表 |
| `/api/channels` | `{}` | `response` | 渠道列表 |
| `/api/agent` | `{agentKey}` | `response` | Agent 详情 |
| `/api/teams` | `{}` | `response` | 团队列表 |
| `/api/skills` | `{tag?}` | `response` | 技能列表 |
| `/api/tools` | `{kind?,tag?}` | `response` | 工具列表 |
| `/api/tool` | `{toolName}` | `response` | 工具详情 |
| `/api/chats` | `{lastRunId?,agentKey?}` | `response` | 会话列表 |
| `/api/chat` | `{chatId,includeRawMessages?}` | `response` | 会话详情 |
| `/api/read` | `{chatId?,runId?,agentKey?}` | `response` | 标记已读 |
| `/api/search` | `{query,agentKey?,teamId?,limit?}` | `response` | 全局搜索 |
| `/api/feedback` | `{chatId,runId,type,comment?}` | `response` | run 反馈 |
| `/api/viewport` | `{viewportKey}` | `response` | 获取视图 payload |

说明：

- 当前实现的观察入口是 `/api/attach`，不是 `/api/run/stream`。
- 当前实现没有公开 `/api/run/status` WS route；如需状态，优先通过 `/api/chat` 的 `activeRun` 或流事件判断。
- 当前实现没有注册 `/api/session-search` WS route；会话内搜索使用 HTTP `POST /api/session-search`。
- 浏览器本地文件上传仍优先使用 HTTP `POST /api/upload`；WS 下 `/api/upload` 和 `/api/pull` 主要用于网关资源拉取/下载协商。

## 4. 平台扩展 WS 映射

以下扩展 route 也已注册为 WS request，可按普通 `response` 使用；接口语义见 [Platform Extensions](platform-extensions.md)。

```text
/api/agent-create
/api/agent-update
/api/agent-delete
/api/agent-editor-options
/api/chat-delete
/api/chat-archive
/api/archives
/api/archive
/api/archive-search
/api/archive-delete
/api/schedules
/api/schedule
/api/schedule-create
/api/schedule-update
/api/schedule-delete
/api/schedule-toggle
/api/schedule-executions
/api/remember
/api/learn
/api/memory/meta
/api/memory/context/preview
/api/memory/scopes
/api/memory/scope
/api/memory/scope/validate
/api/memory/records
/api/memory/record
/api/resource
/api/upload
/api/pull
```

## 5. 连接约束

- 客户端 request `id` 必填。
- 同一连接内，未完成请求的 `id` 不能重复。
- 同一连接内，不能重复观察同一个 `runId`。
- 单连接可观察的 stream 数受服务端 `MaxObservesPerConn` 限制。
- `lastSeq` 表示客户端已收到的最后一个序号，补流只发送 `seq > lastSeq`。
- 如果写队列满，服务端会关闭连接。
- 服务端会周期发送 WebSocket ping 控制帧，并期待 pong 更新读超时。

## 6. 与 SSE 的关系

共享部分：

- `request.* / run.* / content.* / tool.* / awaiting.*` 等业务事件语义。
- `seq/type/timestamp` 业务事件包络。
- `submit/steer/interrupt` 的运行行为。

差异部分：

- SSE 使用 `event: message`、`: heartbeat`、`data: [DONE]`。
- WebSocket 使用 `stream` 事件帧、`stream` 终止帧、`push` 和 `error`。
- WebSocket 支持同一连接上多个 stream 复用。
