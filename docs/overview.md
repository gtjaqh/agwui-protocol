# 协议概览

## 1. 协议定位

AGW UI Interaction Protocol 定义的是前端与多智能体网关（Gateway）之间的交互协议，而不是前端直接与某个具体 Agent 的私有协议。

网关承担三类职责：

- 对外暴露统一的 `/api/*` HTTP API
- 把请求路由到具体智能体或多智能体协作链路
- 通过统一的 SSE 事件流把过程和结果持续回传给前端

推荐理解方式是：前端只对接 Gateway，Gateway 再负责路由、执行编排、资源管理和事件分发。

## 2. 两层协议边界

这是最重要的理解前提。

### 公开 HTTP API 层

- `/api/query`
- `/api/upload`
- `/api/submit`
- `/api/steer`
- `/api/interrupt`
- `/api/viewport`
- `/api/resource`
- `/api/agents`
- `/api/agent`
- `/api/chats`
- `/api/chat`

### SSE 事件层

- `request.*`
- `chat.*`
- `plan.*`
- `run.*`
- `task.*`
- `reasoning.*`
- `content.*`
- `tool.*`
- `artifact.*`
- `action.*`
- `source.*`

这两层不是 1:1 对应关系：

- `POST /api/query` 会在流里出现 `request.query`
- `POST /api/submit` 会在流里出现 `request.submit`
- `POST /api/steer` 会在流里出现 `request.steer`
- `POST /api/interrupt` 不会出现 `request.interrupt`，最终效果由 `run.cancel` 表达

## 3. 三条关键边界

### `query` 直接返回 SSE

- `POST /api/query` 当前固定返回 `text/event-stream`
- 不会返回独立 JSON body
- 业务事件统一走 `event: message`
- 每条业务事件的 `data` 都是一行 JSON

### `steer` 是公开请求，流里体现为 `request.steer`

- `POST /api/steer` 是公开 HTTP API
- 其同步确认来自 HTTP 响应
- 其运行回看事件在原 SSE 流中体现为 `request.steer`

### `interrupt` 在流里体现为 `run.cancel`

- `POST /api/interrupt` 是公开 HTTP API
- 流层不会产生独立的 `request.interrupt`
- 最终停止结果由 `run.cancel` 表达

## 4. 命名约定

### 动作类请求

建议使用 `POST`：

- `query`
- `upload`
- `submit`
- `steer`
- `interrupt`

### 资源类请求

建议使用 `GET`：

- `viewport`
- `resource`
- `agents`
- `agent`
- `chats`
- `chat`

## 5. 典型执行生命周期

一个最小闭环通常长这样：

1. 前端调用 `POST /api/query`
2. 服务端直接返回 `text/event-stream`
3. 流里出现 `request.query`
4. 如果 chat 不存在，出现 `chat.start`
5. 出现 `run.start`
6. 运行过程中持续输出 `reasoning.*`、`content.*`、`tool.*`、`action.*`
7. 正常结束时出现 `run.complete`
8. 流结束时追加 `data:[DONE]`

如果运行中有人工干预：

- 追加指令用 `POST /api/steer`
- 提交前端工具交互结果用 `POST /api/submit`
- 强制中断用 `POST /api/interrupt`

## 6. 共享核心对象

协议中最关键的共享概念如下：

- `Reference`：统一表示文件、图片、工作区选区、截图等引用对象
- `requestId`：用于幂等、重试和链路追踪
- `chatId`：聊天会话标识
- `runId`：一次执行过程的标识
- `taskId`：某次 run 中的任务标识
- `viewportKey`：前端视图载荷的检索键

详细字段见 [共享数据模型](data-models.md)。

## 7. Query 的关键事实

- `POST /api/query` 当前固定返回 SSE，不返回独立 JSON body
- 实时业务事件统一包含 `seq`、`type`、`timestamp`
- SSE 传输层可能出现 `event: heartbeat`
- 结束标志是 `data:[DONE]`，它不是业务事件，也不会进入历史 `events`

## 8. 引用与标记

引用遵循统一规则：

- 所有显式引用放到 `references[]`
- 文本中用 `#{{refid}}` 关联具体引用
- `id` 建议短且在当前 chat 内唯一
- 文件通常用 `f1`、`f2`
- 图片通常用 `i1`、`i2`
- 上传资源也可以用 `r01`、`r02`

例如：

```text
请总结 #{{f1}}，并结合 #{{i1}} 解释其中的交互流程。
```

## 9. 兼容说明

- 历史文档中的 `ViewRequest / ViewResponse` 已统一改称 `ViewportRequest / ViewportResponse`
- 当前协议仍保留对既有前端产品形态的适配说明，例如 `scene`、`params`、`references.meta`
- `Snapshot` 类事件主要用于历史记录，不是当前 `/api/query` 实时流的主路径

## 10. 如何阅读本仓库

不同读者建议的阅读路径：

- 前端接入方：先看 [架构与交互图](architecture.md) 和 [HTTP API](http-api.md)
- 实时事件渲染方：重点看 [SSE 事件模型](sse-events.md)
- 协议维护者：按顺序阅读 [概览](overview.md)、[数据模型](data-models.md)、[HTTP API](http-api.md)、[SSE 事件模型](sse-events.md)
- 产品或方案同学：先看 [概览](overview.md) 和 [接入用例](use-cases.md)
