# HTTP 核心运行接口

本页定义 AGW UI 核心 HTTP 接口。事实源以 `agent-platform` 当前实现为准；记忆、定时任务、归档、Agent 编辑和 Admin Gateway 等平台能力见 [Platform Extensions](50-平台扩展-扩展接口总览.md)。

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
| `agentKey` | `string` | 必填；必须匹配当前 run 所属智能体 |
| `awaitingId` | `string` | 必填；当前等待态 ID |
| `params` | `SubmitParam[]` | 必填；必须是数组，可为空数组表示整批取消 |

`params[i]` 与当前 `awaiting.ask` 中的 `questions/approvals/forms[i]` 按下标对应。`mode=plan` 固定只接受 1 项，对应单个 `awaiting.ask.plan`。`id` 可选，主要用于审计和日志。

常见形态：

- question：`{"id":"q1","answer":"..."}` 或 `{"id":"q2","answers":[...]}`
- question date：`{"id":"startDate","answer":"2026-05-15"}`
- question datetime：`{"id":"startAt","answer":"2026-05-15T09:30"}`
- approval：`{"id":"tool_bash","decision":"approve|approve_rule_run|reject","reason":"..."}`
- form approve：`{"id":"form-1","decision":"approve","form":{...}}`
- form reject：`{"id":"form-1","decision":"reject","reason":"...","form":{...}}`
- plan：`{"id":"run_001_coder_plan_confirm_1","decision":"approve|reject","reason":"..."}`

说明：

- `date` / `datetime` 问题必须提交单值 `answer`，不能提交 `answers`；服务端要求非空字符串并原样归一化。
- form 对外协议推荐使用 `form` 字段；兼容型 generic frontend tool 可接受 `payload`、`value`、`answer` 作为表单对象来源。

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
