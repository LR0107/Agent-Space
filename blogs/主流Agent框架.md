# 主流Agent框架

OpenClaw是全平台个人助手，Hermes Agent是自进化Agent，Claude Code是产品级编程Agent。它们都是Harness Engineering的具体实现

### 底层设计

### Agent Loop的实现方式

三个框架的核心都是一个循环：接收任务→思考→执行→观察→继续或结束。但循环的"编排方式"不同。

**OpenClaw：单Agent线性循环**

OpenClaw是最简单的结构——一个Agent跑一个循环，你给它消息，它调工具，返回结果。没有子Agent，没有复杂编排，就是"输入→工具调用→输出"的线性流程。

简单但意味着复杂任务要么靠Prompt拆，要么靠外部系统调度。

**Hermes Agent：单Agent + 子Agent并行委派**

Hermes在单Agent循环的基础上，加了子Agent并行委派能力——主Agent可以把子任务派给子Agent并行执行，结果汇总后主Agent继续决策。

更关键的是，Hermes在循环里嵌了一个学习闭环：

```
执行任务 → 总结经验 → 生成skill → 存入记忆 → 下次复用
```

这让Hermes不是简单的"跑完就忘"，而是越跑越强。

**Claude Code：while循环 + 三种子Agent**

Claude Code的核心就是一个while循环——不断"思考→行动→观察"，直到模型自己判断任务完成。

但它在这个简单循环上做了三种子Agent扩展：Explore（搜索探索）、Plan（规划拆解）、General-purpose（通用执行）。子Agent是独立上下文窗口，跑完把结果摘要返回主Agent，不污染主窗口。

|        | OpenClaw   | Hermes Agent      | Claude Code          |
| ------ | ---------- | ----------------- | -------------------- |
| 核心循环   | 单Agent线性   | 单Agent + 子Agent并行 | while循环 + 三种子Agent   |
| 编排方式   | 线性执行       | 可并行委派             | 串行为主，子Agent独立上下文     |
| 学习能力   | 无          | 学习闭环              | 无（但靠CLAUDE.md积累项目知识） |
| 复杂任务处理 | 靠Prompt拆解  | 子Agent并行          | 子Agent隔离执行           |
| 语言     | TypeScript | Python            | TypeScript           |

 总结：三个框架核心都是ReAct循环。再说差异：OpenClaw是线性执行，适合单任务场景；Hermes加了并行委派和学习闭环，适合需要积累经验的长期任务；Claude Code用子Agent隔离复杂任务，每个子Agent有独立上下文窗口，不会互相污染。

---

## 记忆机制

### OpenClaw：文件注入 + 本地持久化

OpenClaw的记忆靠三个文件：

- **AGENTS.md**——项目级规范和上下文，每次对话都注入

- **SOUL.md**——Agent的"灵魂"，定义角色、性格、说话风格

- **TOOLS.md**——可用工具的说明和规则

每次对话开始，这三个文件的内容直接塞进System Prompt。对话过程中的中间结果，写到本地文件系统持久化。

问题在于：**OpenClaw没有跨会话记忆**。每次对话都是"新手"，不会记住上次聊了什么、做过什么。AGENTS.md是你手动写的规则，不是Agent自己学到的经验。

### Hermes Agent：分层记忆 + 用户建模 + 跨会话搜索

Hermes的记忆是三个框架里最深的，分了好几层：

**第一层：持久化记忆**——对话历史和执行结果存到本地，下次启动可以加载。

**第二层：用户建模**——Hermes内置了一个用户建模系统，会主动学习和记录用户的偏好、习惯、工作方式。不是简单的"记住你上次说了什么"，而是"理解你是哪种类型的人"。

**第三层：跨会话搜索**——用SQLite的FTS5全文搜索引擎，在历史对话中检索相关片段。你问"上次那个部署脚本怎么写的"，Hermes能从过去的会话里搜出来。

**第四层：自我督促**——Hermes会主动提醒自己"这个用户上次不喜欢长回复"、"这个项目用TypeScript"。相当于Agent自己给自己写备忘录。

最关键的是，Hermes的记忆不只是"记住"，还能**转化为技能**——执行完一个复杂任务，自动从经验中提取模式生成一个skill文件，存在`~/.hermes/skills/`，下次遇到类似任务直接复用。

### Claude Code：CLAUDE.md + .claude/目录 + 外化状态

Claude Code的记忆系统有三个层次：

**第一层：CLAUDE.md（项目知识）**——放在项目根目录，定义项目规范、技术栈、代码风格。每次对话都注入，但不是放在System Prompt里，而是作为用户消息注入——优先级低于安全规则，但高于普通用户消息。这是Anthropic做的一个安全设计。

CLAUDE.md还有层级结构：根目录的`CLAUDE.md`全局生效，子目录的`CLAUDE.md`只在进入该目录时注入。这样不同模块可以有不同的规范。

**第二层：.claude/目录（会话状态）**——Agent的中间状态、任务进度、分析结论都外化到文件系统。这就是Harness Engineering里说的"状态外化"——不在上下文窗口里存状态，而是写到文件里。

好处是：即使Context Reset（整个上下文窗口丢掉换新的），从文件里一读就知道"现在到哪一步了"。

**第三层：记忆文件系统**——Claude Code支持在`~/.claude/`目录下存放跨项目的长期记忆，比如用户偏好、常用命令模式。

---

## 工具调用

### OpenClaw：MCP协议 + ClawHub技能市场

OpenClaw的工具系统基于**MCP协议**，所有工具都通过MCP标准化接口接入。

它还有一个**ClawHub技能市场**——社区贡献的工具和技能包，装上就能用。这就像手机上的App Store，技能生态丰富是OpenClaw的一大优势。

执行环境方面，OpenClaw用Docker沙箱来隔离工具执行——工具跑在沙箱里，不会直接操作宿主系统。

### Hermes Agent：MCP + 自生成技能

Hermes也用MCP协议，但在工具系统上做了两个升级：

**第一，技能可以自动生成**——不是只靠社区贡献，Agent执行完任务后能自己总结出一套技能，存在`~/.hermes/skills/`，遵循`agentskills.io`标准。下次遇到类似任务，搜索已有技能直接复用。

**第二，工具白名单机制**——不是所有MCP工具都给Agent用，而是根据任务动态决定哪些工具可用。减少Agent"选错工具"的概率。

Hermes还支持6种终端后端（本地、Docker、SSH、K8s等），适应不同部署环境。

### Claude Code：18+内置工具 + 权限分级

Claude Code没有用MCP协议（虽然支持MCP Server接入），它走的是**专用内置工具**路线——18+工具全部在代码里定义，每个工具有严格的参数Schema和使用规则。

最关键的设计是权限分级：

```
deny > ask > allow
```

每个工具调用前，先查权限表：

- **deny**：直接拒绝，不执行

- **ask**：弹窗问用户，用户确认才执行

- **allow**：直接执行

---

## 上下文管理

### OpenClaw：文件注入 + 动态裁剪

OpenClaw的上下文管理比较简单：每次对话开始，把AGENTS.md、SOUL.md、TOOLS.md的内容注入，然后对话历史逐步累积。

窗口快满的时候，OpenClaw的策略是**动态裁剪**——按时间顺序把最早的对话内容裁掉，保留最近的。这和大多数聊天应用的"滑动窗口"一样，简单粗暴但没有更精细的策略。

早期把AGENTS.md写成百科全书，内容越来越长，模型注意力被严重稀释。后来改成"目录页"模式——主文件只保留约100行核心索引，详细内容拆到子文档，Agent按需加载。这就是**渐进式披露（Progressive Disclosure）**。

### Hermes Agent：just-in-time retrieval + 分层注入

Hermes的上下文管理更精细，核心是**just-in-time retrieval**——不是一开始就把所有信息塞进去，而是Agent边干活边按需抓取。

分层注入策略：

- **始终注入**：AGENTS.md核心规则、当前任务目标

- **按需加载**：技能详情、历史会话片段、工具说明

- **动态替换**：根据当前步骤，把不再需要的上下文替换成新的

Hermes还用FTS5做上下文检索——不是把所有历史对话都塞进窗口，而是根据当前任务搜索最相关的片段，只把相关片段注入。

### Claude Code：200K窗口 + 三层压缩 + Context Reset

Claude Code的上下文管理是三个框架里最工程化的，分三层：

**第一层：对话历史管理**

200K Token的上下文窗口，按优先级排列：

1. System Prompt（~8,700 Token，不可压缩）

2. 对话历史（最近N轮完整保留）

3. 工具返回结果（大文件自动截断）

**第二层：自动压缩**

当上下文接近窗口上限时，Claude Code自动触发压缩：

- 早期对话压缩成摘要

- 工具返回的大文件只保留关键片段

- 子Agent的执行结果只保留摘要，不保留完整过程

**第三层：Context Reset**

这是Anthropic在Harness Engineering里提出的关键方案——当压缩都不够时，直接把整个上下文窗口丢掉，换一个干净的。

---

## 总结

它们都是Harness Engineering的具体实现——Agent = Model + Harness，这三个框架做的都是Harness这一层。共性是都用了ReAct循环、都支持MCP工具协议、都做了上下文管理。差异是：OpenClaw重工具生态，Hermes重学习闭环，Claude Code重安全机制。


