# 首页维护说明

`index.html` 是对外最短入口，只承担站点导航和主视觉展示，不承担正式协议正文。

## 当前职责

- 展示首页主图
- 提供 overview / reference / guides / visuals 的导航入口
- 给第一次访问的读者一个最短阅读路径

## 不应放在首页的内容

- 完整 HTTP API 定义
- 完整 SSE / WebSocket 字段表
- 大量时序细节
- 临时调试说明或一次性实验结论

## 维护约束

- 协议定义应维护在 `docs/reference/`
- 交互主线应维护在 `docs/overview/` 与 `docs/guides/`
- 辅助视觉页应维护在 `docs/visuals/`
- WebSocket route 以 `agent-platform` 当前注册表为准，正式清单维护在 `docs/reference/websocket-protocol.md`
