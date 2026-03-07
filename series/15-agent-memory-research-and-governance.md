# 15｜Agent 记忆研究速览：从“能记住”到“可治理”的长期记忆体系

企业里做 Agent，最容易把“记忆”理解成一个向量库（RAG）——但很快你会遇到三类现实问题：

- **记不住**：上下文窗口不够，跨会话就失忆
- **记错了**：把一次事故/幻觉当“经验”固化进长期记忆
- **记了也没用**：回忆（retrieve）出来的信息与当前任务无关，反而干扰决策

这篇按论文脉络把“Agent 记忆”拆成可工程化的议题，并映射回本仓库的五层架构：**Data & Memory + Runtime + Governance**。

---

## 1) 先把问题说清楚：Agent 的记忆不是“存储”，而是“读写策略”

对 Agent 来说，记忆的本质是一个策略问题：

- **写入什么**（Write policy）：哪些事件/结论值得进入长期记忆？是否需要人工审批？
- **怎么组织**（Organize）：按主题、按任务、按人、按时间、按因果？
- **怎么检索**（Retrieve）：当前步骤需要什么信息？如何避免把无关信息塞进上下文？
- **怎么遗忘/更新**（Forget/Update）：错误信息怎么撤回？过期信息怎么失效？

只要你的系统允许“自动写入长期记忆”，**治理（Governance）就不是可选项**。

---

## 2) 三条代表性脉络（你可以用它们搭建内部路线图）

### A. “LLM 像操作系统”：把短期/长期记忆当成可分页的工作内存（MemGPT）

- **MemGPT: Towards LLMs as Operating Systems**
  - arXiv: https://arxiv.org/abs/2310.08560
  - 直觉：把上下文窗口当 RAM，把外部存储当 disk，用“调度/分页/摘要”把信息换入换出。

工程启发：

- 你需要明确 **Memory API（接口语义）**：append / summarize / pin / unpin / delete / redact
- 你需要一条 **可审计的记忆链**：谁写的、基于什么证据、什么时候写的、后续是否被引用

### B. “把记忆操作工具化”：让记忆读写变成 agent 可选择的 action（Agentic Memory / A-MEM）

这类论文共同点是：不把“记忆”当被动存储，而是把“记忆操作”显式放进 agent 的动作空间里。

- **Agentic Memory: Learning Unified Long-Term and Short-Term Memory Management for Large Language Model Agents**
  - arXiv: https://arxiv.org/abs/2601.01885
- **A-MEM: Agentic Memory for LLM Agents**
  - arXiv: https://arxiv.org/abs/2502.12110

工程启发：

- 当你把 memory 当 tool，你就能用本仓库的思路把它纳入 **Capability 层**：
  - 统一契约（Schema）
  - 统一调用记录（Observable）
  - 统一权限/隔离（Governance）

### C. “生成式社会模拟”的记忆机制：把长期记忆用于角色一致性与行为稳定（Generative Agents）

很多团队会从“Generative Agents”这一支脉络理解记忆：通过事件记录 + 反思摘要 + 检索，把角色行为变得连续。

（这一类思路对产品形态很有参考价值；但企业任务场景里，仍要用治理约束写入与引用。）

---

## 3) 映射到五层架构：你应该把哪些“记忆能力”产品化

把论文落到工程上，建议把“记忆”拆成下面这些可交付模块：

1. **Memory 数据模型（Data & Memory）**
   - Event / Fact / Preference / Policy / Outcome / Evidence
   - 每条记忆必须带：来源、时间、置信度、作用域（用户/租户/项目/任务）

2. **记忆操作能力（Capability）**
   - write_event / write_fact / write_summary
   - retrieve_by_task / retrieve_by_entity
   - redact / delete / expire

3. **记忆调度策略（Runtime）**
   - 什么时候 summarize？什么时候 pin 关键事实？
   - 什么时候禁止写入（例如：低置信度、无证据、用户明确禁止）？

4. **可观测与可审计（Observable）**
   - 每次检索命中哪些记忆？哪些被注入上下文？
   - 记忆写入是否导致错误决策？（可追溯）

5. **治理（Governance）**
   - 写入/删除权限
   - 脱敏与数据保留策略
   - 人工审批（high-risk memory write）

---

## 4) 一个“很工程”的结论：先做可治理的 Memory API，再谈更聪明的记忆策略

论文会让你相信：只要策略更强，Agent 就会“自己学会记忆”。

但企业里更稳的顺序是：

- 先把 **Memory API** 做成可控能力（Schema + 权限 + 审计 + 删除）
- 再把“何时写/写什么/怎么总结”做成可迭代策略（Runtime）

否则你得到的不是“更聪明的 Agent”，而是一个**不断自我污染的系统**。

---

## 参考阅读

- MemGPT（arXiv:2310.08560）：https://arxiv.org/abs/2310.08560
- Agentic Memory（arXiv:2601.01885）：https://arxiv.org/abs/2601.01885
- A-MEM（arXiv:2502.12110）：https://arxiv.org/abs/2502.12110
- 本仓库系列：series/README.md
- 长文全文：README.md
