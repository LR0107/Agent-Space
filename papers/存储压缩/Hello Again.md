# Hello Again! LLM-powered Personalized Agent for Long-term Dialogue

**NAACL 2025**

提出了一个面向长期对话（Long-term Dialogue）的智能体框架 LD-Agent（Long-term Dialogue Agent），旨在解决当前聊天机器人只能记住短期对话、难以维持长期关系和个性一致性的问题。

## 一、研究背景

虽然 ChatGPT 等大语言模型已经能进行流畅对话，但大多数研究仍然集中在：

- 单次会话（single session）
- 2~15轮短对话
- 当前上下文中的信息利用

现实中的长期陪伴型AI却需要：

（1）长期记忆（Long-term Memory）

能够记住：

- 曾经讨论过什么

- 用户过去的经历

（2）人格一致性（Persona Consistency）

需要记住用户是谁

例如：

```
用户：我是计算机专业学生 喜欢游泳
```

以后还能利用这些信息。

### 现有方法的问题

作者指出现有研究通常只关注：

 路线1：记忆

只研究历史事件存储

例如：Memory Bank

路线2：Persona

只研究人格建模

例如：PersonaChat

很少有工作同时解决：

```
长期记忆+用户人格+Agent人格
```

而且很多方法：

- 依赖特定模型
- 泛化能力弱
- 换个模型就不能用了

因此作者提出：能否设计一个与模型无关（Model-Agnostic）的长期对话Agent框架？

---

## 二、研究方法

整体结构包含三个模块对应三个能力。

### 模块1：事件记忆模块（Event Memory）

作用：记住过去发生的事情

作者把记忆分成：

**（1）长期记忆 Long-term Memory**

保存：历史会话摘要

是一个专门微调过的 LLM 

训练数据使用：DialogSum——作者改造后然后微调（LoRA）

例如：

原始对话：

```
A：我没选上游泳课
B：没关系，下周我教你
```

系统自动总结：

```
B答应下周教A游泳
```

保存进记忆库。

论文称为：Event Summary

（2）短期记忆 Short-term Memory

保存当前会话内容：当前聊天窗口

例如：最近几十轮对话

**记忆检索机制**

作者设计三种因素共同检索：

① 语义相似度：意思是否接近（MiniLM）

② Topic重叠

提取名词：比较主题重合度。

作者发现：仅靠Embedding检索经常找错记忆。

加入Topic后更准确。

 ③ 时间衰减

最近发生的事情更重要：λt​=e−t/τ

时间越久：权重越低

最终得分：soverall​=λt​(ssemantic​+stopic​)——排序

**经验设计**

### 模块2：人格模块（Persona Module）

作用：持续更新人物画像

包括：

**用户画像**

例如：

```
我是学生
喜欢骑行
喜欢游泳
```

**Agent画像**

例如：

```
我是机械高手
乐于助人
```

系统不断从对话中抽取：Persona

保存到：Persona Bank

**人格抽取方法**

作者训练了一个：Persona Extractor

输入：

```
我特别喜欢骑自行车
```

输出：

```
喜欢骑行
```

对于没有人格信息的话：今天天气不错

输出：No Trait

从 MSC 数据集构造了一个 Persona Extraction Dataset。

数据规模：

- Train: 26605
- Valid: 17660

然后使用：

```
ChatGLM+LoRA+Instruction Tuning
```

进行训练。

### 模块3：回复生成模块

最后整合：

```
当前对话+历史记忆+用户人格+Agent人格
```

生成回答。

微调模型，训练数据：MSC + CC

---

### 工作流程

① 检索历史记忆（Memory）

用户发来一句话后，系统先从长期记忆库（外部存储）中检索相关历史事件：

检索依据：

- 语义相似度（Embedding + Cosine）
- Topic重叠度
- 时间衰减

② 更新 Persona

系统从当前对话中抽取人物特征：

同时维护：

- User Persona

- Agent Persona
  
  ③ 生成回复

将以下信息一起送给 Generator：

```
当前上下文+历史记忆+用户Persona+Agent Persona
```

生成更符合历史和人设的回答。

 ④ 写回长期记忆

当前会话结束后：

```
本轮对话
    ↓
Event Summarizer
    ↓
事件摘要
    ↓
Long-term Memory
```

以后检索时还能用到。

---

### 三、实验设计

### 实验环境

**数据集**

两个经典长期对话数据集：

1.MSC

特点：

- 5个会话
- 大约50轮对话
- 真人标注
- 有Persona信息

2.CC

Conversation Chronicles

特点：

- 5个会话
- 时间跨度可达几年
- 由ChatGPT生成

**对比模型**

作者选择：

LLM

- ChatGPT
- ChatGLM

传统模型

- BlenderBot
- BART

SOTA：HAHT

**评价指标**

自动评价：

- BLEU
- ROUGE-L
- METEOR

人格抽取：Accuracy——专门评价Persona Extractor

测试集里有真实回复（Ground Truth）。

人工评价：

- Coherence（连贯性）
- Fluency（流畅度）
- Engagingness（吸引力）

找了8个评测者让他们和机器人长期聊天。然后打分。

### 实验结果

结果1：长期对话效果显著提升

在MSC和CC上：

加入LD-Agent后：几乎所有指标都有明显提升。

结果2：记忆模块最重要

消融实验：

```
Memory+ User Persona+ Agent Persona
```

发现：Memory贡献最大

说明长期记忆是长期对话最关键因素。

结果3：跨领域泛化能力强

作者专门做了：MSC训练→CC测试

以及：CC训练→MSC测试

结果性能下降很少。

说明：LD-Agent不是只记住数据集而是真学会了长期对话机制。

---

## 四、总结

### 创新点

同时建模 Event Memory 和 Persona

模块化 Agent 设计：Memory ≠ Model，不是把记忆塞进模型参数。

### 不足

Retriever太弱：检索分数完全手工设计。（替换为Learn-to-Rank让模型自己学）

**利用训练数据（LLM自动标注），训练一个排序模型**

Persona只是属性标签。

没有 Memory Importance，所有事件几乎平等。

---

### Q&A

**1.这篇论文最大的创新点是什么？**

我认为最大的创新不是某个具体算法，而是系统性地将长期事件记忆和双向Persona建模整合到一个统一Agent框架中。过去很多工作只关注Memory或者Persona，而LD-Agent同时维护历史事件、用户Persona和Agent Persona。

**2.Topic Retrieval和BM25有什么区别？**

这篇论文的Topic Retrieval并不是BM25。作者只是从对话和事件摘要中提取名词集合，然后计算集合重叠率。没有TF、IDF和倒排索引等BM25机制，因此本质上是一个简单的主题重叠特征，而不是传统信息检索模型。

**3.Persona Extractor是怎么训练的？**

作者利用MSC数据集中的Persona标注构建Persona Extraction Dataset，然后使用LoRA对基础模型进行指令微调。输入是用户的一句话，输出是对应的人格特征。如果一句话不包含人格信息，则输出“No Trait”。实验表明微调版Extractor明显优于纯Prompt+CoT方案。

**4.为什么说它是Model-Agnostic？**

因为Memory模块和Persona模块与具体LLM解耦。框架负责记忆管理和Persona管理，而底层Generator可以替换成不同模型。论文实验中分别在ChatGPT、ChatGLM、BART上进行了验证，都取得性能提升，因此证明其具有较好的模型迁移能力。

**5.如果让你改进这篇工作，你会怎么做？**

我会重点改进Memory系统。

首先引入Importance Score，对重要事件长期保留，对低价值事件自动遗忘；其次引入Reflection模块，把多个事件总结成更高层知识；最后用Learn-to-Rank替代当前手工设计的检索公式，让Retriever能够从数据中学习不同特征的重要性。
