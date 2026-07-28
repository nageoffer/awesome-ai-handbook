上一篇我们手写了主从式多智能体：编排者（Orchestrator）负责拆任务和选人，商品、售后、IoT 子 Agent 各自带着独立的人设与工具箱执行任务。

这套架构已经能跑，但还留下了一个关键问题：子 Agent 的上下文是隔离的，它们怎么知道前面的 Agent 做了什么？

假设用户对比特严选提出这样一个请求：

> 订单 88231 的扫地机老掉线，帮我查保修并给出处理建议；如果要换新，预算 2000 元以内，希望能和家里的比特 SoundBox Mini 智能音箱联动。

这个请求至少涉及三个领域：

- 售后 Agent 查询订单商品和保修状态；
- 商品 Agent 根据预算筛选换新候选；
- IoT Agent 核对候选商品与现有音箱的联动能力。

商品 Agent 需要知道售后 Agent 查到的商品型号，IoT Agent 又需要知道商品 Agent 给出的候选。如果每个 Agent 都从用户原话重新推理，容易重复查询；如果把所有聊天记录全部塞给每个 Agent，上下文又会迅速膨胀。

咱们只解决一个基础问题：

> 在保持子 Agent 上下文隔离的前提下，怎样把必要信息准确地交给下一个 Agent？

## 多智能体通信，本质上是上下文工程

上下文工程（Context Engineering）指的是：为一次模型调用选择、组织并注入真正需要的信息。

单 Agent 只有一份上下文。拆成多个 Agent 后，每个 Agent 都有自己的局部上下文，于是系统必须额外回答三个问题：

1. 哪些信息需要在整个请求内长期保存？
2. 哪些信息只是一次性的任务通知？
3. 当前 Agent 应该看到哪些前序结果？

这三类信息不应该混在一个不断增长的聊天记录里。更清晰的划分是：

| 信息类型 | 示例 | 负责组件 |
|---|---|---|
| 请求级状态 | 用户原话、订单号、预算、已有设备 | `AgenticScope` |
| 一次性事件 | 给某个 Agent 的任务、结果就绪或失败通知 | `MessageHub` |
| 子任务结果 | 执行状态、自然语言答复、可共享事实 | `AgentResult` |
| 事实的来源 | 从工具返回里确定性抽取的字段 | `ToolFactExtractor` |
| 依赖关系 | 商品 Agent 依赖售后结果 | `HandoffRunner` |
| 最终输出 | 按完成顺序汇总各领域结果 | `Aggregator` |

道理很简单：

> 状态归状态，消息归消息；事实由工具产生，前序结果只按显式依赖交给下游。

这六个组件各自站在什么位置、谁和谁说话，画出来比表格更好理解。

![](https://oss.open8gu.com/iShot_2026-07-26_21.42.03.png)

注意，这张图没有让三个 Agent 共享同一份 ReAct 对话。它们共享的是经过选择的请求级状态，而不是彼此的完整思考过程。

> 本项目中具体代码已上传 GitHub [TinyAgent](https://github.com/nageoffer/tinyagent)，大家 Clone 项目后，将代码分支切换到 1.16.x，默认主分支是最新代码。运行前复制 `.env.example` 为 `.env`，把自己的 API Key 填进去，默认阿里云百炼平台；`.env` 已加入 `.gitignore`，切分支时不会丢。

## 先定义结果契约：AgentResult

如果 Agent 之间只传一个 `String`，调用方无法区分这次执行是否完成，也无法区分写给用户看的答复和后续任务真正需要的字段。

所以第一步不是造黑板，而是先定义最小结果契约：

```java
public record AgentResult(
        String agentName,
        Status status,
        String summary,
        Map<String, String> facts) {

    public enum Status {
        COMPLETED,
        FAILED
    }

    // 省略空值归一化、不可变 Map 复制，以及 completed / failed 两个静态工厂
}
```

四个字段各有明确职责：

| 字段 | 含义 | 是否适合交给后续 Agent |
|---|---|---|
| `agentName` | 结果由谁产生 | 是 |
| `status` | 这次调用是否正常完成 | 是 |
| `summary` | 这位专家写给用户看的自然语言答复，后面统一叫答复 | 只能当背景，不能当事实证据 |
| `facts` | 显式交接字段 | 是，且必须来自工具或业务系统 |

这里必须守住一个边界：

> `COMPLETED` 只表示 Agent 正常返回了结果，不表示结果中的每一句自然语言都已经被证明正确。

通信层负责把结果送到正确位置，不负责完成业务事实审计。

到这里为止，一切听起来都很顺。但真正的坑在下一节：**这个 `facts` 到底由谁填**？

## facts 从哪来：交接真正的难点

### 1. 一个很容易写出来的空壳

上一篇的普通 `Agent` 只有 `run(String)`。为了不改动它，最自然的做法是加一个很薄的适配接口：

```java
public interface HandoffAgent extends Agent {

    AgentResult execute(String task);

    // 把上一篇的普通 Agent 包一层，让它也能对外返回 AgentResult
    static HandoffAgent from(Agent agent) {
        return new HandoffAgent() {

            // 省略 name、description、run 三个方法对 agent 的原样透传

            @Override
            public AgentResult execute(String task) {
                String answer = agent.run(task);
                return AgentResult.completed(agent.name(), answer, Map.of());
            }
        };
    }
}
```

这段代码能编译、能跑、看起来很合理。但它有两处不声不响的假话：

- `facts` 永远是 `Map.of()`，也就是那个硬编码的空 Map。普通 Agent 的 `run` 只交出一段文本，适配层无从知道哪些是工具确认过的字段。
- `status` 永远是 `COMPLETED`。这里唯一的出口是 `AgentResult.completed`，哪怕 Agent 内部跑满了步数、原地打转、最后只返回一句道歉，下游看到的仍然是完成。

第一条的后果尤其严重。`facts` 一空，下游拿到的就只剩那段面向用户的长文，所谓结构化交接退化成把上游客服话术原样转述给下一个 Agent——这恰好就是咱们要避免的方案。

所以 `from()` 只适合那些不被任何人依赖的收尾型 Agent。**要参与依赖交接，就必须有人真正生产 facts**。

### 2. 让工具成为 facts 的唯一来源

facts 有两种可能的生产方式：

| 方式 | 做法 | 问题 |
|---|---|---|
| 模型自述 | 要求 Agent 最后输出一段 JSON | 又引入一次模型不确定性，字段可能被改写、漏填、编造 |
| 工具抽取 | 从本轮真实发生的工具返回里取字段 | 需要维护一张规则表 |

选第二种。理由很直接：**facts 的价值就在于它是证据，而经过模型转述的证据不再是证据**。

好在 ReAct 循环本来就把每一次工具调用的入参和返回都握在手里，只要把它交出来就行：给 `ReActAgent` 加一个 `runDetailed()` 出口，除了最终答复，再带出一份 `ToolObservation` 列表，每条记着工具名、入参和返回原文，原来 `run(String)` 的行为保持不变。

然后用一张规则表把工具字段映射成交接字段：

```java
public final class ToolFactExtractor {

    // 从 toolName 的返回值里取 fieldPath，落成名为 factKey 的交接字段
    public record Rule(String toolName, String fieldPath, String factKey) {
    }

    public Map<String, String> extract(
            List<ToolObservation> observations) {
        Map<String, String> facts = new LinkedHashMap<>();

        for (ToolObservation observation : observations) {
            JsonNode output = parse(observation.output());
            // 带 error 的返回整条跳过：失败的调用不产生事实
            if (output == null || output.hasNonNull("error")) {
                continue;
            }
            for (Rule rule : rulesOf(observation.toolName())) {
                String value = readPath(output, rule.fieldPath());
                if (!value.isBlank()) {
                    facts.put(rule.factKey(), value);
                }
            }
        }
        return facts;
    }

    // 省略按工具名筛规则，以及 fieldPath 的点号下钻（支持 productA.name、bundles.0.status）。
}
```

规则表长这样：

```java
// 售后专家：订单事实来自 queryOrder，保修结论来自 checkWarranty 的确定性日期计算
ToolFactExtractor.of(
        "queryOrder",    "orderId",         "orderId",
        "queryOrder",    "product",         "productName",
        "queryOrder",    "signTime",        "signDate",
        "queryOrder",    "status",          "orderStatus",
        "checkWarranty", "warrantyStatus",  "warrantyStatus",
        "checkWarranty", "warrantyEndDate", "warrantyEndDate");

// 省略原价和七天无理由状态两条规则。
```

有三个设计细节值得说明：

- 保修状态坚决不从知识库原文里抠。`searchKnowledge` 返回的是政策条款，`checkWarranty` 返回的才是本单结论，两者不能混。顺带说明一下，`checkWarranty` 是这一篇为交接新加的工具：第 21、22 篇里保修是模型看着签收日自己算的，那种结论没法当事实交出去，因为它依赖模型记忆中的当前日期。
- 带 `error` 字段的返回整条跳过，失败的工具调用不产生任何事实。
- 同一个 `factKey` 被多次命中时后写覆盖先写，因为后一次调用通常是模型纠正参数后的重试。

这张规则表就是领域层和通信层之间的契约，生产环境里它应该和工具的输出 schema 一起版本化。

第三条的后写覆盖先写策略，是这一篇里代价最大的一次踩坑，值得单独展开。

商品专家的规则表最早是从 `compareProducts` 的 `productA.name`、`productB.name` 抽换新候选的。字段就在那儿，值也确实是两款真实商品，看起来完全合理。真跑起来，交给下游的 `candidateA` 变成了用户现在正在用、正在掉线的那台 S10 Pro，真正的在售候选被挤没了。下游 IoT 专家拿着这份事实去核验音箱联动，核的是一台根本不在换新范围里的机器，终稿于是自相矛盾。

原因不在模型。`compareProducts` 的 `productA`、`productB` 表示这次比了谁，而不是候选是谁——它是一对位置参数。模型为了跟用户讲清楚升级关系，很自然地又拿在用型号比了一轮，位置一被后写覆盖，候选事实就从在售款漂成了在用款。**判断一个字段能不能当事实源，看的不是它出现在哪儿，而是它的语义是不是恰好等于你要的那个事实。**

改法是把候选集的权威来源交回给在售目录：`searchKnowledge` 直接返回一个结构化的 `candidates` 数组，规则表按 `candidates.0.name`、`candidates.1.name` 下钻，`compareProducts` 一条规则都不配——它只负责生成给用户看的规格对比。

```java
// 商品专家：候选事实全部来自 searchKnowledge 的在售目录
ToolFactExtractor.of(
        "searchKnowledge", "candidates.0.name",  "candidateA",
        "searchKnowledge", "candidates.0.price", "candidateAPrice",
        "searchKnowledge", "candidates.1.name",  "candidateB",
        "searchKnowledge", "candidates.1.price", "candidateBPrice",
        "searchKnowledge", "catalogScope",       "catalogScope");

// 省略两条 App 控制状态，以及候选集完整性和候选条数各一条规则。
```

改完之后，模型爱调几次 `compareProducts` 都不影响候选，因为那张目录只有一处、只会返回一次，反复检索也是同一份内容。**后写覆盖先写不是 bug，选错事实源才是。**

![](https://oss.open8gu.com/iShot_2026-07-26_21.42.04.png)

### 3. status 也不能由适配层拍脑袋

ReAct 循环本来就知道自己是怎么结束的：正常给出答复、跑满步数、连续多圈没有进展、上下文预算耗尽。把这个终止状态如实带出来，`status` 就有了可靠依据：

```java
// HandoffSpecialist 的核心方法，其余部分只是持有专家和规则表
@Override
public AgentResult execute(String task) {
    ReActAgent.RunResult run = specialist.runDetailed(task);

    // ReAct 没有正常收敛就是失败，
    // 不能让下游把一句道歉当成结论继续用
    if (!run.completed()) {
        return AgentResult.failed(
                name(),
                "ReAct 循环未正常收敛，终止原因 " + run.status()
        );
    }
    return AgentResult.completed(
            name(),
            run.answer(),
            factExtractor.extract(run.observations())
    );
}
```

对比一下两种适配的差别：

| | `HandoffAgent.from()` | `HandoffSpecialist` |
|---|---|---|
| `facts` 来源 | 恒为空 | 工具返回，确定性抽取 |
| `status` 依据 | 恒为 `COMPLETED` | ReAct 真实终止状态 |
| 适用位置 | 无人依赖的收尾节点 | 依赖链上的任何节点 |

到这里，Agent 之间要传的东西才算准备好了。下面三节讲的是怎么把它送到位。

## 共享黑板：AgenticScope

### 1. 黑板应该保存什么

共享黑板模式的核心不是大家共用一段聊天记录，而是为一次用户请求建立一个请求级状态容器。

本篇只保存三类内容：

- 用户原始请求；
- 编排层提取的共享约束；
- 各子 Agent 已经发布的 `AgentResult`。

不保存这些内容：

- 模型的 Thought；
- 完整 ReAct 轨迹；
- 每个 Agent 的全部历史消息；
- 没有明确依赖关系的其他 Agent 结果。

打个比方，黑板像项目会议室里的一块任务板。大家可以看到订单号、预算和已经确认的交付物，但不会把每个人脑子里的推理过程全写上去。

### 2. 最小实现

内部结构没什么可讲的：一个 `userRequest` 字符串，一个存共享约束的 `Map`，一个存 `AgentResult` 的 `Map`。两个 Map 都用 `LinkedHashMap`，为的是保留写入顺序；所有方法都加 `synchronized`，为的是让请求内的读写具备清晰的一致性。

只有发布结果这一步值得单独看一眼：

```java
// 同名重复发布是编排层写错了，不能悄悄覆盖上一份结果
public synchronized void publish(AgentResult result) {
    if (results.putIfAbsent(result.agentName(), result) != null) {
        throw new IllegalStateException("同一次请求不能重复发布同名 Agent 结果");
    }
}
```

它仍然只是内存版教学实现。跨进程部署、状态持久化和并发拓扑会在后续工程化章节展开。

### 3. 给子 Agent 拼上下文：给什么，不给什么

黑板最重要的方法不是 `put()`，而是给当前 Agent 组装边界明确的上下文：

```java
public synchronized String briefingFor(
        String task,
        Set<String> dependencies,
        boolean projectUpstreamSummary) {
    StringBuilder briefing = new StringBuilder();
    briefing.append("【用户原始请求】\n").append(userRequest);
    appendConstraints(briefing);

    if (dependencies != null && !dependencies.isEmpty()) {
        // 只有声明过的依赖会进上下文，而且默认只给 facts
        List<AgentResult> upstream = requireCompleted(dependencies);
        appendFacts(briefing, upstream);
        if (projectUpstreamSummary) {
            appendSummaries(briefing, upstream);
        }
    }

    return briefing.toString().stripTrailing()
            + "\n\n【本次子任务】\n" + task.strip()
            + "\n\n" + OUTPUT_SCOPE
            + "\n\n" + SECTION_STYLE;
}
```

这段拼装里有四个刻意的选择，其中第二个是踩坑踩出来的。

**第一，事实和上游答复分成两块，并明确告诉模型只有前者是证据。**

```text
【上游已确认事实】（来自上游工具返回，可直接引用，不要重复查询）
【上游答复】（仅供理解背景，不是事实证据，不要直接复述给用户）
```

混在一起写，模型会把上游那段客服话术也当成可引用的结论。

**第二，上游答复默认不给下游。**

你可能觉得上面已经明确说明这部分不是事实证据、不能直接复述，模型看到后只当背景使用就行了。实际跑下来不是这样——模型根本不听这个标注。

看一段真实运行日志就知道了。`iotSpecialist` 只声明依赖 `productSpecialist`，所以它拿到的九条事实全是候选商品的型号、价格和 App 控制状态，一条保修字段都没有。但它交回来的答复第一段是这样的：

```text
## 保修情况
你的比特 S10 Pro 扫地机目前仍在保修期内（截止 2027-06-21），
掉线问题完全属于售后覆盖范围。

## IoT 联动与网络建议
频繁掉线有可能与 Wi-Fi 信道干扰有关，建议将扫地机切换到 5GHz 频段……
```

先看保修那几行。型号和截止日不在 `iotSpecialist` 的 `facts` 里，也不在它调过的任何工具返回里。追下去会发现，它是从 `productSpecialist` 那段上游答复里读来的——而 `productSpecialist` 又是从 `afterSalesSpecialist` 的答复里抄来的。你在 `facts` 上花的功夫全白费了：自然语言转述从事实那条路堵死了，但从答复这条路绕了回来，还多转了一手。

再看下面那段 Wi-Fi 信道干扰。这句话哪儿都查不到——不在 `facts` 里，不在工具返回里，也不在上游答复里，是模型自己编的。它手里没有任何网络相关的证据，但上游答复给了它一个面面俱到的示范，顺手补一段看起来合理的东西就成了最自然的选择。

原因不难理解。上游答复本身就是一份完整的自然语言文本，带标题、带表格、面面俱到。你把这样一段东西塞进上下文，模型不会只把它当作参考资料，反而容易将其视为本轮输出的格式示范。于是它照着同样的篇幅和覆盖范围，把整单结论重写一遍——包括不归它管的、没有证据撑的那几段。

> 仅靠提示模型这不是证据、不要复述，属于软约束，模型可以违背；不放进上下文是硬约束，模型没得选。能用省略解决的，就不要用禁令解决。

去掉上游答复之后，这条依赖链完全靠 `facts` 跑通：商品 Agent 从售后抽出的 `productName` 拿到在用型号是 S10 Pro，IoT Agent 从商品抽出的 `candidateA`、`candidateB` 拿到在售目录里筛出来的两款换新候选，直接去核验它们能不能和用户的音箱联动。下游真正需要的东西，全在 `facts` 这几个字段里，上游那段写给用户看的长文没有提供任何额外信息。

> 换句话说，上游答复对跨 Agent 交接的贡献量是零——它的读者是用户，不是下一个 Agent。

所以上游答复降级成一个默认关闭的开关（就是 `briefingFor()` 的第三个参数 `projectUpstreamSummary`）。绝大多数场景下 `facts` 已经够用；只有当下游确实需要理解上游的推理过程或决策理由——光靠几个 key-value 字段说不清楚的时候，再显式打开，配合 300 字截断控制膨胀。

![](https://oss.open8gu.com/iShot_2026-07-26_21.42.05.png)

**第三，上游答复按约 300 字、在句号处截断**。即使打开也只是背景，不截断的话每多一级依赖，下游上下文就多一整段长文。硬按字数劈开可能截在半句话上，所以实际取 300 字以内最后一个句号的位置，保证下游读到的每一句都是完整的。

**第四，本次子任务、输出边界和行文统一一起压在最后。**

告诉模型要做什么还不够，还得明确不要做什么。这块分两部分：输出边界管内容不越界，行文统一管格式不打架。

#### 4.1 输出边界：堵住模型越界的三种姿势

先看最终的输出边界长什么样：

```java
private static final String OUTPUT_SCOPE = “””
        【输出边界】
        - 你的回复只是多专家终稿里的一节，只回答上面这一条子任务。
        - 不属于本次子任务的问题由别的专家负责，直接不写：既不要用”资料不足””建议联系客服确认”这类话兜底，那会和负责这块的专家结论冲突；也不要写”这部分由某某专家另行验证”这类分工说明，终稿直接呈现给用户，用户不需要知道内部怎么分工。
        - 不要复述上游已经答过的内容，不要在结尾给整单的总体建议、方案排序或最终推荐。”””;
```

这第二条不是一次性写出来的，是跑了两轮日志、堵了三次才补齐的。每堵一次，模型就换一种方式绕过来。

**第一次越界：兜底话术**。一开始只写了“别答别人的题”。模型答不了的时候没有沉默，而是拐了个弯：`productSpecialist` 在人设明令不许判断音箱兼容性的情况下，仍然写了一句“目前资料中未找到兼容性说明，建议购买前向官方客服确认”。这看起来很委婉，但它也是结论——而且和后面 `iotSpecialist` 真正查到证据后给出的“明确支持”直接打架，两段自相矛盾的话一起拼进了终稿。

> 兜底话术是越界的伪装形式。“我不确定，建议你问问客服”听着谦虚，实际上已经对这个问题表了态，必须单独堵。

**第二次越界：暴露内部分工。** 堵掉兜底之后，模型换了一种绕法：改写成“该部分由 IoT 专家另行验证，此处不做判断”。内容没错，边界也守住了，但终稿是直接给用户看的，用户不该知道后台派了几个 Agent、谁负责哪一块。所以第二条又补了半句。

追查下去发现，这个措辞不是模型自己想出来的，是商品专家的人设里原本就写着“SoundBox 联动由 IoT 专家验证”。

> 你在人设里怎么称呼同事，模型就会怎么讲给用户听。内部角色名只该出现在编排代码里，不该出现在任何一份会被模型读到的提示词里。

**第三次越界：同一个坑换个地方踩。** 人设清干净之后，我给三条 `Step.task` 补交付边界，随手就写成了“换新候选由后面两位专家负责”“音箱联动由 IoT 专家核验”——角色名又回来了，只是从人设挪到了任务描述里。`Step.task` 看着像编排代码，但它会被拼进 Prompt，模型一样会照着复述。现在这三句统一写成“不在本次范围内”，不提是谁的活。

> 凡是最终会进上下文的字符串，不管它写在哪个类里，都按提示词的标准审。

三次修补说明一件事：模型越界的方式不止一种，堵住一种它会换下一种。所以后面校验那节会把这条明确记成软约束——能压低越界频率，但不能保证不发生。

#### 4.2 约束该写在哪：人设膨胀是写错了地方

输出边界写在编排层，不写进各自的人设。原因很简单：同一位专家在不同计划里该交付的东西不一样，写进人设就等于把编排决策焊死在角色里，换个计划就要改人设。

这个道理推广一步：不只是输出边界，多 Agent 系统里每一条约束都该想清楚往哪放。调试过程中我就踩了这个坑——三份人设各堆到了十二三条，打开一看全是补丁：

- 模型把 `UNKNOWN` 说成了不支持 → 补一条“不许改写 UNKNOWN”
- 模型替用户承诺了代下单 → 补一条“不许代办”
- 模型把不归自己管的兼容判断也写了 → 补一条“不要提兼容”

三条补丁并排放在同一份人设里，看着都有来历，但仔细想想：第一条和领域无关，三个专家都要遵守；第二条是这一次分工决定的，换个请求可能就不成立；只有第三条真正属于售后专家自己的领域知识。它们不该放在同一个地方。

> 打个比方：人设像岗位职责说明书，你不会把全公司都不许迟到和本周三你负责值班都写进某个人的岗位职责里——前者是公司制度，后者是排班表，只有熟悉 Java 和中间件才属于岗位本身。

整理之后，约束只有三个该去的地方：

| 约束类型 | 该写在哪 | 举例 | 判断方法 |
|---|---|---|---|
| 与领域无关的通用纪律 | 三个专家共用的一段常量 | `UNKNOWN` 不能改写成不支持 | 换个请求还成立，且和领域无关 |
| 这一次分工的交付边界 | 编排层的 `Step.task` 和输出边界 | 音箱联动不在本次范围内 | 换个请求就不成立 |
| 领域工具的用法和硬约束 | 各自的人设 | 保修判断必须交给 `checkWarranty` | 换个请求还成立，且只对这个领域成立 |

判据就一句：**这条规则换一个请求还成立吗**？还成立且跨领域，放共用常量；还成立但只限本领域，留人设；换个请求就不成立，那就不是人设该管的事，属于编排层。

按这张表把补丁各归各家，三份人设从十二三条降到四五条，再跑日志，没有一条被挪走的禁令复发。**它们本来就不该由人设来管**。

#### 4.3 行文统一：单个 Agent 发现不了的问题

同一个位置还要压一块看起来很琐碎、但不写就一定翻车的东西：

```java
private static final String SECTION_STYLE = “””
        【行文统一】
        - 用一个二级标题（## 开头）作为本节标题，标题概括本节内容；正文里不要再出现其他二级标题。
        - 统一用”你”称呼用户，不要用”您”。”””;
```

这是内容全部答对之后才暴露出来的问题。某一轮日志里，三段答复各自都挑不出毛病，拼起来却是这样：售后那段用加粗当小标题、称呼“您”，商品那段没有标题、称呼“你”，IoT 那段用了 `##` 二级标题。终稿变成一份标题层级忽高忽低、人称跳来跳去的文档，中间那段因为没标题，读起来还像是上一段的续写。

为什么单个 Agent 发现不了这个问题？因为售后 Agent 写“您”的时候，它根本不知道旁边还有个商品 Agent 在写“你”——每个 Agent 只看得见自己那一段，单看确实没毛病。只有最后把三段拼到一起的人（也就是编排层）才看得出不一致，所以行文统一和输出边界一样，只能写在编排层。

> 你可能会问：标题层级这种事，为什么不让 `Aggregator` 在拼装时确定性地统一渲染，模型只写正文？那样确实更可靠。这里没这么做，是因为要给 `Step` 加展示字段、再一路传到聚合器，为了标题层级引入这些结构会冲淡本篇主线。但生产系统里这笔账要重新算：**凡是编排层已经知道的东西，交给模型都是在赌**。

拼出来的上下文长这样：

```text
【用户原始请求】
订单 88231 的扫地机老掉线，帮我查保修并给出处理建议……

【共享约束】
- 订单号：88231
- 预算：换新预算 2000 元以内
- 已有设备：比特 SoundBox Mini 智能音箱

【上游已确认事实】（来自上游工具返回，可直接引用，不要重复查询）
- afterSalesSpecialist.orderId：88231
- afterSalesSpecialist.productName：比特 S10 Pro 扫地机
- afterSalesSpecialist.originalPrice：1999
- afterSalesSpecialist.signDate：2026-06-22
- afterSalesSpecialist.orderStatus：已签收
- afterSalesSpecialist.warrantyStatus：IN_WARRANTY
- afterSalesSpecialist.warrantyEndDate：2027-06-21
- afterSalesSpecialist.noReasonReturnStatus：EXPIRED

【本次子任务】
结合上游给出的在用型号，筛选预算内的换新候选并说明规格差异

【输出边界】
……（就是上面那两块，此处不再重复）

【行文统一】
……
```

商品 Agent 因此不需要重新查一遍订单，也不需要从一段自然语言里猜型号，更不会顺手把保修和联动一起答了。

### 4. 黑板方案的边界

黑板解决的核心问题是：多个 Agent 需要读取同一份稳定状态（订单号、预算、已有设备、上游抽出的事实），而不需要各自传来传去。

**它擅长的事：**

- 请求级状态只维护一份，谁需要谁来读，不用反复转述。
- 可以按依赖关系裁剪——声明了依赖才给，没声明的看不见。
- 不共享 Thought 和完整 ReAct 轨迹，上下文不会无限膨胀。

**它不擅长的事：**

- 状态结构需要提前设计，不能随意往里塞东西。
- 写入错误会影响所有下游节点——黑板上写错一个字段，后面每个读它的 Agent 都会跟着错。
- 内存实现不支持跨进程恢复，进程一挂状态就没了。

但最关键的局限是这一条：**黑板只适合保存状态，不适合表达事件**。

> 打个比方：黑板像会议室的白板，上面写着“预算 2000 元”“在用型号 S10 Pro”，谁都可以随时看。但“请 IoT Agent 现在开始执行”这件事不该写在白板上——它是一条一次性的通知，IoT Agent 领了就该消失。如果也写在白板上，你还得自己判断这条通知有没有被处理过，白板就变成了一个没有消费语义的消息队列。

这就是 `MessageHub` 要解决的问题。

## 显式消息：MessageHub

### 1. 黑板解决不了的那一类信息

上一节说过，黑板只适合保存状态，不适合表达事件。编排层要传递的信息恰好分成这两种：

- **状态**：预算是 2000 元，在用型号是 S10 Pro，售后 Agent 已经跑完了——这些写上去之后谁都可以随时看，看多少遍内容都一样。
- **事件**：请商品 Agent 现在开始执行；售后结果已经就绪，聚合器可以来取了——这些是一次性通知，接收者领了就该消失。

黑板能很好地承载第一种。但第二种放在黑板上会出问题：你还得自己判断这条通知处理过没有、该不该再处理一次，黑板就变成了一个没有消费语义的消息队列。

`MessageHub` 就是为第二种信息准备的。它和黑板的关系不是替代，而是分工：

> 黑板回答现在有什么，消息总线回答接下来谁要做什么。

| 维度 | `AgenticScope`（黑板） | `MessageHub`（消息总线） |
|---|---|---|
| 表达对象 | 当前状态 | 已发生事件 |
| 读取方式 | 可以重复读取 | 读取后从邮箱移除 |
| 典型内容 | 预算、商品型号、Agent 结果 | 子任务、结果就绪、结果失败 |
| 生命周期 | 整个用户请求 | 到接收者消费为止 |

打个比方：黑板像会议室墙上的任务板，预算、订单号、已交付的结论都贴在上面，谁路过都能看；消息总线像每个人桌上的收件格，编排者往里投一封任务信，收件人取走之后格子就空了。

![](https://oss.open8gu.com/iShot_2026-07-26_21.42.06.png)

### 2. 消息结构

每条消息只需要说清楚四件事：谁发的、发给谁、什么类型、带了什么内容。

```java
public record AgentMessage(
        long id, String from, String to, Type type, String content) {

    public enum Type {
        // 编排者派给某个领域 Agent 的一次性子任务
        TASK,
        // 领域 Agent 已把可用结果写进黑板，通知聚合器来取
        RESULT_READY,
        // 领域 Agent 没能正常完成，聚合器据此生成降级终稿
        RESULT_FAILED
    }
}
```

三种类型覆盖了编排层和领域 Agent 之间的全部通信：

- `TASK`：编排者分配子任务——商品 Agent，去筛选预算内的换新候选；
- `RESULT_READY`：领域 Agent 完成后通知聚合器——售后结果已经写进黑板，可以来取了；
- `RESULT_FAILED`：领域 Agent 执行失败的通知——商品 Agent 没跑成功，准备降级。

第三种容易被省掉，但它是必须的。失败同样是一个需要投递的事实：聚合器收到它，才能生成一份写明缺口的降级终稿；如果把失败退化成异常往上抛，聚合器就只能崩溃，用户拿到的是一个堆栈而不是部分结果。

还有一个容易忽略的点：消息只携带定位信息和简短内容，不携带完整的 `AgentResult`。结果本身仍然保存在黑板上，消息只是一张通知单——“你要的东西放在黑板上了，自己去取”。这样消息体不会随着结果变长而膨胀。

### 3. 按接收者维护邮箱

实现非常直白：一个 `Map<String, ArrayDeque<AgentMessage>>`，键是接收者名字，值是它的邮箱队列。三个方法各管一件事：

- `send()`：自增一个 ID，把消息追加到对方队列尾部；
- `drain()`：把某个接收者的整个队列一次性摘走——取走即移除，同一条任务不会因为再读一次而重复出现；
- `pending()`：返回某个邮箱里还没被消费的条数。

方法一律 `synchronized`，和 `AgenticScope` 一样是请求内的单线程同步。

实现没什么可说的，真正容易写错的是怎么用它。

### 4. 发和收必须隔开

很多人第一次写会这么用：

```java
for (Step step : plan) {
    hub.send(ORCHESTRATOR, step.agentName(), TASK, step.task());
    AgentMessage task = hub.drain(step.agentName()).getFirst();
    // ...
}
```

发完下一行自己就取走了，消息从来没有真正在邮箱里待过——发送和消费之间隔了零行代码。这不是消息总线，这是一次 `String` 赋值套了两层壳。

正确的写法是先把所有任务一次性派发出去，再让各个 Agent 依次从自己的邮箱取：

```java
for (Step step : plan) {
    hub.send(ORCHESTRATOR, step.agentName(),
            AgentMessage.Type.TASK, step.task());
}

// 派发结束之后才轮到执行，每个 Agent 从自己的邮箱取
for (Step step : plan) {
    List<AgentMessage> inbox = hub.drain(step.agentName());
    // ...
}
```

分开之后有两个实实在在的好处：

- 发送和消费在时间上真正分离了，邮箱是一个真正的队列，而不是一个永远只有零或一条消息的摆设。
- 执行到一半中断时，还剩几条 `TASK` 没被消费，`pending()` 能直接给出确定答案，不需要靠推算。

> 判断消息总线有没有被滥用，看一个指标就够了：发送和消费之间是否存在真正的时间间隔。如果一条消息从投递到被消费，中间永远不会有第三方碰它，那这条消息就不需要存在。

### 5. 消息方案的边界

**它擅长的事：**

- 每条消息的发送者、接收者和类型一目了然，不需要从一整块黑板上自己找和自己相关的内容；
- 消费语义清楚——取走就是取走，不存在“这条通知我是不是已经处理过了”的疑问；
- 天然适合子任务分配、结果通知这类一次性事件；
- 接口足够简单，后续替换成 Kafka 或 RabbitMQ 时上层协议不用动。

**它不擅长的事：**

- 没有重试机制——消息投递失败后不会自动重发，需要调用方自己处理；
- 内存邮箱不支持进程崩溃恢复——进程一挂，还没被消费的消息就丢了；
- 不适合充当长期共享状态——那是黑板的活；
- 分布式场景下还要额外考虑消息顺序和幂等，本篇不涉及。

本篇不引入 Kafka、RabbitMQ，也不做异步线程池。先把通信语义讲清楚，后续替换底层实现时，上层协议才不会跟着推倒重写。

## 依赖 Handoff：只把必要结果交给下一个 Agent

前面两节分别解决了两类信息的存储问题：黑板保存请求级状态，消息总线传递一次性事件。但还缺一个关键角色——谁来把这两者串起来，按正确的顺序驱动每一步执行？

这就是 `HandoffRunner` 的职责。Handoff 可以翻译成任务交接，它关注的核心问题是：

> 当前步骤依赖谁，执行前必须拿到哪些结果？

### 1. 用执行计划声明依赖

```java
List<HandoffRunner.Step> plan = List.of(
        new HandoffRunner.Step(
                "afterSalesSpecialist",
                "查询订单 88231 的商品和签收日期，核验当前保修状态，"
                        + "并给出扫地机频繁掉线的处理建议"
        ),
        new HandoffRunner.Step(
                "productSpecialist",
                "结合上游给出的在用型号，筛选预算内的换新候选并说明规格差异",
                Set.of("afterSalesSpecialist")
        ),
        new HandoffRunner.Step(
                "iotSpecialist",
                "核对上游给出的候选商品与用户现有智能音箱的联动能力",
                Set.of("productSpecialist")
        )
);
```

每一步的第三个参数 `Set.of(...)` 就是依赖声明。商品 Agent 声明依赖售后，意味着 Runner 在给它组装上下文时，会把售后 Agent 抽出的 `facts`（型号、保修状态等）注入进去；IoT Agent 声明依赖商品，就能拿到商品 Agent 抽出的换新候选。没有声明依赖的售后 Agent 只能看到用户原始请求和共享约束，看不到任何其他 Agent 的结果。

整条链路是串行的：售后 → 商品 → IoT。

> 这里采用固定计划，是为了把通信机制单独讲清楚。上一篇的 `SupervisorAgent` 负责模型自主路由；固定 Handoff 和自主路由解决的是不同问题。

### 2. Runner 如何执行交接

先用大白话说一遍这个循环在做什么，再看代码。对于计划里的每一步，Runner 依次做六件事：

1. 从当前 Agent 的邮箱里取出任务（上一节 `MessageHub` 派发的那封）；
2. 调 `briefingFor()` 组装上下文——用户请求、共享约束、声明过的上游事实、本次子任务；
3. 调 Agent 执行，拿回 `AgentResult`；
4. 把结果发布到黑板上，供下游依赖时读取；
5. 往聚合器的邮箱投一封通知——成功投 `RESULT_READY`，失败投 `RESULT_FAILED`；
6. 如果这一步失败了，后面的步骤全部跳过，只登记不执行。

```java
for (Step step : plan) {
    // 上游失败后，剩下的步骤只登记不执行
    if (aborted) {
        skippedAgents.add(step.agentName());
        continue;
    }

    AgentMessage task = hub.drain(step.agentName()).getFirst();
    String briefing = scope.briefingFor(
            task.content(), step.dependencies(), step.projectUpstreamSummary());

    AgentResult result = executeSafely(step, briefing);
    scope.publish(result);

    hub.send(step.agentName(), Aggregator.RECIPIENT,
            result.completed()
                    ? AgentMessage.Type.RESULT_READY
                    : AgentMessage.Type.RESULT_FAILED,
            step.agentName());

    if (!result.completed()) {
        aborted = true;
    }
}

// 省略执行计划合法性检查和邮箱内容校验。
```

代码里有两处容易忽略的防御，都藏在 `executeSafely()` 里：第一，校验署名——`result.agentName()` 和被调度的 `step.agentName()` 对不上就直接判 `FAILED`，防止 Agent 冒名；第二，捕获 Agent 抛出的 `RuntimeException`，同样收敛成 `FAILED`。Agent 抛异常属于本次执行失败，不属于编排层协议出错，所以收敛成结果继续走完流程，而不是让整条链路带着堆栈退出。

整个循环保证了四条确定性约束：

1. `TASK` 只会投递给指定 Agent——不会派错人；
2. `briefingFor()` 只注入声明过的依赖结果——不会多给；
3. 结果署名必须和被调度的 Agent 一致——不会冒名；
4. 当前步骤失败后停止执行——不会让下游用不完整的输入继续跑。

> 模型仍然可能给出不理想的答案，但它不能自行改变执行计划，也不能读取未声明的其他 Agent 结果。

### 3. 用了共享黑板，隔离不就破了？

不会。Runner 每次执行都为当前 Agent 现场拼一份新的输入，只包含四样东西：

- 用户原始请求。
- 共享约束（订单号、预算、已有设备）。
- 声明过的上游事实（只有 `facts`，不含上游的 Thought 和完整 ReAct 轨迹）。
- 当前子任务，加上输出边界和行文统一。

别的一概不给。其他 Agent 的推理过程、与本次任务无关的历史消息，都不会出现在这份输入里。

> 打个比方：每个 Agent 拿到的不是一份所有人共写共读的群聊记录，而是 Runner 现场打印的一份任务简报——上面只有和你这次任务相关的内容。

下面这张图把三步交接完整走了一遍：

![](https://oss.open8gu.com/iShot_2026-07-26_21.42.07.png)

### 4. 三种上下文策略怎么选

走到这里，可以回头看一眼多 Agent 之间传递上下文的三种常见方案：

| 策略 | 适用场景 | 主要问题 |
|---|---|---|
| 编排者把完整信息写进每个 `task` | Agent 很少、任务短、依赖简单 | 容易重复转述，调用方负担大 |
| 所有 Agent 共享完整聊天历史 | 小型原型、上下文很短 | 无关信息增长快，隔离边界弱 |
| 黑板保存状态，按依赖交付事实 | 多步骤、有明确上下游关系 | 需要设计结果契约和依赖计划 |

没有一种方案在所有场景下都最好。本文选择第三种，是因为比特严选的售后、商品和 IoT 任务有明确的领域边界，也存在清楚的前后依赖。如果你的场景里 Agent 只有两三个、任务之间没有明确的上下游关系，第一种可能更省事。

## 结果聚合：确定性，而且可降级

前面的 Agent 都跑完后，还差最后一步：把各自的结果拼成一段完整的最终回复。

最直觉的做法是再调一次大模型帮你润色拼装。但这又多了一次模型调用——它可能漏掉某段结果，也可能改写原意。所以本篇使用确定性聚合器：不调模型，按通知顺序把各段 `summary` 拼起来就是终稿。

### 1. 两种错误必须走两条路

聚合器最重要的设计决策不是怎么拼，而是怎么处理出错的情况。聚合过程中会遇到两类问题，性质完全不同：

- **协议违约**：通知发错了收件人、同一个 Agent 通知了两次、通知指向的结果根本不在黑板上。这些都是编排层代码写错了，属于 bug，必须直接抛异常——让开发者看到堆栈、去修代码。
- **业务失败**：某个 Agent 跑满了步数没收敛、调工具连续报错。这是完全预期内的运行时结果，不是 bug。

早期版本把两者都写成 `throw`，后果很直接：真实链路上只要有一步 Agent 没跑成功，用户拿到的就是一个 `IllegalStateException` 堆栈——哪怕前面两位专家已经给出了完全可用的结果。

> 异常只留给协议违约，业务失败必须收敛成结果。

分开之后，代码走两条路。协议违约（`requireProtocol` 校验不过）直接抛异常；业务失败（`!result.completed()`）记下缺口、截断后续，把已完成的部分拼成一份降级终稿：

```java
for (AgentMessage notification : notifications) {
    // 收件人不对、同名通知两次、通知类型和黑板上的状态不一致，都是编排层写错了，抛
    String agentName = requireProtocol(notification, seen);
    AgentResult result = scope.resultOf(agentName).orElseThrow();

    // 而某个 Agent 没跑成功是预期内的业务结果，记下缺口然后跳出，不抛
    if (!result.completed()) {
        failedAgent = agentName;
        failureReason = result.summary();
        break;
    }

    completed.add(agentName);
    appendSection(answer, result.summary());
}

if (failedAgent != null) {
    appendGapNotice(answer, failedAgent, failureReason);
}

// 省略协议校验的具体条目、分段拼接与降级说明的文案，以及 Aggregation 的组装
```

各段之间只空一行。这里有个小坑：早期版本给每段加了 `- ` 前缀想凑成列表，但 `summary` 是多行 Markdown，前缀只作用在第一行，拼出来是个坏掉的列表。

### 2. 降级终稿长什么样

链路中断时，用户拿到的是这样一段话，而不是一个堆栈：

```text
订单 88231 购买的是比特 S10 Pro 扫地机，签收日 2026-06-22，
当前仍在保修期内，保修至 2027-06-21。

以上是已经完成的部分。productSpecialist 这一步没有完成
（ReAct 循环未正常收敛，终止原因 MAX_STEPS），依赖它的后续步骤已经停止，
剩余问题需要人工客服跟进，请不要把上面的内容当作完整答复。
```

售后那段已经完成、内容可用，直接给用户看；商品那步挂了，后面的 IoT 也跟着停了，聚合器在末尾写明哪一步没完成、为什么没完成。用户拿到的是部分结果加明确说明，而不是什么都没有。

![](https://oss.open8gu.com/iShot_2026-07-26_21.42.08.png)

### 3. 聚合器能保证什么，不能保证什么

确定性聚合不调模型，所以它能做硬保证的事情很明确：

- 只聚合已经发布到黑板上的结果，不可能凭空多出一段；
- 失败结果不会混进终稿正文，但会在末尾写明缺口；
- 同一结果不会重复聚合——`seen` 集合管着；
- 通知类型和黑板上的状态必须一致——不一致就是协议违约，直接抛；
- 输出顺序与结果通知顺序一致——先到先拼；
- 整个过程不新增任何 LLM 调用。

但它不能保证的事同样要说清楚：

- 每段 `summary` 里的业务事实一定正确——那是 Agent 和工具的事；
- 上游 Agent 没有遗漏该调的工具——那是人设和任务描述的事；
- 工具返回的数据本身没错——那是后端服务的事；
- 最终文案达到客服生产质量——那需要额外的评估流程。

> 如果生产系统确实需要 LLM 来做聚合润色，可以把它放在确定性拼接之后作为二次加工，但必须承认它是一个新的不确定节点。

### 4. 整条链路的正确性清单

下面这张表不只是聚合器的——它是整篇文章六个组件加在一起能提供的正确性边界。放在这里列一次，是因为聚合器是终稿发出去之前的最后一道关：前面哪一层没兜住，到这里就真的没兜住了。

| 正确性层级 | 本篇是否解决 | 说明 |
|---|---|---|
| 消息送给正确 Agent | 是 | 由接收者邮箱保证 |
| 只注入声明过的依赖 | 是 | 由 `briefingFor()` 保证 |
| 交接字段来自工具而非模型 | 是 | 由 `ToolFactExtractor` 保证 |
| 未收敛的执行不被当成成功 | 是 | 由 ReAct 终止状态保证 |
| 失败步骤不进入终稿 | 是 | 由状态检查保证 |
| 链路中断时仍有可用答复 | 是 | 由降级聚合保证 |
| 上游自然语言不泄漏给下游 | 是 | 由默认不给上游答复保证 |
| 每段只回答自己的子任务 | 部分 | 输出边界只是软约束，模型仍可能越界 |
| 各段标题层级和人称一致 | 部分 | 行文统一同样是软约束，做成确定性渲染才有保证 |
| 字段语义确实等于那条事实 | 否 | 规则表选错事实源，机制照抽不误，只能靠人评审 |
| 模型结论符合工具证据 | 否 | 需要结构化证据契约和评估 |
| 最终回复事实完全正确 | 否 | 需要业务校验、评估和人工兜底 |

## 跑通完整 Demo

### 1. Demo 改了什么

Demo 用的还是之前的 ReAct 专家和 `BitMallSpecialists`，接进交接链路时改了三处：

- 给每位专家配一张事实抽取规则表。
- 给售后专家加一把做确定性日期计算的 `checkWarranty`。
- 把共用的知识检索按领域切成三个互不重叠的实例——商品专家检索不到售后政策，售后专家检索不到 IoT 兼容说明。

第三处不是洁癖。上一篇里三位专家共用同一个知识库，谁都能查到全部内容。那么 `briefingFor()` “只给声明过的依赖”这条规则，在检索层就是形同虚设：上游没交给它的事实，它自己查一遍照样拿得到。

> 上下文隔离要是只做在消息通道上，工具会从背面把它绕开。

`main()` 里只有四步：`BitMallHandoffTeam.team(llmClient)` 把三位专家各配一张规则表接进交接链路，声明前面那份三步计划，交给 `HandoffRunner` 跑完，再把黑板和结果通知交给 `Aggregator` 拼出终稿。

完整代码在：

```text
src/main/java/com/nageoffer/ai/tinyagent/react/demo/MultiAgentContextDemo.java
```

### 2. 工具返回值也是你设计的上下文

前面几节讲的都是通信通道上的上下文控制：黑板该放什么、消息该发给谁、依赖该声明哪些。但跑 Demo 时会发现，还有一条路能绕开所有这些控制——工具的返回值。

工具返回会直接进入 Agent 的 ReAct 上下文，不经过 `briefingFor()` 的裁剪。**一个字段只要出现在上下文里，模型就倾向于对它表态**。这里踩了三类坑：

**第一类：字段本身就不该出现**。`compareProducts` 给两款商品各带了一个 `compatibilityDataIncluded: false`，本意是提醒模型别谈兼容，结果商品那一节写出了“两款商品的资料中均未包含兼容性数据，无法判断”——和 IoT 专家真正查过兼容知识库后给出的“明确支持联动”直接打架。删掉这个字段，商品 Agent 手里没有任何和兼容性沾边的东西，自然什么都不会写。

**第二类：字段命名暗示了不存在的可比性**。`compareProducts` 早期给 Lite 的 `control` 只写“机身按键”，给 Pro 写“App 远程控制”，两行并排进对比表，模型顺手就写成“Lite 不能通过 App 远程操控”——可紧挨着的 `appControlStatus` 明明是 `UNKNOWN`。根子在于 Pro 的 `control` 是一个完整答案，Lite 的不是，同名并排就等于宣称两者可比。改法是 Lite 那条不给 `control`，只留 `onDeviceOperation: 机身按键` 和 `appControlStatus: UNKNOWN`，能声明的和不能声明的各归各位。

**第三类：所有取值都相同的字段是零信息量的表态诱饵**。三款扫地机的规格里原本都挂着 `networkDependencyStatus: UNKNOWN`，三条一模一样。可连着两轮日志里，商品那一节都写出了“S10 Lite 无需联网、不依赖 App”——从一个统一的 `UNKNOWN` 里读出了一个具体结论。删掉就好了。

三类坑的根因都一样：**能用省略解决的，就不要用禁令解决**——这条原则在上面讲上游答复时用过一次，在工具 schema 上同样成立。

顺带提一处纯浪费。`searchKnowledge` 早先每条返回都把入参 `query` 原样回显在结果里。问题是模型经常换个近义词把同一篇文档查两遍，比如先查“扫地机 保修政策”，再查“保修期限 扫地机”，命中的是同一篇文档，返回的正文完全一样，但因为 `query` 字段不同，整条返回的字节就不同。这样一来，`ReActAgent` 想按“本轮已经出现过一模一样的返回”做去重就判不了。去掉 `query` 回显后，两次命中同一篇文档的返回变成字节相同，`ReActAgent` 就能把第二次命中替换成一句“与本轮第 N 圈的返回完全相同”，省掉重复的上下文预算。

### 3. 依赖给少了比给多了更难发现

前面两类坑都是上下文里多了不该有的东西。但反过来的问题更隐蔽：该给的没给，而且出错时你很难想到是依赖声明的问题。

回看 Demo 里的依赖链：售后 → 商品 → IoT。IoT Agent 只声明依赖商品 Agent，所以它能拿到的 `facts` 全是商品那一步抽出的候选型号和价格，拿不到售后那一步抽出的 `productName`——也就是用户现在正在用的是哪台机器。

某一轮日志里，IoT Agent 在核验联动能力时，把用户正在用、正在掉线的 S10 Pro 也列进了换新候选的表格。不算事实错误——那台机器确实存在、价格也在预算内——但读者会觉得莫名其妙：修不好就再买一台一模一样的？根子就在于 IoT Agent 手里没有“用户在用的是 S10 Pro”这个信息，它无从区分哪个是在用款、哪个是换新候选。

这个症状后来是在数据层解决的：在售目录里干脆不放在用同款，模型手里没有这个选项，自然不会选错。但依赖缺口本身还在——IoT 那一步至今看不见 `productName`，只是这一次的目录恰好让它无从出错。

要从依赖层面真正补上，有两个方向：

- 把 `afterSalesSpecialist` 也加进 IoT 的依赖声明，这样 IoT 能直接看到 `productName`。代价是售后的八条事实会一起塞进来，其中七条（订单号、签收日、保修状态……）和联动判断毫无关系。
- 由编排层把在用型号提升成共享约束，写进 `AgenticScope` 的约束区。这样只多一个字段，但要改编排逻辑。

> **能在数据层消掉的错误选项，就别留给上下文去劝阻**。但要分清自己消掉的是症状还是成因——数据层的修法只对这一份目录有效，换一份目录同样的缺口还会暴露。

### 4. 整条链路都查不出来的那一类问题

还有一类不一致，上面所有机制都拦不住。

售后那一节说“保修期内经检测确认为非人为硬件故障，可免费维修”，商品那一节紧接着推荐花 1899 元换 S11 Pro。两句话各自都有工具证据、都没越界，读者却会问：既然能免费修，为什么还要我掏钱？

这种矛盾只有站在终稿的位置上才看得见。确定性聚合器不调模型，没有全局收口能力。真要收口，得在拓扑层加一个专门读全部结果的收口 Agent，那是下一篇的事。

## 验收标准定在哪

别拿日志里一个字都挑不出毛病当完成标准。底下跑的是模型，措辞偏差只能压低概率，压不到零。真按这个标准调，你会一轮一轮往人设里加禁令、往数据里塞免责声明，最后收获一份没人看得懂的脚手架。

通信层的验收只管三件事：**该给的证据到了、不该给的没到、各段结论不互相打架**。这三件事都可以确定性检查。超出这个范围的措辞打磨属于单个 Agent 的 Prompt 工程，别放到通信层来拉锯。

## 总结

多智能体通信不是把所有 Agent 拉进同一个聊天群，而是为不同信息选择正确的载体。

| 你要解决的问题 | 使用组件 | 不要拿它做什么 |
|---|---|---|
| 一次请求内共享稳定状态 | `AgenticScope` | 不要保存完整 Thought |
| 给指定 Agent 发送一次性任务 | `MessageHub` | 不要充当长期数据库 |
| 生产可交接的结构化事实 | `ToolFactExtractor` | 不要让模型来填 |
| 把上游结果交给下游 | `HandoffRunner` | 不要隐式注入全部历史 |
| 汇总已经完成的结果 | `Aggregator` | 不要宣称已经验证业务事实 |

本篇只解决了通信机制，没有把它包装成生产级正确性保证。模型事实校验、分布式状态、并发执行、重试幂等和可观测性，属于更高一层的工程治理。

下一篇我们会在这些通信原语之上搭建串行、并行等多智能体拓扑，并放进完整的比特严选端到端任务中。
