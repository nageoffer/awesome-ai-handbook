前面几篇 MCP 文章，你用 Spring AI 的 `@McpTool` 注解定义工具，用 `@McpResource` 暴露资源，用 `@McpPrompt` 注册提示词模板，60 行代码就能把一个 MCP Server 跑起来，Claude Desktop 和 Cursor 都能直接调。

但你有没有想过这么一个问题：当你在方法上标了 `@McpTool`，框架到底做了什么？

它是怎么从你的方法签名里提取出参数名、参数类型、参数描述，然后生成一份完整的 JSON Schema 的？Client 发过来的 JSON-RPC 请求是怎么路由到你的方法的？工具列表是怎么动态返回给 Client 的？

如果你只是用 Spring AI 搭 MCP Server，不搞清楚这些也没关系，框架帮你兜底了。但如果你遇到了这些场景：

- 你的项目不用 Spring Boot（比如纯 Java 应用、或者 Android 端要嵌入一个 MCP Client）
- 你想自定义 Transport 实现（比如走 WebSocket 而不是 SSE）
- 你想理解 Spring AI 为什么能自动发现你的工具，出了 bug 好排查
- 你想参与 MCP 生态建设，给社区贡献一个 MCP Server

这些都绕不开 MCP 的官方 Java SDK。这篇文章就来拆开看看，Spring AI 底下那一层到底长什么样。

## modelcontextprotocol：MCP 的官方大本营

### 1. 这个 GitHub 组织是什么

在 GitHub 搜索 MCP，前排结果基本都来自 [modelcontextprotocol](https://github.com/modelcontextprotocol) 组织——这是 Anthropic 主导成立的开放组织，专门维护 MCP 协议规范和各语言 SDK。

MCP 从设计之初就定位为开放标准：协议规范公开、SDK 全部 MIT 许可证、任何人都可以提 PR 贡献代码。这与 OpenAI 的 Function Call 形成对比——后者是 API 的一部分，由 OpenAI 单方面定义。MCP 的目标是成为大模型工具调用领域的通用协议。

### 2. 官方仓库全景

这个组织下有几个核心仓库，各自分工明确：

| 仓库 | 定位 | 说明 |
|------|------|------|
| `modelcontextprotocol` | 协议规范 | MCP 协议的完整定义，所有接口、消息格式、行为约束都在这里。所有 SDK 都以这份规范为基准来实现 |
| `java-sdk` | Java SDK | 这篇的主角，官方 Java 实现 |
| `typescript-sdk` | TypeScript SDK | 最早的 SDK 实现，也是参考实现（Reference Implementation）。TypeScript 版是第一个跑通协议全流程的 |
| `python-sdk` | Python SDK | 在 AI / ML 生态中使用最广的版本 |
| `servers` | 社区 MCP Server 集合 | 各种现成的 MCP Server：文件系统、GitHub、数据库、Web 搜索等 |

三个 SDK 面向不同的生态：TypeScript SDK 最早最全，是其他 SDK 的参照标准；Python SDK 在 AI 和数据科学领域用得最多；Java SDK 面向企业级应用，和 Spring AI 深度集成。

> 不同语言的 SDK 很多，这里仅列举三个语言举例。

Java SDK 当前稳定版本是 1.1.0，由 Anthropic 团队维护。

## Java SDK 全景：6 个模块各司其职

### 1. 模块总览

打开 Java SDK 的仓库，你会看到它不是一个单体模块项目，而是拆成了 6 个 Maven 模块：

| 模块 | Maven ArtifactId | 作用 |
|------|-------------------|------|
| `mcp-bom` | `mcp-bom` | BOM（Bill of Materials），统一管理所有模块的版本号，引入后不用逐个指定版本 |
| `mcp-core` | `mcp-core` | 核心实现，Client、Server、Transport、Schema 全部核心代码都在这里 |
| `mcp-json-jackson2` | `mcp-json-jackson2` | Jackson 2.x 的 JSON 序列化实现 |
| `mcp-json-jackson3` | `mcp-json-jackson3` | Jackson 3.x 的 JSON 序列化实现 |
| `mcp` | `mcp` | 便捷包，等于 `mcp-core` + `mcp-json-jackson3`，引入这一个就够了 |
| `mcp-test` | `mcp-test` | 测试工具和集成测试 |

大多数场景下，你只需要在 `pom.xml` 里引入一个依赖：

```xml
<dependency>
    <groupId>io.modelcontextprotocol.sdk</groupId>
    <artifactId>mcp</artifactId>
    <version>1.1.0</version>
</dependency>
```

这个 `mcp` 便捷包已经帮你打包了 `mcp-core`（核心实现）和 `mcp-json-jackson3`（JSON 序列化），拿来就能用。

### 2. 为什么 JSON 序列化要单独拆模块

你可能会好奇：JSON 序列化为什么不直接写在 `mcp-core` 里，还要单独拆两个模块出来？

原因是 Jackson 的版本冲突问题。Java 生态里 Jackson 几乎是 JSON 处理的事实标准，但 Jackson 存在大版本分裂：

- Jackson 2.x：Spring Boot 2.x 时代的标配，目前仍有大量项目在用
- Jackson 3.x：2025 年发布的新大版本，包名从 `com.fasterxml.jackson` 改成了 `tools.jackson`，不向后兼容

MCP SDK 把 JSON 序列化抽象成了 `McpJsonMapper` 接口，具体实现可以是 Jackson 2 或 Jackson 3。这样一来：

- 你的项目如果用 Jackson 2，就引入 `mcp-core` + `mcp-json-jackson2`，不会和现有依赖冲突
- 你的项目如果用 Jackson 3（或者新项目没有历史包袱），直接引入 `mcp` 便捷包，默认走 Jackson 3

这个设计在 Java 生态里很常见，Spring 框架对很多第三方库也是这么处理的——核心模块定义接口，具体实现按版本拆开。

### 3. Spring AI 集成模块去哪了

如果你翻过 Java SDK 早期版本（0.18.1 及之前），会发现仓库里还有两个模块：`mcp-spring-webflux` 和 `mcp-spring-webmvc`。但现在不见了。

从 1.0.0 版本开始，这两个 Spring 集成模块迁移到了 Spring AI 2.0 中。Maven 坐标也变了：

| 迁移前（SDK 仓库） | 迁移后（Spring AI） |
|-------|-------|
| `io.modelcontextprotocol.sdk:mcp-spring-webflux` | `org.springframework.ai:mcp-spring-webflux` |
| `io.modelcontextprotocol.sdk:mcp-spring-webmvc` | `org.springframework.ai:mcp-spring-webmvc` |

这也是为什么你用 Spring AI MCP Server 时，引入的是 `org.springframework.ai` 的依赖，而不是 `io.modelcontextprotocol.sdk`。

这个拆分的逻辑很清楚：SDK 本身保持轻量，只负责 MCP 协议的纯 Java 实现，不绑定任何框架；Spring 集成由 Spring AI 团队维护，和 Spring Boot 的自动配置、生命周期管理深度绑定。各管各的，互不耦合。

## SDK 核心架构：四层分明

了解完模块划分，接下来看 SDK 内部的代码结构。整个 `mcp-core` 模块的代码分成四层，从下往上依次是：

![](https://oss.open8gu.com/iShot_2026-03-17_10.28.48.png)

每一层的职责很清晰，从下往上看：Schema 层定义协议里有哪些消息类型，Transport 层负责把消息送达，Session 层管理一次完整的连接会话，Client/Server 层是开发者直接打交道的 API。

### 1. Schema 层：协议的字典

`McpSchema` 是整个 SDK 里最大的一个类——2786 行代码，全是数据结构定义。它就像一本字典，规定了 Client 和 Server 之间能说哪些话、每句话的格式是什么。

![](https://oss.open8gu.com/image-20260318095828289.png)

之前讲 MCP 协议规范 JSON-RPC 2.0 那篇文章里，大家已经知道 MCP 底层用 JSON-RPC 2.0 通信。`McpSchema` 做的事情就是把协议规范里定义的所有消息类型，都变成了 Java 的 Record 类（Java 16+ 的不可变数据类）。

挑几个你最常接触的类型：

| 类型 | 对应的协议操作 | 说明 |
|------|-------------|------|
| `McpSchema.Tool` | 工具定义 | 包含工具名称、描述、参数的 JSON Schema |
| `McpSchema.CallToolRequest` | `tools/call` 请求 | Client 调用工具时发的请求，包含工具名和参数 |
| `McpSchema.CallToolResult` | `tools/call` 响应 | Server 执行工具后返回的结果 |
| `McpSchema.Resource` | 资源定义 | 包含资源 URI、名称、描述、MIME 类型 |
| `McpSchema.Prompt` | Prompt 模板定义 | 包含模板名称、描述、参数列表 |
| `McpSchema.InitializeRequest` | `initialize` 请求 | 连接建立时的握手请求 |
| `McpSchema.InitializeResult` | `initialize` 响应 | 握手响应，包含 Server 的能力声明 |

这些类型最终会被包装成 JSON-RPC 的 Request / Response 在 Client 和 Server 之间传输。比如 Client 调用工具时，实际发出的 JSON-RPC 消息长这样：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "getUserAnnualLeave",
    "arguments": { "employeeId": "E001" }
  }
}
```

其中 `params` 部分就是 `McpSchema.CallToolRequest` 对象序列化后的结果。

### 2. Transport 层：消息的快递公司

Transport 层的职责很单一：把 JSON-RPC 消息从一端送到另一端。它不关心消息内容是什么（工具调用还是资源读取），只管送达。

SDK 提供了三种 Transport 实现，可以理解为三家不同的快递公司，各有各的配送方式：

#### 2.1 Stdio Transport

通过进程的 stdin / stdout 传输消息。MCP Server 作为子进程启动，Client（Host 应用）通过标准输入输出和它通信。每条消息是一行 JSON 文本，用换行符分隔。

你在 Claude Desktop 里配置本地 MCP Server 时，用的就是这种方式。Claude Desktop 启动你的 Java 进程，然后通过 stdin 发请求、从 stdout 读响应。

关键类：

- 客户端：`StdioClientTransport`
- 服务端：`StdioServerTransportProvider`

#### 2.2 SSE Transport

基于 Server-Sent Events 的传输。如果你读过之前的 SSE 系列文章，对这个应该不陌生。

SSE 本身是单向的（只能 Server 向 Client 推送），所以 MCP 的 SSE Transport 实际上走了两条路：

- Server → Client：通过 SSE 长连接推送事件（响应、通知）
- Client → Server：通过单独的 HTTP POST 请求发送（请求）

关键类：

- 客户端：`HttpClientSseClientTransport`（基于 JDK 内置的 `HttpClient`，不依赖第三方 HTTP 库）
- 服务端：`HttpServletSseServerTransportProvider`（基于 Jakarta Servlet）

#### 2.3 Streamable HTTP Transport

SSE Transport 的进化版。双向都走 HTTP，Client 的请求和 Server 的响应都可以流式传输，不需要维持一个额外的 SSE 长连接，按需建立连接即可。

这是 MCP 协议较新引入的传输方式，也是官方推荐的远程传输方案。

关键类：

- 客户端：`HttpClientStreamableHttpTransport`
- 服务端：`HttpServletStreamableServerTransportProvider`

#### 2.4 三种 Transport 怎么选

| 维度 | Stdio | SSE | Streamable HTTP |
|------|-------|-----|-----------------|
| 通信方式 | stdin / stdout | SSE 推送 + HTTP POST | 双向 HTTP 流 |
| 适用范围 | 仅本地 | 本地 + 远程 | 本地 + 远程 |
| 网络要求 | 无（进程间通信） | HTTP | HTTP |
| 实现复杂度 | 低 | 中 | 中 |
| 生产就绪度 | 适合本地开发 | 可用于生产 | 推荐生产使用 |
| 典型场景 | Claude Desktop 本地工具 | 传统 Web 环境 | 企业级远程部署 |

简单来说：**本地开发调试用 Stdio，生产环境远程部署用 Streamable HTTP，SSE 作为兼容性较好的过渡方案**。

### 3. Session 层：对话的窗口

Transport 只管送消息，但一次完整的 MCP 通信不只是单条消息的收发——Client 和 Server 建立连接后，需要先握手（`initialize`），交换各自的能力声明（我支持哪些功能、你支持哪些功能），然后才能正式收发请求。

Session 层管理的就是这整个连接生命周期。它负责三件事：

1. **维护连接状态**：是否已初始化、是否已关闭
2. **请求-响应配对**：每个 JSON-RPC 请求有一个 `id`，Session 要把收到的响应按 `id` 路由回对应的请求（因为可能同时有多个请求在等待响应）
3. **处理通知消息**：JSON-RPC 除了请求-响应，还有通知（Notification）——不需要响应的单向消息，比如工具列表变更通知

关键类：`McpClientSession`（客户端会话）、`McpServerSession`（服务端会话）。

### 4. Client/Server 层：开发者直接打交道的 API

最上层就是你实际使用的 API 了。SDK 提供了 Builder 模式来创建 Client 和 Server，API 设计简洁清晰。

#### 4.1 McpServer

`McpServer` 是服务端的入口类，提供两个静态方法：

- `McpServer.sync(transportProvider)` ——创建同步 Server
- `McpServer.async(transportProvider)` ——创建异步 Server

通过 Builder 链式调用注册工具、资源、Prompt：

```java
McpSyncServer server = McpServer.sync(transportProvider)
    .serverInfo(new McpSchema.Implementation("my-server", "1.0.0"))
    .tool(
        new McpSchema.Tool("getUserAnnualLeave", "查询员工剩余年假天数", jsonSchema),
        (exchange, request) -> {
            String employeeId = request.arguments().get("employeeId").toString();
            String result = "员工 " + employeeId + " 剩余年假：5 天";
            return new McpSchema.CallToolResult(
                List.of(new McpSchema.TextContent(result)), false
            );
        }
    )
    .build();
```

#### 4.2 McpClient

`McpClient` 是客户端的入口类，同样提供同步和异步两个版本：

```java
McpSyncClient client = McpClient.sync(transport)
    .clientInfo(new McpSchema.Implementation("my-client", "1.0.0"))
    .build();

// 建立连接，完成握手
client.initialize();

// 发现 Server 端有哪些工具
McpSchema.ListToolsResult toolsResult = client.listTools();
for (McpSchema.Tool tool : toolsResult.tools()) {
    System.out.println("工具：" + tool.name() + " - " + tool.description());
}

// 调用工具
Map<String, Object> args = Map.of("employeeId", "E001");
McpSchema.CallToolResult result = client.callTool(
    new McpSchema.CallToolRequest("getUserAnnualLeave", args)
);

// 关闭连接
client.closeGracefully();
```

#### 4.3 同步 vs 异步怎么选

SDK 提供了同步和异步两套 API，分别对应 `McpSyncClient` / `McpAsyncClient` 和 `McpSyncServer` / `McpAsyncServer`。

异步 API 基于 Project Reactor（一个响应式编程库），返回 `Mono<T>` 和 `Flux<T>` 类型。如果你用过 Spring WebFlux，对这套东西应该不陌生。

实际上同步 API 内部就是对异步 API 的 `.block()` 封装——调用异步方法然后阻塞等待结果。所以两套 API 的底层实现是同一套代码。

怎么选：

- **传统 Spring MVC 项目**（阻塞式）：用同步 API，简单直观
- **Spring WebFlux 项目**（响应式）或需要处理大量并发连接：用异步 API
- **不确定**：用同步，等遇到性能瓶颈再切异步

## 实战：用纯 SDK 构建 MCP Server

光看架构图不够直观，下面用纯 SDK（不依赖 Spring AI）来搭一个 MCP Server，实现两个工具：`getUserAnnualLeave`（查年假）和 `getOrderStatus`（查订单状态）。和之前 Spring AI 版本做个对比，你就能感受到 Spring AI 帮你省了多少事。

### 1. Maven 依赖

不需要 Spring Boot，只要一个依赖：

```xml
<dependencies>
    <dependency>
        <groupId>io.modelcontextprotocol.sdk</groupId>
        <artifactId>mcp</artifactId>
        <version>1.1.0</version>
    </dependency>
</dependencies>
```

### 2. 完整代码

```java
import io.modelcontextprotocol.server.McpServer;
import io.modelcontextprotocol.server.McpSyncServer;
import io.modelcontextprotocol.server.transport.StdioServerTransportProvider;
import io.modelcontextprotocol.spec.McpSchema;

import java.util.List;
import java.util.Map;
import java.util.concurrent.CountDownLatch;

public class EnterpriseMcpServer {

    public static void main(String[] args) throws InterruptedException {
        // 1. 创建 Transport：使用 Stdio 方式（通过 stdin/stdout 通信）
        StdioServerTransportProvider transportProvider = new StdioServerTransportProvider();

        // 2. 构建 Server，注册工具
        McpSyncServer server = McpServer.sync(transportProvider)
            .serverInfo(new McpSchema.Implementation("enterprise-server", "1.0.0"))
            // 注册工具：查年假
            .tool(
                buildAnnualLeaveTool(),
                (exchange, request) -> handleAnnualLeave(request)
            )
            // 注册工具：查订单状态
            .tool(
                buildOrderStatusTool(),
                (exchange, request) -> handleOrderStatus(request)
            )
            .build();

        System.err.println("Enterprise MCP Server 已启动，等待 Client 连接...");

        // 3. 注册关闭钩子，进程退出时优雅关闭 Server
        Runtime.getRuntime().addShutdownHook(new Thread(server::close));

        // 4. 阻塞主线程，保持进程存活
        // build() 内部的 stdin 监听线程是守护线程，如果 main 方法结束，JVM 会直接退出
        // 所以必须手动阻塞主线程，让 Server 持续运行
        new CountDownLatch(1).await();
    }

    // ========== 工具定义 ==========

    /**
     * 构建查年假工具的定义
     * 需要手动拼 JSON Schema 来描述参数
     */
    private static McpSchema.Tool buildAnnualLeaveTool() {
        // 参数的 JSON Schema
        String inputSchema = """
            {
                "type": "object",
                "properties": {
                    "employeeId": {
                        "type": "string",
                        "description": "员工工号，如 E001"
                    }
                },
                "required": ["employeeId"]
            }
            """;

        return new McpSchema.Tool(
            "getUserAnnualLeave",
            "查询员工剩余年假天数，包括总天数、已使用天数、剩余天数",
            McpSchema.JsonSchema.fromJson(inputSchema)
        );
    }

    /**
     * 构建查订单状态工具的定义
     */
    private static McpSchema.Tool buildOrderStatusTool() {
        String inputSchema = """
            {
                "type": "object",
                "properties": {
                    "orderId": {
                        "type": "string",
                        "description": "订单编号，如 ORD-20260301-001"
                    }
                },
                "required": ["orderId"]
            }
            """;

        return new McpSchema.Tool(
            "getOrderStatus",
            "查询订单的物流状态和详细信息",
            McpSchema.JsonSchema.fromJson(inputSchema)
        );
    }

    // ========== 工具处理函数 ==========

    private static McpSchema.CallToolResult handleAnnualLeave(McpSchema.CallToolRequest request) {
        // 从请求中提取参数
        String employeeId = request.arguments().get("employeeId").toString();

        // 实际项目中这里调用 HR 系统查询
        String result = String.format(
            "员工 %s 的年假信息：总年假 15 天，已使用 10 天，剩余 5 天", employeeId
        );

        return new McpSchema.CallToolResult(
            List.of(new McpSchema.TextContent(result)),
            false  // isError = false，表示执行成功
        );
    }

    private static McpSchema.CallToolResult handleOrderStatus(McpSchema.CallToolRequest request) {
        String orderId = request.arguments().get("orderId").toString();

        // 实际项目中这里调用订单系统查询
        String result = String.format(
            "订单 %s 状态：已发货，快递单号 SF1234567890，预计明天送达", orderId
        );

        return new McpSchema.CallToolResult(
            List.of(new McpSchema.TextContent(result)),
            false
        );
    }
}
```

### 3. 代码解读

对照代码，你可以看到用纯 SDK 构建 MCP Server 需要做四件事：

**第一步：创建 Transport**。这里用 `StdioServerTransportProvider`，表示通过 stdin/stdout 和 Client 通信。如果要远程部署，换成 `HttpServletSseServerTransportProvider` 或 `HttpServletStreamableServerTransportProvider`。

**第二步：构建工具定义**。这是最费劲的部分——你需要手动拼 JSON Schema 来描述每个工具的参数。参数名、参数类型、参数描述、是否必填，都要自己写。对比一下 Function Call 那篇文章里用 Gson 手动构建 JSON Schema 的代码，结构是一样的，只是 SDK 提供了 `McpSchema.Tool` 这个类来承载。

**第三步：编写处理函数**。每个工具需要一个处理函数，接收 `McpSchema.CallToolRequest`（包含工具名和参数），返回 `McpSchema.CallToolResult`（包含执行结果）。

**第四步：阻塞主线程保活**。这一步容易被忽略。`build()` 方法返回后不会阻塞主线程，而 SDK 内部监听 stdin 的线程是守护线程（Daemon Thread）——守护线程不会阻止 JVM 退出。如果 `main` 方法执行完就结束了，JVM 会立刻退出，Server 根本来不及处理任何请求。所以必须用 `new CountDownLatch(1).await()` 手动阻塞主线程，让进程一直活着。同时注册 `ShutdownHook`，在进程被终止时（比如 Ctrl+C）优雅关闭 Server。

> 注意 `System.err.println` 而不是 `System.out.println`。因为 Stdio Transport 用 stdout 传输 JSON-RPC 消息，如果你往 stdout 打日志，会和协议消息混在一起，Client 就解析不了了。日志输出要走 stderr。

### 4. 对比：SDK 直接写 vs Spring AI 注解

同一个查年假工具，两种方式的代码量对比：

**Spring AI 方式——3 行搞定：**

```java
@McpTool(description = "查询员工剩余年假天数，包括总天数、已使用天数、剩余天数")
public String getUserAnnualLeave(
        @McpToolParam(description = "员工工号，如 E001") String employeeId) {
    return String.format("员工 %s 的年假信息：总年假 15 天，已使用 10 天，剩余 5 天", employeeId);
}
```

**SDK 方式——需要 30+ 行：**

```java
// 1. 手动拼 JSON Schema（10+ 行）
String inputSchema = """
    {
        "type": "object",
        "properties": {
            "employeeId": {
                "type": "string",
                "description": "员工工号，如 E001"
            }
        },
        "required": ["employeeId"]
    }
    """;
McpSchema.Tool tool = new McpSchema.Tool(
    "getUserAnnualLeave",
    "查询员工剩余年假天数，包括总天数、已使用天数、剩余天数",
    McpSchema.JsonSchema.fromJson(inputSchema)
);

// 2. 手动编写处理函数，手动提取参数（5+ 行）
BiFunction<McpSyncServerExchange, McpSchema.CallToolRequest, McpSchema.CallToolResult> handler =
    (exchange, request) -> {
        String employeeId = request.arguments().get("employeeId").toString();
        String result = String.format("员工 %s 的年假信息：...", employeeId);
        return new McpSchema.CallToolResult(
            List.of(new McpSchema.TextContent(result)), false
        );
    };

// 3. 注册到 Server Builder
McpServer.sync(transport).tool(tool, handler).build();
```

差距一目了然。Spring AI 帮你做了三件事：

1. **从方法签名自动生成 JSON Schema**：方法参数名变成 Schema 的 `properties`，参数类型变成 `type`，`@McpToolParam` 的 `description` 变成参数描述，方法名变成工具名
2. **自动把请求参数映射到方法参数**：Client 传过来的 `arguments` 里的值，自动按名称匹配到方法参数上
3. **自动把返回值包装成 `CallToolResult`**：你返回一个 `String`，框架帮你包成 `TextContent` 再包成 `CallToolResult`

理解了这些，你就知道 `@McpTool` 注解不是魔法，而是对 SDK API 的工程封装。

## 实战：用纯 SDK 构建 MCP Client

Server 端写完了，再来看 Client 端。下面这段代码展示了一个 MCP Client 的完整生命周期：连接 Server → 发现工具 → 调用工具 → 关闭连接。

### 1. 完整代码

```java
import io.modelcontextprotocol.client.McpClient;
import io.modelcontextprotocol.client.McpSyncClient;
import io.modelcontextprotocol.client.transport.StdioClientTransport;
import io.modelcontextprotocol.spec.McpSchema;

import java.util.Map;

public class EnterpriseMcpClient {

    public static void main(String[] args) {
        // 1. 创建 Transport，指向 Server 进程
        // ServerParameters 定义了要启动的子进程命令
        StdioClientTransport transport = StdioClientTransport.builder("java")
            .args("-jar", "enterprise-mcp-server.jar")
            .build();

        // 2. 创建 Client
        McpSyncClient client = McpClient.sync(transport)
            .clientInfo(new McpSchema.Implementation("enterprise-client", "1.0.0"))
            .build();

        try {
            // 3. 建立连接，完成握手
            client.initialize();
            System.out.println("已连接到 MCP Server");

            // 4. 发现工具
            McpSchema.ListToolsResult toolsResult = client.listTools();
            System.out.println("Server 提供了 " + toolsResult.tools().size() + " 个工具：");
            for (McpSchema.Tool tool : toolsResult.tools()) {
                System.out.println("  - " + tool.name() + "：" + tool.description());
            }

            // 5. 调用工具：查年假
            McpSchema.CallToolResult leaveResult = client.callTool(
                new McpSchema.CallToolRequest(
                    "getUserAnnualLeave",
                    Map.of("employeeId", "E001")
                )
            );
            System.out.println("\n查年假结果：");
            for (McpSchema.Content content : leaveResult.content()) {
                if (content instanceof McpSchema.TextContent text) {
                    System.out.println("  " + text.text());
                }
            }

            // 6. 调用工具：查订单
            McpSchema.CallToolResult orderResult = client.callTool(
                new McpSchema.CallToolRequest(
                    "getOrderStatus",
                    Map.of("orderId", "ORD-20260301-001")
                )
            );
            System.out.println("\n查订单结果：");
            for (McpSchema.Content content : orderResult.content()) {
                if (content instanceof McpSchema.TextContent text) {
                    System.out.println("  " + text.text());
                }
            }

        } finally {
            // 7. 关闭连接
            client.closeGracefully();
            System.out.println("\n连接已关闭");
        }
    }
}
```

运行输出：

```text
已连接到 MCP Server
Server 提供了 2 个工具：
  - getUserAnnualLeave：查询员工剩余年假天数，包括总天数、已使用天数、剩余天数
  - getOrderStatus：查询订单的物流状态和详细信息

查年假结果：
  员工 E001 的年假信息：总年假 15 天，已使用 10 天，剩余 5 天

查订单结果：
  订单 ORD-20260301-001 状态：已发货，快递单号 SF1234567890，预计明天送达

连接已关闭
```

### 2. 交互流程

Client 和 Server 之间的完整交互流程用时序图表示：

![](https://oss.open8gu.com/iShot_2026-03-17_10.28.49.png)

整个流程和 HTTP 的请求-响应很像，只不过多了一个握手阶段。握手时双方交换能力声明——Server 告诉 Client 自己支持哪些功能（Tools？Resources？Prompts？），Client 也告诉 Server 自己支持哪些功能（比如是否支持 Sampling）。握手完成后才能正式通信。

### 3. 工具变更监听

在实际场景中，Server 端的工具列表可能会动态变化——比如运维人员通过管理后台新增了一个工具，或者某个工具因为故障被下线了。SDK 提供了 `toolsChangeConsumer` 来监听这种变化：

```java
McpSyncClient client = McpClient.sync(transport)
    .clientInfo(new McpSchema.Implementation("my-client", "1.0.0"))
    .toolsChangeConsumer(tools -> {
        System.out.println("工具列表已更新，当前工具数量：" + tools.size());
        for (McpSchema.Tool tool : tools) {
            System.out.println("  - " + tool.name());
        }
    })
    .build();
```

当 Server 端发出工具列表变更通知时，这个回调会被自动触发。在 Spring AI 注解方式中，这个监听是自动处理的，你不需要手动注册。

> Server 需要主动发送通知，Client 才能感知到变化。如果 Server 端没有实现通知机制，这个回调不会触发。SDK 内部收到变更通知后会自动获取最新工具列表，然后传给回调。

## Transport 机制详解

前面在架构分层里简单介绍了三种 Transport，这里展开说说它们各自的通信原理。

### 1. Stdio：进程间的传纸条

Stdio Transport 的原理最简单——两个进程之间通过标准输入输出传递消息：

![](https://oss.open8gu.com/iShot_2026-03-17_10.28.50.png)

优点是简单、安全（不暴露网络端口）。缺点是只能本地通信，而且进程的生命周期管理比较复杂——Host 要负责启动 Server 子进程、监控进程状态、在不需要时杀掉进程。

适合的场景：本地开发调试、Claude Desktop / Cursor 集成本地工具。

### 2. SSE：基于事件流的远程通信

SSE Transport 用于远程场景，通信走 HTTP 协议。和之前 SSE 系列文章里讲的一样，SSE 本身只支持服务端向客户端单向推送，所以 MCP 的 SSE Transport 实际上是两条路：

![](https://oss.open8gu.com/iShot_2026-03-17_10.28.51.png)

Client 先通过 GET 请求建立 SSE 长连接，Server 通过这条连接推送响应和通知。Client 发请求时走单独的 HTTP POST。

优点是基于标准 HTTP，穿越防火墙和反向代理方便，SSE 自带断线重连机制。缺点是 Client → Server 方向不是流式的，每次要发一个完整的 HTTP POST 请求。

### 3. Streamable HTTP：双向流式通信

Streamable HTTP 是 MCP 协议较新推出的传输方式，简化了 SSE Transport 的双通道架构：

- Client 的请求通过 HTTP POST 发送，响应体可以是普通 JSON（简单请求），也可以是 SSE 流（需要流式返回或服务器主动推送时）
- 不需要预先建立 SSE 长连接，按需建立连接
- Server 可以在响应头里返回 Session ID，Client 后续请求带上这个 ID 来关联会话

相比 SSE Transport，Streamable HTTP 更简洁——不需要维护一个额外的 SSE 长连接通道，每个 POST 请求的响应本身就可以是流式的。

这是 MCP 协议目前主推的远程传输方式，适合企业级生产环境部署。

## Spring AI 与官方 SDK 的关系

### 1. 一张图看清层次

你用 Spring AI 写的 MCP Server，底层的依赖链条是这样的：

![](https://oss.open8gu.com/iShot_2026-03-17_10.28.52.png)

### 2. Spring AI 在 SDK 之上做了什么

具体来说，Spring AI MCP Starter 在官方 SDK 之上封装了三件事：

**注解驱动的工具注册**：扫描所有标注了 `@McpTool` / `@McpResource` / `@McpPrompt` 的 Bean 和方法，通过反射读取方法签名和注解属性，自动转换为 SDK 的 `McpSchema.Tool` / `McpSchema.Resource` / `McpSchema.Prompt` 对象，再调用 SDK 的 Builder API 注册到 Server 上。

**自动配置 Transport**：根据你引入的 Starter 依赖自动选择 Transport 实现：

| 引入的 Starter | 自动配置的 Transport |
|---------------|---------------------|
| `spring-ai-starter-mcp-server` | `StdioServerTransportProvider` |
| `spring-ai-starter-mcp-server-webmvc` | `HttpServletSseServerTransportProvider` |
| `spring-ai-starter-mcp-server-webflux` | WebFlux 版 Transport |

**Spring Boot 生命周期管理**：Server 的启动和关闭与 Spring 容器的生命周期绑定——容器启动时自动构建并启动 MCP Server，容器关闭时自动优雅关闭 Server。你不需要手动写 `server.build()` 和 `server.closeGracefully()`。

### 3. 什么时候该用 SDK，什么时候用 Spring AI

大多数 Java 项目直接用 Spring AI 就够了——省事，约定优于配置，和 Spring Boot 生态无缝集成。

需要用纯 SDK 的场景：

| 场景 | 为什么需要 SDK |
|------|-------------|
| 非 Spring 项目 | 纯 Java 应用、Android 应用、Vert.x 项目等，没有 Spring Boot 环境 |
| 自定义 Transport | 想实现 WebSocket Transport 或其他自定义传输方式 |
| 深度控制连接管理 | 需要手动管理 Session 生命周期、自定义握手逻辑 |
| MCP 生态贡献 | 给社区写 MCP Server / Client 库、写 Transport 适配器 |
| 源码级排查问题 | 理解 SDK 架构后，Spring AI 层面出了问题能往下追 |

## 版本演进与迁移

### 1. 0.18.1 → 1.0.0：三个 Breaking Changes

Java SDK 在 2025 年初从 0.x 版本升级到了 1.0.0 正式版，有三个不兼容变更需要注意：

1. **Jackson 3 成为默认**：`mcp` 便捷包默认包含 Jackson 3 而不是 Jackson 2。如果你的项目用 Jackson 2，需要改为引入 `mcp-core` + `mcp-json-jackson2`

2. **Spring 集成模块迁移**：Maven 坐标从 `io.modelcontextprotocol.sdk:mcp-spring-*` 变成了 `org.springframework.ai:mcp-spring-*`。如果你从旧版升级，要改 `pom.xml` 的依赖声明

3. **`tool()` 方法签名调整**：旧版的 `tool()` 注册方法在 0.18.1 就已经标记为废弃，1.0.0 正式移除，改为 `toolCall()` 方法名

### 2. 未来方向：2.0.0

SDK 目前正在向 2.0.0 版本演进，对标 MCP 2025-11-25 规范修订版。几个值得关注的新特性：

- **Tasks 管理**：支持长时间运行的任务，Client 可以查询任务进度、取消任务
- **Elicitation**：Server 可以主动向 Client 请求用户输入（比如弹出确认对话框）
- **安全增强**：更完善的认证和授权机制
- **Virtual Threads 支持**：利用 Java 21 的虚拟线程特性提升并发性能

SDK 还在快速演进中，了解底层架构有助于你跟上后续版本的变化。

## 小结

这篇文章从 Spring AI 的 `@McpTool` 注解往下挖了一层，看了看 MCP 官方 Java SDK 的全貌。核心要点：

- `modelcontextprotocol` 是 MCP 的官方 GitHub 组织，由 Anthropic 主导，维护协议规范和 TypeScript / Python / Java 三个语言的 SDK
- Java SDK 分 6 个模块，核心是 `mcp-core`，大多数场景引入 `mcp` 这个便捷包就够了。JSON 序列化单独拆模块是为了避免 Jackson 2 / 3 版本冲突
- SDK 架构四层分明：**Schema 层**定义协议的所有消息类型 → **Transport 层**负责消息收发（Stdio / SSE / Streamable HTTP 三种方式）→ **Session 层**管理连接生命周期和请求-响应配对 → **Client/Server 层**提供开发者直接使用的 Builder API
- 用纯 SDK 可以不依赖 Spring 构建 MCP Server 和 Client，但需要手动拼 JSON Schema、手动提取参数、手动管理连接生命周期——这些正是 Spring AI 帮你自动化的部分
- 三种 Transport 各有适用场景：**Stdio** 适合本地开发和 Claude Desktop 集成，**SSE** 适合传统 Web 环境的远程通信，**Streamable HTTP** 是官方推荐的生产级远程传输方案
- Spring AI MCP 是官方 SDK 的上层封装，核心做了三件事：注解驱动的工具注册、自动配置 Transport、Spring Boot 生命周期管理。大多数项目用 Spring AI 就够了，需要深度定制或在非 Spring 环境中使用时才需要直接操作 SDK

如果大家在实践中遇到问题或有其他想法，欢迎留言交流。

**参考链接：**

- [MCP 官方 GitHub 组织](https://github.com/modelcontextprotocol)
- [MCP Java SDK](https://github.com/modelcontextprotocol/java-sdk)
- [MCP 协议规范](https://modelcontextprotocol.io)
- [Spring AI MCP 文档](https://docs.spring.io/spring-ai/reference/2.0/api/mcp/mcp-overview.html)

> Spring AI MCP 文档引用的是未发布的 2.0.0 版本，后续可能还会演进到 3.0.0 版本，大家注意 URL 中的版本号。