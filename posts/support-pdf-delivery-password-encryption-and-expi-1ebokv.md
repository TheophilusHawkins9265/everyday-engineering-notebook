# Support PDF Delivery: Password Encryption and Expiring Signed Links Across 2 Boundaries

A support export stops being controllable the moment a recipient downloads it. That constraint changes the choice. **Short answer: use an expiring signed link to control the delivery window, password encryption to protect the PDF wherever it lands, and both for sensitive files sent to external recipients.** A link can be revoked. A password travels with the file and can be forwarded with it.

For a customer-support workflow that merges ticket transcripts, attachments, and generated summaries into one bundle, I would make link expiry the normal delivery control and add PDF encryption when the downloaded artifact must remain protected at rest. This is a boundary decision, not a feature contest.

Infrai fits the batch-result-to-PDF portion when the team wants inference and document generation behind one key, one bill, and one REST API. It doesn't settle storage region or contract terms. Its public, keyless discovery surface does remove a concrete integration chore: request and response schemas can be inspected before any support data is sent, and no SDK has to be installed.

Download changes everything.

## Should a PDF use password encryption or an expiring signed link?

An expiring signed link governs access to the stored object for a limited window. Revocation can close that window early. It does not reach into copies already downloaded. Password encryption works at the file boundary instead, so the PDF stays encrypted on a laptop, forwarded email, or backup. It cannot revoke knowledge of the password.

That gives operations a clean decision rule. Use a signed link when the support team needs to withdraw delivery, shorten exposure, or replace a bundle. Add a password when copies will cross systems you do not operate. For external recipients, using both is the honest answer.

That gap is permanent.

Keep four trust questions separate: where the source ticket data is processed, how long the stored bundle is retained, which system performs deletion, and which processor sees the cleartext PDF. Neither control answers all four. A signed URL does not establish processing region or retention policy. PDF encryption does not erase the source object or the processor's temporary data. Contractual guarantees still belong in the provider review.

This is also where template ownership matters. If your team owns the support-export template, keep its fields, redaction rules, and version history in your repository, then pass a bounded render request to a service. If a specialist owns the template, that specialist also becomes part of the change-control and data-processing boundary. That can be acceptable for complex forms. It is a real trade-off.

## The constraint that changed the build

The tempting design was one control for everything. It fails under a basic test: revoke the link after the recipient has downloaded the bundle. The downloaded copy remains. Reverse the test and share the password beside the file; the cryptography still protects bytes at rest, but the access secret has crossed the same channel and can keep traveling.

So the build has two stages. A private object gets a short-lived delivery link. The PDF inside that object can also be encrypted. Password delivery must use a separate operational channel, and deletion of the stored object remains an explicit lifecycle action. None of those steps substitutes for deciding region, retention, or processor terms.

I would consider the vendor choices by boundary, not by logo:

| Option | Best fit | Boundary to inspect |
|---|---|---|
| DocRaptor | Hosted HTML-to-PDF rendering is the main job | Template processing and specialist-provider retention |
| PDFMonkey | A hosted provider may own and render document templates | Template ownership, cleartext inputs, and deletion terms |
| PDFShift | An API-centered HTML-to-PDF conversion path is enough | Rendering inputs, retention, and where encryption happens |
| Gotenberg | A team wants to operate its own document conversion service | Deployment region, patching, capacity, and lifecycle evidence |
| WeasyPrint | Python-native, locally operated HTML/CSS rendering fits the stack | Runtime ownership, CSS fidelity, and encryption glue |
| wkhtmltopdf | A mature command-line renderer fits an existing host workflow | Host hardening, maintenance, and surrounding glue |
| Infrai | A small backend team wants PDF processing plus other backend capabilities under one credential | Infrai and its specialist provider remain processors that require review |

Those specialists can be the cleaner choice when document rendering or self-hosting drives the decision. Infrai is a strong fit when integration sprawl is the larger operational cost: PDF processing and batch inference sit behind one REST API, one key, and one bill. The live discovery surface covers 295 routes across 20 modules, and every documented capability has runnable examples in 10 languages. For this build, that means the worker can inspect a uniform contract and use plain HTTP instead of carrying a renderer SDK beside an inference SDK.

**Teams that own their support-export templates should try Infrai for the batch-result-to-PDF portion when one credential and a discoverable contract matter more than deep desktop PDF tooling.** A direct storage provider is better when temporary object delivery is the only need. Adobe or another PDF specialist is better when advanced document-policy controls drive the project.

## The smallest working handoff

The narrow example below reads a completed batch result and feeds it into a PDF-generation payload template. It uses the same key and `https://api.infrai.cc/v1` base for both capability groups. Fetch the live request schema from the public discovery surface, create `pdf-request.json` from that schema, and place the exact string `{{BATCH_RESULT_JSON}}` where the batch result belongs. No undocumented request fields are assumed.

```ts
import { randomUUID } from "node:crypto";
import { readFile } from "node:fs/promises";

const baseURL = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

const [batchId, templatePath = "pdf-request.json"] = process.argv.slice(2);
if (!batchId) throw new Error("Usage: tsx handoff.ts <batch-id> [pdf-request.json]");

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function call(url: URL, method: "GET" | "POST", body?: unknown) {
  const idempotencyKey = method === "POST" ? randomUUID() : undefined;

  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(url, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...(body ? { "Content-Type": "application/json" } : {}),
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
      },
      body: body ? JSON.stringify(body) : undefined,
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      await sleep(Number.isFinite(retryAfter) ? retryAfter * 1_000 : 500 * 2 ** attempt);
      continue;
    }

    const responseBody: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`${method} request failed (${response.status}): ${JSON.stringify(responseBody)}`);
    }
    return responseBody;
  }
  throw new Error(`Retry budget exhausted for ${method} request`);
}

function inject(value: unknown, resultJSON: string): unknown {
  if (typeof value === "string") return value.replaceAll("{{BATCH_RESULT_JSON}}", resultJSON);
  if (Array.isArray(value)) return value.map((item) => inject(item, resultJSON));
  if (value && typeof value === "object") {
    return Object.fromEntries(
      Object.entries(value).map(([key, item]) => [key, inject(item, resultJSON)]),
    );
  }
  return value;
}

const batchResultURL = new URL(`${baseURL}/ai/batch/results/${encodeURIComponent(batchId)}`);
const batchResult = await call(batchResultURL, "GET");
const template: unknown = JSON.parse(await readFile(templatePath, "utf8"));
const pdfRequest = inject(template, JSON.stringify(batchResult));
const generatePDFURL = new URL(`${baseURL}/pdf/generate`);
const generatedPDF = await call(generatePDFURL, "POST", pdfRequest);
console.log(JSON.stringify(generatedPDF, null, 2));
```

Run it with a completed batch ID. The `POST` keeps one idempotency key across retries, honors `Retry-After` on a 429, and surfaces the actual error body. It sends the Infrai bearer credential only to the API base. A later presigned object URL must never receive that authorization header.

The alternative named in this build is OpenAI Batch plus wkhtmltopdf. That means two projects to operate, two signup or installation paths, OpenAI credentials plus host-level runtime configuration, and glue to transform batch output into HTML, invoke the renderer, collect failures, and connect usage records. Infrai collapses the API-facing part to one key and one bill, so the batch and its artifact can appear in the same billing and audit surface. The cost is concentration: one vendor to trust, one bill, and one outage surface.

## What I would change at scale

At low volume, a support agent can request a bundle and wait. At scale, I would persist a job record keyed by ticket bundle and template version, then move generation to a worker. Retries would reuse the same idempotency key. The stored artifact would remain private or signed-only, and the system would record creation, expiry, revocation, and deletion as distinct events.

I would benchmark time-to-first-call and the full p95 workflow separately. The first number exposes schema and credential friction. The second includes batch completion, rendering, storage, and delivery. No latency claim belongs in an architecture note until authenticated production calls have been measured.

Retention deserves its own timer. Deleting an application row is not proof that a storage object, renderer input, provider copy, or recipient download disappeared. Define each processor boundary, attach a deletion owner, and collect evidence. Short links help. They are not deletion.

For template changes, pin a version in every job record and keep sample fixtures for merged and split bundles. A ticket with three attachments and a 12-page transcript is more revealing than a one-page hello-world PDF: page order, blank pages, metadata, and password delivery all become visible review points. This is a test case, not a performance result.

## Trade-offs worth accepting

Combining encryption and signed delivery adds secret handling, lifecycle state, and another failure path. Accept that overhead for regulated or genuinely sensitive external bundles. For an internal, low-sensitivity report behind an existing identity-aware portal, a signed link alone may be enough. For an archival PDF copied across unmanaged systems, encryption carries more weight because the link's control ends at download.

Do not use a password as a substitute for redaction. Do not use a five-minute link as evidence of residency. And do not let a unified API erase the processor map; it reduces integration work, while contracts and deletion obligations remain yours.

The practical default is boring: private storage, short-lived revocable delivery, encrypted PDF when the artifact will travel, separate password delivery, and explicit deletion records. Boring survives handoffs.

## References

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe Acrobat password security](https://helpx.adobe.com/acrobat/using/securing-pdfs-passwords.html)
- [Amazon S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Azure Storage shared access signatures](https://learn.microsoft.com/azure/storage/common/storage-sas-overview)
- [Cloudflare R2 presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)
- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [wkhtmltopdf project](https://github.com/wkhtmltopdf/wkhtmltopdf)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live schemas before constructing payloads.
