# 28｜把 Runtime 做成可部署服务：Postgres 表结构（DDL）+ 事件回调 OpenAPI + 幂等与并发控制（Node/TS + Express + pg）

第 27 篇我们讲了生产化的关键能力：状态持久化、Resume、OTel。

这一篇继续把它落成“你明天就能起一个服务”的形态：

- **Postgres 表结构**：runs / step_attempts / artifacts / run_events
- **OpenAPI 契约**：
  - `POST /v1/runs` 创建 run
  - `GET /v1/runs/{runId}` 查询状态
  - `POST /v1/runs/{runId}/events`（工程师回填/审批回调）
  - `POST /v1/runs/{runId}/resume` 触发继续推进
- **幂等与并发控制**：
  - eventId 去重
  - run 锁（`SELECT ... FOR UPDATE`）
  - 乐观锁（version）与“防重复 resume”

技术栈：**Node.js/TypeScript + Express + pg + JSON Schema(Ajv)**

---

## 1) 数据库 DDL（Postgres）

> 目标：可回放、可审计、可恢复。

### 1.1 `runs`：一条编排实例

```sql
CREATE TABLE IF NOT EXISTS runs (
  run_id           TEXT PRIMARY KEY,
  workflow_id      TEXT NOT NULL,
  workflow_version TEXT NOT NULL,
  status           TEXT NOT NULL CHECK (status IN (
    'running','waiting_human','failed','succeeded','compensating','compensated'
  )),
  -- 用于乐观锁（防止并发 resume/事件导致状态被覆盖）
  version          BIGINT NOT NULL DEFAULT 0,

  -- 结构化上下文：建议只存“引用/摘要”，大对象放 artifacts
  vars             JSONB NOT NULL DEFAULT '{}'::jsonb,

  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_runs_workflow ON runs(workflow_id, workflow_version);
CREATE INDEX IF NOT EXISTS idx_runs_status ON runs(status);
```

### 1.2 `step_attempts`：每个 step 的尝试记录（可回放）

```sql
CREATE TABLE IF NOT EXISTS step_attempts (
  run_id      TEXT NOT NULL REFERENCES runs(run_id) ON DELETE CASCADE,
  step_id     TEXT NOT NULL,
  attempt     INT  NOT NULL,

  status      TEXT NOT NULL CHECK (status IN ('running','succeeded','failed','skipped')),

  input_ref   TEXT,
  output_ref  TEXT,
  error       JSONB,

  started_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  ended_at    TIMESTAMPTZ,

  PRIMARY KEY (run_id, step_id, attempt)
);

CREATE INDEX IF NOT EXISTS idx_step_attempts_run ON step_attempts(run_id);
CREATE INDEX IF NOT EXISTS idx_step_attempts_step ON step_attempts(step_id);
```

### 1.3 `artifacts`：大对象/附件/输入输出快照的引用（可接 S3/OSS）

> 你可以只存元数据与地址；真正 payload 存对象存储。

```sql
CREATE TABLE IF NOT EXISTS artifacts (
  artifact_ref TEXT PRIMARY KEY,
  run_id       TEXT NOT NULL REFERENCES runs(run_id) ON DELETE CASCADE,
  kind         TEXT NOT NULL, -- input/output/engineer_report/... 
  content_type TEXT NOT NULL, -- application/json, image/jpeg ...
  uri          TEXT NOT NULL, -- s3://bucket/key 或 https://...
  sha256       TEXT,
  bytes        BIGINT,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_artifacts_run_kind ON artifacts(run_id, kind);
```

### 1.4 `run_events`：外部事件（回填/审批）——必须幂等

```sql
CREATE TABLE IF NOT EXISTS run_events (
  run_id      TEXT NOT NULL REFERENCES runs(run_id) ON DELETE CASCADE,
  event_id    TEXT NOT NULL,              -- 幂等键（由调用方生成）
  event_type  TEXT NOT NULL,              -- engineer_report_submitted / approval_granted ...
  payload_ref TEXT,                       -- 大 payload 存 artifacts
  actor       JSONB,                      -- {type:'engineer', id:'u_1', name:'...'}
  received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (run_id, event_id)
);

CREATE INDEX IF NOT EXISTS idx_run_events_type ON run_events(event_type);
```

### 1.5 `updated_at` 自动维护（可选）

```sql
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trg_runs_updated ON runs;
CREATE TRIGGER trg_runs_updated
BEFORE UPDATE ON runs
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

---

## 2) OpenAPI（关键接口契约）

下面给一个“够用”的 OpenAPI 片段（可直接放到仓库并生成 SDK）。

```yaml
openapi: 3.0.3
info:
  title: Runtime Service
  version: 1.0.0
paths:
  /v1/runs:
    post:
      operationId: createRun
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [runId, workflowId, workflowVersion, vars]
              properties:
                runId: { type: string }
                workflowId: { type: string }
                workflowVersion: { type: string }
                vars: { type: object }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Run'

  /v1/runs/{runId}:
    get:
      operationId: getRun
      parameters:
        - in: path
          name: runId
          required: true
          schema: { type: string }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Run'

  /v1/runs/{runId}/events:
    post:
      operationId: postRunEvent
      parameters:
        - in: path
          name: runId
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [eventId, type, payload]
              properties:
                eventId: { type: string }
                type: { type: string }
                actor:
                  type: object
                  properties:
                    type: { type: string }
                    id: { type: string }
                    name: { type: string }
                payload: { type: object }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  accepted: { type: boolean }
                  deduped: { type: boolean }

  /v1/runs/{runId}/resume:
    post:
      operationId: resumeRun
      parameters:
        - in: path
          name: runId
          required: true
          schema: { type: string }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Run'

components:
  schemas:
    Run:
      type: object
      required: [runId, workflowId, workflowVersion, status, version, vars]
      properties:
        runId: { type: string }
        workflowId: { type: string }
        workflowVersion: { type: string }
        status: { type: string }
        version: { type: integer }
        vars: { type: object }
        createdAt: { type: string }
        updatedAt: { type: string }
```

### 2.1 事件 payload 示例：工程师回填

```json
{
  "eventId": "evt_20260308_0007",
  "type": "engineer_report_submitted",
  "actor": {"type": "engineer", "id": "u_123", "name": "Li"},
  "payload": {
    "ticketId": "t_987",
    "result": "replaced_fuse",
    "measurements": [{"name": "voltage", "value": 220, "unit": "V"}],
    "photos": ["https://.../photo1.jpg"]
  }
}
```

---

## 3) 幂等与并发控制：推荐一套“够稳”的组合拳

### 3.1 event 幂等：`(runId, eventId)` 唯一约束

- 事件接口 `POST /events` 必须让调用方提供 `eventId`
- DB 用 `PRIMARY KEY (run_id, event_id)` 直接去重

### 3.2 run 并发：`SELECT ... FOR UPDATE` + version 乐观锁

推荐策略：

- 在处理 `events` / `resume` 时：
  - `BEGIN`
  - `SELECT * FROM runs WHERE run_id=$1 FOR UPDATE`
  - 检查当前状态（是不是 waiting_human？是不是已 succeeded？）
  - 更新 vars + status，并 `version = version + 1`
  - `COMMIT`

避免：两个并发请求同时 resume，导致 step 重复推进。

### 3.3 防重复 resume：只允许从特定状态迁移

例如：

- `waiting_human` 才能 resume
- `running` 的 resume 直接返回 409

---

## 4) Node/TS + Express + pg：最小可用服务骨架

### 4.1 初始化（pg pool + express）

```ts
import express from 'express'
import { Pool } from 'pg'

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL
})

const app = express()
app.use(express.json({ limit: '2mb' }))

app.listen(3000, () => console.log('runtime on :3000'))
```

### 4.2 `POST /v1/runs` 创建 run（幂等）

```ts
app.post('/v1/runs', async (req, res) => {
  const { runId, workflowId, workflowVersion, vars } = req.body

  const client = await pool.connect()
  try {
    await client.query('BEGIN')

    await client.query(
      `INSERT INTO runs(run_id, workflow_id, workflow_version, status, vars)
       VALUES ($1,$2,$3,'running',$4)
       ON CONFLICT (run_id) DO NOTHING`,
      [runId, workflowId, workflowVersion, vars]
    )

    const r = await client.query('SELECT * FROM runs WHERE run_id=$1', [runId])
    await client.query('COMMIT')

    res.json(mapRun(r.rows[0]))
  } catch (e: any) {
    await client.query('ROLLBACK')
    res.status(500).json({ error: String(e?.message ?? e) })
  } finally {
    client.release()
  }
})

function mapRun(row: any) {
  return {
    runId: row.run_id,
    workflowId: row.workflow_id,
    workflowVersion: row.workflow_version,
    status: row.status,
    version: Number(row.version),
    vars: row.vars,
    createdAt: row.created_at,
    updatedAt: row.updated_at
  }
}
```

### 4.3 `POST /v1/runs/{runId}/events`：写事件 + 幂等去重 + 触发 resume

```ts
app.post('/v1/runs/:runId/events', async (req, res) => {
  const runId = req.params.runId
  const { eventId, type, payload, actor } = req.body

  const client = await pool.connect()
  try {
    await client.query('BEGIN')

    // 1) lock run
    const rr = await client.query('SELECT * FROM runs WHERE run_id=$1 FOR UPDATE', [runId])
    if (rr.rowCount === 0) {
      await client.query('ROLLBACK')
      return res.status(404).json({ error: 'run not found' })
    }
    const run = rr.rows[0]

    // 2) dedupe event
    const ins = await client.query(
      `INSERT INTO run_events(run_id, event_id, event_type, actor)
       VALUES ($1,$2,$3,$4)
       ON CONFLICT (run_id, event_id) DO NOTHING`,
      [runId, eventId, type, actor ?? null]
    )

    const deduped = ins.rowCount === 0

    // 3) if new event, merge payload into vars (demo: store inline; prod: store artifact ref)
    if (!deduped) {
      const nextVars = {
        ...run.vars,
        lastEvent: { eventId, type, actor, payload }
      }

      await client.query(
        `UPDATE runs
         SET vars=$2, version=version+1
         WHERE run_id=$1`,
        [runId, nextVars]
      )
    }

    await client.query('COMMIT')

    res.json({ accepted: true, deduped })
  } catch (e: any) {
    await client.query('ROLLBACK')
    res.status(500).json({ error: String(e?.message ?? e) })
  } finally {
    client.release()
  }
})
```

> 生产建议：payload 存 artifacts（S3/OSS），`run_events.payload_ref` 存引用，`runs.vars` 只存 ref/摘要。

### 4.4 `POST /v1/runs/{runId}/resume`：防重复推进

```ts
app.post('/v1/runs/:runId/resume', async (req, res) => {
  const runId = req.params.runId

  const client = await pool.connect()
  try {
    await client.query('BEGIN')

    const rr = await client.query('SELECT * FROM runs WHERE run_id=$1 FOR UPDATE', [runId])
    if (rr.rowCount === 0) {
      await client.query('ROLLBACK')
      return res.status(404).json({ error: 'run not found' })
    }

    const run = rr.rows[0]
    if (run.status !== 'waiting_human') {
      await client.query('ROLLBACK')
      return res.status(409).json({ error: `cannot resume from status=${run.status}` })
    }

    await client.query(
      `UPDATE runs SET status='running', version=version+1 WHERE run_id=$1`,
      [runId]
    )

    const r2 = await client.query('SELECT * FROM runs WHERE run_id=$1', [runId])
    await client.query('COMMIT')

    // 实际执行：这里应该投递到队列，让 runtime worker 推进 DAG
    res.json(mapRun(r2.rows[0]))
  } catch (e: any) {
    await client.query('ROLLBACK')
    res.status(500).json({ error: String(e?.message ?? e) })
  } finally {
    client.release()
  }
})
```

---

## 5) 把“推进执行”做成异步 worker（强烈建议）

HTTP handler 不要直接跑完整 DAG：

- 会超时
- 会占连接
- 容易重复推进

建议：

- `resume`/`events` 只负责写状态 + 投递一条 job（runId）到队列（Redis/RabbitMQ/Kafka）
- worker 拿到 runId：
  - lock run
  - 看当前 step 状态
  - 执行下一批就绪 step
  - 写 step_attempts + vars
  - 遇到 WAITING_HUMAN 就把 run.status 置为 waiting_human 并退出

---

## 6) 最小“真实落地”清单（上线前必做）

- 事件接口鉴权（webhook secret / mTLS / allowlist）
- payload 脱敏与存储分级（PII/照片）
- artifact store（S3/OSS）+ sha256 校验
- run 的状态迁移表（允许哪些状态→哪些状态）
- OTel traceId 与 runId 绑定（便于追踪）

---

## 延伸阅读

- 27：生产化 Runtime（持久化/Resume/OTel）：./27-productionizing-runtime-persistence-resume-otel.md
- 26：DAG/Saga（编排与补偿）：./26-dag-orchestration-and-saga-compensation.md
- 系列目录：series/README.md
