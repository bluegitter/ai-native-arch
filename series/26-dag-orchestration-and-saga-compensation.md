# 26｜能力网络如何编排：DAG 执行器 + 补偿事务（Saga）让长链任务“可回滚、可恢复”（含示例代码）

当你把能力目录、Resolver、Schema 校验都做起来之后，企业落地的下一个硬骨头是：

- 一条任务链调用 5~30 个能力，中间任何一步失败都很常见
- 有些失败可以重试，有些失败必须回滚/补偿
- 有些任务必须并行（提速），但并行会带来依赖与一致性问题

这就是为什么“能力网络”最终一定会走到 **DAG 编排（Orchestration）** 与 **Saga 补偿（Compensation）**。

这篇给你一套可直接落地的最小实现：

- 用 **DAG** 表达步骤依赖（并行、汇聚、条件分支）
- 用 **Saga** 为每个写操作提供补偿动作（compensate）
- 用 **运行时状态机** 实现：重试、回放、断点恢复、人工接管

并给出 TypeScript 代码骨架（可迁移进你的 Runtime 服务）。

---

## 1) 先统一术语：什么是 DAG 编排？什么是 Saga？

### DAG（Directed Acyclic Graph）

- 用节点表示步骤（step）
- 用边表示依赖（A 完成后才能做 B）
- 无环（避免死循环），适合表达：并行 → 汇聚 → 继续

### Saga（补偿事务）

在分布式系统里，很难做强一致的两阶段提交。

Saga 的工程翻译是：

- 每个“写操作”都要配一个“补偿操作”
- 当后续失败时，按逆序执行补偿，把系统拉回可接受状态

典型例子：

- createOrder（补偿：cancelOrder）
- reserveInventory（补偿：releaseInventory）
- sendCoupon（补偿：revokeCoupon 或发一条纠错工单）

> 注意：补偿不一定能做到完全恢复，但必须做到“业务上可接受”（例如：把风险升级到人工处理）。

---

## 2) 企业里最常见的 3 类步骤：决定你怎么做重试/补偿

建议给每个能力打标签：

1. **Read-only（只读）**：可重试，通常不需要补偿
2. **Write-idempotent（幂等写）**：必须带 idempotency key；失败可重试
3. **Write-non-idempotent（非幂等写）**：必须提供补偿；尽量改造为幂等

这会直接决定 Runtime 的默认策略。

---

## 3) 设计一个可落地的 Step Model（包含补偿与重试策略）

```ts
// runtime/step.ts
export type StepId = string

export type StepResult<T = any> =
  | { ok: true; output: T }
  | { ok: false; error: { code: string; message: string; retryable?: boolean } }

export type StepContext = {
  runId: string
  now: () => number
  vars: Record<string, any> // 上下文变量（来自上游输出）
  audit: (ev: any) => void
}

export type Step = {
  id: StepId
  dependsOn?: StepId[]
  // 执行
  run: (ctx: StepContext) => Promise<StepResult>
  // 可选补偿：仅对“已成功的写操作”执行
  compensate?: (ctx: StepContext) => Promise<StepResult>
  // 重试策略
  retry?: {
    maxAttempts: number
    backoffMs: (attempt: number) => number
  }
}
```

这个模型的关键点是：

- `dependsOn` 让你能做 DAG
- `compensate` 让你能做 Saga
- `vars` 让步骤之间用“结构化工件”交接，而不是靠对话文本

---

## 4) 最小 DAG 执行器：拓扑排序 + 就绪队列 + 并发限制

下面的实现支持：

- 依赖满足才能执行
- 并发执行（提升吞吐/降低延迟）
- 记录执行顺序，后续做补偿

```ts
// runtime/dag.ts
import { Step, StepContext, StepResult } from './step'

type RunState = {
  status: 'running' | 'failed' | 'succeeded' | 'compensated'
  outputs: Record<string, any>
  succeededSteps: string[]
  failedStep?: string
}

export async function runDag(
  steps: Step[],
  ctx: StepContext,
  options?: { concurrency?: number }
): Promise<RunState> {
  const concurrency = options?.concurrency ?? 4

  const byId = new Map(steps.map(s => [s.id, s]))
  const deps = new Map<string, Set<string>>()
  const rev = new Map<string, Set<string>>()

  for (const s of steps) {
    deps.set(s.id, new Set(s.dependsOn ?? []))
    for (const d of s.dependsOn ?? []) {
      if (!rev.has(d)) rev.set(d, new Set())
      rev.get(d)!.add(s.id)
    }
  }

  const ready: string[] = []
  for (const [id, ds] of deps) if (ds.size === 0) ready.push(id)

  const state: RunState = { status: 'running', outputs: {}, succeededSteps: [] }
  const inFlight = new Set<string>()

  async function runOne(id: string): Promise<{ id: string; res: StepResult }> {
    const step = byId.get(id)!

    // retry loop
    const max = step.retry?.maxAttempts ?? 1
    for (let attempt = 1; attempt <= max; attempt++) {
      ctx.audit({ type: 'step_start', stepId: id, attempt })
      const res = await step.run({ ...ctx, vars: { ...ctx.vars, ...state.outputs } })
      ctx.audit({ type: 'step_end', stepId: id, attempt, ok: res.ok, error: !res.ok ? res.error : undefined })

      if (res.ok) return { id, res }

      const retryable = res.error.retryable ?? false
      if (!retryable || attempt === max) return { id, res }

      const backoff = step.retry?.backoffMs?.(attempt) ?? 200 * attempt
      await new Promise(r => setTimeout(r, backoff))
    }

    return { id, res: { ok: false, error: { code: 'UNKNOWN', message: 'retry loop escaped' } } }
  }

  // simple worker pool
  const running: Promise<{ id: string; res: StepResult }>[] = []

  function schedule() {
    while (ready.length > 0 && inFlight.size < concurrency) {
      const id = ready.shift()!
      inFlight.add(id)
      running.push(
        runOne(id).finally(() => {
          inFlight.delete(id)
        })
      )
    }
  }

  schedule()

  while (running.length > 0) {
    const done = await Promise.race(running)
    // remove one finished promise
    const idx = running.findIndex(p => (p as any) === (done as any))
    if (idx >= 0) running.splice(idx, 1)

    if (!done.res.ok) {
      state.status = 'failed'
      state.failedStep = done.id
      return state
    }

    state.outputs[done.id] = done.res.output
    state.succeededSteps.push(done.id)

    for (const nxt of rev.get(done.id) ?? []) {
      deps.get(nxt)!.delete(done.id)
      if (deps.get(nxt)!.size === 0) ready.push(nxt)
    }

    schedule()
  }

  state.status = 'succeeded'
  return state
}
```

说明：上面代码为了简洁，省略了“严格的 promise 管理”。在生产里建议用更成熟的并发队列实现（但核心思想就是：就绪队列 + 并发池）。

---

## 5) Saga 补偿：失败后按逆序补偿已成功写步骤

补偿只对“已经成功且定义了 compensate 的步骤”执行。

```ts
// runtime/saga.ts
import { Step, StepContext } from './step'

export async function compensateSaga(
  steps: Step[],
  succeededSteps: string[],
  ctx: StepContext
) {
  const byId = new Map(steps.map(s => [s.id, s]))

  for (let i = succeededSteps.length - 1; i >= 0; i--) {
    const id = succeededSteps[i]
    const step = byId.get(id)
    if (!step?.compensate) continue

    ctx.audit({ type: 'compensate_start', stepId: id })
    const res = await step.compensate({ ...ctx, vars: { ...ctx.vars } })
    ctx.audit({ type: 'compensate_end', stepId: id, ok: res.ok, error: !res.ok ? res.error : undefined })
  }
}
```

工程建议：

- **补偿也要幂等**（compensate 可能重试/回放）
- 补偿失败不要悄悄吞掉：要么重试、要么升级人工（创建工单/报警）

---

## 6) 一个端到端例子：下单链路（库存 → 订单 → 优惠券 → 通知）

### 6.1 能力（简化）

- reserveInventory（补偿：releaseInventory）
- createOrder（补偿：cancelOrder）
- applyCoupon（补偿：revokeCoupon）
- sendNotification（通常不补偿，或补偿为“发送更正通知”）

### 6.2 DAG 定义（并行 + 汇聚）

```ts
import { Step } from './runtime/step'

const steps: Step[] = [
  {
    id: 'reserveInventory',
    run: async (ctx) => {
      // call capability inventory.reserve with idempotencyKey=ctx.runId
      return { ok: true, output: { reservationId: 'r_1' } }
    },
    compensate: async (ctx) => {
      // inventory.release(reservationId)
      return { ok: true, output: {} }
    },
    retry: { maxAttempts: 3, backoffMs: a => 200 * a }
  },
  {
    id: 'createOrder',
    dependsOn: ['reserveInventory'],
    run: async (ctx) => {
      // orders.create(reservationId)
      return { ok: true, output: { orderId: 'o_1' } }
    },
    compensate: async (ctx) => {
      // orders.cancel(orderId)
      return { ok: true, output: {} }
    }
  },
  {
    id: 'applyCoupon',
    dependsOn: ['createOrder'],
    run: async (ctx) => {
      // coupons.apply(orderId)
      return { ok: true, output: { couponApplied: true } }
    },
    compensate: async (ctx) => {
      // coupons.revoke(orderId)
      return { ok: true, output: {} }
    }
  },
  {
    id: 'sendNotification',
    dependsOn: ['applyCoupon'],
    run: async (ctx) => {
      // notify.send(orderId)
      return { ok: true, output: { notified: true } }
    }
  }
]
```

### 6.3 运行：失败则补偿

```ts
import { runDag } from './runtime/dag'
import { compensateSaga } from './runtime/saga'

const ctx = {
  runId: 'run_20260308_0001',
  now: () => Date.now(),
  vars: { customerId: 'c_123', sku: 'sku_9', qty: 1 },
  audit: (ev: any) => console.log(JSON.stringify(ev))
}

const state = await runDag(steps, ctx, { concurrency: 3 })
if (state.status === 'failed') {
  await compensateSaga(steps, state.succeededSteps, ctx)
}
```

---

## 7) 企业落地的关键“坑位”与建议（非常实战）

### 7.1 不要把补偿当成“撤销”，把它当成“纠错路径”

很多现实系统里：

- 发出去的短信/邮件无法撤回
- 下游系统不支持 cancel

此时补偿策略应该是：

- 写一条纠错工单（human-in-the-loop）
- 发一条更正通知
- 在账务上生成冲正记录

**补偿是业务策略，不是纯技术。**

### 7.2 把“写操作”统一加上 idempotency key

否则你会得到：

- 重试导致重复创建
- 回放导致二次扣费

强制约束：所有写能力必须支持 `Idempotency-Key = runId + stepId`。

### 7.3 断点恢复（resume）比“从头跑”更重要

生产里任务经常因为：限流、超时、人工审批卡住。

因此你的 Runtime 状态必须持久化：

- 每步开始/结束
- 输出摘要
- 已成功步骤列表

下次 resume 时：

- 已成功的读步骤可以跳过（可配置）
- 已成功的写步骤必须检查幂等/状态（避免重复）

### 7.4 给每个步骤一个“expected observation”

这是你判断偏航与是否需要 replan 的依据：

- 订单状态必须变为 CREATED
- 库存 reservation 必须存在

否则 Agent 会在错误状态下继续执行，越跑越错。

---

## 8) 推荐的下一步：把这套骨架接入你的 Capability Registry

你现在已经有：

- 能力模型（id/owner/policy/schema）
- Resolver（Top-K 候选）

下一步就是：

- DAG step 的 `run()` 里调用 capability executor（schema 校验 + policy gate + tracing）
- step 输出写入 `vars`，供下游步骤取用

这样你就完成了“从能力网络到可交付执行链”的最后一跃。

---

## 延伸阅读

- 24：能力网络（API-first / Schema-first）：./24-building-capability-network-api-schema-first.md
- 25：Resolver（规则+embedding+反馈）：./25-capability-resolver-routing-rule-embedding-feedback.md
- 系列目录：series/README.md
- 长文全文：README.md
