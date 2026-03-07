# 04｜五层参考架构：从“应用分层”到“能力分层”

> 适合发布：对外分享 / 内部评审（带图更好）

很多企业讨论 Agent-Native 时容易卡在“给现有系统加一个对话入口”。但真正的转折点是：企业 IT 的组织单元从“应用系统”迁移到“可编排能力网络”。

因此，参考架构的价值不在于“多一张分层图”，而在于回答一条端到端执行链路：

- Agent 怎么连上系统？
- 能力怎么被组织成可规划对象？
- 上下文从哪里来、是否可信？
- 任务如何稳定交付而不是一次性生成？
- 风险如何贯穿执行路径被控制？

---

## 五层参考架构（面向企业落地）

1. **Interface Layer**：统一接入面（Gateway / Discovery / Schema / Auth）
2. **Capability Layer**：能力抽象与编排（Registry / Graph / Policy Binding）
3. **Data & Knowledge + Memory Layer**：上下文底座（事实、规则、经验）
4. **Runtime & Orchestration Layer**：任务生命周期（执行、补偿、接管）
5. **Security & Governance Layer**：身份、策略、审计、合规贯穿

---

## 最常见的失败点（按层）

- Interface 不统一：Agent 找不到能力/参数构造错/权限握手复杂
- Capability 缺失：接口多但无法组合，端到端靠人工串
- Data & Memory 不可靠：上下文过期/冲突，导致“看起来合理但不可执行”
- Runtime 不成熟：任务中断后不可恢复，异常变成黑盒
- 治理缺位：不能解释/不能追责/不敢放权

---

## 延伸阅读

- 完整版长文（含每层展开、治理与路线图）：仓库主页 README.md
- 一页总览（可复制）：README.md#one-page
