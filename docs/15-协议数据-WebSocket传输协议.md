# WebSocket 协议

WebSocket 是 AGW 的可选传输层扩展，用于在一条长连接上复用普通请求、运行流观察和服务端主动推送。它不替代 HTTP API 和 SSE 事件语义；业务事件仍以 [SSE 事件模型](13-协议数据-SSE传输与请求运行事件.md) 中的 `EventData` 为准。

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

当前 `agent-platform` 主动发送的 `push.type`：

| `push.type` | 功能域 | 说明 |
| --- | --- | --- |
| `connected` | 连接 | 连接建立后发送，`data.sessionId` 为连接会话 ID。 |
| `heartbeat` | 连接 | 周期保活，`data.timestamp` 为毫秒时间戳。 |
| `auth.expiring` | 鉴权 | token 即将过期，`data.expiresAt` 为毫秒时间戳。 |
| `run.started` | Run | run 已启动，携带 `runId`、`chatId`、`agentKey`。 |
| `run.finished` | Run | run 已结束，携带 `runId`、`chatId`。 |
| `chat.created` | Chat | 新 chat 已创建，携带 `chatId`、`chatName`、`agentKey`、`timestamp`。 |
| `chat.updated` | Chat | chat 最近 run 内容或更新时间变化，携带 `lastRunId`、`lastRunContent`、`updatedAt`。 |
| `chat.unread` | Chat | chat 未读状态或 agent 未读计数变化。 |
| `chat.read` | Chat | 单个 chat 已标记为已读。 |
| `chat.read_all` | Chat | 某 agent 下会话批量标记已读。 |
| `chat.deleted` | Chat | chat 已删除。 |
| `chat.renamed` | Chat | chat 名称已修改。 |
| `chat.archived` | Archive | chat 已归档。 |
| `chat.restored` | Archive | 归档 chat 已恢复。 |
| `archive.deleted` | Archive | 归档记录已删除。 |
| `catalog.updated` | Catalog | agents / tools / skills 等目录配置已重载或更新。 |
| `awaiting.asking` | HITL | run 进入等待用户输入、审批、表单或 plan 确认状态；完整交互定义仍以 stream `awaiting.ask` 为准。 |
| `awaiting.answered` | HITL | 等待项已被提交、超时、中断或出错；完整结果仍以 stream `awaiting.answer` 为准。 |
| `resource.pushed` | Resource | 生成 artifact 已推送到 gateway。 |

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

## 3. 当前 request / response 映射

`agent-platform` 当前注册 52 个 `/api/*` WebSocket request route，另有内置 `auth.refresh`。除 `/api/query` 与 `/api/attach` 成功返回 `stream` 外，其余成功路径返回 `response`，且 `response.type` 与 `request.type` 相同。

| `request.type` | 功能域 | 成功返回 | 说明 |
| --- | --- | --- | --- |
| `auth.refresh` | 鉴权 | `response` | 刷新当前连接认证；这是 handler 内置 type，不是 `/api/*` route。 |
| `/api/locale` | 基础 | `response` | 获取当前 locale。 |
| `/api/agents` | Catalog | `response` | 获取 agent 列表。 |
| `/api/agent` | Catalog | `response` | 获取单个 agent 详情。 |
| `/api/agent/model-config` | Catalog | `response` | 更新或获取 agent 模型配置。 |
| `/api/model-options` | Catalog | `response` | 获取当前可选聊天模型与 reasoning effort。 |
| `/api/teams` | Catalog | `response` | 获取 team 列表。 |
| `/api/chats` | Chat | `response` | 获取 chat 列表，可能包含 active run 与 awaiting 摘要。 |
| `/api/chat` | Chat | `response` | 获取单个 chat 详情与历史事件。 |
| `/api/chat/jsonl` | Chat | `response` | 导出 chat JSONL。 |
| `/api/chat/llm-trace` | Chat | `response` | 导出 LLM trace。 |
| `/api/read` | Chat | `response` | 标记单个 chat 或某 agent 下所有 chat 已读。 |
| `/api/feedback` | Chat | `response` | 提交 run feedback。 |
| `/api/chat/delete` | Chat | `response` | 删除 chat。 |
| `/api/chat/rename` | Chat | `response` | 重命名 chat。 |
| `/api/chat/derive` | Chat | `response` | 从现有 chat 派生新 chat。 |
| `/api/chat/archive` | Archive | `response` | 批量归档 chat。 |
| `/api/archives` | Archive | `response` | 获取归档列表。 |
| `/api/archive` | Archive | `response` | 获取单个归档详情。 |
| `/api/archives/search` | Archive | `response` | 搜索归档。 |
| `/api/archive/delete` | Archive | `response` | 删除归档记录。 |
| `/api/archive/restore` | Archive | `response` | 恢复归档 chat。 |
| `/api/automations` | Automation | `response` | 获取 automation 列表。 |
| `/api/automation` | Automation | `response` | 获取单个 automation 详情。 |
| `/api/automation/executions` | Automation | `response` | 获取 automation 执行记录。 |
| `/api/chats/search` | Search | `response` | 全局搜索 active chats。 |
| `/api/query` | Run | `stream` | 创建 run 并转发 live 事件。 |
| `/api/attach` | Run | `stream` | 观察已有 run，按 `lastSeq` 补流。 |
| `/api/detach` | Run | `response` | 从当前连接 detach 已观察的 run stream。 |
| `/api/submit` | HITL | `response` | 提交 question / approval / form / plan 等等待项答案；后续结果仍在原 stream 中出现。 |
| `/api/steer` | Run | `response` | 对运行中的 run 追加 steer 指令；后续结果仍在原 stream 中出现。 |
| `/api/interrupt` | Run | `response` | 请求中断运行中的 run；真实停止以 `run.cancel`、`run.error` 或 stream 终止帧为准。 |
| `/api/access-level` | Run / HITL | `response` | 更新当前 run 的 access level，并可能触发等待项重新评估。 |
| `/api/terminal/open` | Terminal | `response` | 打开终端会话。 |
| `/api/terminal/input` | Terminal | `response` | 向终端写入输入。 |
| `/api/terminal/resize` | Terminal | `response` | 调整终端尺寸。 |
| `/api/terminal/detach` | Terminal | `response` | detach 终端 stream。 |
| `/api/terminal/close` | Terminal | `response` | 关闭终端。 |
| `/api/terminal/status` | Terminal | `response` | 查询或订阅终端状态。 |
| `/api/terminal/status/detach` | Terminal | `response` | detach 终端状态订阅。 |
| `/api/learn` | Memory | `response` | 触发 chat 学习或记忆整理。 |
| `/api/compact` | Chat / Memory | `response` | 压缩 chat 上下文。 |
| `/api/memory/meta` | Memory Console | `response` | 获取 memory console 枚举元信息。 |
| `/api/memory/context-preview` | Memory Console | `response` | 预览 memory context。 |
| `/api/memory/scope/list` | Memory Console | `response` | 获取 memory scope 列表。 |
| `/api/memory/scope/detail` | Memory Console | `response` | 获取 scope 详情。 |
| `/api/memory/scope/save` | Memory Console | `response` | 保存 scope 配置或记录。 |
| `/api/memory/scope/validate` | Memory Console | `response` | 校验 scope markdown。 |
| `/api/memory/record/list` | Memory Console | `response` | 查询 memory records。 |
| `/api/memory/record/detail` | Memory Console | `response` | 获取单条 memory record。 |
| `/api/viewport` | Viewport | `response` | 获取 viewport payload。 |
| `/api/resource` | Resource | `response` | WS 控制面：把本地资源推送到 gateway `pushURL`。 |
| `/api/upload` | Resource | `response` | WS 控制面：gateway 通知 platform 通过 URL 拉取上传资源。 |

说明：

- 当前实现的观察入口是 `/api/attach`，不是 `/api/run/stream`。
- 当前实现没有公开 `/api/run/status` WS route；如需状态，优先通过 `/api/chat` 的 `activeRun` 或流事件判断。
- 浏览器本地文件上传仍优先使用 HTTP `POST /api/upload`；WS 下 `/api/upload` 和 `/api/resource` 主要用于 gateway 资源协商。
- `/api/pull` 和 `/api/push` 是 gateway HTTP 数据面路径，不是 platform WS `request.type`。

## 4. 未注册的旧 WS type

以下旧 route 不属于当前 platform WebSocket 注册表；发送时会按未知 type 返回 `error`。可用替代入口见 [Platform Extensions](50-平台扩展-扩展接口总览.md)。

```text
/api/agent/create
/api/agent/update
/api/agent/delete
/api/agent/editor-options
/api/channels
/api/skills
/api/tools
/api/tool
/api/schedules
/api/schedule
/api/schedule/create
/api/schedule/update
/api/schedule/delete
/api/schedule/toggle
/api/schedule/executions
/api/remember
/api/memory/context/preview
/api/memory/scopes
/api/memory/scope
/api/memory/records
/api/memory/record
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
