<div align="center">

# 🤖 Awesome AI Handbook

**程序员转型 AI 工程师的全栈知识手册**

系统梳理 AI 基础、LLM、RAG、Agent 等核心知识，助你从入门到拿下 AI Offer。

[![GitHub stars](https://img.shields.io/github/stars/nageoffer/awesome-ai-handbook?style=social)](https://github.com/nageoffer/awesome-ai-handbook)
[![License: Apache2.0](https://img.shields.io/badge/License-Apache2.0-yellow.svg)](https://opensource.org/licenses/Apache-2.0)

[在线阅读](https://nageoffer.com/ai) •
[入门导航](#-入门导航) •
[体系教学](#-体系教学) •
[大厂面经](#-大厂面经) •
[面试问答](#-面试问答)

</div>

## 💡 为什么做这个项目

大模型时代已经到来，越来越多的程序员希望转型 AI 方向，但面临几个痛点：

- 🤯 知识碎片化，不知道从哪学起
- 📚 理论和实战脱节，看了很多论文但不会落地
- 😰 面试不知道考什么，缺少体系化的准备资料

这个仓库就是为了解决这些问题 —— **一份持续更新的 AI 工程师成长手册**。

## 🧭 入门导航

刚接触 AI？从这里开始，快速建立全局认知。

| 主题 | 说明                                 | 状态 |
|:-----|:-----------------------------------|:----:|
| [程序员如何成长为 AI 工程师](./docs/growth/01-how-to-become-ai-engineer.md) | AI 工程师定位、学习路线与技术栈全景                | ✅ |
| AI 学习资料推荐 | 精选书单、课程、博客与开源项目                    | 🔲 |
| AI 常见名词速查表 | RAG、Agent、MCP、Skills、CLI 等核心术语一网打尽 | 🔲 |

## 📖 体系教学

按方向系统学习，每个方向从基础到进阶完整覆盖。

### RAG 系列 · 从零构建检索增强生成系统

| 章节                                                                                                  | 关键内容 | 状态 |
|:----------------------------------------------------------------------------------------------------|:-----|:----:|
| **第一部分：大模型基础**                                                                                      | | |
| [01、大模型到底是什么？一文讲透核心概念](./docs/rag/llm-basics/understanding-llm.md)                                  | Token、参数量、温度等核心术语，建立对 LLM 的第一认知 | ✅ |
| [02、手把手调用大模型 API：从第一次请求到流式输出](./docs/rag/llm-basics/calling-llm-api.md)                             | OpenAI Chat Completions 协议详解，Java 实现流式与非流式调用 | ✅ |
| [03、好 Prompt vs 烂 Prompt：让大模型听话的关键技巧](./docs/rag/llm-basics/prompt-engineering.md)                  | 对比好坏 Prompt 写法，让模型精准基于检索内容作答 | ✅ |
| **第二部分：RAG 核心链路**                                                                                   | | |
| [04、一张图看懂 RAG 全链路架构](./docs/rag/rag-core/rag-overview.md)                                           | RAG 全链路架构总览，Ragent 企业级实战项目背景与目标 | ✅ |
| [05、文档解析没你想的那么简单：Apache Tika 实战](./docs/rag/rag-core/document-parsing-tika.md)                      | PDF、扫描件、Word 等多格式内容提取的复杂性与解决方案 | ✅ |
| [06、数据分块的艺术：Chunk 策略决定检索质量](./docs/rag/rag-core/chunking-strategies.md)                             | 文档分块的必要性，上下文窗口限制与检索精度优化 | ✅ |
| [07、别忽视元数据：让每条 Chunk 都能追根溯源](./docs/rag/rag-core/metadata-management.md)                            | 为 Chunk 附加来源、权限、位置等元数据，保障答案可追溯 | ✅ |
| [08、从文本到向量：Embedding 如何跨越语义鸿沟](./docs/rag/rag-core/understanding-embedding.md)                      | 语义向量化原理，解决关键词检索匹配不到同义表达的痛点 | ✅ |
| **第三部分：向量检索与生成**                                                                                    | | |
| [09、向量数据库选型指南：原理讲透再做选择](./docs/rag/vector-search-and-generation/vector-database-principles.md)      | 向量数据库的必要性、核心原理与主流产品选型对比 | ✅ |
| [10、用 Milvus 构建向量检索实战](./docs/rag/vector-search-and-generation/milvus-vector-search.md)             | Milvus 向量检索实践 | ✅ |
| [11、混合检索 + 重排序：召回率翻倍的秘密武器](./docs/rag/vector-search-and-generation/retrieval-and-reranking.md)      | 向量 + 关键词混合检索与 Reranking 策略，提升召回精度 | ✅ |
| [12、大模型总爱“编”答案？幻觉抑制全攻略](./docs/rag/vector-search-and-generation/generation-and-hallucination.md)    | Prompt 设计与生成控制，防止模型用预训练知识覆盖检索结果 | ✅ |
| **第四部分：工具调用与 MCP 协议**                                                                               | | |
| [13、Function Call：让大模型从只会聊天到能干活](./docs/rag/tool-calling-and-mcp/function-call.md)                  | Function Call 机制，从查知识库扩展到调接口、查实时数据 | ✅ |
| [14、MCP 协议：AI 世界的“USB 接口”标准](./docs/rag/tool-calling-and-mcp/mcp-protocol.md)                       | MCP 协议解决工具规模化管理痛点，标准化注册与调用 | ✅ |
| [15、MCP 的 Resources 与 Prompts 到底怎么用？](./docs/rag/tool-calling-and-mcp/mcp-resources-and-prompts.md) | Resources（只读数据源）与 Prompts（模板复用）深入解析 | ✅ |
| [16、JSON-RPC 2.0：MCP 通信的底层契约](./docs/rag/tool-calling-and-mcp/mcp-json-rpc.md)                      | JSON-RPC 2.0 在 MCP 中的消息封装规范与结构说明 | ✅ |
| [17、MCP 为什么不用 HTTP 或 gRPC？一次讲清楚](./docs/rag/tool-calling-and-mcp/why-not-http-or-grpc.md)           | 对比 HTTP、gRPC 与 JSON-RPC 2.0 的定位差异与选型理由 | ✅ |
| [18、MCP 官方 Java SDK 源码拆解：从注解到工具发现](./docs/rag/tool-calling-and-mcp/mcp-java-sdk-deep-dive.md)       | 工具发现、路由、JSON Schema 生成机制源码级拆解 | ✅ |
| [19、工具调用架构设计：从“能用”到“好用”](./docs/rag/tool-calling-and-mcp/tool-calling-architecture.md)              | 单一职责、描述清晰、参数约束与错误处理设计原则 | ✅ |
| [20、工具调用上线就翻车？稳定性与安全四层防护](./docs/rag/tool-calling-and-mcp/tool-calling-stability-security.md)       | 超时重试、熔断降级、权限防注入、可观测性全方位保障 | ✅ |
| **第五部分：高级检索与对话**                                                                                    | | |
| [21、多轮对话为什么会“失忆”？记忆机制设计详解](./docs/rag/advanced-retrieval-and-conversation/conversation-memory.md)   | 会话记忆实现，将对话历史注入 messages 解决上下文丢失 | ✅ |
| [22、用户问得模糊怎么办？查询重写与语义增强](./docs/rag/advanced-retrieval-and-conversation/query-rewriting.md)         | Query 改写，将模糊追问扩展为完整检索语句提升召回精度 | ✅ |
| [23、一条消息进来，系统怎么知道该走哪条路？](./docs/rag/advanced-retrieval-and-conversation/intent-routing.md)          | 意图分类与路由：闲聊、RAG 检索、工具调用、反问四类路径 | ✅ |
| [24、RAG 效果好不好？别靠感觉，用数据说话](./docs/rag/advanced-retrieval-and-conversation/rag-evaluation.md)         | 系统化评估方法，用量化指标替代主观判断，解决回归检测 | ✅ |
| **第六部分：流式通信**                                                                                       | | |
| [25、SSE 协议详解：为什么大模型都用它做流式输出？](./docs/rag/streaming/sse-protocol.md)                                 | SSE 完整规范，单向服务端推送、连接生命周期与异常处理 | ✅ |
| [26、Spring Boot + SSE：手写一个流式对话接口](./docs/rag/streaming/springboot-sse.md)                           | Spring Boot 实现 SSE，构建全链路流式转发实战 | ✅ |

### Agent 系列 · 敬请期待

> 🚧 正在规划中，将覆盖 Agent 架构、多智能体协作、记忆与规划等核心主题。

## 💼 大厂面经

真实 AI 岗位面试记录，按公司分类，还原面试现场。

> 🚧 内容持续收录中，欢迎提交你的面经 PR！

## ❓ 面试问答

从面经中提炼高频问题，逐题深度解答，助你建立答题框架。

> 🚧 内容整理中，即将上线。

## 📁 项目结构

```
awesome-ai-handbook/
├── docs/
│   ├── growth/                                   # 🧭 成长路径
│   └── rag/                                      # 📖 RAG 体系教学
│       ├── llm-basics/                           #     大模型基础
│       ├── rag-core/                             #     RAG 核心链路
│       ├── vector-search-and-generation/         #     向量检索与生成
│       ├── tool-calling-and-mcp/                 #     工具调用与 MCP 协议
│       ├── advanced-retrieval-and-conversation/  #     高级检索与对话
│       └── streaming/                            #     流式通信
├── assets/                                       # 图片等资源
├── LICENSE
└── README.md
```

## 🤝 贡献

欢迎提交 PR 或 Issue！无论是：

- 📝 纠正错误
- 💡 补充新内容
- 🔗 推荐优质资源
- ❓ 提出你想了解的主题

都非常欢迎！

## ⭐ Star History

如果觉得有帮助，请点个 Star 支持一下！

[![Star History Chart](https://api.star-history.com/svg?repos=nageoffer/awesome-ai-handbook&type=Date)](https://star-history.com/#nageoffer/awesome-ai-handbook&Date)

## 📜 License

[Apache 2.0](./LICENSE) © Ding.ma

> 🔔 **持续更新中，建议 Star + Watch 追踪最新内容！**
