# 27｜把编排跑到生产：检修工单“派发→执行→验收” 的状态持久化、断点恢复（Resume）与 OpenTelemetry 观测（含代码骨架）

在第 26 篇我们有了 DAG 执行器 + Saga 补偿的“可用骨架”。

但企业上线的分水岭不在“能不能跑”，而在：

- **跑到一半能不能恢复**（断电/超时/限流/人工审批卡住）
- **失败能不能定位与复盘**（哪一步、哪个工具、哪个参数、谁授权的）
- **是否具备合规审计链**（谁触发、谁审批、对哪些资产做了什么动作）

因此生产化 Runtime 的核心是三件事：

1. **状态持久化（Persistence）**：Run/StepAttempt/Artifacts 全量可回放
2. **断点恢复（Resume）**：从“中断处”继续，而不是从头重跑
3. **可观测（Observability）**：Trace + Logs + Metrics 三件套

这篇用一个最典型的企业流程做示例：**检修工单派发执行**。

---

## 0) 示例场景：检修工单派发执行（可对接你公司的 CMDB / 工单 / 权限）

目标：收到一条检修请求后，系统自动完成：

- 工单创建（ITSM）
- CMDB 拉取设备信息与位置
- 派发给合适的工程师（排班/技能/区域）
- 通知工程师（IM/短信）
- 工程师回填执行结果（照片/测量值/更换件）
- 验收与关闭（必要时触发复检或升级）

注意：这里天然包含**人工环节**，所以 Resume 与审计更重要。

---

## 1) 生产化第一步：把“任务”定义成可持久化的 Run 与 StepAttempt

### 1.1 Run（一次编排实例）

建议至少包含：

- `runId`：全局唯一（可当幂等根）
- `workflowId` / `workflowVersion`
- `status`：running / waiting_human / failed / succeeded / compensating / compensated
- `input`：请求输入（脱敏或引用）
- `vars`：跨 step 的结构化上下文（artifact references）
- `createdAt/updatedAt`

### 1.2 StepAttempt（一步的某次尝试）

- `stepId`
- `attempt`
- `status`：running / succeeded / failed / skipped
- `startedAt/endedAt`
- `inputRef` / `outputRef`：输入输出存 artifact store（不要直接塞 DB 大字段）
- `error`：code/message/retryable

> 原则：**所有决定都可回放**。你需要能回答“当时的输入是什么、模型/规则选了什么能力、返回是什么”。

---

## 2) 工单流程如何建成 DAG：派发→执行→验收（含人工卡点）

一个可落地的 DAG（简化版）：

1. `createTicket`（写，幂等）
2. `fetchAssetFromCMDB`（读）
3. `selectAssignee`（读/计算）
4. `dispatchWorkOrder`（写，幂等）
5. `notifyAssignee`（写，通常可补偿为“发更正/撤销通知”）
6. `waitForEngineerReport`（等待人工回填：waiting_human）
7. `validateReport`（校验附件/测量值）
8. `closeTicket`（写，幂等）

这类流程的关键不是并行，而是：

- **等待点必须可持久化**（waiting_human）
- **回填必须结构化**（Schema-first）

---

## 3) “等待人工”如何做成可 Resume：事件驱动 + 幂等回调

企业里最常见的中断点就是等待人工。

建议模式：

- Runtime 执行到 `waitForEngineerReport` 时：
  - 将 Run 状态置为 `waiting_human`
  - 生成一个 `humanTaskId`（或回调 token）
  - 把它写入工单系统/表单链接

- 工程师提交回填时：
  - 通过 webhook / 回调 API 告诉 Runtime：`POST /runtime/runs/{runId}/events`
  - Runtime 验证 token + 幂等（eventId）
  - 写入 artifact store，并把 run 继续向下推进

这就是 Resume 的本质：**不是“继续跑进程”，而是“继续推进状态机”。**

---

## 4) OpenTelemetry（OTel）怎么打：Run 是一条 Trace，Step 是 Span

你想要的线上观测结构建议如下：

- `trace`：一个 `runId` 对应一条 trace
- `span`：
  - `workflow.run`（root span）
  - `step.<stepId>`（每步一个 span）
  - `tool.<capabilityId>`（工具调用作为子 span）

建议打的 attributes：

- `run.id` / `workflow.id` / `workflow.version`
- `step.id` / `step.attempt`
- `capability.id` / `capability.domain` / `capability.version`
- `error.code` / `error.retryable`
- `policy.risk` / `policy.scopes_missing`

> 这样你在 Jaeger/Grafana Tempo 里一眼能看到：卡在哪一步、哪次 attempt、哪个工具、为何失败。

---

## 5) 代码骨架（TypeScript）：持久化存储 + Resume + OTel

下面给一个“能落地”的最小骨架：

- 存储接口（你可落到 Postgres/Redis）
- Artifact store（S3/OSS/MinIO）
- OTel tracing（Node SDK）
- wait step 用事件恢复

### 5.1 Run/Attempt 存储接口

```ts
// runtime/store.ts
export type RunStatus =
  | 'running'
  | 'waiting_human'
  | 'failed'
  | 'succeeded'
  | 'compensating'
  | 'compensated'

export type RunRecord = {
  runId: string
  workflowId: string
  workflowVersion: string
  status: RunStatus
  vars: Record<string, any>
  createdAt: number
  updatedAt: number
}

export type AttemptRecord = {
  runId: string
  stepId: string
  attempt: number
  status: 'running' | 'succeeded' | 'failed' | 'skipped'
  inputRef?: string
  outputRef?: string
  error?: { code: string; message: string; retryable?: boolean }
  startedAt: number
  endedAt?: number
}

export interface RunStore {
  getRun(runId: string): Promise<RunRecord | null>
  upsertRun(run: RunRecord): Promise<void>

  listSucceededSteps(runId: string): Promise<string[]>

  getLatestAttempt(runId: string, stepId: string): Promise<AttemptRecord | null>
  insertAttempt(a: AttemptRecord): Promise<void>
  finishAttempt(a: AttemptRecord): Promise<void>
}
```

### 5.2 Artifact store（输出大字段/附件引用）

```ts
// runtime/artifacts.ts
export interface ArtifactStore {
  putJSON(path: string, value: any): Promise<string> // returns ref
  getJSON(ref: string): Promise<any>
}
```

### 5.3 OTel tracing 初始化（极简）

```ts
// runtime/otel.ts
import { NodeSDK } from '@opentelemetry/sdk-node'
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'

export function startOtel(serviceName: string) {
  const sdk = new NodeSDK({
    serviceName,
    traceExporter: new OTLPTraceExporter({
      // OTEL_EXPORTER_OTLP_ENDPOINT env 或这里配置
    }),
    instrumentations: [getNodeAutoInstrumentations()]
  })
  sdk.start()
  return sdk
}
```

### 5.4 Workflow step：waitForEngineerReport（等待 + Resume）

```ts
// workflows/maintenance.ts
import { Step, StepContext, StepResult } from '../runtime/step'

export const waitForEngineerReport: Step = {
  id: 'waitForEngineerReport',
  dependsOn: ['notifyAssignee'],
  run: async (ctx: StepContext): Promise<StepResult> => {
    // 关键：不要阻塞等待，而是把 run 置为 waiting_human
    ctx.audit({ type: 'human_wait', runId: ctx.runId, formUrl: ctx.vars.formUrl })
    return {
      ok: false,
      error: { code: 'WAITING_HUMAN', message: 'waiting for engineer report', retryable: false }
    }
  }
}
```

生产实现里，你会在 Runtime 层把 `WAITING_HUMAN` 解释为：

- 将 run.status 置为 `waiting_human`
- 返回给上层一个 humanTaskId

当 webhook 事件到达时，写入 `vars.engineerReportRef` 并 resume。

---

## 6) Resume 策略：哪些 step 能跳过？哪些必须重放？

建议规则：

- **只读步骤**：如果上次成功且输入未变化，可以跳过（节省成本）
- **幂等写步骤**：可以重放，但必须带 idempotency key（runId+stepId）
- **非幂等写步骤**：避免重放；必须在执行前做“状态检查”（read-before-write）

落地要点：

- 所有 step 的 input/output 都有 artifact ref
- run.vars 里只放引用/摘要，不放大字段

---

## 7) “真实可落地”的日志字段规范（便于 ELK/ClickHouse）

建议你的 audit log（structured log）至少包含：

- `ts`
- `runId`
- `workflowId` / `workflowVersion`
- `stepId` / `attempt`
- `eventType`：step_start/step_end/tool_call/human_wait/human_resume
- `capabilityId`（如果有）
- `ok` / `error.code`
- `latencyMs`
- `cost`（token/API calls）
- `operator`（human approver / engineer）

---

## 8) 把它接回你前面的能力体系（24-26）：闭环就齐了

- 24：能力目录与 schema 门禁
- 25：Resolver 输出 Top-K 候选能力
- 26：DAG 执行 + Saga 补偿
- 27（本篇）：状态持久化 + Resume + OTel 观测

到这里，你的系统已经具备“生产三件套”：

- 可恢复（resume）
- 可观测（trace/log/metric）
- 可治理（policy/audit/human-in-loop）

---

## 延伸阅读

- 24：能力网络（API-first / Schema-first）：./24-building-capability-network-api-schema-first.md
- 25：Resolver（规则+embedding+反馈）：./25-capability-resolver-routing-rule-embedding-feedback.md
- 26：DAG/Saga（编排与补偿）：./26-dag-orchestration-and-saga-compensation.md
- 系列目录：series/README.md
- 长文全文：README.md
