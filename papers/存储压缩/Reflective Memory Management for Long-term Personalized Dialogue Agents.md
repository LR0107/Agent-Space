# In Prospect and Retrospect: Reflective Memory Management for Long-term Personalized Dialogue Agents

**ACL 2025**

## 一.研究背景

在真实应用中（客服 / 助手 / 教育系统）：

- 用户会跨多个 session对话
- 关键信息分散在历史中：
  - 个人信息（年龄、偏好）
  - 长期状态（健康、学习情况）
  - 事件记忆（过去某次咨询）

模型需要：当前问题 + 历史信息 = 更个性化回答

**现有方法问题**

论文指出两类关键缺陷：

(1) 固定粒度的记忆切分 

现有方法通常按：

- turn（单轮）
- session（一次对话）

问题：

- 语义主题可能跨多轮
- 导致 memory 被切碎

结果：

- 信息不完整
- 检索不准

(2) 固定 retriever 

传统做法：用固定 embedding retriever（如 DPR / Contriever）

问题：

- 不同用户 / 场景分布变化大
- retriever 不适配

**总体目标**

设计一个系统：能“自动整理记忆 + 自我优化检索”的长期对话记忆机制

---

## 二.研究方法

系统包含 4 个组件：

- Memory Bank（外部记忆库）
- Retriever（初筛）
- Reranker（重排）
- LLM（生成 + 反馈）

**Prospective Reflection（前瞻反思）**

解决：怎么存记忆

核心思想：不是按 turn/session 存，而是按“语义主题 topic”存记忆

流程：

每个 session 结束后：

1. 用 LLM 从对话中抽取：
   - topic summary（主题）
   - raw dialogue（原始片段）
2. 做 memory merge：
   - 如果已有类似 topic → merge
   - 否则 → add

Retrospective Reflection（回顾反思）

解决：怎么改进检索

核心思想：让 LLM 自己判断“刚刚用到的记忆有没有用？”

做法：

1. Retriever 先取 Top-K
2. Reranker 选 Top-M
3. LLM 生成回答 + citation（引用了哪些 memory）
4. 根据 citation 构造 reward：
- 被引用 → +1（有用）
- 未引用 → -1（无用）

然后用 RL更新 reranker：

```
优化目标：更倾向选择“被 LLM 用到的 memory”
```

本质：LLM 作为 reward model 的在线学习检索系统

| 层级        | 作用              |
| --------- | --------------- |
| Retriever | 召回候选（embedding） |
| Reranker  | 学习排序策略（RL）      |
| LLM       | 生成 + 提供监督信号     |

---

### 工作流程

**1.查询阶段（Retrieval + Generation）**

当用户发来一个问题：

**Step 1：检索记忆（Retriever）**

- 用当前 query 去 Memory Bank 检索
- 取出 Top-K 相关记忆（embedding similarity）

作用：粗筛历史信息

**Step 2：重排序（Reranker）**

- 对 Top-K 记忆进行打分
- 选出 Top-M 最相关记忆

**Step 3：LLM 生成回答**

输入：

- 当前对话
- Top-M memory

输出：

- 最终回答
- citations（引用了哪些 memory）

**2.在线学习阶段（Retrospective Reflection）**

**Step 4：基于引用生成 reward**

如果 memory 被 LLM 在回答中引用：+1（有用）

没有引用：-1（无用）

相当于自动标注“哪些记忆真的有用”

**Step 5：RL更新 reranker**

用 REINFORCE 更新 reranker参数

目标：让未来更容易选中“被证明有用”的记忆

**3.记忆写入阶段（Prospective Reflection）**

当 session 结束：

Step 6：LLM总结当前对话，抽取：

- topic summary（主题）
- 对应原始对话片段

Step 7：写入 / 合并 Memory Bank

- 如果是新信息 → add
- 如果已有相似主题 → merge

形成结构化“主题记忆库”

**LLM负责“理解 + 标注 + 写入”，但不负责“检索排序计算”**

---

## 三.实验设计

### 实验设置

**数据集**

两个 benchmark：

(1) MSC

- 多轮个性化对话
- 偏“回复质量”

指标：

- METEOR
- BERTScore-

 (2) LongMemEval

专门测 long-term memory retrieval

指标：

- Recall@K
- Accuracy（QA correctness）

**baseline**

基础：

- No History
- Long Context
- RAG

记忆系统：

- MemoryBank
- LD-Agent

**配置**

Generator：Gemini-1.5-Flash / Pro

Retriever：

- Contriever（默认）
- Stella
- GTE

---

### 实验结果

(1) 主结果

RMM 全面最优：

| 方法       | MSC METEOR | LongMemEval Acc |
| -------- | ---------- | --------------- |
| RAG      | 27.5       | 63.6            |
| LD-Agent | 25.4       | 59.2            |
| **RMM**  | **33.4**   | **70.4**        |

(2) memory重要性

- MSC：86%样本 memory 提升结果
- LongMemEval：100%

结论：long-term task 基本“必须依赖记忆”

(3) 消融实验

关键发现：

- 只做 RL retriever → 不稳定 / 容易崩
- 加 reranker → 性能显著提升

(4) granularity分析

发现：

- turn / session 都不够好
- PR（topic-level）最好

说明：“语义级记忆”优于“时间切片记忆”

 (5) Top-K / Top-M

结论：

- K ↑ → recall ↑
- M ↑ → QA ↑

但有 trade-off（计算成本）

---

## 四.总结

### 创新点

从“按时间存”→“按语义主题存”，解决 memory碎片化

**用 LLM 自己当“标注器 + reward model”**

### 不足

RL训练成本高，reward 稀疏

依赖 LLM citation，LLM引用 ≠ memory真实有用：reward function 是“LLM自身偏好”，不是ground truth

memory merge 依赖 LLM

retriever本身没有被真正优化：只更新 reranker

---

### Q&A

**1.RMM和普通RAG有什么本质区别？**

RAG只做“检索 + 拼接上下文”，检索策略是静态的，且不具备长期记忆结构优化能力。  
RMM的关键区别在于：

- memory是**topic级结构化存储**
- retrieval不是固定的，而是通过**reranker + RL动态优化**
- 引入LLM citation作为反馈信号，实现**自我改进检索系统**

本质：RAG = 静态检索；RMM = 可学习的记忆系统

**2.Top-K memory是如何检索的？reranker是LLM吗？**

Top-K memory的检索过程完全由embedding-based方法完成，通常使用Contriever或GTE等模型将query和memory编码为向量，并通过cosine similarity或dot product进行相似度计算，再利用FAISS等ANN结构进行快速近邻搜索。该阶段不涉及LLM参与，属于标准的密集向量检索过程。

reranker并不是LLM，而是一个轻量级可学习神经网络模块，其主要作用是对retriever返回的Top-K结果进行再排序。它的本质是一个可训练的排序策略模型，而非生成模型。

**3.为什么不直接优化retriever，而是只训练reranker？**

论文选择只优化reranker而不直接微调retriever，主要是出于稳定性考虑。retriever通常是大规模预训练embedding模型，直接更新容易导致表示空间漂移或灾难性遗忘，并且训练成本较高。而reranker是轻量模块，可以在保持retriever稳定性的前提下进行快速在线学习，因此更适合强化学习场景。

**4.reward设计是否可靠？有没有偏差？**

reward来源于LLM生成回答时的citation，因此本质上属于弱监督信号，而不是严格的真实标注。其潜在问题是LLM的引用行为可能受到生成偏好影响，而不完全等价于“真实有用性”。

**5.Prospective Reflection会不会产生错误合并？**

是存在风险的，因为memory merge依赖LLM判断语义相似性，而LLM本身可能在不同上下文下产生不一致判断，导致不同主题被错误合并或同一主题被拆分。


