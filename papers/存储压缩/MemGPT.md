# MemGPT: Towards LLMs as Operating Systems

**UC Berkeley 2024**

## 一.研究背景

虽然大语言模型（LLM）已经在聊天、问答、代码生成等任务上表现非常强，但它们有一个关键限制： Context Window（上下文窗口）有限

也就是说：模型一次只能看到有限长度的文本。

但现实任务往往远远超过这个长度。因此，传统 LLM 会出现：

- 忘记之前内容
- 无法长期记忆
- 长文档处理失败
- 对话一致性下降

论文认为简单扩大上下文并不是理想方案。

**（1）计算成本太高**

Transformer 的注意力机制复杂度：O(n2)

上下文越长：

- 显存消耗越大
- 推理速度越慢
- 成本急剧增加

**（2）长上下文利用效率不高**

论文引用了 “Lost in the Middle” 的研究：即使模型拥有超长上下文，模型也不一定能有效利用中间部分的信息。

因此作者提出：不应该只扩大模型上下文，而应该设计“记忆管理系统”。

作者借鉴了：Operating System的虚拟内存机制

| 操作系统 | 作用  |
| ---- | --- |
| RAM  | 快但小 |
| 硬盘   | 慢但大 |

操作系统会：把暂时不用的数据放到硬盘、需要时再调回内存

这个过程叫：Paging（分页机制）

论文提出：能否让 LLM 也像操作系统一样管理记忆？

于是作者设计了： MemGPT

目标是：**让固定上下文的 LLM，获得“无限上下文”的效果。**

---

## 二.研究方法

### 1. 分层记忆结构（Hierarchical Memory）

论文设计了类似操作系统的记忆层次结构。

整个系统分成两类记忆：

| 记忆类型             | 类比         |
| ---------------- | ---------- |
| Main Context     | 内存（RAM）    |
| External Context | 外部存储（Disk） |

### 2. Main Context（主上下文）

即模型当前真正能看到的内容。

又被划分为三部分：

| 部分                  | 功能     |
| ------------------- | ------ |
| System Instructions | 系统规则   |
| Working Context     | 长期重要记忆 |
| FIFO Queue          | 最近对话历史 |

**（1）Working Context**

用于存储：

- 用户偏好
- 人物关系
- 长期事实
- 当前任务状态

模型可以主动修改。

论文中的例子：

```
Boyfriend named James  -> Ex-boyfriend named James
```

说明模型会更新记忆。

**（2）FIFO Queue**

用于保存：

- 最近聊天记录
- 系统消息
- 函数调用结果

采用先进先出（FIFO）机制。

当上下文过长时：旧内容会被移出当前窗口。

**普通消息队列**

**3.External Context（外部记忆）**

这些内容平时不在上下文中。

只有需要时：模型才会主动搜索并读取。

（1）Recall Storage

保存：

- 历史聊天
- 消息记录
- 过去 interaction

（2）Archival Storage

保存：

- 长文档
- Wikipedia
- 外部数据库

```
Main Context（主存）-> Recall Storage（历史记忆）-> Archival Storage（长期外存）
```

不同层：容量不同、访问方式不同、生命周期不同

并通过：search、append、flush——实现层间数据迁移。

**4. Queue Manager（队列管理器）**

这是 MemGPT 的“内存调度器”。

作用包括：

- 检测上下文是否即将溢出
- 自动删除旧消息
- 压缩历史信息
- 提醒模型进行记忆管理

当 token 快满时：系统会提醒模型“内存不够了。”

然后模型自己决定：保存什么、删除什么、检索什么

```
if token_count > threshold:  
warn_llm()
```

**5. Function Calling（函数调用）**

利用 GPT-4 已有的 Tool Use 能力

模型不仅生成文本,还能够主动调用函数。

例如：搜索记忆、更新记忆

这样模型就具备了：主动管理记忆的能力

**6. Function Chaining（函数链）**

模型还能连续执行多个函数调用。

例如：

1. 搜索数据库
2. 翻页读取
3. 再次搜索
4. 汇总结果

这使模型能够完成：多步检索与推理

**7. Event-driven（事件驱动机制）**

论文提出：任何事件都能触发模型运行。

包括：

- 用户输入
- 文件上传
- 系统警告
- 定时任务

这类似于：操作系统中的中断机制（Interrupt）

---

### 工作流程

**1. 用户发送消息**

消息先进入当前上下文（FIFO Queue）。

**2. 系统检测内存压力**

如果上下文快满，系统会发出：System Alert: Memory Pressure

提醒模型整理记忆。

**3. 模型自主管理记忆**

模型会主动调用函数：

- 写入长期记忆
- 修改记忆
- 搜索历史信息

**4. 从外部记忆中检索信息**

如果当前上下文没有答案，

模型会去：历史聊天、文档数据库、向量存储

中搜索相关内容。

实际上是一种主动 RAG

**5. 重新加载并回答**

找到相关内容后：

系统把结果重新放入上下文，

模型再生成最终回答。

---

## 三.实验设计

### 1. 长对话实验

数据集： Multi-Session Chat（MSC）

**（1）Deep Memory Retrieval（深度记忆检索）**

目标：测试模型是否能记住很久以前的聊天内容。

评价指标：

- Accuracy（正确率）
- ROUGE-L（文本匹配度）

同时还用 GPT-4 作为 “LLM Judge” 判断回答是否正确。

---

**（2）Conversation Opener**

目标：测试模型能否利用长期记忆生成更自然、更个性化的开场白。

例如：

如果记得用户喜欢冲浪，就主动聊冲浪相关话题。

对比实验（Baseline）

对比对象：

- GPT-3.5
- GPT-4
- GPT-4 Turbo

以及：是否使用 MemGPT

**实验结果**

| 模型             | Accuracy |
| -------------- | -------- |
| GPT-4          | 32.1%    |
| GPT-4 + MemGPT | 92.5%    |

---

### 2. 文档分析实验（Document Analysis）

**（1）Multi-document QA（多文档问答）**

目标：测试模型是否能处理超长文档。

实验流程：

1. 从 Wikipedia 中检索相关文档
2. 将文档提供给模型
3. 模型回答问题

普通 GPT 只能读取有限文档；MemGPT 可以不断搜索、分页读取更多文档。

随着文档数量增加：普通 GPT 的准确率下降；MemGPT 基本保持稳定。

**（2）Nested KV Retrieval（多跳检索）**

目标：测试模型是否具备多步检索能力。

 主要测试：MemGPT 是否能进行链式记忆检索与推理。

普通 GPT超过 3 层后几乎失败；MemGPT依然能稳定完成任务

---

## 四.总结

系统是作者自己开发的。  
MemGPT 不是训练了一个新模型，而是在现有 LLM 外面加了一层操作系统式的记忆管理框架。

外部记忆太大，不可能一次全塞进 context。所以 MemGPT 采用：分页读取。通过 search(page=n) 逐页检索外部记忆。

### 创新点

MemGPT 第一次提出：让 LLM 自己管理 memory

它让 LLM 变成：持续运行的 Agent

MemGPT 不会一次把所有 memory 塞进 context，而是逐页读取。它真正模拟了操作系统 Paging，从而实现**有限上下文 + 无限外部记忆**。

### 不足

1.它并没有真正解决长期记忆，没有真正解决该存什么

模型可能：保存垃圾信息、忘掉关键事实、写入错误记忆

2.MemGPT 的能力很大程度来自 GPT-4 本身，而不是 memory system。

---

### Q&A

**1.MemGPT 和传统 RAG 有什么区别？**

传统 RAG 的检索流程是固定的，即 Retriever 先检索，再把 Top-k 文档送给 LLM，LLM 本身并不参与检索决策。而 MemGPT 属于主动检索，LLM 会自主决定是否检索、检索什么、是否继续翻页搜索，因此它更像一个持续运行的 Agent，而不是一次性的问答系统。

**2.为什么作者要借鉴操作系统（OS）的思想？**

作者认为 LLM 的 context window 很像计算机中的 RAM：访问速度快，但容量有限，而真实世界中的数据规模远远超过 context window。因此论文借鉴了操作系统中的虚拟内存（virtual memory）和分页（paging）思想，把不常用的信息放在外部 memory，需要时再调回 context 中，从而缓解上下文长度限制问题。

**3.MemGPT 需要重新训练模型吗？**

不需要。MemGPT 主要是一种推理阶段的系统架构设计。作者没有训练新的 Transformer，也没有修改 attention 机制，而是利用现有 GPT-4 的 function calling、tool use 和 prompt engineering 能力实现 memory 管理。

**4.论文中的“分层记忆”体现在哪里？**

论文将 memory 明确划分为多个层次，包括 Main Context、Recall Storage 和 Archival Storage。其中 Main Context保存当前最重要的信息；Recall Storage 用于保存历史聊天；Archival Storage 用于长期文档存储。不同 memory 层容量不同、访问方式不同，并通过 search、flush、append 等操作实现层间迁移。

**5.为什么 GPT-4 上的 MemGPT 效果明显优于 GPT-3.5？**

因为 MemGPT 非常依赖 Tool Use 和 Function Calling 能力。GPT-4 在多步推理、函数调用 上更加稳定，而 GPT-3.5 经常出现不会继续 search、提前停止或者 retrieval 出错的问题，因此整体效果明显较差。

**6.你认为这篇论文真正重要的地方是什么？**

我认为这篇论文真正重要的地方不只是提高了一些 benchmark，而是改变了大家对 LLM 系统设计的理解。以前大家主要关注更大的模型和更长的 context，而 MemGPT 提出：即使 context 有限，也可以通过 memory system 和 tool use，让 LLM 表现得像拥有长期记忆和无限上下文一样。
