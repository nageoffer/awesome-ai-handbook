上一篇把意图识别和问题路由讲完之后，RAG 系统的各个环节已经全部串起来了：数据入库 → 向量化 → 检索 → 生成 → 工具调用 → 会话记忆 → Query 改写 → 意图识别。系统跑起来了，用户在用了，看着也挺好的。

但有个问题一直没回答——**这个系统到底好不好？**

你上周把 chunk size 从 512 改成了 1024，改完之后抽了 10 个问题问了一下，感觉回答质量还行。但还行到底有多好？有没有其他问题因为这次修改变差了？你不知道。

你前天换了一个新的 Embedding 模型，换完之后试了几个问题，检索结果看着不错。但看着不错和真的不错之间差着一个评估体系。

你昨天调了 Prompt，加了一条“请基于检索到的内容回答，不要编造信息”。加完之后幻觉是不是真的减少了？减少了多少？你还是不知道。

没有评估，优化就是盲人摸象。改了 Prompt 不知道效果变好还是变差，换了模型不知道值不值，上线新功能不知道有没有引入回归。这就是为什么需要一套系统化的评估方法——用数据说话，而不是靠感觉。

## 为什么需要系统化评估

### 1. 靠感觉优化的困境

假设你在做电商客服 RAG 系统，上线一个月了，老板问你：“系统效果怎么样？”你说：“还行，用户反馈还可以。”老板追问：“具体多少准确率？”你答不上来。

这不是因为你不想量化，而是你没有一套可以量化的方法。日常优化全靠感觉：

- 你把 chunk size 从 512 改成了 1024，抽查了 10 个问题，8 个回答看着还行。但你没法确定这 10 个问题是否有代表性。也许有 50 个其他问题因为 chunk 变大了，关键信息被稀释了，检索效果反而变差了——你不知道，因为你没有测到那些问题。
- 你换了一个 Embedding 模型，试了退货政策和保修期两个问题，检索到了正确的 chunk。但运费承担、质量问题退换这些问题有没有变差？你没有测过。
- 你调了 Prompt，加了限定知识来源的指令，感觉幻觉少了。但到底少了多少？是从 30% 降到了 10%，还是从 30% 降到了 28%？你量化不出来。

10 个样本能说明什么？100 个问题里答对 60 个和答对 90 个，在 10 个样本上可能表现完全一样。

更麻烦的是**回归问题**：改了 A 环节的东西，B 环节可能受影响。你优化了检索参数让某类问题的召回率提高了，但另一类问题的召回率可能下降了。没有全面的评测，你根本发现不了这种按下葫芦浮起瓢的问题。

### 2. 分层评估的思路

RAG 系统是一个多环节的流水线，如果只看最终结果——用户的问题有没有被正确回答——你知道答错了，但不知道错在哪个环节。

打个比方，就像工厂的质检。一个产品从流水线下来不合格，你不能只说产品坏了就完事了。你得搞清楚是原材料有问题（检索到的 chunk 不对），还是加工工序出了问题（模型基于正确的 chunk 生成了错误的答案），还是设计本身就有缺陷（知识库里根本没有这个信息）。

RAG 系统的评估也一样，要**分层**：

![](https://oss.open8gu.com/iShot_2026-03-06_10.13.30.png)

三层评估的核心逻辑：

- **检索阶段**：召回的 chunk 对不对？正确答案有没有在召回的 Top-K 里？排在第几位？——这层出了问题，后面全白搭，模型再厉害也没法基于错误的 chunk 生成正确的答案。
- **生成阶段**：给了正确的 chunk，模型有没有忠实地基于 chunk 内容回答？有没有编出 chunk 里没有的信息（幻觉）？有没有答非所问？——这层出了问题，说明 Prompt 或模型需要调整。
- **端到端**：不管中间过程，最终答案正确吗？用户满意吗？——这是最终的结果指标，但光看这个定位不了具体问题。

分层评估的好处是**定位精准**：检索指标好但生成指标差，说明问题在 Prompt 或模型上；检索指标差，那就先去优化检索环节，调 Prompt 没有用。

## 检索阶段的评估指标

检索阶段是 RAG 系统的基础，chunk 都没召回来，后面的生成再好也是空中楼阁。检索阶段的评估核心问题就一个：**Top-K 个召回的 chunk 里，有没有包含正确答案对应的 chunk？**

围绕这个问题，有四个常用指标。

### 1. 命中率（Hit Rate）

最简单的指标：Top-K 个召回的 chunk 里，有没有包含正确答案？有就是 1，没有就是 0。多个问题取平均值。

用电商客服的例子来算。假设评测集有 5 个问题，每个问题的 Top-3 召回结果：

| 问题                       | Top-3 召回的 chunk           | 正确 chunk 有没有在里面         | 命中 |
| -------------------------- | ---------------------------- | ------------------------------- | ---- |
| iPhone 16 Pro 的退货政策？ | chunk_12, chunk_05, chunk_33 | chunk_12 是正确答案 ✓           | 1    |
| AirPods Pro 的保修期？     | chunk_21, chunk_07, chunk_44 | chunk_21 是正确答案 ✓           | 1    |
| 退货运费谁承担？           | chunk_18, chunk_29, chunk_55 | 正确答案是 chunk_41，不在里面 ✗ | 0    |
| 跨境商品能退吗？           | chunk_41, chunk_03, chunk_67 | chunk_03 是正确答案 ✓           | 1    |
| 质量问题怎么换货？         | chunk_08, chunk_15, chunk_22 | chunk_08 是正确答案 ✓           | 1    |

**Hit Rate = 命中次数 / 总问题数 = 4 / 5 = 0.8（80%）**

命中率很好理解，但它有一个明显的缺点——**不关心正确答案排在第几位**。chunk_12 排在第 1 位和排在第 3 位，命中率上没有区别，都算命中。但实际使用中，排在第 1 位和排在第 3 位的差别很大——排在第 1 位意味着检索系统很自信地找到了正确答案，排在第 3 位可能只是凑巧混进来了。

### 2. MRR（Mean Reciprocal Rank，平均倒数排名）

MRR 在命中率的基础上，还关心**正确答案排在第几位**。

计算规则：如果正确答案排在第 1 位，得 1 分；排在第 2 位，得 1/2 = 0.5 分；排在第 3 位，得 1/3 ≈ 0.33 分……排在第 K 位，得 1/K 分。如果 Top-K 里没有正确答案，得 0 分。多个问题取平均。

还是用刚才的 5 个问题，这次多关注一下正确答案的排名位置：

| 问题                       | 正确答案排在第几位 | 倒数排名（Reciprocal Rank） |
| -------------------------- | ------------------ | --------------------------- |
| iPhone 16 Pro 的退货政策？ | 第 1 位            | 1/1 = 1.0                   |
| AirPods Pro 的保修期？     | 第 1 位            | 1/1 = 1.0                   |
| 退货运费谁承担？           | 没命中             | 0                           |
| 跨境商品能退吗？           | 第 2 位            | 1/2 = 0.5                   |
| 质量问题怎么换货？         | 第 1 位            | 1/1 = 1.0                   |

**MRR = (1.0 + 1.0 + 0 + 0.5 + 1.0) / 5 = 3.5 / 5 = 0.7**

MRR 比 Hit Rate 更能反映检索质量。两个系统 Hit Rate 都是 80%，但一个系统的正确答案大部分排在第 1 位（MRR 接近 0.8），另一个系统的正确答案大部分排在第 3 位（MRR 可能只有 0.4）——后者的检索质量明显不如前者，因为排在第 3 位的 chunk 在实际使用中更容易被忽略或被不相关的 chunk 干扰模型生成。

> MRR 的直觉理解：MRR = 0.7 意味着平均来看，正确答案大约排在第 1.4 位（1 / 0.7 ≈ 1.43）。MRR 越接近 1，说明正确答案越稳定地排在第 1 位。

### 3. 召回率与精确率（Recall & Precision）

命中率和 MRR 都是围绕**一个正确答案**来评估的。但现实中，一个问题的完整答案往往分散在**多个 chunk** 里。

比如用户问退货政策是什么，完整答案涉及三个 chunk：

- chunk_12：退货条件（7 天内、未拆封）
- chunk_13：退货流程（申请 → 审核 → 寄回 → 退款）
- chunk_14：退货运费（质量问题免运费，其他自付）

只命中其中一个，用户拿到的答案就是残缺的。这时候就需要召回率和精确率来衡量找全了没有和找准了没有。

继续用这个例子。系统 Top-5 召回了这些 chunk：

| 排名 | 召回的 chunk             | 是否相关 |
| ---- | ------------------------ | -------- |
| 1    | chunk_12（退货条件）     | 相关     |
| 2    | chunk_05（会员等级说明） | 不相关   |
| 3    | chunk_13（退货流程）     | 相关     |
| 4    | chunk_33（促销活动规则） | 不相关   |
| 5    | chunk_67（配送时效说明） | 不相关   |

**召回率（Recall）**：该找的 chunk，找到了几个？分母是应该找到的总数，分子是实际命中的个数。
$$
Recall = \frac{命中的相关\ chunk\ 数}{标注的相关\ chunk\ 总数} = \frac{2}{3} = 66.7\%
$$

3 个相关 chunk 里命中了 2 个（chunk_12 和 chunk_13），漏掉了 chunk_14（退货运费）。用户问退货政策，结果运费相关的信息丢了。

**精确率（Precision）**：找回来的 chunk 里，有几个是真正有用的？分母是召回的总数，分子是其中相关的个数。
$$
Precision = \frac{命中的相关\ chunk\ 数}{召回的\ chunk\ 总数} = \frac{2}{5} = 40\%
$$

召回了 5 个 chunk，但只有 2 个是相关的，另外 3 个（会员等级、促销活动、配送时效）都是噪音。这些不相关的 chunk 会干扰模型生成，还浪费 Token。

这两个指标往往是跷跷板关系：

- **召回率高但精确率低**——Top-K 设得很大，找了一大堆回来，正确 chunk 确实都包含了，但噪音也很多。多余的 chunk 会干扰模型生成，增加 Token 消耗。
- **精确率高但召回率低**——Top-K 设得很小，找回来的每个 chunk 都很相关，但遗漏了部分正确 chunk。答案可能不完整。

RAG 场景通常**更关注召回率**——宁可多召回几个不太相关的 chunk（后面可以用 Reranker 过滤掉噪音），也不要漏掉正确答案。因为漏掉了就彻底没了，模型不可能基于没看到的信息回答正确。

> 实际项目中，Hit Rate 和 MRR 更常用，因为它们计算简单，评测集标注也简单——每个问题只需要标注正确答案对应哪个 chunk ID 就行。Recall 和 Precision 需要标注所有相关的 chunk，标注成本更高。如果你的评测集每个问题只标了一个正确 chunk ID，那就用 Hit Rate 和 MRR；如果标了多个相关 chunk ID，可以再看 Recall 和 Precision。严格来说，Hit Rate 和 MRR 也可以标多个正确 chunk ID（命中其中任意一个就算 hit），但它们的核心关注点始终是有没有命中这一个判断，不需要像 Recall 那样算命中了几个占总共几个，所以标注工作量确实小得多。

### 4. 检索指标对比

| 指标      | 定义                        | 关注点     | 标注要求                  | 推荐场景                    | 参考阈值 |
| --------- | --------------------------- | ---------- | ------------------------- | --------------------------- | -------- |
| Hit Rate  | Top-K 里有没有正确答案      | 有没有命中 | 每个问题标 1 个正确 chunk | 快速验证检索基本能力        | ≥ 0.85   |
| MRR       | 正确答案排在第几位          | 命中位置   | 每个问题标 1 个正确 chunk | 关注排序质量                | ≥ 0.70   |
| Recall    | 相关 chunk 被找到了多少     | 覆盖度     | 每个问题标所有相关 chunk  | 答案涉及多个 chunk 的场景   | ≥ 0.75   |
| Precision | 找到的 chunk 里多少是相关的 | 噪音控制   | 每个问题标所有相关 chunk  | 控制上下文质量和 Token 成本 | ≥ 0.50   |

> 参考阈值只是经验值，不是绝对标准。不同业务场景的容忍度不一样——医疗问答系统对召回率的要求远高于电商客服。先跑出一个基线数字，后续优化对比变化趋势比绝对值更重要。

## 生成阶段的评估指标

检索阶段找到了正确的 chunk，但模型有没有好好利用这些 chunk 来回答？这是生成阶段评估要回答的问题。

### 1. 忠实度（Faithfulness）

忠实度衡量的是：**模型生成的答案是否忠实于检索到的 chunk 内容，有没有编出 chunk 里没有的信息？**

看一个具体的例子。检索到的 chunk 内容是：

> iPhone 16 Pro 保修期为 1 年，自购买之日起计算。保修范围包括硬件故障、制造缺陷，不包括人为损坏、进水。

模型的回答：

> iPhone 16 Pro 保修期为 **2 年**，自购买之日起计算。保修范围包括硬件故障、制造缺陷，不包括人为损坏、进水。**如需延长保修，可购买 AppleCare+ 服务。**

这个回答有两个忠实度问题：
- 2 年——chunk 里明明写的是 1 年，模型篡改了事实
- AppleCare+ 服务——chunk 里压根没提到，模型自己编的

忠实度关注的是答案有没有超出 chunk 内容，跟答案本身对不对是两件事。这里有一个容易混淆的点：

**忠实度 ≠ 正确率**

chunk 里写的内容可能本身就是错的（比如知识库没有及时更新，保修期已经从 1 年改成了 2 年），模型忠实地转述了过时的信息，忠实度是高的，但答案是错的。这种情况说明**问题出在知识库而不是生成环节**——这恰恰体现了分层评估的价值，帮你精确定位问题出在哪一层。

在实际项目中，团队通常还会在忠实度的基础上统计**幻觉率**作为系统级的红线指标。具体做法是：用 LLM 对每条回答的忠实度打分（1~5 分，5 分表示完全忠实，1 分表示严重编造，后面 LLM-as-Judge 自动评测章节会详细讲评分方法），然后统计忠实度 ≤ 2 分的回答占总回答数的比例，就是幻觉率。

忠实度是给每条回答打分（这条答案有多忠实），幻觉率是看整体比例（100 条回答里有多少条出现了明显幻觉）。RAG 的核心价值就是基于检索到的知识回答，如果幻觉率居高不下，那 RAG 就失去了意义。一般来说，幻觉率控制在 15% 以下是一个参考基线。

### 2. 答案相关性（Answer Relevancy）

答案相关性衡量的是：**模型生成的答案是否回答了用户的问题？有没有答非所问？**

几个例子：

| 用户问题               | 模型回答                                    | 相关性                     |
| ---------------------- | ------------------------------------------- | -------------------------- |
| 退货运费谁承担？       | 退货运费由买家承担，质量问题由卖家承担      | ✓ 高——直接回答了问题       |
| 退货运费谁承担？       | 退货流程如下：1. 提交退货申请 2. 等待审核…… | △ 中——相关但没回答运费问题 |
| iPhone 16 Pro 的价格？ | 我们的退货政策非常完善……                    | ✗ 低——完全答非所问         |

答案相关性和忠实度是两个独立的维度，别搞混了。举个例子，用户问退货运费谁承担，系统检索到了一个关于退货流程的 chunk，模型忠实地转述了退货流程的每一步——忠实度很高，但压根没回答运费问题，答案相关性很低。

反过来，模型编了一句运费由卖家承担——答案相关性高（确实在回答运费问题），但忠实度为零（chunk 里没这个信息）。所以这两个指标要分开看，一个管有没有编，一个管有没有答对题。

### 3. 生成指标对比

| 指标       | 定义                      | 评估方式           | 关注点         | 参考阈值                                |
| ---------- | ------------------------- | ------------------ | -------------- | --------------------------------------- |
| 忠实度     | 答案是否忠实于 chunk 内容 | LLM 评分（1~5 分） | 有没有编造     | 平均分 ≥ 4.0，幻觉率（≤ 2 分占比）≤ 15% |
| 答案相关性 | 答案是否回答了用户的问题  | LLM 评分（1~5 分） | 有没有答非所问 | 平均分 ≥ 4.0                            |

## 端到端的评估指标

检索指标和生成指标分别看各自环节的效果，端到端指标则直接看最终结果——用户的问题有没有被正确回答。

### 1. 答案正确率

最直接的端到端指标：**最终答案是否正确回答了用户的问题？**

这个指标需要有标准答案作为对照。评判正确本身有一定主观性——模型回答的措辞跟标准答案不一样，但意思一样，算不算正确？通常采用**语义匹配**而不是字面匹配，让人工或 LLM 来判断模型答案的含义是否与标准答案一致。

### 2. 兜底率

系统回答“抱歉，找不到相关信息”的比例。在生成策略那篇里设计过兜底回答——当检索不到相关 chunk 或者模型判断无法回答时，返回兜底回复。

兜底率太高说明知识库覆盖不够，或检索效果差，大量问题找不到答案；太低也不正常，可能模型在强行回答不该回答的问题，编造答案。

参考范围：**5%~15%** 是比较合理的区间。低于 5% 要警惕是不是模型在硬答，高于 15% 要检查知识库覆盖度和检索配置。但这个范围跟业务场景强相关——如果你的知识库就是很垂直很小，只覆盖退货退款相关问题，那非退货问题触发兜底是完全正常的，兜底率高不代表系统差。反过来，如果你的知识库覆盖了所有业务场景，兜底率超过 15% 就要认真排查了。

### 3. 用户满意度

最终的北极星指标。前面所有指标都是技术指标，用户满意度才是业务指标。

可以通过两种方式收集：

- **显式反馈**：在回答后面加点赞或点踩按钮，让用户主动评价。简单直接，但参与率低——大部分用户不会主动反馈，反馈的往往是特别满意或特别不满意的，有偏差。
- **隐式反馈**：通过用户行为推断满意度。比如用户在得到回答后没有追问（可能满意了），或者用户重复问了同一个问题（说明对上次回答不满意），或者用户在对话后转了人工客服（说明 RAG 没解决问题）。

> 在实际项目中，用户满意度作为监控指标比作为评测指标更合适。因为它需要线上用户数据，没法在离线评测集上计算。离线评测主要看前面几个指标（Hit Rate、MRR、忠实度、正确率），上线后再关注用户满意度的变化趋势。

## 评测数据集的构建

评估指标再好，没有评测数据集也算不出来。评测数据集就是你的考试题库——一组预先准备好的问题和标准答案，用来系统性地检验 RAG 系统的效果。

### 1. 评测集的格式设计

一条评测数据需要包含以下字段：

```json
{
    "query": "iPhone 16 Pro 的退货政策是什么？",
    "expectedAnswer": "iPhone 16 Pro 支持 7 天无理由退货，需保持商品完好、配件齐全、包装完整。退货运费由买家承担，质量问题除外。",
    "relevantChunkIds": ["chunk_12", "chunk_13"],
    "intent": "knowledge"
}
```

各字段的作用：

- **query**：用户问题，测试时输入给 RAG 系统
- **expectedAnswer**：标准答案，用来评估模型生成的答案是否正确
- **relevantChunkIds**：正确答案对应的 chunk ID 列表，用来评估检索阶段——系统检索到的 chunk 有没有包含这些 ID
- **intent**：意图类别，确保评测覆盖不同类型的问题

其中 `relevantChunkIds` 是评测数据集的关键。没有它，你只能评端到端的正确率，没法单独评检索阶段的效果。有了它，就可以算 Hit Rate、MRR——检索到的 Top-K 里有没有包含 `relevantChunkIds` 里的 chunk。

### 2. 三种标注方式

评测数据集从哪里来？三种方式各有优劣。

#### 2.1 人工标注

让业务专家或客服人员标注：这个问题应该用哪几个 chunk 回答，标准答案是什么。

具体操作：给标注人员一份知识库 chunk 列表和一组问题，让他们标注每个问题对应的 chunk ID 和标准答案。

这是最准确的方式，标注出来的数据质量最高。缺点是**成本高、速度慢**。标注一条数据大概需要 5~10 分钟（需要在 chunk 列表里找对应的 chunk，还要写标准答案），50 条就是一两天的工作量。

标注时要注意**一致性**：如果多个人标注同一个问题，标准答案可能不一样。比如退货政策有人写得详细、有人写得简洁。需要提前约定标注规范，答案粒度保持一致。不一致的要讨论对齐。

#### 2.2 用户反馈收集

从线上真实对话中收集评测数据：

- 用户点赞的对话 → 说明回答正确 → 可以作为正向评测数据（问题 + 模型回答作为标准答案）
- 用户点踩或转人工的对话 → 说明回答不好 → 可以用来分析 bad case，但不能直接作为评测数据（因为你不知道标准答案应该是什么，需要人工补标）

优点是**数据最真实**，来自真实用户的真实问题。缺点是用户反馈有偏差——满意的用户很少会主动点赞，不满意的用户有时候也懒得点踩。能收集到的数据量通常不多。

#### 2.3 大模型辅助生成

用大模型根据知识库文档自动生成 QA 对。做法是：给模型一段 chunk 内容，让它根据 chunk 生成 2~3 个可能的用户问题和对应的标准答案。

比如给模型这段 chunk：

```
iPhone 16 Pro 支持 7 天无理由退货，需保持商品完好、配件齐全、包装完整。
退货运费由买家承担，质量问题由卖家承担运费。
```

模型可能生成：

```json
[
    {
        "query": "iPhone 16 Pro 可以退货吗？",
        "expectedAnswer": "可以。iPhone 16 Pro 支持 7 天无理由退货，需保持商品完好、配件齐全、包装完整。"
    },
    {
        "query": "iPhone 16 Pro 退货运费谁出？",
        "expectedAnswer": "正常退货运费由买家承担，如果是质量问题则由卖家承担运费。"
    }
]
```

优点是**快速低成本**，半小时就能生成上百条。缺点是模型生成的问题可能不够真实——真实用户不会这样问问题（真实用户可能会说“买了 iPhone 16 Pro 不想要了还能退吗”），而且模型可能会遗漏一些边界场景。

**建议做法**：大模型先批量生成，然后**人工校验筛选**——删掉质量差的、补充遗漏的边界场景、调整措辞让问题更贴近真实用户的表达方式。这样既快又能保证质量。

### 3. 评测集的规模与覆盖

50~100 条起步，但要注意覆盖度，不要全是简单直接的问题：

- **不同意图类型**：知识检索、工具调用、闲聊、模糊问题都要覆盖
- **不同难度**：简单直接的问题（保修期多久）和复杂问题（质量问题的退货流程和普通退货有什么区别）都要有
- **边界场景**：知识库里没有答案的问题（应该触发兜底）、跨多个 chunk 的问题（需要综合多个来源）、模糊问题（有推荐的吗）
- **按问题类型均衡分布**：如果 80% 的评测数据都是简单的产品信息查询，那评测结果只能说明简单查询效果不错，复杂场景的效果你还是不知道

建议按比例分配：简单问题 40%、中等问题 30%、复杂/边界问题 30%。

## 自动化评测：LLM-as-Judge

评测数据集有了，指标也定义好了，接下来的问题是——**怎么算这些指标？**

检索指标好算：对比一下检索返回的 chunk ID 和评测数据标注的 `relevantChunkIds`，程序就能自动计算 Hit Rate、MRR。

但生成指标怎么算？忠实度、答案相关性、答案正确率——这些都需要理解答案的语义。比如保修期是 1 年和保修期为一年，含义一样但字面不同，简单的字符串匹配根本不行。让人来评判准确但太慢太贵——100 条评测数据，每条要看 chunk 内容、模型回答、标准答案，逐个打分，一个人评一天都不一定评得完。

### 1. 用大模型做自动评分

思路很直接：**用一个大模型来充当评委，对另一个大模型的回答打分**。这就是 LLM-as-Judge。

给评委模型一个评分 Prompt，明确告诉它评分维度、评分标准，让它输出结构化的评分结果。设计三个评分 Prompt，分别评估忠实度、相关性和正确率。

**忠实度评分 Prompt**：

```
你是一个专业的 RAG 系统评估员。你的任务是评估模型的回答是否忠实于给定的参考文档内容。

评分标准：
- 5 分：回答完全基于参考文档，没有添加任何文档中没有的信息
- 4 分：回答基本基于参考文档，有极少量合理推断但不影响准确性
- 3 分：回答部分基于参考文档，但添加了一些文档中没有的信息
- 2 分：回答包含较多文档中没有的信息，存在明显编造
- 1 分：回答与参考文档内容严重不符或大量编造

参考文档内容：
{chunks}

模型的回答：
{answer}

请按以下 JSON 格式输出评分结果，不要输出其他内容：
{"score": <1-5的整数>, "label": "<faithful/partially_faithful/unfaithful>", "reason": "<简要说明评分理由>"}
```

**答案相关性评分 Prompt**：

```
你是一个专业的 RAG 系统评估员。你的任务是评估模型的回答是否回答了用户的问题。

评分标准：
- 5 分：直接、完整地回答了用户的问题
- 4 分：回答了用户的问题，但不够完整或包含了多余信息
- 3 分：部分回答了用户的问题，但遗漏了关键信息
- 2 分：回答与用户的问题有关，但没有真正回答问题
- 1 分：回答与用户的问题完全无关

用户问题：
{query}

模型的回答：
{answer}

请按以下 JSON 格式输出评分结果，不要输出其他内容：
{"score": <1-5的整数>, "label": "<relevant/partially_relevant/irrelevant>", "reason": "<简要说明评分理由>"}
```

**答案正确率评分 Prompt**：

```
你是一个专业的 RAG 系统评估员。你的任务是评估模型的回答是否正确。

评分标准：
- 5 分：回答与标准答案的含义完全一致
- 4 分：回答与标准答案基本一致，核心信息正确，细节略有差异
- 3 分：回答部分正确，但遗漏或错误了一些重要信息
- 2 分：回答包含正确信息，但主要结论有误
- 1 分：回答与标准答案完全不一致

用户问题：
{query}

标准答案：
{expectedAnswer}

模型的回答：
{answer}

请按以下 JSON 格式输出评分结果，不要输出其他内容：
{"score": <1-5的整数>, "label": "<correct/partially_correct/incorrect>", "reason": "<简要说明评分理由>"}
```

三个 Prompt 的设计思路是一致的：明确角色 → 给出评分标准（5 分制，每个分数有清晰的定义）→ 提供评估所需的信息 → 要求 JSON 格式输出。JSON 格式输出很重要——程序需要解析评分结果，自由文本没法自动统计。

### 2. LLM-as-Judge 的局限与校准

LLM 做评委不是 100% 准确的，有几个已知的偏差：

| 偏差类型 | 表现                             | 影响                                       |
| -------- | -------------------------------- | ------------------------------------------ |
| 位置偏差 | 倾向于给排在前面的答案更高分     | 多组对比评分时，换一下顺序结果可能不同     |
| 冗长偏差 | 倾向于给更长的答案更高分         | 简洁但正确的答案可能被低估                 |
| 自我偏好 | 某些模型对自己生成的内容评分偏高 | 评委模型和生成模型用同一个时，评分可能虚高 |

应对方式：**定期用人工评分校准 LLM 评分的一致性**。具体做法：

1. 从评测结果中随机抽取 20~30 条
2. 人工对这 20~30 条打分（用同样的评分标准）
3. 对比人工评分和 LLM 评分的一致率
4. 一致率低于 80% → 调整评分 Prompt（比如增加评分示例、细化评分标准的描述）
5. 如果调了 Prompt 还是不行，考虑换一个评委模型

> 一个实用的小技巧：评委模型尽量用跟生成模型不同的模型。比如生成用 Qwen，评委用 DeepSeek，避免自我偏好问题。

## Java 实战：RAG 评测流程的实现

前面讲了评估指标、评测数据集、LLM-as-Judge 的原理，接下来用 Java 代码把整个评测流程串起来。

整个流程分三步：

1. **定义评测数据集**：构建一组评测数据（问题 + 标准答案 + 正确 chunk ID）
2. **批量执行与评分**：遍历评测集，对每个问题模拟 RAG 检索和生成，用 LLM 自动评分
3. **输出评估报告**：汇总各项指标，列出 bad case

### 1. 评测数据集定义

先定义评测数据的结构和一个小型评测集：

```java
import com.google.gson.Gson;
import com.google.gson.JsonArray;
import com.google.gson.JsonObject;
import com.google.gson.JsonParser;
import okhttp3.*;

import java.io.IOException;
import java.util.*;
import java.util.concurrent.TimeUnit;

public class RAGEvaluator {

    private static final String API_URL = "https://api.siliconflow.cn/v1/chat/completions";
    private static final String API_KEY = "sk-xxx"; // 替换为你的 API Key
    private static final String JUDGE_MODEL = "deepseek-ai/DeepSeek-V3";
    private static final Gson gson = new Gson();
    private static final OkHttpClient client = new OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(120, TimeUnit.SECONDS)
            .build();

    // 评测数据结构
    static class EvalCase {
        String query;            // 用户问题
        String expectedAnswer;   // 标准答案
        List<String> relevantChunkIds;  // 正确答案对应的 chunk ID
        String intent;           // 意图类别

        EvalCase(String query, String expectedAnswer, List<String> relevantChunkIds, String intent) {
            this.query = query;
            this.expectedAnswer = expectedAnswer;
            this.relevantChunkIds = relevantChunkIds;
            this.intent = intent;
        }
    }

    // 评分结果
    static class ScoreResult {
        int score;       // 1-5 分
        String reason;   // 评分理由
    }

    // 单条评测结果
    static class EvalResult {
        EvalCase evalCase;
        List<String> retrievedChunkIds;  // 实际检索到的 chunk ID 列表
        String actualAnswer;             // 模型实际生成的答案
        boolean hit;                     // 检索是否命中
        double reciprocalRank;           // 倒数排名
        ScoreResult faithfulness;        // 忠实度评分
        ScoreResult relevancy;           // 相关性评分
        ScoreResult correctness;         // 正确率评分
    }

    // 构建评测数据集
    static List<EvalCase> buildEvalDataset() {
        return List.of(
            new EvalCase(
                "iPhone 16 Pro 的退货政策是什么？",
                "iPhone 16 Pro 支持 7 天无理由退货，需保持商品完好、配件齐全、包装完整。退货运费由买家承担，质量问题由卖家承担运费。",
                List.of("chunk_12", "chunk_13"),
                "knowledge"
            ),
            new EvalCase(
                "AirPods Pro 的保修期是多久？",
                "AirPods Pro 保修期为 1 年，自购买之日起计算。保修范围包括硬件故障和制造缺陷，不包括人为损坏和进水。",
                List.of("chunk_21"),
                "knowledge"
            ),
            new EvalCase(
                "退货运费谁承担？",
                "正常退货运费由买家承担。如果是商品质量问题导致的退货，运费由卖家承担。",
                List.of("chunk_13"),
                "knowledge"
            ),
            new EvalCase(
                "跨境商品能退货吗？",
                "跨境商品支持退货，但需要在签收后 7 天内提出。退货运费由买家承担，且需要自行办理退货物流。部分商品可能不支持退货，以商品详情页说明为准。",
                List.of("chunk_35", "chunk_36"),
                "knowledge"
            ),
            new EvalCase(
                "质量问题怎么换货？",
                "质量问题换货流程：1. 在订单详情页提交换货申请并上传质量问题照片 2. 等待客服审核（1-2 个工作日）3. 审核通过后寄回商品，运费由卖家承担 4. 收到商品后 3 个工作日内寄出新商品。",
                List.of("chunk_08", "chunk_09"),
                "knowledge"
            ),
            new EvalCase(
                "Apple Watch Ultra 的防水等级是多少？",
                "抱歉，当前知识库中没有找到 Apple Watch Ultra 防水等级的相关信息。建议您查看商品详情页或联系人工客服获取准确信息。",
                List.of(),
                "knowledge"
            )
        );
    }
```

评测集包含 6 条数据：5 条正常的知识检索问题，1 条知识库里没有答案的问题（用来测兜底）。实际项目中建议 50~100 条起步。

### 2. 批量执行与评分

核心逻辑：遍历评测集，对每个问题模拟检索和生成，然后用 LLM 评分。

```java
    // ========== 模拟 RAG 检索和生成 ==========

    /**
     * 模拟 RAG 检索：实际项目中这里调用向量数据库检索
     * 这里用硬编码模拟，重点展示评测流程
     */
    static Map<String, List<String>> simulateRetrieval() {
        Map<String, List<String>> results = new HashMap<>();
        // 模拟每个问题的 Top-3 检索结果
        results.put("iPhone 16 Pro 的退货政策是什么？",
                List.of("chunk_12", "chunk_05", "chunk_33"));
        results.put("AirPods Pro 的保修期是多久？",
                List.of("chunk_21", "chunk_07", "chunk_44"));
        results.put("退货运费谁承担？",
                List.of("chunk_18", "chunk_29", "chunk_55")); // 没命中 chunk_13
        results.put("跨境商品能退货吗？",
                List.of("chunk_35", "chunk_03", "chunk_67")); // 只命中了 chunk_35
        results.put("质量问题怎么换货？",
                List.of("chunk_08", "chunk_15", "chunk_22")); // 命中了 chunk_08
        results.put("Apple Watch Ultra 的防水等级是多少？",
                List.of("chunk_50", "chunk_51", "chunk_52")); // 全不相关
        return results;
    }

    /**
     * 模拟 RAG 生成：实际项目中这里用检索到的 chunk 内容 + Prompt 调用大模型生成
     * 这里用硬编码模拟不同质量的回答，展示评分效果
     */
    static Map<String, String> simulateGeneration() {
        Map<String, String> results = new HashMap<>();
        results.put("iPhone 16 Pro 的退货政策是什么？",
                "iPhone 16 Pro 支持 7 天无理由退货，需要保持商品完好、配件齐全。退货运费由买家承担，质量问题运费由卖家承担。");
        results.put("AirPods Pro 的保修期是多久？",
                "AirPods Pro 的保修期为 1 年，从购买日期开始计算。保修覆盖硬件故障和制造缺陷，人为损坏和进水不在保修范围内。");
        results.put("退货运费谁承担？",
                "一般情况下退货运费由买家自行承担。不过如果是因为商品本身的质量问题需要退货，运费会由卖家来承担。");
        results.put("跨境商品能退货吗？",
                "跨境商品可以退货，需要在签收后 7 天内申请。退货运费由买家承担。需要注意的是，跨境退货支持全球免费上门取件服务。");  // 全球免费上门取件是编造的
        results.put("质量问题怎么换货？",
                "质量问题换货步骤：1. 提交换货申请并上传照片 2. 等待审核 1-2 个工作日 3. 寄回商品（运费卖家承担）4. 收到后 3 个工作日寄出新商品。");
        results.put("Apple Watch Ultra 的防水等级是多少？",
                "抱歉，目前没有找到 Apple Watch Ultra 防水等级的相关信息，建议您查看商品详情页或联系人工客服确认。");
        return results;
    }

    // ========== 检索指标计算 ==========

    /**
     * 计算命中率：Top-K 里有没有包含正确 chunk
     */
    static boolean calculateHit(List<String> retrievedIds, List<String> relevantIds) {
        if (relevantIds.isEmpty()) {
            return false;  // 兜底样本没有相关 chunk 标注，不参与命中判断
        }
        for (String id : retrievedIds) {
            if (relevantIds.contains(id)) {
                return true;
            }
        }
        return false;
    }

    /**
     * 计算倒数排名：正确 chunk 排在第几位
     */
    static double calculateReciprocalRank(List<String> retrievedIds, List<String> relevantIds) {
        if (relevantIds.isEmpty()) {
            return 0.0;  // 兜底样本没有相关 chunk 标注，不参与 MRR 计算
        }
        for (int i = 0; i < retrievedIds.size(); i++) {
            if (relevantIds.contains(retrievedIds.get(i))) {
                return 1.0 / (i + 1);
            }
        }
        return 0.0;  // Top-K 里没有正确答案
    }

    // ========== LLM 评分 ==========

    /**
     * 调用大模型进行评分
     */
    static ScoreResult llmScore(String scorePrompt) throws IOException {
        JsonObject requestBody = new JsonObject();
        requestBody.addProperty("model", JUDGE_MODEL);

        JsonArray messages = new JsonArray();
        JsonObject userMessage = new JsonObject();
        userMessage.addProperty("role", "user");
        userMessage.addProperty("content", scorePrompt);
        messages.add(userMessage);

        requestBody.add("messages", messages);
        requestBody.addProperty("temperature", 0.1);
        requestBody.addProperty("max_tokens", 200);

        Request request = new Request.Builder()
                .url(API_URL)
                .addHeader("Authorization", "Bearer " + API_KEY)
                .addHeader("Content-Type", "application/json")
                .post(RequestBody.create(requestBody.toString(),
                        MediaType.parse("application/json")))
                .build();

        try (Response response = client.newCall(request).execute()) {
            String body = response.body().string();
            JsonObject json = JsonParser.parseString(body).getAsJsonObject();
            String content = json.getAsJsonArray("choices")
                    .get(0).getAsJsonObject()
                    .getAsJsonObject("message")
                    .get("content").getAsString()
                    .trim();

            // 提取 JSON 部分（模型可能输出额外文字）
            int start = content.indexOf("{");
            int end = content.lastIndexOf("}") + 1;
            if (start >= 0 && end > start) {
                content = content.substring(start, end);
            }

            return gson.fromJson(content, ScoreResult.class);
        }
    }

    /**
     * 忠实度评分
     */
    static ScoreResult scoreFaithfulness(String chunks, String answer) throws IOException {
        String prompt = "你是一个专业的 RAG 系统评估员。你的任务是评估模型的回答是否忠实于给定的参考文档内容。\n\n"
                + "评分标准：\n"
                + "- 5 分：回答完全基于参考文档，没有添加任何文档中没有的信息\n"
                + "- 4 分：回答基本基于参考文档，有极少量合理推断但不影响准确性\n"
                + "- 3 分：回答部分基于参考文档，但添加了一些文档中没有的信息\n"
                + "- 2 分：回答包含较多文档中没有的信息，存在明显编造\n"
                + "- 1 分：回答与参考文档内容严重不符或大量编造\n\n"
                + "参考文档内容：\n" + chunks + "\n\n"
                + "模型的回答：\n" + answer + "\n\n"
                + "请按以下 JSON 格式输出评分结果，不要输出其他内容：\n"
                + "{\"score\": <1-5的整数>, \"label\": \"<faithful/partially_faithful/unfaithful>\", "
                + "\"reason\": \"<简要说明评分理由>\"}";
        return llmScore(prompt);
    }

    /**
     * 相关性评分
     */
    static ScoreResult scoreRelevancy(String query, String answer) throws IOException {
        String prompt = "你是一个专业的 RAG 系统评估员。你的任务是评估模型的回答是否回答了用户的问题。\n\n"
                + "评分标准：\n"
                + "- 5 分：直接、完整地回答了用户的问题\n"
                + "- 4 分：回答了用户的问题，但不够完整或包含了多余信息\n"
                + "- 3 分：部分回答了用户的问题，但遗漏了关键信息\n"
                + "- 2 分：回答与用户的问题有关，但没有真正回答问题\n"
                + "- 1 分：回答与用户的问题完全无关\n\n"
                + "用户问题：\n" + query + "\n\n"
                + "模型的回答：\n" + answer + "\n\n"
                + "请按以下 JSON 格式输出评分结果，不要输出其他内容：\n"
                + "{\"score\": <1-5的整数>, \"label\": \"<relevant/partially_relevant/irrelevant>\", "
                + "\"reason\": \"<简要说明评分理由>\"}";
        return llmScore(prompt);
    }

    /**
     * 正确率评分
     */
    static ScoreResult scoreCorrectness(String query, String expectedAnswer, String actualAnswer) throws IOException {
        String prompt = "你是一个专业的 RAG 系统评估员。你的任务是评估模型的回答是否正确。\n\n"
                + "评分标准：\n"
                + "- 5 分：回答与标准答案的含义完全一致\n"
                + "- 4 分：回答与标准答案基本一致，核心信息正确，细节略有差异\n"
                + "- 3 分：回答部分正确，但遗漏或错误了一些重要信息\n"
                + "- 2 分：回答包含正确信息，但主要结论有误\n"
                + "- 1 分：回答与标准答案完全不一致\n\n"
                + "用户问题：\n" + query + "\n\n"
                + "标准答案：\n" + expectedAnswer + "\n\n"
                + "模型的回答：\n" + actualAnswer + "\n\n"
                + "请按以下 JSON 格式输出评分结果，不要输出其他内容：\n"
                + "{\"score\": <1-5的整数>, \"label\": \"<correct/partially_correct/incorrect>\", "
                + "\"reason\": \"<简要说明评分理由>\"}";
        return llmScore(prompt);
    }
```

### 3. 评估报告输出

把所有评测结果汇总，输出一份完整的评估报告：

```java
    // ========== 评估报告 ==========

    static void printEvalReport(List<EvalResult> results) {
        System.out.println("=" .repeat(70));
        System.out.println("                    RAG 系统评估报告");
        System.out.println("=" .repeat(70));

        // --- 检索指标 ---
        List<EvalResult> retrievalResults = results.stream()
                .filter(r -> !r.evalCase.relevantChunkIds.isEmpty())
                .toList();
        long hitCount = retrievalResults.stream().filter(r -> r.hit).count();
        double hitRate = (double) hitCount / retrievalResults.size();
        double mrr = retrievalResults.stream()
                .mapToDouble(r -> r.reciprocalRank).average().orElse(0);

        System.out.println("\n【检索阶段指标】");
        System.out.printf("  命中率（Hit Rate）：%.1f%%（%d / %d）%n",
                hitRate * 100, hitCount, retrievalResults.size());
        System.out.printf("  MRR（平均倒数排名）：%.3f%n", mrr);

        // --- 生成指标 ---
        double avgFaithfulness = results.stream()
                .filter(r -> r.faithfulness != null)
                .mapToInt(r -> r.faithfulness.score).average().orElse(0);
        double avgRelevancy = results.stream()
                .filter(r -> r.relevancy != null)
                .mapToInt(r -> r.relevancy.score).average().orElse(0);
        long hallucinationCount = results.stream()
                .filter(r -> r.faithfulness != null && r.faithfulness.score <= 2)
                .count();
        double hallucinationRate = (double) hallucinationCount / results.size();

        System.out.println("\n【生成阶段指标】");
        System.out.printf("  忠实度平均分：%.2f / 5.0%n", avgFaithfulness);
        System.out.printf("  相关性平均分：%.2f / 5.0%n", avgRelevancy);
        System.out.printf("  明显幻觉率：%.1f%%（%d / %d 条存在明显幻觉）%n",
                hallucinationRate * 100, hallucinationCount, results.size());

        // --- 端到端指标 ---
        double avgCorrectness = results.stream()
                .filter(r -> r.correctness != null)
                .mapToInt(r -> r.correctness.score).average().orElse(0);
        long correctCount = results.stream()
                .filter(r -> r.correctness != null && r.correctness.score >= 4)
                .count();
        double correctRate = (double) correctCount / results.size();

        // 兜底率：回答中包含抱歉、找不到、没有找到等关键词的比例
        long fallbackCount = results.stream()
                .filter(r -> r.actualAnswer.contains("抱歉") || r.actualAnswer.contains("找不到")
                        || r.actualAnswer.contains("没有找到"))
                .count();
        double fallbackRate = (double) fallbackCount / results.size();

        System.out.println("\n【端到端指标】");
        System.out.printf("  正确率评分均值：%.2f / 5.0%n", avgCorrectness);
        System.out.printf("  答案正确率（≥4 分）：%.1f%%（%d / %d）%n",
                correctRate * 100, correctCount, results.size());
        System.out.printf("  兜底率：%.1f%%（%d / %d）%n",
                fallbackRate * 100, fallbackCount, results.size());

        // --- Bad Case 列表 ---
        System.out.println("\n【Bad Case 列表】（正确率评分 < 4 分的问题）");
        System.out.println("-".repeat(70));
        boolean hasBadCase = false;
        for (EvalResult r : results) {
            if (r.correctness != null && r.correctness.score < 4) {
                hasBadCase = true;
                System.out.printf("  问题：%s%n", r.evalCase.query);
                System.out.printf("  期望答案：%s%n", r.evalCase.expectedAnswer);
                System.out.printf("  实际答案：%s%n", r.actualAnswer);
                System.out.printf("  检索命中：%s | 忠实度：%d 分 | 相关性：%d 分 | 正确率：%d 分%n",
                        r.hit ? "是" : "否",
                        r.faithfulness != null ? r.faithfulness.score : 0,
                        r.relevancy != null ? r.relevancy.score : 0,
                        r.correctness.score);
                // 问题归因
                if (!r.hit) {
                    System.out.println("  → 问题归因：【检索阶段】未命中正确 chunk");
                } else if (r.faithfulness != null && r.faithfulness.score <= 3) {
                    System.out.println("  → 问题归因：【生成阶段】回答与 chunk 内容不够一致，存在编造或额外推断");
                } else {
                    System.out.println("  → 问题归因：【知识库】chunk 内容可能不完整或过时");
                }
                System.out.println("-".repeat(70));
            }
        }
        if (!hasBadCase) {
            System.out.println("  无 Bad Case，所有评测问题的正确率评分均 ≥ 4 分");
        }

        System.out.println("\n" + "=".repeat(70));
    }
```

### 4. 完整运行

`main` 方法把整个评测流程串起来：

```java
    public static void main(String[] args) throws Exception {
        // 1. 构建评测数据集
        List<EvalCase> evalDataset = buildEvalDataset();
        System.out.println("评测数据集：" + evalDataset.size() + " 条");

        // 2. 模拟检索和生成（实际项目中替换为真实的 RAG 流程）
        Map<String, List<String>> retrievalResults = simulateRetrieval();
        Map<String, String> generationResults = simulateGeneration();

        // 模拟 chunk 内容（实际项目中从向量数据库获取）
        Map<String, String> chunkContents = Map.of(
            "chunk_12", "iPhone 16 Pro 支持 7 天无理由退货，需保持商品完好、配件齐全、包装完整。",
            "chunk_13", "退货运费由买家承担，质量问题由卖家承担运费。",
            "chunk_21", "AirPods Pro 保修期为 1 年，自购买之日起计算。保修范围包括硬件故障和制造缺陷，不包括人为损坏和进水。",
            "chunk_35", "跨境商品支持退货，需在签收后 7 天内提出。退货运费由买家承担，需自行办理退货物流。",
            "chunk_08", "质量问题换货流程：1. 提交换货申请并上传照片 2. 等待审核 1-2 个工作日 3. 寄回商品运费由卖家承担 4. 收到后 3 个工作日寄出新商品。"
        );

        // 3. 逐条评测
        List<EvalResult> evalResults = new ArrayList<>();
        for (int i = 0; i < evalDataset.size(); i++) {
            EvalCase evalCase = evalDataset.get(i);
            System.out.printf("\n评测第 %d/%d 条：%s%n", i + 1, evalDataset.size(), evalCase.query);

            EvalResult result = new EvalResult();
            result.evalCase = evalCase;

            // 获取模拟的检索和生成结果
            result.retrievedChunkIds = retrievalResults.getOrDefault(evalCase.query, List.of());
            result.actualAnswer = generationResults.getOrDefault(evalCase.query, "");

            // 计算检索指标
            result.hit = calculateHit(result.retrievedChunkIds, evalCase.relevantChunkIds);
            result.reciprocalRank = calculateReciprocalRank(result.retrievedChunkIds, evalCase.relevantChunkIds);
            if (evalCase.relevantChunkIds.isEmpty()) {
                System.out.println("  检索评估：跳过（兜底样本，无相关 chunk 标注）");
            } else {
                System.out.printf("  检索命中：%s，倒数排名：%.2f%n", result.hit ? "是" : "否", result.reciprocalRank);
            }

            // 组装检索到的 chunk 内容
            StringBuilder chunkText = new StringBuilder();
            for (String chunkId : result.retrievedChunkIds) {
                if (chunkContents.containsKey(chunkId)) {
                    chunkText.append("[").append(chunkId).append("] ")
                            .append(chunkContents.get(chunkId)).append("\n");
                }
            }
            String chunks = !chunkText.isEmpty() ? chunkText.toString() : "（未检索到相关内容）";

            // LLM 评分（三个维度）
            System.out.println("  正在评分...");
            result.faithfulness = scoreFaithfulness(chunks, result.actualAnswer);
            System.out.printf("  忠实度：%d 分 - %s%n", result.faithfulness.score, result.faithfulness.reason);

            result.relevancy = scoreRelevancy(evalCase.query, result.actualAnswer);
            System.out.printf("  相关性：%d 分 - %s%n", result.relevancy.score, result.relevancy.reason);

            result.correctness = scoreCorrectness(evalCase.query, evalCase.expectedAnswer, result.actualAnswer);
            System.out.printf("  正确率：%d 分 - %s%n", result.correctness.score, result.correctness.reason);

            evalResults.add(result);
        }

        // 4. 输出评估报告
        System.out.println();
        printEvalReport(evalResults);
    }
}
```

运行输出（评分由 LLM 实际打出，以下是典型的输出效果）：

```
评测数据集：6 条

评测第 1/6 条：iPhone 16 Pro 的退货政策是什么？
  检索命中：是，倒数排名：1.00
  正在评分...
  忠实度：3 分 - 模型正确提取了退货政策及主要条件，但添加了参考文档中未提及的运费承担规则，且遗漏了‘包装完整’这一条件，符合‘部分基于参考文档，但添加了一些文档中没有的信息’的标准。
  相关性：5 分 - 模型直接且完整地回答了用户的问题，涵盖了退货期限、商品状态要求及运费承担等关键政策细节。
  正确率：4 分 - 模型回答涵盖了退货期限、商品状态及运费责任等核心信息，仅遗漏了‘包装完整’这一细节，符合 4 分标准（基本一致，核心信息正确，细节略有差异）。

评测第 2/6 条：AirPods Pro 的保修期是多久？
  检索命中：是，倒数排名：1.00
  正在评分...
  忠实度：5 分 - 模型回答完全基于参考文档，准确复述了保修期限、起算时间、覆盖范围及排除项，未添加任何文档中没有的信息。
  相关性：5 分 - 模型直接且完整地回答了保修期为 1 年，准确满足了用户的查询需求，并补充了相关保修范围信息。
  正确率：5 分 - 模型回答与标准答案在保修时长、起算时间、保修范围及免责条款上完全一致，语义准确无误。

评测第 3/6 条：退货运费谁承担？
  检索命中：否，倒数排名：0.00
  正在评分...
  忠实度：1 分 - 参考文档明确显示“未检索到相关内容”，即没有任何可用信息。模型却提供了具体的退货政策细节，完全未基于参考文档进行回答，属于在缺乏上下文情况下的外部知识生成，严重违背了 RAG 系统对答案忠实于参考文档的核心要求。
  相关性：5 分 - 模型直接且完整地回答了用户的问题，清晰说明了通常情况下买家承担以及因质量问题时卖家承担的两种情形。
  正确率：5 分 - 模型回答与标准答案的核心信息完全一致，准确涵盖了正常退货和因质量问题退货两种情况下的运费承担方，语义无偏差。

评测第 4/6 条：跨境商品能退货吗？
  检索命中：是，倒数排名：1.00
  正在评分...
  忠实度：3 分 - 模型准确提取了退货时效和运费承担方等核心信息，但添加了参考文档中完全未提及的‘全球免费上门取件服务’，属于添加了文档中没有的信息。
  相关性：5 分 - 模型直接确认了跨境商品可以退货，并补充了申请时限和运费责任等关键信息，完整且清晰地回应了用户的核心问题。
  正确率：3 分 - 模型正确回答了退货可行性、时限及运费承担等核心信息，但与标准答案在退货物流方式上存在矛盾（标准称需自行办理，模型称免费上门取件），且遗漏了‘部分商品可能不支持退货’这一重要限制条件。

评测第 5/6 条：质量问题怎么换货？
  检索命中：是，倒数排名：1.00
  正在评分...
  忠实度：5 分 - 模型回答完整复述了参考文档中的四个步骤及关键信息（如时间、费用承担方），未添加任何外部信息或编造内容，仅对个别措辞进行了同义替换或格式微调，完全忠实于原文。
  相关性：5 分 - 模型直接、清晰地列出了质量问题换货的具体步骤，涵盖了申请、审核、寄回及补发全流程，完整回答了用户关于操作方法的疑问。
  正确率：4 分 - 模型回答准确涵盖了换货流程的四个核心步骤及关键信息（审核时间、运费承担、补发时效），逻辑正确。但在第一步中遗漏了具体的操作入口（订单详情页）和照片的具体要求（质量问题照片），属于细节上的缺失，符合 4 分标准（核心信息正确，细节略有差异）。

评测第 6/6 条：Apple Watch Ultra 的防水等级是多少？
  检索评估：跳过（兜底样本，无相关 chunk 标注）
  正在评分...
  忠实度：3 分 - 参考文档为空（未检索到相关内容），模型正确反馈了未找到信息的情况，但额外添加了‘查看商品详情页或联系人工客服’的建议，该内容不属于参考文档范围。
  相关性：2 分 - 模型明确表示未找到相关信息，未能提供用户询问的具体防水等级数值，属于话题相关但未真正回答问题。
  正确率：5 分 - 模型回答与标准答案的核心语义完全一致，均表明知识库中未检索到相关信息，并给出了相同的后续建议（查看商品详情页或联系客服），仅个别措辞有细微差别，不影响整体含义。

======================================================================
                    RAG 系统评估报告
======================================================================

【检索阶段指标】
  命中率（Hit Rate）：80.0%（4 / 5）
  MRR（平均倒数排名）：0.800

【生成阶段指标】
  忠实度平均分：3.33 / 5.0
  相关性平均分：4.50 / 5.0
  明显幻觉率：16.7%（1 / 6 条存在明显幻觉）

【端到端指标】
  正确率评分均值：4.33 / 5.0
  答案正确率（≥4 分）：83.3%（5 / 6）
  兜底率：16.7%（1 / 6）

【Bad Case 列表】（正确率评分 < 4 分的问题）
----------------------------------------------------------------------
  问题：跨境商品能退货吗？
  期望答案：跨境商品支持退货，但需要在签收后 7 天内提出。退货运费由买家承担，且需要自行办理退货物流。部分商品可能不支持退货，以商品详情页说明为准。
  实际答案：跨境商品可以退货，需要在签收后 7 天内申请。退货运费由买家承担。需要注意的是，跨境退货支持全球免费上门取件服务。
  检索命中：是 | 忠实度：3 分 | 相关性：5 分 | 正确率：3 分
  → 问题归因：【生成阶段】回答与 chunk 内容不够一致，存在编造或额外推断
----------------------------------------------------------------------

======================================================================
```

从报告中可以很清楚地看到：

- **检索阶段**：Hit Rate 80%，有一条退货运费谁承担没有检索到正确的 chunk。MRR 0.8，说明命中的问题排名都不错
- **生成阶段**：忠实度平均分 3.33，不太理想，有三条低分值得分析：
  - iPhone 16 Pro 退货政策忠实度 3 分——检索命中了 chunk_12（退货条件），但模型回答里提到了退货运费由买家承担，这个信息在 chunk_13 里，而 chunk_13 没有被检索到。模型把自身知识和 chunk 内容混在了一起，这是 RAG 系统很常见的一种幻觉形式
  - 退货运费谁承担忠实度 1 分——没检索到 chunk，模型完全靠自身知识回答。但有意思的是，这条的正确率是 5 分（答案恰好是对的）。这正好印证了前面讲的**忠实度 ≠ 正确率**——模型没基于 chunk 回答，忠实度很低；但碰巧自身知识是对的，正确率反而很高。如果哪天知识库更新了运费政策，模型还是用旧知识回答，就会又不忠实又不正确
  - 跨境商品退货忠实度 3 分——编造了全球免费上门取件服务

  明显幻觉率 16.7%（1 条 ≤ 2 分），刚好踩在 15% 的红线上
- **端到端**：正确率 83.3%，兜底率 16.7%（示例集中只有 6 条样本，1 条兜底就会达到这个比例，在演示场景里属正常现象）
- **Bad Case**：跨境商品退货这条问题归因到生成阶段——检索命中了正确 chunk，但模型编造了额外信息。优化方向：调整 Prompt，加强严禁添加参考文档中未出现的信息的指令

## 评估驱动的优化

评估报告出来之后，不是看看就完了。评估的价值在于**指导优化**——根据指标表现定位问题环节，针对性地改进。

### 1. 根据评估结果定位问题

用一张决策表来理清哪个指标差、问题在哪、该怎么优化的逻辑：

![](https://oss.open8gu.com/iShot_2026-03-06_10.13.31.png)

拿前面的评估报告举例：

- Hit Rate 80%，低于 85% → 检索阶段有问题，退货运费谁承担没有命中正确 chunk。分析原因：可能是退货运费和 chunk_13 的文本退货运费由买家承担语义匹配度不够，或者 Top-3 太少了。优化思路：把 Top-K 从 3 调到 5，或者加入重排序。
- 忠实度 3.33，整体偏低，退货运费忠实度 1 分、跨境商品退货和 Apple Watch 防水各 3 分 → 生成阶段的 Prompt 需要加强限定。优化思路：在 System Prompt 里加一条严禁添加参考文档中未出现的信息，对未检索到内容的场景强化兜底指令。

### 2. 优化后的验证

每次优化之后，**重新跑一遍评测集**，对比优化前后的指标变化。这里有一个关键原则：**不要只看改善的指标，还要检查有没有其他指标变差。**

比如你把 Top-K 从 3 调到 5，Hit Rate 从 80% 提升到了 90%，看起来不错。但要同时检查：
- 忠实度有没有下降？多召回了 2 个 chunk，如果多出来的 chunk 质量不高，可能干扰模型生成，导致幻觉增加。
- 兜底率有没有变化？多召回可能让一些原本触发兜底的问题强行找到了不太相关的 chunk，模型基于不相关 chunk 编了个答案，兜底率下降了但正确率也下降了。

用表格记录每次优化的效果对比（预设结果）：

| 指标     | 优化前 | 调整 Top-K 5 后 | 加 Reranker 后 |
| -------- | ------ | --------------- | -------------- |
| Hit Rate | 80.0%  | 90.0% ↑         | 90.0%          |
| MRR      | 0.800  | 0.760 ↓         | 0.850 ↑        |
| 忠实度   | 3.33   | 3.10 ↓          | 4.20 ↑         |
| 正确率   | 83.3%  | 83.3%           | 91.7% ↑        |
| 兜底率   | 16.7%  | 16.7%           | 16.7%          |

从这个表可以看到：单纯增加 Top-K 虽然 Hit Rate 提升了，但 MRR 下降（正确 chunk 排名变差了）、忠实度也下降了（多出来的不相关 chunk 干扰了生成）。加上 Reranker 之后，各项指标才全面改善——Reranker 本身改善的是检索排序，但排序变好意味着高相关的 chunk 排到前面、噪音 chunk 被过滤掉，模型看到的上下文质量更高，忠实度自然也跟着提升。**这就是评估的价值——没有数据，你可能以为调 Top-K 就够了，实际上还需要配合 Reranker 才行。**

### 3. 持续优化的闭环

评估不是做一次就完事的，而是一个持续迭代的过程：

1. **跑评测** → 发现问题（Hit Rate 低、忠实度低、某类问题集中出错）
2. **分析归因** → 定位到具体环节（检索？生成？知识库？）
3. **针对性优化** → 调参数、改 Prompt、补知识库
4. **重新评测** → 确认改善、检查回归
5. **上线** → 收集线上用户反馈
6. **扩充评测集** → 把线上发现的新 bad case 补充到评测集中
7. 回到第 1 步

> 条件允许的话，把评测流程集成到你的 CI/CD 中。每次修改代码、更新 Prompt 或知识库后，自动跑评测集，评分低于阈值就阻止上线。这样可以在上线前发现回归问题，避免改了 A 问题、B 问题变差了的情况悄悄上线。

## 小结

这篇讲了 RAG 系统的评估与优化，核心要点：

1. **靠感觉优化是不行的**——没有评估体系，改了东西不知道效果变好还是变差，更发现不了回归问题。量化评估是系统化优化的前提
2. **分层评估定位问题**——不是只看答案对不对，而是分三层：检索阶段（chunk 找对了吗）、生成阶段（答案忠实吗）、端到端（用户满意吗）。哪层指标差，就优化哪层
3. **检索指标**：Hit Rate（命中了没有）和 MRR（排在第几位）最常用；Recall 和 Precision 适合需要多 chunk 回答的场景
4. **生成指标**：忠实度（有没有编造）和答案相关性（有没有答非所问）是核心。其中幻觉率（忠实度 ≤ 2 分的占比）是系统级红线指标，RAG 的核心价值就是基于检索回答，幻觉率居高不下就失去了意义
5. **评测数据集是基础**：50~100 条起步，覆盖不同类型和难度，大模型辅助生成 + 人工校验是性价比最高的方式
6. **LLM-as-Judge 实现自动化**：用大模型做评委，设计好评分 Prompt，JSON 格式输出方便程序解析。定期用人工校准一致性
7. **评估驱动优化**：根据指标定位问题环节，针对性改进，改完重新评测，确认改善且无回归

到这里，整个 RAG 系列从概念到实现、从各环节到评估，一条完整的链路已经讲完了：

**数据入库 → 分块 → 元数据 → 向量化 → 向量数据库 → 检索策略 → 生成策略 → Function Call → MCP 协议 → 会话记忆 → Query 改写 → 意图识别与路由 → 评估与优化**

每个环节都有原理讲解、Java 代码实战和生产级的注意事项。你已经具备了从零搭建一套 RAG 系统的完整知识，也知道了怎么用数据驱动的方式持续优化它。剩下的就是在实际项目中不断实践、不断踩坑、不断改进。
