# AGW UI Interaction Protocol

AGW UI Interaction Protocol 用于定义前端应用如何与智能体平台通信，包括核心 HTTP API、实时 SSE 事件流、可选 WebSocket 传输层、共享数据模型、平台扩展接口，以及典型接入用例。

这里的 Agent Platform 是协议默认主体；Gateway 只是其中一种兼容部署模式，协议本身不要求中间必须存在 Gateway。

浏览器入口见 [index.html](index.html)。仓库正文采用编号专题文档结构：正式协议、交互容器、接入用例、时序资产和平台扩展分层维护。

本仓库是协议文档与静态站点仓库，不是运行时代码库；它不实现 Agent Platform、Gateway、前端客户端或后端服务，只维护协议事实、阅读路径、静态时序图和辅助视觉页。

## 阅读顺序

1. [文档总导航](docs/README.md)
2. [协议定位与阅读主线](docs/01-概览-协议定位与阅读主线.md)
3. [术语边界与架构主线](docs/02-概览-术语边界与架构主线.md)
4. [HTTP 核心运行接口](docs/10-协议数据-HTTP核心运行接口.md)
5. [HTTP 目录会话与资源接口](docs/11-协议数据-HTTP目录会话与资源接口.md)
6. [SSE 传输与请求运行事件](docs/13-协议数据-SSE传输与请求运行事件.md)
7. [SSE 内容工具来源与产物事件](docs/14-协议数据-SSE内容工具来源与产物事件.md)
8. [WebSocket 传输协议](docs/15-协议数据-WebSocket传输协议.md)
9. [共享数据模型](docs/16-协议数据-共享数据模型.md)
10. [HITL Question 与 Approval](docs/20-交互容器-HITL-Question与Approval.md)
11. [HITL Form 与 Plan](docs/21-交互容器-HITL-Form与Plan.md)
12. [接入用例](docs/30-接入用例-基础问答与工具调用.md)
13. [交互时序图](docs/40-交互时序-编号化主图与资产引用.md)
14. [平台扩展接口总览](docs/50-平台扩展-扩展接口总览.md)
15. [资源导航](docs/60-资源导航-SDK测试项目与视觉页.md)

## 内容分区

| 分区 | 说明 |
| --- | --- |
| `docs/01-02` | 协议定位、术语边界、架构和参与方 |
| `docs/10-16` | 正式协议定义：HTTP、SSE、WebSocket、共享数据模型 |
| `docs/20-22` | HITL、运行控制、资源和产物交互 |
| `docs/30-33` | 按场景拆分的接入用例 |
| `docs/40` | 时序图与 SVG 资产引用 |
| `docs/50-51` | 平台扩展接口与非核心 WS route |
| `docs/60` | SDK、测试项目、Demo、视觉页资源导航 |
| `docs/visuals/` | 辅助理解页面，不属于正式规范正文 |
| `assets/diagrams/` | 时序图与架构图静态资产 |

## 站点入口

- 首页：[index.html](index.html)
- 文档导航：[docs/README.md](docs/README.md)
- 视觉预览：[docs/visuals/sse-event-color-preview.html](docs/visuals/sse-event-color-preview.html)

## 协议边界

- 所有公开请求都以 `/api` 为前缀。
- `POST /api/query` 在 HTTP 形态下直接返回 `text/event-stream`，不是普通 JSON 响应。
- `GET /api/attach` 用于观察已有 run 或按 `lastSeq` 补流，同样返回 `text/event-stream`。
- `request.*`、`run.*`、`task.*`、`content.*` 等名称属于实时事件层，不要求和 HTTP API 形成 1:1 命名映射。
- `POST /api/interrupt` 的结果在流层体现为 `run.cancel`，不会额外产生 `request.interrupt`。
- WebSocket 是额外传输层，不改变既有 HTTP API 与事件语义。
- 记忆、automation、归档、管理面与资源协商等接口属于平台扩展，见 [平台扩展接口总览](docs/50-平台扩展-扩展接口总览.md)。

## 目录结构

```text
.
├── README.md
├── index.html
├── docs/
│   ├── README.md
│   ├── 01-概览-协议定位与阅读主线.md
│   ├── 10-协议数据-HTTP核心运行接口.md
│   ├── 20-交互容器-HITL-Question与Approval.md
│   ├── 30-接入用例-基础问答与工具调用.md
│   ├── 40-交互时序-编号化主图与资产引用.md
│   ├── 50-平台扩展-扩展接口总览.md
│   ├── 60-资源导航-SDK测试项目与视觉页.md
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

## 维护约定

- 根 README 只维护项目入口、阅读顺序、内容分区、协议边界和目录说明。
- 正式正文只维护在 `docs/` 根部编号专题文档中。
- 旧的 `docs/overview/`、`docs/reference/`、`docs/guides/` 路径只作为兼容入口。
- SDK、测试项目、Demo 和视觉资源入口统一维护在 [资源导航](docs/60-资源导航-SDK测试项目与视觉页.md)。
