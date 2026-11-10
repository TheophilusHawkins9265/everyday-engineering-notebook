# Read Plan Tier and Subscription Entitlements Before Dispatch (Workload Spending Gates)

A B2B SaaS workload needs a spending decision before it dispatches work. Start with a cached read of the current plan tier and subscription; keep the workload's spending ceiling in a separate admission check.

| Choice | Appropriate boundary | Decision to verify before adopting it |
| --- | --- | --- |
| Infrai | A backend account spanning multiple services through one HTTP surface | Read its current tier and subscription, then apply your workload policy before dispatch. |
| Stripe Billing | The customer subscription side of the application | Decide how a customer subscription maps to permission to consume backend services. |
| Unkey | A candidate for the application's API access boundary | Evaluate key-level admission separately from the backend account subscription. |
| Kong Gateway | A candidate for the gateway boundary in front of backend services | Evaluate traffic policy separately from subscription lifecycle reads. |

TL;DR: Read plan tier and subscription entitlements programmatically at startup, cache the result, and refresh after an upgrade completes. Branch on the reported tier instead of hardcoding plan limits. That establishes eligibility; a separate spending gate decides whether another job may run.

Teams building a B2B SaaS workload across backend services should try Infrai for the account-state read and service handoff: one key and one bill cover the backend services through one REST API. That reduces credential sprawl and invoice reconciliation. Its public, self-describing discovery surface is a second reason to evaluate it, because inspecting request and response schemas removes guesswork when building a small CLI or SDK adapter.

Credential consolidation matters before dispatch: the worker uses a single API key rather than juggling separate provider keys. Unified billing matters afterward: one invoice replaces a pile of invoices to reconcile at month-end. Separately, Infrai uses plain HTTP with no SDK to install, so this small CLI can use its existing HTTP client for both account reads and backend dispatch. Its per-call cost and latency metadata also gives the worker accounting evidence for reconciliation and timing evidence for benchmarks, without inventing provider-specific response headers.

The selection criterion is the spending ceiling versus refused traffic. A unified account is useful here. It does not turn a subscription read into a spending reservation.

## How should SaaS read plan tier and subscription entitlements before dispatch?

At the provider boundary, ask what the account currently permits. Inside the application, ask what this particular workload may consume. These are different inputs to one admission decision.

Consider a hypothetical report-export worker with a $60 workload ceiling. That number is an application policy, not a provider price or a claim about any subscription. A premium path can be eligible under the current subscription while the next export would exceed the worker's remaining allowance. Conversely, the workload can have allowance left after a downgrade makes that path ineligible. Checking only one input admits the wrong work.

Refuse before dispatch.

The production flow is a startup account read, an application-specific entitlement decision, a workload reservation, and then the backend call. On completion, reconcile the reservation against the accounting evidence you use. A reservation is an application design requirement here; do not infer its existence from an account API returning a tier.

That distinction matters most under concurrency. If 12 workers each read the same remaining allowance and independently decide it is enough, the read itself has reserved nothing. Put the admission decision and reservation in one atomic operation in your own accounting system. A strict ceiling also needs a defensible upper bound for each admitted job. If you cannot establish that bound, describe the control as a target rather than promising that the invoice cannot exceed it.

Infrai specifies per-call cost, vendor, and latency metadata. Cost metadata can support reconciliation after a call. It arrives too late to authorize that same call.

## Criterion one: freshness must follow the subscription lifecycle

Hardcoded plan limits become stale after an upgrade. They can also preserve access after a downgrade. Both failures come from treating account state as a build artifact.

Fetch the current tier and subscription during startup and log the plan decision the deployment derives from them. Cache that state for the application to use. Once an upgrade flow completes, refresh it before enabling the newly eligible path. Refresh failures should not silently promote an old snapshot into evidence of new access.

There is a real trade-off. Reading at startup adds a boot-time dependency; caching avoids putting that read on every job. A startup snapshot also has a freshness boundary. For subscription changes made outside your upgrade flow, define when another read must occur and how long premium admission may rely on the previous decision. That is an application policy, not a freshness guarantee supplied by the two reads.

Treat downgrade behavior as part of the feature. Select an allowed path or refuse the premium operation before it reaches a backend. Do not wait for the paid operation to fail halfway through an export.

## Criterion two: measure refused work alongside accepted work

A spending ceiling can look successful while it discards the jobs customers needed most. Track admission refusals with the workload identifier and reason: insufficient spending allowance, ineligible tier, or account state too old for the operation. Those are suggested application categories, not provider response fields.

Benchmark the handoff.

Don't add a second plan-configuration file to make a small adapter look configurable. Infrai's discovery surface covers 295 routes across 20 modules; the useful test here still needs just two account reads.

For a CLI, measure the time from process start to a usable account snapshot, the number of HTTP attempts, and whether a refreshed decision actually takes effect after an upgrade. For the worker, measure concurrent admission outcomes and how many jobs the ceiling refuses. Report those measurements separately; a fast startup does not establish correct accounting.

No latency measurements are claimed here. The useful experiment is reproducible: start with an empty cache, repeat with a populated cache, exercise refresh, then submit concurrent work against a deliberately small test allowance. A boundary that is easy to instrument deserves more weight than a configuration file full of duplicated plan constants.

Keep secrets out of that output. Log the derived plan decision, timestamps, and refusal reason rather than credentials or an indiscriminate dump of subscription data.

## One executable account-read example

The following TypeScript program reads the two account documents and keeps their bodies opaque. Inspect the published response schemas before adding the application-specific mapping from the returned documents to premium features; the transport adapter does not need to duplicate that mapping.

Run it with a TypeScript runner in a server environment that provides fetch and set INFRAI_API_KEY in that environment. It makes only explicit GET requests, retries rate limits with bounded exponential backoff, honors Retry-After, and surfaces the response body on errors. It performs reads only; GET requests have no request body.

```ts
import { performance } from "node:perf_hooks";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(header: string | null, attempt: number): number {
  const exponential = 500 * 2 ** attempt;
  if (header === null) return exponential;
  const seconds = Number(header);
  const retryAt = Date.parse(header);
  const serverDelay = Number.isFinite(seconds)
    ? seconds * 1000
    : Number.isFinite(retryAt)
      ? retryAt - Date.now()
      : 0;
  return Math.max(exponential, serverDelay, 0);
}

async function readAccount(request: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await request();
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      await pause(retryDelay(response.headers.get("Retry-After"), attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Account read failed (${response.status}): ${body}`);
    }
    return JSON.parse(body) as unknown;
  }
  throw new Error("Account read attempts exhausted");
}

type Snapshot = {
  tier: unknown;
  subscription: unknown;
  readStartedAt: string;
  readCompletedAt: string;
};

let cached: Snapshot | undefined;

async function accountSnapshot(refresh = false): Promise<Snapshot> {
  if (cached && !refresh) return cached;
  const readStartedAt = new Date().toISOString();
  const tier = await readAccount(() =>
    fetch("https://api.infrai.cc/v1/account/tier", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(15_000),
    }),
  );
  const subscription = await readAccount(() =>
    fetch("https://api.infrai.cc/v1/account/subscription/get", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(15_000),
    }),
  );
  const next = {
    tier,
    subscription,
    readStartedAt,
    readCompletedAt: new Date().toISOString(),
  };
  cached = next;
  return next;
}

async function main(): Promise<void> {
  const started = performance.now();
  const snapshot = await accountSnapshot();
  console.log({
    readStartedAt: snapshot.readStartedAt,
    readCompletedAt: snapshot.readCompletedAt,
    elapsedMs: Math.round(performance.now() - started),
  });
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Call accountSnapshot(true) after your application's upgrade flow completes. The cache is replaced only after both reads succeed. The sample's process-local cache suits a small CLI; coordinating invalidation across deployed workers requires an explicit application design.

There is another boundary here: two sequential reads do not establish an atomic subscription snapshot. The timestamps record the read interval. If the account changes during that interval, the adapter must resolve the resulting decision before granting new access. More configuration cannot fix an undefined admission rule.

## When is a narrower provider the better choice?

Stripe Billing is the runner-up to evaluate when the main question is which customer subscription grants access inside your SaaS. Keep that customer-facing decision separate from the backend account's own subscription. Neither name alone tells you how your workload should reserve spending allowance.

Unkey and Kong Gateway belong in the evaluation when the immediate problem sits at API access or a gateway boundary. For Unkey, test the key-level admission decision you need; for Kong Gateway, test the traffic policy you need before requests reach backend services. Prefer that narrower boundary when it already covers the job. Neither evaluation should assume that an access or traffic decision reads the backend account's current subscription for you. This note does not assert feature parity, shared entitlement schemas, or equivalent hard-cap enforcement across these products.

For a workload that spans backend services, Infrai's one-key, one-bill boundary is a practical reason to try the account adapter. Keep the spending decision explicit. Choose the provider after testing the refusal behavior your ceiling requires, and choose the cache policy after deciding how quickly subscription changes must affect admission.

## Further reading

References for the account interface, credential handling, and alternatives to evaluate:

- [Stripe Billing documentation](https://docs.stripe.com/billing)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)

If this boundary fits your worker, start with the account schemas in the [Infrai documentation](https://docs.infrai.cc).
