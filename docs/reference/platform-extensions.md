# Platform Extensions

本页列出 `agent-platform` 当前实现中已暴露、但不属于 AGW UI 核心对话协议的扩展接口。核心接入优先阅读 [HTTP API](http-api.md)、[SSE 事件模型](sse-events.md) 和 [WebSocket 协议](websocket-protocol.md)。

## 1. 使用约定

- 除特别说明外，HTTP 返回 `ApiResponse<T>`。
- 对应 WS route 已注册时，`request.type` 复用同名 HTTP 路径，成功返回 `response` 帧。
- WS 注册表以 [WebSocket 协议](websocket-protocol.md) 的当前 request 映射为准；HTTP 存在不代表 WS 也注册。
- 本页只记录入口、用途和核心字段，不把内部产品能力展开成完整业务手册。

## 2. Admin Catalog 扩展

Agent、skill、tool 的管理接口是 HTTP 管理面；当前不注册为普通 WebSocket `request.type`。

| HTTP | WS | 用途 | 核心字段 |
| --- | --- | --- | --- |
| `GET /api/admin/agents` | 否 | 列出可编辑 Agent | 列表、排序和编辑元信息 |
| `GET /api/admin/agents/detail` | 否 | 获取可编辑 Agent 详情 | `agentKey` |
| `/api/admin/agents/order` | 否 | 读取或保存 Agent 排序 | GET / PUT |
| `POST /api/admin/agents/create` | 否 | 创建可编辑 Agent | `key`、`definition`、prompt 文件内容 |
| `POST /api/admin/agents/update` | 否 | 更新可编辑 Agent | `key`、`definition`、prompt 文件内容 |
| `POST /api/admin/agents/update-name` | 否 | 更新 Agent 展示名称 | `key`、`name` |
| `POST /api/admin/agents/delete` | 否 | 删除可编辑 Agent | `key` |
| `GET /api/admin/agents/editor-options` | 否 | 获取编辑器选项 | models、modes、schema 等 |
| `GET /api/admin/skills` | 否 | 列出 skill 目录 | skills summary |
| `GET /api/admin/tools` | 否 | 列出 tool 目录 | tools summary |

旧的 `/api/agent/create`、`/api/agent/update`、`/api/agent/delete`、`/api/agent/editor-options` 不是当前公开入口，也不是当前 WS route。

## 3. Chat 管理与归档

| HTTP | WS | 用途 | 核心字段 |
| --- | --- | --- | --- |
| `GET /api/chats` | 是 | chat 列表 | `agentKey`、分页参数 |
| `GET /api/chat` | 是 | chat 详情 | `chatId`、`includeRawMessages` |
| `GET /api/chat/jsonl` | 是 | 导出 chat JSONL | `chatId` |
| `GET /api/chat/llm-trace` | 是 | 导出 LLM trace | `chatId` |
| `GET /api/chat/export` | 否 | 导出 chat | `chatId` |
| `POST /api/chats/search` | 是 | 搜索 active chats | `query`、可选 `agentKey/teamId/limit` |
| `POST /api/read` | 是 | 标记已读 | `chatId` 或 `agentKey` |
| `POST /api/feedback` | 是 | run 反馈 | `chatId`、`runId`、`type` |
| `POST /api/chat/delete` | 是 | 删除 chat | `chatId` |
| `POST /api/chat/rename` | 是 | 重命名 chat | `chatId`、`chatName` |
| `POST /api/chat/derive` | 是 | 派生 chat | `chatId` |
| `POST /api/chat/archive` | 是 | 批量归档 chat | `chatIds[]` |
| `GET /api/archives` | 是 | 归档列表 | `agentKey`、`limit`、`offset` |
| `GET /api/archive` | 是 | 归档详情 | `chatId` |
| `POST /api/archives/search` | 是 | 搜索归档 | `query`、可选 `agentKey/limit` |
| `POST /api/archive/delete` | 是 | 删除归档 | `chatId` |
| `POST /api/archive/restore` | 是 | 恢复归档 chat | `chatIds[]` |

删除、归档或恢复前如果存在活跃 run，可能返回 `active_run_conflict`。

## 4. Automation 扩展

Automation 取代旧 schedule 文档口径。HTTP 提供管理入口；WS 当前只提供列表、详情和执行记录读取。

| HTTP | WS | 用途 | 核心字段 |
| --- | --- | --- | --- |
| `POST /api/automations` | 是 | 列出 automation | 可选过滤字段 |
| `POST /api/automation` | 是 | 获取单个 automation | `id` |
| `POST /api/automation/create` | 否 | 创建 automation | `name/description/cron/agentKey/query` 等 |
| `POST /api/automation/update` | 否 | 更新 automation | `id` 加可选更新字段 |
| `POST /api/automation/delete` | 否 | 删除 automation | `id` |
| `POST /api/automation/toggle` | 否 | 启用或暂停 | `id/enabled` |
| `POST /api/automation/executions` | 是 | 执行历史 | `id` 或 `automationId`、`limit`、`offset` |

旧的 `/api/schedules`、`/api/schedule*` 不属于当前 platform API / WS route。

## 5. Run、HITL 与 Terminal 扩展

| HTTP | WS | 用途 | 核心字段 |
| --- | --- | --- | --- |
| `POST /api/compact` | 是 | 压缩 chat 上下文 | `chatId` 等压缩参数 |
| `POST /api/access-level` | 是 | 更新运行权限级别 | `runId`、`agentKey`、`accessLevel` |
| 无 | 是：`/api/detach` | detach run stream | `runId`、可选 `agentKey` |
| 无 | 是：`/api/terminal/open` | 打开终端 | shell / cwd / env 等终端参数 |
| 无 | 是：`/api/terminal/input` | 写入终端输入 | `terminalId`、`data` |
| 无 | 是：`/api/terminal/resize` | 调整终端尺寸 | `terminalId`、`cols`、`rows` |
| 无 | 是：`/api/terminal/detach` | detach 终端 stream | `terminalId` |
| 无 | 是：`/api/terminal/close` | 关闭终端 | `terminalId` |
| 无 | 是：`/api/terminal/status` | 查询或订阅终端状态 | `terminalId` |
| 无 | 是：`/api/terminal/status/detach` | detach 终端状态订阅 | `terminalId` |

`/api/query`、`/api/attach`、`/api/submit`、`/api/steer`、`/api/interrupt` 属于核心交互入口，详见 [HTTP API](http-api.md) 与 [WebSocket 协议](websocket-protocol.md)。

## 6. Memory 扩展

| HTTP | WS | 用途 | 核心字段 |
| --- | --- | --- | --- |
| `POST /api/learn` | 是 | 学习 chat 观察结果 | `requestId/chatId/subjectKey` |
| `GET /api/memory/meta` | 是 | 获取枚举元信息 | categories、types、scopeTypes、statuses、sourceTypes |
| `POST /api/memory/context-preview` | 是 | 预览记忆上下文 | `chatId/message` 等 |
| `GET /api/memory/scope/list` | 是 | 列出 scope | `agentKey` 等过滤字段 |
| `GET /api/memory/scope/detail` | 是 | 读取 scope | `agentKey/scopeType/scopeKey` |
| `POST /api/memory/scope/save` | 是 | 保存 scope | `agentKey/scopeType/scopeKey/mode/markdown/records` |
| `POST /api/memory/scope/validate` | 是 | 校验 scope markdown | `agentKey/scopeType/markdown` |
| `GET /api/memory/record/list` | 是 | 查询记忆记录 | 过滤参数按实现处理 |
| `GET /api/memory/history` | 否 | 记忆变更历史 | 过滤参数按实现处理 |
| `GET /api/memory/record/detail` | 是 | 记忆记录详情 | `agentKey/id` 或 `recordId` |
| `GET /api/memory/record/timeline` | 否 | 单条记忆时间线 | `id`、`limit` |

旧的 `/api/remember`、`/api/memory/context/preview`、`/api/memory/scopes`、`/api/memory/scope`、`/api/memory/records`、`/api/memory/record` 不是当前 WS route。live 流中的 `memory.*` 事件只表达运行过程中的记忆操作，不等于这些管理 API 的完整响应模型。

## 7. 资源网关扩展

| HTTP | WS | 用途 | 核心字段 |
| --- | --- | --- | --- |
| `GET /api/resource` | 是 | HTTP 直接读取资源；WS 将本地资源推送到 gateway `pushURL` | `file/t` 或 `file/pushURL` |
| `POST /api/upload` | 是 | HTTP multipart 上传；WS 按 gateway 下发 URL 拉取资源并复用上传流程 | `requestId/chatId/file` 或下载 payload |
| `GET /api/tool-result` | 否 | 读取 `.tools/results/<toolId>.json` 完整工具结果 | `chatId/path/t` |

浏览器前端上传文件时仍应优先使用 HTTP multipart。`/api/pull` 与 `/api/push` 是 gateway HTTP 数据面路径，不是 platform WS `request.type`。

## 8. 当前已注册但非核心的 WS Route

下列 route 当前已注册为 WebSocket `request.type`，但不属于最小对话闭环：

```text
/api/agent/model-config
/api/model-options
/api/chat/jsonl
/api/chat/llm-trace
/api/chat/delete
/api/chat/rename
/api/chat/derive
/api/chat/archive
/api/archives
/api/archive
/api/archives/search
/api/archive/delete
/api/archive/restore
/api/automations
/api/automation
/api/automation/executions
/api/detach
/api/access-level
/api/terminal/open
/api/terminal/input
/api/terminal/resize
/api/terminal/detach
/api/terminal/close
/api/terminal/status
/api/terminal/status/detach
/api/learn
/api/compact
/api/memory/meta
/api/memory/context-preview
/api/memory/scope/list
/api/memory/scope/detail
/api/memory/scope/save
/api/memory/scope/validate
/api/memory/record/list
/api/memory/record/detail
/api/resource
/api/upload
```

如果前端只接入标准 AGW UI 对话体验，可以忽略本页大部分扩展，只实现核心 HTTP/SSE/WS 三层协议。
