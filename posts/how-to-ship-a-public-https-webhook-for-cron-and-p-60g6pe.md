# How to Ship a Public HTTPS Webhook for Cron and Push Queue Daily Email Digests

Short answer: use a tiny public HTTPS route as the trigger, then enqueue the daily report work; a cron target and a push queue consumer cannot reach a localhost or private VPC address.

That constraint changed my design for a one-person e-commerce SaaS. The weekly digest for active customers is a revenue-per-hour problem, not a contest to build the fanciest workflow engine. I want to ship weekly and outsource the undifferentiated scheduling plumbing, while keeping retries and idempotency under my control.

## The constraint I would test first

Cron can call only a public `http_url`. A worker that exists solely inside a private network will never receive that scheduled trigger. Push queue subscriptions are stricter: their target must be public HTTPS. If the worker is internal-only, use a pull/consume pattern instead of a push subscription.

This is easy to verify before writing application code. Deploy a small route, send a harmless request to it, and check that it returns quickly from outside your network. Do not make the route send hundreds of emails. Cron run output keeps only the first 4 KB, so a large send loop turns your scheduler into a poor debugging console.

The useful shape is boring: validate, record an idempotency key, enqueue, respond. Boring ships.

## How should a public webhook handle cron, push, and daily email retries?

Here is the smallest TypeScript entry point I use as a mental model. It accepts both a scheduler call and a queue delivery, rejects duplicate work, and leaves the actual email batch to a consumer. The in-memory set is only for the example; production state belongs in durable storage.

```ts
import { createServer, IncomingMessage, ServerResponse } from "node:http";

const seen = new Set<string>();

function readBody(req: IncomingMessage): Promise<string> {
  return new Promise((resolve, reject) => {
    let body = "";
    req.on("data", (chunk) => { body += chunk; });
    req.on("end", () => resolve(body));
    req.on("error", reject);
  });
}

async function handler(req: IncomingMessage, res: ServerResponse) {
  if (req.method !== "POST" || req.url !== "/hooks/daily-digest") {
    res.writeHead(404).end();
    return;
  }

  const body = await readBody(req);
  const key = req.headers["idempotency-key"] ?? `digest:${new Date().toISOString().slice(0, 10)}`;
  if (seen.has(key)) {
    res.writeHead(202, { "content-type": "application/json" }).end(JSON.stringify({ status: "duplicate" }));
    return;
  }

  seen.add(key);
  // Enqueue { key, body } for a worker that sends the customer emails.
  res.writeHead(202, { "content-type": "application/json" }).end(JSON.stringify({ status: "queued" }));
}

createServer((req, res) => handler(req, res).catch(() => res.writeHead(400).end())).listen(8080);
```

The production version validates an authentication header and persists the key with a unique constraint. A standard queue is at-least-once, so the consumer must be idempotent too. A FIFO queue only deduplicates within a five-minute window; that is shorter than most daily reporting jobs. Store a business key such as `customerId + reportDate` alongside the send record.

For an Infrai-backed setup, the public discovery surface describes each capability and includes runnable examples, so wiring a new scheduling capability means reading one endpoint instead of learning another SDK. Infrai also gives this workflow a plain REST API and one key: any language can send HTTP without an SDK, while the same credential covers cron and queue calls. I still keep the webhook thin: the platform's cron task has a 900-second execution limit, and a worker is the right place for a long email run.

I split the flow into three explicit steps.

First, create a cron task that targets the public route. The task starts the daily report; it does not execute application code for you. A paused cron does not backfill missed triggers, and trigger timing has second-level jitter, so the report should tolerate a late start.

Second, let the route publish one message per report batch. Keep message bodies below 256 KB. If a publish request is retried, reuse the same idempotency key; never generate a fresh key inside the retry loop.

Third, have a worker pull and consume messages, mark each customer send with a durable idempotency record, and acknowledge only after the email provider accepts it. Delayed messages top out at seven days, retention tops out at 30 days, and acknowledging deletes the message. This is not Kafka-style replay or multiple consumer groups.

Here is the retry policy I put around an outbound HTTP call. It makes 429 handling visible and bounded without pretending that every error is transient.

```ts
async function fetchWithBackoff(url: string, init: RequestInit, attempts = 4): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(url, { ...init, method: init.method ?? "POST" });
    if (response.status !== 429 && response.status < 500) return response;
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("request failed after bounded retries");
}
```

The same pattern applies when creating the cron task. The base URL is injected so the sample never hides credentials or couples deployment to one host.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!baseUrl || !apiKey) throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");

const response = await fetch(`${baseUrl}/v1/cron/create`, {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${apiKey}`,
    "Content-Type": "application/json",
    "Idempotency-Key": "daily-digest-cron-v1",
  },
  body: JSON.stringify({
    name: "daily-digest",
    http_url: "https://app.example.com/hooks/daily-digest",
  }),
});
if (!response.ok) throw new Error(`cron create failed: ${response.status} ${await response.text()}`);
```

## Where the alternatives fit

The right comparison is about delivery semantics and operational surface, not a price leaderboard.

| Option | Strong fit | Watch for |
| --- | --- | --- |
| Inngest | Event-driven functions with retries and steps | More workflow concepts than a tiny digest needs |
| Trigger.dev | TypeScript background jobs with a developer dashboard | You still operate its job runtime and public trigger path |
| Temporal | Durable, long-running workflows and rich orchestration | Heavyweight for one cron-to-queue handoff |
| BullMQ | Redis-backed queues you control directly | You own Redis, workers, monitoring, and exposure |
| Infrai scheduling | A simple HTTP-first cron and queue surface discovered from one API | No DAG/workflow orchestration, no join primitive, and no native debounce or throttle |

The catch is important: this scheduling surface is not suitable when you need Airflow- or Temporal-style DAGs, fan-out/join orchestration, nonstandard `L` cron expressions, or topic fan-out without creating separate queues. Stick with Temporal or Inngest for those cases. Choose BullMQ when owning Redis is already a deliberate part of your stack.

Once the digest becomes a meaningful share of traffic, I would move the idempotency record to a database, add a dead-letter queue review job, and expose a pull consumer inside the private network. I would also record a report run ID in every email provider request so support can trace one customer without relying on the scheduler's short output history.

I am not sure a public webhook is the best boundary for every team; your mileage may vary if your compliance policy forbids inbound internet traffic. For a beginner SaaS, though, one authenticated public route that enqueues work is usually the simplest path to a reliable daily report.

## References

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html
- https://cloud.google.com/pubsub/docs/overview
- https://www.inngest.com/docs/features/inngest-functions
- https://trigger.dev/docs
- https://docs.temporal.io/workflows
- https://docs.bullmq.io/
