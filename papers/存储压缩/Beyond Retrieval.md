# Beyond Retrieval: Embracing Compressive Memory in Real-World Long-Term Conversations

**COLING 2025**

核心思想是：不再依赖传统的检索记忆（Retrieval），而是把过去所有对话压缩成一份浓缩记忆（Compressive Memory），让大模型直接利用这份记忆进行对话。

## 一、研究背景

**长期对话的难题**

普通聊天机器人通常只能看见当前上下文。

但真实场景中，用户会：

- 今天聊工作
- 明天聊感情
- 一周后继续之前的话题

因此系统需要：

- 记住用户信息
- 记住过去发生的事情
- 理解用户和AI关系的变化
- 在未来对话中正确使用这些信息

**现有方法有什么问题？**

目前主流方法叫：Retrieval-based Memory（检索式记忆）

```
历史对话
   ↓
记忆提取
   ↓
存入数据库
   ↓
检索器搜索相关记忆
   ↓
生成回复
```

**问题1：检索不一定准**

检索模型可能找错内容。

结果回答变得奇怪。

**问题2：数据库越来越大**

长期使用后：

```
几百次对话
↓
几千条记忆
↓
检索越来越困难
```

维护成本高。

**问题3：真实场景更复杂**

很多研究数据：

- 人工构造
- 众包模拟

而现实聊天：

- 话题跳跃
- 口语化严重
- 信息杂乱

检索模型更容易失败。

---

## 二、研究方法

既然检索这么麻烦，不如直接把所有历史压缩成一份长期记忆。

这就是：Compressive Memory（压缩记忆）——COMEDY

**Step1：会话级记忆提取**

每个历史会话先生成一个摘要。

例如：

Session 1

用户：今天去射箭了，很开心。

提取：

```
事件：用户参加射箭活动
用户画像：喜欢射箭
```

论文称之为：Session-Level Memory。

**Step2：记忆压缩**

把所有会话摘要继续压缩（llm）。

最终形成：Compressive Memory

包含三部分：

1.用户画像

例如：

```
性格开朗
喜欢旅行
```

2.用户与AI关系

例如：

```
曾经分手
后来复合关系
逐渐升温
```

3.历史事件

例如：

```
一起讨论旅行
一起讨论运动
```

这样过去几十次对话就被浓缩成几百字。

**Step3：生成回复**

输入：

```
当前对话+Compressive Memory
```

输出：

```
最终回复
```

不需要：

- 向量数据库
- FAISS
- 检索器

完全依靠一个LLM完成。

**数据集 Dolphin**

作者从中国AI社交平台：X Eva官方平台收集真实用户聊天。

数据规模

共：**102,882条样本**

包含：

| 任务     | 训练样本   |
| ------ | ------ |
| 会话记忆提取 | 39,999 |
| 记忆压缩   | 30,695 |
| 记忆驱动回复 | 31,131 |

涉及：

- 3998个AI角色
- 大量真实用户对话

是当时最大的中文长期对话数据集之一。

---

## 三、实验设计

作者设计了三个任务。

Task 1：会话记忆提取

输入：

```
一段历史会话
```

输出：

```
事件总结 用户画像 机器人画像
```

评价指标：

- BLEU
- F1
- Distinct

Task 2：记忆压缩

输入：

```
多个Session Memory
```

输出：

```
Compressive Memory
```

评价：

- BLEU
- F1
- Distinct

Task 3：记忆驱动回复生成

输入：

```
当前对话+Compressive Memory
```

输出：

```
回复
```

### Baseline

作者比较了三大类方法。

1 Context-only

只看上下文：

- LLaMA2-7B
- LLaMA2-13B
- ChatGPT

2 Retrieval-based

传统检索方法：

```
记忆库+向量检索+生成器
```

包括：

- GPT4 Retrieval
- LLaMA Retrieval

3 Memory-related

先进记忆方法：

- MemoryBank
- Resum
- COMEDY

人工评测指标

评估5个维度：

| 指标           | 含义    |
| ------------ | ----- |
| Coherence    | 连贯性   |
| Consistency  | 一致性   |
| Memorability | 记忆能力  |
| Engagingness | 吸引力   |
| Humanness    | 像真人程度 |

COMEDY-GPT4：总体得分最高。

---

## 四、总结

### 创新点

提出Compressive Memory替代 Retrieval Memory

### 不足

压缩记忆是有损的。没有讨论：压缩以后丢失的信息怎么办？

没有真正证明比RAG更强：MemGPT、LongMem

---

### Q&A

**1.COMEDY 和传统 RAG 记忆系统最大的区别是什么？**

传统长期记忆系统采用“存储-检索-生成”的模式，需要维护一个 Memory Bank，并通过 Retriever 从大量历史记忆中找到相关内容。而 COMEDY 不再检索历史记忆，而是先把所有历史会话压缩成一个统一的 Compressive Memory，再直接用于回复生成。作者认为长期对话中更重要的是用户状态和关系状态，而不是精确找回某一句历史对话。

**2.为什么作者认为 Retrieval 不够好？**

作者认为 Retrieval 存在两个主要问题。第一，检索质量不稳定，Retriever 可能找不到真正相关的记忆；第二，随着会话数量增加，Memory Bank 会越来越大，维护和检索成本不断上升。

**3.第二步 Memory Compression 到底压缩了什么？**

压缩的对象不是原始对话，而是 Session-Level Memory。模型把多个会话摘要进一步整合成三类信息：用户画像（User Profile）、用户与 AI 的关系状态（Relationship Description）以及关键历史事件（Event Records）。本质上是把时间序列的对话记录转化为一个长期状态表示。

**4.COMEDY 有没有可能丢失信息？**

有。用户偏好和状态会随时间变化，而压缩后的记忆可能只保留最终结论，忽略变化过程。例如用户曾经喜欢某个游戏，后来不喜欢了，这种时间维度的信息可能会被压缩掉。论文没有专门讨论这一问题。
