# 交互时序图

本页把 AGW 的 live 协议交互收敛为 5 张编号化主图，只覆盖前端真实可见的 `HTTP + live SSE`，不画 snapshot / persisted 历史事件。为兼容既有静态资源，图号继续沿用原编号。

如果启用 WebSocket，业务先后关系不变；变化的是建流方式与传输承载：

- `/api/query` 可以改为 `/ws` 上的 `/api/query request frame`
- live 事件改由 `stream frame` 承载
- `submit / steer / interrupt` 可以通过同一条 WS 连接发出

因此本页不再复制一套“WebSocket 版时序图”，而是继续用 HTTP + SSE 主路径表达业务语义。

## 01 总览图

![01 AGW Sequence Overview](../../assets/diagrams/sequences/01-agw-seq-overview.svg)

总览图只负责说明 Query、Submit、Steer、Interrupt、HITL、Artifact 的入口关系与分流，不展开复杂分支细节。

## 02 基础多轮 Query

![02 AGW Basic Sequence](../../assets/diagrams/sequences/02-agw-seq-basic.svg)

这张图只画基础 Query 主流，并把同一个 chat 中的两次 run 并列出来：

- 首次请求：`POST /api/query -> [chat.start] -> run.start -> ... -> run.complete`
- 后续请求：复用 `chatId`，再次 `POST /api/query -> run.start`
- 关键差异：`chat.start` 只在创建新 chat 时出现，后续 turn 不再重复

## 03 Steer

![03 AGW Steer Sequence](../../assets/diagrams/sequences/03-agw-seq-steer.svg)

这张图单独展开运行中的 steer 控制分支：

- 图从运行中已有的 `content.start` 开始，不展示 `/api/query` 建流阶段
- `POST /api/steer -> HTTP ack` 之后，原正文先 `content.end`，再插入 `request.steer`
- 随后在同一条原始 SSE 流里开启新的 `content.start`，并正常 `run.complete`

## 04 Interrupt

![04 AGW Interrupt Sequence](../../assets/diagrams/sequences/04-agw-seq-interrupt.svg)

这张图单独展开运行中的 interrupt 控制分支：

- 图从运行中已有的 `content.start` 开始，不展示 `/api/query` 建流阶段
- `POST /api/interrupt -> HTTP ack -> run.cancel`
- 流里不会出现 `request.interrupt`
- `[DONE]` 属于通用传输层收尾，不在这张细分图里单独展开

## HITL 独立说明

Human-in-the-loop 已从本页拆到独立指南：[HITL 交互指南](hitl.md)。

独立页会分别展开：

- `question`：`awaiting.ask` 先于 `tool.args / tool.end`
- `approval`：`tool.args / tool.end` 之后再进入 `awaiting.ask`
- `form`：与 approval 同序，但额外保留 `viewportType / viewportKey / viewportPayload`

三态共享的提交边界：

- `POST /api/submit` 的 HTTP body 统一为 `runId + awaitingId + params[]`
- 流里先写 `request.submit` 记录原始 `params[]`
- 随后写 `awaiting.answer` 记录归一化结果

## 07 Artifact

![07 AGW Artifact Sequence](../../assets/diagrams/sequences/07-agw-seq-artifact.svg)

这张图只画运行中的 artifact 分支：

- 图从运行中的 `tool.start` 开始，不展示 `/api/query` 建流阶段
- `tool.start -> tool.args -> tool.end -> tool.result`
- 一次工具调用可以在 `tool.result` 之后连续发出多条 `artifact.publish`

## 统一边界

- 默认主体是 `Frontend / Agent Platform`
- Gateway 只是兼容部署模式，不是协议必须角色
- 不画 `reasoning.snapshot`、`content.snapshot`、`tool.snapshot`、`action.snapshot`
- 不画不存在的 `request.interrupt`
- 不在 `tool.start` 上标注当前实现没有的 `toolType`、`viewportKey`、`toolTimeout`
- `POST /api/submit` 的稳定形状是 `runId + awaitingId + params[]`
- HITL 的流内边界是 `request.submit` 记录原始输入，`awaiting.answer` 记录归一化结果
