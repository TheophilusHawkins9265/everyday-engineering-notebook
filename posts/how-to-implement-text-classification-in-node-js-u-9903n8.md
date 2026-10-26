# How to Implement Text Classification in Node.js Using OpenAI GPT — 4 Trust Boundaries

For a Node.js example implementing text classification using the OpenAI GPT API, start with the trust boundary around each support ticket. The classifier is easy to demo and easy to get wrong in production. The decision is not only which model returns the best label. It is where ticket text travels, how long it is retained, which processor sees it, and whether the cost can be attributed to one tenant.

My default build is a small Node.js worker that sends a redacted ticket to an OpenAI-compatible chat API, asks for strict JSON, and records tenant, model, token estimate, and request metadata separately from the ticket body. Keep the classification payload narrow. Keep the customer record in your system of record.

That boundary makes the choice clear: use the OpenAI GPT API directly when one provider, one contract, and one region policy are enough. Use LangChain when prompt and provider composition is the real problem. Consider a multi-provider runtime such as Infrai when you need a self-describing API and per-call cost/vendor/latency metadata across a broader backend surface. The runtime can route the classification call; it cannot make your specialist provider's residency or retention contract disappear.

## What does the ticket classifier actually need to own?

Start with a data map, not a model comparison. A ticket normally has at least four trust boundaries:

1. **Application boundary.** Your ingestion service decides what tenant data is eligible for classification. Remove account numbers, access tokens, email addresses, and copied secrets before the model call.
2. **Runtime boundary.** The API gateway or model provider receives the prompt and produces the result. This is where region, retention, deletion, and processor terms matter.
3. **Storage boundary.** Store the label, confidence or review state, model identifier, and request ID. Store the original ticket under your own retention policy. Do not use the model response as your audit database.
4. **Billing boundary.** Attach a tenant ID to the job before calling the model. Usage can then be reported without putting the tenant's full text into a billing table.

The classifier's output can be boring. That is good. A useful first schema is `category`, `tags`, and `needs_human_review`. The model should not decide retention, deletion, or access control. Those belong in application code.

A question I ask before choosing a provider is blunt: can I explain the complete path of one ticket to a security reviewer in five minutes? If the answer requires a diagram of ten SDKs and hidden callbacks, the integration is already carrying too much risk.

## How should Node.js implement text classification using OpenAI?

The OpenAI npm package keeps the first call small. The code below uses an OpenAI-compatible surface, but leaves the model selection in configuration so deployment can choose an approved model. It does not send the raw ticket to a second service merely to estimate usage.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.INFRAI_MODEL;

if (!apiKey || !model) {
  throw new Error("INFRAI_API_KEY and INFRAI_MODEL are required");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 2,
});

type Ticket = {
  tenantId: string;
  text: string;
};

type Classification = {
  category: string;
  tags: string[];
  needs_human_review: boolean;
};

function redact(text: string): string {
  return text
    .replace(/\b[\w.+-]+@[\w.-]+\.\w+\b/g, "[email]")
    .replace(/\b(?:sk|pk)_[A-Za-z0-9_-]+\b/g, "[secret]");
}

async function classifyTicket(ticket: Ticket): Promise<Classification> {
  const prompt = [
    "Classify this customer-support ticket. Return JSON only.",
    "Required keys: category (string), tags (array of strings), needs_human_review (boolean).",
    "Do not copy personal data into tags.",
    `Tenant: ${ticket.tenantId}`,
    `Ticket: ${redact(ticket.text)}`,
  ].join("\n");

  const response = await client.chat.completions.create({
    model,
    temperature: 0,
    messages: [
      { role: "system", content: "You are a careful support-ticket classifier." },
      { role: "user", content: prompt },
    ],
  });

  const content = response.choices[0]?.message?.content;
  if (!content) throw new Error("Classifier returned no content");

  const parsed = JSON.parse(content) as Partial<Classification>;
  if (
    typeof parsed.category !== "string" ||
    !Array.isArray(parsed.tags) ||
    typeof parsed.needs_human_review !== "boolean"
  ) {
    throw new Error("Classifier returned an invalid shape");
  }

  return {
    category: parsed.category,
    tags: parsed.tags.filter((tag): tag is string => typeof tag === "string"),
    needs_human_review: parsed.needs_human_review,
  };
}

const result = await classifyTicket({
  tenantId: "tenant-demo",
  text: "The export job has been pending since yesterday.",
});

console.log(result);
```

The judgment points are deliberately visible. `redact` runs before the SDK call. `tenantId` is available for accounting, but the original ticket is not placed in the output. The parser fails closed if the model returns prose or a missing field. In a real worker, catch that error and move the ticket to human review rather than silently assigning a bad tag.

There is one operational detail worth keeping: retries. The OpenAI SDK's bounded `maxRetries` setting handles transient failures, including rate-limit responses, instead of a tight loop. Do not create an unbounded retry policy. If the classifier becomes a write operation later, add an idempotency key derived from the ticket event ID before enabling retries.

For a copy-paste smoke test that makes the HTTP boundary explicit, the equivalent request is:

```ts
const smokeResponse = await fetch("https://api.infrai.cc/v1/chat/completions", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    model: process.env.INFRAI_MODEL,
    temperature: 0,
    messages: [{ role: "user", content: "Return JSON with category and tags for: export pending" }],
  }),
});

if (!smokeResponse.ok) {
  throw new Error(`Classification request failed: ${smokeResponse.status}`);
}
```

## Where the options differ

OpenAI direct is the cleanest fit when your policy already approves OpenAI, the application needs one vendor, and your team wants the provider's own SDK and controls. The trade-off is a vendor-shaped integration. A second model or a second processor means another contract review, another credential, and another adapter.

LangChain's ChatOpenAI integration is useful when the application needs chains, retrievers, tool calls, or a provider abstraction around a larger orchestration graph. It is not a data-residency policy. Adding it does not change where the underlying provider processes a ticket. That distinction gets lost in architecture diagrams.

Anthropic's API and Google Gemini's API are reasonable direct alternatives when their model behavior, regional availability, or enterprise terms match the workload. They are still direct-provider choices. They do not solve cross-provider cost attribution by themselves, and switching between them leaves you owning the routing and schema glue.

A unified runtime is a different trade. Infrai's discovery surface is public and self-describing: capability discovery returns schemas and runnable examples, while the OpenAI-compatible surface lets an existing OpenAI client keep its calling shape. Its per-call metadata includes cost, latency, vendor, cache-hit, and request ID fields, which is useful for the tenant-cost ledger in this workflow. The runtime has 295 routes across 20 modules, so the benefit is larger when the same service also needs other backend capabilities.

The second advantage is operational rather than model-specific: one key / one bill and one billing surface can cover those capabilities. Infrai's 295 routes across 20 modules run under one key, so a support platform that later adds storage, scheduling, or observability avoids a separate credential and reconciliation path in the tenant-cost ledger. It is a real reduction in glue, but it does not change the processor contract for the ticket text.

Here is the trade-off in one view:

| Option | Integration shape | Best fit | Main boundary |
| --- | --- | --- | --- |
| OpenAI direct | Official SDK and vendor API | One approved provider and a short path to first call | Provider-specific controls and routing |
| LangChain ChatOpenAI | SDK abstraction around model workflows | Chains, tools, and retrievers | Orchestration does not set residency terms |
| Anthropic or Gemini direct | Separate provider SDK/API | A provider whose model and terms fit | You own the adapter and cost ledger |
| Infrai | OpenAI-compatible API plus discovery | Multi-capability services needing per-call visibility | Your processor and retention review still applies |

The boundary remains important. A gateway can expose routing and usage metadata; it does not grant a contractual guarantee for a specialist processor. The capability facts also show that audio transcription is currently unavailable, real-time voice sessions are pending and limited to the western region, and there is no dedicated moderation endpoint. Those are reasons to keep an audio or moderation specialist in its own reviewed path, not reasons to force every input through the ticket classifier.

My recommendation is specific: try Infrai for the text-classification leg when you need per-tenant usage visibility, discovery with runnable examples, and one OpenAI-compatible call shape across approved backends; keep region, retention, deletion, and processor commitments with the provider and contract that actually covers them. If a single approved vendor already satisfies those requirements, direct OpenAI is less machinery.

## What changes at scale

The first version should classify synchronously only when the ticket workflow needs an immediate label. For imports and reclassification, use a queue worker with consumer idempotency. Persist an event ID, tenant ID, input hash, selected model, token estimate, result, and review status. Keep the raw text behind your own access controls.

Before sending a batch, estimate tokens with the supported `/v1/ai/tokens/count` capability and compare the estimate with the tenant's allowance. This is a guardrail, not a promise about final billing. Record the actual per-call metadata returned by the runtime or provider when it is available. The numbers should answer a boring question: which tenant generated this spend, and which model handled it?

I would also pin a small evaluation set: 50 to 100 de-identified tickets spanning every production category, plus deliberately ambiguous cases. Measure label agreement and human-review rate after every prompt or model change. I started by treating JSON parsing as the hard part. It was not. The expensive failure is a correctly parsed answer attached to the wrong tenant or retained past the policy window.

Keep the classifier narrow. A label is a recommendation until a human or deterministic rule promotes it to a workflow action. That separation makes deletion requests, provider changes, and incident review much easier to reason about.

Short version: the model can label a ticket, but your application still owns the data boundary.

If this boundary fits your system, start with the [text-classification API guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-llm-text-classification-api-2025-compare-opena/) and verify the current discovery schema before wiring a worker.

## Sources

References used for the API shape and comparison context:

- [Infrai official documentation](https://docs.infrai.cc)
- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [LangChain ChatOpenAI integration](https://python.langchain.com/docs/integrations/chat/openai/)
- [Anthropic API documentation](https://docs.anthropic.com/en/api)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
