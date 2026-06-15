# LLM Powered Autonomous Agents

https://lilianweng.github.io/posts/2023-06-23-agent/

## 概述

Agent = LLM + Planning + Memory + Tool Use

LLM 充当 Agent 的“大脑”

### 规划（Planning）

- **子目标划分与任务分解（Subgoal and decomposition）** ：Agent 会将大型任务拆解为更小、更易管理的子目标，从而能够高效处理复杂任务。
- **反思与改进（Reflection and refinement）**  ：Agent 能够对过去的行为进行自我批评与自我反思，从错误中学习，并在后续步骤中加以改进，从而提升最终结果的质量。

### 记忆（Memory）

- **短期记忆（Short-term memory）** ：上下文学习（in-context learning）
- **长期记忆（Long-term memory）** ：长期记忆赋予 Agent 在较长时间内保存和检索（近乎无限）信息的能力，通常通过外部向量数据库（vector store）和快速检索机制来实现。

### 工具使用（Tool use）

Agent 学会调用外部 API，以获取模型参数中缺失的信息（这些信息通常在预训练后较难修改），包括：

- 最新实时信息
- 代码执行能力
- 访问专有数据源

![](C:\Users\LR\AppData\Roaming\marktext\images\2026-05-14-15-43-10-image.png)

---

## 规划（Planning）

复杂任务通常包含许多步骤。一个 Agent 需要知道这些步骤是什么，并能够提前进行规划。

### 任务分解（Task Decomposition）

**思维链（Chain of Thought，CoT；Wei 等，2022）** ：模型会被指示“逐步思考（think step by step）”，以便在推理阶段使用更多计算资源，将困难任务拆解为更小、更简单的步骤。

CoT 能够：

- 将大型任务转化为多个可管理的小任务；
- 同时也让人们能够更直观地理解模型的推理过程。

**思维树（Tree of Thoughts；Yao 等，2023）** 在 CoT 基础上进一步扩展，在每一步探索多种推理可能性。

其核心流程包括：

1. 将问题拆解为多个“思考步骤（thought steps）”；
2. 每一步生成多个可能的“thoughts”；
3. 从而形成一个树状结构。

搜索过程可以采用：

- BFS（广度优先搜索）
- DFS（深度优先搜索）

并且每个状态会通过以下方式进行评估：

- 使用分类器（通常通过 Prompt 实现）
- 或采用多数投票（majority vote）

### 自我反思（Self-Reflection）

自我反思是自主 Agent 的关键能力之一，它使 Agent 能够通过：

- 复盘过去的动作决策；
- 修正之前的错误；

从而进行迭代式改进（iterative improvement）。

在现实世界任务中，由于“试错（trial and error）”几乎不可避免，因此这一能力尤为重要。

**ReAct (Yao et al. 2023）** 将推理（reasoning）与行动（acting）整合进 LLM 中。

二者的作用分别是：

- 前者使 LLM 能够与外部环境交互  
  （例如调用 Wikipedia Search API）
- 后者则促使 LLM 生成自然语言形式的推理轨迹（reasoning traces）

ReAct 的 Prompt 模板通常会显式加入思考（Thought）步骤，大致格式如下：

- Thought: ...
- Action: ...
- Observation: ...

**Reflexion (Shinn & Labash 2023)** 是一个为 Agent 提供：

- 动态记忆（dynamic memory）
- 自我反思（self-reflection）

能力的框架，用于提升推理能力。

在每次动作之后：

- Agent 会计算一个启发式函数（heuristic）
- 并基于自我反思结果，决定是否：重置环境、开启一次新的尝试

因此，Reflexion 的关键思想是：Agent 不只是“执行”，还会主动评估自己的表现，并根据反思结果重新尝试。

自我反思的生成方式是：

向 LLM 提供 few-shot 示例（具体为 two-shot examples），其中每个示例由以下内容组成：

1. 失败的轨迹
2. 用于指导未来规划改进的反思

随后，这些 reflection 会被加入 Agent 的工作记忆（working memory）中，最多保留三条，并作为后续调用 LLM 时的上下文（context）使用。

---

## 记忆（Memory）

### **短期记忆（STM）/ 工作记忆（Working Memory）**

短期记忆可以类比为上下文学习（in-context learning）。它是短暂且有限的，因为受到 Transformer 有限上下文窗口的限制。

### **长期记忆（LTM）**

长期记忆（Long-term Memory）则可以类比为 Agent 在查询时能够访问的外部向量存储（external vector store），并通过快速检索机制进行调用。

外部记忆能够缓解有限注意力跨度（finite attention span）带来的限制。

一种标准做法是：将信息的 embedding 表示保存到支持快速最大内积搜索（MIPS）的向量数据库（vector store database）中。

为了优化检索速度，通常会采用：

- 近似最近邻算法（Approximate Nearest Neighbors，ANN）

其目标是：返回近似的 Top-k 最近邻结果，以牺牲少量准确率换取巨大的速度提升。

## 工具使用（Tool Use）

为大语言模型（LLM）配备外部工具，能显著扩展其能力。

**MRKL**（Modular Reasoning, Knowledge and Language）是一种面向智能体的神经符号架构。它由多个“专家模块”组成，而通用 LLM 负责根据任务类型，将请求分配给最合适的模块。这些模块既可以是神经网络模型，也可以是符号工具，例如计算器、汇率转换器或天气 API。

**TALM** （Tool Augmented Language Models）与 **Toolformer** 都通过微调语言模型，使其学会调用外部工具 API，并根据 API 调用是否提升输出质量来扩展训练数据。

现实中的典型例子包括：

- ChatGPT Plugins
- OpenAI API 的 function calling

工具 API 既可以来自第三方开发者（如插件），也可以由开发者自行定义（如函数调用）。

**HuggingGPT**（Shen 等，2023）则进一步将 ChatGPT 作为任务规划器，根据 Hugging Face 平台上的模型描述选择合适模型，并基于执行结果汇总最终响应。

---

## Case Studies

### ChemCrow

是一个领域专用（domain-specific）的 Agent ，用于完成：

- 有机合成（organic synthesis）
- 药物发现（drug discovery）
- 材料设计（materials design）

等任务。

其工作流基于 LangChain 实现，并体现了前文提到的：ReAct、MRKL等思想，将：

- CoT 推理
- 与任务相关的工具调用

结合在一起。

虽然基于 LLM 的自动评测认为：GPT-4 与 ChemCrow 的表现几乎相同

但由化学专家进行的人工评测（任务完成度、正确性）

却表明：ChemCrow 显著优于 GPT-4。

这说明：使用 LLM 评估其自身在高专业领域中的表现，可能存在问题。

因为：缺乏领域专业知识的 LLM、可能无法意识到自身错误，因而难以准确判断结果是否正确。

---

### Generative Agents Simulation

是一个非常有趣的实验：25 个虚拟角色；每个角色都由一个 LLM 驱动的 Agent 控制；它们在一个类似 The Sims 的沙盒环境中生活与互动。

目标是：为交互式应用创建可信的人类行为模拟（believable simulacra of human behavior）。

其设计结合了：

- LLM
- Memory（记忆）
- Planning（规划）
- Reflection（反思）

使 Agent 能够：

- 基于过去经验行动；
- 与其他 Agent 互动。

**Memory Stream（记忆流）**

Memory Stream 是一个长期记忆模块（外部数据库），用于以自然语言形式记录 Agent 的完整经历。

**Retrieval Model（检索模型）**

检索模型会根据：

- 相关性（relevance）——衡量记忆与当前情境或查询的关联程度
- 新近性（recency）——最近发生的事件得分更高
- 重要性（importance）——区分日常琐事与核心记忆、直接询问 LM 

来提取上下文，以指导 Agent 的行为。

**Reflection Mechanism（反思机制）**

反思机制会：

- 随时间将记忆整合为更高层次的推断（higher-level inferences）；
- 并指导 Agent 未来行为。

具体实现方式：

1. 向 LM 提供最近的 100 条 observation；
2. 生成 3 个最重要的高层问题；
3. 再让 LM 回答这些问题。

**Planning & Reacting（规划与反应）**

将 reflection与环境信息，转换为具体行动。

---

## Challenges

### 有限的上下文长度（Finite context length）

受限的上下文容量（context capacity）限制了以下信息的纳入：

- 历史信息（historical information）
- 详细指令（detailed instructions）
- API 调用上下文
- 响应内容（responses）

因此，系统设计必须在这种有限通信带宽（communication bandwidth）下运行。

然而像**自我反思**这类需要从过去错误中学习的机制，其实会非常受益于：更长甚至无限的上下文窗口（context windows）。

虽然向量数据库（vector stores）与检索机制（retrieval）能够让模型访问更大的知识池，但它们的表示能力（representation power）仍然不如完整注意力机制（full attention）。

---

### 长期规划与任务分解的挑战（Challenges in long-term planning and task decomposition）

对于 LLM 来说：

- 基于长历史进行规划（planning over lengthy history）；
- 高效探索解空间（solution space）；

仍然是困难问题。

当遇到意外错误时：LLM 往往难以及时调整原有计划，

因此其鲁棒性（robustness）仍弱于人类。

而人类通常能够：

- 通过试错（trial and error）
- 不断学习并修正策略。

---

### 自然语言接口的可靠性问题（Reliability of natural language interface）

当前的大多数 Agent 系统，都依赖自然语言作为：

- LLM
- 与外部组件（如 memory、tools）

之间的接口。

但模型输出的可靠性仍存在问题，因为 LLM 可能会：

- 出现格式错误
- 偶尔表现出反抗行为

例如：拒绝遵循指令。

因此许多 Agent Demo 的代码实际上主要都在处理：

> 如何解析（parsing）模型输出。
