# Effective context engineering for AI agents

[Effective context engineering for AI agents \ Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

构建基于语言模型的系统，已经不再只是寻找正确的提示词，而更多是在回答这样一个更广泛的问题：

> “怎样配置上下文，最有可能让模型产生我们期望的行为？”

这里的上下文（context）指的是：在从大语言模型（LLM）进行采样时，被包含进去的那一组 token。  
当前的问题在于：如何在 LLM 固有约束下，优化这些 token 的效用，从而稳定地产生目标结果。

有效地驾驭 LLM，往往需要以“上下文”的视角来思考——换句话说，需要考虑：

- 在任意时刻，LLM 能看到的完整状态是什么；
- 以及这种状态可能诱发哪些行为。

## 上下文工程 vs. 提示工程

上下文工程是提示工程的自然演进。

- **提示工程（prompt engineering）**：  
  指的是编写和组织 LLM 指令的方法，以获得最佳输出效果（例如系统提示词的设计）。
- **上下文工程（context engineering）**：  
  指的是在 LLM 推理过程中，围绕“如何整理、维护最优 token 集合（信息）”的一整套策略。  
  这里不仅包括 prompt 本身，还包括所有可能进入上下文窗口的信息。

提示工程关注的重点是：

- 如何写出有效的 prompt；
- 尤其是 system prompt（系统提示词）。

然而，随着我们开始构建能力更强的 agent——它们能够：

- 进行多轮推理；
- 在更长时间跨度上运行；

我们就需要新的策略，来管理整个上下文状态，包括：

- 系统指令（system instructions）
- 工具（tools）
- 模型上下文协议（Model Context Protocol，MCP）
- 外部数据
- 消息历史

一个在循环中运行的 agent，会不断生成新的数据，而这些数据都可能与下一轮推理相关，因此这些信息必须被持续地筛选和精炼。

---

## 为什么上下文工程对构建强大的 Agent 至关重要

尽管 LLM 的速度越来越快、能够处理的数据量也越来越大，但我们观察到：LLM 和人类一样，在某个临界点之后会开始“失焦”或产生混乱。

**上下文腐化（context rot）**：随着上下文窗口中的 token 数量增加，模型从上下文中准确召回信息的能力会逐渐下降。

虽然不同模型的退化程度不同，有些模型衰减更平缓，但这种现象在所有模型中都会出现。

因此：

> 上下文必须被视为一种有限资源，而且它的边际收益会递减。

## 注意力稀缺的根源

这种“注意力稀缺”来源于 LLM 的架构限制。

LLM 基于 Transformer 架构，而 Transformer 的核心机制是：

> 每一个 token 都可以关注上下文中的其他所有 token。

token 之间的关系数量会以平方级增长。

随着上下文长度不断增加，模型对这些两两关系的捕捉能力会被不断稀释。

此外，模型的注意力模式来源于训练数据分布。

而在训练数据中：短序列通常远比长序列更常见。

因此模型：

- 对长上下文的经验更少；
- 针对超长依赖关系学习到的专门参数也更少。

这些因素共同导致：模型性能下降并不是突然坍塌（hard cliff），而是渐进式衰减（performance gradient）。

也就是说：模型在长上下文下依然很强；

但相比短上下文，它在以下方面可能会变差：

- 信息检索精度
- 长距离推理能力
- 细节保持能力

## 高效上下文的结构（The anatomy of effective context）

### System Prompt 应该如何设计

系统提示词（system prompt）应该：

- 极其清晰；
- 使用简单、直接的语言；
- 并且以适合 agent 的抽象层级来表达思想。

### 失败模式一：过度硬编码（too low-level）

一种极端情况是：工程师在 prompt 中硬编码复杂而脆弱的逻辑，以强行诱导 agent 产生精确行为。

这种做法的问题在于：

- 系统会变得非常脆弱；
- prompt 难以维护；
- 随时间推移，复杂度会不断增加。

换句话说：prompt 被写成了“伪程序”，而不是行为指导。

### 失败模式二：过度抽象（too high-level）

另一种极端则是：工程师只提供模糊、宏观的指导。

这种情况下：

- LLM 得不到明确的行为信号；
- 输出目标不具体；
- 甚至错误地假设模型与人类共享某些隐含背景知识。

结果就是：

- agent 行为漂移；
- 输出不稳定；
- 模型容易“自由发挥”。

### 最优状态：正确的抽象层级

理想的 system prompt 应该做到：

- 足够具体，能够有效引导行为；
- 同时又足够灵活，不把行为写死。

也就是说：prompt 不应该替模型思考，而应该为模型提供强有力的启发式信号（heuristics）。

真正优秀的 system prompt 更像：

- 行为原则；
- 决策框架；
- 注意力引导；

而不是：

- if/else 脚本；
- 硬编码工作流；
- 巨型规则列表。

建议将 prompt 组织成多个清晰的独立部分，例如：

- `<background_information>`
- `<instructions>`
- `Tool guidance`
- `Output description`

同时，也建议使用：

- XML 标签（XML tagging）
- Markdown 标题（Markdown headers）

来明确划分这些区域。

### Prompt 的核心原则：最小充分信息

无论你采用怎样的 system prompt 结构，都应该追求：

> 用“最少但足够”的信息，完整定义期望行为。

这里要注意：“最小（minimal）”并不等于“短（short）”。

你依然需要提前向 agent 提供足够的信息，以确保它能够遵循目标行为。

推荐的方法是：

1. 先用最小 prompt 测试当前最强模型；
2. 观察它在任务上的表现；
3. 基于实际失败模式（failure modes），逐步加入：
   - 更清晰的指令；
   - 更好的示例；
   - 更明确的约束。

换句话说：不要一开始就写一个 3000 行的 system prompt。

而应该：从最小可行提示（minimal viable prompt）开始迭代。

### Tool（工具）设计的重要性

工具让 agent 能够：

- 与外部环境交互；
- 获取新的上下文信息；
- 执行动作。

由于工具定义了：agent 与信息空间 / 行为空间之间的契约（contract），

因此：**工具设计必须高度关注效率。**

这里的效率包括：

- 返回的信息要 token-efficient；
- 工具设计要鼓励 agent 形成高效行为

**一个优秀工具应具备：**

> “它到底应该什么时候被使用”

这一点上，必须非常清晰。

工具的输入参数也应该：

- 描述明确；
- 无歧义；
- 利用模型本身擅长的能力。

### 常见失败模式：工具集膨胀（bloated tool sets）

我们最常见的问题之一是：

> 工具集过于庞大，覆盖太多功能。

这会导致：

- 工具之间边界模糊；
- agent 不知道该调用哪个工具；
- 决策点变得含混。

这里有一个非常重要的判断标准：如果一个人类工程师都无法明确判断该用哪个工具，  
那么你不能指望 agent 做得更好。

### Few-shot 示例：质量比数量更重要

提供示例（few-shot prompting）仍然是非常重要的最佳实践，我们依然强烈推荐。

但很多团队会犯一个问题：把大量边缘案例（edge cases）塞进 prompt 中， 试图枚举 LLM 应遵循的所有规则。

并不推荐这种做法。

更好的方法是：

> 精心挑选一组“多样化但具有代表性”的经典示例。

这些示例应该能够有效展示：

- agent 的目标行为；
- 输出风格；
- 决策模式。

### 总体原则

对于上下文中的各个组成部分：

- system prompt
- tools
- examples
- message history

总体建议是：

保持深思熟虑（thoughtful）;  
让上下文既信息充足（informative）;  
又足够紧凑（tight）。

## 上下文检索与 Agent 式搜索（Context retrieval and agentic search）

**Agent = LLM 在循环中自主使用工具**

也就是说：

- 不只是一次性调用模型；
- 而是模型能够：
  - 自主决定调用什么工具；
  - 根据结果继续行动；
  - 在循环中持续推进任务。

随着底层模型越来越强：agent 的自主性也能够不断提升。

更聪明的模型意味着：

- agent 能独立处理复杂问题空间；
- 能从错误中恢复；
- 能在不确定环境中继续推进任务。

### 基于 embedding 的预检索（pre-inference retrieval）

也就是：在模型推理前，先通过向量搜索，把相关信息提前放入上下文。

但随着 agentic 方法兴起：越来越多团队开始采用“Just-in-time Context（即时上下文）”策略。

### Just-in-time Context 的核心思想

传统方式：一开始就把所有相关数据塞进上下文。

而 JIT（just-in-time）方式则不同：agent 只保留轻量级引用（references）， 在运行过程中**按需动态加载信息**。

这些引用可能包括：

- 文件路径（file paths）
- 存储查询（stored queries）
- Web 链接
- 数据库引用

然后 agent 在运行时：通过工具动态获取真正需要的数据。

Claude Code 就采用了这种方法。

例如：它可以对大型数据库执行复杂数据分析，但并不会一次性把整个数据库塞进 context。

相反，它会：

- 编写针对性的查询；
- 保存结果；
- 使用 Bash 命令（如 `head`、`tail`）分析数据；
- 只加载当前真正需要的信息。

### 元数据（metadata）本身也是上下文信号

除了节省 token 之外：这些引用的元数据本身，也能帮助 agent 推断行为。

例如：

一个叫 `test_utils.py` 的文件：

- 如果位于 `/tests/` ：很可能是测试辅助代码；
- 如果位于 `/src/core_logic/` ：则可能是核心业务逻辑。

因此：

- 文件层级
- 命名规范
- 时间戳

都会成为重要信号。

### Progressive Disclosure（渐进式信息揭示）

agent 可以通过探索，逐步发现真正相关的上下文。

每次交互都会生成新的上下文，用于指导下一步决策。

例如：

- 文件大小 → 暗示复杂度
- 命名方式 → 暗示用途
- 时间戳 → 暗示相关性

于是：agent 能一层层构建理解， 而不是一开始就被海量信息淹没。

**当然，这种方法也有代价**

运行时探索（runtime exploration）：一定比直接读取预计算结果更慢。

此外：这对工程设计提出了更高要求。

你必须确保：

- agent 有合适工具；
- 有合理 heuristics（启发式规则）；
- 能有效搜索信息空间。

否则 agent 可能会：

- 滥用工具；
- 走入死胡同；
- 错过关键数据；
- 浪费上下文窗口。

### Hybrid Strategy（混合策略）

**一部分数据提前加载**

用于：

- 提高速度；
- 提供基础上下文。

**剩余信息按需探索**

由 agent 自主决定：

- 是否检索；
- 何时检索；
- 检索什么。

Claude Code 就采用了这种混合模式。

例如：CLAUDE.md文件会直接放入上下文；

而像 glob、grep 这样的原语工具，则允许它：

- 自主浏览环境；
- 动态读取文件；
- Just-in-time 获取信息。

这样还能绕开：

- 索引陈旧（stale indexing）
- 复杂语法树（complex syntax trees）

等问题。

作者认为：法律、金融等“动态变化较少”的领域， 可能尤其适合 Hybrid Strategy。

随着模型能力持续增强：agent 设计会越来越倾向于让智能模型自主地发挥智能。

也就是说：

- 更少人工编排；
- 更少硬编码；
- 更少人工筛选；
- 更多自主探索。

## 面向长时间跨度任务的上下文工程（Context engineering for long-horizon tasks）

长时间跨度任务（long-horizon tasks）要求 agent：在连续多步行动中，持续保持：连贯性（coherence）、上下文理解（context）、目标导向行为（goal-directed behavior）

而此时：总 token 数往往已经超过 LLM 的上下文窗口限制。

**更大的 Context Window 并不是根本解法。**

一个直觉上的想法是：那就等待更大的 context window 不就好了？

但作者认为：在可预见的未来，无论 context window 多大，都仍然会受到：上下文污染（context pollution）、信息相关性下降（information relevance）的影响。

因此，为了让 agent 能长期稳定工作，Anthropic 开发了一些专门技术来直接解决这些问题：

**三类关键技术**

- Compaction（压缩）
- Structured note-taking（结构化笔记）
- Multi-agent architectures（多 Agent 架构）

---

### 1. Compaction（上下文压缩）

Compaction 指的是：当对话即将达到 context window 上限时， 对内容进行总结，然后用摘要重新开启新的上下文窗口。

Compaction 通常是上下文工程中，提升长期一致性的第一杠杆（first lever）。

核心目标是：高保真地提炼上下文内容， 让 agent 能在最小性能损失下继续工作。

在 Claude Code 中：

系统会：

1. 把完整消息历史交给模型；
2. 让模型自动总结与压缩；
3. 保留最关键的信息。

模型会重点保留：

- 架构决策（architectural decisions）
- 未解决 bug（unresolved bugs）
- 实现细节（implementation details）

同时丢弃：

- 重复工具输出
- 冗余消息
- 无价值历史

之后 agent 会基于：

- 压缩后的摘要；
- 外加最近访问的 5 个文件；

继续工作。

于是用户能够获得：连续性体验（continuity），而无需担心 context window 限制。

Compaction 的难点：保留什么，丢弃什么

因为过度压缩（overly aggressive compaction）可能会丢失那些“当前看似不重要，但未来会变得关键”的上下文。

**Anthropic 的建议**

第一步：优先 Recall（召回率）

先确保：compaction prompt 能捕获所有潜在相关信息。

也就是说：宁可保留多一点、不要漏掉关键内容。

第二步：再优化 Precision（精确率）

之后再逐步：

- 删除冗余内容；
- 提高压缩效率；
- 缩小 token 占用。

一个典型可删除对象：**工具调用结果**

原因很简单：一个很早之前调用过的工具， 为什么 agent 还需要反复看到完整原始输出？

很多工具结果：token 极大；信息密度却很低。

因此：删除历史工具输出是一种风险非常低、但收益很高的 compaction 方式。

---

### Structured Note-taking

也被称为：Agentic Memory

agent 会定期把笔记写入 context window 之外的持久化存储中，并在之后需要时重新取回这些内容。

为什么这种方式重要

这种策略能够：

- 提供长期持久记忆（persistent memory）；
- 同时几乎不增加 context window 负担。

例如：Claude Code 会维护一个待办列表（to-do list）；你的自定义 agent 可能会维护一个 `NOTES.md` 文件。

这些其实都是：Agentic Memory 的简单实现。

**Context Reset 后仍能继续任务**

当 context 被重置后，agent 会：

1. 重新读取自己的笔记；
2. 恢复之前状态；
3. 继续执行长时间任务。

本质上：**Context Window 不再是唯一记忆空间。**

而 agent 开始拥有：

- Working Memory（工作记忆）
- Long-term Memory（长期记忆）
- External Storage（外部存储）

这样的分层认知结构。

---

### Sub-agent Architectures

它的核心思想是：不要让一个 agent 独自维护整个项目的全部状态。

而是：主 agent 负责总体协调；多个 specialized sub-agents 负责局部任务。

主 agent：

- 维护高层计划；
- 做总体分析与综合；
- 分配任务。

子 agent：

- 负责深入技术工作；
- 或执行大量探索与检索。

每个 sub-agent 都可以：

- 独立使用大量 token；
- 深度探索问题空间；
- 调用工具；
- 执行复杂搜索。

但最终：它只返回一个压缩后的摘要。

通常子 agent 可能使用数万 token；最后只返回 1000～2000 token 的 distilled summary。

这种架构的核心优势：实现了 Clear Separation of Concerns（关注点分离）

子 agent 负责：大量细节搜索；深度上下文；高 token 消耗工作。

而 主 agent 只负责：综合结果；推理；决策。

于是详细搜索的上下文不会污染主 context window。

主 agent 能持续保持：高层抽象；长期目标；全局一致性。

## 不同方法适用于不同任务

### 1. Compaction

适合：需要大量连续对话的任务。

例如：

- 长时间 coding assistant
- 连续 debugging
- 多轮用户协作

因为它能保持：conversational flow（对话连续性）

---

### 2. Structured Note-taking

适合：有明确阶段性里程碑的迭代开发。

例如：

- 软件开发
- 游戏 agent
- 长周期任务

因为它擅长：

- 长期状态保持；
- checkpoint 式记忆。

---

### 3. Multi-agent Architectures

适合：复杂研究与分析任务。

尤其当：

- 并行探索（parallel exploration）
- 能显著提升效率时。

例如：

- research agents
- 大规模代码分析
- 多方向信息收集

即使模型未来继续变强；

文章认为：如何在长时间交互中保持一致性（coherence） 仍然会是 agent 构建中的核心挑战。

---

## Conclusion

**Context Engineering 是一种根本性的范式转变。**

随着模型能力越来越强，挑战已经不再只是：如何写一个完美 prompt。

而变成了：如何在每一步中，精心筛选进入模型注意力预算的信息。

无论你是在：

- 为长时间任务实现 compaction；
- 设计 token-efficient tools；
- 构建 just-in-time retrieval；
- 设计 multi-agent systems；

背后的核心原则始终相同：用最少的高信号 token，最大化目标结果出现的概率。

目前更强的模型，往往需要更少规定式工程（prescriptive engineering）。

也就是说：

- 更少硬编码；
- 更少人工控制；
- 更多自主性。

但即便模型能力不断提升：**把 context 当作珍贵且有限的资源**仍将是构建可靠 agent 的核心原则。
