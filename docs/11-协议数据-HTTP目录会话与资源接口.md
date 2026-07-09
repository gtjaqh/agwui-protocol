# HTTP 目录会话与资源接口

本页从 HTTP API 中拆出 catalog、chat、搜索、反馈、资源与上传相关接口。核心运行入口见 [HTTP 核心运行接口](10-协议数据-HTTP核心运行接口.md)。

## 1. Catalog 与工具

### 1.1 `GET /api/agents`

获取智能体列表。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `scope` | `string` | 可选；agent summary 可见范围 |
| `includeChats` | `number` | 可选；每个 agent 附带最近 chat 数，范围 0-50 |

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
| `chats` | `ChatSummary[]` | 可选；由 `includeChats` 控制 |
| `meta` | `object` | 可选 |

### 1.2 `GET /api/agent`

获取单个智能体详情。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `agentKey` | `string` | 必填 |

响应 `data` 为 `AgentDetailResponse`，核心字段包括 `key/name/icon/description/role/wonders/model/mode/tools/skills/controls/meta`。如果智能体可编辑，还可能返回 `definition/soulPrompt/agentsPrompt/source`。

### 1.3 `POST /api/agent/model-config`

更新或读取 agent 模型配置。请求体包含 `agentKey` 以及模型配置字段；响应为更新后的 agent 配置摘要。

### 1.4 `GET /api/model-options`

获取聊天输入区可展示的模型与 reasoning effort 选项。前端按当前 agent `mode` 自行决定是否展示该控件。

### 1.5 `GET /api/teams`

获取团队列表。响应 `data` 为 `TeamSummary[]`，核心字段包括 `teamId/name/icon/agentKeys/meta`。

### 1.6 Admin catalog

Skill、tool 与可编辑 agent 的管理接口位于 `/api/admin/*`，属于平台扩展，见 [Platform Extensions](50-平台扩展-扩展接口总览.md)。旧的 `/api/channels`、`/api/skills`、`/api/tools`、`/api/tool` 不是当前公开入口。

## 2. Chat、搜索与反馈

### 2.1 `GET /api/chats`

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

### 2.2 `GET /api/chat`

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

### 2.3 `POST /api/read`

标记单个会话或某个智能体下全部会话为已读。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 可选；指定会话 |
| `runId` | `string` | 可选；指定读到的 run |
| `agentKey` | `string` | 可选；当 `chatId` 为空时，按智能体全部标记已读 |

响应 `data` 为 `MarkChatReadResponse`，包含 `chatId/agentKey/lastRunId/read/agentUnreadCount/updatedCount`。

### 2.4 `POST /api/chats/search`

全局搜索会话内容。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `query` | `string` | 必填 |
| `agentKey` | `string` | 可选 |
| `teamId` | `string` | 可选 |
| `limit` | `number` | 可选；缺省为 20 |

响应 `data` 为 `{ query, count, results }`，`results[]` 包含 `chatId/chatName/agentKey/teamId/runId/kind/role/timestamp/snippet/score`。

### 2.5 `POST /api/feedback`

设置或清除 run 反馈。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 必填 |
| `runId` | `string` | 必填 |
| `type` | `"thumbs_down" \| "clear"` | 必填 |
| `comment` | `string` | 可选 |

响应 `data` 为 `{ chatId, runId, type, setAt }`。

## 3. 资源与上传

### 3.1 `GET /api/viewport`

获取工具或表单对应的视图 payload。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `viewportKey` | `string` | 必填 |

响应 `data` 为视图 payload。本地 HTML viewport 通常包装为 `{ "html": "..." }`。

### 3.2 `GET /api/resource`

读取 chat 资源、附件或产物。

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `file` | `string` | 必填；相对 data 根目录的文件路径，兼容 `/data/...` 形式 |
| `t` | `string` | 可选；resource ticket。启用资源票据且没有 Bearer principal 时必填 |

成功时直接返回文件内容。失败时返回 JSON 失败响应。

### 3.3 `POST /api/upload`

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
