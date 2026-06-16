# Stop Wasting Your Tokens: Towards Efficient Runtime Multi-Agent Systems

**ICLR 2026**

提出了一种用于多智能体系统（Multi-Agent Systems, MAS）的运行时监督框架——**SupervisorAgent**

## 一.研究背景

近年来，大模型推动了多智能体系统的发展。

然而作者发现：系统越复杂，效率反而越低。

**两大核心问题**

**问题1：错误传播（Error Propagation）**

例如：

Agent A搜索网页时产生了错误信息：“苹果公司成立于1980年”

随后：

- Agent B相信了它
- Agent C继续引用
- Agent D基于错误事实推理

最终整个系统被污染。

**问题2：Token浪费**

作者发现大量Token被浪费在：

**（1）超长观察结果**

例如搜索工具返回：

- 整个网页HTML
- 几千字网页内容
- 冗长API结果

Agent往往会把这些全部塞进上下文。

结果：

- 成本增加
- 关键信息被淹没

**（2）低效循环**

例如Agent不断执行：

```
Page Down
Page Down
```

而实际上：Search关键词一步就能找到答案。

---

## 二.研究方法

作者提出：给整个MAS再增加一个“监督Agent”。

这个Agent不负责解题，而是负责：

- 看别人干活
- 发现问题
- 及时纠正

### 1.监督什么（What to Supervise）

作者认为MAS中最危险的是三类交互：

**Agent ↔ Agent**

风险：

- 幻觉传播
- 错误结论扩散

 **Agent ↔ Tool**

调用工具：

- 搜索引擎
- Python
- API

风险：

- 返回错误结果
- 返回无关结果

**Agent ↔ Memory**

读取记忆库。

风险：

- 调用过期信息
- 调用错误经验

### 2.什么时候监督（When to Supervise）

如果每一步都监督：成本太高。

因此作者设计了一个：Adaptive Filter（自适应过滤器）

特点：

- 不使用LLM
- 规则驱动
- 非常便宜

只有检测到高风险情况才触发Supervisor。

**三种触发条件**

**① Error**

出现明确错误：

例如：

```
IndexError
KeyError
```

立即介入。

 **② Inefficient Behavior**

发现低效行为：

例如：

```
PageDown × 10
```

明显不合理。

**③ Excessive Observation**

观察结果太长：

例如：

```
HTML 5000 tokens
```

### 3.如何监督（How to Supervise）

作者设计了三种干预策略。

**策略1：Error Correction**

主动纠错

当检测到错误Supervisor会：

- 诊断原因
- 提供修正方案

例如：

Agent：

```
工具报错
```

Supervisor：

```
参数格式错误请改为JSON格式
```

**策略2：Guidance**

效率指导

发现Agent走弯路时：给出提示。

例如：

Supervisor提示：不要继续翻页了，直接搜索关键词。

从而减少大量无意义步骤。

**策略3：Observation Purification**

Supervisor会：

把长文本：

```
网页HTML
广告
导航栏
噪声信息
```

压缩成：真正有用的信息

只保留关键内容。

作者发现：这是降低Token成本贡献最大的模块。

---

### 工作流程

**监控 → 检测 → 干预 → 继续执行**

**监控（Monitor）**

- 实时观察 Agent 与 Agent、Tool、Memory 之间的交互。

**检测（Adaptive Filter）**

- 用轻量级过滤器判断当前是否存在：
  - 错误（Error）
  - 低效行为（Inefficient Behavior）
  - 过长观察结果（Excessive Observation）

**干预（Supervision）**

- 针对错误：纠错（Correction）
- 针对低效行为：给出指导（Guidance）
- 针对超长内容：压缩和净化信息（Purification）

**继续执行**

- 将修正后的信息或建议返回给原 Agent，由原 Agent 继续完成任务。

```
Agent执行
    ↓
Adaptive Filter（规则判断）
    ↓
是否高风险？
   ↙      ↘
 否         是
 ↓          ↓
继续执行   调用LLM Supervisor
               ↓
         纠错/指导/压缩
```

**每类问题对应一套专门的 Prompt 和处理逻辑**

---

## 三.实验设计

实验平台：Smolagent

HuggingFace的Agent框架。

作者在其基础上增加SupervisorAgent。

**测试数据集**

覆盖三个领域六个Benchmark：

综合Agent任务

- GAIA

数学推理

- AIME 2024
- GSM8K-Hard

代码生成

- HumanEval
- MBPP

问答

- DROP

**评价指标**

Accuracy：任务完成率

Token Cost：关注的是能否同时提高效率且不降低准确率。

### 实验结果

GAIA实验

加入Supervisor后：Token下降29.68%。准确率保持不变

| 数据集       | 效果            |
| --------- | ------------- |
| AIME      | 准确率 +6.67%    |
| HumanEval | Token ↓23.74% |
| GSM-Hard  | 准确率提升         |
| DROP      | Token下降       |

几乎所有任务都更省Token，同时保持甚至提高性能。

**消融实验**

作者分别删除三个模块。

**去掉 Purification**

Token节省：

```
29.68% → 15.96%
```

说明：Purification是节省Token的主力

去掉 Correction：准确率明显下降。

去掉 Guidance：准确率也下降。

---

## 四.总结

**创新点**

从“事后分析”转向“运行时干预”

轻量级触发机制（Adaptive Filter）

**不足**

Adaptive Filter过于规则化

Supervisor本身可能出错
