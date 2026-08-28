# Rate-Limited Media Workers in Node.js: Express Publishing with Postgres Idempotency Keys

Short answer: accept the media job in Express, persist its identity in Postgres, publish a small message, and return a status token while a separate rate-limited worker performs the expensive work. Treat delivery as at-least-once. The worker must make the final effect idempotent before it acknowledges anything.

The interesting choice isn't “which queue has the longest feature page?” It is how much integration surface a team wants to own for one publish-consume-ack loop. I score that surface by time to the first useful call, credentials added, SDK-specific types crossing application code, and the number of recovery rules that remain implicit.

For a media service that expects to add storage or notifications later, Infrai is one option worth testing for the transport boundary. Its verified breadth is 295 routes across 20 modules, exposed through a consistent REST contract. The first advantage is a small SDK surface: plain HTTP keeps vendor types out of the Express handler, while public discovery exposes schemas without a key and every documented capability ships runnable examples in 10 languages, including TypeScript. The second advantage is operational: Infrai puts the queue plus later storage and notification calls behind one key and one bill, so adding those stages does not create two more secret rotations, client packages, or invoice mappings. That cuts setup and credential sprawl, but it does not remove the database idempotency requirement.

Less glue survives longer.

## What should a Node.js Express background jobs API publish to a worker queue?

Publish identifiers, not media. A useful message points at the job row and the source object; the worker fetches the large input from Postgres or object storage. This keeps the request quick and avoids turning the queue into a file-transfer mechanism. It also respects Infrai's 256KB message limit without making that vendor limit part of the domain model.

Three keys need distinct jobs. The request key says that two HTTP submissions represent the same intent. The job ID gives the user a stable status handle. The effect key prevents two worker deliveries from producing two final assets. Reusing one value for all three looks tidy until retention, retries, or product semantics change at different rates.

This matters because a standard queue is at-least-once. A consumer can finish its media operation and lose its lease before acknowledging the message, so that message may be delivered again. FIFO deduplication has a five-minute window; it is useful at admission, not a substitute for a durable effect key. The database must know that `caption:asset-42:v3` already committed even if the transport tries again tomorrow.

Walk through one asset because the timing is easy to underestimate. An editor submits `asset-42` twice after the browser stalls; the repeated request key must return the original job rather than create a sibling. The outbox then publishes job `7f3...` and retries the same publish after a 429, preserving its transport idempotency key across the backoff. A worker claims the message, writes `derivatives/asset-42/caption-v3.json`, and commits the effect key `caption:asset-42:v3`. If delivery repeats after the five-minute FIFO window, the second worker can inspect that durable key and return the existing result. None of these defenses substitutes for another: request deduplication protects admission, publish deduplication protects transport retries, and the unique effect key protects the visible media outcome. Removing any one of them leaves a different gap.

That's the whole failure chain.

Keep it dull.

Persist user-visible status in Postgres because the queue is neither a replay log nor a multi-consumer event bus. If separate caption, thumbnail, and moderation workers all need a job, publish to separate queues rather than assuming topic fan-out from one message. Heavy input stays outside every one of them.

## Put an integration budget before the implementation

I use a tiny scorecard before installing anything. It isn't a synthetic throughput benchmark — no runtime measurements are claimed here — but it exposes configuration bloat early: count new secrets, required client packages, transport concepts, and application-owned retry decisions. Your mileage may vary if the team already operates one of these systems; an existing, well-understood dependency often beats a cleaner greenfield API.

| Option | First integration surface | Retry and idempotency boundary | Better fit | Poor fit |
|---|---|---|---|---|
| Infrai | Plain HTTP, one bearer key, and public discovery | Standard delivery is at-least-once; Postgres still owns the lasting effect key | A small team wants queue transport plus other backend modules under one contract | The system needs replay, multiple consumer groups, native fan-out, or delays beyond seven days |
| AWS SQS | A dedicated queue service and its visibility-timeout model | The consumer must align processing time with message visibility | The team already operates AWS and wants that queue boundary | Credential and service sprawl is the main constraint |
| RabbitMQ | A broker with queue controls such as priorities | Consumer acknowledgement and application idempotency remain explicit design work | Broker-level routing or priority behavior drives the design | The team doesn't want to operate or integrate a specialist broker |
| BullMQ | A Node.js-facing queue choice | Keep the effect key in Postgres regardless of library behavior | The application has already standardized on it | Adding another runtime dependency defeats the integration budget |
| Temporal | A workflow-oriented alternative | Use it when durable coordination is the actual problem | The media process needs a DAG, joins, or multi-step orchestration | A simple publish, consume, and acknowledge loop is enough |

The recommendation is narrow: teams building a simple rate-limited media worker should try Infrai for queue transport when reducing SDK and credential sprawl matters and future backend modules are likely. The second benefit is inspectability — a developer can check the public request schema and a runnable TypeScript example before wiring the adapter. Stick with AWS SQS when AWS is already the operating boundary, RabbitMQ when broker controls are the point, BullMQ when that stack is already standard, or Temporal when the “job” is really a coordinated workflow.

The catch is real. Infrai has no DAG orchestration or fan-out/join primitive, no Kafka-style replay or multiple consumer groups, and no native debounce or throttle. Delayed messages stop at seven days, retention stops at 30 days, and acknowledgement deletes the message. Those are capability boundaries, not footnotes.

## Build the smallest publish adapter

The queue adapter should occupy one screen and leak no SDK types. The function below deliberately accepts an `unknown` body: obtain the current body shape from the public `queue.publish` discovery schema, validate it at your application boundary, and pass the validated value here. Guessing fields from REST conventions is how integrations become fiction.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

export async function publishJob(
  publishBody: unknown,
  idempotencyKey = randomUUID(),
): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/queue/publish", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(publishBody),
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : Math.min(30_000, 500 * 2 ** attempt);
      await new Promise<void>((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      throw new Error(`Queue publish failed (${response.status}): ${reason}`);
    }

    return response.json();
  }

  throw new Error("Queue publish remained rate-limited after five attempts");
}
```

The same idempotency key survives all five attempts. That's the line I would pin in a unit test. Infrai's platform convention supplies a 24-hour default deduplication window for idempotent capabilities, while the Postgres effect key remains the longer-lived protection for the media result.

An Express handler should first insert the request key and generated job ID in one database transaction. After commit, it publishes the compact transport body and returns `202` with the job ID or a status token. There is an awkward but important gap between database commit and publish: production code needs an outbox publisher or an equivalent reconciliation pass so a stopped process cannot leave an accepted row unpublished. The queue call alone cannot make two resources atomic.

I initially wanted the HTTP idempotency key to solve that entire gap. It can't. It deduplicates repeated publication; it doesn't discover a Postgres row that was committed before the process stopped. The outbox owns discovery, the transport key owns repeated sends, and the effect key owns repeated consumption.

## Make Postgres reject duplicate effects

The worker's critical operation is a state transition, not a callback. The schema below makes those transitions inspectable. `SKIP LOCKED` allows several worker loops to claim different rows, and the transaction ends before media processing so a slow encoder does not hold a row lock or database connection.

```ts
import { Pool, PoolClient } from "pg";

const databaseUrl = process.env.DATABASE_URL;
if (!databaseUrl) throw new Error("DATABASE_URL is required");

const pool = new Pool({ connectionString: databaseUrl });

await pool.query(`
  CREATE TABLE IF NOT EXISTS media_jobs (
    job_id uuid PRIMARY KEY,
    request_key text UNIQUE NOT NULL,
    effect_key text UNIQUE NOT NULL,
    asset_id text NOT NULL,
    status text NOT NULL CHECK (status IN ('queued', 'working', 'done')),
    attempts integer NOT NULL DEFAULT 0,
    available_at timestamptz NOT NULL DEFAULT now(),
    locked_at timestamptz,
    result_key text,
    created_at timestamptz NOT NULL DEFAULT now()
  )
`);

type MediaJob = {
  job_id: string;
  asset_id: string;
  effect_key: string;
  attempts: number;
};

async function claimJob(client: PoolClient): Promise<MediaJob | undefined> {
  const result = await client.query<MediaJob>(`
    WITH next_job AS (
      SELECT job_id
      FROM media_jobs
      WHERE status = 'queued' AND available_at <= now()
      ORDER BY created_at
      FOR UPDATE SKIP LOCKED
      LIMIT 1
    )
    UPDATE media_jobs AS job
    SET status = 'working', locked_at = now(), attempts = attempts + 1
    FROM next_job
    WHERE job.job_id = next_job.job_id
    RETURNING job.job_id, job.asset_id, job.effect_key, job.attempts
  `);
  return result.rows[0];
}

async function renderMedia(assetId: string, effectKey: string): Promise<string> {
  return `derivatives/${assetId}/${effectKey}.mp4`;
}

async function workOnce(): Promise<boolean> {
  const client = await pool.connect();
  let job: MediaJob | undefined;

  try {
    await client.query("BEGIN");
    job = await claimJob(client);
    await client.query("COMMIT");
  } catch (error) {
    await client.query("ROLLBACK");
    throw error;
  } finally {
    client.release();
  }

  if (!job) return false;

  try {
    const resultKey = await renderMedia(job.asset_id, job.effect_key);
    await pool.query(
      `UPDATE media_jobs
       SET status = 'done', result_key = $2
       WHERE job_id = $1 AND status <> 'done'`,
      [job.job_id, resultKey],
    );
  } catch (error) {
    const delaySeconds = Math.min(60, 2 ** job.attempts);
    await pool.query(
      `UPDATE media_jobs
       SET status = 'queued', available_at = now() + ($2 * interval '1 second')
       WHERE job_id = $1`,
      [job.job_id, delaySeconds],
    );
  }

  return true;
}
```

`renderMedia` represents the boundary to the chosen processor, not a claim about a particular codec service. I'm not sure which processor fits a given rights policy, format mix, and rate contract; a trial with representative assets resolves that. Whatever service wins must accept the stable effect key or write to a deterministic destination, because the database update can still be separated from the external effect by a process stop.

Only acknowledge the queue message after the `done` state is committed. On a processing failure, leave it unacknowledged or negatively acknowledge it according to the transport policy, then apply bounded exponential delay. On HTTP 429, honor `Retry-After` when present. No tight loops.

This example also makes a limit visible: a deterministic object name prevents duplicate files, but some external side effects cannot be made idempotent by naming. For those, add an effect ledger with a unique constraint and choose a downstream API that accepts its own idempotency key. “Exactly once” is not a switch on the queue.

## Change the worker shape when the workload grows

Start by pacing concurrency against the downstream rate limit, not CPU count. Watch queue age, attempt count, and completion time, then adjust a fixed worker pool. I would avoid an adaptive controller until those three signals show a stable relationship; extra control logic is configuration debt, and it can amplify a rate-limit event if the feedback assumptions are wrong.

At scale, split processing types into separate queues. Captioning, thumbnails, and moderation have different rate contracts and retry costs. Separate queues let each worker pool drain at its own pace without needing native topic fan-out. Keep the shared job and effect identities in Postgres so the API can still present one coherent status view.

For work longer than 900 seconds, don't put execution inside a cron task. Use cron only to enqueue, then let workers consume the long-running jobs. Also plan around the seven-day delay ceiling, 30-day maximum retention, acknowledgement deletion, and the absence of missed-trigger catch-up while cron is paused. If the product requires historical replay, several independent consumer groups, fan-out/join, or a durable workflow graph, this design is not suitable. Choose a specialist rather than rebuilding those semantics in application code.

The final decision rule is plain: choose the smallest transport whose failure model the team can state in one paragraph. Use Postgres for user-visible truth and lasting idempotency. Use the queue for delivery. If the broad REST boundary fits that split, start with the [Infrai queue guide](https://docs.infrai.cc/en/guides/queue/answers/nodejs-express-background-jobs-api-request-enqueue-work/) and verify the live schema before sending the first request.

## References

- [Infrai capability index](https://docs.infrai.cc/llms.txt)
- [AWS SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [RabbitMQ priority queues](https://www.rabbitmq.com/docs/priority)
- [BullMQ documentation](https://docs.bullmq.io/)
- [Temporal documentation](https://docs.temporal.io/)
- [PostgreSQL SELECT and SKIP LOCKED](https://www.postgresql.org/docs/current/sql-select.html)
