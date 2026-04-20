# 架构与交互图

## 1. 系统参与方

协议涉及的主要角色如下：

- Frontend App：发起 HTTP 请求、订阅 SSE 或建立 WebSocket 连接，渲染内容并提交交互
- Agent Platform：协议统一入口，对外暴露 `/api/*`，并负责运行编排、推理/工具/动作执行与 live 事件回传
- Resource Store：存放上传文件、图片、产物和可下载资源
- Viewport Renderer：承载前端工具/表单/UI 视图

Gateway 可以是 Agent Platform 的一种兼容部署模式，但不是协议必须角色。

## 2. 交互总览图

![01 AGW Sequence Overview](../../assets/diagrams/sequences/01-agw-seq-overview.svg)

### 逻辑说明

- 主入口是 `POST /api/query`
- 平台立即建立 SSE 通道，把执行过程以事件流回传
- 如果启用 WebSocket，也可以通过 `/ws` 承载相同业务语义
- 运行过程中可以继续发送 `submit`、`steer`、`interrupt`
- `submit` 用于前端工具交互结果提交
- `steer` 用于给正在进行的 run 注入额外用户指令
- `interrupt` 用于终止当前 run，流层结果表现为 `run.cancel`

## 3. 编号化主图

完整细分图请见：[交互时序图](../guides/interaction-sequences.md)

- `02-agw-seq-basic.svg`：基础 Query 主流，展示一个 chat 中连续两次 run
- `03-agw-seq-steer.svg`：运行中的 steer 控制分支，`request.steer` 插入原始事件流
- `04-agw-seq-interrupt.svg`：运行中的 interrupt 控制分支，流层结果表现为 `run.cancel`
- `07-agw-seq-artifact.svg`：一次工具结果，多条 `artifact.publish`
- [HITL 交互指南](../guides/hitl.md)：独立展开 `question / approval / form` 三态，对应 `05 / 06 / 08` 三张交互图

## 4. 补充架构示意

![AGW Architecture](../../assets/diagrams/architecture/agw-architecture.svg)

### 结构说明

- 前端默认只与 Agent Platform 交互，不直接调用某个具体 Agent
- 平台负责请求接入、运行编排、推理/工具/动作执行、资源访问和 live SSE 回传
- 上传文件、图片、产物和资源下载统一通过 Resource Store 管理
- Viewport 相关内容通过 `GET /api/viewport` 获取
- 现有主图仍按 HTTP + SSE 展示；WebSocket 作为可选长连接控制与事件接入通道补充说明

这张架构图只是补充说明部署与组件关系；协议主入口与时序理解以这些编号主图为准。

## 5. 三条关键链路

### 查询链路

1. 前端发送 `POST /api/query`
2. 平台建立 SSE 响应
3. 输出 `request.query`
4. 根据需要输出 `chat.start`、`plan.update`、`run.start`
5. 运行过程中输出 `reasoning.*`、`content.*`、`tool.*`、`action.*`
6. 结束时输出 `run.complete` 或 `run.cancel` / `run.error`
7. 传输层输出 `[DONE]`

### 资源链路

1. 前端发送 `POST /api/upload`
2. 平台返回 `upload` 信息
3. 前端把上传结果映射为 `Reference`
4. 后续在 `query` 中通过 `references[]` 或 `#{{refid}}` 使用
5. 资源内容通过 `GET /api/resource` 获取

### 视图交互链路

1. 运行时通过 live SSE 发出 `tool.start`、`awaiting.ask`
2. `awaiting.ask.mode` 统一为 `question`、`approval`、`form` 三态，其中 question / approval 直接内联交互定义，form 额外保留 `viewportType`、`viewportKey`、`viewportPayload`
3. 前端根据 `mode` 选择合适 UI；仅 form 默认依赖 viewport 相关语义，必要时再通过 `GET /api/viewport` 拉取视图
4. 用户在前端完成问题回答、审批决策或表单填写
5. 前端通过 `POST /api/submit` 回传 `runId + awaitingId + params[]`
6. 后续结果继续在同一 SSE 流中体现，常见顺序是 `request.submit -> awaiting.answer -> tool.result`

如果多个终端同时订阅同一 run，`awaiting.ask` 也可以通过多个 observer 广播；任一终端 `submit` 后，其它终端都会继续收到后续事件。

补充说明：

- question：`awaiting.ask` 先于 `tool.args / tool.end`，提交项使用 `answer` 或 `answers`
- approval：`approvals[]` 直接位于 `awaiting.ask` 顶层，提交项使用 `decision` / `reason`
- form：`forms[]` 直接位于 `awaiting.ask` 顶层，且是唯一保留 `viewportType:"html"` + `viewportKey` 的模式
