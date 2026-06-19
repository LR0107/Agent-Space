# ROUTELLM: Learning to Route LLMs with Preference Data

**ICLR 2025**

**核心问题**：如何让系统自动决定一个问题应该交给大模型还是小模型处理，从而在保证效果的同时，大幅降低成本。

## 一.研究背景

GPT-4 回答质量高，但推理成本很高；小模型便宜很多，但复杂问题容易答错

如果：

- 所有请求都发给 GPT-4 → 太贵 
- 所有请求都发给小模型 → 效果差 

所以作者提出：能不能先判断这个问题难不难，再决定用哪个模型？

这就叫：LLM Routing

目标是：用尽可能少的大模型调用，获得接近 GPT-4 的效果。

论文里提到：Claude Haiku 比 Opus 便宜 50 倍以上

因此：如果路由做得好，成本能大幅下降。

---

## 二.研究方法

训练一个问题难度判断器

它预测：“这个问题是不是必须用 GPT-4 才能答好？”

如果：

- 小模型足够 → 用小模型
- 小模型可能答不好 → 用大模型

用偏好数据（Preference Data）训练路由器而不是人工定义规则。

### 1. 什么是 Preference Data？

例如：

用户提问：解释相对论

系统给出：

- GPT-4 的回答
- Mixtral 的回答

用户投票：GPT-4 更好

于是得到：这个问题 → 强模型胜出

论文把这种数据叫：人类偏好数据

来源是： Chatbot Arena 数据集

### 2. Router 的数学定义

作者把路由过程拆成两部分：

**（1）Win Prediction Model**

预测：P(强模型赢 | 问题q)

也就是：强模型比弱模型更好的概率。

例如：

| 问题       | GPT-4赢的概率 |
| -------- | --------- |
| “1+1等于几” | 5%        |
| “证明黎曼猜想” | 99%       |

**（2）Threshold（阈值）**

设定一个阈值 α：

- 如果概率 < α → 小模型
- 如果概率 ≥ α → 大模型

```
问题 → 预测需不需要大模型 → 决定调用谁
```

---

**作者研究了 4 种 Router。**

### 方法1：Similarity-weighted Ranking（相似度排序）

思想：找训练集中“类似的问题”。

例如：

用户问：解释 Transformer

系统会找：“解释 Attention”“Transformer 如何工作”这些相似问题。

优点：

- 不需要训练
- 很简单

缺点：

- 推理速度较慢
- 要做向量检索

### 方法2：Matrix Factorization（矩阵分解）

这是论文效果最好的方法之一。

思想：类似推荐系统

作者认为：“模型能力”和“问题难度”存在一个隐含匹配关系。

于是：

- 给模型一个 embedding
- 给问题一个 embedding
- 学习两者匹配程度

例如：GPT-4 更擅长：数学、复杂推理；Mixtral 更擅长：简单聊天、普通问答

模型会自动学到这种结构。

### 方法3：BERT 分类器

输入问题  
↓  
BERT  
↓  
预测： 是否需要强模型

### 方法4：Causal LLM Classifier

作者用**Llama3-8B**来当 Router

也就是这个问题适合强模型还是弱模型让 LLM 自己判断。

---

### 数据增强

作者发现： 原始 Arena 数据不够

因为：

- 某些模型对比数据太少
- 某些领域（数学）数据不足

所以做了数据增强

**方法A：Golden Label 数据**

比如：MMLU 这种标准考试题有标准答案。

如果：

- GPT-4 做对
- Mixtral 做错

则：这个问题 → 强模型更适合

**方法B：GPT-4 Judge**

作者还用 GPT-4 当裁判。

流程：

```
问题 → GPT-4 回答 、Mixtral 回答 → GPT-4 评判谁更好 → 生成偏好标签
```

他们生成了12 万条数据，花了约700 美元。

---

### 工作流程

1. 收集偏好数据（Preference Data）  

2. 训练 Router（概率分类器）  

3. 输入用户 query  

4. Router 预测：  P(strong wins | q)   
   
   Router 可采用：SW Ranking、Matrix Factorization 、BERT 、LLM classifier

5. 与阈值 α 比较  

6. 决定路由：  
- strong model  
- weak model  
7. 返回最终回答

---

### 解释

这篇论文本质上是在做：一个二分类问题

即：这个query该不该调用大模型？

 Router 怎么学会判断？

靠 Preference Data——这是监督信号、本质上是标签（label）

数学定义其实只是：把这个分类问题形式化

Router 本质上是一个概率分类器。

四种 Router 是四种不同的概率分类器，只是实现方式不同。

---

## 三.实验设计

### 实验环境

**模型**

| 强模型   | 弱模型          |
| ----- | ------------ |
| GPT-4 | Mixtral-8x7B |

**benchmark**

| Benchmark | 类型   |
| --------- | ---- |
| MT-Bench  | 开放聊天 |
| MMLU      | 知识考试 |
| GSM8K     | 数学推理 |

**评价指标**

**1.APGR（Average Performance Gap Recovered）**

意思：Router 恢复了多少 GPT-4 的能力。

例如：小模型 = 60分、GPT-4 = 100分、Router = 90分

则Router 恢复了：(90-60)/(100-60)=75%

**2.CPT（Call Performance Threshold）**

意思：达到某个效果，需要调用多少次 GPT-4。

例如：达到 GPT-4 80%效果只需调用 30% GPT-4

只有30%的请求会调用 GPT-4，其余70% 的请求会走小模型。

CPT 越低越好

---

### 实验结果

1.成本降低 2~3.6 倍，但效果接近 GPT-4。

2.数据增强非常有效

没有增强时：BERT 几乎和随机差不多；增强后：效果涨了

说明：Router 非常依赖训练数据质量。

3.可以泛化到新模型

说明：Router 学到的是问题难度，而不是某个特定模型。

---

## 四.总结

### 创新点

1.把 LLM Routing 转化成 Preference Learning 问题（最大）。

以前很多 routing 方法是： heuristic（经验规则），Router 变成一个标准监督学习问题。

2.数据增强对 Router 极其有效

Router 极度依赖数据分布。

### 不足

1.本质上仍是 binary routing，但真实通常是多层级 routing。

而binary routing 比多路 routing 简单很多。

因为只需要学：谁更好；而不是：谁最好。

2.Preference Signal 本身不稳定

同一个 query，不同人会：偏好不同风格、不同长度

尤其**GPT-4-as-a-judge**本身也有 bias。

---

## Q&A

1.为什么先预测概率，再分类？

因为概率更灵活。Router 可以通过调整 threshold α，动态控制成本、性能。α 本质上是成本控制旋钮。

2.数据增强为什么这么重要？

因为原始 Arena 数据分布不均衡。例如聊天数据多、数学数据少

导致Router 在：MMLU、GSM8K表现很差。

3.CPT 指标是什么意思？

CPT：达到某个性能目标，需要多少 strong model 调用。

例如CPT(80%) = 30%——表示想恢复 strong/weak 之间 80% 的性能差距，只需要 30% query 调用 GPT-4。

4.为什么 Matrix Factorization 效果这么好？

因为routing 本质上类似推荐系统。

论文认为：query 有难度向量、model 有能力向量

两者在隐空间中匹配。

因此Matrix Factorization 能很好建模： 什么模型擅长什么 query。

5.Router 学的是“query 难度”吗？

Router 学的是 strong model 的相对优势。

即：strong model 是否明显优于 weak model

而不是：问题绝对难不难

例如：有的问题两个模型都会，有的问题两个模型都不会

这种情况下P都不会高。
