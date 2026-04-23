上一篇讲了 Function Call，你已经能让 RAG 系统从只能查知识库升级到能查数据、能调接口、能干活。模型自己判断什么时候该调工具，输出标准化的调用意图，你的代码执行函数——整个流程跑得很顺。

但文章最后留了一个尾巴：Function Call 很好用，工具一多就管不过来了。

假设你在一家公司做企业知识库助手，一开始只有两个工具：查年假、查订单。你手写两份 JSON Schema，代码里写两个 if-else 路由，没什么问题。但半年过去了，产品经理不断加需求：查考勤、查销售数据、查会议室、查报销进度、查项目排期、查库存、查物流、查合同……工具从 2 个变成了 20 个。

这时候你会发现：

- 20 个工具 × 每个工具 5~10 个参数 = 几百行 JSON Schema 要手写和维护
- Python 团队写了一个数据分析工具，你的 Java 系统调不了
- 新来的同事问“系统里有哪些工具可用”，你翻了半天代码才找全
- 某个工具的参数改了，JSON Schema 忘了同步更新，模型传了错误的参数，线上出了 bug

你需要的不是更多的 if-else，而是一个标准化的工具管理协议。这就是今天要讲的 MCP。

## Function Call 的痛点：工具一多就管不过来

### 1. 回顾：Function Call 做对了什么

在展开痛点之前，先肯定一下 Function Call 的核心价值：

- **让模型自己判断什么时候该调工具**：不用写规则匹配，模型理解自然语言，“我还剩几天年假”和“假期余额还有多少”都能识别
- **输出格式标准化**：模型输出 JSON 格式的 `tool_calls`，易于解析，不会出现“模型在回答里夹了一段代码”的混乱情况
- **多轮对话机制成熟**：定义工具 → 模型输出调用意图 → 执行函数 → 返回结果 → 生成答案，流程清晰

这些能力本身没问题，Function Call 解决了让模型调工具的核心问题。但当工具规模化之后，Function Call 协议本身没有覆盖的那些“管理层面”的问题就暴露出来了。

### 2. 工具规模化之后的四大痛点

#### 2.1 工具定义的维护噩梦

上一篇的代码你应该还有印象，定义一个工具要写多少 JSON Schema：

```java
JsonObject tool1 = new JsonObject();
tool1.addProperty("type", "function");
JsonObject function1 = new JsonObject();
function1.addProperty("name", "getUserAnnualLeave");
function1.addProperty("description", "查询用户的年假余额，包括总天数、已使用天数、剩余天数");
JsonObject parameters1 = new JsonObject();
parameters1.addProperty("type", "object");
JsonObject properties1 = new JsonObject();
JsonObject userId1 = new JsonObject();
userId1.addProperty("type", "string");
userId1.addProperty("description", "用户 ID");
properties1.add("userId", userId1);
parameters1.add("properties", properties1);
JsonArray required1 = new JsonArray();
required1.add("userId");
parameters1.add("required", required1);
function1.add("parameters", parameters1);
tool1.add("function", function1);
```

这只是一个工具、一个参数。如果一个工具有 5 个参数，代码量翻 5 倍。20 个工具就是几百行纯粹的 JSON Schema 构建代码。

更要命的是维护问题：

- 工具的参数改了（比如 `getUserAnnualLeave` 新增了一个 `year` 参数），你要同步修改 JSON Schema，忘了改就会出 bug
- 没有工具文档，新同事不知道系统里有哪些工具可用，只能翻代码
- 工具定义散落在代码各处，没有统一的注册中心

#### 2.2 跨语言跨系统的集成困境

你的企业知识库助手需要调用的工具来自不同团队：

- Java 团队写的 HR 工具（查年假、查考勤）
- Python 团队写的数据分析工具（销售报表、用户画像）
- 第三方 HTTP API（物流查询、天气查询）

三种不同的调用方式，你的代码里要写三套集成逻辑：

```java
if ("getUserAnnualLeave".equals(functionName)) {
    // 直接调用本地 Java 方法
    return hrService.getAnnualLeave(userId);
} else if ("analyzeSalesData".equals(functionName)) {
    // 调用 Python 服务的 HTTP API
    return httpClient.post("http://python-service:8000/analyze", params);
} else if ("getWeather".equals(functionName)) {
    // 调用第三方 API
    return httpClient.get("https://api.weather.com/v1/current?city=" + city);
}
```

每接入一个新系统，就要写一套适配代码。而且每个系统的认证方式不同（有的用 Token，有的用 API Key，有的用 OAuth），错误处理方式也不同。

#### 2.3 权限和安全的黑洞

Function Call 协议本身没有任何权限机制。所有的权限校验都要你自己写：

- 用户 A 能不能查用户 B 的年假？
- 实习生能不能调用删除订单这个工具？
- 某个工具只允许管理员使用，怎么控制？

工具越多，权限逻辑越复杂。而且权限代码和业务代码混在一起，容易出漏洞。

#### 2.4 可观测性的缺失

20 个工具在线上跑，你需要知道：

- 每个工具被调用了多少次？
- 平均耗时多少？哪个工具最慢？
- 调用失败率是多少？失败的原因是什么？
- 某次调用的完整链路：用户问了什么 → 模型选了哪个工具 → 传了什么参数 → 返回了什么结果 → 最终答案是什么？

Function Call 协议不管这些，全靠你自己埋点、写日志、搭监控。工具调用链路一长（调用工具 A → 工具 A 内部调用工具 B → 工具 B 查数据库），排查问题就像大海捞针。

### 3. 我们需要的是一个标准化的工具管理协议

总结一下，Function Call 解决了让模型调工具的问题，但没有解决怎么管理工具的问题：

| 维度 | Function Call 解决了吗 |
|------|----------------------|
| 模型判断是否调用工具 | 解决了 |
| 标准化的调用意图输出 | 解决了 |
| 工具定义的自动化管理 | 没有，手写 JSON Schema |
| 工具的动态发现和注册 | 没有，硬编码在代码里 |
| 跨语言跨系统的统一调用 | 没有，每个系统写一套适配 |
| 权限和安全控制 | 没有，自己实现 |
| 调用链路的可观测性 | 没有，自己埋点 |

我们需要的是在 Function Call 之上，加一层标准化的工具管理框架——统一工具的定义、注册、发现、调用、权限控制。

这就是 MCP 要做的事。

## MCP 是什么

### 1. MCP 的核心思想：大模型世界的 USB 接口

回想一下 USB 出现之前的世界：打印机用并口，鼠标用 PS/2 接口，手机充电有 Micro USB、Lightning、Type-C 好几种。每换一个设备，就要换一根线。USB 出来之后，统一了接口标准——不管你是键盘、鼠标、U 盘还是手机，只要有 USB 接口，插上就能用。

MCP 做的是同样的事，只不过它统一的不是硬件接口，而是大模型调用工具的接口。

MCP，全称 Model Context Protocol（模型上下文协议），由 Anthropic（Claude 的母公司）于 2024 年 11 月开源发布。它不是一个具体的产品或框架，而是一个开放的协议规范。核心思想用一句话概括：

> 任何工具只要实现了 MCP 协议，就能被任何支持 MCP 的客户端调用，不用关心对方是什么语言、什么平台。

打个比方：

- 没有 MCP 之前：你的 Java 工具只能被你的 Java 代码调用，Python 团队的工具只能被 Python 代码调用，每个系统都是一座孤岛
- 有了 MCP 之后：Java 工具、Python 工具、Node.js 工具都实现 MCP 协议，Claude Desktop、Cursor、你自己的企业助手都支持 MCP 协议，任意组合，即插即用

这就像 USB 接口一样——工具是“设备”，客户端是“电脑”，MCP 是“USB 标准”。

### 2. MCP 的三层架构：Host、Client、Server

MCP 的架构设计分三层：Host（宿主应用）、Client（MCP 客户端）、Server（MCP 服务端）。这三层的关系用一张图说明：

![](https://oss.open8gu.com/iShot_2026-02-08_21.46.36.png)

这张图最近在网上被广泛引用。不过坦白说，我刚开始学习时，看着它并没有完全理解，尤其是 MCP Server 和 Hosts 之间的关系，总觉得差点意思。

于是我重新画了一张图，结合我们自己的文章脉络重新拆解了一遍，希望能让结构更清晰，也方便大家建立完整认知。

![](https://oss.open8gu.com/image-20260227171359056.png)

#### 2.1 Host（宿主应用）

Host 是用户直接交互的应用，比如：

- Claude Desktop（Anthropic 的桌面客户端）
- Cursor（AI 编程 IDE）
- 你自己开发的企业知识库助手（比如咱们的 Ragent）

Host 的职责是：接收用户输入 → 调用大模型 → 根据模型的指令通过 MCP Client 调用工具 → 把结果返回给模型 → 展示最终答案给用户。

Host 内部包含一个或多个 MCP Client，每个 Client 负责和一个 MCP Server 通信。

#### 2.2 Client（MCP 客户端）

Client 是 Host 内部的通信组件，负责和 MCP Server 建立连接、发送请求、接收响应。

关键点：**一个 Client 只连接一个 Server**，但一个 Host 可以有多个 Client，连接多个 Server。就像你的电脑有多个 USB 接口，每个接口插一个设备。

Client 的职责包括：

- 和 Server 建立连接（通过 Stdio 或 HTTP）
- 发现 Server 提供的工具列表
- 调用 Server 的工具并获取结果
- 管理连接的生命周期

#### 2.3 Server（MCP 服务端）

Server 是提供工具的一方。每个 Server 暴露一组工具，Client 通过 MCP 协议调用这些工具。

Server 可以是：

- 本地进程（通过 Stdio 通信，比如一个本地的文件操作工具）
- 远程 HTTP 服务（通过 Streamable HTTP 通信，比如部署在服务器上的 HR 系统工具）

Server 的职责包括：

- 声明自己提供哪些工具（工具名、描述、参数定义）
- 接收 Client 的调用请求
- 执行工具逻辑并返回结果

#### 2.4 一个完整的调用流程

用户在 RAG Desktop 里问：查下我的年假，顺便看周五有没有会。完整的调用流程是这样的：

![](https://oss.open8gu.com/image-20260227173627177.png)

注意初始化阶段：Client 启动时会自动从 Server 获取工具列表（包括工具名、描述、参数定义）。这意味着你不需要在 Host 端手写 JSON Schema——Server 端定义好工具，Client 自动发现。这就解决了 Function Call 的第一个痛点：工具定义的维护。

### 3. MCP 的三大核心能力

MCP 协议定义了三大核心能力：Tools（工具调用）、Resources（资源访问）、Prompts（提示词模板）。

#### 3.1 Tools（工具调用）

这是 MCP 最核心的能力，也是和 Function Call 直接对应的部分。

Server 定义工具，Client 发现并调用工具。和 Function Call 的区别在于：

- **Function Call**：工具定义在客户端代码里（手写 JSON Schema），和模型一起发送

- **MCP**：工具定义在 Server 端，Client 通过协议动态获取

> 注意：MCP 协议本身只规定工具定义在 Server 端，Client 通过协议动态获取工具列表和元数据。至于 Server 端用什么方式定义工具（注解、手写 JSON、配置文件），那是实现框架的选择。下面展示的 `@Tool` 注解是 Spring AI 框架提供的便利性封装。

举个例子，Function Call 里你要这样定义工具：

```json
{
  "type": "function",
  "function": {
    "name": "getUserAnnualLeave",
    "description": "查询用户的年假余额",
    "parameters": {
      "type": "object",
      "properties": {
        "userId": { "type": "string", "description": "用户 ID" }
      },
      "required": ["userId"]
    }
  }
}
```

在 Spring AI 框架中，你可以用 `@Tool` 注解定义工具（框架会自动生成符合 MCP 协议的工具元数据）：

```java
@Tool(description = "查询用户的年假余额，包括总天数、已使用天数、剩余天数")
public String getUserAnnualLeave(@ToolParam(description = "用户 ID") String userId) {
    // 查询 HR 系统
    return "{\"remainingDays\": 5, \"totalDays\": 10, \"usedDays\": 5}";
}
```

Spring AI 框架会扫描 `@Tool` 注解，自动提取方法名作为工具名、`description` 作为工具描述、方法参数和 `@ToolParam` 注解作为参数定义，生成符合 MCP 协议的工具元数据（JSON Schema 格式）。Client 启动时通过 MCP 协议从 Server 获取这些元数据，不用在 Client 端手写 JSON Schema，也不用担心定义和实现不同步。

#### 3.2 Resources（资源访问）

Server 可以暴露资源供 Client 读取，比如：

- 文件内容（配置文件、日志文件）
- 数据库记录
- API 返回的数据

Resources 和 Tools 的区别是：Tools 是执行操作（查年假、下订单），Resources 是提供数据（读取一个文件的内容）。Resources 更像是给模型提供额外的上下文信息。

#### 3.3 Prompts（提示词模板）

Server 可以提供预定义的提示词模板，Client 可以使用这些模板来构建和模型的交互。适合标准化的交互场景，比如“代码审查模板"”、“文档总结模板”。

> 本篇重点讲 Tools，Resources 和 Prompts 在实际项目中用得相对少一些，了解即可。

### 4. MCP vs Function Call：不是替代，是增强

很多人第一次听到 MCP 会问：MCP 是不是要替代 Function Call？

答案是：不是。MCP 底层仍然依赖模型的 Function Call 能力。模型判断是否调用工具、输出 `tool_calls` JSON——这个能力是 Function Call 提供的，MCP 不会重新发明这个轮子。

MCP 做的是在 Function Call 之上，提供一层标准化的管理框架。用一个类比：Function Call 是"发动机"，MCP 是"整车"。发动机提供动力，但你还需要方向盘、刹车、仪表盘才能上路。

下面用一张对比图说明两者的关系：

![](https://oss.open8gu.com/image-20260227174510999.png)

用表格对比更直观：

| 对比维度 | Function Call | MCP |
|---------|--------------|-----|
| 本质 | 模型的原生能力（输出调用意图） | 标准化的工具管理协议 |
| 工具定义 | 手写 JSON Schema，和代码分离 | 代码注解自动生成，定义和实现一体 |
| 工具发现 | 没有，硬编码在请求里 | Client 启动时自动从 Server 获取 |
| 跨语言支持 | 不支持，每个语言自己实现 | 协议统一，Java / Python / TS 都能互通 |
| 传输方式 | 依赖具体的 HTTP API | 标准化传输（Stdio / Streamable HTTP） |
| 权限控制 | 没有，自己实现 | 协议层面支持（还在完善中） |
| 可观测性 | 没有，自己埋点 | 协议层面支持日志和追踪 |
| 生态 | 各家模型厂商各自实现 | 开放协议，社区共建 MCP Server |

### 5. 澄清：MCP 协议 vs 实现框架

看到这里，你可能会有个疑问：前面展示的 `@Tool` 注解、自动注册工具，这些不都是 Spring AI 框架提供的能力吗？MCP 协议本身到底解决了什么问题？

这个问题问得很好。我们需要区分两个层面：

**MCP 协议层面**（标准制定者）：
- 定义了工具元数据的标准格式（name、description、parameters 的 JSON Schema）
- 定义了 Client 和 Server 的通信协议（JSON-RPC over Stdio/HTTP）
- 定义了工具发现机制（Client 启动时从 Server 获取工具列表）
- 解决的核心问题：**工具定义从 Client 端转移到 Server 端，实现跨语言跨系统互通**

**Spring AI 框架层面**（协议实现者）：

- 提供 `@Tool` 注解，让你用注解方式定义工具（而不是手写 JSON）
- 提供自动扫描注解并生成符合 MCP 协议的工具元数据
- 处理 MCP 协议的通信细节（JSON-RPC 请求解析、响应构建等）
- 解决的核心问题：**让 Java 开发者更方便地实现 MCP 协议**

打个比方：MCP 协议就像 HTTP 协议，Spring AI 就像 Spring MVC 框架。HTTP 定义了请求响应格式，但你可以用任何语言、任何框架实现 HTTP 服务器。Spring MVC 提供了 `@RestController` 注解让你更方便地写 HTTP 接口，但这不是 HTTP 协议本身的能力。

即使不用 Spring AI，你也可以用纯 Java 手写 JSON 来实现 MCP Server，只是更麻烦。MCP 协议的价值在于：**一旦你的 Server 实现了 MCP 协议，任何支持 MCP 的 Client（Claude Desktop、Cursor、Python 客户端等）都能调用你的工具，不需要为每个 Client 写适配代码**。

这就是为什么说 MCP 解决了 Function Call 的工具定义和维护问题——不是因为注解写起来更方便，而是因为工具定义的位置变了（从 Client 端移到 Server 端），发现机制变了（从硬编码变成动态获取），互通性变了（从各自实现变成统一协议）。

## MCP 的传输机制

MCP Client 和 Server 之间怎么通信？MCP 协议定义了两种传输方式：Stdio（标准输入输出）和 Streamable HTTP（可流式 HTTP）。

### 1. Stdio（标准输入输出）

Stdio 是最简单的传输方式。MCP Server 作为本地子进程运行，Client 通过操作系统的标准输入（stdin）和标准输出（stdout）和 Server 通信。

打个比方：你在终端里运行 `grep "error" log.txt`，你敲的命令是输入（stdin），grep 打印出来的结果是输出（stdout）。MCP 的 Stdio 传输就是这个原理：Client 启动 Server 进程，往 Server 的 stdin 写请求（JSON 格式），Server 从 stdout 返回结果（也是 JSON 格式）。整个过程就像你在和一个命令行程序对话，只不过对话内容是结构化的 JSON 数据。

通信过程：

1. Client 启动 Server 进程（比如 `java -jar mcp-server.jar`）
2. Client 往 Server 的 stdin 写入 JSON-RPC 请求
3. Server 从 stdin 读取请求，执行工具，把结果以 JSON-RPC 格式写入 stdout
4. Client 从 Server 的 stdout 读取响应

> JSON-RPC 是一种轻量级的远程过程调用协议，用 JSON 格式传输请求和响应。你可以把它理解为用 JSON 格式约定好请求和响应的结构，和 RESTful API 类似，但更简单。

Stdio 的优点：

- 简单，不需要网络配置，不暴露端口
- 安全，通信只在本地进程之间，不经过网络
- 适合本地工具（文件操作、本地数据库查询、命令行工具）

Stdio 的缺点：

- 只能本地调用，不支持远程访问
- Server 和 Client 必须在同一台机器上

### 2. Streamable HTTP（可流式 HTTP）

Streamable HTTP 是 MCP 协议在 2025 年 3 月引入的传输方式（替代了早期的 HTTP+SSE 方案）。MCP Server 作为 HTTP 服务运行，Client 通过 HTTP 请求调用。

通信过程：

1. Server 启动 HTTP 服务，监听某个端口（比如 `http://localhost:8080`）
2. Client 向 Server 发送 HTTP POST 请求（JSON-RPC 格式）
3. Server 处理请求，返回 HTTP 响应
4. 如果需要流式返回（比如长时间运行的工具），Server 通过 SSE（Server-Sent Events）推送增量结果

Streamable HTTP 的优点：

- 支持远程访问，Server 可以部署在任何地方
- 支持团队共享，多个 Client 可以连接同一个 Server
- 支持流式响应（SSE），适合长时间运行的工具
- 可以利用 HTTP 生态的认证、负载均衡、监控等基础设施

Streamable HTTP 的缺点：

- 需要网络配置（端口、防火墙、HTTPS）
- 相比 Stdio 多了网络开销

### 3. 怎么选

![](https://oss.open8gu.com/image-20260227180906767.png)

用表格总结：

| 对比维度 | Stdio | Streamable HTTP |
|---------|-------|-----------------|
| 部署方式 | 本地子进程 | HTTP 服务（本地或远程） |
| 通信方式 | stdin / stdout | HTTP POST + SSE |
| 网络要求 | 不需要 | 需要网络（至少 localhost） |
| 多客户端 | 不支持（一对一） | 支持（多对一） |
| 适用场景 | 本地开发、个人工具 | 团队共享、生产环境 |
| 安全性 | 天然安全（本地进程） | 需要配置认证和 HTTPS |
| 典型用法 | Claude Desktop 本地插件 | 企业内部工具平台 |

一句话总结：本地开发和个人工具用 Stdio，团队共享和生产环境用 Streamable HTTP。企业级项目基本上都是后者。

## Java 实战：搭建一个 MCP Server

### 1. 场景设定

延续前几篇的企业知识库助手场景，我们用 Spring AI MCP Server 框架搭建一个 MCP Server，提供两个工具：

- `getUserAnnualLeave`：查询用户年假余额
- `getOrderStatus`：查询订单状态

搭建完成后，这个 MCP Server 可以被 Claude Desktop、Cursor 等任何支持 MCP 的客户端直接调用。

### 2. 技术选型

Spring AI 从 1.0 版本开始提供了 MCP Server 的 Boot Starter，开箱即用。选 Spring AI 的理由很简单：

- Java 生态，和现有的 Spring Boot 项目无缝集成
- 用 `@Tool` 注解定义工具，不用手写 JSON Schema
- 同时支持 Stdio 和 Streamable HTTP 两种传输方式
- 社区活跃，文档完善

Spring AI MCP Server 提供了三个 Starter，根据传输方式选择：

| Starter | 传输方式 | 适用场景 |
|---------|---------|---------|
| `spring-ai-starter-mcp-server` | Stdio | 本地工具，被 Claude Desktop / Cursor 调用 |
| `spring-ai-starter-mcp-server-webmvc` | Streamable HTTP（基于 Spring MVC） | 远程工具，团队共享 |
| `spring-ai-starter-mcp-server-webflux` | Streamable HTTP（基于 WebFlux） | 远程工具，响应式编程 |

本篇先用 Stdio 方式演示（最简单，能直接被 Claude Desktop 调用），后面再介绍 HTTP 方式。

> 项目已经提交到 GitHub [mcp-server-demo](https://github.com/nageoffer/mcp-server-demo)，代码在 v1.0 分支，可直接下载。
>
> 注意：SpringAI 1.0 版本注解是 `@Tool`，2.0 版本修改为 `@McpTool`，后续文章以 `@McpTool` 为准。

### 3. Maven 依赖

创建一个 Spring Boot 项目，`pom.xml` 如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.3</version>
        <relativePath/>
    </parent>

    <groupId>com.nageoffer.ai</groupId>
    <artifactId>mcp-server-demo</artifactId>
    <version>1.0.0</version>
    <name>MCP Server Demo</name>
    <description>企业知识库助手 MCP Server</description>

    <properties>
        <java.version>17</java.version>
        <spring-ai.version>1.0.0</spring-ai.version>
    </properties>

    <dependencies>
        <!-- Spring AI MCP Server Starter（Stdio 传输） -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-starter-mcp-server</artifactId>
            <version>${spring-ai.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

> 注意：`spring-ai-starter-mcp-server` 是 Stdio 传输方式的 Starter。如果你需要 HTTP 传输，换成 `spring-ai-starter-mcp-server-webmvc`。

### 4. 定义工具

这是 MCP 相比 Function Call 最爽的地方——用 `@Tool` 注解定义工具，参数用 `@ToolParam` 注解描述，框架自动生成工具的 JSON Schema。

创建一个工具类 `EnterpriseTools.java`：

```java
package com.nageoffer.ai.mcp.tools;

import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Service;

/**
 * 企业知识库助手工具集
 * 每个 @Tool 方法会自动注册为一个 MCP 工具
 */
@Service
public class EnterpriseTools {

    /**
     * 查询用户年假余额
     */
    @Tool(description = "查询用户的年假余额，包括总天数、已使用天数、剩余天数。当用户询问年假、假期余额、还有多少天假等问题时使用此工具。")
    public String getUserAnnualLeave(
            @ToolParam(description = "用户 ID，例如 user_12345") String userId) {

        // 实际项目中这里调用 HR 系统 API
        // 这里用 mock 数据演示
        return String.format("""
                {
                    "userId": "%s",
                    "remainingDays": 5,
                    "totalDays": 10,
                    "usedDays": 5,
                    "year": 2026
                }
                """, userId);
    }

    /**
     * 查询订单状态
     */
    @Tool(description = "查询订单的物流状态和详细信息。当用户询问订单状态、物流进度、快递到哪了等问题时使用此工具。")
    public String getOrderStatus(
            @ToolParam(description = "订单号，例如 ORD-12345") String orderId) {

        // 实际项目中这里调用订单系统 API
        return String.format("""
                {
                    "orderId": "%s",
                    "status": "运输中",
                    "location": "北京市朝阳区分拨中心",
                    "estimatedDelivery": "2026-03-01",
                    "carrier": "顺丰速运",
                    "trackingNumber": "SF1234567890"
                }
                """, orderId);
    }
}
```

对比一下 Function Call 里定义同样两个工具需要多少代码——上一篇的 `callModelWithTools` 方法里，光构建 JSON Schema 就写了 40 多行。MCP 里只需要两个方法加注解，清爽多了。

### 5. 注册工具到 MCP Server

创建配置类，把工具注册到 MCP Server：

```java
package com.nageoffer.ai.mcp.config;

import com.nageoffer.ai.mcp.tools.EnterpriseTools;
import org.springframework.ai.tool.ToolCallbackProvider;
import org.springframework.ai.tool.method.MethodToolCallbackProvider;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class McpServerConfig {

    /**
     * 注册工具提供者
     * MethodToolCallbackProvider 会扫描 EnterpriseTools 中所有 @Tool 注解的方法，
     * 自动注册为 MCP 工具
     */
    @Bean
    public ToolCallbackProvider enterpriseToolProvider(EnterpriseTools enterpriseTools) {
        return MethodToolCallbackProvider.builder()
                .toolObjects(enterpriseTools)
                .build();
    }
}
```

`MethodToolCallbackProvider` 会自动扫描 `EnterpriseTools` 类中所有带 `@Tool` 注解的方法，提取方法名作为工具名、`description` 作为工具描述、方法参数和 `@ToolParam` 注解作为参数定义，自动生成完整的工具元数据。

### 6. 配置 MCP Server

`application.yml` 配置：

```yaml
spring:
  ai:
    mcp:
      server:
        name: enterprise-assistant
        version: 1.0.0
        type: SYNC
  main:
    web-application-type: none
    banner-mode: off

logging:
  file:
    name: ./logs/mcp-server.log
  level:
    root: OFF
```

几个关键配置说明：

- `spring.ai.mcp.server.name`：Server 名称，Client 连接时会看到这个名字
- `spring.ai.mcp.server.version`：Server 版本号
- `spring.ai.mcp.server.type`：工具执行模式，`SYNC`（同步）或 `ASYNC`（异步）
- `spring.main.web-application-type: none`：Stdio 模式不需要 Web 容器
- `spring.main.banner-mode: off`：关闭 Spring Boot 启动 banner，避免 banner 输出到 stdout 干扰 MCP 通信

> Stdio 模式下，stdout 是 MCP 协议的通信通道，任何非协议内容（banner、日志）输出到 stdout 都会导致通信失败。这是一个常见的坑。

### 7. 启动类

```java
package com.nageoffer.ai.mcp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class McpServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(McpServerApplication.class, args);
    }
}
```

项目结构：

```
mcp-server-demo/
├── pom.xml
└── src/main/
    ├── java/com/nageoffer/ai/mcp/
    │   ├── McpServerApplication.java
    │   ├── config/
    │   │   └── McpServerConfig.java
    │   └── tools/
    │       └── EnterpriseTools.java
    └── resources/
        └── application.yml
```

先用 Maven 打包：

```bash
mvn clean package -DskipTests
```

打包完成后，在 `target` 目录下会生成 `mcp-server-demo-1.0.0.jar`。

### 8. 测试：用 MCP 客户端调用

#### 8.1 在 Cursor 中配置

打开 Cursor 的 MCP 配置（Settings → Tools & MCP），如下图所示：

![](https://oss.open8gu.com/image-20260227200317384.png)

添加：

```json
{
  "mcpServers": {
    "enterprise-assistant": {
      "command": "java",
      "args": [
        "-jar",
        "/path/to/mcp-server-demo-1.0.0.jar"
      ]
    }
  }
}
```

把 `/path/to/` 替换成你的 jar 包实际路径。

咱们 MCP Demo 项目用的 JDK17，如果你的电脑默认不是 JDK17，需要将 command 设置为 Java 的绝对路径。我的默认 JDK 就不是 17，所以需要调整下，比如：

```json
{
  "mcpServers": {
    "enterprise-assistant": {
      "command": "/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home/bin/java",
      "args": [
        "-jar",
        "/path/to/mcp-server-demo-1.0.0.jar"
      ],
      "env": {
        "JAVA_HOME": "/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home"
      }
    }
  }
}
```

在对话框里进行问答，Cursor 会自动调用你的 MCP Server 查询年假余额和订单状态。

![](https://oss.open8gu.com/image-20260227200604554.png)

> 因为是个 Demo 的 MCP 示例，所以问题有点呆，需要用户提示词自己把账号和订单号带进去。常规上肯定是程序里自动带入，大家忽略即可。

#### 8.2 使用 Streamable HTTP 方式

如果你想让 MCP Server 以 HTTP 服务的方式运行（支持远程访问、多客户端共用），只需要做两个改动：

第一步，把 Maven 依赖换成 WebMVC 版本：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
    <version>${spring-ai.version}</version>
</dependency>
```

第二步，修改 `application.yml`：

```yaml
spring:
  ai:
    mcp:
      server:
        name: enterprise-assistant
        version: 1.0.0
        type: SYNC

server:
  port: 8080
```

启动后，MCP Server 会在 `http://localhost:8080` 上监听。Client 配置改为：

```json
{
  "mcpServers": {
    "enterprise-assistant": {
      "url": "http://localhost:8080/mcp"
    }
  }
}
```

工具代码和配置类完全不用改，只是传输方式从 Stdio 变成了 HTTP。

### 9. 对比：Function Call vs MCP + Spring AI 的代码量

同样实现查年假和查订单两个工具，对比一下两种方式的代码量：

| 对比项     | Function Call（上一篇）                         | MCP + Spring AI（本篇）        |
| ---------- | ----------------------------------------------- | ------------------------------ |
| 工具定义   | 40+ 行 JSON Schema 构建代码                     | 2 个 `@Tool` 注解方法          |
| 工具路由   | 手写 if-else 或策略模式                         | 框架自动路由                   |
| 协议处理   | 手写 HTTP 请求、解析 tool_calls、构建第二轮消息 | 框架自动处理                   |
| 新增工具   | 加 JSON Schema + 加路由 + 改代码 + 重新部署     | 加一个 `@Tool` 方法 + 重新部署 |
| 客户端适配 | 只能被你自己的代码调用                          | 任何 MCP 客户端都能调用        |
| 总代码量   | ~200 行（含 JSON Schema 构建）                  | ~60 行（含工具实现）           |

代码量减少了 70%，而且新增工具只需要加一个方法，不用改任何框架代码。

> 说明：这里对比的是纯 Function Call 实现 vs MCP 协议 + Spring AI 框架实现。代码量的减少主要来自 Spring AI 框架的便利性封装（注解、自动路由等）。MCP 协议本身带来的核心价值是最后一行客户端适配——一旦你的工具实现了 MCP 协议，任何支持 MCP 的客户端都能调用，不需要为每个客户端写适配代码。

## MCP 在 RAG 系统中的应用

讲完了 MCP 的原理和实战，回到我们的主线——RAG 系统。MCP 在 RAG 系统中能发挥什么作用？

### 1. 知识检索作为 MCP 工具

上一篇讲 Function Call 在 RAG 中的应用时，我们定义了一个 `searchKnowledgeBase` 工具，让模型自动判断是查知识库还是调业务工具。用 MCP 实现同样的能力，只需要把知识检索封装成一个 MCP 工具：

```java
@Tool(description = "在企业知识库中搜索相关文档。当用户询问公司制度、产品文档、操作指南等静态知识时使用此工具。")
public String searchKnowledgeBase(
        @ToolParam(description = "搜索关键词，用自然语言描述") String query,
        @ToolParam(description = "返回结果数量，默认 5") int topK) {

    // 调用向量数据库检索
    // 实际项目中这里执行 Embedding → Milvus 检索 → Reranking
    List<Document> results = ragService.search(query, topK);
    return formatResults(results);
}
```

这样做的好处是：你的知识检索能力不再局限于自己的系统，任何支持 MCP 的客户端都能调用你的知识库。比如团队成员在 Cursor 里写代码时，可以直接问公司的代码规范是什么，Cursor 通过 MCP 调用你的知识库检索工具，返回相关文档。

### 2. 多工具编排：知识检索 + 业务工具

MCP 的真正威力在于多个 Server 协同工作。假设用户问我的订单 #12345 能退货吗，这个问题需要两部分信息：

1. 订单的当前状态（需要调用订单系统）
2. 退货政策（需要检索知识库）

在 MCP 架构下，这两个能力可以由不同的 MCP Server 提供：

![](https://oss.open8gu.com/image-20260227210315852.png)

每个 MCP Server 专注做一件事（订单查询、知识检索、HR 查询……），Host 根据模型的判断，并行调用多个 Server，综合结果生成答案。这种架构的好处是：

- **职责清晰**：每个 Server 独立开发、独立部署、独立维护
- **灵活组合**：新增一个能力只需要部署一个新的 MCP Server，不用改现有代码
- **团队协作**：不同团队各自维护自己的 MCP Server，通过协议互通

### 3. 企业级场景：Ragent 中的 MCP 实践

在我们的 Ragent 项目（企业级 RAG 智能体平台）中，MCP 被用来管理所有的外部工具调用。简要介绍一下架构思路：

- **MCPToolRegistry**：工具注册表，管理所有已注册的 MCP Server 和工具
- **MCPToolExecutor**：工具执行器，负责根据模型的 `tool_calls` 路由到对应的 MCP Server 并执行
- **自动发现机制**：系统启动时自动连接配置的 MCP Server，获取工具列表，注册到工具注册表

这种架构让 Ragent 可以灵活接入各种外部工具，而不需要为每个工具写适配代码。具体的代码实现会在后续的 Ragent 项目实战文章中详细展开。

## MCP 的生态现状

### 1. 支持 MCP 的客户端

MCP 协议发布一年多以来，已经有不少客户端支持：

| 客户端 | 类型 | MCP 支持情况 |
|-------|------|-------------|
| Claude Desktop | AI 对话客户端 | 完整支持，Anthropic 官方出品 |
| Cursor | AI 编程 IDE | 完整支持，MCP 工具可在编程中使用 |
| Windsurf | AI 编程 IDE | 支持 MCP |
| Continue | VS Code AI 插件 | 支持 MCP |
| Cline | VS Code AI 插件 | 支持 MCP |
| Claude Code | CLI 工具 | 完整支持 |
| Spring AI | Java AI 框架 | 提供 MCP Client 和 Server Starter |
| Kiro | AI IDE | 支持 MCP |

这意味着你开发一个 MCP Server，上面这些客户端都能直接调用，不需要为每个客户端写适配代码。

### 2. 社区 MCP Server

GitHub 上已经有大量社区开发的 MCP Server，覆盖常见场景：

- **文件系统**：读写本地文件、目录操作
- **数据库**：MySQL、PostgreSQL、SQLite 查询
- **Web 搜索**：Brave Search、Google Search
- **代码仓库**：GitHub 操作（创建 Issue、提交 PR、查看代码）
- **知识管理**：Notion、Obsidian 集成
- **通信工具**：Slack、Discord 消息发送
- **云服务**：AWS、GCP 资源管理

你可以在 [MCP Servers 目录](https://github.com/modelcontextprotocol/servers) 找到这些社区 Server，直接配置使用，不需要自己开发。

### 3. MCP 的局限性和未来

MCP 的方向是对的，但目前还有一些局限：

- **协议还在快速演进**：从 2024 年 11 月发布到现在，传输层已经从 HTTP+SSE 改为 Streamable HTTP，API 也在不断调整。早期开发的 MCP Server 可能需要适配新版本
- **部分客户端支持不完整**：有些客户端只支持 Stdio，不支持 HTTP；有些只支持 Tools，不支持 Resources 和 Prompts
- **生产环境的稳定性**：MCP 在本地开发场景下表现很好，但在高并发的生产环境中，稳定性和性能还需要更多验证
- **权限模型还在完善**：MCP 协议层面的权限控制还比较基础，企业级的细粒度权限（比如基于角色的工具访问控制）仍然需要自己实现
- **认证标准化**：远程 MCP Server 的认证方式（OAuth、API Key 等）还没有统一标准

不过，标准化是大趋势。就像 HTTP 协议从 0.9 到 1.0 到 1.1 到 2.0 到 3.0，MCP 也会不断完善。现在学习和使用 MCP，是一个好的时机。

## 小结

这篇从 Function Call 的四大痛点出发，引出了 MCP 协议——大模型世界的 USB 接口。

核心要点回顾：

1. **为什么需要 MCP**：Function Call 解决了让模型调工具的问题，但工具规模化后，定义维护、跨语言集成、权限控制、可观测性都成了问题。MCP 在 Function Call 之上提供了标准化的工具管理框架
2. **MCP 的三层架构**：Host（宿主应用）→ Client（通信组件）→ Server（工具提供方），一个 Host 可以连接多个 Server
3. **三大核心能力**：Tools（工具调用，最核心）、Resources（资源访问）、Prompts（提示词模板）
4. **两种传输方式**：Stdio（本地进程通信，简单安全）和 Streamable HTTP（远程 HTTP 通信，支持团队共享）
5. **Java 实战**：用 Spring AI MCP Server 框架，`@Tool` 注解定义工具，60 行代码搞定，比 Function Call 的 200 行减少 70%
6. **MCP 在 RAG 中的应用**：知识检索封装为 MCP 工具、多 Server 协同工作、企业级工具管理

到这里，RAG 系列的核心环节已经讲完了：数据分块 → 元数据管理 → 向量化 → 向量数据库 → 检索策略 → 生成策略 → Function Call → MCP 协议。从数据准备到检索生成，再到工具调用和协议标准化，一条完整的链路。
