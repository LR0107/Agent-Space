# Equipping agents for the real world with Agent Skills

[原文链接](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

**Agent Skills**：由指令、脚本和资源组成的文件夹，Agent可以动态发现和加载这些技能，从而在特定任务上表现得更好。

A skill is a directory containing a **SKILL.md** file that contains organized folders of instructions, scripts, and resources that give agents additional capabilities.

---

## The anatomy of a skill

Skill 就是一个包含 **SKILL.md** 文件的目录。该文件必须以 YAML 开头，其中包含一些必需的元数据：**名称** 和 **描述**。在启动时，代理会将每个已安装技能的名称和描述预先加载到系统提示中。

这些元数据构成了技能的第一级渐进式信息披露：它提供了足够的信息，让 Claude 知道何时应该使用某个技能，而无需将技能的全部内容加载到上下文中。

文件的实际主体部分则是第二级细节。如果 Claude 认为某个技能与当前任务相关，它会通过将完整的 **SKILL.md** 文件加载到上下文中来使用该技能。

随着技能的复杂度增加，它们可能包含过多的上下文信息而无法完全放入单个 **SKILL.md** 文件，或者包含仅在特定场景下相关的内容。在这种情况下，技能可以在技能目录中捆绑额外的文件，并在 **SKILL.md** 中通过名称进行引用。这些额外的关联文件构成了技能的第三级（及更高级别）细节，Claude 仅在需要时才会选择访问和使用它们。

在下面展示的 PDF 技能中，**SKILL.md** 文件引用了两个额外文件（**reference.md** 和 **forms.md**），这是与**SKILL.md** 一起捆绑的。Claude 仅在填写表单时才会读取 **forms.md**

![](file://C:\Users\LR\AppData\Roaming\marktext\images\2026-05-16-11-04-43-image.png?msec=1778900683917)

渐进式信息披露是使 Agent Skills 灵活且可扩展的核心设计原则。就像一本结构清晰的手册，先有目录，然后是具体章节，最后才是详细附录，技能允许 Claude **仅在需要时加载信息**。

拥有文件系统和代码执行工具的 Agent 在处理特定任务时，不需要将整个技能的内容全部加载到上下文窗口中。这意味着可以捆绑到技能中的上下文量实际上是**无限的**。

---

### 工作流程

1. **初始化**：上下文窗口包含核心系统提示、每个已安装技能的元数据，以及用户的初始消息。
2. **技能判断**：Claude 判断需要哪一个技能。
3. **调用技能**：调用文件系统读取该技能的 **SKILL.md** 文件和关联文件。
4. **完成任务**：最终完成用户任务。

这种机制使得 Agent 只把相关技能内容纳入上下文，避免过度占用 token。

PDF 技能包含一个预编写的 Python 脚本，用于读取 PDF 并提取所有表单字段。Claude 可以执行这个脚本，而无需将脚本或 PDF 加载到上下文中。由于代码具有确定性，这种工作流程能够保持一致且可重复。

![](file://C:\Users\LR\AppData\Roaming\marktext\images\2026-05-16-11-14-00-image.png?msec=1778901240921)

Skill 不仅仅是说明性文本，还可以包含可执行代码：

- 某些操作更适合传统代码来实现（如排序、解析等），Skill 可以将这些脚本纳入其中。
- Claude 可自主决定何时执行这些代码片段。
- 代码执行的结果不必全部载入模型上下文。

---

## Developing and evaluating skills

- **从评估开始**：让 Agent 执行任务，观察困难点和需要额外上下文的地方，识别能力缺口，然后逐步构建技能弥补不足。
- **结构化以便扩展**：当 **SKILL.md** 文件过大时，将内容拆分为独立文件并引用；互斥或不常同时使用的上下文分开可减少 token 消耗。代码既可作为可执行工具，也可作为文档，并明确 Claude 是直接执行还是参考加载。
- **从 Claude 的角度思考**：监控 Claude 在实际场景中的技能使用，关注意外操作路径或过度依赖的上下文，特别注意技能名称和描述，因为 Claude 会据此决定是否触发技能。
- **与 Claude 一起迭代**：在任务中让 Claude 捕获成功方法和常见错误到技能的上下文和代码中；若偏离方向，要求其自我反思。通过此过程发现实际需要的上下文，而不是事先完全预测。
