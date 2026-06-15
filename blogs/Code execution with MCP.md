# Code execution with MCP: Building more efficient agents

https://www.anthropic.com/engineering/code-execution-with-mcp

**MCP是什么**：MCP（Model Context Protocol）是一个开放标准协议，目标是让agent（例如Claude）统一访问不同工具、服务和数据源，不需要为每种组合手工实现集成代码。

## **Excessive token consumption from tools makes agents less efficient**

随着 MCP 的使用规模扩大，有两种常见模式会增加代理的成本和延迟：

1. 工具定义使上下文窗口过载；
   
   多数 MCP 客户端会在代理上下文中预先加载所有可用工具定义。即便某任务只用到其中几个工具，也会被迫加载全部内容。

2. 工具的中间结果会额外消耗 token。
   
   在多步工具调用中，每次调用返回的数据都会被模型作为上下文读取或再次传递。

## **Code execution with MCP improves context efficiency**

让agent生成代码来调用 MCP 工具，而不是将工具 API 直接放进模型的上下文。

1. **按需加载定义** — 只载入任务需要的代码文件，而不是所有工具定义。

2. **预处理数据** — 在数据到达模型前进行过滤

## Benefits of code execution with MCP

### 渐进式披露（Progressive disclosure）

模型擅长浏览文件系统。将工具以代码形式呈现在文件系统中，允许模型按需读取工具定义，而不是一次性全部加载。

### Context efficient tool results

对于大数据集（如10,000行表格），代码执行允许代理在本地筛选、聚合甚至过滤，因此模型看到的只有精简结果，而非全部数据。

### More powerful and context-efficient control flow

代码自然支持复杂控制结构（循环、条件、错误处理），避免了传统连续调用里对于if/sleep等行为逐步交互的开销。

### Privacy-preserving operations

大块敏感数据无需穿过模型上下文，仅在代码执行环境中处理，可在返回前tokenize或部分遮蔽敏感字段，降低泄漏风险。

### State persistence and skills

具备文件系统访问权限的代码执行，使agent能够在操作间维持状态。agent可以将中间结果写入文件，从而能够恢复工作并跟踪进度。

## Challenge

代码执行本身带来复杂性。运行 agent 生成的代码需要安全的沙箱、资源限制和监控，这增加了操作成本和安全风险，而直接调用工具则不需这些。

评估代码执行的优势（如降低 token 消耗、减少延迟、增强工具组合能力）时，应权衡这些成本。

## Conclusion

MCP 为代理连接多种工具和系统提供基础协议，但连接过多服务器可能导致工具定义和结果消耗过多 token，降低效率。

虽然上下文管理、工具组合和状态持久化等问题看似新颖，但软件工程已有成熟解决方案。代码执行将这些模式应用于代理，使其能用熟悉的编程结构更高效地与 MCP 服务器交互。

## 补充

MCP = “模型调用工具的标准化协议”，重点是 **工具访问**。

A2A = “agent之间的通信协议/协作方式”，重点是 **多agent协同与信息流管理**。

如果有 m 个工具，每个工具定义 n个token，总上下文是 **m × n**

agent **生成调用这些工具的代码**，实际的工具函数不在模型上下文里，而在代码执行环境里。

上下文开销为 **m + n**——从乘法级别降到加法级别
