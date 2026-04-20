# WebSocket 协议

## 1. 定位

WebSocket 是 AGW 的可选传输层扩展，用于在一条长连接上复用：

- 公开业务请求
- 流式事件转发
- 服务端主动推送

它不替代现有 HTTP API 和 SSE 事件语义，只是提供另一种承载方式。

默认理解方式仍然是：

- HTTP 定义公开请求语义
- SSE / WebSocket 都可以承载 live 事件

## 2. 连接入口

```http
GET /ws
```

推荐认证方式：

- `Sec-WebSocket-Protocol: bearer.<JWT>`

兼容方式：

- `GET /ws?token=<JWT>`

服务端在握手阶段校验 JWT，失败返回 `401`，不会升级连接。

## 3. 帧模型

所有消息均为 JSON 文本帧，使用 `frame` 区分类型：

```json
{ "frame": "request" | "response" | "stream" | "push" | "error" }
```

方向约定：

- 客户端只发送 `request`
- 服务端发送 `response`、`stream`、`push`、`error`

### 3.1 `request`

客户端向服务端发起业务请求：

```json
{
  "frame": "request",
  "type": "/api/query",
  "id": "req_001",
  "payload": {
    "message": "请总结最新的 HITL 协议变化"
  }
}
```

说明：

- `type` 直接复用原始 REST 路径，如 `/api/query`、`/api/submit`
- `id` 是同一连接内的请求唯一标识
- `payload` 与对应 HTTP request body / query 参数语义一致；`/api/submit` 时仍然是 `runId + awaitingId + params[]`

### 3.2 `response`

服务端对普通请求返回统一响应：

```json
{
  "frame": "response",
  "type": "/api/submit",
  "id": "req_003",
  "code": 0,
  "msg": "success",
  "data": {}
}
```

说明：

- `code / msg / data` 语义与 HTTP JSON 包装保持一致
- `code = 0` 表示成功

### 3.3 `stream`

服务端用 `stream` 帧承载 live 事件：

```json
{
  "frame": "stream",
  "id": "req_001",
  "streamId": "s_001",
  "event": {
    "seq": 12,
    "type": "content.delta",
    "delta": "hello",
    "timestamp": 1707000000
  }
}
```

说明：

- `event` 是已有 `EventData` 的 JSON 形态
- `streamId` 是传输层流标识，不等于业务 `runId`
- 一个连接可同时存在多个 stream

### 3.4 `stream` 终止帧

```json
{
  "frame": "stream",
  "id": "req_001",
  "streamId": "s_001",
  "reason": "done",
  "lastSeq": 127
}
```

说明：

- 带 `event` 的 `stream` 是事件帧；带 `reason/lastSeq` 的 `stream` 是终止帧
- `reason` 取值：`done | error | cancelled | detached`
- `lastSeq` 用于断线重连补流
- WebSocket 下不使用 SSE 的 `[DONE]`

### 3.5 `push`

服务端主动推送连接级通知：

```json
{
  "frame": "push",
  "type": "heartbeat",
  "data": {
    "timestamp": 1707000000
  }
}
```

常见 `push.type`：

- `connected`
- `heartbeat`
- `catalog.updated`
- `chat.created`
- `chat.updated`
- `chat.read`
- `run.started`
- `run.finished`

### 3.6 `error`

```json
{
  "frame": "error",
  "type": "invalid_request",
  "id": "req_009",
  "code": 400,
  "msg": "unknown type"
}
```

## 4. 请求映射

建议直接沿用原始 REST 路径作为 WS `type`：

| `request.type` | 典型返回 | 说明 |
| --- | --- | --- |
| `/api/query` | `stream` | 建立流式事件输出，随后收到多条事件帧与一条终止帧 |
| `/api/run/stream` | `stream` | 观察已有 run 的流式事件 |
| `/api/run/status` | `response` | 返回 run 状态快照 |
| `/api/submit` | `response` | 同步确认接收；后续结果继续在既有 `stream` 中出现 |
| `/api/steer` | `response` | 同步确认接收；后续结果继续在既有 `stream` 中出现 |
| `/api/interrupt` | `response` | 同步确认接收；真实停止以 `run.cancel` 或终止帧为准 |
| `/api/agents`、`/api/chats`、`/api/chat`、`/api/tools` 等 | `response` | 普通查询类请求 |

说明：

- 上传继续使用 `POST /api/upload`
- 下载继续使用 `GET /api/resource`
- WebSocket 不处理二进制资源传输

## 5. 关键约束

- 同一连接内，未完成请求的 `id` 不能重复
- 同一连接内，不能重复订阅同一个 `run`
- `lastSeq` 表示客户端已收到的最后一个序号
- 服务端只补发 `seq > lastSeq` 的事件
- 对同一 `awaitingId` 的 `submit` 采用 first-writer-wins

常见错误语义：

- `duplicate_id`
- `duplicate_observe`
- `SEQ_EXPIRED`
- `already_resolved`

## 6. 与 SSE 的关系

两者共享的部分：

- `request.* / run.* / content.* / tool.*` 等业务事件语义
- `seq / type / timestamp`
- `submit / steer / interrupt` 的业务行为

两者不同的部分：

- SSE 使用 `event: message`、`: heartbeat`、`data: [DONE]`
- WebSocket 使用 `stream / push / error` 帧
- WebSocket 支持同一连接上多 stream 复用

如果你先读了 [HTTP API](http-api.md) 和 [SSE 事件模型](sse-events.md)，这里主要补的是“另一种承载方式”，不是重新定义一套事件系统。
