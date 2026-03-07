# 17｜怎么评估 LLM Agent：从“跑通 demo”到“可上线”的基准与指标体系

Agent 最容易陷入一个假象：

- demo 看起来能做事
- 线上一放量就到处翻车

原因是你缺的不是“更聪明”，而是**评估体系**：什么算成功？失败能不能被发现？成本是否可控？风险是否可审计？

这篇把近年的 Agent benchmark 脉络整理成工程视角，并给出一套落地指标建议。

---

## 1) 评估 Agent 的难点：它不是一个静态 QA

传统 LLM 评估（如 MMLU/阅读理解）主要测“答案质量”。

但 Agent 的关键能力是：

- 选择工具
- 正确填参
- 在交互环境里多步推进
- 面对失败时恢复
- 控制成本与风险

所以 Agent 评估更像“系统测试 + 运行指标”，而不是纯 NLP 指标。

---

## 2) 代表性基准：用它们校准你在测什么

### A. AgentBench：把“LLM as Agent”做成多维基准

- **AgentBench: Evaluating LLMs as Agents**
  - arXiv: https://arxiv.org/abs/2308.03688

你可以把它当作一个提醒：

- 只测最终结果不够，还要测交互过程与工具调用
- agent 的稳定性往往比峰值能力更重要

### B. Web/软件工程/真实环境：评估“在现实系统里做事”的能力

企业最关心的是：Agent 在真实系统里能不能稳。

研究社区常见评估环境包括：

- Web 交互环境（如 WebArena 这一类）
- 软件工程任务（如 SWE-bench 这一类）
- 受控 OS/应用环境（如 OSWorld/AndroidWorld 这一类）

（具体 benchmark 名称在快速演进，但方向是稳定的：从静态问答走向真实交互。）

### C. Survey：帮你建立“评估地图”，避免东测一枪西测一炮

- **Evaluation and Benchmarking of LLM Agents: A Survey**
  - arXiv: https://arxiv.org/html/2507.21504v1

---

## 3) 映射到本仓库：评估 = Observable 的落地抓手

在本仓库的四原则里，**Observable** 是最容易被忽略、但最决定能不能上线的部分。

把“评估”做成工程能力，建议分三层：

### (1) 任务级指标（Task-level）

- success_rate：任务成功率（按任务定义）
- time_to_success：完成时间分布（P50/P95）
- cost_to_success：调用次数/token/外部 API 成本
- human_takeover_rate：人工接管比例

### (2) 步骤级指标（Step-level）

- tool_call_valid_rate：工具调用参数是否通过 schema 校验
- tool_error_rate：工具错误（4xx/5xx/业务错误）
- retry_rate：重试次数与原因分布
- recovery_rate：失败后是否能正确恢复（replan/换工具/降级）

### (3) 风险与治理指标（Governance-level）

- policy_violation_rate：越权/越域/敏感操作触发次数
- audit_coverage：关键动作是否都有审计记录
- data_retention_compliance：记忆/日志的保留与删除是否合规

---

## 4) 实用建议：先做“离线回放 + 线上守护”，再追求 SOTA 分数

很多团队一上来就追 benchmark 分数，反而把工程节奏打乱。

更稳的路线：

1. **离线回放（offline replay）**
   - 用真实线上任务日志构造可回放用例
   - 评估版本迭代是否真的提升成功率/成本

2. **线上守护（online guardrails）**
   - schema 校验、权限拦截、预算限制、人工接管
   - 让系统“失败得体面”，而不是“静悄悄地做错事”

---

## 参考阅读

- AgentBench（arXiv:2308.03688）：https://arxiv.org/abs/2308.03688
- LLM Agent 评估综述（arXiv:2507.21504）：https://arxiv.org/html/2507.21504v1
- 本仓库系列：series/README.md
- 长文全文：README.md
