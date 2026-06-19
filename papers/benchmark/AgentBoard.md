# AgentBoard: An Analytical Evaluation Board of Multi-turn LLM Agents

**NeurIPS 2024**

## 一.研究背景

近年来，大语言模型（LLM）被广泛用于“Agent”（智能体），即：

- 能在环境中感知（observe）
- 进行多步决策（multi-turn action）
- 并完成任务目标（goal-oriented tasks）

典型应用包括：

- 网页自动操作 （WebAgent）
- 游戏任务 （Jericho）
- 工具调用 （API / spreadsheet）
- 具身环境 （AlfWorld 等）

### 现有评估框架的不足

**(1) 任务不统一 & 场景割裂**

- 有的任务是 web
- 有的是 game
- 有的是 tool use
- 但缺乏统一评估框架

**(2) 只看“最终成功率”**

现有方法普遍只用：Success Rate（成功/失败）

问题：

- 无法区分“快完成 vs 完全失败”
- 很多模型成功率接近 0，无法比较细粒度能力

**(3) 忽视多轮交互过程**

真实 agent 是：

- 长链条决策（long horizon）
- 部分可观测环境（POMDP）
- 需要逐步探索

但传统 benchmark 多为：单步任务或静态 QA

---

## 二.研究方法

AgentBoard = 一个统一 + 可分析 + 细粒度评估 LLM Agent 的 benchmark

### 1.Benchmark 设计

涵盖 4 大任务类别 + 9 个环境 共1013 environments

 **(1) Embodied AI（具身任务）**

- AlfWorld 
- ScienceWorld 
- BabyAI 

特点：

- 需要探索环境
- 动作空间复杂
- 局部可观测

**(2) Game 类任务**

- Jericho
- PDDL planning games

特点：

- 强规划能力
- 长时序依赖
- 状态隐含

**(3) Web 任务**

- WebShop 
- WebArena 

特点：

- 网页点击 / 搜索 /信息提取
- 非结构化环境

**(4) Tool 使用任务**

- Tool-Query（查询数据库）
- Tool-Operation（表格 / todo）

特点：

- API 调用
- 精确操作
- 结构化结果

**主要来自已有经典 benchmark，但经过“统一改造 + 重构 + 再标注”**

直接使用已有 benchmark 环境（改接口）：都是经典 benchmark

| 环境           | 来源                        |
| ------------ | ------------------------- |
| AlfWorld     | Shridhar et al.           |
| ScienceWorld | Wang et al.               |
| BabyAI       | Chevalier-Boisvert et al. |
| Jericho      | Hausknecht et al.         |
| WebShop      | Yao et al.                |
| WebArena     | Zhou et al.               |
| PDDL         | Vallati et al.            |

主要改动包括：

- 全部转成 纯文本 observation + action
- 统一 action interface（LLM-friendly）
- 调整 task goal 结构
- 增加 subgoal 标注
- 修复原环境问题

新增/重构部分 

- Tool-Query（构造问题 + 数据库接口）
- Tool-Operation（表格/任务操作重设计）

---

将所有任务统一为 POMDP：

Agent 工作方式（Reflex）

1. 观察环境 ot​
2. 更新 memory mt​
3. 基于 goal + history 生成 action at​
4. 环境返回新状态

**Progress Rate**

问题：

Success Rate 只能输出：0 / 1

无法表达：

- 做到一半
- 接近完成
- 有明显进步但未成功

两种 Progress Rate 计算方式

**① 连续型任务（continuous matching）**

适用于：

- Web navigation
- 某些 tool / table state

计算方式：rt​=maxf(si​,g)

 核心思想：

- 当前状态 vs goal 做“相似度匹配”
- 取历史最高 progress

**② 子目标型任务（subgoal-based）**

适用于大多数任务（Embodied / Game / Tool）

Step 1：人工拆解任务

Step 2：逐步匹配

Step 3：计算 progress

关键点

- subgoal 是 人工标注的
- 不是模型自动推断
- 每个任务都有固定 subgoal chain

---

## 三.总结

**创新点**

Progress Rate（过程级评估指标）：终点评估（final outcome）→ 过程轨迹评估（trajectory-level evaluation）

**不足**

强依赖人工 subgoal 标注

### Q&A

**1.Progress Rate 为什么比 Success Rate 更好？**

Success Rate 只能表示任务是否最终完成，是一个二值信号，因此在复杂任务中容易出现“全部模型都接近 0”的情况，缺乏区分度。Progress Rate 则通过衡量 agent 在任务执行过程中的阶段性完成程度，这使得不同模型即使最终都失败，也能通过过程差异进行比较，更适合分析 long-horizon agent 行为。

**2.Progress Rate 是如何计算的？是否有 ground truth？**

Progress Rate 并不是依赖单一标准答案。对于不同任务分为两种情况：一类是基于 subgoal 的任务，作者将整体目标人工拆解为多个可验证子目标，通过检查当前状态是否满足这些子目标来计算完成比例；另一类是连续型任务，则通过状态与目标状态的相似度函数进行匹配。




