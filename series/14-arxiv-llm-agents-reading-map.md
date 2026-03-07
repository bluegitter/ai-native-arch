# 14｜LLM Agent 研究速览：把 arXiv 论文脉络映射到企业 Agent-Native 架构（读论文不迷路）

如果你在企业里做 Agent-Native，读论文最容易出现两种错位：

- 论文讲“模型如何思考/规划”，你落地卡在“接口语义/权限/审计/状态机”
- 你把某个方法（比如 ReAct/Toolformer）当成“完整智能体”，结果只补了推理，没补执行与治理

这篇文章的目标是：用一张“阅读地图”把 LLM Agent 的研究脉络整理成**可落地的工程问题**，并映射到本仓库的四原则/五层架构。

---

## 1) 先给一个统一框架：LLM Agent = 规划（Plan）+ 行动（Act）+ 记忆（Memory）+ 反思（Reflect）+ 运行时（Runtime）

学术论文经常从某个子能力切入（比如“推理+行动”），但企业落地需要把它们拼回一条闭环。

你可以用下面这句“工程版定义”统一沟通：

> 一个可上线的 Agent，不是一个会推理的模型，而是一个能在约束下持续执行任务的系统：**能发现能力、能按契约调用、能在失败时恢复、能被审计与治理**。

---

## 2) 论文阅读地图（按你关心的问题来分组）

下面列的是最常被引用、且对工程落地有启发的论文方向。每一组我都写了“它解决什么”以及“落地时你还缺什么”。

### A. Reasoning + Acting：让模型在推理与工具调用之间来回切换

- **ReAct: Synergizing Reasoning and Acting in Language Models**
  - arXiv: https://arxiv.org/abs/2210.03629
  - 解决：把“想”和“做”交织在一起，减少一次性长计划的脆弱性
  - 企业落地还缺：接口语义（Schema）、权限、幂等/重试、审计链

### B. Tool Use Learning：让模型学会更会用工具（或更会选工具）

- **Toolformer: Language Models Can Teach Themselves to Use Tools**
  - arXiv: https://arxiv.org/abs/2302.04761
  - 解决：工具调用能力的学习与触发
  - 企业落地还缺：工具“能不能稳定用”（输入输出契约、错误语义、版本治理），不是“会不会用”

### C. Survey / Taxonomy：把领域全景整理成统一框架（适合做路线图/评审）

- **A Survey on Large Language Model based Autonomous Agents**
  - arXiv: https://arxiv.org/abs/2308.11432
  - 贡献：给出统一框架、应用谱系、评估方法、挑战与方向
  - 建议用法：拿它的分类做你内部“能力清单”，再用本仓库五层架构做“工程补全”

### D. Reflection / Self-improvement：让 Agent 从失败中修正策略（更像“可运营系统”）

这一类工作常见思路是：执行后把结果与过程写回记忆，再用反思提示让下一次更稳。

- 工程映射：这类论文通常对应你系统里的 **Observable + Memory 治理 + 运行时回路**。
- 落地提醒：企业场景里“反思写入”必须受治理控制（避免把错误经验固化）。

### E. Planning / Decomposition：把大目标拆解成可执行子任务

- 解决：任务分解与计划生成
- 工程映射：对应 **Runtime 层的 Planner**
- 落地提醒：计划再好也会撞上现实，关键还是失败恢复与治理（补偿/接管/审计）。

---

## 3) 把论文落到你的四原则/五层架构：一张对照表

| 论文常解决 | 对应企业问题 | 在本仓库里属于 | 常见落地缺口 |
|---|---|---|---|
| 推理与行动（ReAct） | “怎么边想边做” | Runtime（执行循环） | 工具契约、幂等、权限、审计 |
| 工具学习（Toolformer） | “什么时候该用什么工具” | Capability/Tool 选择策略 | 工具稳定性与治理（不是触发能力） |
| 记忆/反思 | “怎么越用越稳” | Data & Memory + Observable | 记忆治理、纠偏、合规清理 |
| 任务规划 | “怎么拆任务” | Runtime（Planner） | 异常恢复、转人工机制 |
| 评估 | “怎么量化好不好” | Observable（指标） | 任务级 SLO：成功率/成本/风险 |

---

## 4) 一个实用建议：用论文补“方法”，用架构补“系统”

你可以把学术工作当作“方法库”，但不要把它当作“生产系统蓝图”。

更稳的落地顺序仍然是：

- **API-first**：先让关键动作可调用
- **Schema-first**：让语义可校验
- **Capability-based**：让能力可组合
- **Observable**：让执行可运营

有了这四个底座，ReAct/Toolformer/反思类方法才真正能在生产环境持续发挥价值。

---

## 参考阅读

- 本仓库长文：README.md
- 一页总览：README.md#one-page
- 系列目录：series/README.md
