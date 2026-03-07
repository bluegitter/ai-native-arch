# 16｜长链任务怎么做稳：Plan-then-Execute / Plan-and-Act 的论文脉络与工程落地

一旦你把 Agent 从“单轮问答”推向“长链任务”（多步、跨系统、跨时间），失败率会快速上升。

原因并不神秘：

- 计划（Plan）需要全局视角；执行（Act）需要局部细节
- 把两者混在一个 prompt 里，会让模型在中途被细节带偏
- 计划越长，越容易在第 N 步偏离目标，且很难恢复

因此近两年一个非常实用的研究与工程共识是：**明确分离规划与执行**。

---

## 1) 一句话解释 Plan-then-Execute（P-t-E）

> 用一个“Planner”生成可检查的计划；再用一个“Executor”按步骤执行，并把执行反馈回灌给 Planner 迭代。

你可以把它理解成“把长链任务变成一个可控的状态机”，而不是让模型一次性脑补到底。

---

## 2) 代表性论文/框架（读它们主要是拿思路，不是抄实现）

### A. Plan-and-Act：显式分离规划与执行，提升长链任务稳定性

- **Plan-and-Act: Improving Planning of Agents for Long-Horizon Tasks**
  - arXiv: https://arxiv.org/abs/2503.09572
  - 关键词：explicit separation of planning and execution

工程启发：

- Planner 输出最好是结构化的（JSON/DSL），方便校验与审计
- Executor 的每一步都要有“预期效果”（expected outcome），否则你无法判断偏航

### B. Plan-then-Execute 的安全/韧性视角：把“可控性”当一等公民

- **Architecting Resilient LLM Agents: A Guide to Secure Plan-then-Execute Implementations**
  - arXiv: https://arxiv.org/abs/2509.08646

工程启发：

- 规划阶段可以做：权限预检、风险分级、资源预算
- 执行阶段可以做：逐步授权（step-up auth）、失败回滚、转人工接管

---

## 3) 映射到本仓库五层架构：P-t-E 不是“提示词技巧”，而是运行时设计

把 P-t-E 做成生产系统，关键点在 Runtime + Governance：

- **Runtime：两段式循环**
  1) plan（产生可执行 plan）
  2) execute step（执行一步）
  3) observe（记录结果/错误/成本）
  4) replan（必要时重规划）

- **Governance：把 plan 当成“待审批的意图”**
  - 高风险动作必须在计划层就暴露出来（例如：转账、删库、改权限）
  - 你可以对“计划”而不是对“模型输出句子”做审批

- **Interface/Capability：让步骤具备契约**
  - 每一步调用的工具必须是 schema 化的（输入输出、错误语义、幂等性）

---

## 4) 实用落地建议：三件事做对了，P-t-E 才会显著变稳

### 建议 1：让 Planner 输出“可验证计划”

最少包含：

- step_id / action / tool / parameters
- precondition（前置条件）
- expected_observation（期望看到什么）
- rollback_or_escape（失败怎么办）

### 建议 2：给 Executor 一条“偏航检测”规则

Executor 每做一步都要问：

- 这一步的结果是否满足 expected_observation？
- 如果不满足，是重试、换工具、还是触发 replan？

### 建议 3：把“预算”写进运行时

长链任务最常见的线上事故是“越做越贵、越做越慢”。

因此你的 Runtime 至少需要：

- token/时间/调用次数预算
- 失败重试上限
- 人工接管阈值

---

## 参考阅读

- Plan-and-Act（arXiv:2503.09572）：https://arxiv.org/abs/2503.09572
- Resilient Plan-then-Execute（arXiv:2509.08646）：https://arxiv.org/abs/2509.08646
- 本仓库系列：series/README.md
- 长文全文：README.md
