<p align="center">
  <a href="https://github.com/nageoffer/awesome-ai-handbook">
    <picture>
      <source srcset="assets/ai-handbook-banner.png">
      <img src="assets/ai-handbook-banner.png" alt="Awesome AI Handbook">
    </picture>
  </a>
</p>

<p align="center">
  <strong>程序员转型 AI 工程师的全栈知识手册</strong><br/>
</p>

<p align="center">
  <a href="https://github.com/nageoffer/awesome-ai-handbook/stargazers"><img alt="GitHub stars" src="https://img.shields.io/github/stars/nageoffer/awesome-ai-handbook?style=social" /></a>
  <a href="https://github.com/nageoffer/awesome-ai-handbook/network/members"><img alt="GitHub forks" src="https://img.shields.io/github/forks/nageoffer/awesome-ai-handbook?style=social" /></a>
  <a href="./LICENSE"><img alt="License: Apache 2.0" src="https://img.shields.io/badge/License-Apache%202.0-orange.svg?style=flat-square" /></a>
  <a href="https://nageoffer.com/ai"><img alt="Online Docs" src="https://img.shields.io/badge/在线阅读-nageoffer.com-blue.svg?style=flat-square" /></a>
</p>

## 💡 为什么做这个项目

大模型时代已经到来，越来越多的程序员希望转型 AI 方向，但面临几个痛点：

- **知识碎片化**：不知道从哪学起，找不到系统的学习路径。
- **理论和实战脱节**：看了很多论文但不会落地，缺少工程化实践。
- **面试无从准备**：不知道 AI 岗位考什么，缺少体系化的备战资料。

这个仓库就是为了解决这些问题 —— **一份持续更新的 AI 工程师成长手册**，系统梳理 AI 基础、LLM、RAG、Agent 等核心知识，助你从入门到转型 AI 工程师。

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

| 章节 | 关键内容 | 状态 |
|:-----|:-----|:----:|
| **第一部分：大模型基础** | | |
| [01、大模型到底是什么？核心概念讲透](./docs/rag/llm-basics/understanding-llm.md) | Token、参数量、温度等核心术语全解析 | ✅ |
| [02、手把手调用大模型 API 全流程](./docs/rag/llm-basics/calling-llm-api.md) | Chat Completions 协议，流式与非流式实现 | ✅ |
| [03、好 Prompt vs 烂 Prompt 写法对比](./docs/rag/llm-basics/prompt-engineering.md) | 对比差异写法，让模型精准基于内容作答 | ✅ |
| **第二部分：RAG 核心链路** | | |
| [04、一张图看懂 RAG 全链路架构](./docs/rag/rag-core/rag-overview.md) | 全链路架构总览与企业实战项目背景 | ✅ |
| [05、文档解析没你想的那么简单](./docs/rag/rag-core/document-parsing-tika.md) | Apache Tika 解析 PDF、扫描件等多格式 | ✅ |
| [06、Chunk 分块策略决定检索质量](./docs/rag/rag-core/chunking-strategies.md) | 上下文窗口限制下的分块与精度优化 | ✅ |
| [07、别忽视元数据：答案要能追根溯源](./docs/rag/rag-core/metadata-management.md) | 为 Chunk 附加来源、权限等追溯信息 | ✅ |
| [08、Embedding 如何跨越语义鸿沟](./docs/rag/rag-core/understanding-embedding.md) | 语义向量化原理，解决同义词匹配不到的痛点 | ✅ |
| **第三部分：向量检索与生成** | | |
| [09、向量数据库：原理讲透再做选型](./docs/rag/vector-search-and-generation/vector-database-principles.md) | 核心原理与 Milvus、Qdrant 等产品对比 | ✅ |
| [10、用 Milvus 构建向量检索实战](./docs/rag/vector-search-and-generation/milvus-vector-search.md) | 从建库、写入到语义检索完整流程 | ✅ |
| [11、混合检索 + 重排序提升召回率](./docs/rag/vector-search-and-generation/retrieval-and-reranking.md) | 向量 + 关键词混合检索与 Reranking 策略 | ✅ |
| [12、大模型总爱编答案？幻觉抑制攻略](./docs/rag/vector-search-and-generation/generation-and-hallucination.md) | Prompt 设计与生成控制，防止知识覆盖 | ✅ |
| **第四部分：工具调用与 MCP 协议** | | |
| [13、Function Call 让模型从聊天到干活](./docs/rag/tool-calling-and-mcp/function-call.md) | 从查知识库扩展到调接口、查实时数据 | ✅ |
| [14、MCP 协议：AI 世界的 USB 接口](./docs/rag/tool-calling-and-mcp/mcp-protocol.md) | 解决工具规模化管理，标准化注册调用 | ✅ |
| [15、MCP 的 Resources 与 Prompts 详解](./docs/rag/tool-calling-and-mcp/mcp-resources-and-prompts.md) | 只读数据源与 Prompt 模板复用机制 | ✅ |
| [16、JSON-RPC 2.0：MCP 的通信底座](./docs/rag/tool-calling-and-mcp/mcp-json-rpc.md) | MCP 中消息封装规范与结构说明 | ✅ |
| [17、MCP 为什么不用 HTTP 或 gRPC？](./docs/rag/tool-calling-and-mcp/why-not-http-or-grpc.md) | 三种协议定位差异与选型理由对比 | ✅ |
| [18、MCP Java SDK 源码深度拆解](./docs/rag/tool-calling-and-mcp/mcp-java-sdk-deep-dive.md) | 工具发现、路由与 Schema 生成机制 | ✅ |
| [19、工具调用架构：从能用到好用](./docs/rag/tool-calling-and-mcp/tool-calling-architecture.md) | 职责划分、描述规范与错误处理原则 | ✅ |
| [20、工具调用上线翻车？四层防护兜底](./docs/rag/tool-calling-and-mcp/tool-calling-stability-security.md) | 超时重试、熔断降级、权限与可观测性 | ✅ |
| **第五部分：高级检索与对话** | | |
| [21、多轮对话为什么会失忆？记忆设计](./docs/rag/advanced-retrieval-and-conversation/conversation-memory.md) | 对话历史注入 messages 解决上下文丢失 | ✅ |
| [22、用户问得模糊？查询重写来兜底](./docs/rag/advanced-retrieval-and-conversation/query-rewriting.md) | 模糊追问扩展为完整检索语句 | ✅ |
| [23、意图识别：消息进来该走哪条路](./docs/rag/advanced-retrieval-and-conversation/intent-routing.md) | 闲聊、检索、工具调用、反问四路分发 | ✅ |
| [24、RAG 效果评估：别靠感觉用数据](./docs/rag/advanced-retrieval-and-conversation/rag-evaluation.md) | 量化指标替代主观判断与回归检测 | ✅ |
| **第六部分：流式通信** | | |
| [25、SSE：大模型为什么都用它做流式](./docs/rag/streaming/sse-protocol.md) | 单向推送规范、生命周期与异常处理 | ✅ |
| [26、Spring Boot + SSE 流式接口实战](./docs/rag/streaming/springboot-sse.md) | 手写全链路流式转发服务端实现 | ✅ |

### Agent 系列 · 从 RAG 到自主智能体

| 章节                                                                                                      | 关键内容 | 状态 |
|:--------------------------------------------------------------------------------------------------------|:-----|:----:|
| **第一部分：从 RAG 到 Agent**                                                                                  | | |
| [01、RAG 和 Agent 的边界到底在哪？](./docs/agent/from-rag-to-agent/01-rag-vs-agent.md)                            | 被动检索与主动决策的区别，理解 Agent 循环价值 | ✅ |
| [02、Agent 架构全景：大脑、工具、记忆与规划](./docs/agent/from-rag-to-agent/02-agent-architecture.md)                    | LLM、工具、记忆、规划四要素协作机制 | ✅ |
| [03、比特严选智能体：从蓝图开始设计](./docs/agent/from-rag-to-agent/03-bitmall-agent-intro.md)                          | 电商客服场景拆解、任务分级与 MVP 范围 | ✅ |
| **第二部分：ReAct 核心**                                                                                       | | |
| [04、ReAct 是什么？推理与行动交替运转](./docs/agent/react-core/04-what-is-react.md)                                   | Thought、Action、Observation 三元组与完整轨迹 | ✅ |
| [05、用纯 Java 手写最小 ReAct Agent](./docs/agent/react-core/05-react-loop.md)                                 | OkHttp 调用大模型，串起工具注册表和主循环 | ✅ |
| [06、工具定义别靠猜：用 JSON Schema 约束入参](./docs/agent/react-core/06-tool-definition.md)                          | 结构化描述工具参数，提升调用稳定性 | ✅ |
| [07、Prompt 是 Agent 的灵魂：ReAct 提示词设计](./docs/agent/react-core/07-react-prompt-design.md)                  | 结构化提示词、Few-shot 示例与负面约束 | ✅ |
| [08、使用 OpenAI Function Calling 原生工具](./docs/agent/react-core/08-function-calling.md)                    | 模型直接返回结构化数据，替代文本解析 | ✅ |
| [09、多维度终止控制：让 Agent 运行更安全](./docs/agent/react-core/09-termination-control.md)                           | 四道防线解决死循环、空转与 Token 超预算 | ✅ |
| **第三部分：记忆与上下文**                                                                                         | | |
| [10、Agent 为什么需要记忆？](./docs/agent/memory-and-context/10-why-memory.md)                                   | 多轮对话上下文丢失问题与记忆机制引入 | ✅ |
| [11、记忆管理三板斧：窗口、摘要与混合策略](./docs/agent/memory-and-context/11-memory-module.md)                            | 滑动窗口、摘要压缩、混合策略控制记忆预算 | ✅ |
| [12、会话记忆持久化：用数据库存住每一轮对话](./docs/agent/memory-and-context/12-persistent-memory.md)                       | 用户与会话隔离，记忆重启不丢、多实例共享 | ✅ |
| [13、长期记忆：让 Agent 跨会话认识用户](./docs/agent/memory-and-context/13-long-term-memory.md)                       | 用户画像、交互记录与相关性检索注入 | ✅ |
| [14、上下文工程：每一个 Token 都花在刀刃上](./docs/agent/memory-and-context/14-context-engineering.md)                  | 预算分配、工具裁剪与 Observation 折叠 | ✅ |
| **第四部分：规划与编排**                                                                                          | | |
| [15、Plan-and-Execute：先规划再执行](./docs/agent/planning-and-orchestration/15-plan-and-execute.md)            | 复杂任务拆解、动态重规划与模式路由 | ✅ |
| [16、Skill 是什么？从规范到框架实现](./docs/agent/planning-and-orchestration/16-skill-design.md)                     | SKILL.md 标准、渐进式披露与主流框架模式 | ✅ |
| [17、用 TinyAgent 从零实现 Skill 机制](./docs/agent/planning-and-orchestration/17-skill-implementation.md)      | 技能解析、工具隔离与动态注入完整实现 | ✅ |
| [18、跨品类编排：对比、搭配与售后一起跑](./docs/agent/planning-and-orchestration/18-cross-category-orchestration.md)      | Tool、Skill 与 Plan-and-Execute 组合编排实战 | ✅ |
| [19、Reflection：让 Agent 学会自我纠错](./docs/agent/planning-and-orchestration/19-reflection-error-handling.md) | 三级反思结论与重试、重规划错误恢复机制 | ✅ |
| [20、RAG 作为 Tool：让 Agent 自主决定何时检索](./docs/agent/planning-and-orchestration/20-rag-as-tool.md)            | pgvector 向量检索、知识导入与三道工具决策防线 | ✅ |
| **第五部分：多智能体**                                                                                           | | |
| [21、多智能体架构全景：从单体到专家团队](./docs/agent/multi-agent/21-multi-agent-landscape.md)                            | Agent 四要素、协作范式与任务驱动的架构选型 | ✅ |
| [22、手写主从式多智能体：主 Agent 编排子 Agent](./docs/agent/multi-agent/22-multi-agent-supervisor.md)                 | SpecialistAgent、子 Agent 即工具与 LLM Supervisor 编排 | ✅ |
| [23、主从式的上下文与通信：消除上下文割裂](./docs/agent/multi-agent/23-multi-agent-communication.md)                       | AgenticScope、MessageHub、结果聚合与依赖 Handoff | ✅ |

## 💼 AI Agent 面试题库

精选 AI Agent 领域高频面试题，覆盖 Agent、RAG、LLM、MCP 等核心方向，数据来源于牛客、小红书、脉脉等多平台。

| 模块 | 说明 | 状态 |
|:-----|:-----|:----:|
| [Top50 必刷题](./docs/interview/top50.md) | 按出现频次×难度加权排序的 50 道高频核心题 | ✅ |
| [按分类浏览](./docs/interview/categories.md) | Agent / RAG / Prompt / MCP 等分类维度 | 🔲 |
| [按公司浏览](./docs/interview/companies.md) | 字节、阿里、腾讯、百度等公司维度 | 🔲 |
| [趋势洞察](./docs/interview/insights.md) | 各分类题目数量分布与趋势变化 | 🔲 |

## 📁 项目结构

```
awesome-ai-handbook/
├── docs/
│   ├── agent/                                    # 🤖 Agent 体系教学
│   │   ├── from-rag-to-agent/                    #     从 RAG 到 Agent
│   │   ├── react-core/                           #     ReAct 核心
│   │   ├── memory-and-context/                   #     记忆与上下文
│   │   ├── planning-and-orchestration/           #     规划与编排
│   │   └── multi-agent/                          #     多智能体
│   ├── growth/                                   # 🧭 成长路径
│   ├── interview/                                # 💼 AI Agent 面试题库
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

AI HandBook 开源小册仍在持续更新之中，欢迎您参与本项目，一同为读者提供更优质的学习内容。

- **📌内容修正**：请您协助修正或在评论区指出语法错误、内容缺失、文字歧义、无效链接或代码 bug 等问题。
- **📝补充新内容**：期待您贡献 RAG、Agent、LLM 等方向的原创教程或实践总结。
- **🔗推荐优质资源**：欢迎推荐高质量的书单、课程、博客与开源项目。

欢迎您提出宝贵意见和建议，如有任何问题请提交 [Issues](https://github.com/nageoffer/awesome-ai-handbook/issues)。

感谢以下各位为此作出贡献的伙伴：

<p align="left">
    <a href="https://github.com/nageoffer/awesome-ai-handbook/graphs/contributors">
        <img src="https://contrib.rocks/image?repo=nageoffer/awesome-ai-handbook&columns=8" />
    </a>
</p>

## ⭐ Star History

如果觉得有帮助，请点个 Star 支持一下！

[![Star History Chart](https://api.star-history.com/svg?repos=nageoffer/awesome-ai-handbook&type=Date)](https://star-history.com/#nageoffer/awesome-ai-handbook&Date)

## 📜 License

该项目遵循的是 Apache 2.0 许可协议——详情请参阅 [LICENSE](./LICENSE) 文件。

> 🔔 **持续更新中，建议 Star + Watch 追踪最新内容！**
