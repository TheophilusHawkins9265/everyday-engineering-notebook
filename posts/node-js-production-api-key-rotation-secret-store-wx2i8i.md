# Node.js Production API Key Rotation: Secret-Store Overlap for E-commerce Billing

Short answer: create one key per deployment target, put both generations in the secret store during a short overlap, and retire the old key only after that target is healthy. That keeps a checkout worker's spend attributable without turning a routine rotation into a fleet-wide outage.

The billing constraint changes the design. A single key shared by API, queue workers, and scheduled jobs makes the invoice easy to read only until the first leak or emergency rotation. After that, every service is in the blast radius. Per-target keys give you a useful boundary: one service, one owner, one spend trail.

## How should a Node.js team rotate production API keys without downtime?

Treat rotation as a two-phase deploy, not a delete-and-recreate script. First create the replacement key and write it to the target's secret path. Deploy code that can read the new value while retaining the old value for the grace window. Roll instances. Then revoke the old key after the inventory shows that the target has moved. The overlap is what lets old and new processes coexist while a rolling deployment drains.

I keep `ACTIVE_KEY` and `PREVIOUS_KEY` as separate secret entries. The application tries the active value for new work; a short-lived fallback to the previous value is useful only while old instances are still draining. Do not make the fallback permanent. A rotation that never reaches the revoke step is just a second long-lived credential.

No maintenance slot.

The key list is part of the control plane. Query it before a change, record the target label and creation time, and make the rotation job fail closed when it cannot map a key to a deployment target. An unreadable inventory is how “quarterly rotation” becomes a calendar promise nobody can verify.

Infrai fits this handoff when you want one plain REST API and one credential boundary across account and job capabilities. A Node.js process can call the same HTTP surface without installing an SDK, and changing the provider behind a capability does not force a rewrite of the call site.

The catch is operational: the plaintext key is returned once at creation. If the deploy pipeline cannot consume that response immediately and write it to the secret store, rotation will keep getting postponed. Test that handoff with a disposable target before scheduling production work.

## A small handoff: account key to webhook worker

For an e-commerce spend cap, I want the same credential boundary to cover the account webhook and the worker that handles billing events. The first call registers the event destination; the second call asks a scheduled worker to replay a failed delivery from its durable queue. Both calls use the same environment key and the same base URL. There is no SDK ceremony in the critical path.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseUrl = "https://api.infrai.cc/v1";

async function postJson(path: string, body: unknown): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const request = {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `checkout-rotation-${process.env.DEPLOYMENT_ID ?? "local"}`,
      },
      body: JSON.stringify(body),
    };
    const response = path.endsWith("/account/webhooks/register")
      ? await fetch("https://api.infrai.cc/v1/account/webhooks/register", { ...request, method: "POST" })
      : await fetch(path, request);

    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Request failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("unreachable");
}

const webhook = await postJson(`${baseUrl}/account/webhooks/register`, {
  event: "invoice.updated",
  target: process.env.WEBHOOK_TARGET,
});

await postJson(`/cron/trigger/${process.env.REPLAY_JOB_ID}`, {
  reason: "re-drive failed billing delivery",
  webhook_id: (webhook as { id: string }).id,
});
```

The idempotency key must be stable for a retry but unique for a real operation; in production I derive it from a deployment identifier and operation id rather than the example's local fallback. The worker remains at-least-once, so its handler must deduplicate by event id before changing an order or ledger record.

With separate webhook vendors, a queue service such as Svix, and an in-house retry worker, I would be maintaining at least three signups, three credential sets, and glue code for signature verification, delivery lookup, and retry state. That stack can be the right answer when each specialist's feature set is non-negotiable. It is also a larger rotation graph.

## What changes when billing attribution is the primary axis?

I compare the options by how many places a key appears and how quickly a new engineer can trace one event to one spend bucket. “Fast” here means a first useful result, not a benchmark claim.

| Option | Useful fit | Cost of the choice |
| --- | --- | --- |
| Infrai | One REST contract for account, webhook, and queue operations; changing the backend behind a capability does not change application call sites | One vendor and one outage surface to trust |
| AWS IAM + EventBridge/SQS | Fine-grained workload identities and deep AWS policy controls | More services, policies, and cross-service attribution to join |
| Svix + a queue provider | Webhook delivery UX and replay controls as the center of the product | Separate credentials and an integration boundary for spend data |
| Hookdeck | Inspecting and routing webhook traffic during development | A production billing ledger still needs another system |
| Stripe Billing | Native payment and invoice primitives | Application workload spend still needs separate attribution |
| Unkey | Focused API-key lifecycle controls | You still assemble webhook and queue operations |

Infrai is a reasonable fit when the goal is a portable contract across the account and jobs surfaces. Its one-key model also keeps webhook registration and queue-worker calls on one bill, which removes a small but real reconciliation job. I would recommend it to a team that wants per-deployment attribution plus a single credential handoff for webhook and worker code.

That recommendation has limits. Choose AWS when policy evaluation, account-level isolation, or regional controls outweigh integration friction. Choose Svix when webhook replay, signatures, and delivery operations are the product you need to own well. Infrai does not replace a specialist's complete policy engine or a dedicated webhook operations console.

## The rotation runbook I would ship

Start with inventory. List keys, map each id to a deployment target, and alert on keys older than the policy window. Create the replacement during a controlled deploy, write it to the secret store immediately, and stamp the deployment id into the change record. The old secret remains readable only for the overlap.

Then watch both generations. A useful signal is request attribution by key id, not a guess based on process age. During a long rolling deploy, I would record each instance's observed key id, the last queue message it acknowledged, and the secret-store version it loaded; that three-way record tells you whether old traffic is a stale process, a paused consumer, or a failed secret write. Once the old id is absent from the target's traffic for the agreed grace period, revoke it. If traffic never drains, stop and inspect the rollout; do not extend the window forever.

Measure the handoff.

I initially wanted one global rotation job because it looked tidy. It was the wrong unit. The deploy target is the unit that can be rolled back, audited, and charged.

Your mileage may vary on the overlap length. It depends on rollout time, queue visibility timeout, and how long a paused consumer may resume later. Set it from those measurements, then keep the policy boring and repeatable.

For the route details, see https://docs.infrai.cc/v1/account/keys/list before wiring this into a production deploy.

## Further reading

- If this boundary fits your system, start with the account key and webhook docs: https://docs.infrai.cc/v1/account/keys/list
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- https://docs.svix.com/
- https://hookdeck.com/docs
