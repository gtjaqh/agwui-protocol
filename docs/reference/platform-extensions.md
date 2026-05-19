# Platform Extensions

本页列出 `agent-platform` 当前实现中已暴露、但不属于 AGW UI 核心协议的扩展接口。核心接入优先阅读 [HTTP API](http-api.md)、[SSE 事件模型](sse-events.md) 和 [WebSocket 协议](websocket-protocol.md)。

## 1. 使用约定

- 除特别说明外，HTTP 返回 `ApiResponse<T>`。
- 对应 WS route 已注册时，`request.type` 复用同名 HTTP 路径，成功返回 `response` 帧。
- 本页只记录入口、用途和核心字段，不把内部产品能力展开成完整业务手册。

## 2. Agent 编辑扩展

| HTTP | WS | 用途 | 核心字段 |
| --- | --- | --- | --- |
| `POST /api/agent-create` | 是 | 创建可编辑 Agent | `key`、`definition`、可选 `soulPrompt/agentsPrompt` |
| `POST /api/agent-update` | 是 | 更新可编辑 Agent | `key`、`definition`、可选 `soulPrompt/agentsPrompt` |
| `POST /api/agent-delete` | 是 | 删除可编辑 Agent | `key` |
| `GET /api/agent-editor-options` | 是 | 获取编辑器选项 | 返回 `models/contextTags/modes/proxyConfigSchema` |

响应：

- create/update 返回 `AgentDetailResponse`。
- delete 返回 `{ key, deleted: true }`。
- 如果当前 registry 不支持编辑，返回 `503` / `unavailable`。

## 3. Chat 管理与归档

| HTTP | WS | 用途 | 核心字段 |
| --- | --- | --- | --- |
| `POST /api/chat-delete` | 是 | 删除 chat | `chatId` |
| `POST /api/chat-archive` | 是 | 批量归档 chat | `chatIds[]` |
| `GET /api/chat-export` | 否 | 导出 chat | query 参数按实现处理 |
| `GET /api/archives` | 是 | 归档列表 | `agentKey`、`limit`、`offset` |
| `GET /api/archive` | 是 | 归档详情 | `chatId` |
| `POST /api/archive-search` | 是 | 搜索归档 | `query`、可选 `agentKey/limit` |
| `POST /api/archive-delete` | 是 | 删除归档 | `chatId` |
| `GET /api/archive-resource` | 否 | 读取归档资源 | `file` 等资源参数 |

注意：

- 删除 chat 前，如果存在活跃 run，可能返回 `active_run_conflict`。
- 归档列表响应包含 `total/items`，归档摘要包含 `chatId/chatName/agentKey/teamId/createdAt/updatedAt/archivedAt/lastRunId/lastRunContent/hasAttachments/usage`。

## 4. Schedule 扩展

| HTTP | WS | 用途 | 核心字段 |
| --- | --- | --- | --- |
| `POST /api/schedules` | 是 | 列出定时任务 | 可选 `tag` |
| `POST /api/schedule` | 是 | 获取单个定时任务 | `id` |
| `POST /api/schedule-create` | 是 | 创建定时任务 | `name/description/cron/agentKey/query` |
| `POST /api/schedule-update` | 是 | 更新定时任务 | `id` 加可选更新字段 |
| `POST /api/schedule-delete` | 是 | 删除定时任务 | `id` |
| `POST /api/schedule-toggle` | 是 | 启用或暂停 | `id/enabled` |
| `POST /api/schedule-executions` | 是 | 执行历史 | `id/limit/offset` |

核心模型：

- `ScheduleSummaryResponse`：`id/name/description/cron/agentKey/enabled/teamId/zoneId/sourceFile/remainingRuns/nextFireTime/lastExecution`。
- `ScheduleDetailResponse`：在 summary 基础上增加 `query`。
- `ScheduleQueryRequest`：`message/chatId/role/params/hidden`。

## 5. Memory 扩展

| HTTP | WS | 用途 | 核心字段 |
| --- | --- | --- | --- |
| `POST /api/remember` | 是 | 从 chat 生成或写入记忆 | `requestId/chatId` |
| `POST /api/learn` | 是 | 学习 chat 观察结果 | `requestId/chatId/subjectKey` |
| `GET /api/memory/meta` | 是 | 获取枚举元信息 | 返回 `categories/types/scopeTypes/statuses/sourceTypes` |
| `POST /api/memory/context/preview` | 是 | 预览记忆上下文 | `chatId/message` |
| `GET /api/memory/scopes` | 是 | 列出 scope | `agentKey` 等 query 参数 |
| `/api/memory/scope` | 是 | 读取或保存 scope | GET/PUT 按实现分发 |
| `POST /api/memory/scope/validate` | 是 | 校验 scope markdown | `agentKey/scopeType/markdown` |
| `GET /api/memory/records` | 是 | 查询记忆记录 | 过滤参数按实现处理 |
| `GET /api/memory/history` | 否 | 记忆变更历史 | 过滤参数按实现处理 |
| `GET /api/memory/record` | 是 | 记忆记录详情 | `agentKey/id` |
| `GET /api/memory/record/timeline` | 否 | 单条记忆时间线 | `agentKey/id` |

说明：

- `remember` 返回 `RememberResponse`，包含 `accepted/status/requestId/chatId/memoryCount/items/stored/promptPreview` 等字段。
- `learn` 当前可能返回 `not_connected`，表示学习管线未连接。
- live 流中出现的 `memory.*` 事件只表达运行过程中的记忆操作，不等于这些管理 API 的完整响应模型。

## 6. 资源网关扩展

| HTTP | WS | 用途 | 核心字段 |
| --- | --- | --- | --- |
| `GET /api/resource` | 是 | HTTP 直接读取资源；WS 用于网关资源协商 | `file/t` 或 payload 资源描述 |
| `POST /api/upload` | 是 | HTTP multipart 上传；WS 用于网关下载后复用上传流程 | `requestId/chatId/file` 或下载 payload |
| `/api/pull` | 是 | 网关拉取资源 | 内部/网关协商路径 |

浏览器前端上传文件时仍应优先使用 HTTP multipart；WS 的资源 route 主要服务 gateway bridge 场景，不是通用二进制 WebSocket 上传协议。

## 7. Admin Gateway 扩展

| HTTP | WS | 用途 | 核心字段 |
| --- | --- | --- | --- |
| `GET /api/admin/gateways` | 否 | 列出动态 gateway | 返回 gateway entries，token 不回显 |
| `POST /api/admin/gateways` | 否 | 注册 gateway | `id/channel/url/token` 等 |
| `DELETE /api/admin/gateways/{id}` | 否 | 注销 gateway | path 中的 `id` |

约束：

- Admin API 是 desktop 本机动态注册 gateway 的管理面。
- 未注入 `GatewayAdmin` 时端点返回 `404`。
- 管理路由是 loopback-only。

## 8. 非核心但已注册的 WS Route

下列 route 当前已注册为 WebSocket `request.type`，但不属于核心协议：

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

如果前端只接入标准 AGW UI 对话体验，可以忽略本页大部分扩展，只实现核心 HTTP/SSE/WS 三层协议。
