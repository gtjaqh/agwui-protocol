# HTTP 与 WebSocket 对照

本页只维护 HTTP 成功形态与 WebSocket 成功形态的对照关系。WebSocket 帧模型见 [WebSocket 传输协议](15-协议数据-WebSocket传输协议.md)。

## 1. HTTP 与 WS 对照

| HTTP 路径 | HTTP 成功形态 | WS 成功形态 |
| --- | --- | --- |
| `POST /api/query` | SSE | `stream` |
| `GET /api/attach` | SSE | `stream` |
| `POST /api/submit` | JSON | `response` |
| `POST /api/steer` | JSON | `response` |
| `POST /api/interrupt` | JSON | `response` |
| 无（WS 内置）`auth.refresh` | 无 | `response`，刷新当前 WebSocket 认证 |
| catalog/chat/search/feedback/archive/automation/memory/viewport | JSON | 已注册 WS route 返回 `response`；完整列表见 [WebSocket 协议](15-协议数据-WebSocket传输协议.md) |
| terminal 控制面 | 无 | 已注册 WS route 返回 `response` |
| `GET /api/resource` | 文件内容 | `response`，用于网关/资源协商，不是二进制直传 |
| `POST /api/upload` | multipart 上传 + JSON | WS 形态用于网关下载/拉取协商；浏览器文件上传仍优先使用 HTTP |
