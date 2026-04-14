# 接入用例

用例以“公开 HTTP API 请求 + 对应 SSE 事件流”为最小闭环展示。基础场景通常从 `POST /api/query` 开始，运行中也可以继续发送 `POST /api/steer` 或 `POST /api/interrupt`。

说明：

- `reasoning.delta`、`tool.args`、`content.delta` 都是流式“吐字”，会拆成多段
- 业务事件统一采用 `event: message + data: {"seq":...,"type":"...","timestamp":...}` 的单行 JSON 格式
- 流结束时会追加 `data:[DONE]`

## 1. 基础问答：Query + Content

基础场景：只有 query 和正文输出，不涉及 Tool、Action 或多任务。

```json
// POST /api/query
{
  "requestId": "req_001",
  "chatId": "chat_123",
  "agentKey": "auto",
  "role": "user",
  "message": "用一句话解释事件流模型是什么？并说明关键事件有哪些（chat.start、run.start、content.start、content.delta、run.complete）。",
  "scene": { "url": "https://app.example.com/chat", "title": "Chat" },
  "stream": true
}
```

```text
// SSE stream (each data: is ONE-LINE JSON)
event: message
data: {"seq":1,"type":"request.query","timestamp":1707000000,"requestId":"req_001","chatId":"chat_123","agentKey":"auto","role":"user","message":"用一句话解释事件流模型是什么？并说明关键事件有哪些（chat.start、run.start、content.start、content.delta、run.complete）。","scene":{"url":"https://app.example.com/chat","title":"Chat"},"stream":true}

event: message
data: {"seq":2,"type":"chat.start","timestamp":1707000001,"chatId":"chat_123","chatName":"Minimal QA"}

event: message
data: {"seq":3,"type":"run.start","timestamp":1707000002,"runId":"run_001","chatId":"chat_123","agentKey":"auto"}

event: message
data: {"seq":4,"type":"content.start","timestamp":1707000003,"contentId":"ctx_001","runId":"run_001"}

event: message
data: {"seq":5,"type":"content.delta","timestamp":1707000004,"contentId":"ctx_001","delta":"事件流模型："}

event: message
data: {"seq":6,"type":"content.delta","timestamp":1707000005,"contentId":"ctx_001","delta":"后端把一次请求拆成多个可订阅事件，"}

event: message
data: {"seq":7,"type":"content.delta","timestamp":1707000006,"contentId":"ctx_001","delta":"按序推送给前端实现边生成边渲染（如 chat.start→run.start→content.*→run.complete）。"}

event: message
data: {"seq":8,"type":"content.end","timestamp":1707000007,"contentId":"ctx_001"}

event: message
data: {"seq":9,"type":"run.complete","timestamp":1707000008,"runId":"run_001","finishReason":"end_turn"}

event: message
data: [DONE]
```

## 2. Thinking + Tooling + Content

先推理（`reasoning.*`），再工具（`tool.*`），再回答（`content.*`）。

```json
// POST /api/query
{
  "requestId": "req_002",
  "chatId": "chat_123",
  "agentKey": "auto",
  "role": "user",
  "message": "今天北京天气如何？请先查，再给出穿衣建议与注意事项。",
  "scene": { "url": "https://app.example.com/chat" },
  "stream": true
}
```

```text
// SSE stream (reasoning.delta / tool.args / content.delta are streaming chunks)
event: message
data: {"seq":1,"type":"request.query","timestamp":1707000100,"requestId":"req_002","chatId":"chat_123","agentKey":"auto","role":"user","message":"今天北京天气如何？请先查，再给出穿衣建议与注意事项。","scene":{"url":"https://app.example.com/chat"},"stream":true}

event: message
data: {"seq":2,"type":"chat.start","timestamp":1707000101,"chatId":"chat_123","chatName":"Weather / Beijing"}

event: message
data: {"seq":3,"type":"run.start","timestamp":1707000102,"runId":"run_002","chatId":"chat_123","agentKey":"auto"}

event: message
data: {"seq":4,"type":"reasoning.start","timestamp":1707000103,"reasoningId":"re_002_1","runId":"run_002"}

event: message
data: {"seq":5,"type":"reasoning.delta","timestamp":1707000104,"reasoningId":"re_002_1","delta":"先"}

event: message
data: {"seq":6,"type":"reasoning.delta","timestamp":1707000105,"reasoningId":"re_002_1","delta":"检索北京今天天气要素（温度/降水/风/空气质量），"}

event: message
data: {"seq":7,"type":"reasoning.delta","timestamp":1707000106,"reasoningId":"re_002_1","delta":"再基于结果给出穿衣建议与注意事项。"}

event: message
data: {"seq":8,"type":"reasoning.end","timestamp":1707000107,"reasoningId":"re_002_1"}

event: message
data: {"seq":9,"type":"tool.start","timestamp":1707000108,"toolId":"tool_002_1","runId":"run_002","toolName":"web_search"}

event: message
data: {"seq":10,"type":"tool.args","timestamp":1707000109,"toolId":"tool_002_1","delta":"{\"query\":\"Beijing weather","chunkIndex":0}

event: message
data: {"seq":11,"type":"tool.args","timestamp":1707000110,"toolId":"tool_002_1","delta":" today\",\"locale\":\"zh-CN\"","chunkIndex":1}

event: message
data: {"seq":12,"type":"tool.args","timestamp":1707000111,"toolId":"tool_002_1","delta":"}","chunkIndex":2}

event: message
data: {"seq":13,"type":"tool.end","timestamp":1707000112,"toolId":"tool_002_1"}

event: message
data: {"seq":14,"type":"tool.result","timestamp":1707000113,"toolId":"tool_002_1","result":"{\"summary\":\"Sunny\",\"tempC\":25,\"wind\":\"NW 3\",\"aqi\":\"Good\"}"}

event: message
data: {"seq":15,"type":"content.start","timestamp":1707000114,"contentId":"ctx_002","runId":"run_002"}

event: message
data: {"seq":16,"type":"content.delta","timestamp":1707000115,"contentId":"ctx_002","delta":"根据检索结果：北京今天晴，约 25°C，西北风 3 级，空气质量良好。\\n"}

event: message
data: {"seq":17,"type":"content.delta","timestamp":1707000116,"contentId":"ctx_002","delta":"穿衣建议：短袖/薄长袖为主，备一件轻薄外套以防早晚温差。\\n"}

event: message
data: {"seq":18,"type":"content.delta","timestamp":1707000117,"contentId":"ctx_002","delta":"注意事项：防晒、补水；风较干时注意润唇与眼部防护。"}

event: message
data: {"seq":19,"type":"content.end","timestamp":1707000118,"contentId":"ctx_002"}

event: message
data: {"seq":20,"type":"run.complete","timestamp":1707000119,"runId":"run_002","finishReason":"end_turn"}

event: message
data: [DONE]
```

## 3. 计划与多任务：Query + Plan + Tasks

适合“先检索，再写总结”这类分步骤任务。

```json
// POST /api/query
{
  "requestId": "req_003",
  "chatId": "c_01",
  "agentKey": "auto",
  "role": "user",
  "message": "帮我先查资料，再写一段总结。要求：先列出检索关键词，再给 5 条要点，最后给 1 段总结。",
  "scene": { "url": "https://app.example.com/chat" },
  "stream": true
}
```

```text
// SSE stream
event: message
data: {"seq":1,"type":"request.query","timestamp":1707000200,"requestId":"req_003","chatId":"c_01","agentKey":"auto","role":"user","message":"帮我先查资料，再写一段总结。要求：先列出检索关键词，再给 5 条要点，最后给 1 段总结。","scene":{"url":"https://app.example.com/chat"},"stream":true}

event: message
data: {"seq":2,"type":"chat.start","timestamp":1707000201,"chatId":"c_01","chatName":"Research then Write"}

event: message
data: {"seq":3,"type":"plan.update","timestamp":1707000202,"planId":"p_01","chatId":"c_01","plan":[{"taskId":"task_search","description":"检索资料并整理要点","status":"pending"},{"taskId":"task_write","description":"基于检索结果输出结构化要点与总结","status":"pending"}]}

event: message
data: {"seq":4,"type":"run.start","timestamp":1707000203,"runId":"r_01","chatId":"c_01","agentKey":"auto"}

event: message
data: {"seq":5,"type":"task.start","timestamp":1707000204,"taskId":"task_search","runId":"r_01","taskName":"Search","description":"检索资料并整理要点"}

event: message
data: {"seq":6,"type":"reasoning.start","timestamp":1707000205,"reasoningId":"re_search_1","runId":"r_01","taskId":"task_search"}

event: message
data: {"seq":7,"type":"reasoning.delta","timestamp":1707000206,"reasoningId":"re_search_1","delta":"先列关键词，"}

event: message
data: {"seq":8,"type":"reasoning.delta","timestamp":1707000207,"reasoningId":"re_search_1","delta":"检索 3-5 个来源，"}

event: message
data: {"seq":9,"type":"reasoning.delta","timestamp":1707000208,"reasoningId":"re_search_1","delta":"抽取共识并归纳要点。"}

event: message
data: {"seq":10,"type":"reasoning.end","timestamp":1707000209,"reasoningId":"re_search_1"}

event: message
data: {"seq":11,"type":"tool.start","timestamp":1707000210,"toolId":"t_search_1","runId":"r_01","taskId":"task_search","toolName":"web_search"}

event: message
data: {"seq":12,"type":"tool.args","timestamp":1707000211,"toolId":"t_search_1","delta":"{\"query\":\"topic key","chunkIndex":0}

event: message
data: {"seq":13,"type":"tool.args","timestamp":1707000212,"toolId":"t_search_1","delta":"words\",\"topK\":5","chunkIndex":1}

event: message
data: {"seq":14,"type":"tool.args","timestamp":1707000213,"toolId":"t_search_1","delta":"}","chunkIndex":2}

event: message
data: {"seq":15,"type":"tool.end","timestamp":1707000214,"toolId":"t_search_1"}

event: message
data: {"seq":16,"type":"tool.result","timestamp":1707000215,"toolId":"t_search_1","result":"{\"keywords\":[\"keyword1\",\"keyword2\"],\"sources\":[{\"title\":\"Source A\",\"url\":\"https://example.com/a\"}]}"}

event: message
data: {"seq":17,"type":"task.complete","timestamp":1707000216,"taskId":"task_search"}

event: message
data: {"seq":18,"type":"task.start","timestamp":1707000217,"taskId":"task_write","runId":"r_01","taskName":"Write","description":"基于检索结果输出结构化要点与总结"}

event: message
data: {"seq":19,"type":"content.start","timestamp":1707000218,"contentId":"ctx_write_1","runId":"r_01","taskId":"task_write"}

event: message
data: {"seq":20,"type":"content.delta","timestamp":1707000219,"contentId":"ctx_write_1","delta":"检索关键词：keyword1、keyword2\\n"}

event: message
data: {"seq":21,"type":"content.delta","timestamp":1707000220,"contentId":"ctx_write_1","delta":"要点：\\n1) ...\\n2) ...\\n3) ...\\n4) ...\\n5) ...\\n"}

event: message
data: {"seq":22,"type":"content.delta","timestamp":1707000221,"contentId":"ctx_write_1","delta":"总结：...（一段话）"}

event: message
data: {"seq":23,"type":"content.end","timestamp":1707000222,"contentId":"ctx_write_1"}

event: message
data: {"seq":24,"type":"task.complete","timestamp":1707000223,"taskId":"task_write"}

event: message
data: {"seq":25,"type":"run.complete","timestamp":1707000224,"runId":"r_01","finishReason":"end_turn"}

event: message
data: [DONE]
```

## 4. 文件 + 图片引用：Upload -> Query

上传资源后，再通过 `#{{refid}}` 标记和 `references[]` 显式引用。

```json
// 1) POST /api/upload (multipart/form-data)
{
  "requestId": "req_up_0001",
  "chatId": "7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1",
  "file": "(binary requirements.md)",
  "sha256": "b9c3...e21a"
}

// 1) Response
{
  "requestId": "req_up_0001",
  "chatId": "7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1",
  "upload": {
    "id": "r01",
    "type": "file",
    "name": "requirements.md",
    "mimeType": "text/markdown",
    "sizeBytes": 48291,
    "url": "/api/resource?file=7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1%2Frequirements.md",
    "sha256": "b9c3...e21a"
  }
}

// 2) POST /api/upload (image multipart/form-data)
{
  "requestId": "req_up_0002",
  "chatId": "7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1",
  "file": "(binary ui.png)"
}

// 2) Response
{
  "requestId": "req_up_0002",
  "chatId": "7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1",
  "upload": {
    "id": "r02",
    "type": "image",
    "name": "ui.png",
    "mimeType": "image/png",
    "sizeBytes": 245991,
    "url": "/api/resource?file=7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1%2Fui.png",
    "sha256": "7b1a...f02c"
  }
}

// 3) POST /api/query (use markers + explicit references)
{
  "requestId": "req_0002",
  "chatId": "7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1",
  "agentKey": "agent_researcher",
  "role": "user",
  "message": "请总结 #{{r01}} 的关键需求，并结合 #{{r02}} 指出协议中可能遗漏的字段。",
  "references": [
    {
      "id": "r01",
      "type": "file",
      "name": "requirements.md",
      "mimeType": "text/markdown",
      "sizeBytes": 48291,
      "url": "/api/resource?file=7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1%2Frequirements.md",
      "sha256": "b9c3...e21a",
      "sandboxPath": "/workspace/requirements.md",
      "meta": { "origin": "upload", "textExcerpt": "## Requests\n- query: chatId, agent, content..." }
    },
    {
      "id": "r02",
      "type": "image",
      "name": "ui.png",
      "mimeType": "image/png",
      "sizeBytes": 245991,
      "url": "/api/resource?file=7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1%2Fui.png",
      "sandboxPath": "/workspace/ui.png",
      "meta": { "origin": "upload", "caption": "协议图：Query/Upload/Viewport/Submit 四类请求" }
    }
  ],
  "scene": { "url": "https://app.example.com/workspace/protocol" },
  "stream": true
}
```

```text
// SSE (snippet)
event: message
data: {"seq":1,"type":"request.query","timestamp":1707000400,"requestId":"req_0002","chatId":"7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1","agentKey":"agent_researcher","role":"user","message":"请总结 #{{r01}} 的关键需求，并结合 #{{r02}} 指出协议中可能遗漏的字段。","references":[{"id":"r01","type":"file","name":"requirements.md","sandboxPath":"/workspace/requirements.md"},{"id":"r02","type":"image","name":"ui.png","sandboxPath":"/workspace/ui.png"}],"scene":{"url":"https://app.example.com/workspace/protocol"},"stream":true}

event: message
data: {"seq":2,"type":"chat.start","timestamp":1707000401,"chatId":"7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1","chatName":"Protocol Review"}

event: message
data: {"seq":3,"type":"run.start","timestamp":1707000402,"runId":"run_9001","chatId":"7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1","agentKey":"agent_researcher"}

event: message
data: {"seq":24,"type":"run.complete","timestamp":1707000419,"runId":"run_9001","finishReason":"end_turn"}

event: message
data: [DONE]
```

## 5. 工作区 selection：仍用 `type="file"`，差异放 `meta`

工作区选区不需要新类型，仍然使用 `"file"`，差异放进 `meta`。

```json
// POST /api/query
{
  "requestId": "req_0003",
  "chatId": "c40a08d0-b61d-4be0-99b5-3a9e8c7466c2",
  "agentKey": "agent_code_review",
  "role": "user",
  "message": "请审查我选中的代码片段（见 #{{f1}}），指出潜在 bug、边界条件与改进建议。",
  "references": [
    {
      "id": "f1",
      "type": "file",
      "name": "app/src/main.ts",
      "mimeType": "text/plain",
      "url": "https://workspace.example.com/ws/proj/file/app%2Fsrc%2Fmain.ts?selection=120:1-180:1",
      "meta": {
        "origin": "workspace",
        "filePath": "app/src/main.ts",
        "range": { "startLine": 120, "startCol": 1, "endLine": 180, "endCol": 1 },
        "textExcerpt": "function boot() {\n  // selected code here\n}"
      }
    }
  ],
  "params": { "switch": { "strictMode": true }, "select": { "severity": "high" } },
  "scene": { "url": "https://app.example.com/ide/editor?file=app/src/main.ts" },
  "stream": true
}
```

```text
// SSE (snippet)
event: message
data: {"seq":1,"type":"request.query","timestamp":1707000500,"requestId":"req_0003","chatId":"c40a08d0-b61d-4be0-99b5-3a9e8c7466c2","agentKey":"agent_code_review","role":"user","message":"请审查我选中的代码片段（见 #{{f1}}），指出潜在 bug、边界条件与改进建议。","references":[{"id":"f1","type":"file","name":"app/src/main.ts"}],"params":{"switch":{"strictMode":true},"select":{"severity":"high"}},"scene":{"url":"https://app.example.com/ide/editor?file=app/src/main.ts"},"stream":true}

event: message
data: {"seq":2,"type":"chat.start","timestamp":1707000501,"chatId":"c40a08d0-b61d-4be0-99b5-3a9e8c7466c2","chatName":"Code Review / main.ts"}

event: message
data: {"seq":3,"type":"run.start","timestamp":1707000502,"runId":"run_cr_001","chatId":"c40a08d0-b61d-4be0-99b5-3a9e8c7466c2","agentKey":"agent_code_review"}

event: message
data: {"seq":18,"type":"run.complete","timestamp":1707000518,"runId":"run_cr_001","finishReason":"end_turn"}

event: message
data: [DONE]
```

## 6. 图片为主：带 OCR 和 caption

图片场景建议补充 `caption` 和 `ocrText`，帮助模型少走一步。

```json
// POST /api/query
{
  "requestId": "req_0004",
  "chatId": "9d81c2c8-2c0c-4f7e-a2f0-5f5fd0a9b6b1",
  "agentKey": "auto",
  "role": "user",
  "message": "请解释截图 #{{i1}} 的报错原因，并给出排查步骤。",
  "references": [
    {
      "id": "i1",
      "type": "image",
      "name": "billing-error.png",
      "mimeType": "image/png",
      "sizeBytes": 188120,
      "url": "https://files.example.com/chat/9d81.../i1",
      "meta": {
        "origin": "upload",
        "caption": "支付页报错截图（包含错误码与时间戳）",
        "ocrText": "Error: BILLING_403\nRequestId: abcd-1234\nTime: 2026-02-06 10:01:22"
      }
    }
  ],
  "scene": { "url": "https://app.example.com/billing/checkout" },
  "stream": true
}
```

```text
// SSE (snippet)
event: message
data: {"seq":1,"type":"request.query","timestamp":1707000600,"requestId":"req_0004","chatId":"9d81c2c8-2c0c-4f7e-a2f0-5f5fd0a9b6b1","agentKey":"auto","role":"user","message":"请解释截图 #{{i1}} 的报错原因，并给出排查步骤。","references":[{"id":"i1","type":"image","name":"billing-error.png"}],"scene":{"url":"https://app.example.com/billing/checkout"},"stream":true}

event: message
data: {"seq":2,"type":"chat.start","timestamp":1707000601,"chatId":"9d81c2c8-2c0c-4f7e-a2f0-5f5fd0a9b6b1","chatName":"Billing Error Triage"}

event: message
data: {"seq":3,"type":"run.start","timestamp":1707000602,"runId":"run_bill_001","chatId":"9d81c2c8-2c0c-4f7e-a2f0-5f5fd0a9b6b1","agentKey":"auto"}

event: message
data: {"seq":16,"type":"run.complete","timestamp":1707000616,"runId":"run_bill_001","finishReason":"end_turn"}

event: message
data: [DONE]
```

## 7. 运行中引导：Query 之后追加 Steer

`steer` 请求层是 `/api/steer`，流里体现为 `request.steer`。

```json
// 1) POST /api/query
{
  "requestId": "req_steer_001",
  "chatId": "chat_live_001",
  "agentKey": "auto",
  "role": "user",
  "message": "先给我一版三点总结，主题是我们新的 AGW 协议文档。",
  "stream": true
}
```

```json
// 2) POST /api/steer
{
  "requestId": "req_steer_002",
  "chatId": "chat_live_001",
  "runId": "run_live_001",
  "steerId": "steer_001",
  "agentKey": "auto",
  "teamId": "docs",
  "message": "补充一个要求：输出前先列出 HTTP API 和 SSE event 的边界，再继续完成总结。",
  "planningMode": false
}
```

```json
// 2) Response
{
  "accepted": true,
  "status": "accepted",
  "runId": "run_live_001",
  "steerId": "steer_001",
  "detail": "steer accepted"
}
```

```text
// SSE stream (same run continues)
event: message
data: {"seq":1,"type":"request.query","timestamp":1707000700,"requestId":"req_steer_001","chatId":"chat_live_001","agentKey":"auto","role":"user","message":"先给我一版三点总结，主题是我们新的 AGW 协议文档。","stream":true}

event: message
data: {"seq":2,"type":"chat.start","timestamp":1707000701,"chatId":"chat_live_001","chatName":"AGW Doc Draft"}

event: message
data: {"seq":3,"type":"run.start","timestamp":1707000702,"runId":"run_live_001","chatId":"chat_live_001","agentKey":"auto"}

event: message
data: {"seq":4,"type":"reasoning.start","timestamp":1707000703,"reasoningId":"re_live_001","runId":"run_live_001"}

event: message
data: {"seq":5,"type":"reasoning.delta","timestamp":1707000704,"reasoningId":"re_live_001","delta":"先整理文档边界，"}

event: message
data: {"seq":6,"type":"request.steer","timestamp":1707000705,"requestId":"req_steer_002","chatId":"chat_live_001","runId":"run_live_001","steerId":"steer_001","message":"补充一个要求：输出前先列出 HTTP API 和 SSE event 的边界，再继续完成总结。","role":"user"}

event: message
data: {"seq":7,"type":"reasoning.delta","timestamp":1707000706,"reasoningId":"re_live_001","delta":"先说明公开 HTTP API 与流内回看事件不是同一层协议，"}

event: message
data: {"seq":8,"type":"reasoning.end","timestamp":1707000707,"reasoningId":"re_live_001"}

event: message
data: {"seq":9,"type":"content.start","timestamp":1707000708,"contentId":"ctx_live_001","runId":"run_live_001"}

event: message
data: {"seq":10,"type":"content.delta","timestamp":1707000709,"contentId":"ctx_live_001","delta":"边界说明：/api/query、/api/submit、/api/steer、/api/interrupt 是公开请求；request.query、request.submit、request.steer 是流内回看事件。\\n"}

event: message
data: {"seq":11,"type":"content.delta","timestamp":1707000710,"contentId":"ctx_live_001","delta":"三点总结：1) 文档需要把公开 API 与 SSE 拆开说明；2) interrupt 应映射到 run.cancel；3) 用例需要同时展示请求层与流层。"}

event: message
data: {"seq":12,"type":"content.end","timestamp":1707000711,"contentId":"ctx_live_001"}

event: message
data: {"seq":13,"type":"run.complete","timestamp":1707000712,"runId":"run_live_001","finishReason":"end_turn"}

event: message
data: [DONE]
```

## 8. 运行中中断：Interrupt -> `run.cancel`

请求层是 `/api/interrupt`，流里直接表现为 `run.cancel`，不会出现独立的 interrupt request 事件。

```json
// 1) POST /api/query
{
  "requestId": "req_interrupt_001",
  "chatId": "chat_live_002",
  "agentKey": "auto",
  "role": "user",
  "message": "帮我写一份很长的 AGW 协议对比说明，覆盖所有事件类型。",
  "stream": true
}
```

```json
// 2) POST /api/interrupt
{
  "requestId": "req_interrupt_002",
  "chatId": "chat_live_002",
  "runId": "run_live_002",
  "agentKey": "auto",
  "teamId": "docs",
  "message": "先停止当前回答，我要改需求。",
  "planningMode": false
}
```

```json
// 2) Response
{
  "accepted": true,
  "status": "accepted",
  "runId": "run_live_002",
  "detail": "interrupt accepted"
}
```

```text
// SSE stream
event: message
data: {"seq":1,"type":"request.query","timestamp":1707000800,"requestId":"req_interrupt_001","chatId":"chat_live_002","agentKey":"auto","role":"user","message":"帮我写一份很长的 AGW 协议对比说明，覆盖所有事件类型。","stream":true}

event: message
data: {"seq":2,"type":"chat.start","timestamp":1707000801,"chatId":"chat_live_002","chatName":"AGW Protocol Compare"}

event: message
data: {"seq":3,"type":"run.start","timestamp":1707000802,"runId":"run_live_002","chatId":"chat_live_002","agentKey":"auto"}

event: message
data: {"seq":4,"type":"content.start","timestamp":1707000803,"contentId":"ctx_live_002","runId":"run_live_002"}

event: message
data: {"seq":5,"type":"content.delta","timestamp":1707000804,"contentId":"ctx_live_002","delta":"先从 Query / Upload / Submit / Steer / Interrupt 五类公开 API 开始，"}

event: message
data: {"seq":6,"type":"content.end","timestamp":1707000805,"contentId":"ctx_live_002"}

event: message
data: {"seq":7,"type":"run.cancel","timestamp":1707000806,"runId":"run_live_002"}

event: message
data: [DONE]
```

## 9. 文件产出物发布：一次工具调用，多条 `artifact.publish`

一次 `_artifact_publish_` 工具调用可以发布多个文件，并通过多条独立 `artifact.publish` 事件把这些文件发布到当前 chat；`runId` 仅用于标记来源 run。

```json
// 1) tool call (one invocation, multiple artifacts)
{
  "name": "_artifact_publish_",
  "arguments": {
    "artifacts": [
      {
        "path": "/workspace/reports/final-report.md",
        "description": "最终交付报告"
      },
      {
        "path": "/workspace/reports/summary.md",
        "name": "summary.md"
      }
    ]
  }
}
```

```text
// SSE stream
event: message
data: {"seq":9,"type":"tool.start","timestamp":1707000908,"toolId":"tool_publish_1","runId":"run_artifact_001","toolName":"_artifact_publish_"}

event: message
data: {"seq":10,"type":"tool.args","timestamp":1707000909,"toolId":"tool_publish_1","delta":"{\"artifacts\":[{\"path\":\"/workspace/reports/final-report.md\",\"description\":\"最终交付报告\"},{\"path\":\"/workspace/reports/summary.md\",\"name\":\"summary.md\"}]}","chunkIndex":0}

event: message
data: {"seq":11,"type":"tool.end","timestamp":1707000910,"toolId":"tool_publish_1"}

event: message
data: {"seq":12,"type":"tool.result","timestamp":1707000911,"toolId":"tool_publish_1","result":"{\"ok\":true,\"artifacts\":[{\"artifactId\":\"asset_5d4c3b2a1f00\",\"artifact\":{\"type\":\"file\",\"name\":\"final-report.md\",\"mimeType\":\"text/markdown\",\"url\":\"/api/resource?file=chat_artifact_001%2Fartifacts%2Frun_artifact_001%2Ffinal-report.md\"}},{\"artifactId\":\"asset_5d4c3b2a1f01\",\"artifact\":{\"type\":\"file\",\"name\":\"summary.md\",\"mimeType\":\"text/markdown\",\"url\":\"/api/resource?file=chat_artifact_001%2Fartifacts%2Frun_artifact_001%2Fsummary.md\"}}]}"}

event: message
data: {"seq":13,"type":"artifact.publish","timestamp":1707000912,"artifactId":"asset_5d4c3b2a1f00","chatId":"chat_artifact_001","runId":"run_artifact_001","artifact":{"type":"file","name":"final-report.md","mimeType":"text/markdown","url":"/api/resource?file=chat_artifact_001%2Fartifacts%2Frun_artifact_001%2Ffinal-report.md"}}

event: message
data: {"seq":14,"type":"artifact.publish","timestamp":1707000913,"artifactId":"asset_5d4c3b2a1f01","chatId":"chat_artifact_001","runId":"run_artifact_001","artifact":{"type":"file","name":"summary.md","mimeType":"text/markdown","url":"/api/resource?file=chat_artifact_001%2Fartifacts%2Frun_artifact_001%2Fsummary.md"}}

event: message
data: [DONE]
```
