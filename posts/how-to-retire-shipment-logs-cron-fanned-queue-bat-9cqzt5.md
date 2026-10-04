# How to Retire Shipment Logs: Cron-Fanned Queue Batches and Idempotent Postgres Workers

For scheduled data cleanup in Node.js, use cron only to publish small queue batches, then let an idempotent Postgres worker delete old shipment logs by primary-key window. The deciding constraint is retry behavior: a 900-second cron ceiling makes one large delete a poor unit of work, while a queue gives each bounded window its own success, retry, or dead-letter outcome.

Short answer: schedule the scan, enqueue deterministic windows, and acknowledge a message only after its database transaction commits.

That split fits an e-commerce service that fans out shipment updates and later expires delivery-event logs. Shipping features every week leaves little revenue-per-hour upside in maintaining a homegrown scheduler. The cleanup still needs boring, explicit correctness because standard queue delivery is at least once.

## Why does a shipment-log cleanup need two stages?

A retention job has two different responsibilities. The planner identifies eligible primary-key windows. The worker owns deletion. Combining them means a slow window can consume the cron run, and restarting the whole scan makes progress harder to reason about.

Keep the cron callback tiny.

For example, suppose shipment events become eligible by `created_at`, but new events continue arriving during cleanup. At the start of a run, capture one cutoff timestamp and one maximum eligible ID. Build jobs such as IDs `1..500`, `501..1000`, and `1001..1500`, all carrying that same cutoff. A retry then describes exactly the same deletion set. It doesn't slide forward with the clock, and it can't reach rows inserted after planning. This is more useful than relying on FIFO deduplication because that window lasts only five minutes; a delayed retry can arrive later.

The message should contain coordinates, not rows. A 256KB message limit is ample for a small JSON window, but not a reason to put database records on the queue. Retention can be configured for no more than 30 days, and an acknowledged message is removed rather than retained for Kafka-style replay.

## How should a Node.js cron trigger queue batches for idempotent Postgres cleanup?

Use a stable job ID derived from the table, cutoff, and key window. The database transaction records that ID beside the delete. If an at-least-once queue delivers the message again, the second attempt sees the completed job and exits without applying the mutation twice.

This TypeScript file is the worker-side core. It uses `pg`, accepts one JSON message, and is runnable with a normal PostgreSQL connection string. The schema stores shipment update logs plus a compact cleanup ledger.

```ts
import { createHash } from "node:crypto";
import { Pool, PoolClient } from "pg";

type CleanupJob = {
  cutoff: string;
  firstId: number;
  lastId: number;
};

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

function jobId(job: CleanupJob): string {
  return createHash("sha256")
    .update(`shipment_event_log:${job.cutoff}:${job.firstId}:${job.lastId}`)
    .digest("hex");
}

async function deleteWindow(client: PoolClient, job: CleanupJob): Promise<number> {
  const id = jobId(job);
  const claimed = await client.query(
    `INSERT INTO cleanup_job (job_id, completed_at)
     VALUES ($1, NULL)
     ON CONFLICT (job_id) DO NOTHING
     RETURNING job_id`,
    [id],
  );

  if (claimed.rowCount === 0) {
    const prior = await client.query(
      "SELECT completed_at FROM cleanup_job WHERE job_id = $1 FOR UPDATE",
      [id],
    );
    if (prior.rows[0]?.completed_at !== null) return 0;
  }

  const result = await client.query(
    `DELETE FROM shipment_event_log
     WHERE id BETWEEN $1 AND $2
       AND created_at < $3`,
    [job.firstId, job.lastId, job.cutoff],
  );

  await client.query(
    "UPDATE cleanup_job SET completed_at = NOW() WHERE job_id = $1",
    [id],
  );
  return result.rowCount ?? 0;
}

async function processMessage(job: CleanupJob): Promise<number> {
  const client = await pool.connect();
  try {
    await client.query("BEGIN");
    const deleted = await deleteWindow(client, job);
    await client.query("COMMIT");
    return deleted;
  } catch (error) {
    await client.query("ROLLBACK");
    throw error;
  } finally {
    client.release();
  }
}

const raw = process.argv[2];
if (!raw) throw new Error("Pass one cleanup job as JSON");

processMessage(JSON.parse(raw) as CleanupJob)
  .then(async (deleted) => {
    console.log(JSON.stringify({ deleted }));
    await pool.end();
  })
  .catch(async (error: unknown) => {
    console.error(error);
    await pool.end();
    process.exitCode = 1;
  });
```

Create the ledger once:

```ts
import { Client } from "pg";

const client = new Client({ connectionString: process.env.DATABASE_URL });
await client.connect();
await client.query(`
  CREATE TABLE IF NOT EXISTS cleanup_job (
    job_id text PRIMARY KEY,
    completed_at timestamptz
  )
`);
await client.end();
```

After installing `pg` and its types, a local invocation looks like this:

```bash
npm install pg
npm install --save-dev tsx @types/pg
DATABASE_URL=postgres://localhost/shop npx tsx cleanup-worker.ts '{"cutoff":"2026-07-01T00:00:00Z","firstId":1,"lastId":500}'
```

Ack only after `processMessage` resolves. On a database error, nack so the queue can retry; after repeated failure, inspect the DLQ rather than repeatedly blocking healthy windows. A transport-level `429` also needs exponential backoff and `Retry-After` handling. Don't tight-loop.

## The smallest scheduling boundary

The scheduler needs a public HTTP callback because the cron task does not host application code. That callback plans fixed windows and publishes them through `POST /v1/queue/publish_batch`; a worker receives work through `POST /v1/queue/consume`. The cron itself should have a timeout at or below 900 seconds, but planning and publishing ought to finish far earlier.

The publisher below accepts the batch body as JSON because the exact fields should come from the live public discovery schema, not a copied article. It makes the relevant request mechanics explicit and keeps a retry from publishing the same logical batch twice.

```ts
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const batchBody = process.env.QUEUE_BATCH_BODY;
if (!apiKey || !baseUrl || !batchBody) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_BASE_URL, and QUEUE_BATCH_BODY");
}

const payload: unknown = JSON.parse(batchBody);
const idempotencyKey = createHash("sha256").update(batchBody).digest("hex");

async function publishBatch(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseUrl}/v1/queue/publish_batch`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return publishBatch(attempt + 1);
  }

  const body = await response.text();
  if (!response.ok) {
    throw new Error(`Queue publish failed (${response.status}): ${body}`);
  }
  return body ? JSON.parse(body) : null;
}

console.log(await publishBatch());
```

Infrai is one reasonable boundary here because one key authenticates cron and queue through one plain REST contract without requiring an SDK. Its more important indie-SaaS advantage is replaceability: the application calls one stable contract while the vendor behind a capability can change, so scheduling plumbing doesn't spread through feature code. The API is self-describing: public discovery returns full request and response JSON Schema without requiring a key, which lets the publisher validate its input against the current contract.

There is a separate operational benefit. One key authenticates all 295 routes across 20 modules, and one consolidated bill covers those capabilities. For this cleanup path, that means cron and queue don't create separate credentials to rotate or separate vendor invoices to reconcile. It saves account work, but it is not the decision rule.

There are constraints. The callback must be publicly reachable, and push subscriptions require public HTTPS. Paused cron schedules do not backfill missed triggers; timing can have seconds of jitter; cron expressions do not support nonstandard extensions such as `L`; and run output retains only the first 4KB. Delayed messages top out at seven days. Those boundaries are fine for routine retention, but they should be explicit in the runbook.

## What changes when the workload grows?

Start with primary-key windows because they are cheap to describe and easy to replay. Tune window size from database lock time and replica pressure, not from queue throughput alone. I'm not sure which size is right for a given shop without its query plan, row width, and production load; `500` is an example boundary, not a benchmark.

At higher volume, keep the protocol and change the planner. It can checkpoint by date range, cursor, or primary key, then publish more windows while workers cap concurrency. Partitioning old logs by date may eventually make retention a partition-drop operation, but that is a database design choice and still benefits from a small, retryable control message.

Fan-out for shipment notifications is a related but different job. This queue has no topic primitive, so separate subscriber classes need separate queues. There is also no fan-out/fan-in join. If cleanup evolves into a dependency graph with branching, compensation, and a final join, stop stretching cron plus queues and use Temporal or Apache Airflow. If replay by multiple independent consumer groups is required, Apache Kafka is the more natural log.

## Trade-offs against common alternatives

The choice is operational, not tribal. Outsource undifferentiated scheduling while it stays a two-stage job; bring in a workflow engine only when the workflow itself becomes product-critical.

| Option | Best fit | Retry and idempotency burden | Poor fit |
|---|---|---|---|
| Infrai cron plus queue | HTTP-triggered SaaS retention with one stable REST boundary | At-least-once consumers still require an idempotent worker | Private-only callbacks, DAGs, joins, or Kafka-style replay |
| RabbitMQ | Teams already operating a broker and needing explicit ack/nack controls | Consumer acknowledgements are strong primitives; application idempotency remains necessary | A solo service that doesn't want broker operations |
| GitHub Actions schedule | Low-frequency maintenance already packaged as a repository workflow | The job must define its own checkpoint and retry behavior | Queue fan-out and sustained worker throughput |
| Temporal | Multi-step durable workflows, compensation, and long-running orchestration | Workflow semantics carry more of the retry model | A simple planner plus batch worker where the platform overhead won't earn back time |
| Apache Airflow | Data pipelines with dependencies, backfills, and operator visibility | Task retries and DAG state are first-class | A latency-sensitive application worker path |
| Apache Kafka | Retained event streams and multiple consumer groups | Offsets help replay; side effects still need idempotency | Small cleanup command queues where broker complexity dominates |
| BullMQ | Node.js teams already operating Redis and wanting queue-native retries | Job IDs help deduplicate; database side effects still need a transaction boundary | Teams avoiding Redis operations or needing durable event replay |
| Inngest | Event-triggered application functions with managed step retries | Step IDs and retries reduce orchestration code; database writes remain the application's responsibility | Private-only execution paths or teams wanting direct broker control |
| Trigger.dev | TypeScript background jobs integrated closely with application code | Managed attempts handle execution retries; cleanup mutations still need idempotency | Polyglot workers or an intentionally provider-neutral HTTP boundary |

The catch is clear: cron plus a queue is not a general workflow system. Stick with RabbitMQ when it is already a well-run part of the stack. Choose Temporal or Airflow when dependencies and joins are the actual problem. Choose Kafka when durable replay is a requirement. For ordinary SaaS retention, a deterministic Postgres window, transactional ledger, and ack-after-commit rule keep the design small enough to ship and inspect.

## References

- https://www.rabbitmq.com/docs/confirms
- https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows
- https://docs.temporal.io/
- https://airflow.apache.org/docs/
- https://kafka.apache.org/documentation/
- https://docs.bullmq.io/
- https://www.inngest.com/docs
- https://trigger.dev/docs
