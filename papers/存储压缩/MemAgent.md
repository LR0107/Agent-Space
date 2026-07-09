# MEMAGENT: Reshaping Long-Context LLM with Multi-Conv RL-Based Memory Agent

**ICLR 2026**

## 一.研究背景

### 1. 当前大模型的核心问题：上下文太长会崩

现在的大模型虽然支持：128K、1M甚至更长 context

但实际上：文本一旦极长，模型性能会明显下降。

论文里举了一个典型现象：

- 很多模型在几十万 token 后性能急剧下滑
- 到 896K token 时，有些模型准确率已经接近 0

而 MEMAGENT 在：8K context 训练、3.5M token 测试——仍然能保持较高性能。

### 2. 现有方法有哪些问题？

总结了三类主流方案：

**（1）扩大上下文窗口（最主流）**

例如：RoPE、YaRN、LongRoPE

问题：

- Attention 复杂度是 O(n²)
- 文本越长越慢
- 极长文本性能仍下降

**（2）稀疏注意力 / 线性注意力**

例如：Sparse Attention、Linear Attention、Mamba

问题：

- 需要修改 Transformer 结构
- 很难兼容已有模型
- 有些训练困难

**（3）压缩上下文**

例如：Memory module、Context compression、Retrieval

问题：

- 容易丢信息
- 泛化差
- 常需要外挂模块

---

## 二.研究方法

作者观察：人类读长文档时，并不会记住全部内容。

而是：保留重要信息、忽略冗余内容、动态更新笔记、最后根据笔记回答问题

### 整体架构

**Step 1：把长文切块（Chunk）**

假设有一本超长小说：

模型不会一次读完，而是逐块读取。

**Step 2：维护一个固定大小 Memory**

模型始终保留一个：Memory

例如1024 token

这个 memory 相当于：工作记忆、摘要

**Step 3：每读完一块，就更新 memory**

旧 memory + 当前 chunk = 生成新 memory

**Step 4：最后仅根据 memory 回答问题**

当所有 chunk 都读完后，模型不再看全文。

而只看：问题、最终 memory——来生成答案。

**为什么这种方法好？**

**关键：Memory 长度固定**

普通 Transformer：文本越长attention 越贵

复杂度：O(n²)

而 MEMAGENT：

每次只看：一个 chunk、一个固定 memory

所以：每一步计算量恒定

总复杂度：O(n)——即线性增长。

---

### 用 RL 训练 memory 更新策略

作者认为：什么该记住、什么该忘掉。其实是一个强化学习问题。

核心目标：最终答对问题

如果最后回答正确：说明memory 保留了关键内容

如果回答错误：说明memory 丢了重要信息或记了无关内容

于是 RL 会自动优化：如何写 memory

**Multi-Conversation DAPO**

这是论文提出的新 RL 算法。

为什么普通 RL 不够？

普通 RL：整个过程 = 一次对话

最后只优化：**Answer**

MEMAGENT 不是一次对话。

而是：

Chunk1 → 更新memory  
Chunk2 → 更新memory  
Chunk3 → 更新memory  
...  
最终回答

即一个长任务里有很多中间步骤。

作者把每次 memory 更新，看作一个独立 conversation。

把 reward 反向传播到所有 memory 更新步骤。

不是只训练最后答案，而是训练模型如何长期维护关键信息

---

### 工作流程

1.长文本切块

把超长文档拆成很多 chunk：Chunk1, Chunk2, Chunk3...

2.初始化 Memory

模型维护一个固定大小的 memory（类似笔记）

3.边读边更新 Memory

每次输入：当前 Chunk + 旧 Memory

模型输出：新 Memory

4.重复直到读完整篇文章

不断循环：读 chunk→ 更新 memory → 再读下一个

5.根据最终 Memory 回答问题

最后不再看全文，只看：Question + Final Memory

生成答案。

**RL 的作用**

负责训练：如何维护 memory

也就是学会：

- 什么该记
- 什么该忘
- 如何长期保留关键信息

---

## 三.实验设计

### 训练设置

**Backbone：**

- Qwen2.5-7B
- Qwen2.5-14B

训练 context： 8K

但测试最长达到：3.5M token

 **Benchmark** 

**（1）RULER-HQA**

超长 QA。

测试：长度外推能力

**（2）LongBench-QA**

真实世界 QA：新闻、小说、Wiki

测试：信息压缩、推理能力

**（3）NIAH（Needle in a Haystack）**

“草堆找针”。

超长文本里：只有极少关键信息。

测试：模型是否能一直记住关键事实

**（4）LongBench-SUM**

长文本摘要。

测试：memory 是否真的学会概括，而不是单纯 retrieval

### 实验结果

**1.超强长度外推**

训练：8K；测试：3.5M

性能只下降不到 10%。

**2.NIAH 表现非常强**

512K context 下：MEMAGENT 仍然超过 95%

而很多 baseline崩掉。

**3.RL 非常关键**

论文做了消融实验去掉 RL 后：

- 性能明显下降
- 长度增加时迅速退化

说明：memory能力不是prompt自动出现的，而是 RL 真正训练出来的。

---

## 四.总结

### 创新点

1.将长上下文问题，从“扩大 context window”转向“长期记忆管理”。

过去的方法依赖更长的 attention、更大的上下文窗口或稀疏注意力，本质是让模型“看到更多”。而 MEMAGENT 的核心在于让模型学会“记住什么、忘掉什么”。

把 memory 更新视为强化学习中的决策过程，用最终任务结果来训练模型的长期记忆能力。

2.固定 Memory 机制。模型不会一次读取全部长文本，而是把文本切成多个 chunk，边读边更新一个固定大小的 memory。由于 memory 长度始终不变，所以复杂度从O(n2) → O(n)

### 不足

1.如果模型在某一步错误地丢弃了重要信息，后续可能无法恢复。

2.它目前主要验证的是 QA 和摘要任务，对于代码生成、数学推理等需要保留大量细节的任务是否有效，还缺乏充分实验。

3.这种基于 RL 的训练方式成本较高，而且 memory 内部到底学到了什么、是否真正具备可解释性，也没有深入分析。

---

## Q&A

1.为什么“长期记忆管理”比单纯扩大上下文更重要？

因为上下文窗口继续增大会带来计算和推理成本上升，而且大量历史信息并不都重要。真实任务中，模型需要像人一样筛选关键信息。MEMAGENT 认为核心问题不是“能装多少”，而是“哪些值得长期保留”。

2.MEMAGENT 为什么使用强化学习？

作者把 memory 更新视为一个序列决策问题：模型需要不断决定“存什么、删什么、什么时候更新”。这些决策的好坏往往无法立即判断，而是要看最终任务是否成功，因此适合用强化学习，通过最终 reward 来优化长期记忆策略。

3.为什么作者不用 summary，而要用 RL memory？

因为普通 summary 通常是静态压缩。

但 MEMAGENT 的 memory 是 task-oriented 的，它通过 reward 学习“什么信息对最终 answer 最有帮助。

4.这篇文章最大的价值是什么？

我认为最大的价值不是简单“扩 context”，而是提出了一种新的 long-context 范式：

从“让模型记住所有 token”，转变为“让模型学会主动遗忘与压缩”。

这更接近人类处理长文本的方法，也可能是未来 scalable long-context LLM 的一个重要方向。
