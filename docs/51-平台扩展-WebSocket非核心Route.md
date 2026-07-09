# WebSocket 非核心 Route

本页列出已注册为 WebSocket `request.type`、但不属于最小对话闭环的扩展 route。

## 1. 当前已注册但非核心的 WS Route

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
