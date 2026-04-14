# 共享数据模型

## 1. 引用与标记总原则

协议把附件、工作区文件、selection、截图和图片统一抽象为 `Reference`，并通过两种方式与文本输入关联：

1. 在 `message` 中嵌入 `#{{refid}}` 标记
2. 在请求体里显式携带 `references[]`

推荐同时使用，便于渲染、权限控制、引用追踪和执行优化。

示例：

```json
{
  "message": "请总结 #{{f1}} 并解释 #{{i1}} 中的流程",
  "references": [
    { "id": "f1", "type": "file", "name": "requirements.md" },
    { "id": "i1", "type": "image", "name": "ui.png" }
  ]
}
```

## 2. `Reference`

`Reference` 是统一引用对象：附件、工作区文件、selection、截图等都用它表达；通过 `reference.id` 与文本标记或显式 `references[]` 关联。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | `string` | 短 ID；要求在当前 chat 内唯一，例如 `f1`、`i1`、`r01` |
| `type` | `"file" \| "image"` | 当前只允许两类 |
| `name` | `string` | 可选；展示名。前端请求可省略，服务端生成的引用建议提供 |
| `mimeType` | `string` | 建议总是提供 |
| `sizeBytes` | `number` | 可选；上传或本地可得时提供 |
| `url` | `string` | 网关可访问地址，或可下载地址 |
| `sha256` | `string` | 可选；用于去重、校验、缓存 |
| `sandboxPath` | `string` | 可选；沙箱内可直接访问的路径 |
| `meta` | `object` | 可选；承载来源差异和增强信息 |

### 2.1 `Reference.id` 规则

- 必须简短且在当前 chat 内不重复。
- 推荐命名：
  - 文件：`f1`、`f2`、`f3`
  - 图片：`i1`、`i2`、`i3`
  - 上传结果：`r01`、`r02`
- 也可以使用 base36 递增，只要能稳定映射。

### 2.2 `Reference.type` 规则

| 值 | 用途 |
| --- | --- |
| `"file"` | 文件、工作区 selection、打开文件等 |
| `"image"` | 图片、截图、视觉输入 |

工作区 selection 仍然使用 `"file"`，差异放到 `meta`。

## 3. `Reference.meta`

`meta` 用来表达“来源差异”与增强信息，避免为了不同来源再发明新类型。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `origin` | `"upload" \| "workspace" \| "clipboard" \| "url"` | 引用来源 |
| `filePath` | `string` | 可选；工作区文件路径 |
| `range` | `object` | 可选；selection 范围 |
| `textExcerpt` | `string` | 可选；selection 或文件摘要，减少必须先拉全文的情况 |
| `caption` | `string` | 可选；图片注释或用户说明 |
| `ocrText` | `string` | 可选；图片 OCR 结果 |

### 3.1 `meta.range`

`range` 推荐结构：

| 字段 | 类型 |
| --- | --- |
| `startLine` | `number` |
| `startCol` | `number` |
| `endLine` | `number` |
| `endCol` | `number` |

### 3.2 常见来源映射

| 场景 | `type` | 关键 `meta` |
| --- | --- | --- |
| 浏览器上传文件 | `"file"` | `origin: "upload"` |
| 浏览器上传图片 | `"image"` | `origin: "upload"`、可带 `caption` |
| 工作区文件选区 | `"file"` | `origin: "workspace"`、`filePath`、`range`、`textExcerpt` |
| 截图/OCR 图像 | `"image"` | `origin: "upload"` 或 `"clipboard"`、`caption`、`ocrText` |

## 4. 上传对象到 `Reference` 的映射

`POST /api/upload` 成功后，返回的 `upload` 可以直接映射为 `Reference` 使用。

### `upload`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | `string` | chat 内短 ID，如 `r01` |
| `type` | `"file" \| "image"` | 上传资源类型 |
| `name` | `string` | 文件名 |
| `mimeType` | `string` | MIME 类型 |
| `sizeBytes` | `number` | 文件大小 |
| `url` | `string` | 上传完成后的资源地址 |
| `sha256` | `string` | 可选；校验值 |

### 映射建议

- `upload.id -> reference.id`
- `upload.type -> reference.type`
- `upload.name -> reference.name`
- `upload.mimeType -> reference.mimeType`
- `upload.sizeBytes -> reference.sizeBytes`
- `upload.url -> reference.url`
- `upload.sha256 -> reference.sha256`
- `meta.origin` 推荐补为 `"upload"`

## 5. 文本标记规则

在 `message` 中使用：

```text
#{{refid}}
```

示例：

```text
请总结 #{{r01}} 的关键需求，并结合 #{{r02}} 指出协议中可能遗漏的字段。
```

规则：

- 前端应优先直接发送带标记的 `message`。
- 如果遗漏标记，网关可以在执行前补齐引用提示，但不应改变原始请求结构。
- 同时建议在 `references[]` 中显式列出本次问题依赖的引用。

## 6. 通用 ID 词汇表

| 字段 | 所属层级 | 说明 |
| --- | --- | --- |
| `requestId` | Request | 前端生成，用于幂等、重试和链路追踪 |
| `chatId` | Chat | 聊天会话 ID；推荐前端生成 UUIDv4，便于离线、重试和跨端关联 |
| `agentKey` | Routing | 可选；不传或传 `auto` 时由网关按策略选择默认智能体或最合适智能体 |
| `teamId` | Routing | 团队或租户路由信息 |
| `runId` | Run | 一次执行过程的唯一标识 |
| `taskId` | Task | 某次 run 中的任务标识 |
| `planId` | Plan | 计划聚合对象的唯一标识 |
| `reasoningId` | Reasoning | 一段推理输出的标识 |
| `contentId` | Content | 一段正文输出的标识 |
| `toolId` | Tool | 一次工具调用的标识 |
| `actionId` | Action | 一次动作渲染或执行的标识 |
| `artifactId` | Artifact | 单个发布产物的标识 |
| `steerId` | Steer | 一次运行中追加指令的标识 |
| `viewportKey` | Viewport | 获取某个前端视图 payload 的键 |

## 7. Query 公共字段

`POST /api/query` 常用到的结构化字段如下：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `role` | `"user" \| "system" \| "assistant" \| "developer" \| "other"` | 当前消息角色 |
| `message` | `string` | 用户输入正文 |
| `references` | `Reference[]` | 显式引用列表 |
| `params` | `object` | 结构化输入，如 `input/select/datepicker/switch` |
| `scene` | `object` | 页面上下文 |
| `scene.url` | `string` | 建议提供 |
| `scene.title` | `string` | 可选 |
| `hidden` | `boolean` | 是否隐藏部分前端展示内容 |
| `stream` | `boolean` | 保留兼容字段；当前 query 固定返回 SSE |

## 8. SSE 事件公共包络

实时业务事件统一包含如下三元组：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `seq` | `number` | 事件序号 |
| `type` | `string` | 事件类型，如 `run.start` |
| `timestamp` | `number` | 事件时间戳 |

调试场景下还可能带：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `rawEvent` | `object` | 原始事件对象，仅调试用途 |

## 9. Chat 聚合对象

除了单次事件，协议还会在会话层维护聚合态。

### `plan`

- 表示 chat 当前最新计划状态。
- 典型结构为 `{ planId, tasks }`。
- 实时流里以 `plan.update` 表达。

### `artifact`

- 表示 chat 当前最新产物集合。
- 典型结构为 `{ items: [...] }`。
- 实时流里每个产物对应一条 `artifact.publish`。

### `references`

- 可理解为 query 引用与 chat 资产聚合后的引用池。
- 常见于 `GET /api/chat` 的会话详情返回。

## 10. 实践建议

- `mimeType` 和 `url` 是跨端渲染最关键的两个字段，建议总是提供。
- 对工作区引用优先补 `meta.filePath`、`meta.range`、`meta.textExcerpt`。
- 对图片引用优先补 `meta.caption`、`meta.ocrText`。
- `id` 尽量短，方便在文本中用 `#{{refid}}` 直接引用。
