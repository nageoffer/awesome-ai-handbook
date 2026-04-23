上一篇 MCP 文章重点讲了 Tools——用 `@McpTool` 注解定义工具，模型判断什么时候调、调哪个，你的代码负责执行。整个流程跑通之后，你可能觉得 MCP 的核心就是 Tools，Resources 和 Prompts 只是协议里的附属品。

当时那篇也确实写了一句：“本篇重点讲 Tools，Resources 和 Prompts 在实际项目中用得相对少一些，了解即可。”在很多简单项目里，Resources 和 Prompts 确实不会第一时间用到。但在多 Client 接入、需要可复用架构的场景下，它们的价值会变得很明显。

但实际项目做下来，你会发现，有些场景用 Tools 硬做反而别扭。

比如：你的 AI 应用需要在对话前加载当前系统的配置信息，或者读取某张数据库表的表结构。用 Tools 怎么做？写一个 `getAppConfig` 工具，模型调用它，返回配置内容。能跑通，但总觉得哪里不对——这个操作没有任何副作用，不会修改数据，不会触发流程，它只是读一份资料而已。用 Tools 来做，就像你去餐厅点了一道菜，结果服务员端上来的是一本菜单。

再比如：你们团队沉淀了一套效果很好的知识库问答 Prompt（角色定义、回答规则、引用要求、兜底策略）。这套 Prompt 不只是一段文本——它精确地规定了指令和问题怎么组装、消息的顺序和结构。你希望所有接入的 Client 都能直接用这套模板，用户在 Client 里选一下模板、传入参数，就能拿到一组结构化的 messages 数组。不用每个 Client 自己去理解这套 Prompt 的结构，也不用自己拼装消息。当然，Client 拿到这组 messages 后是原样发给模型还是再做加工，仍然由 Client 自己决定——Prompts 提供的是标准化的模板描述与参数化机制，不是自动发送机制。

这正是 Resources 和 Prompts 在协议层面被单独抽出来的原因。今天把这个坑补上。

## Resources：让 Server 暴露数据给模型

### 1. Resources 是什么

Resources（资源）是 MCP 协议提供的三类能力之一。一句话概括：**Server 暴露数据，Client 来读取，给模型提供上下文**。

和 Tools 的区别很关键，用餐厅来打比方：

- **Tools 像点菜**——你告诉厨房要做一道红烧肉，厨房开火炒菜，这个过程有副作用（消耗食材、产生一道菜）。对应到系统里就是：下订单（写数据库）、发邮件（触发外部操作）、提交退货申请（修改订单状态）
- **Resources 像看菜单**——你拿起菜单翻了翻，看看有什么菜、价格多少。这个过程没有任何副作用，菜单不会因为你看了一眼就少一页。对应到系统里就是：获取应用配置、查看数据库表结构、读取系统状态

协议层面的定义也很直接：

| 维度 | Tools | Resources |
|------|-------|-----------|
| 本质 | 执行操作 | 提供数据 |
| 副作用 | 可能有（可读可写） | 无（只读） |
| 谁来决定调用 | 模型决定（model-controlled） | 用户/应用决定（application-controlled） |
| 协议方法 | `tools/call` | `resources/read` |
| 类比 | 点菜 | 看菜单 |

注意谁来决定调用这一行。Tools 是模型驱动的——模型分析用户意图后自己决定要不要调工具。而 Resources 是应用驱动的——通常由 Host 应用或用户手动选择要读取哪些资源，作为上下文提供给模型。

> 打个比方：Tools 像是模型的手，它自己伸手去做事；Resources 像是你递给模型的参考资料，你决定给它看什么。

### 2. 实际场景理解 Resources

#### 2.1 客服系统的上下文预加载

光看定义可能还是觉得抽象，用一个企业客服场景来感受一下 Resources 的价值。

用户打开在线客服对话窗口，还没开口说话，你的系统就已经知道了这个用户是谁。这时候，程序员在 Host 应用的代码里预设好逻辑：用户进入对话时，主动从 MCP Server 读取一批 Resources，作为上下文提供给模型：

- `customer://users/user_12345/profile` → 这个用户的会员等级、注册时间、历史投诉记录
- `customer://users/user_12345/recent-orders` → 最近 3 笔订单信息
- `docs://return-policy` → 当前退货政策
- `docs://vip-privileges` → VIP 会员专属权益

Host 代码大致长这样：

```java
// 程序员在 Host 应用里写的编排逻辑
public void onUserEnterChat(String userId) {
    // 主动读取这个用户相关的 Resources
    String profile = mcpClient.readResource("customer://users/" + userId + "/profile");
    String orders = mcpClient.readResource("customer://users/" + userId + "/recent-orders");
    String policy = mcpClient.readResource("docs://return-policy");

    // 把这些内容拼到上下文里，发给模型
    String context = profile + "\n" + orders + "\n" + policy;
    callLLM(systemPrompt, context, userMessage);
}
```

用户开口说：“我买的东西有问题想退”，模型已经知道他是金卡会员、上周刚买了一台 iPhone、享受 15 天无理由退货——**一轮对话就能给出精准回答**。

> 注意：这里读取 Resources 后塞进上下文是 Host 应用的编排策略，不是 MCP Resources 协议自动完成的。MCP 协议定义了资源如何被列出（`resources/list`）、如何被读取（`resources/read`），但读到之后怎么用、要不要放进模型上下文，由 Host 应用自己决定。

如果这些信息全用 Tools 来做会怎样？模型需要先判断“我该查什么”，然后依次调用工具：

```text
用户：我买的东西有问题想退
模型：（分析后决定调用工具）→ tool_calls: getUserProfile, getRecentOrders, getReturnPolicy
Host：执行三个工具调用，返回结果
模型：根据查到的信息，您的 iPhone 可以在 15 天内退货...
```

能跑通，但有两个问题：一是**额外的推理和调用延迟**——模型要先推理出需要调哪些工具，每个工具调用都有网络开销；二是**漏调用的风险**——模型不一定每次都能意识到要查这三样东西，它可能只查了订单，忘了查会员等级，给出的答案就少了 VIP 专属权益这块。

Resources 的价值就在这里：**应用提前把该给的上下文全给了，模型不用猜，没有额外的推理链路和工具调用延迟，也不会漏掉关键信息。**

#### 2.2 Java 微服务项目里 Resources 可能有点鸡肋

你可能会想：那我不用 MCP Resources，直接在 Host 代码里查数据库拼上下文，效果不是一样吗？

如果你是 Java 程序员，项目本身就是微服务架构——用户服务、订单服务、内容服务各自独立部署，通过 Feign / Dubbo / gRPC 互相调用。那上面这个场景，你完全可以这么写：

```java
public void onUserEnterChat(String userId) {
    // 直接通过微服务远程调用获取数据
    UserProfile profile = userServiceClient.getProfile(userId);
    List<Order> orders = orderServiceClient.getRecentOrders(userId);
    String policy = contentServiceClient.getReturnPolicy();

    String context = buildContext(profile, orders, policy);
    callLLM(systemPrompt, context, userMessage);
}
```

效果一模一样，而且你对 Feign / Dubbo 这套东西已经很熟了，调用链路清晰，类型安全，还有现成的熔断、重试、监控。绕一圈走 MCP Resources 协议，反而多了一层抽象，没有明显收益。

**所以结论是：如果你的 AI 应用只有一个 Client（你自己的后端服务），而且已经有成熟的微服务体系，Resources 确实不是必选项，直接用现有的远程调用就好。**

MCP Resources 真正有优势的场景是**跨 Client 共享数据源**——同一个 MCP Server 暴露的资源，Claude Desktop 能读、Cursor 能读、你自己的 Web 应用也能读，大家通过统一的 `resources/read` 协议获取数据，数据获取逻辑写在 Server 端一次，不用每个 Client 各写一套。如果你的系统只有一个 Client，这个优势就不存在了。

### 3. 两种资源类型

MCP 协议定义了两种资源类型：直接资源（Direct Resources）和资源模板（Resource Templates）。

#### 3.1 直接资源（Direct Resources）

直接资源有一个固定的 URI，指向一个确定的数据。就像一个固定的文件路径，任何时候访问都是同一份数据（内容可能更新，但地址不变）。

```text
docs://product-manual          → 产品手册
config://app/settings          → 应用配置
file:///var/log/app.log        → 应用日志
```

直接资源适合那些数量有限、相对固定的数据——你的系统里有哪些资源是明确的，可以在 Server 启动时就注册好。

#### 3.2 资源模板（Resource Templates）

资源模板使用 URI 模板（遵循 [RFC 6570](https://datatracker.ietf.org/doc/html/rfc6570) 规范），URI 里有参数占位符。Client 填入具体参数后，才能访问到对应的资源。

```text
order://users/{userId}/orders/{orderId}   → 某用户的某个订单详情
file:///logs/{date}/error.log             → 某天的错误日志
db://tables/{tableName}/schema            → 某张表的表结构
```

资源模板适合那些数量不固定、需要动态生成的数据——比如你不可能为每个订单都注册一个直接资源，但你可以注册一个模板，让 Client 传入订单号来获取具体数据。

#### 3.3 怎么选

判断标准很简单：

| 场景 | 选择 | 理由 |
|------|------|------|
| 系统公告、服务条款、编码规范 | 直接资源 | 内容固定，数量有限 |
| 应用配置、系统状态 | 直接资源 | 地址固定，不依赖参数 |
| 用户订单详情、用户个人信息 | 资源模板 | 需要用户 ID / 订单号作为参数 |
| 日期维度的日志文件 | 资源模板 | 需要日期作为参数 |
| 数据库表结构 | 资源模板 | 需要表名作为参数 |

### 4. 协议层面怎么交互

Client 和 Server 之间关于 Resources 的交互，涉及四个协议方法：

| 协议方法 | 用途 | 返回内容 |
|---------|------|---------|
| `resources/list` | 列出所有可用的直接资源 | 资源描述数组（uri、name、mimeType 等） |
| `resources/templates/list` | 列出所有资源模板 | 资源模板定义数组（uriTemplate、name 等） |
| `resources/read` | 读取指定资源的内容 | 资源数据（文本或二进制），附带元信息 |
| `resources/subscribe` | 订阅资源变更通知 | 订阅确认（可选能力，非必须支持） |

前三个是核心，第四个是可选的高级能力（后面会单独说）。交互流程如下：

![](https://oss.open8gu.com/iShot_2026-03-06_10.13.32.png)

展开说一下前三个方法：

- **`resources/list`**：Client 问 Server：“你有哪些直接资源？”，Server 返回资源列表，每个资源包含 `uri`（资源地址）、`name`（资源名称）、`description`（描述）、`mimeType`（内容类型，比如 `text/plain`、`application/json`）
- **`resources/templates/list`**：Client 问：“你有哪些资源模板？”，Server 返回模板列表，每个模板包含 `uriTemplate`（URI 模板，如 `order://users/{userId}/orders/{orderId}`）
- **`resources/read`**：Client 传入具体的 URI，Server 返回资源内容

资源内容的返回格式是一个 `contents` 数组，支持两种内容类型：

```json
// 文本内容
{
  "uri": "docs://product-manual",
  "mimeType": "text/plain",
  "text": "退货政策：自收货之日起 7 天内可无理由退货..."
}

// 二进制内容（Base64 编码）
{
  "uri": "diagrams://architecture",
  "mimeType": "image/png",
  "blob": "iVBORw0KGgo..."
}
```

大多数场景用文本内容就够了。二进制内容主要用于图片、PDF 这类非文本数据。

> 你可能注意到了，`resources/read` 一次可以返回多个内容（`contents` 是数组）。比如读取一个目录时，Server 可以把目录下所有文件的内容一次性返回。

### 5. Java 实战：用 Spring AI 实现 Resources

场景还是那个企业知识库助手。我们在之前的 MCP Server 基础上，新增两个资源：

1. **直接资源**：退货政策文档（`docs://return-policy`）——返回公司的退货政策文本
2. **资源模板**：订单详情（`order://{orderId}`）——根据订单号返回订单信息

#### 5.1 定义资源

创建 `EnterpriseResources.java`：

```java
/**
 * 企业知识库助手资源集
 * 每个 @McpResource 方法会自动注册为一个 MCP 资源
 */
@Component
public class EnterpriseResources {

    /**
     * 直接资源：退货政策文档
     * URI 是固定的，不带参数
     */
    @McpResource(
            uri = "docs://return-policy",
            name = "退货政策",
            description = "公司的退货政策文档，包含退货条件、时限、流程等信息",
            mimeType = "text/plain"
    )
    public ReadResourceResult getReturnPolicy() {
        // 实际项目中可能从数据库或文件系统读取
        String content = """
                【退货政策】

                1. 退货时限：自收货之日起 7 天内可申请无理由退货。
                2. 退货条件：
                   - 商品未拆封、未使用、不影响二次销售
                   - 赠品需一并退回
                   - 定制商品、鲜活易腐商品不支持无理由退货
                3. 退货流程：
                   - 在“我的订单”中提交退货申请
                   - 等待客服审核（1~2 个工作日）
                   - 审核通过后按指定地址寄回商品
                   - 收到商品并验收后 3~5 个工作日内退款
                4. 退款方式：原路退回（支付宝/微信/银行卡）。
                5. 运费说明：因质量问题退货，运费由公司承担；无理由退货，运费由用户承担。
                """;

        return new ReadResourceResult(List.of(
                new TextResourceContents("docs://return-policy", "text/plain", content)
        ));
    }

    /**
     * 资源模板：订单详情
     * URI 带参数 {orderId}，Client 填入具体订单号后访问
     */
    @McpResource(
            uri = "order://{orderId}",
            name = "订单详情",
            description = "根据订单号查询订单的详细信息，包括商品、金额、状态等",
            mimeType = "application/json"
    )
    public ReadResourceResult getOrderDetail(String orderId) {
        // 实际项目中调用订单系统查询
        String content = String.format("""
                {
                    "orderId": "%s",
                    "productName": "iPhone 16 Pro 256GB 沙漠钛金属",
                    "price": 8999,
                    "quantity": 1,
                    "status": "已签收",
                    "orderTime": "2026-03-01 14:30:00",
                    "deliveryTime": "2026-03-05 10:15:00",
                    "address": "北京市朝阳区xxx小区"
                }
                """, orderId);

        return new ReadResourceResult(List.of(
                new TextResourceContents("order://" + orderId, "application/json", content)
        ));
    }
}
```

对比一下 Tools 的定义方式：

| 对比项 | Tools（`@McpTool`） | Resources（`@McpResource`） |
|--------|------------------|-----------------------------|
| 注解 | `@McpTool` | `@McpResource` |
| 必填属性 | `description` | `uri`、`name`、`description` |
| 返回类型 | 结构化内容（`CallToolResult`） | `ReadResourceResult` |
| 参数注解 | `@McpToolParam` | 方法参数自动映射 URI 模板变量 |
| 语义 | 执行一个操作 | 读取一份数据 |

几个要注意的点：

1. **`uri` 属性**：直接资源写固定 URI（如 `docs://return-policy`），资源模板写带占位符的 URI（如 `order://{orderId}`）。框架会根据 URI 中是否包含 `{...}` 来自动判断是直接资源还是资源模板
2. **返回类型是 `ReadResourceResult`**：里面包一个 `TextResourceContents`（文本内容）或 `BlobResourceContents`（二进制内容）。注意构造 `TextResourceContents` 时要传入实际的 URI（模板参数已填入的），不是模板 URI
3. **方法参数自动映射**：如果 URI 模板是 `order://{orderId}`，方法参数名写 `orderId` 就能自动接收到 Client 传入的值

#### 5.2 和 Tools 对比：同一个需求用两种方式实现

拿查订单详情来说，用 Tools 和 Resources 都能实现。区别在于：

**Tools 方式**（之前那篇的做法）：

```java
@McpTool(description = "查询订单的物流状态和详细信息")
public String getOrderStatus(@ToolParam(description = "订单号") String orderId) {
    return "{\"orderId\": \"" + orderId + "\", \"status\": \"运输中\"}";
}
```

模型在对话过程中自己决定要不要调这个工具。用户问“我的订单到哪了”，模型分析后输出 `tool_calls`，你的代码执行，结果返回给模型。

**Resources 方式**（本篇的做法）：

```java
@McpResource(uri = "order://{orderId}", name = "订单详情", description = "...")
public ReadResourceResult getOrderDetail(String orderId) {
    return new ReadResourceResult(List.of(
        new TextResourceContents("order://" + orderId, "application/json", content)
    ));
}
```

Host 应用或用户主动选择要读取哪个资源，把资源内容作为上下文提供给模型。模型不参与是否读取的决策。

**怎么选**：如果这个操作会产生副作用（下单、退款、发邮件），或者需要模型根据对话内容自主判断要不要执行——用 Tools。如果只是给模型提供参考资料，由应用或用户决定要不要看——用 Resources。

> 实际项目中，两者经常配合使用。比如：先通过 Resources 把应用配置加载到上下文，再通过 Tools 让模型在需要时查询具体订单状态。

### 6. 资源变更订阅

#### 6.1 资源变更订阅流程

MCP 协议支持资源变更订阅：Client 通过 `resources/subscribe` 订阅某个资源，资源内容发生变化时，Server 发送 `notifications/resources/updated` 通知；Client 收到通知后，再调用 `resources/read` 获取最新内容。

![](https://oss.open8gu.com/iShot_2026-03-06_10.13.33.png)

注意：通知的是“这个资源更新了”这个事件，不是直接把新内容推给 Client。Client 收到通知后需要自己重新读一次。

这个机制适合配置热更新、缓存失效通知、实时数据看板等场景。

#### 6.2 什么时候需要订阅，什么时候不需要

你可能会想：前面 Java 实战里的资源内容都是代码里写死的，怎么通知？

关键区分是：**URI 写死 ≠ 内容写死**。判断要不要支持订阅，看的是同一个 URI 在运行期是否会返回不同内容：

| 场景 | 是否需要订阅 | 理由 |
|------|-------------|------|
| 资源内容写在代码常量里 | 不需要 | 内容是编译期常量，进程不重启就不会变 |
| 配置项存在数据库 / Nacos / Apollo | 需要 | URI 固定（如 `config://app/settings`），但底层数据会被运维修改 |
| 实时销售看板 | 需要 | URI 固定（如 `dashboard://sales/today`），但聚合数据持续变化 |
| 用户资料缓存 | 看情况 | 用户改了个人信息后，`customer://users/{id}/profile` 的内容就变了 |

换句话说，订阅通知的触发点不在资源定义代码本身，而在资源背后的数据源变化事件——可能是文件变化（WatchService）、数据库配置变化（轮询 / CDC / 消息队列）、配置中心回调（Nacos Listener）、或者业务代码主动触发事件。

#### 6.3 是否支持订阅能力？

是否支持订阅，取决于 Server 和 Client 的 capabilities 声明以及各自实现情况。在当前生态里，订阅相关的 API 仍在迭代中，不少客户端对资源订阅的支持还不算完善。如果你的资源内容本身就是常量，不需要强行上订阅。

我写这篇文档的时候，是 2026.3.16 号，[MCP Java SDK 1.1.0](https://github.com/modelcontextprotocol/java-sdk/releases/tag/v1.1.0) 是 3.13 号发布，我看才支持了 Resources 支持订阅功能。至于像 SpringAI 或者 LangChain4j 集成的话，可能又得晚个一段时间。

![](https://oss.open8gu.com/image-20260316141851137.png)

## Prompts：把最佳实践封装成可复用模板

### 1. Prompts 是什么

在 Tools、Resources 之外，MCP 还定义了第三类能力：Prompts（提示词模板）。一句话概括：**Server 预定义 Prompt 模板，Client 传入参数后获取一组可直接用于模型调用的 messages**。

用公司内部的文档模板来类比：公司有标准的周报模板、请假单模板、项目复盘模板。你写周报时不用从零开始，打开模板填空就行——项目名称填这里、本周进展填那里、下周计划填那里。格式统一，内容完整，新员工也不会漏写关键信息。

Prompts 做的就是这件事，只不过模板不是给人用的，是给模型用的。Server 定义好一套经过验证的 Prompt 模板（角色定义、回答规则、引用要求、兜底策略都写好了），Client 只需要传入参数（比如检索到的 chunk 和用户问题），就能拿到一组完整的 messages 数组，可直接用于模型调用。

#### 1.1 和直接写 Prompt 有什么区别

你可能会问：我在 Client 端直接拼 Prompt 不也一样吗？为什么要通过 MCP Server 来获取？

区别在于**管理和复用**：

| 维度 | Client 端自己写 Prompt | Server 端 MCP Prompts |
|------|----------------------|----------------------|
| 版本统一 | 每个 Client 各写各的，版本不一致 | 统一维护，所有 Client 拿到同一版本 |
| 更新方式 | 改 Prompt 要改每个 Client 的代码 | 只改 Server 端，Client 下次获取就是新版 |
| 最佳实践沉淀 | 经验散落在各处 | 集中沉淀在 Server 端 |
| 参数校验 | 各 Client 自行校验 | Server 端统一定义参数和校验规则 |

如果你的系统只有一个 Client，直接写 Prompt 完全没问题。但如果有多个 Client（比如 Claude Desktop、Cursor、你自己的 Web 应用都接入了同一个 MCP Server），Prompts 的价值就体现出来了。

#### 1.2 Prompts 的控制模式

和 Tools、Resources 一样，Prompts 也有自己的控制模式：**Client 驱动（通常由用户选择）**。

三种能力的控制模式对比：

- **Tools**：模型驱动——模型自己决定什么时候调用什么工具
- **Resources**：应用驱动——Host 应用决定加载哪些资源作为上下文
- **Prompts**：Client 驱动——Client 决定何时获取哪个模板（通常由用户在 UI 里通过斜杠命令或菜单选择触发，但也可以是自动化流程调用）

比如在 Claude Desktop 里，用户输入 `/knowledge-qa` 就能触发知识库问答模板，输入 `/doc-summary` 就能触发文档摘要模板。

### 2. 协议层面怎么交互

Prompts 的交互很简单，就两个核心请求：

- **`prompts/list`**：Client 问 Server：“你有哪些 Prompt 模板？”，Server 返回模板列表
- **`prompts/get`**：Client 传入模板名称和参数，Server 返回填好参数的完整 messages 数组

`prompts/list` 返回的每个模板包含：

```json
{
  "name": "knowledge-qa",
  "description": "知识库问答模板，基于检索到的内容回答用户问题",
  "arguments": [
    {
      "name": "context",
      "description": "检索到的知识片段，多个片段用换行分隔",
      "required": true
    },
    {
      "name": "question",
      "description": "用户的原始问题",
      "required": true
    }
  ]
}
```

`prompts/get` 的请求和响应：

```json
// 请求
{
  "method": "prompts/get",
  "params": {
    "name": "knowledge-qa",
    "arguments": {
      "context": "[1] AirPods Pro 保修期为 1 年\n[2] AppleCare+ 可延长至 2 年",
      "question": "AirPods Pro 的保修期多久？"
    }
  }
}

// 响应
{
  "description": "知识库问答",
  "messages": [
    {
      "role": "user",
      "content": {
        "type": "text",
        "text": "你是一个企业知识库助手。请严格基于以下参考资料回答问题...\n\n参考资料：\n[1] AirPods Pro 保修期为 1 年\n[2] AppleCare+ 可延长至 2 年\n\n问题：AirPods Pro 的保修期多久？"
      }
    }
  ]
}
```

注意返回的是 **messages 数组**，不是纯文本字符串。这意味着 Server 可以精确控制消息的结构——把角色定义、回答规则、参考资料和问题组装成完整的对话。Client 拿到这个数组后，可以直接用于模型调用，也可以根据自己的需要做进一步处理（比如把指令部分放进 system 消息）。

> 注意：MCP 协议规范中，PromptMessage 的 role 只支持 `"user"` 和 `"assistant"` 两种角色，不支持 `"system"`。如果你需要设置 system 级别的指令，可以把指令内容放在第一条 user 消息里，或者由 Client 拿到 messages 后自行拆分到 system 消息中——这属于 Client 的编排策略，不是 MCP Prompts 协议负责的。

### 3. Java 实战：用 Spring AI 实现 Prompts

继续在那个企业知识库助手的 MCP Server 上加功能。这次新增两个 Prompt 模板：

1. **知识库问答模板**（`knowledge-qa`）：传入检索到的 chunk 和用户问题，返回包含完整回答规则的 Prompt
2. **文档摘要模板**（`doc-summary`）：传入文档内容和摘要长度，返回标准化的摘要生成 Prompt

#### 3.1 定义 Prompt 模板

创建 `EnterprisePrompts.java`：

```java
/**
 * 企业知识库助手 Prompt 模板集
 * 每个 @McpPrompt 方法会自动注册为一个 MCP Prompt 模板
 */
@Component
public class EnterprisePrompts {

    /**
     * 知识库问答模板
     * 封装了角色定义、回答规则、引用要求、兜底策略
     */
    @McpPrompt(
            name = "knowledge-qa",
            description = "知识库问答模板，基于检索到的知识片段回答用户问题，包含引用规则和兜底策略"
    )
    public GetPromptResult knowledgeQaPrompt(
            @McpArg(name = "context", description = "检索到的知识片段，多个片段用换行分隔，每个片段带编号", required = true)
            String context,
            @McpArg(name = "question", description = "用户的原始问题", required = true)
            String question) {

        String userMessage = String.format("""
                你是一个企业知识库助手。请严格遵守以下规则：

                【回答规则】
                1. 只基于「参考资料」中的内容回答，不要使用你自己的知识。
                2. 如果参考资料中没有相关信息，请回答：抱歉，我在知识库中没有找到相关信息，建议您联系人工客服获取帮助。
                3. 不要编造、推测或补充参考资料中没有的细节。

                【引用规则】
                1. 回答中引用参考资料时，使用 [编号] 标注来源，例如 [1]、[2]。
                2. 只引用你实际使用的片段，不要空挂引用。
                3. 如果多个片段支持同一个观点，可以同时引用，例如 [1][2]。

                【格式要求】
                1. 先给结论，再给详细解释。
                2. 使用简洁的中文回答。
                3. 如果信息有冲突，以更新时间较近的片段为准。

                参考资料：
                %s

                问题：%s
                """, context, question);

        return new GetPromptResult(
                "知识库问答",
                List.of(
                        new PromptMessage(Role.USER, new TextContent(userMessage))
                )
        );
    }

    /**
     * 文档摘要模板
     * 标准化的文档摘要生成，控制摘要长度和格式
     */
    @McpPrompt(
            name = "doc-summary",
            description = "文档摘要生成模板，将长文档压缩为指定长度的结构化摘要"
    )
    public GetPromptResult docSummaryPrompt(
            @McpArg(name = "document", description = "需要摘要的文档内容", required = true)
            String document,
            @McpArg(name = "maxLength", description = "摘要的最大字数，默认 200", required = false)
            String maxLength) {

        int length = 200;
        if (maxLength != null && !maxLength.isEmpty()) {
            try {
                length = Integer.parseInt(maxLength);
            } catch (NumberFormatException e) {
                // 解析失败用默认值
            }
        }

        String userMessage = String.format("""
                你是一个专业的文档摘要助手。请按以下要求生成摘要：

                1. 摘要长度不超过 %d 字。
                2. 保留文档的核心观点和关键数据。
                3. 使用以下结构：
                   - 【主题】一句话概括文档主题
                   - 【要点】3~5 个核心要点，每个要点一句话
                   - 【结论】一句话总结
                4. 不要添加文档中没有的信息。
                5. 使用简洁的中文。

                请为以下文档生成摘要：

                %s
                """, length, document);

        return new GetPromptResult(
                "文档摘要",
                List.of(
                        new PromptMessage(Role.USER, new TextContent(userMessage))
                )
        );
    }
}
```

几个要注意的点：

1. **`@McpPrompt` 注解**：`name` 是模板的唯一标识（Client 用这个名字来获取模板），`description` 是模板的描述
2. **`@McpArg` 注解**：定义模板的参数。`required = true` 表示必填参数，`required = false` 表示可选参数
3. **返回类型是 `GetPromptResult`**：包含一个描述和一个 `PromptMessage` 列表。每个 `PromptMessage` 有 `Role`（USER / ASSISTANT）和内容（`TextContent`）。注意 MCP 协议只支持 `user` 和 `assistant` 两种角色，不支持 `system`——如果需要设置系统指令，把它放在 user 消息的开头即可
4. **参数类型都是 `String`**：即使语义上是数字（如 `maxLength`），MCP 协议传递的参数都是字符串，需要在方法内自行转换

#### 3.2 Prompts 和系列前面讲的 Prompt 工程的关系

你可能注意到了，`knowledge-qa` 模板里的那些规则（限定知识来源、引用标注、兜底指令），和系列中讲 RAG 之 Prompt 工程里讲的内容一模一样。

没错，MCP Prompts 就是把你在 Prompt 工程中沉淀的最佳实践，封装成可复用的模板。以前这些 Prompt 写在每个 Client 的代码里，现在集中放到 MCP Server 上，所有 Client 共用一套。

Prompt 工程解决的是怎么写好 Prompt，MCP Prompts 解决的是怎么把好 Prompt 分发出去。

## Tools vs Resources vs Prompts：怎么选

三大能力各有定位，互补而非竞争。一张表理清楚：

| 维度 | Tools | Resources | Prompts |
|------|-------|-----------|---------|
| 本质 | 执行操作 | 提供数据 | 提供模板 |
| 控制方 | 模型驱动 | 应用驱动 | Client 驱动（通常由用户选择） |
| 有无副作用 | 可能有（可读可写） | 无（只读） | 无 |
| 返回内容 | 操作结果（结构化内容） | 资源内容（text / blob） | messages 数组 |
| 典型场景 | 查年假、下订单、发邮件 | 获取应用配置、查看表结构、读取日志 | 知识库问答模板、文档摘要模板 |
| Spring AI 注解 | `@McpTool` | `@McpResource` | `@McpPrompt` |
| 协议方法 | `tools/call` | `resources/read` | `prompts/get` |
| 类比 | 点菜（触发厨房动作） | 看菜单（读取信息） | 用点菜模板（标准化的点菜流程） |

拿到一个需求时，用这个决策流程来判断：

![](https://oss.open8gu.com/iShot_2026-03-06_10.13.34.png)

一个成熟的 MCP Server 通常会同时提供三种能力。拿企业知识库助手来说：

- **Tools**：查年假（`getUserAnnualLeave`）、查订单状态（`getOrderStatus`）——需要调用外部系统，可能有副作用
- **Resources**：应用配置（`config://app/settings`）、数据库表结构（`db://tables/{tableName}/schema`）、系统运行状态（`status://health`）——不走检索，作为固定上下文提供
- **Prompts**：知识库问答模板（`knowledge-qa`）、文档摘要模板（`doc-summary`）——标准化的交互流程

三者各司其职，配合使用。

## 小结

核心要点回顾：

1. **Resources 是看资料**：Server 暴露数据，Client 读取后作为上下文提供给模型。和 Tools 的关键区别是只读、无副作用。有两种类型——直接资源（固定 URI）和资源模板（带参数的 URI 模板），通过 `resources/list`、`resources/templates/list`、`resources/read` 三个协议方法交互
2. **Prompts 是用模板**：Server 预定义标准化的 Prompt 模板，Client 传入参数获取完整的 messages 数组。解决的是怎么把好 Prompt 分发出去的问题，让多个 Client 共用一套经过验证的最佳实践
3. **三大能力互补**：Tools 是模型的手（执行操作），Resources 是模型的参考书（提供数据），Prompts 是模型的操作手册（标准化交互）。判断用哪个：有副作用用 Tools，只读数据用 Resources，标准化模板用 Prompts
4. **Spring AI 支持**：`@McpResource` 定义资源，`@McpPrompt` + `@McpArg` 定义 Prompt 模板，在 `application.yml` 中开启对应的 `capabilities` 即可

这篇把之前 MCP 文章留下的坑补上了——MCP 的三大核心能力：Tools、Resources、Prompts，现在全部讲完。
