## 问题的本质：MCP 要解决什么问题？

在讨论技术选型之前，我们需要先明确 MCP（Model Context Protocol）要解决的核心问题。

MCP 的目标是让大模型能够以**统一、标准、可互操作**的方式调用外部工具能力。这意味着：

- 不同语言实现的 MCP Server（Python、Java、Node.js）都能被同一个 MCP Client 调用
- 不同厂商的大模型（OpenAI、Anthropic、本地模型）都能用相同方式调用工具
- 工具的调用方式应该是明确、可预测的，而不是各自为政

要实现这个目标，MCP 需要的不是如何传输数据，而是如何表达一次调用。

## HTTP、gRPC 和 JSON-RPC 2.0 的定位差异

### 1. HTTP：传输协议，不是调用协议

HTTP 是一个**传输层协议**，它定义了：
- 如何建立连接（TCP/TLS）
- 如何发送请求（GET、POST、PUT、DELETE）
- 如何返回响应（状态码、Header、Body）

但 HTTP **不定义**：
- 方法名怎么表示？（是放在 URL 路径？还是 Body 里？）
- 参数怎么传递？（Query String？JSON Body？Form Data？）
- 错误怎么表达？（HTTP 状态码？还是 Body 里的错误对象？）
- 通知消息（不需要响应的调用）怎么处理？

这导致基于 HTTP 的 API 设计千差万别：
- RESTful API：`POST /users`、`GET /users/123`
- RPC-style API：`POST /api` + `{"method": "getUser", "params": {...}}`
- GraphQL：`POST /graphql` + Query DSL

每种风格都有自己的约定，**缺乏统一标准**。

### 2. gRPC：强类型 RPC 框架，但过于重量级

gRPC 是一个完整的 RPC 框架，它提供了：
- 基于 Protocol Buffers 的强类型接口定义
- HTTP/2 传输层
- 流式调用支持
- 多语言代码生成

gRPC 的优势在于**性能和类型安全**，但它也带来了额外的复杂度：

1. **需要预先定义 .proto 文件**：每个工具都要写 Protocol Buffers 定义，增加了开发成本
2. **强依赖代码生成**：客户端和服务端都需要生成代码，动态调用不方便
3. **二进制协议不易调试**：无法直接用 curl 或浏览器测试，必须用专门工具
4. **HTTP/2 依赖**：部分环境（如浏览器、某些代理）对 HTTP/2 支持不完善

对于 MCP 这种需要**轻量、灵活、易于调试**的场景，gRPC 显得过于重量级。

### 3. JSON-RPC 2.0：协议层标准，专注消息格式

JSON-RPC 2.0 的定位与 HTTP、gRPC 都不同，它是一个**消息格式规范**，只定义：

- 一次调用的请求结构：`{"jsonrpc": "2.0", "method": "...", "params": {...}, "id": 1}`
- 成功响应的结构：`{"jsonrpc": "2.0", "result": {...}, "id": 1}`
- 错误响应的结构：`{"jsonrpc": "2.0", "error": {...}, "id": 1}`
- 通知消息的结构：`{"jsonrpc": "2.0", "method": "...", "params": {...}}`（无 `id`）

它**不绑定传输层**，可以跑在：
- HTTP 之上（最常见）
- WebSocket 之上（实时通信）
- stdio 之上（进程间通信）
- 任何能传输 JSON 的通道

## MCP 选择 JSON-RPC 2.0 的核心原因

### 1. 统一的消息格式

JSON-RPC 2.0 提供了一套**明确、标准、无歧义**的消息结构：

```json
// 请求
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {"city": "Beijing"}
  },
  "id": 1
}

// 成功响应
{
  "jsonrpc": "2.0",
  "result": {
    "content": [{"type": "text", "text": "北京今天晴，25°C"}]
  },
  "id": 1
}

// 错误响应
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": "city is required"
  },
  "id": 1
}
```

无论是 Python 实现的 MCP Server，还是 Java 实现的 MCP Client，都能按照**完全相同的方式**解析和构造消息。

### 2. 轻量且易于实现

JSON-RPC 2.0 的规范非常简洁（完整规范只有几页），核心概念只有 4 个：
- Request Object
- Response Object
- Notification
- Error Object

任何语言只需要：
1. 能解析 JSON
2. 能构造 JSON
3. 理解 4 种消息结构

就能实现一个完整的 JSON-RPC 2.0 客户端或服务端。不需要代码生成、不需要复杂的框架、不需要学习新的 IDL。

### 3. 人类可读，易于调试

JSON-RPC 2.0 使用纯文本 JSON 格式，可以直接：
- 用 `curl` 测试：`curl -X POST http://localhost:3000 -d '{"jsonrpc":"2.0","method":"ping","id":1}'`
- 在浏览器 DevTools 中查看请求和响应
- 用任何文本编辑器编辑和调试
- 在日志中直接阅读，无需反序列化

相比之下，gRPC 的 Protocol Buffers 是二进制格式，必须用专门工具才能查看。

### 4. 传输层无关，灵活性高

MCP 的使用场景多样：
- **本地工具调用**：大模型通过 stdio 调用本地进程（如 Python 脚本）
- **远程工具调用**：大模型通过 HTTP 调用远程服务
- **实时通信**：大模型通过 WebSocket 与工具保持长连接

JSON-RPC 2.0 不绑定传输层，可以适配所有这些场景。而 gRPC 强依赖 HTTP/2，无法用于 stdio 场景。

### 5. 支持通知消息（Notification）

MCP 中有些场景不需要响应，例如：
- 日志上报：工具向大模型发送日志，不需要等待确认
- 进度更新：工具向大模型报告任务进度，不需要响应

JSON-RPC 2.0 原生支持 Notification（不带 `id` 的请求），服务端不会返回响应，减少了不必要的网络开销。

HTTP 和 gRPC 都没有原生的单向消息概念，需要额外设计。

### 6. 批量调用支持

JSON-RPC 2.0 支持 Batch Request，可以一次发送多个请求：

```json
[
  {"jsonrpc": "2.0", "method": "tools/list", "id": 1},
  {"jsonrpc": "2.0", "method": "resources/list", "id": 2},
  {"jsonrpc": "2.0", "method": "prompts/list", "id": 3}
]
```

服务端可以并发处理，减少网络往返次数。这对于大模型一次性获取多个工具信息非常有用。

## JSON-RPC 2.0 的局限性

当然，JSON-RPC 2.0 也不是完美的，它有一些明确的局限性：

### 1. 不包含传输层细节

JSON-RPC 2.0 只定义消息格式，不管：
- 如何建立连接（TCP？WebSocket？）
- 如何处理超时和重试
- 如何做负载均衡
- 如何做服务发现

这些需要在 MCP 的实现层或基础设施层补齐。

### 2. 不包含鉴权机制

JSON-RPC 2.0 不定义如何鉴权，需要在传输层（如 HTTP Header）或应用层（如在 `params` 中加 `auth` 字段）自行实现。

### 3. 不包含类型定义

JSON-RPC 2.0 不强制类型检查，参数和返回值都是动态的 JSON。这意味着：
- 需要在运行时校验参数类型
- 无法在编译期发现类型错误
- 需要额外的文档或 Schema 定义（如 JSON Schema）

MCP 通过在协议层定义 Schema（如工具的 `inputSchema`）来弥补这一点。比如：

```json
{
  "name": "get_weather",
  "description": "获取天气信息",
  "inputSchema": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string",
        "description": "城市名称"
      }
    },
    "required": ["city"]
  }
}

```

### 4. 性能不如二进制协议

JSON 是文本格式，序列化和反序列化开销比 Protocol Buffers 大。但对于 MCP 的使用场景（大模型调用工具），这点性能差异通常可以忽略。

## 总结：为什么是 JSON-RPC 2.0？

MCP 选择 JSON-RPC 2.0，本质上是在**标准化、简单性、灵活性**之间做了权衡：

| 维度           | HTTP                 | gRPC                   | JSON-RPC 2.0                |
| -------------- | -------------------- | ---------------------- | --------------------------- |
| 定位           | 传输协议             | 完整 RPC 框架          | 消息格式规范                |
| 消息格式标准化 | ❌ 无统一标准         | ✅ Protocol Buffers     | ✅ 统一 JSON 结构            |
| 实现复杂度     | ⚠️ 需自行设计消息格式 | ⚠️ 需 .proto + 代码生成 | ✅ 简单，只需解析 JSON       |
| 人类可读性     | ⚠️ 取决于设计         | ❌ 二进制格式           | ✅ 纯文本 JSON               |
| 传输层灵活性   | ⚠️ 仅 HTTP            | ❌ 强依赖 HTTP/2        | ✅ 传输层无关                |
| 通知消息支持   | ❌ 需自行设计         | ⚠️ 需用流式 API         | ✅ 原生支持 Notification     |
| 批量调用支持   | ❌ 需自行设计         | ⚠️ 需用流式 API         | ✅ 原生支持 Batch            |
| 类型安全       | ❌ 无                 | ✅ 强类型               | ❌ 动态类型（需额外 Schema） |
| 性能           | ⚠️ 取决于实现         | ✅ 高性能               | ⚠️ JSON 序列化开销           |

对于 MCP 这种需要**跨语言、跨环境、易于调试、快速迭代**的协议，JSON-RPC 2.0 是最合适的选择。

它不是最快的（gRPC 更快），也不是最灵活的（HTTP 可以自由设计），但它是**最标准、最简单、最容易互操作**的。

这正是 MCP 需要的：**让不同实现之间能够无缝通信，而不是追求极致性能或极致灵活性**。