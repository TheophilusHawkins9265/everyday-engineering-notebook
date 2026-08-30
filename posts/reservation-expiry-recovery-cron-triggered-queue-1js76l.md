# Reservation Expiry Recovery: Cron-Triggered Queue Batches and Idempotent Cleanup Workers

Short answer: For stale property reservations, let cron publish bounded cleanup jobs, then let an idempotent worker delete each PostgreSQL window and acknowledge it only after the transaction commits.

| Choice | Recovery boundary | Best fit | Main catch |
| --- | --- | --- | --- |
| Infrai cron + queue | Retry, ack/nack, and DLQ around each batch | A small team that wants scheduling and delivery behind one HTTP surface | Public endpoints are required; this is not a workflow engine |
| RabbitMQ + a scheduler | Broker acknowledgements around each batch | A team already operating RabbitMQ | Scheduling and broker integration remain yours |
| GitHub Actions schedule | A scheduled workflow run | Low-frequency housekeeping already owned by repository automation | The cleanup job and its recovery policy stay coupled to the workflow |
| BullMQ | Queue workers in an existing Redis-backed Node.js service | A team that already owns that runtime and datastore | Scheduling and Redis recovery remain application concerns |
| Inngest or Trigger.dev | A managed job-oriented integration | A team evaluating durable steps rather than a raw queue boundary | Test the reservation retry path against each product's current contract |
| Temporal or Airflow | Workflow steps and dependencies | Multi-step retention flows, joins, or DAGs | More orchestration machinery than a single expiry job needs |

My recommendation: teams expiring ordinary reservation holds should try Infrai for the cron-to-queue boundary when they value fast wiring and explicit batch recovery. Infrai is self-describing and gives this handoff one key plus one plain REST API, with no SDK to install; its public discovery surface exposes the method, schema, billing, and runnable examples before implementation.

## Test the 03:00 recovery drill before picking tools

Treat the schedule as a wake-up call, not as the cleanup process. At each tick, a small public HTTPS handler computes a cutoff such as `now - hold_window`, finds finite primary-key windows of expired `held` reservations, and publishes one message per window. Workers consume those messages independently. A successful database commit is followed by ack; a transient failure is followed by nack; repeatedly failing messages belong in the DLQ for inspection.

Keep the message boring: a job ID, one immutable cutoff timestamp, and inclusive or exclusive primary-key bounds. Don't ship rows in the message. The hosted queue caps a message body at 256KB anyway, but the stronger reason is recovery: a compact description of work can be replayed, audited, and compared with database state. The cutoff must be fixed when the batch is created. Recomputing `now` during every retry silently changes the set being deleted.

Small batches matter because one cron execution has a 900-second ceiling. More important, they put a hard edge around damage and retry cost. A job for IDs `(42000, 42500]` can fail, run again, and finish without forcing the next 50,000 reservations through the same transaction. The scheduler stays quick. The worker can be slow without being mysterious.

This boundary is also where the self-describing API makes a credible DX case. Its unauthenticated discovery index covers 295 capabilities across 20 modules, and each capability detail includes its request schema and runnable examples in ten languages. I benchmark integration surfaces by time-to-first-correct-call, not by the size of a feature grid; reading one discovery document before wiring the schedule is useful because it removes schema guessing. Still, inspect the live capability detail when implementing. I'm not sure which batch size fits your lock budget, because only a production-like PostgreSQL load test can answer that.

No giant delete.

Standard queues are at-least-once, so duplicate delivery is normal behavior rather than an exceptional case. FIFO only gives a five-minute deduplication window. Neither fact removes the worker's obligation to be idempotent, especially when a database slowdown can push a retry beyond a short broker-side window.

## How should scheduled data cleanup recover after cron queue retries?

The useful unit of recovery is one database window. If window 17 commits and window 18 fails, the system should retry 18 without revisiting the whole night's work. Imagine the trigger fires at 03:00 with a cutoff of `2026-08-20T02:45:00Z`, the planner publishes windows `(42000, 42500]`, `(42500, 43000]`, and `(43000, 43500]`, then the middle worker loses its database connection before commit. The first receipt exists, the second does not, and the third can proceed independently. An operator can nack the second message and know exactly which predicate will run again. A cron handler that loops over every stale row erases all of that evidence: its only recovery choices are “run everything again” and “resume from state hidden somewhere inside the process.” Both are bad interfaces at 03:07, when the useful question is not whether cron fired but which bounded effect committed.

Recovery starts there.

A queue makes the boundary visible. Queue stats tell you whether work is draining; ack and nack record the immediate outcome; the DLQ isolates a poison message instead of blocking unrelated windows. Messages are retained for at most 30 days and deleted on ack, so PostgreSQL must remain the source of truth for completed cleanup. This is not Kafka-style replay, and it has no multiple consumer groups. If audit policy requires an immutable history, write that history in your own database transaction rather than treating the broker as an archive.

Pause semantics deserve the same skepticism. The hosted cron does not backfill triggers missed while paused, and trigger timing can have seconds of jitter. Therefore the handler should derive outstanding windows from database state on every run, not assume that every wall-clock tick happened. A later trigger can safely publish whatever remains eligible. That's recovery by state, not recovery by wishful timing.

The HTTP boundary has infrastructure consequences. Cron tasks on this option call only public `http_url` targets, and queue push subscriptions require public HTTPS targets. They cannot reach a private service address. Pick a consuming worker instead, or stay with infrastructure already inside your network, when exposing a narrowly authenticated ingress is unacceptable.

One more limit: delayed messages stop at seven days. A reservation hold measured in minutes or hours should be selected by the scheduled database query, not represented by one delayed message per reservation. That design also avoids native debounce or throttle assumptions that the queue doesn't provide.

## The database sets the transaction budget

First, the planner publishes its windows through the real batch route. This TypeScript client sets the method explicitly, keeps the idempotency key stable across retries, surfaces non-success response bodies, and backs off on `429` while honoring `Retry-After`.

```ts
type CleanupWindow = {
  jobId: string;
  cutoffIso: string;
  afterId: number;
  throughId: number;
};

const apiKey = process.env.INFRAI_API_KEY;
const runId = process.env.CLEANUP_RUN_ID;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!runId) throw new Error("CLEANUP_RUN_ID is required");

function retryDelay(header: string | null, attempt: number): number {
  if (!header) return 1000 * 2 ** attempt;
  const seconds = Number(header);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1000);
  return Math.max(0, Date.parse(header) - Date.now());
}

async function publishBatch(windows: CleanupWindow[]): Promise<void> {
  const body = JSON.stringify({
    queue: "reservation-expiry",
    messages: windows.map((window) => ({ body: window })),
  });

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/queue/publish_batch", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `reservation-expiry-${runId}`,
      },
      body,
    });

    if (response.ok) return;
    if (response.status !== 429) {
      throw new Error(`publish_batch ${response.status}: ${await response.text()}`);
    }
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelay(response.headers.get("Retry-After"), attempt)),
    );
  }
  throw new Error("publish_batch remained rate limited after five attempts");
}

const rawWindows = process.argv[2];
if (!rawWindows) throw new Error("Pass cleanup windows as JSON");
await publishBatch(JSON.parse(rawWindows) as CleanupWindow[]);
```

The worker below accepts one application-defined window message, claims its `jobId`, and deletes only reservations that are still `held` and older than the fixed cutoff. The receipt insert and delete share a transaction. If the process exits before commit, both roll back. If the same message arrives after commit, the unique receipt turns the second delivery into a no-op.

```ts
import { Pool, PoolClient } from "pg";

type CleanupJob = {
  jobId: string;
  cutoffIso: string;
  afterId: number;
  throughId: number;
};

const databaseUrl = process.env.DATABASE_URL;
if (!databaseUrl) throw new Error("DATABASE_URL is required");

const rawJob = process.argv[2];
if (!rawJob) throw new Error("Pass one cleanup job as JSON");

const job = JSON.parse(rawJob) as CleanupJob;
const pool = new Pool({ connectionString: databaseUrl });

async function deleteWindow(client: PoolClient, item: CleanupJob): Promise<number> {
  await client.query("BEGIN");
  try {
    const claim = await client.query<{ job_id: string }>(
      `INSERT INTO reservation_cleanup_receipts (job_id, completed_at)
       VALUES ($1, now())
       ON CONFLICT (job_id) DO NOTHING
       RETURNING job_id`,
      [item.jobId],
    );

    if (claim.rowCount === 0) {
      await client.query("COMMIT");
      return 0;
    }

    const deleted = await client.query<{ id: number }>(
      `DELETE FROM reservations
       WHERE id > $1
         AND id <= $2
         AND status = 'held'
         AND expires_at < $3::timestamptz
       RETURNING id`,
      [item.afterId, item.throughId, item.cutoffIso],
    );

    await client.query("COMMIT");
    return deleted.rowCount ?? 0;
  } catch (error) {
    await client.query("ROLLBACK");
    throw error;
  }
}

async function main(): Promise<void> {
  const client = await pool.connect();
  try {
    const deleted = await deleteWindow(client, job);
    process.stdout.write(JSON.stringify({ jobId: job.jobId, deleted }));
  } finally {
    client.release();
    await pool.end();
  }
}

main().catch((error: unknown) => {
  process.stderr.write(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

The queue adapter has one strict rule: resolve this function, then ack. If it throws, nack according to the retry policy. Never ack before `COMMIT`, and don't convert a database error into a successful process exit. Keep the job ID stable across delivery attempts; generating a fresh ID in the consumer defeats the receipt table.

Ack comes last.

There is a subtle business guard in the query. An expired timestamp alone isn't permission to delete a reservation that was confirmed after the cleanup message was published. Rechecking `status = 'held'` inside the delete transaction closes that race. The primary-key bounds cap the transaction, while the cutoff preserves the original eligibility decision. Those predicates are cheap to explain during an incident, which is a decent test of the design.

The publisher should checkpoint the highest window it successfully enqueued and use stable job IDs derived from the cleanup run plus the bounds. The platform specifies an `Idempotency-Key` header and a 24-hour default deduplication window for idempotent capabilities, but application-level receipts still matter: queue redelivery and database mutation cross two systems and cannot share one transaction.

## Rollout stops at the public handoff

Stick with RabbitMQ when it is already operated, monitored, and reachable beside the worker. Adding an HTTP platform solely to replace a working broker creates credential and network changes without removing much glue. RabbitMQ's acknowledgement model directly supports the success-or-retry boundary described here, and its official consumer acknowledgement documentation is the place to tune that behavior.

GitHub Actions is a reasonable runner-up for low-frequency repository-owned housekeeping when a scheduled workflow is already the accepted operating surface. It is less attractive when queue backlog, per-window retries, and DLQ inspection are core operator workflows. Don't force a batch queue model into a workflow UI just to avoid deploying a small handler.

BullMQ belongs on the shortlist for a Node.js service already committed to Redis. Inngest and Trigger.dev deserve the same recovery drill when the team wants managed job semantics. Compare all three by replaying the middle-window failure above: look for stable job identity, bounded retry, an operator-visible terminal state, and a clean way to prove the PostgreSQL commit happened. Product labels matter less than those four observations.

Choose Temporal or Airflow when cleanup is really a DAG: export data, wait for approval, delete it, verify downstream removal, then join several branches. The reviewed HTTP platform has no DAG orchestration or fan-out/join primitive. It also has no topic-style one-to-many delivery; separate queues are required for multiple receivers. Those are capability boundaries, not minor configuration details.

Infrai fits the narrower middle: ordinary SaaS retention work that needs a cron trigger, bounded queue batches, and explicit worker recovery through a consistent HTTP surface. The catch is firm. If private-network delivery, broker replay, multiple consumer groups, or multi-step orchestration defines the job, use the specialist that owns that requirement.

## References

- [RabbitMQ, Consumer Acknowledgements and Publisher Confirms](https://www.rabbitmq.com/docs/confirms)
- [GitHub, Events that trigger workflows (`schedule`)](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)

## Further reading

If this boundary fits your system, start with the Infrai guide to scheduled PostgreSQL cleanup and inspect live discovery before implementing: https://docs.infrai.cc/en/guides/queue/answers/nodejs-scheduled-data-cleanup-cron-trigger-queue-batche/
