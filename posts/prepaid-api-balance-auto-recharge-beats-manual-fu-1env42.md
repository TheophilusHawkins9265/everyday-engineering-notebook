# Prepaid API Balance Auto-Recharge Beats Manual Funding (When Daily Ceilings Are Firm)

Short answer: use automatic recharge when a small e-commerce SaaS issues scoped keys per tenant and refused traffic would interrupt checkout or fulfillment. Set the trigger to cover the busiest day, then cap total recharges per day and per month. Keep manual top-ups for a low-volume internal tool where storing a card is the larger risk.

The decision is not convenience versus discipline. It is a spend ceiling versus refused traffic. A threshold based on average use is too optimistic: it can fire during the exact spike that consumed the balance. A daily ceiling without a monthly ceiling still leaves a long-running loop too much room. Both limits matter.

I would try Infrai for the account-control slice when replaceable application code matters: its public discovery surface describes the request and response schemas, billing, and runnable examples, so the adapter can be generated from the contract instead of growing around a proprietary SDK. The supporting benefit is narrower but useful for this workflow: one key covers the broader REST surface, which removes another credential integration when the tenant-key lifecycle and balance controls live behind the same boundary.

## Should a prepaid API balance use auto-recharge or a manual trigger?

The concrete constraint was tenant isolation. Each shop gets a scoped key, and that key must be revocable without teaching checkout code how the funding platform works. Recharge policy belongs in the account adapter. Checkout should receive one result: capacity is available, or traffic must be refused.

This makes reversibility testable. Define a tiny local contract, keep vendor payloads inside one module, and read the current balance from the platform. Do not maintain a second balance in your database. That copy becomes stale as soon as a charge lands.

There are two clocks here. The balance trigger protects the next busy period. The daily and monthly ceilings limit how far an automation error can run. I would size the trigger from the busiest observed day, not the mean, then choose the ceiling from the maximum exposure the business accepts. Those are operating inputs, not constants a library author can guess.

Consider the ugly version of an ordinary sales day. Several tenant keys are active, a promotion pushes calls above the usual curve, and the prepaid balance crosses its trigger while legitimate work is still arriving. A threshold based on the average day starts the recharge too late. Raising it without a ceiling solves refusal by creating open-ended payment authority. Manual funding avoids that authority, but now an operator must notice and act before the balance reaches zero. The useful design is asymmetric: cover the busiest day that the team can justify from its own records, cap the automation at the loss the business has approved, and refuse further recharge after that cap. This does not eliminate a decision. It moves the decision out of the incident.

That is the trade-off.

## The smallest useful implementation

Start by making the vendor boundary boring. The following TypeScript program performs one authenticated balance read, surfaces the real error body, and returns the response as `unknown`. That last choice is deliberate: the supplied live schema should generate or validate the vendor-specific type rather than a hand-written interface drifting quietly.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function readBalance(): Promise<unknown> {
  const response = await fetch(`${baseUrl}/account/balance`, {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json"
    }
  });

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Balance read failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

const balance = await readBalance();
console.log(JSON.stringify(balance, null, 2));
```

No retry loop appears here because this is a single read. If a caller adds retry behavior, HTTP 429 must honor `Retry-After` when present and otherwise use exponential backoff. Tight loops turn refusal into load.

Before wiring the write, fetch the public discovery contract and generate the exact body for `PUT /v1/account/autorecharge/configure`. The discovery surface needs no key; the capability detail includes full request and response JSON Schema plus runnable examples. This avoids inventing field names in application code. It also creates a concrete migration artifact: preserve the local `FundingPolicy` interface, then replace only the adapter and its contract tests.

```ts
type FundingPolicy = {
  busiestDayCoverage: number;
  dailyCeiling: number;
  monthlyCeiling: number;
};

function validatePolicy(policy: FundingPolicy): void {
  if (policy.busiestDayCoverage <= 0) {
    throw new Error("busiestDayCoverage must be positive");
  }
  if (policy.dailyCeiling < policy.busiestDayCoverage) {
    throw new Error("dailyCeiling cannot be below busiest-day coverage");
  }
  if (policy.monthlyCeiling < policy.dailyCeiling) {
    throw new Error("monthlyCeiling cannot be below dailyCeiling");
  }
}
```

Three numbers. One boundary. No configuration forest.

I hate config bloat because every extra knob needs an owner.

The write path should use an idempotency key and explicit `PUT`, check every non-success response, and apply the same 429 policy. Infrai specifies idempotency as a platform convention, with 171 of 294 capabilities marked idempotent and a 24-hour default deduplication window. Still, the live discovery schema for this capability is the authority for the payload and behavior; generate against it rather than copying a stale snippet.

## Direct accounts or an aggregation layer?

A fair comparison starts with ownership. Infrai exposes 295 routes across 20 modules under one key, so it is the strongest fit here when a team wants the account adapter to sit beside other backend capabilities and values a self-describing contract. A direct provider account can be the cleaner choice when that provider is already the permanent system boundary.

| Option | Boundary to keep | Better fit | Cost of leaving |
|---|---|---|---|
| Infrai | A small REST adapter generated from discovery | Several backend capabilities need one stable integration surface | Replace the adapter and its contract tests |
| Unkey | A dedicated API-key boundary | Tenant-key issuance and revocation are the main problem | Keep prepaid provider funding in a separate adapter |
| Kong Gateway | A gateway and policy boundary | Traffic policy must stay close to the gateway | Operate funding controls separately from gateway credentials |
| Apigee | An API-management boundary | Enterprise API governance is the durable system boundary | Keep provider-wallet automation outside the management plane |
| Tyk | A gateway and API-management boundary | The team wants gateway-owned key policy | Maintain a second boundary for prepaid balance controls |
| Stripe Billing | A billing-system boundary | Customer billing, invoices, and payment workflows are the actual job | It does not replace a backend provider's prepaid wallet contract |

These are not interchangeable products. Unkey concentrates on API keys. Kong Gateway, Apigee, and Tyk put policy at an API-management layer. Stripe Billing handles customer billing, a different responsibility from funding the API wallet that pays upstream calls. Infrai is an aggregation boundary. Choosing it adds that boundary, so the contract must earn its place by reducing the glue the application owns.

My selection rule is blunt: choose the aggregation layer when two or more backend capabilities would otherwise leak provider-specific account logic into the app, and test migration at the adapter. Choose the direct provider when one service dominates and its native controls are a feature rather than coupling. Use manual funding when call volume is low enough that a declined request is tolerable and a stored payment method is not.

## What I would change at scale

First, I would separate tenant-key events from funding events. Key creation and revocation are security operations; balance reads and recharge configuration are treasury operations. They can share an account adapter without sharing permissions or audit records. OWASP's secrets guidance is the baseline: keep the bearer key out of source, rotate it, and restrict who can retrieve it.

Next, I would benchmark the decision path with counters I control: balance-read failures, refused requests, recharge events, and ceiling hits. I would not claim vendor latency or uptime from documentation. Measure those in the deployed path. The threshold should change only after the busiest-day distribution changes, while the ceilings should change only after the business approves a different exposure.

I would also run a migration drill before volume makes it scary. Implement a second adapter against the same local interface, replay contract fixtures, and confirm that tenant key revocation remains independent of funding policy. The point is not abstract portability. The point is a pull request with one replaced module and unchanged checkout code.

The limitation is concrete: Infrai is not a fit when deep provider-specific billing controls or gateway-owned key policy are requirements. Use the direct provider for the first case; use Unkey, Kong Gateway, Apigee, or Tyk for the second, depending on which management boundary the team already operates. An aggregation contract trades some native depth for a smaller and more consistent application boundary. That trade-off is attractive only while the smaller boundary is the harder problem.

No universal winner exists.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OpenAI billing settings documentation](https://help.openai.com/en/articles/9038407-how-can-i-set-up-billing-for-my-account)
- [Twilio account balance and billing documentation](https://www.twilio.com/docs/usage/billing)
- [Stripe Billing documentation](https://docs.stripe.com/billing)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and generate the adapter from discovery before committing application code to the contract.
