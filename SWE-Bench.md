# SWE-Bench: Can Language Models Resolve Real-World GitHub Issues?

**ICLR 2024**

## 一.研究背景

### 1.现有代码评测已经“太简单”了

在 ChatGPT、GPT-4、Claude 等模型出现后，很多代码能力测试（如 HumanEval）已经接近被“刷满分”。

但这些基准有一个共同特点：

- 输入很短
- 任务独立
- 通常只需写几十行代码
- 不需要理解大型项目

例如：给一个函数说明，实现该函数。

这种任务更像编程考试题。

### 2.真实软件开发完全不同

现实中的软件工程师修 Bug 时需要：

- 阅读上万行代码
- 理解多个文件之间关系
- 找到 Bug 所在位置
- 修改多个函数
- 保证旧功能不被破坏
- 通过所有测试

例如：GitHub Issue"Gradient Boosting 在某种情况下会产生数据泄露"

工程师需要：

1. 阅读 Issue
2. 定位相关模块
3. 修改代码
4. 添加测试
5. 验证通过

这远比 HumanEval 难得多。

### 3.作者提出的问题

作者希望回答：“如果直接给大模型一个真实 GitHub Issue，它能像工程师一样修复吗？”

于是构建了：SWE-Bench

一个真实软件工程能力测试集

---

## 二.研究方法

**数据来源**

作者从：

- Django
- Scikit-learn
- Matplotlib
- Flask
- Requests

等 12 个知名 Python 开源项目收集数据总共抓取约 90,000 个 Pull Request。

### 三阶段筛选流程

**Stage 1：抓取 Pull Request**

从热门 GitHub 仓库收集：

- Issue
- Pull Request
- Commit

信息。

**Stage 2：属性过滤**

只保留满足条件的 PR：

- 已合并（Merged）

- 解决某个 Issue

- 修改了测试代码

因为：修改测试说明开发者专门验证了这个 Bug。

**Stage 3：执行验证**

作者实际运行代码：

修复前：Test FAIL

修复后：Test PASS

必须出现：FAIL → PASS这样的变化。

否则说明：

- Bug 不明确
- 测试无效
- 修复无意义

会被剔除。

最终得到的数据集

从90,000个 PR中

筛选到**2,294**个高质量软件工程任务。

---

### 任务形式

模型会收到：

**1.GitHub Issue**

例如：

```
HGBC with categorical_crossentropy fails silently on binary classification
```

描述 Bug。

**2.项目代码库**

可能包含：

- 几千个文件
- 数十万行代码

模型需要生成：

```
- old code
+ new code
```

形式的 Patch（补丁）。

然后自动部署到到项目里。

---

可以把 SWE-Bench 看成一个包含 **2294 道真实软件工程题目** 的考试。

每一道题（一个 task）对应：

GitHub Issue、对应版本的代码仓库、用于验证修复是否正确的测试集

模型需要完成：

Issue描述
      ↓
理解问题
      ↓
阅读代码库
      ↓
找到需要修改的位置
      ↓
生成Patch
      ↓
运行测试

如果测试通过：1分

否则： 0分

最后统计：

<img src="file:///C:/Users/LR/AppData/Roaming/marktext/images/2026-06-02-20-49-04-image.png" title="" alt="" width="276">

---

## 三.实验设计

**评价方式**

采用：真正运行测试

流程：

Issue
 ↓
LLM生成Patch
 ↓
应用Patch
 ↓
运行测试
 ↓
是否全部通过

如果：所有相关测试 PASS

则认为：Issue Resolved

成功修复。

**测试模型**

包括：

- ChatGPT-3.5
- GPT-4
- Claude 2
- SWE-Llama 7B
- SWE-Llama 13B

其中 SWE-Llama 是作者自己训练的模型。

### 实验结果

| 模型            | 解决率    |
| ------------- | ------ |
| Claude 2      | 1.96%  |
| SWE-Llama 13B | 0.70%  |
| SWE-Llama 7B  | 0.70%  |
| ChatGPT-3.5   | 0.17%  |
| GPT-4         | 0.00%* |

即使最强的 Claude 2：2294个任务只解决约45个——成功率不到 2%。

### 原因分析

**1.代码库太大**

平均：43.8万行代码

模型很难知道：Bug到底在哪个文件

**2.长上下文会干扰模型**

给模型更多代码，性能反而下降。

因为模型会：

- 被无关代码干扰
- 找不到真正需要修改的位置

论文引用了“Lost in the Middle”现象。

**3.修改往往跨文件**

真实 PR 平均修改：

- 1.7 个文件
- 3 个函数
- 32.8 行代码

很多 Bug 需要：文件A改一点、文件B改一点、文件C补测试

而不是单函数补全。


