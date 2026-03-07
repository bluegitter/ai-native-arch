# 23｜多智能体综述与架构模式：把“研究百花齐放”压缩成可执行的架构选型

多智能体论文非常多，最容易读着读着迷路：

- 今天看辩论，明天看社会模拟
- 今天看软件工程团队，明天看机器人协作

这篇的目标是：用两篇代表性 survey 帮你建立“架构选型地图”，并给出几种可落地的模式。

---

## 1) 两篇代表性综述：用它们统一词汇与分类

- **Large Language Model based Multi-Agents: A Survey of Progress and Challenges**
  - arXiv: https://arxiv.org/abs/2402.01680

- **A Survey on LLM-based Multi-Agent System: Recent Advances and New Frontiers in Application**
  - arXiv: https://arxiv.org/abs/2412.17481

这两篇常见分类方式是：

- Task-solving（任务求解型）
- Simulation（社会/场景模拟型）
- Evaluation（用于评估 generative agents 的环境）

工程上你可以把它简化成一句话：

> 企业落地 90% 属于 task-solving；simulation 更像产品/研究；evaluation 是工程护城河。

---

## 2) 架构模式：多智能体系统常见的 4 种组织结构

### 模式 A：扁平协作（Peer-to-peer）

- 特点：多个 agent 平等讨论
- 适用：探索性问题、创意发散
- 风险：不收敛、成本高

### 模式 B：导演/调度（Orchestrator-centric）

- 特点：一个 orchestrator 负责调度多个专家 agent
- 适用：需要收敛、需要可控
- 工程关键：状态机、终止条件、预算、回放

### 模式 C：层级组织（Hierarchical）

- 特点：Manager 拆解任务、Worker 执行
- 适用：长链任务、强约束流程（审批/权限）
- 工程关键：派工与验收、升级路径（escalation）

### 模式 D：工件驱动（Artifact-centric）

- 特点：协作围绕工件（PRD/设计/代码/测试）流转
- 适用：软件工程、文档生产、知识库建设
- 工程关键：模板、版本、质量门禁

---

## 3) 映射到本仓库五层架构：选型时看“你缺哪一层”

选型不要从“哪个框架火”出发，而从你系统缺口出发：

- 缺 **Interface/Schema**：先把工具契约与数据契约做稳
- 缺 **Runtime**：先把调度、回放、终止条件做成状态机
- 缺 **Governance**：先把权限/审计/预算做硬约束

多智能体会放大缺口：底座不稳，上层越堆越乱。

---

## 4) 实用建议：用最小模式落地，再逐步升级

一个很稳的企业演进路线：

1. 单智能体 + reviewer（先把质量门禁跑通）
2. orchestrator + 专家 agent（把分工做成可控调度）
3. hierarchical（把派工/验收/升级流程制度化）
4. artifact-centric（把协作沉淀成可复用工件资产）

---

## 参考阅读

- Multi-Agent Survey（arXiv:2402.01680）：https://arxiv.org/abs/2402.01680
- LLM-MAS Survey（arXiv:2412.17481）：https://arxiv.org/abs/2412.17481
- 本仓库系列：series/README.md
- 长文全文：README.md
