## 为什么需要 JSON-RPC？

对于 Java 开发者来说，远程调用并不是新鲜事。

在业务系统中，我们已经很熟悉 REST、Feign、Dubbo、gRPC 这些调用方式，它们解决的是“服务与服务之间如何通信”的问题。

但 MCP 要解决的，不完全是同一个层面的问题。

在 MCP 中，重点不在于某个业务接口是用 Spring MVC 还是 Dubbo 实现的，而在于：**当一个 MCP Client 想调用某个工具能力时，客户端和服务端是否有一套统一、标准、可互操作的消息格式。**

比如，一次调用中：

- 方法名怎么表示？
- 参数放在哪？
- 调用成功返回什么结构？
- 调用失败如何返回错误码和错误信息？
- 不需要响应的通知消息又该怎么表示？

如果这些内容没有统一规范，不同 MCP 实现之间就很难兼容，大模型也无法稳定地以相同方式调用外部工具。

因此，MCP 选择 JSON-RPC 2.0，并不是为了替代 Java 领域已有的微服务调用框架，而是为了在**协议层**统一“请求—响应—通知”的表达方式。

它定义的是一套消息结构标准，让不同语言、不同框架、不同运行环境下的 MCP Client 和 MCP Server，都能按照同一套规则通信。

所以可以把它理解为：

- **REST / Dubbo / gRPC**：偏业务接口调用方式
- **JSON-RPC 2.0**：偏协议消息封装规范
- **MCP**：在 JSON-RPC 2.0 之上，进一步约定工具、资源、提示词等能力模型

它本身并不绑定具体传输层，既可以跑在 HTTP 之上，也可以跑在 WebSocket、stdio 等通道之上。只要客户端和服务端都遵守 JSON-RPC 2.0 的约定，就能够以一致的方式完成请求、响应和通知。

但它也有明确的边界：**JSON-RPC 只定义消息格式与处理规则**，并不负责传输层细节，也不包含鉴权、服务发现等工程能力。这些需要在网关、框架或系统规范里另外补齐。

## 用一句话概括 JSON-RPC 2.0 是什么

JSON-RPC 是一种**轻量的远程过程调用（RPC）协议**，使用 JSON 作为消息格式。它的目标很简单：**用统一的 JSON 结构描述一次远程方法调用**。其设计哲学是“简单至上”（It is designed to be simple）。

需要注意版本：**本文讨论的是 JSON-RPC 2.0**。JSON-RPC 2.0 发布于 **2010 年 3 月**，并且与 JSON-RPC 1.0 **不兼容**。区分方式很简单：

- **JSON-RPC 2.0** 的请求和响应对象都必须包含 `"jsonrpc": "2.0"`
- **JSON-RPC 1.0** 没有这个字段

## JSON-RPC 2.0 的核心对象

从 Java 开发者的视角看，JSON-RPC 2.0 其实不复杂，核心无非就是几种固定的消息结构：

- **Request Object**：发起一次调用，请求服务端执行某个方法

- **Notification**：也是请求，但因为没有 `id`，所以不要求服务端回包

- **Response Object**：服务端对请求的响应，里面要么是结果，要么是错误

- **Batch**：把多个请求一起发出去，减少多次交互开销

除了消息结构，协议里还定义了两个交互角色：

- **Client**：发送请求的一方
- **Server**：接收请求并返回结果的一方

注意：**同一个程序可以同时扮演 Client 和 Server**。例如在微服务调用链中：

- 服务 A 调用服务 B 时，A 是 Client
- B 调用服务 C 时，B 又成为 Client

## 核心对象：Request、Notification、Response、Error

### 1. Request Object（请求对象）

一次 RPC 调用通过向服务端发送 **Request Object** 来表示。Request 对象包含以下字段。

**jsonrpc（必需）**

- 类型：String
- 值：必须精确等于 `"2.0"`
- 作用：标识协议版本

**method（必需）**

- 类型：String
- 含义：要调用的方法名
- 约束：以 `rpc.` 开头的方法名是**保留方法**，用于 RPC 内部或扩展，不应作为业务方法名

例如（合法方法名示例）：

- `"getUserInfo"`
- `"order.create"`
- `"calculateSum"`

**params（可选）**

- 类型：Object 或 Array
- 含义：方法调用参数
- 可以省略，表示无参数调用

两种参数形式：

- **按位置传参**：`params` 为 Array，例如 `[42, 23]`
- **按名称传参**：`params` 为 Object，例如 `{"minuend": 42, "subtrahend": 23}`
  使用命名参数时，字段名必须与服务端期望的参数名**完全匹配（包括大小写）**。

**id（条件必需）**

- 类型：String、Number 或 Null
- 作用：用于关联请求和响应

规则：

- 如果 **不存在 `id` 字段**，该请求被视为 **Notification**
- 服务端必须在 Response 中返回**相同的 id**

规范层面的建议：

- **避免使用 Null 作为 id**
- Number 类型 **不应使用小数**

Request 示例：

```json
{
  "jsonrpc": "2.0",
  "method": "getUserInfo",
  "params": {"userId": "12345"},
  "id": 1
}
```

### 2. Notification（通知）

Notification 是一种**没有 `id` 字段的 Request**。它表示客户端不期望收到任何响应。

规范要求：

- 服务端 **不得返回 JSON-RPC Response Object**
- 即使发生错误，也不会返回错误对象（客户端无法感知错误）

Notification 示例：

```json
{
  "jsonrpc": "2.0",
  "method": "logEvent",
  "params": {
    "level": "info",
    "message": "User logged in"
  }
}
```

如果使用 HTTP 作为传输层：服务器仍然需要返回 HTTP 响应（例如 `204 No Content`），但 **不会返回 JSON-RPC Response 对象**。

> 什么时候发送 Notification 请求？
>
> 当调用方不关心结果/错误，且不希望为这次调用付出一次响应的成本（等待、解析、重试、幂等等）时，用 Notification。比如：日志/埋点/旁路异步触发等，调用方只要“发出去”即可，不管任务结果或可以在后台处理。

### 3. Response Object（响应对象）

当服务端处理 Request 时，必须返回一个 **Response Object**，除非该请求是 Notification。Response 包含以下字段。

**jsonrpc（必需）**

- 值：必须精确等于 `"2.0"`

**result（成功时必需）**

- 含义：调用成功的返回值
- 规则：成功时必须存在；失败时不得存在

**error（失败时必需）**

- 含义：调用失败的错误信息
- 规则：失败时必须存在；成功时不得存在
- 值：必须是 Error Object

**id（必需）**

- 含义：用于关联请求和响应
- 规则：
  - 正常情况下必须与 Request 中的 `id` 相同
  - 如果服务端无法识别请求 ID（例如解析错误、无效请求），`id` 必须为 `null`

关键规则：`result` 和 `error` **必须有且只有一个存在**。

成功响应示例：

```json
{
  "jsonrpc": "2.0",
  "result": {
    "userId": "12345",
    "name": "张三",
    "role": "admin"
  },
  "id": 1
}
```

错误响应示例：

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": "userId is required"
  },
  "id": 1
}
```

### 4. Error Object（错误对象）

当 RPC 调用失败时，Response 必须包含 `error` 对象。Error Object 包含三个字段：

**code（必需）**

- 类型：Number
- 必须是整数
- 含义：错误类型

**message（必需）**

- 类型：String
- 含义：简短的人类可读错误描述

**data（可选）**

- 类型：任意 JSON 值
- 含义：额外错误信息（调试/上下文等）

------

## 规范预定义错误码

JSON-RPC 规范保留了 `-32768` 到 `-32000` 的错误码区间，其中部分被定义为标准错误：

| code            | message          | 含义                        |
| --------------- | ---------------- | --------------------------- |
| -32700          | Parse error      | JSON 解析错误               |
| -32600          | Invalid Request  | 请求不是有效的 Request 对象 |
| -32601          | Method not found | 方法不存在                  |
| -32602          | Invalid params   | 参数错误                    |
| -32603          | Internal error   | JSON-RPC 内部错误           |
| -32000 ~ -32099 | Server error     | 保留给实现定义              |

其余错误码空间可用于**应用自定义错误**。

Error 示例：

```json
{
  "code": -32601,
  "message": "Method not found",
  "data": "Method 'getUserInfo' is not registered"
}
```

## Batch（批处理）到底怎么用？

Batch 允许客户端一次发送多个 Request 对象，服务端批量处理后返回多个 Response。这样可以减少网络往返次数，提高吞吐量。但在实际实现中，Batch 也是 JSON-RPC 2.0 里**最容易被实现错误的一部分**。

### 1. Batch 的基本规则

**请求格式**

- 客户端可以发送一个 **Array**，其中包含多个 Request 对象
- Array 中可以混合 **普通 Request** 和 **Notification**

**响应格式**

- 服务端应返回一个 **Array**，包含对应的 Response 对象
- 每个 **Request** 应对应一个 **Response**
- **Notification 不应该返回 Response**
- Response 的顺序 **不要求与请求顺序一致**
- 客户端必须通过 `id` 字段来匹配请求与响应
- 服务端可以并发处理 Batch 中的请求

**特殊情况**

- 如果 Batch **不是有效 JSON**，服务端必须返回 **单个错误 Response**（不是 Array）
- 如果 Batch 是一个 **空 Array**，这也是无效请求，应返回单个 `Invalid Request`
- 如果 Batch 中 **没有任何需要返回 Response 的请求**（例如全部是 Notification），服务端 **不应返回 JSON-RPC 响应**

### 2. 示例：混合 Request 和 Notification

**请求**

```json
[
  {"jsonrpc": "2.0", "method": "sum", "params": [1, 2, 4], "id": "1"},
  {"jsonrpc": "2.0", "method": "notify_hello", "params": [7]},
  {"jsonrpc": "2.0", "method": "subtract", "params": [42, 23], "id": "2"},
  {"jsonrpc": "2.0", "method": "get_data", "id": "9"}
]
```

**响应**

```json
[
  {"jsonrpc": "2.0", "result": 7, "id": "1"},
  {"jsonrpc": "2.0", "result": 19, "id": "2"},
  {"jsonrpc": "2.0", "result": ["hello", 5], "id": "9"}
]
```

**要点**

- 请求中有 4 个对象，其中 `notify_hello` 是 Notification（没有 `id`）
- 因此响应只有 3 个对象
- 响应顺序 **不保证与请求一致**
- 客户端通过 `id` 进行匹配

### 3. 示例：全部是 Notification

**请求**

```json
[
  {"jsonrpc": "2.0", "method": "notify_sum", "params": [1, 2, 4]},
  {"jsonrpc": "2.0", "method": "notify_hello", "params": [7]}
]
```

**响应**

无 JSON-RPC 响应。

在 HTTP 传输场景下，服务器通常返回：

- `204 No Content`
- 或 `200 OK` 且 body 为空

关键点是：**不返回 JSON-RPC Response 对象**。

### 4. 错误示例：空 Array

**请求**

```json
[]
```

**响应**

```json
{
  "jsonrpc": "2.0",
  "error": {"code": -32600, "message": "Invalid Request"},
  "id": null
}
```

**原因**

规范要求 Batch **必须包含至少一个元素**。空数组不是有效请求。

### 5. 错误示例：Array 中不是 Request 对象

**请求**

```json
[1, 2, 3]
```

**响应**

```json
[
  {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null},
  {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null},
  {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null}
]
```

**原因**

Batch 中的每个元素都必须是 **Request Object**。
`1`、`2`、`3` 都是无效请求，因此分别返回错误。

### 6. 错误示例：Batch 不是合法 JSON

**请求**

```json
[
  {"jsonrpc": "2.0", "method": "sum", "params": [1, 2, 4], "id": "1"},
  {"jsonrpc": "2.0", "method"
]
```

**响应**

```json
{
  "jsonrpc": "2.0",
  "error": {"code": -32700, "message": "Parse error"},
  "id": null
}
```

**原因**

JSON 本身无法解析，因此返回 **Parse error**。

## 常见实现错误（实践中最容易踩的坑）

### 1. 忘记包含 `jsonrpc: "2.0"`

错误示例

```json
{
  "method": "getUserInfo",
  "params": {"userId": "12345"},
  "id": 1
}
```

原因：规范要求 Request 和 Response **必须包含 `jsonrpc` 字段**。

正确写法

```json
{
  "jsonrpc": "2.0",
  "method": "getUserInfo",
  "params": {"userId": "12345"},
  "id": 1
}
```

### 2. Notification 返回了响应

错误示例

客户端请求

```json
{
  "jsonrpc": "2.0",
  "method": "logEvent",
  "params": {"level": "info", "message": "User logged in"}
}
```

服务端响应

```json
{
  "jsonrpc": "2.0",
  "result": "OK",
  "id": null
}
```

原因：Notification **不允许返回 JSON-RPC Response**。

正确做法：不返回 JSON-RPC 响应（HTTP 场景可返回 204）。

### 3. `result` 和 `error` 同时存在

错误示例

```json
{
  "jsonrpc": "2.0",
  "result": {"userId": "12345"},
  "error": {"code": -32603, "message": "Internal error"},
  "id": 1
}
```

原因：规范要求 **两者只能存在一个**。

### 4. `method` 不是字符串

错误示例

```json
{
  "jsonrpc": "2.0",
  "method": 123,
  "params": [1, 2],
  "id": 1
}
```

原因：`method` **必须是字符串类型**。

### 5. `params` 不是 Object 或 Array

错误示例

```json
{
  "jsonrpc": "2.0",
  "method": "echo",
  "params": "hello",
  "id": 1
}
```

原因：`params` **必须是 Object 或 Array**。

正确示例

```json
{
  "jsonrpc": "2.0",
  "method": "echo",
  "params": ["hello"],
  "id": 1
}
```

### 6. 命名参数大小写不匹配

错误示例

```json
{
  "jsonrpc": "2.0",
  "method": "getUserInfo",
  "params": {"userid": "12345"},
  "id": 1
}
```

如果服务端期望参数名是 `userId`，则会返回 `Invalid params`。

JSON 字段名 **大小写敏感**。

### 7. Batch 全是 Notification 却返回 `[]`

错误示例

```json
[]
```

原因：如果 Batch 中 **没有需要返回的 Response**，规范要求 **不返回 JSON-RPC 响应**。

正确做法：

- HTTP 返回 `204 No Content`
- 或返回空 body

### 8. 业务错误使用了协议错误码

错误示例

```json
{
  "jsonrpc": "2.0",
  "error": {"code": -32603, "message": "User not found"},
  "id": 1
}
```

`-32603` 是 **JSON-RPC 内部错误**，不应该用于业务逻辑。

正确示例

```json
{
  "jsonrpc": "2.0",
  "error": {"code": 1001, "message": "User not found"},
  "id": 1
}
```

建议：

- `-32768 ~ -32000` 用于 **协议层错误**
- 业务错误使用 **自定义错误码（例如 1000+）**

## 工程落地建议：规范之外,你还得做这些

### 1. 输入校验：不要相信任何客户端

规范只定义了协议格式，但没说怎么校验业务参数。实际项目中，你需要在服务端做多层校验：

**协议层校验**（必须做）：
- `jsonrpc` 字段是否存在且等于 `"2.0"`
- `method` 字段是否存在且为 String 类型
- `params` 字段（如果存在）是否为 Object 或 Array
- `id` 字段（如果存在）是否为 String、Number 或 Null
- Request 和 Response 的 `result`/`error` 互斥性

**业务层校验**（根据方法定义）：
- 参数类型是否正确（如 `userId` 应该是 String，`age` 应该是 Number）
- 参数范围是否合法（如 `age` 应该在 0~150 之间）
- 必填参数是否存在
- 参数之间的逻辑关系（如 `startDate` 必须早于 `endDate`）

**安全层校验**（防攻击）：
- 参数长度限制（防止超大 JSON 攻击）
- 参数内容过滤（防止 SQL 注入、XSS）
- 方法名白名单（防止调用内部方法）
- 频率限制（防止暴力调用）

**伪代码示例**：

```java
// 协议层校验
if (!request.has("jsonrpc") || !request.get("jsonrpc").equals("2.0")) {
    return errorResponse(-32600, "Invalid Request: missing or invalid jsonrpc field");
}

// 业务层校验
if (method.equals("getUserInfo")) {
    if (!params.has("userId") || !params.get("userId").isString()) {
        return errorResponse(-32602, "Invalid params: userId must be a string");
    }
    String userId = params.getString("userId");
    if (userId.length() > 100) {
        return errorResponse(-32602, "Invalid params: userId too long");
    }
}
```

### 2. 超时与重试：网络不可靠

JSON-RPC 是无状态协议，不管传输层的可靠性。实际项目中，你需要自己处理超时和重试：

**客户端超时策略**：
- 设置合理的超时时间（如 5 秒、10 秒），不要无限等待
- 超时后返回明确的错误（如 `{"code": -32000, "message": "Request timeout"}`）
- 区分读超时（服务端没响应）和写超时（请求没发出去）

**客户端重试策略**：
- **幂等方法可以重试**（如查询、删除），非幂等方法慎重（如创建、支付）
- 重试时用**指数退避**（第一次等 1 秒，第二次等 2 秒，第三次等 4 秒）
- 重试次数有上限（如最多 3 次）
- 重试时**保持相同的 `id`**，方便服务端去重

**服务端超时处理**：
- 设置方法执行超时（如 30 秒），防止慢查询拖垮服务
- 超时后返回 `-32000 Server error`（或自定义错误码如 `5001 Execution timeout`）
- 记录超时日志，方便排查问题

### 3. 幂等与去重：同一个请求别处理两次

客户端重试时，服务端可能收到重复请求。你需要根据 `id` 字段做去重：

**去重策略**：
- 用 Redis 或内存缓存存储最近处理过的请求 ID（如最近 5 分钟）
- 收到请求时，先检查 `id` 是否已处理过
- 如果已处理，直接返回缓存的响应（不重复执行）
- 如果未处理，执行方法并缓存响应

**伪代码示例**：

```java
String requestId = request.getString("id");
if (cache.has(requestId)) {
    return cache.get(requestId); // 返回缓存的响应
}

Response response = executeMethod(request);
cache.set(requestId, response, 300); // 缓存 5 分钟
return response;
```

**注意事项**：
- 只对有 `id` 的请求做去重，Notification 不需要去重（因为没有响应）
- 去重窗口不要太长（如 5~10 分钟），避免内存占用过大
- 去重 key 可以加上客户端标识（如 `clientId:requestId`），避免不同客户端的 `id` 冲突

### 4. 日志与链路追踪：出问题能快速定位

JSON-RPC 调用链路可能很长（客户端 → 网关 → 服务 A → 服务 B），你需要记录完整的调用链路：

**日志记录内容**：
- 请求 ID（`id` 字段）
- 方法名（`method` 字段）
- 参数（`params` 字段，敏感信息脱敏）
- 响应结果（`result` 或 `error`）
- 执行耗时
- 客户端 IP、User-Agent
- 调用链路 ID（Trace ID，用于关联多个服务的日志）

**链路追踪**：
- 在 HTTP Header 或 JSON-RPC 扩展字段中传递 Trace ID（如 `X-Trace-Id`）
- 每个服务收到请求时，从 Header 中提取 Trace ID，记录到日志
- 调用下游服务时，把 Trace ID 传递下去
- 用 ELK、Jaeger、Zipkin 等工具聚合日志，可视化调用链路

**伪代码示例**：

```java
String traceId = request.getHeader("X-Trace-Id");
if (traceId == null) {
    traceId = UUID.randomUUID().toString();
}

logger.info("JSON-RPC request: traceId={}, id={}, method={}, params={}",
    traceId, request.get("id"), request.get("method"), maskSensitive(request.get("params")));

long startTime = System.currentTimeMillis();
Response response = executeMethod(request);
long duration = System.currentTimeMillis() - startTime;

logger.info("JSON-RPC response: traceId={}, id={}, duration={}ms, result={}",
    traceId, request.get("id"), duration, response);
```

### 5. 鉴权与签名：不是谁都能调你的接口

JSON-RPC 协议本身不管鉴权，你需要在传输层或应用层加鉴权机制：

**传输层鉴权**（推荐）：
- 用 HTTPS + API Key（在 HTTP Header 中传递，如 `Authorization: Bearer <token>`）
- 用 HTTPS + JWT（在 HTTP Header 中传递，服务端验证签名和过期时间）
- 用 mTLS（双向 TLS，客户端和服务端互相验证证书）

**应用层鉴权**（不推荐，但有时必须）：
- 在 JSON-RPC 请求中加鉴权字段（如 `{"jsonrpc": "2.0", "method": "getUserInfo", "params": {...}, "auth": {"token": "..."}, "id": 1}`）
- 服务端先验证 `auth` 字段，再执行方法
- 缺点：鉴权逻辑和业务逻辑耦合，不符合分层设计

**签名防篡改**：
- 客户端用私钥对请求内容签名，放在 HTTP Header 或 JSON-RPC 扩展字段中
- 服务端用公钥验证签名，确保请求未被篡改
- 签名内容包括：请求体 + 时间戳 + nonce（防重放攻击）

### 6. 兼容性与灰度：新老版本共存

实际项目中，你可能需要同时支持多个版本的 API（如 v1、v2），或者灰度发布新功能：

**版本管理策略**：
- 在方法名中加版本号（如 `getUserInfo_v1`、`getUserInfo_v2`）
- 在 HTTP 路径中加版本号（如 `/api/v1/jsonrpc`、`/api/v2/jsonrpc`）
- 在 JSON-RPC 扩展字段中加版本号（如 `{"jsonrpc": "2.0", "version": "v2", "method": "getUserInfo", ...}`）

**灰度发布策略**：
- 根据客户端标识（如 User-Agent、IP、用户 ID）路由到不同版本
- 用特性开关（Feature Flag）控制新功能的开启/关闭
- 监控新版本的错误率、响应时间，出问题快速回滚

### 7. 错误码映射：统一错误处理

JSON-RPC 预定义了 6 个错误码（`-32700` 到 `-32603`），但实际项目中你需要更多错误码：

**错误码分层**：
- **协议层错误**（`-32768` 到 `-32000`）：解析错误、方法不存在、参数无效等，直接用规范预定义的错误码
- **基础设施错误**（`-32000` 到 `-32099`）：数据库连接失败、Redis 超时、消息队列异常等，用规范保留的 Server error 范围
- **业务错误**（正数，如 `1000~9999`）：用户不存在、余额不足、订单已关闭等，自定义错误码

**错误码文档化**：
- 维护一个错误码表格，包含：错误码、错误消息、含义、解决方案
- 在 API 文档中公开错误码表格，方便客户端处理错误
- 用枚举或常量定义错误码，避免硬编码

**错误码示例**：

| code | message | 含义 | 解决方案 |
|------|---------|------|----------|
| -32700 | Parse error | JSON 解析失败 | 检查请求体是否是合法 JSON |
| -32600 | Invalid Request | 请求对象无效 | 检查 `jsonrpc`、`method`、`params` 字段 |
| -32601 | Method not found | 方法不存在 | 检查方法名是否正确 |
| -32602 | Invalid params | 参数无效 | 检查参数类型、范围、必填项 |
| -32603 | Internal error | 内部错误 | 联系技术支持 |
| -32001 | Database error | 数据库错误 | 稍后重试或联系技术支持 |
| 1001 | User not found | 用户不存在 | 检查用户 ID 是否正确 |
| 2001 | Insufficient balance | 余额不足 | 充值后重试 |

关于更详细的工程落地建议（如负载均衡、熔断降级、监控告警等），可以参考本系列的其他文档：
- 《工具调用架构设计指南》
- 《工具调用稳定性与安全保障》

## 一张图串起来：一次调用从客户端到服务端发生了什么？

下面用 PlantUML 时序图展示 JSON-RPC 2.0 的完整调用流程，包括普通 Request、Notification、错误处理、Batch 等场景。

![](https://oss.open8gu.com/iShot_2026-02-08_21.46.39.png)

**图示要点**：

1. **普通 Request（成功）**：客户端发送带 `id` 的请求 → 服务端校验格式 → 调用业务逻辑 → 返回带 `result` 的响应
2. **普通 Request（失败）**：业务逻辑抛出异常 → 服务端返回带 `error` 的响应，`id` 保持一致
3. **Notification**：客户端发送不带 `id` 的请求 → 服务端执行业务逻辑 → 不返回任何响应
4. **协议错误**：JSON 解析失败 → 服务端返回 Parse error，`id` 为 `null`（因为无法识别请求 ID）
5. **Batch Request**：客户端发送 Array → 服务端并发处理（可选）→ 只为有 `id` 的请求返回响应 → 响应顺序可以任意

## 文末小结

### 1. 核心要点

JSON-RPC 2.0 本质上是一套**远程调用的消息格式规范**。

它使用 JSON 来描述一次调用，但**只规定请求、响应、通知、错误这些消息该怎么表达**，并不绑定具体传输层，也不负责鉴权、服务发现、限流等工程能力。

理解这一点后，这一节其实只需要记住几条核心规则：

- 一次标准调用由 Request 和 Response 构成
- Notification 本质上也是 Request，只是因为没有 `id`，所以不要求服务端返回响应
- Response 中 `result` 和 `error` 必须二选一
- Batch 只是把多个请求或通知打包一起发送，是否返回响应仍然取决于每个请求是否带 `id`
- 协议内置错误码解决的是协议层问题，不等于业务异常码

如果从实现角度理解，可以把 JSON-RPC 2.0 看成：**用一套固定字段，把“调用哪个方法、传什么参数、成功返回什么、失败返回什么”表达清楚**。

### 2. 最容易踩的坑

实际编码时，最常见的问题通常不是“大方向理解错了”，而是这些细节没处理严谨：

- 忘记带上 `jsonrpc: "2.0"`，导致消息不符合协议格式

- 把 Notification 当成普通请求处理，错误地返回了 Response

- 响应结构写错，`result` 和 `error` 同时出现，或者两者都缺失

- `method`、`params` 类型不符合规范，导致请求无法正确解析

- 使用命名参数时，字段名与服务端约定不一致，尤其容易出现在大小写或命名风格上

- Batch 请求里全部都是 Notification，却仍然返回一个空数组 `[]`

- 把业务失败误写成协议错误，比如滥用 `-32603` 这类保留错误码

> JSON-RPC 2.0 解决的核心问题，不是业务逻辑怎么实现，而是**远程调用的消息该如何被标准化表达**。

### 3. 进一步阅读

**协议规范：**

- JSON-RPC 2.0 规范：https://www.jsonrpc.org/specification
- JSON 规范（RFC 4627）：https://www.ietf.org/rfc/rfc4627.txt
- RFC 2119（规范关键词定义）：https://www.ietf.org/rfc/rfc2119.txt

------

**本文协议语义依据**：JSON-RPC 2.0 Specification — https://www.jsonrpc.org/specification （发布于 2010-03-26）

