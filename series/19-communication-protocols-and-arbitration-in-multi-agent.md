# 19｜多智能体“怎么聊才不翻车”：通信协议、冲突仲裁与收敛机制

多智能体系统最常见的失败模式不是“不会推理”，而是：

- **吵不出结论**：无限讨论、循环确认
- **共识是错的**：群体幻觉（group hallucination）
- **对齐失败**：每个 agent 按自己的目标跑，互相拆台

所以这篇只谈一个核心：**多智能体的通信协议（protocol）与仲裁机制（arbitration）**。

---

## 1) 把对话当协议：消息需要结构化，不要只靠自然语言

如果你的 agent 之间只交换自然语言，你很难做到：

- 约束消息范围（避免 prompt 膨胀）
- 自动判定“是否收敛”（什么时候该停）
- 在冲突出现时触发仲裁

因此工程上最实用的做法是：

- 消息分为：proposal / critique / decision / action_request / action_result
- 关键字段结构化：assumptions、evidence、risk、cost、next_step

这会直接映射到本仓库的 **Schema-first + Observable**：

- Schema：消息格式可校验
- Observable：可度量“讨论回合数”“冲突次数”“仲裁触发率”

---

## 2) 代表性论文切入点：CAMEL 的启发是什么

- **CAMEL: Communicative Agents for "Mind" Exploration of Large Language Model Society**
  - arXiv: https://arxiv.org/abs/2303.17760

你不必照搬它的设定，但它很重要的启发是：

- 多智能体的价值很大程度来自“沟通被约束”
- 角色设定（role prompt）可以视作一种“协议初始化”

工程翻译：

- 在对话开始前，先把任务目标、成功标准、可用工具与禁用边界写清
- 每回合明确谁输出什么类型的消息（proposal/critique/...）

---

## 3) 冲突与仲裁：必须引入“裁判”，否则成本会爆炸

当你引入多个 agent，你需要显式定义：

- 什么算冲突（结论相反？风险评估差异？工具选择不同？）
- 冲突如何解决（投票？裁判模型？规则优先？人类审批？）

常见仲裁策略（从便宜到贵）：

1. **规则仲裁**：优先级、硬约束、合规规则（最稳）
2. **裁判 agent**：给一个 judge 角色负责 decision
3. **人类介入**：高风险/高成本时触发

落地建议：优先把“硬约束”写进 Governance（权限/预算/黑白名单），用规则直接截断。

---

## 4) 收敛机制：让系统知道“什么时候该停”

多智能体如果没有终止条件，系统会在讨论中消耗预算。

建议至少有三条收敛阈值：

- max_turns：最大回合数
- max_cost：最大 token/API 成本
- success_signal：达到可验证里程碑（例如：测试通过/接口返回成功/工件验收通过）

这其实就是 Runtime 层的“状态机”。

---

## 5) 映射到企业 Agent-Native：协议与仲裁属于 Runtime + Governance，不是 prompt

把它落到五层架构：

- **Runtime**：回合调度、终止条件、重试与 replan
- **Governance**：角色权限、风险分级、仲裁规则、人工审批点
- **Observable**：冲突率、仲裁率、收敛时间、成本分布

一句话：你最终在做的是“协作系统”，不是“聊天系统”。

---

## 参考阅读

- CAMEL（arXiv:2303.17760）：https://arxiv.org/abs/2303.17760
- AutoGen（arXiv:2308.08155）：https://arxiv.org/abs/2308.08155
- 本仓库系列：series/README.md
- 长文全文：README.md
