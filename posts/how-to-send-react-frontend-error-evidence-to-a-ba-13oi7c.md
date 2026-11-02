# How to Send React Frontend Error Evidence to a Backend Collector (Marketplace Privacy)

| Evidence strategy | Customer incident reconstruction | Attribution boundary |
| --- | --- | --- |
| Aggregate error counts | Shows changes in failure volume; lacks individual stack locations | Useful for operation totals |
| Restricted browser events plus a backend collector | Preserves release, environment, and stack positions for a reported failure | Associates retained bytes with a marketplace operation |
| Broad browser context capture | Requires a larger privacy review before retention | Extra context needs an explicit owner and purpose |

TL;DR: For React frontend error tracking, send restricted stack evidence to a backend collector with release and environment labels. For a marketplace, retain enough evidence to locate the failing checkout code and attribute retained event bytes to checkout or listing. Keep customer identifiers, raw messages, and URLs out of this channel. This is the least complex choice here that preserves event-level evidence.

The example below connects a React frontend's global error capture to a TypeScript collector running on Node.js. It handles the browser events associated with `window.onerror` and `window.onunhandledrejection`, sends restricted stack positions with a release and environment, and rejects bodies outside that contract. It does not establish which customer saw an error. A support report needs the deployment and approximate time; more precise correlation requires a separately reviewed design.

Keep that boundary explicit.

## What should a React frontend error tracking backend collector retain?

Two criteria decide this design: whether the evidence can locate the failing code, and whether each retained byte has an attributable purpose. Those criteria are related. An error counter can show that checkout became unhealthy, but it cannot identify a stack location from an individual failure. An unrestricted error message might explain more, yet it can also contain a customer address or a seller's free-form text. Collecting everything makes the privacy boundary depend on every future exception message.

Use a deliberately smaller contract: capture time, operation, generic error category, release, environment, and stack line/column pairs. Positions alone need the matching build artifact to be useful. Preserve that artifact through your incident investigation window, and verify the relationship with a synthetic failure during deployment. If you need source maps, keep their access policy separate from public event intake. The example does not upload or resolve them.

This trades immediate readability for less exposed context.

Anonymous positions can be ambiguous across multiple bundles, so treat this implementation as a starting point for a controlled build. A multi-bundle deployment should add a build-generated, allowlisted artifact identifier before relying on it for reconstruction. Do not substitute the browser's raw script URL: query parameters and path segments can carry context you never meant to collect. Consider a synthetic checkout failure after a deployment that splits checkout and listing into different bundles. Both bundles can contain a line 42, column 7. A release identifier narrows the artifact set, but those two numbers alone do not choose a file. That is the specific trap: the payload looks precise while the reconstruction remains ambiguous. The acceptance test must resolve the fixture to the intended artifact before the team relies on this evidence for customer incidents.

## 2. Attribute bytes before debating infrastructure

The primary accounting unit here is accepted payload bytes per operation, with accepted event count beside it.

Call it a storage input measure.

It excludes indexes, replication, transport overhead, and staff time. Keeping those exclusions visible prevents a tidy counter from masquerading as a complete bill.

Use bounded labels. `checkout`, `listing`, and `unknown` are enough for this demonstration. A customer ID would create a different privacy contract, while a label containing an arbitrary URL would make the accounting series grow with user input. Metrics describe aggregated measurements; the metrics concepts reference below is useful background for keeping those aggregates distinct from individual incident evidence.

Benchmark the actual pipeline before expanding the schema. Record encoded bytes, acceptance ratio, and time to the first accepted synthetic event. Compare the same fixture across releases. No benchmark result is assumed here, and a collector returning success proves intake, not durable retention. This distinction matters when a CLI smoke test becomes someone's deployment gate.

## 3. Implement capture and a strict intake boundary

Start in the frontend entry module, before mounting the application. This browser-side TypeScript uses event listeners so it can coexist with existing handler properties. The `error` listener covers the global error event behind `window.onerror`; `unhandledrejection` covers the event behind `window.onunhandledrejection`. A React render error handled by an error boundary needs a separate call to the same reporting function. Global listeners are not complete component coverage.

The example values are deployment configuration, not observed release data. Replace the release in both programs together.

Run the browser application and collector behind the same origin, routing `/client-errors` to the collector.

```ts
type Operation = "checkout" | "listing" | "unknown";
type Category = "runtime" | "rejection";

const release = "market-demo-1";
const environment = "staging";
let operation: Operation = "unknown";

export function setOperation(next: Operation): void {
  operation = next;
}

export async function reportError(
  category: Category,
  value: unknown,
): Promise<boolean> {
  try {
    const frames: Array<{ line: number; column: number }> = [];
    const stack = value instanceof Error ? value.stack : undefined;
    for (const row of (stack ?? "").slice(0, 8192).split("\n")) {
      const match = /:(\d+):(\d+)\)?\s*$/.exec(row);
      if (!match) continue;
      const line = Number(match[1]);
      const column = Number(match[2]);
      if (!Number.isSafeInteger(line) || line < 1) continue;
      if (!Number.isSafeInteger(column) || column < 1) continue;
      frames.push({ line, column });
      if (frames.length === 8) break;
    }
    const response = await fetch("/client-errors", {
      method: "POST",
      credentials: "omit",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        release,
        environment,
        operation,
        category,
        capturedAt: new Date().toISOString(),
        frames,
      }),
    });
    return response.ok;
  } catch {
    return false;
  }
}

window.addEventListener("error", (event: ErrorEvent) => {
  void reportError("runtime", event.error);
});
window.addEventListener("unhandledrejection", (event: PromiseRejectionEvent) => {
  void reportError("rejection", event.reason);
});
```

Check the payload in the browser's network inspector. It should contain positions, never the original stack string or message. Stack formatting varies, so this parser intentionally accepts a narrow trailing line/column pattern and permits an empty result. Test it against the browsers and emitted bundles you support. Missing positions must remain visible as a coverage gap; inventing a frame would corrupt the evidence.

Next, save this collector as a TypeScript server module and run it with a runtime configured to execute TypeScript and resolve the built-in server modules. It uses in-memory retention to make the example runnable. The buffer holds at most 1,000 accepted events and is lost on process exit. Replace that buffer with an acknowledged durable write before treating acceptance as an incident retention guarantee.

```ts
import { createServer } from "node:http";

const expectedRelease = "market-demo-1";
const expectedEnvironment = "staging";
const expectedOrigin = "http://localhost:3000";
const maxBytes = 4096;
const retained: Array<{ receivedAt: string; event: Record<string, unknown> }> = [];
const totals = new Map<string, { events: number; bytes: number }>();

function record(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null && !Array.isArray(value);
}

function valid(value: unknown): value is Record<string, unknown> {
  if (!record(value)) return false;
  const keys = [
    "release", "environment", "operation", "category", "capturedAt", "frames",
  ];
  if (Object.keys(value).length !== keys.length) return false;
  if (!keys.every((key) => Object.hasOwn(value, key))) return false;
  if (value.release !== expectedRelease) return false;
  if (value.environment !== expectedEnvironment) return false;
  if (!["checkout", "listing", "unknown"].includes(String(value.operation))) {
    return false;
  }
  if (!["runtime", "rejection"].includes(String(value.category))) return false;
  if (typeof value.capturedAt !== "string") return false;
  const timestamp = Date.parse(value.capturedAt);
  if (!Number.isFinite(timestamp)) return false;
  if (new Date(timestamp).toISOString() !== value.capturedAt) return false;
  if (!Array.isArray(value.frames) || value.frames.length > 8) return false;
  return value.frames.every((frame: unknown) => {
    if (!record(frame) || Object.keys(frame).length !== 2) return false;
    return ["line", "column"].every((key) =>
      typeof frame[key] === "number" &&
      Number.isSafeInteger(frame[key]) && frame[key] > 0
    );
  });
}

createServer(async (req, res) => {
  if (req.method !== "POST" || req.url !== "/client-errors") {
    res.writeHead(404).end();
    return;
  }
  if (req.headers.origin !== expectedOrigin) {
    res.writeHead(403).end();
    return;
  }
  if (req.headers["content-type"] !== "application/json") {
    res.writeHead(415).end();
    return;
  }
  const chunks: Buffer[] = [];
  let bytes = 0;
  try {
    for await (const chunk of req) {
      const buffer = Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk);
      bytes += buffer.length;
      if (bytes > maxBytes) {
        res.writeHead(413, { Connection: "close" }).end();
        return;
      }
      chunks.push(buffer);
    }
    const event: unknown = JSON.parse(Buffer.concat(chunks).toString("utf8"));
    if (!valid(event)) {
      res.writeHead(400).end();
      return;
    }
    retained.push({ receivedAt: new Date().toISOString(), event });
    if (retained.length > 1000) retained.shift();
    const key = String(event.operation);
    const total = totals.get(key) ?? { events: 0, bytes: 0 };
    totals.set(key, { events: total.events + 1, bytes: total.bytes + bytes });
    res.writeHead(202).end();
  } catch {
    if (!res.headersSent) res.writeHead(400).end();
  }
}).listen(4319, "127.0.0.1");
```

This collector rejects unknown fields rather than quietly storing them. That is intentional: adding `email` to the frontend object must fail the contract. The origin check constrains ordinary browser submissions; it does not authenticate a caller, since a non-browser client can supply that header. Put ingress rate limits, request timeouts, and concurrency bounds at the deployment boundary before exposing intake publicly. A 4,096-byte body limit alone does not prevent request floods.

Finally, exercise acceptance and rejection from a separate TypeScript test module. This is a synthetic fixture, not a customer incident or a performance result. It targets the local collector directly and supplies the origin header that the browser would set.

```ts
import assert from "node:assert/strict";

const fixture = {
  release: "market-demo-1",
  environment: "staging",
  operation: "checkout",
  category: "runtime",
  capturedAt: new Date().toISOString(),
  frames: [{ line: 42, column: 7 }],
};

async function send(body: unknown): Promise<number> {
  const response = await fetch("http://127.0.0.1:4319/client-errors", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Origin: "http://localhost:3000",
    },
    body: JSON.stringify(body),
  });
  await response.arrayBuffer();
  return response.status;
}

assert.equal(await send(fixture), 202);
assert.equal(await send({ ...fixture, email: "synthetic@example.invalid" }), 400);
assert.equal(await send({ ...fixture, release: "unrecognized-build" }), 400);
assert.equal(await send({ ...fixture, frames: [{ line: -1, column: 7 }] }), 400);
assert.equal(await send({ padding: "x".repeat(5000) }), 413);
```

The meaningful next check is an actual browser failure from the deployed build. Confirm that its release matches the retained artifact, that the positions locate the intended fixture, and that a collector outage doesn't trigger recursive reporting. This implementation returns a failed delivery result and does not automatically retry. Adding retries needs an explicit bound and duplicate handling; otherwise an incident can amplify its own telemetry traffic.

## 4. When are aggregate counts the better choice?

Use the runner-up, aggregate counts, when the question is limited to failure volume by marketplace operation and individual stack evidence is unnecessary.

It gives up event reconstruction.

That can be the correct trade if an existing, separately governed incident channel already retains the evidence you need.

For the reconstruction job, close the loop before expanding capture. Define who can read retained events, when they expire, and how deployment artifacts expire with them. Count rejected payloads without logging their raw contents, and audit proxy access logs separately: omitting customer data from JSON does not control every surrounding log. Keep cumulative accepted bytes separate from current retained bytes, especially when a bounded buffer evicts old events.

The decision rule is concrete: retain a field only if it improves incident reconstruction or supports the chosen operation-level accounting. Measure its cost on the same synthetic fixture. A larger payload needs evidence of value, not another configuration switch.

## References

The browser references describe the event surfaces and stack caveats used above. The metrics reference supports the distinction between aggregates and event evidence; the appender reference provides background on the separate responsibility of writing events to an output destination.

- MDN, Window error event.
- MDN, Window unhandledrejection event.
- MDN, Error stack.
- OpenTelemetry, Metrics concepts.
- Logback manual, Appenders.

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/API/Window/error_event
- https://developer.mozilla.org/en-US/docs/Web/API/Window/unhandledrejection_event
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/stack
- https://opentelemetry.io/docs/concepts/signals/metrics/
- https://logback.qos.ch/manual/appenders.html
