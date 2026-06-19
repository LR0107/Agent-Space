# RECOMP: Improving Retrieval-Augmented LMs with Compression and Selective Augmentation

**ICLR 2024**

提出了一种新的检索增强框架 **RECOMP（Retrieve → Compress → Prepend）**，核心思想是：不要把检索到的大量文档直接塞给大语言模型，而是先压缩成简短摘要，再提供给模型。

## 一.研究背景

传统大语言模型（LLM）知识有限且可能过时。

因此研究者常采用：RAG

### 现有RAG的问题

论文指出直接把**检索结果**塞进Prompt有三个主要问题：

（1）计算成本高 

检索文档往往几百甚至上千token。

Transformer计算复杂度随序列长度增长很快。

因此推理成本显著增加。

**（2）长上下文利用率不高**

即使模型能读很长上下文，也不一定能有效利用。

已有研究发现：

> 模型经常忽略放在上下文中间的重要信息（Lost in the Middle）。

**（3）无关文档会干扰模型**

检索器并不总是正确。

很多检索结果：

- 不相关
- 部分相关
- 存在噪声

直接放进去反而降低性能。

---

## 二.研究方法

**Retrieve → Compress → Prepend**

作者设计了两种压缩器：

### 1.Extractive Compressor（抽取式压缩）

不生成新句子，模型只从检索文档里挑最有用的句子。

**模型结构**：**Dual Encoder**

分别编码：问题 x、候选句子 s

得到向量：Embedding(x)、Embedding(s)

计算：Similarity(x,s)

相似度越高，越可能有帮助

最后选 Top-K 句子

**如何训练？**

句子排序（Ranking）问题

**Step 1**

从检索文档中提取很多候选句子。

**Step 2**

逐个测试：

```
句子 + 问题
```

送给目标LLM。

观察答案质量。

**Step 3**

如果某句子能让LLM更容易得到正确答案：正样本

否则：负样本

注：不是人工标注，利用目标 LM 自动构造正负样本。

对于每个句子，检索得到若干句子单独拼接到问题前送入 LM：

**计算正确答案在LM看来有多大概率Score**

负样本怎么选？

不是把所有剩余句子都当负样本。

而是先设一个阈值 ε：

如果

Score(sj) + ε<Score(positive)

则认为它明显比正样本差，进入负样本池

**Step 4**

使用对比学习（Contrastive Learning）训练编码器。

---

### 2.Abstractive Compressor（生成式压缩）

它会自己写摘要。

难点：没有人工摘要标注。

**解决方案：知识蒸馏（Distillation）**

作者使用：GPT-3.5作为老师模型（Teacher）。

检索文档
   ↓
GPT-3.5生成摘要
   ↓
形成训练数据
   ↓
训练小模型(T5-large)

**Selective Augmentation（选择性增强）**

作者发现：有些检索根本没用

甚至会害模型。

如果 Compressor 判断：检索结果没用

直接输出：空字符串

这样：有价值 → 加摘要；没价值 → 不加

---

### 工作流程

用户提问：

```
When did Nissan Xterra stop production?
```

**Step1：检索**

Retriever 找到 5 篇相关维基百科文档：

总长度：700+ token

**Step2：压缩**

RECOMP 不直接把 700 token 给大模型。

而是用 Compressor 提炼：

```
The Nissan Xterra was discontinued in 2015.
```

只剩：20~50 token

**Step3：拼接 Prompt**

形成：

```
Summary:
The Nissan Xterra was discontinued in 2015.

Question:
When did Nissan Xterra stop production?
```

**Step4：LLM 生成答案**

模型直接输出：2015

**Compressor 内部怎么工作？**

论文提供两种方案：

**1.Extractive Compressor（抽取式）**

从文档里挑关键句：

选出：

```
Xterra production ended in 2015.
```

作为摘要。

特点：

- 简单
- 快
- 不容易产生幻觉

**2.Abstractive Compressor（生成式）**

先理解多个文档，再自己生成摘要：

```
文档A+文档B+文档C      
生成："Nissan discontinued the Xterra in 2015."
```

特点：

- 更短
- 信息融合能力更强
- 可能产生幻觉

```
传统RAG：
Query → Retrieve → LLM

RECOMP：
Query → Retrieve → Compress → LLM
```

也就是说，它并没有改 Retriever，也没有改 LLM，而是在两者之间增加了一个 Compressor（压缩器）

---

## 三.实验设计

### 实验1：语言建模（Language Modeling）

数据集：WikiText-103

模型：

- GPT-2（117M）
- GPT2-XL（1.5B）
- GPT-J（6B）

任务：预测下一个词。

评价指标：Perplexity (PPL)

越低越好。

### 实验2：开放域问答（Open-domain QA）

使用三个经典数据集：

- Natural Questions (NQ)
- TriviaQA
- HotpotQA

评价指标：

EM：答案完全一致

F1：答案部分匹配程度

### 实验结果

**结果1：压缩率极高**

Oracle实验压缩后只保留：6% ~ 13%的token。

性能反而比直接塞完整文档更好。

**结果2：训练后的压缩器效果很好**

Language Modeling

原始：141 token

压缩后：31 token

约25%长度，但性能损失极小

QA任务

NQ：660 token → 36 token

压缩率约5%

EM只下降约2个百分点。

**结果3：比普通摘要模型强很多**

比较了：

- GPT-3.5摘要
- T5摘要
- 自己的RECOMP

结果：RECOMP生成的摘要更有利于最终任务。

因为普通摘要关注：给人看

而RECOMP关注：帮助LM答题

---

## 四.总结

### 创新点

压缩的目标不是保留文档信息，而是保留对下游任务最有帮助的信息。训练时直接用 LM 的表现来定义正负样本。

### 不足

训练成本高

抽取式训练：

100个候选句子 → 100次LM打分

才能找到Positive Sentence

---

## Q&A

**1.RECOMP 和普通 RAG 的区别是什么？**

RECOMP 本质上是在传统 RAG 的检索器和语言模型之间增加了一个压缩模块。传统 RAG 会直接将检索到的多个文档拼接到 Prompt 中，而 RECOMP 会先把这些文档压缩成少量关键句子或摘要，再输入语言模型。这样可以减少 Token 数量、降低推理成本，同时减少无关信息对模型的干扰。

**2.为什么不用普通摘要模型代替？**

因为普通摘要模型关注的是人类可读性和信息完整性，而不一定有利于语言模型完成任务。论文实验表明，即使 GPT-3.5 或 T5 能生成较好的摘要，也不一定能带来最佳问答效果。RECOMP 的训练目标直接面向下游任务，因此效果更好。
