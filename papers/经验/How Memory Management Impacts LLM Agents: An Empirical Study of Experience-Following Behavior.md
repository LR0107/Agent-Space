# How Memory Management Impacts LLM Agents: An Empirical Study of Experience-Following Behavior

**ACL 2026**

LLM Agent 的“记忆”到底应该怎么存、怎么删，以及记忆管理为什么会让 Agent 越用越好，或者反而越用越差。

从最基本的“添加记忆（addition）”和“删除记忆（deletion）”入手，分析长期运行过程中 Memory Bank 是怎样影响 Agent 行为的。

## 一、研究背景

现在很多 LLM Agent 都有一个类似“经验库”的东西，也就是 episodic memory（情景记忆）。

以后遇到类似任务，先从 Memory Bank 中检索几个最相似的历史经验，把它们当作 few-shot demonstration，再让 LLM 完成当前任务。论文把这种机制称为 episodic memory，并指出它与 semantic memory、procedural memory 不同，它主要保存的是具体任务经历。

问题在于：**Agent 自己产生的经验并不一定是正确的**。

所以 Memory Bank 和普通知识库有一个很大的区别：

> 普通知识库往往相对静态、干净；  
> Agent Memory 是不断变化的，而且里面的数据很多是 Agent 自己生成的，因此天然存在噪声和错误。

于是就产生一个关键问题：是不是记忆越多越好？

论文的答案很明确：**不是。**

真正关键的是：Memory 的数量 + Memory 的质量 + 判断 Memory 好坏的 evaluator。

---

## 二、研究方法

作者发现 LLM Agent 存在一种很明显的：Experience-Following Property——“经验跟随”现象

简单理解就是：

> 当前任务越像历史任务，就越容易模仿历史输出。

论文测量了两个东西：

**Input Similarity**

当前任务输入，与检索到的历史任务输入有多相似。

**Output Similarity**

当前 Agent 的执行结果，与历史经验中的执行结果有多相似。

结果发现：

**Input similarity ↑ → Output similarity 也明显 ↑。**

如果历史经验是对的：就形成正反馈、自我提升。

但如果历史经验是错的：就会形成负反馈、自我退化。

**Experience-Following 带来的两个问题**

**第一个问题：Error Propagation——错误传播**

假设 Memory 中存在一个错误经验：

**错误 Memory → 被检索 → Agent 模仿错误 → 得到新的错误结果 → 新错误又存入 Memory**

就会形成：

**错误经验 → 错误执行 → 新错误经验 → 更多错误执行……**

**第二个问题：Misaligned Experience Replay——错位经验重放**

有些 Memory：**当初是“正确经验”，但以后拿来指导别的任务时，未必有用。**

当以后检索到它时，它反而会误导 Agent。

> **历史经验本身可能没明显错误，但它和当前任务并不真正匹配，所以作为 demonstration 时会产生负作用。**

这也是为什么论文认为：**仅在 Memory 写入时判断“这条经验对不对”还不够。**

还应该观察：**它以后被别人检索使用之后，到底有没有帮助。**

---

**当前任务 → 检索相似 Memory → Agent 执行 → Evaluator 评价结果 → 决定是否加入 Memory / 删除旧 Memory → 下一个任务**

作者没有只在一个任务上实验，而是用了 **4 个 Agent**，以验证现象不是某个任务特有的。大部分实验采用 GPT-4o-mini 作为 backbone。

| Agent             | 任务           | 数据/规模               | 每次检索 |
| ----------------- | ------------ | ------------------- | ---- |
| **RegAgent**      | 人工构造的回归预测    | 4000 个执行样本          | 6 条  |
| **EHRAgent**      | 电子病历问答/代码生成  | MIMIC-III，2392 个任务  | 4 条  |
| **AgentDriver**   | 自动驾驶轨迹预测     | nuScenes，2000 个测试场景 | 1 条  |
| **CIC-IoT Agent** | IoT 网络流量攻击分类 | 8 类，1000 个样本        | 3 条  |

- **RegAgent**：是作者自己设计的一个**人工合成 Agent**，专门用于可控实验。它做一个简单的线性回归预测任务，方便作者精确控制 Memory 中的噪声、计算预测误差。
- **EHRAgent**：基于已有工作 **EHRAgent (Shi et al., 2024)** 的设定，作者按照该系统构建实验版本，用来处理电子健康记录任务。
- **AgentDriver**：基于已有工作 **AgentDriver (Mao et al., 2024)**，作者采用其 LLM 自动驾驶 Agent，并对原始检索流程做了一些简化。
- **CIC-IoT Agent**：作者根据 **CIC-IoT benchmark** 自己实现了一个网络流量检测 Agent。

### 实验一：Memory Addition

作者比较了四类策略。

**Fixed Memory**：Memory 固定，不再添加新经验。

**Add All**：所有新执行都存进去，不管好坏。

**Coarse Evaluator**：先自动判断经验质量，通过才存。作者用了 C1、C2、C3 三种不同质量的 evaluator。

- **C1：GPT-4o-mini**
- **C2：GPT-4.1-mini**
- **C3：GPT-4.1-mini + fine-tuning**

其中 C3 是作者用一个独立训练集上的 300 条正确的 judge 数据微调得到的。

**Strict Evaluator**：利用 Ground Truth 做严格判断，相当于一个理想的高质量 evaluator。

理想情况下由人判断“这次执行到底对不对”；但实验中不可能人工检查每一次执行，所以作者利用数据集中的 **Ground Truth** 来模拟这个严格的人类评审器。

指标：

- **SR / ACC ↑**：越高越好
- **Mem Size ↓**：最终 Memory 数量，越小存储成本越低

| Judge / Addition | RegAgent SR ↑ | Mem Size | EHRAgent ACC ↑ | Mem Size | AgentDriver SR ↑ | Mem Size | CIC-IoT ACC ↑ | Mem Size |
| ---------------- | ------------- | -------- | -------------- | -------- | ---------------- | -------- | ------------- | -------- |
| **Fixed**        | 67.53         | 100      | 16.75          | 100      | 40.11            | 180      | 71.50         | 50       |
| **Add All**      | 55.48         | 4100     | 13.05          | 2411     | 32.32            | 2125     | 59.90         | 1050     |
| **Coarse C1**    | 63.18         | 3511     | 26.19          | 1447     | 36.92            | 1161     | 74.00         | 1030     |
| **Coarse C2**    | 65.78         | 3347     | 32.21          | 1467     | 40.01            | 1119     | 68.80         | 936      |
| **Coarse C3**    | 67.35         | 3139     | 34.66          | 1094     | 47.37            | 1285     | 79.50         | 952      |
| **Strict**       | **70.95**     | 2938     | **38.50**      | 1012     | **51.00**        | 1178     | **85.40**     | 904      |

说明**盲目扩大 Memory 会因为错误经验积累而降低性能**。

而 Strict 在四个任务上都是最高的。

此外 Coarse Evaluator 越可靠，总体表现越好

所以 evaluator 实际上是整个 Memory 系统的“质量控制器”。

### 实验二：Memory Deletion

作者研究了三种删除方法。

Periodical-based deletion：一段时间内很少被检索的 Memory 就删掉。

相当于“长期没人用的东西忘掉”。

History-based deletion

不是看：“这条 Memory 当初是不是正确？”

而是看：“过去它被检索出来以后，后续任务表现怎么样？”

如果某条 Memory 已经被使用足够多次，而且**使用它时的平均下游效果一直不好**，就把它删掉。

这个设计正好用于解决前面的 **Misaligned Experience Replay**。

Combined deletion：把 Periodical 和 History-based 两种删除结合起来：

既删掉**长期没用的**，也删掉**经常使用但效果不好的**。

发现：Periodic deletion 很适合压缩 Memory。

它能大幅减少 Memory 数量，同时通常只造成比较小的性能损失

而在 Strict evaluator下，History-based deletion 对真实 Agent 很有效：

① Coarse Evaluator（C1）

| 删除策略         | RegAgent SR ↑ | Mem Size | EHRAgent ACC ↑ | Mem Size | AgentDriver SR ↑ | Mem Size | CIC-IoT ACC ↑ | Mem Size |
| ------------ | ------------- | -------- | -------------- | -------- | ---------------- | -------- | ------------- | -------- |
| **No del**   | 63.18         | 3511     | 25.91          | 1447     | 36.92            | 1161     | 74.00         | 1030     |
| **Period**   | 60.88         | 1012     | 26.65          | 338      | 36.38            | 426      | **78.10**     | 355      |
| **History**  | 62.10         | 3205     | **33.55**      | 1004     | 34.00            | 1019     | 73.70         | 952      |
| **Combined** | 59.32         | **951**  | 31.47          | **279**  | 35.62            | **372**  | 68.80         | **352**  |

② Strict Evaluator

| 删除策略         | RegAgent SR ↑ | Mem Size | EHRAgent ACC ↑ | Mem Size | AgentDriver SR ↑ | Mem Size | CIC-IoT ACC ↑ | Mem Size |
| ------------ | ------------- | -------- | -------------- | -------- | ---------------- | -------- | ------------- | -------- |
| **No del**   | **70.95**     | 2938     | 38.67          | 1012     | 51.00            | 1178     | 85.40         | 904      |
| **Period**   | 67.65         | 949      | 38.59          | 302      | 50.94            | 467      | 80.80         | 310      |
| **History**  | 69.80         | 2286     | 42.06          | 784      | **51.81**        | 846      | **89.60**     | 788      |
| **Combined** | 66.58         | **890**  | **42.34**      | **248**  | 49.97            | **323**  | 85.50         | **188**  |

History deletion 偏向“提高质量”

在 Strict evaluator 下：

- EHRAgent：38.67 → **42.06**
- AgentDriver：51.00 → **51.81**
- CIC-IoT：85.40 → **89.60**

说明删除那些“过去被使用后经常导致差结果”的 Memory，确实可以提高性能。

而 Combined deletion 偏向“压缩 Memory”。

例如：

- EHRAgent：1012 → **248**
- AgentDriver：1178 → **323**
- CIC-IoT：904 → **188**

它在性能和 Memory 大小之间做 trade-off。

作者进一步做了 Memory Size Matching。

在 RegAgent 中，同样只保留 1000 条 Memory：

Strict + History deletion：74.4

Strict only：72.8

说明 History-based deletion 的提升不单是 Memory 大小造成的，而是真的提高了平均 Memory 质量。

不过有一个重要前提：**Evaluator 必须足够可靠。**

如果 evaluator 本身比较差，History-based deletion 甚至可能出现“好经验删掉、坏经验留下”的情况；论文在 AgentDriver 上就观察到了这种现象。微调后的 evaluator 整体明显更稳定。

### 其他实验

**Task Distribution Shift**

现实中的任务不是一直同分布的。

比如自动驾驶 Agent：

一开始主要遇到城市道路，后来突然更多变成高速公路场景。

作者人为重新排列 EHRAgent 和 AgentDriver 的任务，制造明显的 task distribution shift。

实验发现在分布发生变化之后仍然能维持较稳定的性能。

**Memory Resource Constraint**

作者还限制 Memory Bank 的容量，比如 EHRAgent 只能保留 **100 条**、AgentDriver 只能保留 **180 条**。

结果发现：只要利用 evaluator 选择性保留真正高价值的经验，即使 Memory 容量很小，依然可以得到不错的长期表现。

**换成开源 Qwen 模型后的结果**

作者为了证明结果不是 GPT 特有的，又在 RegAgent 上测试：

- **Qwen3-32B**
- **Qwen3-14B**

对于 Addition：数字 = SR，括号 = Pearson correlation r

对于 History deletion：数字 = SR，括号 = 保留 Memory 的平均质量 / 删除 Memory 的平均质量

| 方法                | Qwen3-32B          | Qwen3-14B              |
| ----------------- | ------------------ | ---------------------- |
| **Add All**       | 56.9 (0.74)        | 55.4 (0.82)            |
| **Coarse**        | 67.9 (0.69)        | 65.7 (0.76)            |
| **Strict**        | **72.9 (0.72)**    | 72.9 (0.89)            |
| **Coarse + Hist** | 66.5 (0.61 / 1.01) | 64.4 (0.63 / 0.97)     |
| **Strict + Hist** | 68.4 (0.40 / 0.54) | **73.6 (0.41 / 0.51)** |

结果依然支持：

**Strict > Coarse > Add All**，

---

## 三、总结

### 创新点

**1.发现并系统量化了 Experience-Following**

作者发现：当前任务和检索到的历史任务越相似，Agent 当前输出也越倾向于模仿历史输出。

作者不是只做定性观察，而是分别计算：Input Similarity 和 Output Similarity，并发现二者具有很强相关性。

所以这篇论文实际上给很多 Agent Memory 工作补了一个比较重要的行为机制解释。

**2.提出“用未来任务表现反过来评价历史 Memory”的思路**

普通 Memory 管理一般是：

> 经验刚生成时判断一次好不好，然后决定存不存。

### 不足

**1.只研究了最基本的 Addition / Deletion**

这是作者明确承认的主要 limitation。

现实中的 Agent Memory 不只是：存 / 不存 / 删除。

还有很多更复杂的操作：

- Summarization
- Merging

- Memory rewriting

但这篇论文为了隔离变量，只研究 addition 和 deletion。

**2.Strict Evaluator 很大程度依赖 Ground Truth**

论文最好的结果大量来自 **Strict Evaluator**。

实验中用 ground truth 模拟 human/oracle evaluator。现实部署时恰恰经常不存在 Ground Truth。

比如真正的自动驾驶 Agent 在线运行：“现在这个决策到底是不是最优？”

通常并没有一个标准答案马上告诉你。

这也是它非常明显的现实落地 gap。


