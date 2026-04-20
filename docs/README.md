# 文档导航

`agwui-protocol` 采用内容仓库化结构，把“协议是什么”“协议怎么定义”“怎么接入理解”“辅助视觉页”分开维护。

## 阅读顺序

1. `overview/`：先理解协议定位、术语边界和整体架构
2. `reference/`：再查看 HTTP API、SSE 事件、WebSocket 与数据模型
3. `guides/`：最后看时序图、接入用例、资源导航
4. `visuals/`：需要辅助理解时再看预览页和设计辅助页

## 分区说明

### `overview/`

- [协议定位](overview/protocol-positioning.md)
- [术语与边界](overview/terminology-and-boundaries.md)
- [架构与交互图](overview/architecture.md)

这一组文档回答“这是什么、怎么理解、主线是什么”。

### `reference/`

- [HTTP API](reference/http-api.md)
- [SSE 事件模型](reference/sse-events.md)
- [WebSocket 协议](reference/websocket-protocol.md)
- [共享数据模型](reference/data-models.md)

这一组文档回答“协议怎么定义、字段和行为是什么”。

### `guides/`

- [交互时序图](guides/interaction-sequences.md)
- [HITL 交互指南](guides/hitl.md)
- [接入用例](guides/use-cases.md)
- [资源导航](guides/resources.md)

这一组文档回答“怎么接入、怎么联调、怎么阅读时序”。

### `visuals/`

- [SSE Event Color Preview](visuals/sse-event-color-preview.html)

这一组内容是辅助视觉页，不属于正式规范正文。
