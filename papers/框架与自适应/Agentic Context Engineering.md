## Agentic Context Engineering: Evolving Contexts for Self-Improving Language Models

**ICLR 2026**

## 一、研究背景

现代LLM应用（Agent、RAG系统、工具调用系统）主要通过**修改输入上下文（context）**来提升能力，而不是训练模型权重。

典型“上下文”包括：

- system prompt（系统提示词）
- memory（记忆）
- tool usage instructions（工具使用策略）
- external knowledge（外部知识）

本质：“模型能力 = 固定参数 + 可变上下文”

### 现有方法的问题

**(1) Brevity Bias（过度简化偏差）**

现有 prompt 优化方法（如 GEPA）倾向于：

- 把复杂策略压缩成短指令
- 丢失细节（工具使用技巧、失败经验等）

结果：prompt 变短，但能力变弱（信息不足）

(**2) Context Collapse（上下文坍缩）**

如果不断让模型“重写总结历史上下文”：

- 上下文越来越短
- 细节逐渐丢失
- 性能显著下降

本质问题：LLM在“压缩历史”时会破坏知识结构

## 二、研究方法

ACE（Agentic Context Engineering）

ACE 的核心不是“优化prompt”，而是把 context 当成一个不断演化的策略手册（playbook）

它会：

- 累积经验
- 结构化整理策略
- 持续增量更新

而不是反复 rewrite

ACE = 三个角色协作：

**(1) Generator（生成器）**

作用：

- 执行任务
- 产生 reasoning trajectory（推理轨迹）
- 记录成功/失败行为

**(2) Reflector（反思器）**

作用：

- 分析 execution traces
- 提取经验：
  - 成功策略
  - 常见错误
  - failure patterns

关键：自然语言总结，它负责“总结经验”，但不直接修改最终context

**(3) Curator（策展器）**

作用：

- 将 Reflector 输出整理成：
  - structured bullets（结构化条目）
  - 可复用策略
- 写入 context memory

还会决定：

- 写入 or 丢弃
- 更新旧条目 or 新增
- 合并相似规则

结构化、标准化、去噪 & 压缩

**Context结构：bullet-based memory**

每条 context 不是段落，而是：

- strategy（策略）
- code snippet（代码模板）
- failure case（错误案例）
- tool usage rule（工具规则）

并带 metadata：

- 使用次数
- 有效性标记

**两个关键机制**

**(1) Incremental Delta Updates（增量更新）**

不再“重写整个context”，而是：

- 只新增/修改小条目（delta）
- 局部更新

优点：

- 避免信息丢失
- 降低计算成本
- 可并行更新

**(2) Grow-and-Refine**

过程分两步：

**Grow：**

- 不断增加新策略

**Refine：**

- 去重
- 合并语义相似条目
- 清理冗余

注：都是llm来完成

最终产物是Context Playbook（上下文策略库）

是一个**结构化知识集合**，通常包含：

- 解题策略（how to solve）
- 常见失败模式（what to avoid）
- 工具使用规则（how to use APIs/tools）
- 代码片段

每次模型运行时：LLM 会读取整个 playbook，然后选择相关条目来辅助推理

playbook ：每次调用 LLM 时，被塞进 context window 的输入

所以它更像操作手册（外部维护了一个不断增长的“知识库）。

playbook 会增长，但不会无限膨胀；它通过去重、合并和剪枝机制，使其逐渐收敛成一个高质量的知识库。

**两种模式**

**Offline adaptation（离线适配）**

在部署/测试之前，用一批数据先把 context playbook 优化好。

```
训练/开发集
   ↓
Generator + Reflector + Curator（多轮）
   ↓
构建最终 context playbook
   ↓
固定下来 → 用于测试
```

**Online adaptation（在线适配）**

在推理过程中，每做一个任务就更新 context

```
task 1 → 执行 → 更新 context
task 2 → 执行 → 更新 context
task 3 → 执行 → 更新 context
...
```

### 工作流程

1️⃣ **执行（Generator）**  
用当前 context 解决任务，生成推理过程和结果。

2️⃣ **反思（Reflector）**  
分析成功/失败，提炼可复用经验。

3️⃣ **整理（Curator）**  
把经验写成结构化条目（策略 / 规则 / 示例）。

4️⃣ **更新（Context Update）**  
将新条目增量写入 context（避免整体重写）。

---

## 三、实验设计

 **(1) Agent任务**

数据集：AppWorld

特点：

- 多轮交互
- API调用
- 代码生成
- 环境反馈

指标：

- TGC（Task Goal Completion）：单个任务是否完成 成功 = 1、失败 = 0
- SGC（Scenario Goal Completion）：只有整体流程完成才算 SGC=1

**(2) Domain reasoning**

包括：

- 金融（FiNER / Formula）
- 医疗（DDXPlus）
- Text-to-SQL（BIRD-SQL）

**Base model：DeepSeek-V3.1（671B）**

**Baselines**

包括：

- Base LLM（无优化）
- ICL（in-context learning）
- GEPA（prompt进化）
- MIPROv2（DSPy优化）
- Dynamic Cheatsheet（记忆系统）

### 实验结果

**1 Agent任务（AppWorld）**

ACE显著提升：

- +10.6% 平均提升
- +17.1% 最多提升

甚至：

- 接近/超过 GPT-4.1 级生产系统（IBM CUGA）
- 使用更小的开源模型（DeepSeek-V3.1）

**2 金融任务**

- 平均 +8.6% 性能提升
- 在复杂结构任务（XBRL）表现突出

**3 成本优势**

ACE的工程优势：

- latency ↓ 86.9%
- rollouts ↓ 75%+
- token cost ↓ 80%+

避免“全量重写 + 重复处理”，改为增量更新 + cache复用。

整个系统在执行任务 + 自适应更新过程中的累计token成本

结论：不只是更准，而且更便宜、更快

---

## 四、总结

### 创新点

把 Context 从“prompt”升级为“可演化系统”：是一个“LLM自我改进架构”，而不是prompt技巧

不更新模型参数，只更新外部context

用 incremental updates 替代 full rewrite



### 不足

强依赖 Reflector 质量：论文自己承认如果 Reflector 提炼能力差，会污染 context

context 会“积累噪声”

---

### Q&A

**1.这篇论文解决什么问题？**

ACE主要解决LLM在上下文优化中的两个问题：**brevity bias（过度简化）**和**context collapse（反复重写导致信息丢失）**。传统prompt优化会不断压缩prompt或重写历史内容，导致关键策略丢失，而ACE提出用结构化、增量式的context演化来避免这一问题。

**2.ACE的核心思想是什么？**

ACE的核心思想是把LLM的context看作一个持续演化的策略库（playbook），而不是一次性的prompt。通过不断执行任务、反思经验、结构化写入，实现context的自我增长与优化。

**3.ACE和prompt engineering有什么区别？**

传统prompt engineering是“设计一段静态指令”，而ACE是“动态构建知识系统”。前者一次性使用，后者会在运行过程中不断更新、积累和优化context。

**4.为什么需要Reflector和Curator分开？**

Reflector负责“理解和总结经验”，但输出通常是非结构化文本；Curator负责把这些经验转成统一格式的标准化、结构化条目，并决定是否写入、合并或删除。这样可以保证知识库稳定、可维护。

**5.什么是context collapse？为什么会发生？**

context collapse是指反复让模型“总结或重写整个context”时，信息逐渐被压缩丢失，最终变成短但不完整的描述。原因是全局重写会不断重新筛选信息，导致细节逐步消失。


