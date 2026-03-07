# 18｜多智能体协作研究速览：从“一个人干活”到“团队作业”的论文脉络

当单智能体（single agent）开始做长链任务，你很快会碰到瓶颈：

- 同时要“想方案、查资料、写代码、测回归、写文档”，上下文和注意力都会被打散
- 一旦出错，既要定位问题又要修复，单体循环很容易陷入自证/自嗨
- 任务越复杂，越需要**分工、对齐、复核与交接**

于是“多智能体”（multi-agent）成为一条自然路线：让多个专长角色通过对话协作完成任务。

这篇给你一张阅读地图：主流论文大致在解决哪些协作问题，以及落地到企业 Agent-Native 时，真正要补的工程部件是什么。

---

## 1) 一个统一框架：多智能体 = 角色分工 + 通信协议 + 协作记忆 + 仲裁机制

你可以用下面四个问题把多智能体问题讲清楚：

1. **谁负责什么（Roles）**：角色怎么定义？是静态岗位还是动态分配？
2. **怎么沟通（Protocol）**：沟通模板、消息格式、共享上下文边界？
3. **怎么共享状态（Shared State/Memory）**：共享哪些工件？如何版本化？
4. **怎么做决策（Arbitration）**：冲突谁裁决？怎么防止“吵起来”？

如果只做“多开几个 prompt 互相聊天”，你得到的通常是：更贵、更慢、更不稳定。

---

## 2) 论文脉络（按工程问题分组）

### A. 通信与角色扮演：让角色对话更“有组织”（CAMEL）

- **CAMEL: Communicative Agents for "Mind" Exploration of Large Language Model Society**
  - arXiv: https://arxiv.org/abs/2303.17760
  - 关注点：通过角色设定与对话约束，让两个 agent 形成可控协作

工程映射：

- 这类工作更像是在定义 **“协议”**：对话如何开始、如何推进、如何收敛
- 企业落地还缺：工具调用、权限、可观测与审计（否则只是“会聊天的角色扮演”）

### B. 多智能体应用框架：把对话变成可复用基础设施（AutoGen / AgentVerse）

- **AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation**
  - arXiv: https://arxiv.org/abs/2308.08155
- **AgentVerse: Facilitating Multi-Agent Collaboration and Exploring Emergent Behaviors**
  - arXiv: https://arxiv.org/abs/2308.10848

工程映射：

- 重点是“怎么编排多个 agent 的回合、怎么插入工具、怎么让人参与”
- 真正的上线关键：把消息、状态、工件做成 **Runtime 的一等公民**（可回放、可追责、可限额）

### C. 面向软件工程的团队协作：把人类研发流程嵌入多智能体（ChatDev / MetaGPT）

- **ChatDev: Communicative Agents for Software Development**
  - arXiv: https://arxiv.org/abs/2307.07924
- **MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework**
  - arXiv: https://arxiv.org/abs/2308.00352

工程映射：

- 核心不是“更多 agent”，而是“更像团队”：PRD → 设计 → 实现 → 测试 → 交付工件
- 对企业 Agent-Native 的启发：把多智能体协作落到 **工件流（artifact flow）** 上，而不是对话流

---

## 3) 映射到五层架构：多智能体会放大哪些工程缺口

多智能体把单智能体的问题放大一倍：

- **Interface/Capability**：每个角色都要调用工具，契约不清晰会指数级翻车
- **Data & Memory**：共享记忆会引入污染、越权、泄露；需要作用域与隔离
- **Runtime**：你需要一个“调度器/导演”来控制回合、终止条件、并发与重试
- **Governance**：权限要按角色分配（RBAC/ABAC），并且要有仲裁机制

一句话：多智能体不是“更聪明”，是“更复杂”，所以必须更工程。

---

## 4) 落地建议：先把协作做成“可审计的工件协同”，再谈 emergent behavior

企业里最稳的落地路径通常是：

1. 明确角色与权限（谁能调用哪些能力）
2. 规定输出工件（每个角色要交付什么）
3. 用 Runtime 记录完整协作链路（可回放）
4. 在关键点引入仲裁（规则/人类/裁判模型）

把这些底座做稳以后，多智能体的优势才会真正出现：并行、复核、分工与更高的复杂度上限。

---

## 参考阅读

- CAMEL（arXiv:2303.17760）：https://arxiv.org/abs/2303.17760
- AutoGen（arXiv:2308.08155）：https://arxiv.org/abs/2308.08155
- AgentVerse（arXiv:2308.10848）：https://arxiv.org/abs/2308.10848
- ChatDev（arXiv:2307.07924）：https://arxiv.org/abs/2307.07924
- MetaGPT（arXiv:2308.00352）：https://arxiv.org/abs/2308.00352
- 本仓库系列：series/README.md
- 长文全文：README.md
