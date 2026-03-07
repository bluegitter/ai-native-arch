# 25｜能力选择器（Resolver）怎么做：规则路由 + Embedding 检索 + 负反馈闭环（可落地实现）

当你的能力目录从 20 个涨到 200+，“让 LLM 自己选工具”会迅速变成不稳定的玄学：

- 选错域：把 CRM 的能力拿去做工单
- 选错版本：新旧接口混用
- 选到高风险能力：把查询当成写入
- 选到不可用能力：缺权限、缺依赖、流控被打

因此企业里必须把“选能力”工程化：把它从 prompt 里拿出来，变成一个**可治理、可观测、可迭代**的 Resolver。

这篇给你一套可直接落地的 Resolver 方案：

- 先用**规则**做硬约束（域/风险/权限/版本/黑白名单）
- 再用 **embedding 检索**做候选召回（semantic recall）
- 最后用**负反馈闭环**做持续变稳（从线上错误里学习）

并提供一份最小可用的 TypeScript 代码骨架。

---

## 1) Resolver 的职责边界：只做“候选召回与过滤”，不要做“最终决策”

一个实用分层是：

- **Resolver（平台能力）**：给定任务意图 + 上下文 → 召回 Top-K 候选能力（并过滤不合规）
- **Planner/Agent（模型）**：在 Top-K 里做最终选择、填参、执行

这样你能获得两个好处：

1. 模型不会在 200 个工具里瞎选（减少错选）
2. 你能在 Resolver 侧做治理与观测（可控、可回放）

---

## 2) 一个可落地的 Resolver Pipeline（推荐顺序）

> 目标：把“工具选择”变成一个可审计的流水线。

### Step A：规则过滤（Hard Filters）

先过滤掉“绝对不该出现”的能力：

- domain 不匹配（或不在允许域集合）
- policy 不满足（缺 scope / 风险超过 budget）
- 状态不可用（deprecated / maintenance / rate limited）
- 版本不满足（只允许 v>=x）

### Step B：候选召回（Recall）

召回策略建议**并联**，然后做 union：

- 关键词/标签召回（tags, operationId, domain）
- embedding 语义召回（query vs capability description）
- 任务模板召回（比如“退款”→ payments.refund.*）

### Step C：重排（Rerank）

对候选做一个简单但很有效的打分：

- semantic score（embedding 相似度）
- reliability score（历史成功率/错误率）
- cost score（调用成本/延迟）
- risk penalty（风险越高越扣分）

最后输出 Top-K 给模型。

---

## 3) 负反馈闭环：让 Resolver 从线上“变稳”，而不是永远靠 prompt

你需要把每次 tool-call 的结果写进一个“反馈事件”：

- 选了哪个能力（capabilityId）
- 任务意图（query / task type）
- 结果：success / INVALID_ARGUMENT / FORBIDDEN / NOT_FOUND / TRANSIENT…
- 失败原因：schema 错？权限错？依赖缺？

然后 Resolver 在重排阶段使用它：

- 某能力对某类 query 频繁 `FORBIDDEN` → 降权（或直接过滤）
- 某能力对某类 query 频繁 `INVALID_ARGUMENT` → 提示 schema 需要更强约束，或加入参数修复器
- 某能力 `TRANSIENT` 高但可重试 → 不必降太多（看重试后成功率）

这就是“企业版学习”：不是训练模型，而是训练你的调度策略。

---

## 4) 最小可用实现（TypeScript）：Resolver + Embedding + Feedback

下面这套代码是“可跑”的骨架：

- 不依赖向量库：先用内存向量（你可以后续换 Milvus/pgvector）
- 不依赖复杂在线学习：先用简单的 EWMA 成功率做降权

### 4.1 类型定义

```ts
// resolver/types.ts
export type Risk = 'low' | 'medium' | 'high'

export type Capability = {
  id: string
  name: string
  domain: string
  version: string
  description: string
  tags?: string[]
  policy: {
    risk: Risk
    requires: string[]
  }
}

export type CallerContext = {
  userId: string
  scopes: string[]
  riskBudget: Risk
  allowedDomains?: string[]
}

export type ResolveRequest = {
  query: string               // 任务意图（给 resolver 的短文本）
  domainHint?: string         // 可选：上层路由给的域提示
  tagsHint?: string[]         // 可选：业务侧标签
  topK?: number
}

export type ResolvedCandidate = {
  capability: Capability
  score: number
  reasons: string[]
}
```

### 4.2 规则过滤（policy + domain）

```ts
// resolver/policy.ts
import { Capability, CallerContext, Risk } from './types'

const order: Record<Risk, number> = { low: 0, medium: 1, high: 2 }

export function policyAllows(ctx: CallerContext, cap: Capability): { ok: boolean; reason?: string } {
  // scopes
  for (const req of cap.policy.requires) {
    if (!ctx.scopes.includes(req)) return { ok: false, reason: `missing scope: ${req}` }
  }
  // risk
  if (order[cap.policy.risk] > order[ctx.riskBudget]) {
    return { ok: false, reason: `risk too high: ${cap.policy.risk}` }
  }
  // domain allowlist
  if (ctx.allowedDomains?.length && !ctx.allowedDomains.includes(cap.domain)) {
    return { ok: false, reason: `domain not allowed: ${cap.domain}` }
  }
  return { ok: true }
}
```

### 4.3 Embedding 存储与相似度（最小版）

你可以把“embedding 生成”放到离线任务里：每次能力注册/更新时生成向量。

```ts
// resolver/embedding.ts
export type Vector = number[]

export function cosine(a: Vector, b: Vector): number {
  let dot = 0, na = 0, nb = 0
  for (let i = 0; i < Math.min(a.length, b.length); i++) {
    dot += a[i] * b[i]
    na += a[i] * a[i]
    nb += b[i] * b[i]
  }
  if (na === 0 || nb === 0) return 0
  return dot / (Math.sqrt(na) * Math.sqrt(nb))
}

export class InMemoryEmbeddingIndex {
  private map = new Map<string, Vector>()

  upsert(id: string, v: Vector) {
    this.map.set(id, v)
  }

  get(id: string) {
    return this.map.get(id)
  }
}
```

> 真实环境：把 `InMemoryEmbeddingIndex` 替换成 pgvector/Milvus/Qdrant 即可。

### 4.4 反馈模型（EWMA 成功率）

```ts
// resolver/feedback.ts
export type FeedbackEvent = {
  capabilityId: string
  ok: boolean
  errorCode?: string // INVALID_ARGUMENT/FORBIDDEN/TRANSIENT...
}

export class FeedbackStore {
  // EWMA success rate: s = a*ok + (1-a)*s
  private alpha = 0.15
  private success = new Map<string, number>()
  private forbidden = new Map<string, number>()

  record(ev: FeedbackEvent) {
    const prev = this.success.get(ev.capabilityId) ?? 0.8
    const next = this.alpha * (ev.ok ? 1 : 0) + (1 - this.alpha) * prev
    this.success.set(ev.capabilityId, next)

    if (ev.errorCode === 'FORBIDDEN') {
      const pf = this.forbidden.get(ev.capabilityId) ?? 0
      this.forbidden.set(ev.capabilityId, pf + 1)
    }
  }

  successRate(id: string) {
    return this.success.get(id) ?? 0.8
  }

  forbiddenCount(id: string) {
    return this.forbidden.get(id) ?? 0
  }
}
```

### 4.5 Resolver：并联召回 + 过滤 + 重排

```ts
// resolver/resolver.ts
import { Capability, CallerContext, ResolveRequest, ResolvedCandidate } from './types'
import { policyAllows } from './policy'
import { InMemoryEmbeddingIndex, cosine, Vector } from './embedding'
import { FeedbackStore } from './feedback'

export class CapabilityResolver {
  constructor(
    private caps: Capability[],
    private embedIndex: InMemoryEmbeddingIndex,
    private feedback: FeedbackStore
  ) {}

  resolve(ctx: CallerContext, req: ResolveRequest, queryVec: Vector): ResolvedCandidate[] {
    const topK = req.topK ?? 8

    // A) hard filter
    const filtered: Capability[] = []
    for (const cap of this.caps) {
      if (req.domainHint && cap.domain !== req.domainHint) continue
      const p = policyAllows(ctx, cap)
      if (!p.ok) continue
      filtered.push(cap)
    }

    // B) recall (tags + embedding)
    const tagHits = new Set<string>()
    const tags = new Set([...(req.tagsHint ?? []), ...req.query.toLowerCase().split(/\W+/)].filter(Boolean))
    for (const cap of filtered) {
      if ((cap.tags ?? []).some(t => tags.has(t.toLowerCase()))) tagHits.add(cap.id)
    }

    // C) scoring
    const scored: ResolvedCandidate[] = []
    for (const cap of filtered) {
      const v = this.embedIndex.get(cap.id)
      const sim = v ? cosine(queryVec, v) : 0
      const reliability = this.feedback.successRate(cap.id) // 0..1
      const forbiddenPenalty = Math.min(0.4, this.feedback.forbiddenCount(cap.id) * 0.05)
      const tagBoost = tagHits.has(cap.id) ? 0.08 : 0
      const score = 0.65 * sim + 0.35 * reliability + tagBoost - forbiddenPenalty

      const reasons = [
        `sim=${sim.toFixed(3)}`,
        `reliability=${reliability.toFixed(2)}`,
        tagBoost ? 'tag_hit' : '',
        forbiddenPenalty ? `forbidden_penalty=${forbiddenPenalty.toFixed(2)}` : ''
      ].filter(Boolean)

      scored.push({ capability: cap, score, reasons })
    }

    scored.sort((a, b) => b.score - a.score)
    return scored.slice(0, topK)
  }
}
```

你现在已经能做到：

- 规则硬过滤（合规/权限/风险）
- embedding 语义召回（候选更相关）
- 反馈降权（越跑越稳）

---

## 5) 一个“真实可落地”的使用方式（端到端示例）

### 5.1 能力定义（最简）

```ts
const caps = [
  {
    id: 'crm.customer.get',
    name: 'Get Customer',
    domain: 'crm',
    version: '1.2.0',
    description: 'Fetch customer profile by id.',
    tags: ['customer', 'read', 'crm'],
    policy: { risk: 'low', requires: ['scope:crm:read'] }
  },
  {
    id: 'tickets.create',
    name: 'Create Ticket',
    domain: 'support',
    version: '1.0.0',
    description: 'Create a customer support ticket.',
    tags: ['ticket', 'write', 'support'],
    policy: { risk: 'medium', requires: ['scope:support:write'] }
  }
] as const
```

### 5.2 Resolver 输出 Top-K 给模型

输出建议格式（给 Planner/Agent 的 tool list）：

```json
{
  "candidates": [
    {"id": "tickets.create", "why": ["sim=0.71", "reliability=0.86", "tag_hit"]},
    {"id": "crm.customer.get", "why": ["sim=0.55", "reliability=0.92"]}
  ]
}
```

然后你让模型在候选里选一个，进行 schema 校验与执行（见第 24 篇的 Executor）。

---

## 6) 线上落地 Checklist（非常重要）

要让 Resolver 真正生产可用，建议至少做到：

- **可回放**：每次 resolve 记录 req + ctx（脱敏）+ candidate list + scores
- **可降级**：embedding 服务挂了还能走 tags/规则
- **可控 topK**：不同风险任务给不同 topK（越高风险越小）
- **灰度与 A/B**：新策略先灰度，不要“一把梭”

---

## 参考阅读

- 系列 24：能力网络与 Executor（API-first / Schema-first）：./24-building-capability-network-api-schema-first.md
- 系列目录：series/README.md
- 长文全文：README.md
