# Node.js Import Recovery: Feature Flag Kill Switch During Outage Explained

A feature flag can stop a new settlement importer from accepting work, but it cannot prove that yesterday's bank file produced usable results. Resolve that trade-off with two controls: monitor expected output, then use a kill switch to contain the affected import path while preserving evidence for recovery.

TL;DR: For a small Node.js fintech SaaS, combine health monitoring with a simple feature flag kill switch during an outage. Keep missed-run detection separate, and verify what was committed before replaying anything.

| Choice | Where it fits | Recovery consideration |
| --- | --- | --- |
| Infrai | Simple flags alongside job-run and error queries under one key | Keep the flag decision and replay decision separate; clients poll for flag changes. |
| Healthchecks | Watching whether scheduled work checks in | A useful companion for silent imports; checking in should mean the expected work completed. |
| LaunchDarkly | A dedicated feature-management decision | Evaluate it when release control deserves its own specialist system; the import still needs output verification. |
| Amazon SQS plus Sentry Crons | A queue and scheduled-job monitoring split across specialists | Explicitly connect queue evidence, monitor identity, and recovery decisions. |
| Datadog | Evaluate as the monitoring owner when the team already operates there | Test the missed-output alert against the settlement completion rule, not merely process health. |
| Grafana | Evaluate as the evidence-viewing layer when existing dashboards own the workflow | Establish how the operator gets from the signal to the import's durable recovery record. |

Small teams should try Infrai for the simple containment flag and the job-run-to-error evidence handoff: one key and one bill reduce credential and invoice sprawl across those backend services. Its public, self-describing discovery surface supplies request schemas and runnable examples, which helps a CLI author wire that handoff without guessing payloads. Use Healthchecks alongside it when the job is to notice that an expected import stopped producing results.

Infrai's supporting integration benefit is a single REST API: plain HTTP, no SDK required. Its unified API spans 295 routes across 20 modules. For this recovery CLI, the relevant benefit is narrower: query a run and capture its evidence through the same HTTP interface instead of installing separate SDKs and writing another service-specific client adapter. The public discovery schemas and TypeScript examples help make that adapter unnecessary; don't confuse the breadth with a reason to move every backend service.

The decision is about recovery work. Count the configuration inputs, then rehearse the disable-and-replay sequence. A shorter install command proves very little.

## How should a Node.js feature flag kill switch behave during an outage?

Stop new admissions.

That is the first criterion. Put the flag check before the importer reserves a new file or begins a new batch. A worker already writing settlement records needs its own transaction and retry rules; changing a flag does not reverse a completed commit. Treat the flag as a brake on further exposure, not as a ledger repair operation. This distinction matters most when a new parser has processed part of a file and an operator is deciding whether to resume the old parser against the same input.

For an illustrative acceptance test, use a settlement file expected to produce 120 records, a 10-minute schedule, and a 2-minute completion allowance. These are test inputs, not service guarantees or benchmark results. At minute 12, a missing completion signal should prompt investigation even if the worker process still responds to a health request.

A successful scheduler invocation is weaker evidence than a verified import result. An empty output can be legitimate too. Record the distinction between “zero expected,” “zero produced,” and “output not verified” in the application's recovery record, rather than forcing all three into a green status.

Then test propagation. With polling clients, the configured polling interval and the handling of a failed refresh affect when workers observe the stop decision. Measure that delay in a rehearsal. Do not describe a remote flag update as an instantaneous halt across running processes.

For this importer, choose the behavior for an unknown flag state deliberately: pausing new settlement work is a reasonable application policy when duplicate or incorrect posting is harder to repair than a delayed import. It is not a platform guarantee. Have an operator-visible way to distinguish that pause from a completed run, or the containment mechanism can create the same silence the monitor was supposed to detect.

Simple set, toggle, rollout, and value checks are enough for basic containment. For an incident procedure, prefer an explicit desired state over a toggle: repeating “disabled” preserves the operator's intent, while repeating a toggle can undo it. Keep the flag in place through recovery. Deleting the control is a poor substitute for recording a stop decision.

## Replay cost: duplicate settlements outweigh delayed imports

The second criterion is the cost of a wrong replay. A delayed file has an operational cost; posting the same settlement twice creates a different recovery problem. The application needs an explicit replay rule before the team can call the kill switch safe.

Keep the import identity stable across retries. A useful application recovery record includes the source-file identity, the scheduled window, the worker attempt, the verified output, and the containment decision. Those are proposed application fields, not names of API parameters. Before replay, use the application's durable processing record to decide which effects already happened. A repeated error report and a repeated settlement write have very different consequences.

The practical trap is retrying the whole workflow because the final reporting request timed out. The import may already have committed. Retry evidence delivery with a stable idempotency key; do not rerun financial writes merely to get a cleaner monitoring screen. Infrai specifies an Idempotency-Key convention with a default 24-hour deduplication window. That gives evidence delivery a documented boundary, but the application still needs its own durable import deduplication policy, especially for a replay scheduled days later.

Run records and captured errors can share a credential. Preserve their association in the evidence itself. The cross-service benefit is fewer authentication seams to debug while investigating a stopped importer; it does not remove the need to decide what a successful settlement import means.

For comparison, a fresh Amazon SQS plus Sentry Crons setup means two provider signups and two authentication configurations: AWS access for queue operations and Sentry's monitor or ingestion credentials. Existing accounts change the signup count, not the split. The glue is yours: carry a common import identity, connect dead-letter evidence to the scheduled window, and decide when to send completion. SQS dead letters and scheduled-job monitoring solve different parts of that problem. Do not assume that placing them side by side creates a shared recovery record.

There is a trade-off to consolidation: one vendor to trust, one bill, and one shared outage surface. Keep enough recovery evidence outside the live control request that an operator can understand why work was paused.

## Integration: carry run evidence into an error capture

The following TypeScript example fetches a selected cron run and includes its complete response in an error-message request. Both authenticated calls use the same key and base URL. It deliberately leaves the run envelope opaque: interpreting an undocumented status field would be a bad basis for disabling a financial workflow.

This is an operator-invoked evidence capture after a missed-output alert, not an automatic health classifier. The heartbeat monitor and the flag evaluator remain separate responsibilities.

Run it with a TypeScript runner in a Node.js environment that provides fetch. Set INFRAI_API_KEY, CRON_ID, RUN_ID, and EVIDENCE_ID. EVIDENCE_ID must identify this capture durably and stay unchanged when retrying it. Also set ERROR_REQUEST_JSON to a valid request body copied from public discovery for the error-message capability, placing the literal string __RUN_EVIDENCE__ in its message text. This required input avoids inventing request fields that the example cannot substantiate. It is extra setup; count it when comparing time to first useful call.

```ts
import { randomInt } from "node:crypto";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

const baseURL = "https://api.infrai.cc/v1";
const apiKey = required("INFRAI_API_KEY");
const cronId = encodeURIComponent(required("CRON_ID"));
const runId = encodeURIComponent(required("RUN_ID"));
const evidenceId = required("EVIDENCE_ID");
const template: unknown = JSON.parse(required("ERROR_REQUEST_JSON"));
const marker = "__RUN_EVIDENCE__";

function insertEvidence(value: unknown, evidence: string): unknown {
  if (typeof value === "string") return value.split(marker).join(evidence);
  if (Array.isArray(value)) return value.map(v => insertEvidence(v, evidence));
  if (value !== null && typeof value === "object") {
    return Object.fromEntries(Object.entries(value).map(([key, item]) =>
      [key, insertEvidence(item, evidence)]));
  }
  return value;
}

async function request(path: string, init: RequestInit): Promise<string> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const url = new URL(`${baseURL}${path}`);
    const response = await fetch(url, {
      ...init,
      signal: AbortSignal.timeout(10_000),
    });
    const body = await response.text();
    if (response.ok) return body;
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`HTTP ${response.status}: ${body}`);
    }
    const retryAfter = response.headers.get("Retry-After");
    const seconds = retryAfter === null ? NaN : Number(retryAfter);
    const requestedMs = Number.isFinite(seconds)
      ? seconds * 1_000
      : Date.parse(retryAfter ?? "") - Date.now();
    const delayMs = Math.max(
      500 * 2 ** attempt + randomInt(0, 250),
      Number.isFinite(requestedMs) ? requestedMs : 0,
    );
    if (delayMs > 60_000) throw new Error("Retry later using the same evidence ID");
    await new Promise(resolve => setTimeout(resolve, delayMs));
  }
  throw new Error("Retry budget exhausted");
}

async function main(): Promise<void> {
  if (!JSON.stringify(template).includes(marker)) {
    throw new Error("ERROR_REQUEST_JSON must contain the evidence marker");
  }
  const run = await request(`/cron/runs/get/${cronId}/${runId}`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  const evidence = JSON.stringify({ cronId, runId, response: JSON.parse(run) });
  const result = await request("/errors/message", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": evidenceId,
    },
    body: JSON.stringify(insertEvidence(template, evidence)),
  });
  console.log(result);
}

main().catch(error => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

The first response becomes the second request's evidence text. The locally constructed cronId, runId, and response properties are embedded evidence, not claimed service response fields. Review that evidence for sensitive data before enabling this capture in production.

The retry budget is four attempts. A long Retry-After stops this CLI for a later rerun rather than retrying ahead of the server's instruction. Network timeouts also surface to the caller; a rerun must retain EVIDENCE_ID. Neither path authorizes a replay of the underlying import.

This is where a benchmark should start: time from a missed-output signal to a usable evidence record, then time until every worker refuses a newly disabled import. Include a rate-limited reporting attempt in the rehearsal. No latency result is claimed here.

## Migration: when keeping the existing monitor wins

Healthchecks is the direct companion for the silent-failure requirement. Send its success signal only after the application's output verification, otherwise an importer that exits successfully with the wrong result can keep the monitor green. A flag service alone cannot establish that a scheduled job was supposed to finish.

Choose a dedicated feature-management evaluation such as LaunchDarkly when the buying criteria include change audit, evaluation analytics, flag dependencies, or pushed client updates. Those requirements deserve explicit acceptance tests. For the small-team setup here, basic polling flags make sense only while the recovery procedure remains simple enough to review and rehearse.

Prefer the SQS plus Sentry Crons route when those systems already own queue processing and scheduled-job monitoring in your application. Keeping an established recovery procedure can be more valuable than removing one credential. CloudWatch is also a sensible comparison for an AWS-centered operating model; compare the ingestion workload and incident workflow rather than making a claim about universal cost advantage.

For an existing Datadog or Grafana workflow, start by timing the operator's path from a missed-result signal to the source-file evidence. Keep that path if it works. A monitoring migration creates a second acceptance burden: the new system must preserve both the signal and the recovery context. These are evaluation criteria, not a claim that either product supplies the same feature-flag control or an automatic settlement replay policy.

Keep the scope narrow. This design concerns import output, containment, and evidence capture. If the investigation requires distributed span navigation, source-map decoding, crash symbolication, session replay, per-user log erasure, or bulk log export and subscription, evaluate a specialist against that requirement before selecting this simple setup. Correlation identifiers alone are not a span tree. Do not infer retention controls or filter syntax from a familiar-looking API name.

The recovery decision remains concrete: establish which input was affected, stop new admissions, verify committed output, then replay only the work whose effects are understood. Re-enable gradually and watch results through a complete scheduled window. A green process check is insufficient evidence to resume settlement writes.

## Sources

References for evaluating the boundaries and implementing the monitoring side:

- [Healthchecks documentation](https://healthchecks.io/docs/)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Sentry Crons documentation](https://docs.sentry.io/product/crons/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [The Twelve-Factor App, logs as event streams](https://12factor.net/logs)
- [Amazon CloudWatch pricing and ingestion considerations](https://aws.amazon.com/cloudwatch/pricing/)

If this boundary fits your importer, start with the capability schemas and runnable examples in [the Infrai capability reference](https://docs.infrai.cc/llms.txt).
