上一篇大家已经能消费大模型的流式 API 了。`SseStreamClient` 类封装好了，`onToken` 回调里能实时拿到每个 token，`onComplete` 回调里能拿到完整内容和 Token 统计。

但现在有个问题：这些 token 只在你的 Java 进程里。用户在浏览器里什么都看不到。

你需要把后端收到的流式数据**实时转发给前端**，这个时候 Spring Boot 服务要扮演双重角色——对大模型 API 来说它是 SSE 客户端，对前端浏览器来说它是 SSE 服务端。

这篇就来讲怎么用 Spring Boot 搭建 SSE 服务端，实现**前端 → Spring Boot → 大模型 API → 前端**的完整流式转发链路，以及生产环境绕不开的那些工程细节。

## 为什么需要后端中间层

### 1. 前端直连大模型 API 的三个问题

你可能会想：前端直接调大模型 API 不就行了？何必多一层后端中间层？

不行。有三个硬伤：

- **API Key 暴露**：前端代码跑在用户的浏览器里，你的 JavaScript 代码、网络请求对用户是完全透明的。打开 F12 → Network 面板，请求头里的 `Authorization: Bearer sk-xxx` 一目了然。API Key 泄露意味着任何人都能用你的额度调用大模型，账单直接爆炸。

- **跨域限制**：大模型 API 的域名（比如 `api.siliconflow.cn`）和你的前端域名（比如 `www.yourapp.com`）不同，浏览器的同源策略会拦截请求。虽然可以通过 CORS 解决，但 CORS 的响应头需要**服务端**配置——你控制不了第三方大模型 API 的 CORS 策略。

- **无法加业务逻辑**：用户是否登录了？是否有调用权限？单个用户每分钟最多调几次？对话记录要不要存？敏感词要不要过滤？这些业务逻辑都需要在后端处理。前端直连等于跳过了整个业务层，你的系统就是一个没有门禁的大楼。

所以架构必须是这样的：

```java
前端浏览器 → 你的 Spring Boot 后端 → 大模型 API
```

后端负责鉴权、限流、日志、对话历史存储，然后代理调用大模型 API，把流式响应转发给前端。

### 2. 完整链路概览

完整的流式转发链路用时序图展示：

![](https://oss.open8gu.com/iShot_2026-03-06_10.13.29.png)

有一个关键点需要注意：**Controller 线程不会被长时间占用**。Controller 方法创建 `SseEmitter` 对象后立即返回，线程就释放了。真正的数据推送是在另一个异步线程里进行的。这意味着即使流式生成需要 30 秒，Tomcat 的 Controller 线程也只占用了几毫秒。

## Spring Boot SseEmitter 核心机制

### 1. SseEmitter 是什么

`SseEmitter` 是 Spring MVC 提供的异步响应工具，专门用于 SSE 服务端推送。

普通的 Controller 返回值（比如 `ResponseEntity`、`String`、`Map`）是**同步响应**——方法执行完，响应就发了，HTTP 连接就关了。一问一答，干脆利落。

`SseEmitter` 是**异步响应**——Controller 方法返回一个 `SseEmitter` 对象后，HTTP 连接**不会关闭**。你可以在其他线程里通过这个 `SseEmitter` 对象持续向客户端推送事件，直到你调用 `emitter.complete()` 主动关闭连接。

打个比方：普通 Controller 像自动售货机——投币、出货、走人。`SseEmitter` 像外卖平台——你下了单（发了请求），然后骑手持续给你推送状态更新（已接单、正在配送、即将到达），直到送达（`complete()`）。

### 2. SseEmitter 的核心 API

`SseEmitter` 的 API 不多，核心就这几个：

| API | 作用 | 说明 |
|-----|------|------|
| `new SseEmitter(timeout)` | 创建实例，设置超时 | 超时单位毫秒，超时后自动关闭连接 |
| `send(Object data)` | 推送默认类型事件 | 事件类型为 `message`，data 会被序列化为字符串 |
| `send(SseEventBuilder event)` | 推送自定义事件 | 可指定 event 名称、id、data、comment |
| `complete()` | 正常结束 | 关闭 SSE 连接 |
| `completeWithError(Throwable ex)` | 异常结束 | 关闭连接并触发 onError 回调 |
| `onCompletion(Runnable)` | 连接关闭回调 | 不管正常还是异常关闭都会触发 |
| `onTimeout(Runnable)` | 超时回调 | 超时时触发 |
| `onError(Consumer<Throwable>)` | 错误回调 | 出错时触发 |

其中 `send(SseEventBuilder)` 是最常用的，因为它可以指定事件类型：

```java
// 推送一个 token 事件
emitter.send(SseEmitter.event()
        .name("token")                    // event: token
        .data("{\"content\":\"你好\"}"));  // data: {"content":"你好"}

// 推送一个 done 事件
emitter.send(SseEmitter.event()
        .name("done")
        .data("{\"usage\":{\"total_tokens\":128}}"));

// 推送一个注释（心跳保活）
emitter.send(SseEmitter.event()
        .comment("keepalive"));
```

还记得上一篇讲的 SSE `event:` 字段吗？大模型 API 通常不用 `event:` 字段，所有事件都是默认的 `message` 类型。但你自己搭建 SSE 服务端时，`event:` 字段就很有用了——前端可以根据不同的事件类型做不同的处理。

### 3. SseEmitter 的线程模型

`SseEmitter` 的异步本质是理解它的关键。整个过程涉及两个线程：

**Controller 线程**（Tomcat 线程池里的线程）：
1. 接收 HTTP 请求
2. 创建 `SseEmitter` 对象
3. 启动异步任务（把 `SseEmitter` 对象传给异步线程）
4. 返回 `SseEmitter` 对象
5. Controller 线程结束，**释放回 Tomcat 线程池**

**推送线程**（你自己创建的异步线程）：
1. 拿着 `SseEmitter` 对象
2. 调用大模型流式 API
3. 在回调里调用 `emitter.send()` 逐 Token 推送
4. 调用 `emitter.complete()` 结束

这个设计的好处是 **Controller 线程不会被长时间占用**。如果不用 `SseEmitter`，而是在 Controller 里直接写一个循环发送数据，那这个请求会占住一个 Tomcat 线程直到流结束——30 秒的流式生成就占住一个线程 30 秒。Tomcat 默认 200 个线程，200 个并发用户就把线程池打满了。

用 `SseEmitter`，Controller 线程几毫秒就释放了。推送操作在独立的线程中进行，不占 Tomcat 线程池。

> 如果你在 Controller 线程里直接调 `send()` 发完所有数据再 `complete()`，那和普通接口没什么区别——线程还是被占住了。`SseEmitter` 的价值在于**跨线程推送**：Controller 线程返回，推送线程接管。

## Java 实战：三个层次的 SSE 服务端

### 1. 最简版：Hello SSE

> 下述为示例代码，大家可自行创建项目进行运行验证，也可仅关注其中的核心思想。关于基础的 Spring Boot 框架代码逻辑，本文在概念章节中不再做过多展开。

先用最少的代码跑通一个 SSE 接口，理解 `SseEmitter` 的基本套路。

```java
@RestController
@RequestMapping("/api/sse")
public class SimpleSseController {

    @GetMapping("/hello")
    public SseEmitter hello() {
        // 创建 SseEmitter，超时 60 秒
        SseEmitter emitter = new SseEmitter(60_000L);

        // 启动一个新线程做数据推送（不能在 Controller 线程里做）
        new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    // 推送事件：event 名称为 "message"（默认），data 是字符串
                    emitter.send(SseEmitter.event()
                            .name("token")
                            .data("第 " + i + " 条消息"));
                    Thread.sleep(1000);  // 每秒推送一条
                }
                // 推送完毕，关闭连接
                emitter.send(SseEmitter.event()
                        .name("done")
                        .data("{\"msg\":\"推送完毕\"}"));
                emitter.complete();
            } catch (Exception e) {
                emitter.completeWithError(e);
            }
        }).start();

        // Controller 线程立即返回，不等推送完成
        return emitter;
    }
}
```

启动 Spring Boot 之后，用浏览器直接访问 `http://localhost:8080/api/sse/hello`，你会看到数据一条一条地出现——每秒出一条，5 秒后结束。

也可以用下面这段 HTML 在浏览器里测试（后面完整版也用这种方式验证）：

```html
<!DOCTYPE html>
<html>
<body>
<h3>SSE 测试</h3>
<div id="output"></div>
<script>
    const output = document.getElementById('output');
    const eventSource = new EventSource('/api/sse/hello');

    eventSource.addEventListener('token', function(e) {
        output.innerHTML += '<p>收到 token: ' + e.data + '</p>';
    });

    eventSource.addEventListener('done', function(e) {
        output.innerHTML += '<p style="color:green">完成: ' + e.data + '</p>';
        eventSource.close();  // 收到 done 事件后关闭连接
    });

    eventSource.onerror = function() {
        output.innerHTML += '<p style="color:red">连接断开</p>';
        eventSource.close();
    };
</script>
</body>
</html>
```

这个最简版能跑通，但离生产可用还有距离——没有接入大模型、没有心跳、没有断连检测。接下来一步步加。

### 2. 完整版：接入大模型流式转发

完整版要实现的链路是：前端发请求 → Spring Boot 接收 → 调用大模型流式 API → 逐 Token 转发给前端。

事件类型设计：

| 事件名称 | 用途 | data 格式 |
|---------|------|----------|
| `token` | 内容增量 | `{"content": "一段文字"}` |
| `done` | 流结束 | `{"usage": {"prompt_tokens":15, "completion_tokens":12}}` |
| `error` | 错误通知 | `{"code": 500, "message": "错误描述"}` |

前端根据事件类型做不同处理——收到 `token` 就拼接内容，收到 `done` 就结束，收到 `error` 就展示错误信息。

#### 2.1 请求和响应结构

```java
/** 前端请求体 */
public class ChatRequest {
    private String question;
    // getter/setter 省略
}
```

#### 2.2 Controller 层

```java
@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatService chatService;

    public ChatController(ChatService chatService) {
        this.chatService = chatService;
    }

    @GetMapping("/stream")
    public SseEmitter stream(@RequestParam String question) {
        // 创建 SseEmitter，超时 3 分钟
        SseEmitter emitter = new SseEmitter(180_000L);

        // 设置回调（连接关闭时的清理逻辑）
        emitter.onCompletion(() ->
                System.out.println("SSE 连接关闭"));
        emitter.onTimeout(() ->
                System.out.println("SSE 连接超时"));
        emitter.onError(e ->
                System.err.println("SSE 连接异常: " + e.getMessage()));

        // 异步线程中调用大模型并转发
        chatService.streamChat(question, emitter);

        return emitter;
    }
}
```

> 这里用 `@GetMapping` + `@RequestParam` 是为了兼容浏览器原生的 `EventSource` API（它只支持 GET 请求）。如果你的前端用 `fetch` 来消费 SSE（后面会讲），可以改成 `@PostMapping` + `@RequestBody`，支持更复杂的请求体。

#### 2.3 Service 层：流式转发核心逻辑

```java
@Service
public class ChatService {

    private static final String API_URL = "https://api.siliconflow.cn/v1/chat/completions";
    private static final String API_KEY = "sk-xxx";  // 替换为你的 API Key
    private static final String MODEL = "Qwen/Qwen3-32B";

    private final OkHttpClient httpClient = new OkHttpClient.Builder()
            .connectTimeout(15, TimeUnit.SECONDS)
            .readTimeout(60, TimeUnit.SECONDS)
            .writeTimeout(15, TimeUnit.SECONDS)
            .build();

    private final Gson gson = new Gson();

    /**
     * 异步调用大模型流式 API，逐 Token 通过 SseEmitter 转发给前端
     */
    public void streamChat(String question, SseEmitter emitter) {
        // 在新线程中执行（生产环境应使用线程池，后面会讲）
        new Thread(() -> doStreamChat(question, emitter)).start();
    }

    private void doStreamChat(String question, SseEmitter emitter) {
        // 1. 构建大模型请求体
        JsonObject requestBody = new JsonObject();
        requestBody.addProperty("model", MODEL);
        requestBody.addProperty("temperature", 0.7);
        requestBody.addProperty("max_tokens", 2048);
        requestBody.addProperty("stream", true);

        JsonArray messages = new JsonArray();
        JsonObject systemMsg = new JsonObject();
        systemMsg.addProperty("role", "system");
        systemMsg.addProperty("content", "你是一个技术专家，回答简洁清晰。");
        messages.add(systemMsg);

        JsonObject userMsg = new JsonObject();
        userMsg.addProperty("role", "user");
        userMsg.addProperty("content", question);
        messages.add(userMsg);
        requestBody.add("messages", messages);

        // 2. 发起 HTTP 请求
        Request request = new Request.Builder()
                .url(API_URL)
                .addHeader("Authorization", "Bearer " + API_KEY)
                .addHeader("Content-Type", "application/json")
                .addHeader("Accept", "text/event-stream")
                .post(RequestBody.create(requestBody.toString(),
                        MediaType.parse("application/json")))
                .build();

        // 3. 解析 SSE 流并转发
        try (Response response = httpClient.newCall(request).execute()) {
            if (!response.isSuccessful()) {
                sendError(emitter, response.code(),
                        "大模型 API 调用失败: HTTP " + response.code());
                return;
            }

            BufferedReader reader = new BufferedReader(
                    new InputStreamReader(response.body().byteStream(),
                            StandardCharsets.UTF_8));
            String line;

            while ((line = reader.readLine()) != null) {
                if (line.isEmpty() || line.startsWith(":")) {
                    continue;
                }
                if (!line.startsWith("data:")) {
                    continue;
                }

                String data = line.substring(5);
                if (data.startsWith(" ")) {
                    data = data.substring(1);
                }

                if ("[DONE]".equals(data)) {
                    // 流结束，发送 done 事件
                    emitter.send(SseEmitter.event()
                            .name("done")
                            .data("{\"msg\":\"completed\"}"));
                    emitter.complete();
                    return;
                }

                // 解析 JSON
                JsonObject chunk;
                try {
                    chunk = JsonParser.parseString(data).getAsJsonObject();
                } catch (Exception e) {
                    continue;  // JSON 解析失败，跳过
                }

                // 提取 content
                JsonArray choices = chunk.getAsJsonArray("choices");
                if (choices == null || choices.isEmpty()) {
                    // 可能是只有 usage 的 chunk
                    if (chunk.has("usage") && !chunk.get("usage").isJsonNull()) {
                        JsonObject usage = chunk.getAsJsonObject("usage");
                        emitter.send(SseEmitter.event()
                                .name("done")
                                .data(gson.toJson(usage)));
                        emitter.complete();
                        return;
                    }
                    continue;
                }

                JsonObject choice = choices.get(0).getAsJsonObject();
                JsonObject delta = choice.getAsJsonObject("delta");

                if (delta != null && delta.has("content")) {
                    JsonElement contentElement = delta.get("content");
                    if (!contentElement.isJsonNull()) {
                        String token = contentElement.getAsString();
                        if (!token.isEmpty()) {
                            // 核心：通过 SseEmitter 把 token 推送给前端
                            emitter.send(SseEmitter.event()
                                    .name("token")
                                    .data("{\"content\":\"" +
                                            escapeJson(token) + "\"}"));
                        }
                    }
                }

                // 检查 finish_reason，提取 usage
                JsonElement finishElement = choice.get("finish_reason");
                if (finishElement != null && !finishElement.isJsonNull()) {
                    if (chunk.has("usage") && !chunk.get("usage").isJsonNull()) {
                        JsonObject usage = chunk.getAsJsonObject("usage");
                        emitter.send(SseEmitter.event()
                                .name("done")
                                .data(gson.toJson(usage)));
                    }
                }
            }

            // readLine 返回 null 但没收到 [DONE]，连接异常
            sendError(emitter, 500, "流式响应异常结束");

        } catch (IOException e) {
            sendError(emitter, 500, "连接异常: " + e.getMessage());
        }
    }

    /** 向前端推送错误事件 */
    private void sendError(SseEmitter emitter, int code, String message) {
        try {
            emitter.send(SseEmitter.event()
                    .name("error")
                    .data("{\"code\":" + code + ",\"message\":\"" +
                            escapeJson(message) + "\"}"));
            emitter.complete();
        } catch (IOException e) {
            emitter.completeWithError(e);
        }
    }

    /** 简单的 JSON 字符串转义 */
    private String escapeJson(String s) {
        return s.replace("\\", "\\\\")
                .replace("\"", "\\\"")
                .replace("\n", "\\n")
                .replace("\r", "\\r")
                .replace("\t", "\\t");
    }
}
```

核心逻辑就是一句话：**在大模型 SSE 客户端的回调里，调用 `emitter.send()` 把数据转发给前端**。

整个链路是这样的：

1. 前端发请求 → Controller 创建 `SseEmitter`，启动异步线程，立即返回
2. 异步线程调用大模型流式 API → 逐行读取 `data:` 行
3. 解析出 `delta.content` → 通过 `emitter.send()` 推送 `token` 事件给前端
4. 收到 `[DONE]` → 推送 `done` 事件 → `emitter.complete()` 关闭连接
5. 出错 → 推送 `error` 事件 → `emitter.complete()` 关闭连接

> 注意 `sendError` 方法的设计——出错时不是直接 `completeWithError()`，而是先通过 SSE 推送一个 `error` 事件让前端知道出了什么问题，然后再 `complete()`。如果直接 `completeWithError()`，前端只知道连接断了，不知道为什么断了。

#### 2.4 前端测试页面

配一个简单的前端页面验证效果：

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>大模型流式对话</title>
    <style>
        body { font-family: sans-serif; max-width: 800px; margin: 40px auto; }
        #output { border: 1px solid #ddd; padding: 16px; min-height: 100px;
                  white-space: pre-wrap; line-height: 1.6; }
        input { width: 70%; padding: 8px; }
        button { padding: 8px 16px; }
        .meta { color: #888; font-size: 0.9em; margin-top: 8px; }
    </style>
</head>
<body>
<h3>大模型流式对话</h3>
<div>
    <input id="question" placeholder="输入你的问题..." value="用两三句话解释什么是SSE协议">
    <button onclick="startChat()">发送</button>
</div>
<div id="output"></div>
<div id="meta" class="meta"></div>

<script>
function startChat() {
    const question = document.getElementById('question').value;
    const output = document.getElementById('output');
    const meta = document.getElementById('meta');
    output.textContent = '';
    meta.textContent = '正在生成...';

    const eventSource = new EventSource(
        '/api/chat/stream?question=' + encodeURIComponent(question));

    eventSource.addEventListener('token', function(e) {
        const data = JSON.parse(e.data);
        output.textContent += data.content;
    });

    eventSource.addEventListener('done', function(e) {
        meta.textContent = '生成完毕 | ' + e.data;
        eventSource.close();
    });

    eventSource.addEventListener('error', function(e) {
        if (e.data) {
            const data = JSON.parse(e.data);
            meta.textContent = '错误: ' + data.message;
        } else {
            meta.textContent = '连接断开';
        }
        eventSource.close();
    });
}
</script>
</body>
</html>
```

启动 Spring Boot 后打开这个页面，输入问题点击发送，你就能看到逐字输出的打字机效果——和 ChatGPT 的体验一样。

> 复制出来创建个结尾为 .html 的文件，浏览器直接打开即可。

### 3. 生产版：连接管理与健壮性

完整版能跑了，但在生产环境还有几个问题需要解决。

#### 3.1 超时设置

`SseEmitter` 的默认超时是 30 秒。大模型生成一次回答可能需要 30~60 秒（特别是长文本、深度思考场景），30 秒一到 `SseEmitter` 自动关闭连接，前端直接断了。

建议把 `SseEmitter` 的超时设为 **3~5 分钟**：

```java
SseEmitter emitter = new SseEmitter(180_000L);  // 3 分钟
```

这里有一个容易混淆的地方：`SseEmitter` 的超时和 OkHttp 的 `readTimeout` 是两回事。

| 超时类型 | 含义 | 建议值 |
|---------|------|-------|
| `SseEmitter(timeout)` | 整个 SSE 连接的**生命周期上限** | 3~5 分钟 |
| OkHttp `readTimeout` | 调用大模型 API 时，两个 chunk 之间的**最大等待间隔** | 30~60 秒 |

`SseEmitter` 超时是这次对话最多持续多久，OkHttp `readTimeout` 是大模型多久没吐出新 token 就认为它卡住了。两个维度不同，不要搞混。

#### 3.2 心跳保活

模型在思考时，可能几秒甚至十几秒没有输出。如果你的 Spring Boot 前面有 Nginx 反向代理，Nginx 的 `proxy_read_timeout` 默认 60 秒——超过 60 秒没有数据传输，Nginx 会主动断开连接。

解决方案：定期发送心跳注释事件。上一篇讲过，SSE 的注释行（`:` 开头）会被客户端忽略，但对中间件来说有数据在传输，就不会超时断开。

```java
@Service
public class ChatService {

    // 心跳定时器
    private final ScheduledExecutorService heartbeatScheduler =
            Executors.newScheduledThreadPool(2);

    public void streamChat(String question, SseEmitter emitter) {
        // 启动心跳：每 15 秒发送一次注释事件
        ScheduledFuture<?> heartbeat = heartbeatScheduler.scheduleAtFixedRate(() -> {
            try {
                emitter.send(SseEmitter.event().comment("keepalive"));
            } catch (IOException e) {
                // 发送失败说明连接已断开，不需要处理
            }
        }, 15, 15, TimeUnit.SECONDS);

        new Thread(() -> {
            try {
                doStreamChat(question, emitter);
            } finally {
                // 不管成功还是失败，都要停止心跳
                heartbeat.cancel(false);
            }
        }).start();
    }
}
```

心跳间隔建议 **15~20 秒**。太频繁浪费带宽，太稀疏可能还是会被中间件断开。

#### 3.3 客户端断连检测

用户关闭浏览器标签页、刷新页面、或者网络断开，SSE 连接会从客户端侧断开。这时候后端可能还在调用大模型 API——用户已经走了，你还在花钱调 API，纯浪费。

`SseEmitter` 提供了三个回调来感知连接状态变化：

```java
SseEmitter emitter = new SseEmitter(180_000L);

emitter.onCompletion(() -> {
    System.out.println("连接关闭（正常或异常）");
    // 清理资源：停止心跳、从连接池移除
});

emitter.onTimeout(() -> {
    System.out.println("连接超时");
    // 超时也会触发 onCompletion
});

emitter.onError(e -> {
    System.out.println("连接异常: " + e.getMessage());
    // 异常也会触发 onCompletion
});
```

> `onCompletion` 是最终回调——不管连接是正常 `complete()` 结束、超时结束、还是异常结束，`onCompletion` 都会被触发。资源清理逻辑放在 `onCompletion` 里就行。

另外，当客户端断开后，`emitter.send()` 会抛出 `IOException`。所以推送代码里要做好异常处理：

```java
try {
    emitter.send(SseEmitter.event()
            .name("token")
            .data("{\"content\":\"" + escapeJson(token) + "\"}"));
} catch (IOException e) {
    // 客户端已断开，停止继续推送
    // 如果还在调用大模型 API，可以尝试取消请求
    break;
}
```

#### 3.4 线程池管理

前面的代码用 `new Thread()` 来启动异步任务，这在生产环境是不可接受的——线程创建销毁的开销大，也没法控制并发数。

> 在 Ragent AI 项目中，大量使用了线程池，如果大家学习过 oneThread 动态线程池项目，可以写 1-2 个亮点到 Ragent AI，两者结合说了属于是。

应该用专门的线程池：

```java
@Configuration
public class ThreadPoolConfig {

    /**
     * SSE 推送专用线程池
     * 注意：不要用 Tomcat 的线程池，SSE 推送是长耗时任务，会阻塞 Tomcat 线程
     */
    @Bean("sseExecutor")
    public ExecutorService sseExecutor() {
        return new ThreadPoolExecutor(
                10,                          // 核心线程数
                50,                          // 最大线程数
                60, TimeUnit.SECONDS,        // 空闲线程存活时间
                new LinkedBlockingQueue<>(100),  // 等待队列
                new ThreadFactory() {
                    private final AtomicInteger counter = new AtomicInteger(1);
                    @Override
                    public Thread newThread(Runnable r) {
                        Thread t = new Thread(r, "sse-push-" + counter.getAndIncrement());
                        t.setDaemon(true);
                        return t;
                    }
                },
                new ThreadPoolExecutor.CallerRunsPolicy()  // 队列满时由调用线程执行
        );
    }
}
```

Service 里改用线程池：

```java
@Service
public class ChatService {

    @Resource(name = "sseExecutor")
    private ExecutorService sseExecutor;

    public void streamChat(String question, SseEmitter emitter) {
        sseExecutor.submit(() -> {
            // 心跳和推送逻辑...
        });
    }
}
```

> 线程池的参数需要根据你的并发量和服务器配置来调。核心线程数可以参考你的预期并发 SSE 连接数。`CallerRunsPolicy` 拒绝策略意味着队列满时由调用线程（Controller 线程）执行——相当于降级为同步处理，不会丢失请求。

## 前端怎么接：EventSource API

后端开发需要知道前端怎么接 SSE，才能设计好接口。这里简要介绍前端消费 SSE 的两种方式。

### 1. 原生 EventSource 的基本用法

浏览器原生提供了 `EventSource` API 来消费 SSE：

```javascript
// 创建 EventSource，自动发起 GET 请求
const eventSource = new EventSource('/api/chat/stream?question=什么是SSE');

// 监听自定义事件（对应后端 SseEmitter.event().name("token") 设置的名称）
eventSource.addEventListener('token', function(event) {
    const data = JSON.parse(event.data);
    document.getElementById('output').textContent += data.content;
});

eventSource.addEventListener('done', function(event) {
    console.log('完成:', event.data);
    eventSource.close();  // 必须手动关闭，否则 EventSource 会自动重连
});

eventSource.addEventListener('error', function(event) {
    if (event.data) {
        console.error('服务端错误:', event.data);
    }
    eventSource.close();
});

// 也可以监听默认的 message 事件（event 名称为 "message" 或未指定时触发）
// eventSource.onmessage = function(event) { ... };
```

注意这里和上一篇 SSE 协议的呼应：前端 `addEventListener('token', ...)` 里的 `'token'`，就是后端 `SseEmitter.event().name("token")` 里设置的事件名称。所以设计事件类型时，前后端要约定好。

还有一个重要的点：**`EventSource` 在连接断开后会自动重连**。如果你在 `done` 事件里不调用 `eventSource.close()`，`EventSource` 发现连接断了会尝试重新连接，导致重复请求。收到结束信号后一定要手动关闭。

### 2. EventSource 的局限和替代方案

`EventSource` 有两个主要局限：

**只支持 GET 请求**。你的问题文本只能放在 URL 参数里。如果问题很长（几百字），URL 长度可能超出限制。更重要的是，如果请求体比较复杂（包含对话历史、系统配置等），塞在 URL 参数里既不优雅也不安全。

**不能自定义请求头**。想在请求头里加 `Authorization` Token 做鉴权？`EventSource` 做不到。

替代方案是用 `fetch` API + `ReadableStream`，可以发 POST 请求、自定义请求头：

```javascript
async function streamChat(question) {
    const response = await fetch('/api/chat/stream', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': 'Bearer your-token',  // 可以自定义请求头
            'Accept': 'text/event-stream'
        },
        body: JSON.stringify({ question: question })
    });

    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let buffer = '';

    while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        buffer += decoder.decode(value, { stream: true });
        // 按行分割，处理 SSE 格式
        const lines = buffer.split('\n');
        buffer = lines.pop();  // 最后一个可能不完整，留到下一次

        for (const line of lines) {
            if (line.startsWith('event: ')) {
                // 解析事件类型
            } else if (line.startsWith('data: ')) {
                // 解析数据
            }
        }
    }
}
```

> 这也是为什么很多 AI 产品的前端不用原生 `EventSource`，而是用 `fetch` 手动处理 SSE 流——因为对话请求通常是 POST 且需要带 Token。如果你的后端用 `@PostMapping`，前端就必须用 `fetch` 方案。

如果后端用 `@PostMapping`，Controller 只需要改一行：

```java
@PostMapping("/stream")
public SseEmitter stream(@RequestBody ChatRequest request) {
    // ... 其余逻辑不变
}
```

## 生产环境的关键配置

### 1. Nginx 反向代理 SSE

这是生产环境**最常踩的坑**。

默认情况下，Nginx 开启了 `proxy_buffering`——它会把后端的响应数据缓冲在内存里，攒够一定量（或等一段时间）再一次性转发给客户端。对普通 HTTP 请求来说，缓冲可以提升性能，减少后端压力。

但对 SSE 来说，缓冲是致命的。后端 `emitter.send()` 了一个 token，Nginx 缓冲住了不转发，前端收不到数据。等 Nginx 缓冲区满了或者超时了，一堆数据才一次性涌过来。前端看到的效果就是：**卡了好几秒，然后"哗"一下出来一大段**——完全不是逐字输出的打字机效果。

解决方案：在 Nginx 的 SSE 接口配置中关闭缓冲。

```nginx
location /api/chat/stream {
    proxy_pass http://your-backend;

    # 关键：关闭代理缓冲，数据立即转发
    proxy_buffering off;

    # 关闭代理缓存
    proxy_cache off;

    # 设置足够长的读取超时（SSE 是长连接）
    proxy_read_timeout 300s;

    # 确保使用 HTTP/1.1 长连接
    proxy_http_version 1.1;
    proxy_set_header Connection '';

    # 开启 chunked 传输编码
    chunked_transfer_encoding on;

    # 告诉 Nginx 不要缓冲 SSE 响应
    # 有些 Nginx 版本即使关了 proxy_buffering，也需要这个头
    proxy_set_header X-Accel-Buffering no;
}
```

每个配置项的作用：

| 配置项 | 作用 | 不配的后果 |
|-------|------|----------|
| `proxy_buffering off` | 关闭代理缓冲，后端数据立即转发 | 数据被缓冲，前端收不到实时推送 |
| `proxy_cache off` | 关闭代理缓存 | SSE 响应可能被缓存，返回过期数据 |
| `proxy_read_timeout 300s` | 读取超时设为 5 分钟 | 默认 60 秒，长文本生成可能超时断开 |
| `proxy_http_version 1.1` | 使用 HTTP/1.1 | HTTP/1.0 不支持长连接 |
| `proxy_set_header Connection ''` | 清除 Connection 头 | 可能导致连接被提前关闭 |
| `X-Accel-Buffering no` | 额外的缓冲关闭指令 | 某些场景下 proxy_buffering off 不生效 |

你也可以在 Spring Boot 侧通过响应头来关闭缓冲：

```java
@GetMapping("/stream")
public SseEmitter stream(@RequestParam String question,
                          HttpServletResponse response) {
    // 告诉 Nginx 不要缓冲这个响应
    response.setHeader("X-Accel-Buffering", "no");
    response.setHeader("Cache-Control", "no-cache");

    SseEmitter emitter = new SseEmitter(180_000L);
    // ...
    return emitter;
}
```

> `proxy_buffering off` 和 `X-Accel-Buffering: no` 最好都配上。有些 Nginx 版本或配置场景下，只配一个可能不生效。双重保险总没错。

### 2. 连接数与线程池

SSE 是长连接，每个活跃的 SSE 连接在后端占用的资源包括：

- 一个 `SseEmitter` 对象（内存）
- 一个 SSE 推送线程（线程池）
- 一个到大模型 API 的 HTTP 连接（OkHttp 连接池）
- Tomcat 的一个 NIO 连接

`SseEmitter` 本身是异步的，Controller 线程立即返回，**不占 Tomcat 线程池**。但它占用 Tomcat 的 NIO 连接数。Tomcat 的 `max-connections` 默认 8192，通常够用。真正的瓶颈往往在推送线程池和大模型 API 的并发限制上。

线程池的配置建议：

| 参数 | 建议值 | 说明 |
|-----|--------|------|
| 核心线程数 | 10~20 | 覆盖正常并发量 |
| 最大线程数 | 50~100 | 覆盖峰值 |
| 队列容量 | 100~200 | 超过的请求等待或降级 |
| 拒绝策略 | `CallerRunsPolicy` | 队列满时降级为同步，不丢失请求 |

还有一个容易忽略的点：调用大模型 API 的 OkHttpClient 应该是**全局单例**，不要每次请求都 `new OkHttpClient()`。OkHttpClient 内部维护了连接池和线程池，每次 new 都会创建新的连接池，连接无法复用，资源浪费严重。

### 3. 内存泄漏防范

`SseEmitter` 对象在连接期间一直被 Spring 持有。如果连接异常断开但 `SseEmitter` 没有被 `complete()`，它会一直留在内存里直到超时。

防范措施：

1. **设置合理的超时时间**：不要设 `0`（表示永不超时）——如果客户端断开而你没感知到，这个 `SseEmitter` 就永远不会被清理。建议 3~5 分钟
2. **`onCompletion` 回调里清理资源**：停止心跳定时任务、记录日志
3. **定期巡检连接池**：虽然 `onCompletion` 回调应该能覆盖所有清理场景，但作为兜底，可以用一个定时任务每分钟检查一下连接池，清理超过超时时间还存在的连接

### 4. 负载均衡注意事项

如果你的 Spring Boot 服务做了多实例部署 + 轮询负载均衡，SSE 长连接会带来一个问题：

SSE 是一个持续的 HTTP 连接。连接建立后，所有的 `send()` 数据都必须通过**同一个后端实例**推送。如果中间负载均衡器把连接切到了另一个实例，数据就丢了。

对于 SSE 场景，有两种处理方式：

- **Sticky Session（会话粘滞）**：配置负载均衡器，让同一个客户端的请求始终打到同一个后端实例。Nginx 可以用 `ip_hash` 或 `sticky cookie` 实现
- **无状态设计**：SSE 连接本身是有状态的（绑定在某个实例上），但业务逻辑做成无状态的——对话历史存在 Redis/MySQL 而不是内存里，任何实例都能处理

实际上，对于大模型流式转发这个场景，SSE 连接只在一次对话的生成过程中存在（几秒到几十秒），生成完就断开了。只要负载均衡器不在一次 SSE 连接中间切换后端实例（Nginx 的默认行为不会这么做），就不需要特殊配置。

## 小结

两篇 SSE 文章合在一起，覆盖了 SSE 从协议理解到生产部署的完整链路。

**第 1 篇《SSE 协议与流式响应》**回顾：
1. SSE 协议完整规范：四个字段（`data:`、`event:`、`id:`、`retry:`）加注释心跳机制
2. 为什么大模型选 SSE：单向推送匹配、HTTP 原生、实现简单
3. 流式响应数据结构：`delta` 增量解析、`finish_reason` 值的含义、Token 统计的三种获取方式
4. 健壮的 SSE 客户端：回调接口、超时控制（30~60 秒）、JSON 解析容错、空 delta 兼容、错误处理不丢内容

**第 2 篇《Spring Boot SSE 服务端实战》**（本篇）：
1. 后端中间层的必要性：API Key 安全、跨域限制、业务逻辑
2. `SseEmitter` 核心机制：异步推送、线程模型（Controller 线程立即释放，推送在独立线程）、生命周期回调
3. 三个层次的实现：最简版（理解 SseEmitter 基本套路）→ 完整版（接入大模型流式转发）→ 生产版（心跳保活、断连检测、连接池管理、线程池配置）
4. 事件类型设计：`token`（内容增量）、`done`（流结束+统计）、`error`（错误通知）
5. 前端对接：`EventSource` 基本用法和 `fetch` + `ReadableStream` 替代方案
6. Nginx 配置要点：**关闭 `proxy_buffering` 是关键**，不关就没有实时推送效果
7. 生产环境关注点：连接数控制、线程池配置、内存泄漏防范、负载均衡注意事项
