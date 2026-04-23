## 引言

在之前的文章中，咱们讲了 MCP 协议如何解决工具管理的标准化问题，让工具的注册、发现、调用变得更规范。再往前一篇咱们搞清楚了 Function Call 的基本原理和协议细节。

但这里有个关键问题：**协议和框架只是解决了“怎么调用工具”的问题，工具本身的质量才决定了系统的上限**。

打个比方，MCP 协议就像是给你提供了一套标准的餐具和上菜流程，但菜好不好吃，还得看厨师的手艺。工具定义写得烂，模型选错工具、参数传错、调用失败，用户体验就会很差。

举个真实的线上问题：

> 用户问：我还剩几天年假？
>
> 系统定义了一个工具 `getUserInfo`，description 写的是查询用户信息，参数有 `userId`、`infoType`（可选值：annualLeave、sickLeave、salary、attendance）。
>
> 结果模型调用时，`infoType` 传成了 `annual_leave`（下划线格式），工具执行失败返回“参数格式错误”。模型拿到这个错误后，又重试了一次，这次传成了 `AnnualLeave`（首字母大写），还是失败。最后模型只能回复用户："抱歉，系统出错了，请稍后再试。"

这个问题的根源在哪？不是 Function Call 协议有问题，也不是模型太笨，而是**工具定义本身设计得不好**：

1. 参数太多太灵活（`infoType` 枚举值没有明确约束）
2. 描述太模糊（查询用户信息太宽泛，模型不知道什么时候该用）
3. 错误处理不友好（返回参数格式错误，模型不知道怎么修正）
4. 没有参数校验和容错（应该支持多种格式或给出明确的格式要求）

**工具调用不只是技术问题，更是设计问题**。好的工具定义能让模型更容易选对工具、传对参数、处理好异常，最终给用户更好的体验。

这篇文章就来聊聊**工具调用的最佳实践**：怎么设计好的工具定义、怎么写清晰的工具描述、怎么处理错误和异常、怎么保证安全性和可观测性。这些原则和技巧适用于 Function Call 和 MCP 两种方式，也适用于任何需要让 AI 调用外部工具的场景。

## 工具定义的设计原则

### 1. 单一职责原则

**一个工具只做一件事，不要把多个功能塞进一个工具。**

反例：

```json
{
  "name": "getUserInfo",
  "description": "查询用户信息，包括年假、病假、考勤、工资等",
  "parameters": {
    "type": "object",
    "properties": {
      "userId": {
        "type": "string",
        "description": "用户 ID"
      },
      "infoType": {
        "type": "string",
        "description": "信息类型：annualLeave（年假）、sickLeave（病假）、attendance（考勤）、salary（工资）"
      }
    },
    "required": ["userId", "infoType"]
  }
}
```

这个工具定义有什么问题？

1. **功能太杂**：年假、病假、考勤、工资是完全不同的业务领域，查询逻辑、权限控制、数据来源都不一样
2. **模型容易选错**：用户问“我还剩几天年假”，模型要先判断该用这个工具，再判断 `infoType` 该传什么值，多了一层判断就多了一次出错的机会
3. **维护成本高**：后续要加新的信息类型（比如加班时长），就得改这个工具的定义和实现，改一个地方可能影响其他功能

正例：

```json
[
  {
    "name": "getUserAnnualLeave",
    "description": "查询用户的年假余额，包括剩余天数、已用天数、总天数。适用于用户询问年假、假期余额、剩余天数等问题。",
    "parameters": {
      "type": "object",
      "properties": {
        "userId": {
          "type": "string",
          "description": "用户 ID"
        }
      },
      "required": ["userId"]
    }
  },
  {
    "name": "getUserSickLeave",
    "description": "查询用户的病假余额，包括剩余天数、已用天数、总天数。适用于用户询问病假相关问题。",
    "parameters": {
      "type": "object",
      "properties": {
        "userId": {
          "type": "string",
          "description": "用户 ID"
        }
      },
      "required": ["userId"]
    }
  }
]
```

拆分后的好处：

- **模型更容易选对工具**：用户问年假，模型直接匹配到 `getUserAnnualLeave`，不需要再判断参数
- **工具描述更清晰**：每个工具的 description 可以加入更多关键词（年假、假期余额、剩余天数），提高匹配准确率
- **维护更简单**：要改年假查询逻辑，只改 `getUserAnnualLeave`，不影响其他工具
- **权限控制更精细**：可以单独控制每个工具的权限（比如工资信息只有 HR 能查）

### 2. 参数最小化原则

**只暴露必要的参数，不要把所有可能的参数都加上。**

反例：

```json
{
  "name": "searchKnowledgeBase",
  "description": "在知识库中搜索相关文档",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "搜索关键词"
      },
      "topK": {
        "type": "integer",
        "description": "返回结果数量，默认 5"
      },
      "threshold": {
        "type": "number",
        "description": "相似度阈值，0-1 之间，默认 0.7"
      },
      "enableRerank": {
        "type": "boolean",
        "description": "是否启用重排序，默认 true"
      },
      "filter": {
        "type": "object",
        "description": "过滤条件，如 {\"category\": \"产品文档\"}"
      },
      "sortBy": {
        "type": "string",
        "description": "排序方式：relevance（相关性）、time（时间）"
      },
      "includeMetadata": {
        "type": "boolean",
        "description": "是否返回元数据，默认 false"
      }
    },
    "required": ["query"]
  }
}
```

这个工具定义有 7 个参数，只有 `query` 是必填的。问题在哪？

1. **模型容易传错参数**：参数越多，模型越容易搞混（`threshold` 该传 0.7 还是 70？`sortBy` 该传 relevance 还是 Relevance？）
2. **用户体验差**：用户只是想问个问题，模型却要花时间判断这么多参数，响应变慢
3. **大部分参数用不上**：实际场景中，90% 的查询用默认值就够了，暴露这么多参数反而增加复杂度

正例：

```json
{
  "name": "searchKnowledgeBase",
  "description": "在知识库中搜索相关文档，返回最相关的 5 条结果。适用于用户询问产品功能、使用方法、常见问题等。",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "搜索关键词或问题"
      }
    },
    "required": ["query"]
  }
}
```

只保留 `query` 一个参数，其他参数在工具实现中用合理的默认值：

- `topK = 5`（大部分场景够用）
- `threshold = 0.7`（经验值）
- `enableRerank = true`（提高准确率）
- `sortBy = relevance`（按相关性排序）
- `includeMetadata = true`（方便引用）

如果真的需要灵活控制这些参数（比如高级用户场景），可以提供另一个工具 `searchKnowledgeBaseAdvanced`，但大部分用户用不到。

**设计原则：尽量减少必填参数，降低模型出错概率。参数越少，模型越容易用对。**

### 3. 幂等性原则

**查询类工具天然幂等，操作类工具要设计成幂等。**

什么是幂等性？同样的请求执行多次，结果和执行一次一样。

查询类工具（`getUserAnnualLeave`、`searchKnowledgeBase`）天然幂等，调用多少次都只是查数据，不会产生副作用。

但操作类工具（`submitExpense`、`createOrder`、`sendEmail`）就要小心了，模型可能重复调用工具：

- 网络超时重试
- 用户重复提问（帮我提交报销 → 等了 10 秒没反应 → 再问一次帮我提交报销）
- 模型自己判断失败重试

反例：

```json
{
  "name": "submitExpense",
  "description": "提交报销申请",
  "parameters": {
    "type": "object",
    "properties": {
      "amount": {
        "type": "number",
        "description": "报销金额"
      },
      "reason": {
        "type": "string",
        "description": "报销事由"
      }
    },
    "required": ["amount", "reason"]
  }
}
```

这个工具每次调用都会创建一条新的报销记录，如果模型重复调用，用户就会提交多次报销。

正例：

```json
{
  "name": "submitExpense",
  "description": "提交报销申请。如果相同的 requestId 已经提交过，则返回已有的报销记录，不会重复创建。",
  "parameters": {
    "type": "object",
    "properties": {
      "requestId": {
        "type": "string",
        "description": "请求 ID，用于幂等控制，建议使用 UUID"
      },
      "amount": {
        "type": "number",
        "description": "报销金额"
      },
      "reason": {
        "type": "string",
        "description": "报销事由"
      }
    },
    "required": ["requestId", "amount", "reason"]
  }
}
```

工具实现中，用 `requestId` 做幂等控制：

```java
public ToolResult submitExpense(String requestId, double amount, String reason) {
    // 先查询是否已经提交过
    Expense existing = expenseRepository.findByRequestId(requestId);
    if (existing != null) {
        return ToolResult.success(existing); // 返回已有记录，不重复创建
    }

    // 创建新的报销记录
    Expense expense = new Expense();
    expense.setRequestId(requestId);
    expense.setAmount(amount);
    expense.setReason(reason);
    expenseRepository.save(expense);

    return ToolResult.success(expense);
}
```

**设计原则：操作类工具必须支持幂等，避免重复执行产生副作用。**

### 4. 返回值结构化原则

**返回 JSON 格式，不要返回纯文本或 HTML。包含足够的信息让模型生成好的答案，但不要冗余。**

反例 1：返回纯文本

```java
public String getUserAnnualLeave(String userId) {
    User user = userRepository.findById(userId);
    return "您还剩 " + user.getRemainingDays() + " 天年假";
}
```

模型拿到这个文本后，只能原样返回给用户，无法做进一步处理（比如用户追问“我总共有多少天年假”，模型答不上来）。

反例 2：返回太多冗余信息

```json
{
  "success": true,
  "data": {
    "userId": "12345",
    "userName": "张三",
    "department": "技术部",
    "position": "高级工程师",
    "email": "zhangsan@example.com",
    "phone": "13800138000",
    "annualLeave": {
      "totalDays": 10,
      "usedDays": 3,
      "remainingDays": 7,
      "expiryDate": "2026-12-31"
    }
  }
}
```

用户只是问年假，返回这么多无关信息（姓名、部门、职位、邮箱、电话）浪费 token，也增加模型处理负担。

正例：

```json
{
  "success": true,
  "data": {
    "totalDays": 10,
    "usedDays": 3,
    "remainingDays": 7,
    "expiryDate": "2026-12-31"
  }
}
```

只返回年假相关的信息，模型可以灵活组织答案：

- 用户问：我还剩几天年假 → 您还剩 7 天年假
- 用户问：我总共有多少天年假 → 您总共有 10 天年假，已用 3 天，还剩 7 天
- 用户问：年假什么时候过期 → 您的年假将在 2026 年 12 月 31 日过期

错误响应也要结构化：

```json
{
  "success": false,
  "errorCode": "PERMISSION_DENIED",
  "errorMessage": "您没有权限查询该用户的年假信息",
  "details": {
    "requestedUserId": "67890",
    "currentUserId": "12345"
  }
}
```

模型拿到这个错误后，可以生成友好的提示：“抱歉，您只能查询自己的年假信息，无法查询其他用户的数据。”

**设计原则：返回结构化的 JSON，包含足够但不冗余的信息，错误响应要包含错误码和错误信息。**

### 5. 设计原则对比总结

| 设计原则 | 反例 | 正例 | 核心优势 |
|---------|------|------|---------|
| 单一职责 | getUserInfo（年假+病假+工资+考勤） | getUserAnnualLeave、getUserSickLeave | 模型更容易选对工具，维护成本低 |
| 参数最小化 | searchKnowledgeBase（7 个参数） | searchKnowledgeBase（1 个参数 query） | 降低模型出错概率，响应更快 |
| 幂等性 | submitExpense（无 requestId） | submitExpense（带 requestId） | 避免重复提交产生副作用 |
| 返回值结构化 | 返回纯文本"您还剩 7 天年假" | 返回 JSON：totalDays/usedDays/remainingDays | 模型可灵活组织答案，支持追问 |

## 工具描述的编写技巧

工具的 `description` 字段是模型判断该不该用这个工具的关键依据。description 写得好，模型选对工具的概率就高；写得烂，模型就会选错工具或者根本不用。

### 1. 描述的三要素

一个好的工具描述应该包含三个要素：

**（1）功能说明：这个工具做什么**

用一句话说清楚工具的核心功能，不要模糊不清。

- ❌ 差：查询用户信息（太宽泛）
- ✅ 好：查询用户的年假余额，包括剩余天数、已用天数、总天数

**（2）适用场景：什么时候用这个工具**

加入用户可能使用的关键词，帮助模型匹配。

- ❌ 差：查询用户的年假余额（只说了功能，没说场景）
- ✅ 好：查询用户的年假余额，包括剩余天数、已用天数、总天数。适用于用户询问年假、假期余额、剩余天数、还有几天假等问题。

**（3）参数说明：每个参数的含义和示例**

在参数的 `description` 中说清楚参数的含义、格式、示例值。

```json
{
  "name": "searchKnowledgeBase",
  "description": "在知识库中搜索相关文档，返回最相关的 5 条结果。适用于用户询问产品功能、使用方法、常见问题、故障排查等。",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "搜索关键词或问题，例如：'如何重置密码'、'产品支持哪些支付方式'"
      }
    },
    "required": ["query"]
  }
}
```

### 2. 关键词优化

模型根据 description 匹配用户问题，关键词越准确，匹配越精准。

举个例子，用户问：我还有多少天年假？

如果工具描述是：

```json
{
  "name": "getUserAnnualLeave",
  "description": "查询用户的年假余额"
}
```

模型可能匹配不到，因为用户说的是“多少天年假”，描述里只有“年假余额”。

优化后：

```json
{
  "name": "getUserAnnualLeave",
  "description": "查询用户的年假余额，包括剩余天数、已用天数、总天数。适用于用户询问年假、假期余额、剩余天数、还有几天假、年假什么时候过期等问题。"
}
```

加入了剩余天数、还有几天假等关键词，模型更容易匹配到。

**技巧：把用户可能问的各种说法都加到 description 里，提高匹配准确率。**

### 3. 多工具场景的描述策略

如果有多个相似的工具，description 要突出差异，避免模型选错。

举个例子，系统有两个工具：

```json
[
  {
    "name": "getUserAnnualLeave",
    "description": "查询用户的年假余额"
  },
  {
    "name": "getUserSickLeave",
    "description": "查询用户的病假余额"
  }
]
```

用户问：我还有几天假？

模型可能不知道该用哪个工具（年假还是病假？）。

优化后：

```json
[
  {
    "name": "getUserAnnualLeave",
    "description": "查询用户的年假余额（带薪休假），包括剩余天数、已用天数、总天数。适用于用户询问年假、带薪假期、休假余额等问题。注意：年假和病假是不同的假期类型。"
  },
  {
    "name": "getUserSickLeave",
    "description": "查询用户的病假余额（因病请假），包括剩余天数、已用天数、总天数。适用于用户询问病假、因病请假等问题。注意：病假和年假是不同的假期类型。"
  }
]
```

加入了带薪休假、因病请假等区分性描述，模型更容易判断。

如果用户问的是“我还有几天假”（没说年假还是病假），模型可能会反问用户：您是想查询年假还是病假？

**技巧：用优先使用、仅当等引导词控制工具选择优先级。**

举个例子，系统有两个搜索工具：

```json
[
  {
    "name": "searchKnowledgeBase",
    "description": "在知识库中搜索相关文档。优先使用此工具回答产品功能、使用方法、常见问题等。"
  },
  {
    "name": "searchOrderHistory",
    "description": "查询用户的历史订单。仅当用户明确询问订单、物流、退款等问题时使用此工具。"
  }
]
```

用户问：如何申请退款？

模型会优先用 `searchKnowledgeBase`（因为有优先使用），而不是 `searchOrderHistory`（因为有仅当用户明确询问订单）。

### 4. 避免歧义的描述

**不要用模糊词**

- ❌ 差：查询用户信息（什么信息？）
- ✅ 好：查询用户的年假余额，包括剩余天数、已用天数、总天数

**不要用否定句**

- ❌ 差：不适用于查询工资信息（模型可能理解反了）
- ✅ 好：仅适用于查询年假信息

**用祈使句和主动语态**

- ❌ 差：可以用来查询年假（被动语态，不够直接）
- ✅ 好：查询用户的年假余额（祈使句，简洁明确）

## 工具参数的设计模式

### 1. 必填 vs 可选参数

**必填参数**：模型必须提供，否则工具无法执行。

**可选参数**：有合理的默认值，模型可以不提供。

设计原则：**尽量减少必填参数，降低模型出错概率。**

举个例子：

```json
{
  "name": "searchKnowledgeBase",
  "description": "在知识库中搜索相关文档",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "搜索关键词或问题"
      },
      "topK": {
        "type": "integer",
        "description": "返回结果数量，默认 5，范围 1-20"
      }
    },
    "required": ["query"]
  }
}
```

`query` 是必填的（不知道搜什么就没法搜），`topK` 是可选的（默认 5 就够用）。

如果把 `topK` 也设为必填，模型每次都要判断该传多少，增加出错概率。

### 2. 参数的默认值设计

默认值要符合大多数场景的需求，不要用极端值。

- `topK` 默认 5（不是 1 也不是 100）
- `timeout` 默认 30 秒（不是 1 秒也不是 300 秒）
- `enableCache` 默认 true（大部分场景需要缓存）

不要用 `null` 或空字符串作为默认值，容易引发错误。

### 3. 参数校验

工具执行前要校验参数，不要相信模型传的参数一定是对的。

**（1）类型校验**

```java
public ToolResult searchKnowledgeBase(String query, Integer topK) {
    if (query == null || query.trim().isEmpty()) {
        return ToolResult.error("INVALID_PARAMETER", "参数 query 不能为空");
    }
    if (topK != null && (topK < 1 || topK > 20)) {
        return ToolResult.error("INVALID_PARAMETER", "参数 topK 必须在 1-20 之间");
    }
    // ...
}
```

**（2）格式校验**

```java
public ToolResult getUserInfo(String userId) {
    if (!userId.matches("^[0-9]{6}$")) {
        return ToolResult.error("INVALID_PARAMETER", "参数 userId 格式错误，应为 6 位数字");
    }
    // ...
}
```

**（3）业务校验**

```java
public ToolResult getUserAnnualLeave(String userId, String currentUserId) {
    // 权限校验：只能查自己的年假
    if (!userId.equals(currentUserId)) {
        return ToolResult.error("PERMISSION_DENIED", "您只能查询自己的年假信息");
    }
    // ...
}
```

校验失败时返回清晰的错误信息，帮助模型理解问题。

### 4. 参数的命名规范

- 使用驼峰命名（`userId` 而不是 `user_id`）
- 使用语义化的名字（`query` 而不是 `q`）
- 避免缩写（除非是行业通用缩写，如 `id`、`url`）
- 布尔值用 `is`/`enable`/`has` 开头（`isEnabled`、`hasPermission`）

## 工具返回值的格式规范

### 1. 成功响应的格式

统一使用以下格式：

```json
{
  "success": true,
  "data": {
    // 实际数据
  }
}
```

示例：

```json
{
  "success": true,
  "data": {
    "totalDays": 10,
    "usedDays": 3,
    "remainingDays": 7,
    "expiryDate": "2026-12-31"
  }
}
```

### 2. 错误响应的格式

统一使用以下格式：

```json
{
  "success": false,
  "errorCode": "ERROR_CODE",
  "errorMessage": "错误信息",
  "details": {
    // 可选的详细信息
  }
}
```

示例：

```json
{
  "success": false,
  "errorCode": "PERMISSION_DENIED",
  "errorMessage": "您没有权限查询该用户的年假信息",
  "details": {
    "requestedUserId": "67890",
    "currentUserId": "12345"
  }
}
```

### 3. 错误码设计

使用语义化的错误码，不要用 HTTP 状态码（403、500）。

常见错误码分类：

| 错误码 | 含义 | 示例 |
|--------|------|------|
| `INVALID_PARAMETER` | 参数错误 | 参数格式不对、必填参数缺失 |
| `PERMISSION_DENIED` | 权限错误 | 无权访问该资源 |
| `RESOURCE_NOT_FOUND` | 资源不存在 | 用户不存在、订单不存在 |
| `BUSINESS_ERROR` | 业务错误 | 余额不足、库存不足 |
| `SYSTEM_ERROR` | 系统错误 | 数据库连接失败、第三方服务超时 |
| `RATE_LIMIT_EXCEEDED` | 限流 | 调用频率超过限制 |

> 或者参考阿里巴巴开发手册规范那种，设计前置错误码+数字形式标识，都是可以的。

### 4. 返回值的信息密度

不要返回太多冗余信息（浪费 token），也不要返回太少信息（模型无法生成好的答案）。

**反例：信息太少**

```json
{
  "success": true,
  "data": {
    "remainingDays": 7
  }
}
```

用户追问"我总共有多少天年假"，模型答不上来。

**反例：信息太多**

```json
{
  "success": true,
  "data": {
    "userId": "12345",
    "userName": "张三",
    "department": "技术部",
    "position": "高级工程师",
    "email": "zhangsan@example.com",
    "phone": "13800138000",
    "hireDate": "2020-01-01",
    "annualLeave": {
      "totalDays": 10,
      "usedDays": 3,
      "remainingDays": 7,
      "expiryDate": "2026-12-31",
      "history": [
        {"date": "2026-01-10", "days": 1, "reason": "事假"},
        {"date": "2026-02-15", "days": 2, "reason": "旅游"}
      ]
    }
  }
}
```

用户只是问年假，返回这么多无关信息浪费 token。

**正例：信息适中**

```json
{
  "success": true,
  "data": {
    "totalDays": 10,
    "usedDays": 3,
    "remainingDays": 7,
    "expiryDate": "2026-12-31"
  }
}
```

包含了回答问题所需的所有信息，但没有冗余。

## 小结

工具调用是 RAG 系统从“只会查资料”到“能干活”的关键一步。Function Call 和 MCP 协议解决了怎么调用工具的问题，但工具本身的质量才决定了系统的上限。

这篇文章从设计原则、描述技巧、返回值规范三个维度，讲清楚了怎么定义好的工具。核心要点：

**设计阶段：**

- 单一职责：一个工具只做一件事，拆分比合并好
- 参数最小化：只暴露必要参数，能后端处理的不让模型传
- 幂等性：操作类工具传入 requestId，避免重复执行
- 返回值结构化：统一 JSON 格式，success + data/errorCode

**描述阶段：**

- 三要素齐全：功能说明 + 适用场景 + 参数说明
- 关键词优化：加入用户可能的各种说法
- 突出差异：多工具场景用“优先使用”、“仅当”引导
- 避免歧义：参数描述要具体，给示例

**返回值阶段：**
- 信息密度适中：不冗余、不缺失
- 错误码语义化：INVALID_PARAMETER、PERMISSION_DENIED、SYSTEM_ERROR
- 错误信息友好：告诉模型怎么修正，而不是只说“错了”

好的工具定义能让模型选对工具、传对参数、处理好异常，最终给用户更好的体验。但工具定义写得再好，线上跑起来还是会遇到各种问题：网络超时、第三方服务挂了、用户传了非法参数、权限不足、SQL 注入攻击……

下一篇咱们聊聊工具调用的错误处理、安全防护、测试验证、监控告警，讲清楚怎么让工具调用在生产环境稳定运行。

