# AppWorld: A Controllable World of Apps and People for Benchmarking Interactive Coding Agents

**ACL 2024**

## 一.研究背景

当前大语言模型（LLM）在“工具使用 / agent能力”方面的评测存在明显不足：

现有 benchmark 多为：

- 单次或少量 API 调用（1–4步）
- 线性流程
- 不需要复杂代码逻辑

缺少真实数字生活中的复杂性：

- 多应用协作（如 Gmail + Amazon）
- 状态依赖（前一步结果影响后续操作）
- 需要写“程序级别逻辑”（循环、条件、错误处理）
- 动态交互（需要不断试错）

 **真实世界任务的特点**

论文指出真实数字任务通常具备：

- 多应用交互（multi-app workflows）
- 多步骤推理 + 编程
- 环境反馈驱动决策（interactive decision making）

---

## 二.研究方法

### 1. AppWorld Engine（模拟数字世界）

构建一个“可控的数字世界”，模拟真实应用和用户行为。

**（1）9类模拟应用**

包括：

- Gmail（邮件）
- Amazon（购物）
- Spotify（音乐）
- Venmo（转账）
- Todo / Notes / FileSystem 等

共：

- 457 APIs
- 101数据库表
- 模拟约100个虚拟用户

**（2）API设计特点**

- 接近真实应用接口（REST风格）
- 支持状态变化（stateful）
- 支持跨应用联动（例如下单 → 发邮件确认）
- 支持权限与错误机制

**（3）执行环境**

类 Jupyter Notebook

支持：

- 多轮代码执行
- 变量保留
- API调用
- 错误反馈（traceback）

**（4）可控数据库系统**

- 106个虚拟用户
- 370K+ 数据记录
- 模拟真实数字生活行为（朋友、室友、工作关系等）

**DB = 整个数字世界的“真实状态”**

在 AppWorld 里：

- Gmail 邮件存在 DB 表
- Amazon 订单存在 DB 表

所有“应用”，只是 DB 的不同视图 + API 操作接口

**agent 本质是：通过 API / 代码 → 修改数据库（DB）**

## 2.AppWorld Benchmark（任务集）

构建 **750**个复杂任务，用于测试 agent 的真实能力。

**（1）任务特点**

每个任务通常：

- 涉及 1–6 个应用
- 使用平均 9.5 个 API
- 需要约 40–60 行代码
- 需要循环 / 条件 / 错误处理
- 依赖中间结果再决策

**（2）任务生成方式**

每个任务由“场景模板”生成

流程：

1. Setup（构造初始状态）
   - 修改数据库
   - 设置时间
   - 注入干扰信息（distractors）
2. Task Input
   - 给 agent 一个自然语言指令
3. Evaluation Data
   - 定义正确答案的“状态变化”

一个 scenario 可以变成很多 instance

agent：写代码 + 调 API + 操作这个虚拟世界

**（3）难点控制**

任务刻意加入：

- Distractors（干扰信息）
- Hurdles（现实障碍，如支付卡过期）
- Cross-app dependency（跨应用依赖）
- Contrast sets（同一任务不同变体）

### 评估方法

**传统问题**

以前方法：

- 对比“参考代码”
- 或 LLM judge
- 或人工判断

问题：不适用于多解任务

**AppWorld方法：状态级评估（State-based Evaluation）**

核心思想：**不看过程，只看最终数据库状态**

不看“过程”，只看“结果”

举例：

任务：订购 T-shirt

agent 可以：

- 用 Amazon API
- 或 email API
- 或 search + order

只要最终 DB 对，就算对

约束条件是 任务生成系统：人工设计 + 程序化定义的

```
Agent执行 → 修改DB  
↓  
计算DΔ（变化）  
↓  
Evaluation Program检查约束  
↓  
得到 task pass/fail  
↓  
统计 TGC  
↓  
按scenario聚合得到 SGC
```

---

### 工作流程

 ① 构造世界（Engine）

用数据库模拟 Gmail / Amazon / Spotify 等

所有数据都在 DB 里

② 生成任务（Task）

从场景模板（任务类型） + DB真实数据生成具体问题

加入干扰信息（更真实、更难）

③ Agent 执行

写 Python 代码

调用 API

间接修改 DB

④ 自动评分

只看 DB 最终状态

对比初始 DB vs 最终 DB

检查：

必须发生的变化

不允许的变化

---

## 三.实验设计

### **实验设置**

**方法**

- ReAct
- Plan-and-Execute
- CodeAct
- ToolLLaMA
- FullCode + Reflexion
- IP Function Calling

**模型**

- GPT-4o
- GPT-4 Turbo
- LLaMA 3
- DeepSeek Coder
- Mistral

**评估指标**

- **TGC（Task Goal Completion）**：单个任务是否完成
- **SGC（Scenario Goal Completion）**：同一场景多个任务是否全部完成

**数据划分**

- Train / Dev / Test-N（正常）
- Test-C（挑战集）

Test-C 包含 unseen app（如 Gmail / Amazon）更难，更真实

### 实验结果

**（1）整体难度很高**

最强模型 GPT-4o：

- Test-N ≈ 49%
- Test-C ≈ 30%

**（2）模型失败原因**

主要问题：

- 不进行真实环境交互（幻想数据）
- API使用错误
- 复杂逻辑处理失败
-  忘记状态（memory问题）

**（3）API不是主要瓶颈**

即使给“正确API”，提升有限

真正难点是：

- 多步代码逻辑
- 动态环境适应
- 错误恢复

---

## 四.总结

### 创新点

构建了一个 统一数据库驱动的数字世界

提出 state-based evaluation（不用参考答案）

### 不足

环境仍然是 DB 模拟，不是真实 UI/互联网

### Q&A

**1.AppWorld 和普通 tool-use benchmark 有什么区别？**
1）任务更复杂（多应用 + 多步代码逻辑）  
2）需要交互式执行（不是一次性输出）  
3）评估方式不同（基于最终状态而不是参考答案）

**2.为什么一定要引入数据库（DB）？**

因为 AppWorld 本质是一个“状态驱动系统”。所有应用（Gmail、Amazon）都被抽象成数据库表。这样可以统一表示系统状态，并且可以通过数据库变化来判断任务是否完成。

**3.AppWorld 怎么评分？是不是有标准答案？**

没有固定标准答案。  
评分是基于“数据库状态是否满足约束条件”。系统检查：

- 必须发生的变化是否出现
- 是否有不允许的副作用
- 是否满足任务约束


