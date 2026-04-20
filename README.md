# AGW UI Interaction Protocol

AGW UI Interaction Protocol 用于定义前端应用如何与智能体平台通信，包括公开 HTTP API、实时 SSE 事件流、可选 WebSocket 传输层、共享数据模型，以及典型接入用例。

这里的 Agent Platform 是协议默认主体；Gateway 只是其中一种兼容部署模式，协议本身不要求中间必须存在 Gateway。

浏览器入口见 [index.html](index.html)。仓库正文采用“内容仓库化”结构：正式规范、接入指南、视觉预览页和静态资产分层维护。

## 阅读顺序

1. [文档总导航](docs/README.md)
2. [协议定位](docs/overview/protocol-positioning.md)
3. [术语与边界](docs/overview/terminology-and-boundaries.md)
4. [架构与交互图](docs/overview/architecture.md)
5. [HTTP API](docs/reference/http-api.md)
6. [SSE 事件模型](docs/reference/sse-events.md)
7. [WebSocket 协议](docs/reference/websocket-protocol.md)
8. [共享数据模型](docs/reference/data-models.md)
9. [交互时序图](docs/guides/interaction-sequences.md)
10. [HITL 交互指南](docs/guides/hitl.md)
11. [接入用例](docs/guides/use-cases.md)
12. [资源导航](docs/guides/resources.md)

## 内容分区

| 分区 | 说明 |
| --- | --- |
| [`docs/overview/`](docs/overview/protocol-positioning.md) | 协议定位、术语边界、架构主线 |
| [`docs/reference/`](docs/reference/http-api.md) | 正式协议定义：HTTP、SSE、WebSocket、数据模型 |
| [`docs/guides/`](docs/guides/interaction-sequences.md) | 时序图、HITL 指南、接入用例、资源导航 |
| [`docs/visuals/`](docs/visuals/sse-event-color-preview.html) | 辅助理解页面，不属于正式规范正文 |
| [`assets/diagrams/`](assets/diagrams/sequences/02-agw-seq-basic.svg) | 时序图与架构图静态资产 |

## 站点入口

- 首页：[index.html](index.html)
- 文档导航：[docs/README.md](docs/README.md)
- 视觉预览：[docs/visuals/sse-event-color-preview.html](docs/visuals/sse-event-color-preview.html)

## 协议边界

- 所有公开请求都以 `/api` 为前缀。
- `POST /api/query` 在 HTTP 形态下直接返回 `text/event-stream`，不是普通 JSON 响应。
- `request.*`、`run.*`、`task.*`、`content.*` 等名称属于实时事件层，不要求和 HTTP API 形成 1:1 命名映射。
- `POST /api/interrupt` 的结果在流层体现为 `run.cancel`，不会额外产生 `request.interrupt`。
- WebSocket 是额外传输层，不改变既有 HTTP API 与事件语义。

## 目录结构

```text
.
├── README.md
├── index.html
├── docs/
│   ├── README.md
│   ├── overview/
│   ├── reference/
│   ├── guides/
│   └── visuals/
├── assets/
│   ├── diagrams/
│   │   ├── sequences/
│   │   ├── architecture/
│   │   └── overview/
│   └── images/
└── site/
```
