# 24｜企业落地指南：API-first / Schema-first 之上，能力网络（Capability Graph）怎么建？

你要把 Agent 从 PoC 推到生产，最容易“卡死”的往往不是模型，而是三件事：

1. **API-first**：关键动作到底能不能被稳定调用（权限/幂等/错误语义）
2. **Schema-first**：调用参数与结果能不能被校验（否则工具调用=玄学）
3. **能力网络**：当动作多了、系统多了，Agent 如何“发现/选择/组合/治理”这些能力

这篇从企业落地角度给一套**可直接照抄的工程方案**：数据结构、目录、示例、以及一份最小可用的“能力注册 + 选择 + 观测”代码骨架。

---

## 0) 先定一个工程目标：让 Agent 的“工具调用”像微服务治理一样可管理

你最终要的不是“LLM 会调用函数”，而是：

- 能力可发现（catalog）
- 能力可组合（composition）
- 能力可治理（policy）
- 执行可观测（traces/metrics）

把它翻译成一个落地指标：

> 任意一个工具调用失败，都能回答：**为什么失败、是否可重试、由谁负责、如何回滚、影响哪些任务**。

---

## 1) API-first：先把“动作”变成稳定 API（别急着做 Agent）

### 1.1 三条硬约束（企业里最好当成准入门槛）

**（A）幂等性（Idempotency）**

- 写操作必须支持 idempotency key
- 典型：创建订单、发券、转账、发通知

**（B）错误语义（Error Semantics）**

- 不能只靠 HTTP 500/200
- 必须区分：
  - `INVALID_ARGUMENT`（别重试）
  - `UNAUTHORIZED` / `FORBIDDEN`（需要升级权限）
  - `NOT_FOUND`（可能是上下文错了）
  - `TRANSIENT`（可重试，带 backoff）
  - `CONFLICT`（需要读最新状态再执行）

**（C）可审计（Auditability）**

- 谁在什么上下文触发了什么动作
- 动作请求/响应（或脱敏后的摘要）必须可追溯

> 这三条不满足，Agent 只会把“偶发故障”放大成“系统事故”。

### 1.2 一个最小 API 例子（带幂等与错误语义）

```http
POST /v1/tickets
Idempotency-Key: 7c8f...  

{
  "title": "客户无法登录",
  "severity": "S2",
  "customerId": "c_123",
  "evidence": [{"type":"log","url":"..."}]
}
```

返回：

```json
{
  "ticketId": "t_987",
  "status": "open"
}
```

错误返回（建议统一）：

```json
{
  "error": {
    "code": "INVALID_ARGUMENT",
    "message": "severity must be one of S1,S2,S3",
    "retryable": false
  }
}
```

---

## 2) Schema-first：把“工具”变成可校验契约（让工具调用从玄学变工程）

Schema-first 的价值是：

- 在 **调用前** 就能校验参数（减少 LLM 乱填参）
- 在 **调用后** 就能校验返回（避免下游被污染）
- 能把工具接入做成流水线：生成/校验/回归测试/版本化

### 2.1 推荐：OpenAPI + JSON Schema 双轨

- 对 HTTP API：用 **OpenAPI**
- 对内部函数/消息：用 **JSON Schema**

示例（简化版 OpenAPI 片段）：

```yaml
openapi: 3.0.3
info: { title: Ticket API, version: 1.0.0 }
paths:
  /v1/tickets:
    post:
      operationId: createTicket
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateTicketRequest'
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CreateTicketResponse'
components:
  schemas:
    CreateTicketRequest:
      type: object
      required: [title, severity, customerId]
      properties:
        title: { type: string, minLength: 1 }
        severity: { type: string, enum: [S1, S2, S3] }
        customerId: { type: string }
```

### 2.2 让 Agent 真的受益：把 schema 变成“工具调用门禁”

一个可落地的调用链：

1. LLM 产出 tool call（可能错）
2. 先用 schema 校验（必做）
3. 不通过：把校验错误反馈给 LLM 修正（而不是直接打 API）
4. 通过：执行调用
5. 对返回再校验，写入 trace

---

## 3) 能力网络（Capability Graph）：不是“工具清单”，而是“可组合的能力地图”

当工具数从 10 变成 200，你会发现：

- 仅靠关键词检索工具名，Agent 会选错
- 仅靠“描述文本”，很难治理版本、权限、依赖

能力网络要解决的问题是：

- **发现**：我有哪些能力？属于哪个域？
- **选择**：给定任务/上下文，选哪些能力？
- **组合**：能力之间怎么拼（依赖/前置条件/产物）
- **治理**：谁能用、何时用、怎么审计、怎么下线

### 3.1 最小能力模型（建议直接落成一个 JSON）

```json
{
  "id": "crm.customer.get",
  "name": "Get Customer",
  "domain": "crm",
  "version": "1.2.0",
  "description": "Fetch customer profile by id.",
  "inputSchema": {"$ref": "schemas/crm.customer.get.input.json"},
  "outputSchema": {"$ref": "schemas/crm.customer.get.output.json"},
  "owner": "crm-platform@company.com",
  "policy": {
    "risk": "low",
    "requires": ["scope:crm:read"],
    "pii": ["email", "phone"],
    "rateLimit": {"rps": 10}
  },
  "deps": ["auth.token.exchange"],
  "tags": ["read", "customer"]
}
```

关键点：**policy 与 deps** 是能力网络能“可治理/可组合”的核心。

### 3.2 能力图的边（edges）怎么建：两类边最有用

- **依赖边（dependency）**：调用 A 之前必须先调用 B（拿 token、拿上下文、查状态）
- **工件边（artifact/dataflow）**：A 的输出字段能喂给 C 的输入字段

你不需要一开始就做全自动“字段级”映射；企业里更稳的路径是：

1. 先做依赖边（手工/半自动）
2. 再做工件边（从日志与 schema 推导 + 人工确认）

---

## 4) 可落地的架构：Registry + Resolver + Policy Gate + Tracing

一个最小可用的企业方案长这样：

- **Capability Registry（目录服务）**：存能力元数据（schema/policy/owner/version）
- **Resolver（选择器）**：给定 task，挑候选能力（按 domain/tag/embedding/规则）
- **Policy Gate（治理门）**：权限、风险、预算、审批
- **Executor（执行器）**：带 schema 校验 + 重试策略 + 幂等
- **Tracing（观测）**：记录每一步 tool call 的输入摘要/输出摘要/错误码/耗时/成本

> 这套东西做出来，你就从“玩具 agent”进化成“可运营系统”。

---

## 5) 一份最小代码骨架（TypeScript）：能力注册 + 选择 + 校验 + 执行

下面给一个非常小但实用的骨架（可直接迁移到你自己的平台服务里）。

### 5.1 数据结构

```ts
// capability.ts
export type Risk = 'low' | 'medium' | 'high'

export type Capability = {
  id: string
  name: string
  domain: string
  version: string
  description: string
  inputSchema: object
  outputSchema: object
  owner: string
  deps?: string[]
  tags?: string[]
  policy: {
    risk: Risk
    requires: string[]
    pii?: string[]
  }
}
```

### 5.2 Registry（先用内存/JSON 文件也能跑）

```ts
// registry.ts
import { Capability } from './capability'

export class CapabilityRegistry {
  private map = new Map<string, Capability>()

  register(cap: Capability) {
    this.map.set(cap.id, cap)
  }

  get(id: string) {
    return this.map.get(id)
  }

  list(filter?: { domain?: string; tags?: string[] }) {
    let caps = [...this.map.values()]
    if (filter?.domain) caps = caps.filter(c => c.domain === filter.domain)
    if (filter?.tags?.length) {
      const tags = new Set(filter.tags)
      caps = caps.filter(c => (c.tags ?? []).some(t => tags.has(t)))
    }
    return caps
  }
}
```

### 5.3 Policy Gate（把权限/风险当硬约束）

```ts
// policy.ts
import { Capability } from './capability'

export type CallerContext = {
  userId: string
  scopes: string[]
  riskBudget: 'low' | 'medium' | 'high'
}

export function checkPolicy(ctx: CallerContext, cap: Capability) {
  for (const req of cap.policy.requires) {
    if (!ctx.scopes.includes(req)) {
      return { ok: false as const, reason: `missing scope: ${req}` }
    }
  }

  const order = { low: 0, medium: 1, high: 2 } as const
  if (order[cap.policy.risk] > order[ctx.riskBudget]) {
    return { ok: false as const, reason: `risk too high: ${cap.policy.risk}` }
  }

  return { ok: true as const }
}
```

### 5.4 Schema 校验 + Executor（用 AJV 做 JSON Schema 校验）

```ts
// executor.ts
import Ajv from 'ajv'
import { Capability } from './capability'

const ajv = new Ajv({ allErrors: true })

export async function executeCapability(
  cap: Capability,
  input: unknown,
  impl: (input: any) => Promise<any>
) {
  const validateIn = ajv.compile(cap.inputSchema)
  if (!validateIn(input)) {
    throw new Error(`input schema invalid: ${ajv.errorsText(validateIn.errors)}`)
  }

  const output = await impl(input)

  const validateOut = ajv.compile(cap.outputSchema)
  if (!validateOut(output)) {
    throw new Error(`output schema invalid: ${ajv.errorsText(validateOut.errors)}`)
  }

  return output
}
```

你把这四块拼起来，再接一个“根据任务选能力”的 Resolver（先规则、后 embedding），就能形成企业可跑的最小闭环。

---

## 6) 一个非常现实的落地顺序（建议按周推进）

**第 1 周：把 10 个关键动作 API 化 + Schema 化**

- 选 Top-10 高价值动作（查询客户/创建工单/查订单/退款/发通知…）
- 每个动作补：幂等、错误语义、审计字段

**第 2 周：做 Capability Registry（目录）**

- 每个能力必须有：owner/version/policy/schema
- 做一个“能力准入 CI”：schema 校验 + contract test

**第 3 周：做 Policy Gate + Tracing**

- 先把高风险能力挡住（审批/禁用）
- 先把观测打通（失败可复盘）

**第 4 周：再开始做 Agent 的自动选择与组合**

- 先做规则/域内路由
- 再做 embedding 辅助（但永远让 policy 与 schema 先行）

---

## 延伸阅读

- 系列目录：series/README.md
- 长文全文：README.md
