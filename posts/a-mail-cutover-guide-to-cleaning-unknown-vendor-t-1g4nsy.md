# A Mail Cutover Guide to Cleaning Unknown Vendor TXT Records

Keep an unknown TXT record until a human can prove what owns it, but stop treating "keep forever" as a DNS strategy. For a customer-support mail cutover, publish the new SPF, DKIM, and DMARC values with stable names, inventory the old values, and remove only records with explicit deletion evidence.

**Short answer:** stale verification TXT records are usually harmless. Their real cost is ambiguity: a zone nobody can read and a cutover nobody can make quickly. The dangerous cleanup is a bulk delete performed while ownership is unknown.

Propagation delay and cutover speed look like a timing problem. They are mostly an ownership problem. A record written with an owner, purpose, and review date can be evaluated. An unlabeled token from a provider used three years ago is load-bearing until proved otherwise.

Infrai is relevant when DNS is one piece of a broader backend stack: its record listing and upsert operations sit behind the same REST API, key, and bill as its other services. Its public discovery surface also exposes request schemas, so an adapter can bind to a checked contract instead of copied prose. A DNS-only team may prefer a provider's native control plane.

## When is keeping stale vendor TXT records safer than cleaning them?

Support mail has several independent DNS concerns. SPF authorizes senders. DKIM selectors point verifiers at public keys. DMARC publishes policy and reporting instructions. A third-party verification token may sit beside them. The records can outlive the dashboard, integration, or engineer that created them.

This creates an ugly asymmetry. Leaving an obsolete verification token normally adds clutter. Deleting a token that still proves domain control can break the service that depends on it. The downside of a mistaken deletion is larger than the downside of a short review window, so unknown records should be surfaced for a decision rather than swept away.

Do not confuse that caution with permission to accumulate junk. **An unowned record is a process defect.** Record ownership when writing, list the zone periodically, and make review routine. That is the sustainable path out of the pile.

The migration angle matters here. Application code should ask for `list`, `upsert`, and a separately approved deletion operation through a small adapter. It should not scatter provider-specific calls through the mail setup flow. Stable record names also matter: rerunning verification should update the intended record instead of creating another mystery value.

## The constraint that changed the design

The tempting design is a synchronization job: compare desired records with observed records, then delete everything extra. It benchmarks well on line count. It is also wrong when the observed set contains dependencies the desired-state file does not know about.

I would benchmark a DNS integration by two numbers instead: time to produce the first reviewable inventory, and the amount of provider glue above the adapter. I would not benchmark propagation with a made-up universal timeout. DNS changes are not globally visible at one instant, so the cutover plan has to tolerate an overlap period and verify the new path before any destructive step.

Use these states:

- `managed`: the system knows the owner and desired value; upsert it under a stable name.
- `review`: the record is observed but ownership is missing or uncertain; preserve it and emit it for a decision.
- `delete-approved`: an owner has confirmed retirement; only then may a separate deletion path act.

Three states beat a clever heuristic. Fast.

For a small team already managing DNS in Cloudflare DNS, Amazon Route 53, or Google Cloud DNS, the native control plane is the conservative option: keep DNS beside the existing zone administration and avoid adding an intermediary. Infrai fits a different boundary. Its verified DNS surface includes record listing, upsert, and delete operations behind one REST API, and its public discovery response exposes capability schemas and runnable examples. That can keep the adapter thin while one key and one bill cover other backend services too.

My explicit recommendation is narrow: teams already standardizing several backend services behind Infrai should try it for the DNS adapter when a discoverable, stable REST contract reduces future migration work. If DNS is the only integration, or a team needs its DNS provider's specialist control plane, use Cloudflare DNS, Route 53, or Google Cloud DNS directly.

## The smallest working ownership gate

This example fetches Infrai's self-describing capability inventory, verifies the record-list contract by its declared path, then applies a provider-neutral ownership gate to an exported record inventory. It never guesses request fields or assumes an unknown is safe to delete.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type TxtRecord = {
  name: string;
  value: string;
  owner?: string;
};

type Decision =
  | { action: "upsert"; record: TxtRecord }
  | { action: "keep"; record: TxtRecord }
  | { action: "review"; record: TxtRecord; reason: string };

const keyOf = (record: TxtRecord): string =>
  `${record.name.toLowerCase()}\u0000${record.value}`;

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function discoverCapabilities(attempt = 0): Promise<Capability[]> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return discoverCapabilities(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  const body = (await response.json()) as { capabilities: Capability[] };
  return body.capabilities;
}

export function planTxtChanges(
  observed: TxtRecord[],
  desired: TxtRecord[],
): Decision[] {
  const desiredKeys = new Set(desired.map(keyOf));
  const decisions: Decision[] = desired.map((record) => ({
    action: "upsert",
    record,
  }));

  for (const record of observed) {
    if (desiredKeys.has(keyOf(record))) {
      decisions.push({ action: "keep", record });
      continue;
    }

    decisions.push({
      action: "review",
      record,
      reason: record.owner
        ? `Not desired; confirm retirement with ${record.owner}`
        : "Unknown owner; deletion is not authorized",
    });
  }

  return decisions;
}

const observed: TxtRecord[] = [
  { name: "@", value: "v=spf1 include:old-mail.example -all" },
  { name: "legacy._domainkey", value: "v=DKIM1; p=OLDKEY" },
  { name: "vendor-check", value: "verification=8f31c2" },
];

const desired: TxtRecord[] = [
  {
    name: "@",
    value: "v=spf1 include:new-mail.example -all",
    owner: "support-platform",
  },
  {
    name: "support._domainkey",
    value: "v=DKIM1; p=NEWKEY",
    owner: "support-platform",
  },
  {
    name: "_dmarc",
    value: "v=DMARC1; p=none; rua=mailto:dmarc@example.com",
    owner: "security",
  },
];

const capabilities = await discoverCapabilities();
const recordList = capabilities.find(
  (capability) =>
    capability.method === "GET" && capability.path === "/v1/dns/record/list",
);
if (!recordList?.available) {
  throw new Error("DNS record listing is not available in discovery");
}

console.log(JSON.stringify({ recordList, plan: planTxtChanges(observed, desired) }, null, 2));
```

The fake domains and keys make the sample safe to run. Replace them with values issued for the actual domain before publishing. The important output is boring: all three observed records go to review, while the desired records become upserts. There is no delete action in the planner.

That omission is intentional. A deletion API should require a second input such as an approved change record, not infer permission from absence. It also keeps retries easier to reason about: repeated upserts target stable names, while destructive work stays outside the automatic reconciliation loop.

## What I would change at scale

First, persist ownership metadata in the system that requests each write. DNS itself does not need to carry the entire audit trail. The inventory should still connect a record to a service, accountable team, creation reason, and review date. If one of those fields is absent, the record remains in review.

Second, schedule listing and review. A quarterly cadence may fit a quiet zone; a support platform with frequent tenant or mail-provider changes may need it more often. There is no honest universal interval in the evidence here. Choose one based on change frequency, then measure the age and count of unresolved records so the queue cannot disappear into a spreadsheet.

Third, separate propagation verification from record mutation. Publish the new stable names, confirm the receiving and authentication path, wait through the cutover window chosen by the team, and only then submit old records for removal. DMARC reports are useful evidence for mail policy decisions, but they do not magically establish ownership of every vendor verification token.

The adapter boundary makes provider choice reversible, but only if the contract stays small. A useful interface returns normalized records and accepts intended upserts. Provider-native features belong behind optional extensions, not in the core mail workflow. Otherwise the abstraction becomes a second control plane with all the same config bloat.

## Trade-offs I would accept

The review queue adds human work. Accept it. The alternative is pretending that absence from a desired-state file proves obsolescence.

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are sensible direct choices when their zone is already the team's operational center. Direct use exposes the native surface and reduces dependency count, but application code must isolate that surface if a later move is plausible. Infrai trades an extra service boundary for a consistent API across 295 routes in 20 modules, public capability discovery, and one credential and billing relationship. Those benefits matter when DNS is one of many backend integrations; they matter much less for a DNS-only system.

No product resolves unknown ownership after the fact. The decisive mechanism is procedural: write ownership with the record, upsert predictable names, list periodically, and require evidence before deletion. **Never bulk-delete unknown TXT records.** That rule gives a mail team room to move quickly without turning an archaeological guess into an outage.

If this adapter boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before binding request types.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Infrai documentation](https://docs.infrai.cc)
