# Debugging json_schema Responses in Node.js: Invalid Text, Parse, and Retry

An LLM cannot extract structured JSON from text reliably if a streamed response is parsed before it is complete. That constraint changes the design: parsing belongs after transport assembly, while validation and retry policy belong behind separate boundaries.

Short answer: in Node.js, buffer one complete LLM response, run `JSON.parse` once, validate the resulting value against the intended `json_schema`, then retry only the failed model operation with a small, explicit budget. Never treat an incomplete stream fragment as invalid JSON, and never let a retry repeat downstream writes.

This sounds fussy. It isn't. “Invalid JSON” is usually one label pasted over several different failures, and a generic retry loop erases the evidence needed to tell them apart.

## How should Node.js parse an LLM structured JSON response?

Start with four boundaries: transport, syntax, schema, and meaning. Each one answers a different question, and each needs a different disposition.

| Stage | Question | Failure disposition |
|---|---|---|
| Transport | Did the complete response arrive? | Reassemble or retry the model operation |
| Syntax | Is the completed text one valid JSON value? | Reject it; optionally request one correction |
| Schema | Does the parsed value match the required shape? | Return field issues; optionally request one correction |
| Meaning | Does the value agree with the source? | Apply domain checks or send it to review |

For streamed output, the first boundary matters most. Server-sent events deliver a sequence of events over an open HTTP connection. Imagine the intended object is `{"city":"Boston","units":"metric"}` but the first observed fragment ends after `"Bos`. `JSON.parse` must reject that fragment; the closing quote and braces do not exist yet. That rejection says nothing about the final response. The next fragment may complete the same event, or another event may carry more data, depending on the adapter's framing. If the application labels the early exception `E_JSON_SYNTAX` and launches another model call, it has converted normal transport behavior into an extraction failure. If it labels the state `E_TRANSPORT_INCOMPLETE`, continues assembly, and invokes the decoder only after the adapter reports completion, the parser gets one coherent value and its result becomes useful evidence. The distinction also changes observability: a partial buffer belongs in transport metrics, while a rejected completed value belongs in syntax metrics. Mixing them inflates the apparent invalid-response rate and sends engineers toward prompt changes when the defect is in stream handling. Assemble the response according to the transport protocol, observe the runtime's completion signal, and only then hand the final string to the decoder.

No shortcuts.

Brace slicing is especially tempting: find the first `{`, find the last `}`, and discard everything outside them. I don't put that in a default extraction path. It can silently choose the wrong object when the source contains examples, nested quotations, or two candidate objects. A hard failure with a small diagnostic is better DX than a plausible object selected by punctuation. If a runtime promises schema-constrained output, enforce that contract at its adapter. If it returns plain text, make cleanup an explicit, tested policy rather than an invisible substring trick.

After syntax comes schema. `JSON.parse` proves only that the text belongs to JSON's grammar; it doesn't prove that `quantity` is an integer, that required keys exist, or that extra keys are forbidden. Then comes semantic validation. A schema-valid extraction can still copy the wrong invoice number or assign a total from the wrong paragraph. Those cases need deterministic domain checks, comparison with the source, or human review. Another parse attempt cannot solve them.

## The smallest implementation worth shipping

The useful abstraction is a decoder with no network, database, queue, or logging dependency. Give it completed text and a validator. It returns a typed value or an error with a stable stage code. That makes every failure reproducible in a unit test and keeps provider-specific response envelopes outside the core.

```ts
type ValidationResult<T> =
  | { ok: true; value: T }
  | { ok: false; issues: readonly string[] };

type Validator<T> = (value: unknown) => ValidationResult<T>;

type DecodeError = {
  code: "E_JSON_SYNTAX" | "E_SCHEMA_SHAPE";
  message: string;
};

type DecodeResult<T> =
  | { ok: true; value: T }
  | { ok: false; error: DecodeError };

export function decodeCompletedJson<T>(
  completedText: string,
  validate: Validator<T>,
): DecodeResult<T> {
  let candidate: unknown;

  try {
    candidate = JSON.parse(completedText);
  } catch (error) {
    const message = error instanceof Error ? error.message : "Unknown JSON error";
    return { ok: false, error: { code: "E_JSON_SYNTAX", message } };
  }

  const result = validate(candidate);
  if (!result.ok) {
    return {
      ok: false,
      error: {
        code: "E_SCHEMA_SHAPE",
        message: result.issues.join("; "),
      },
    };
  }

  return { ok: true, value: result.value };
}
```

The adapter around this function owns transport completion. The application owns semantic checks and writes. That division looks almost too small to name — which is exactly why it survives a provider swap without dragging config through the codebase.

Here is a generic extraction loop. `callModel` is an injected port, not a guessed HTTP route. The loop records the completed response, classifies it locally, and permits one corrective call. It does not insert a row or publish an event.

```ts
type ModelReply = {
  completedText: string;
  completion: "complete" | "output_limit" | "transport_interrupted";
};

type CallModel = (request: {
  sourceText: string;
  correction?: string;
}) => Promise<ModelReply>;

export async function extract<T>(
  sourceText: string,
  callModel: CallModel,
  validate: Validator<T>,
): Promise<T> {
  let correction: string | undefined;

  for (let attempt = 1; attempt <= 2; attempt += 1) {
    const reply = await callModel({ sourceText, correction });

    if (reply.completion !== "complete") {
      correction = `Previous response was incomplete: ${reply.completion}`;
      continue;
    }

    const decoded = decodeCompletedJson(reply.completedText, validate);
    if (decoded.ok) return decoded.value;

    correction = `${decoded.error.code}: ${decoded.error.message}`;
  }

  throw new Error("E_EXTRACTION_EXHAUSTED: two model attempts failed");
}
```

The number two is an example budget in this implementation, not a universal optimum. I'm not sure a universal optimum exists. Measure your own distribution of first-pass success, corrective-pass success, latency, and review cost; then choose a cap. What matters is that the cap is visible and that exhaustion becomes a normal state the caller can route, not an unbounded loop.

## Retry the operation, not the workflow

A retry decision should follow the failure stage. An interrupted transport or output-limit completion can justify another model call because no complete candidate exists. Completed but syntactically invalid output may justify one corrective call that includes the parser's compact error. A schema mismatch may justify one correction with field-level issues. A semantic contradiction should usually leave the automatic lane, because the system already produced well-formed data and repeating the same request gives no proof of a better answer.

Keep the model operation pure from the application's point of view. The extracted value can be written later through an idempotent command whose key comes from stable input identity. This isn't an excuse to retry every write. It is a containment rule: model retries cannot accidentally replay database inserts, webhooks, emails, or queue publications because none of those effects live inside the extraction function.

I use stage codes because stack traces change and dashboards need boring labels. `E_TRANSPORT_INCOMPLETE`, `E_JSON_SYNTAX`, `E_SCHEMA_SHAPE`, `E_SEMANTIC_CONFLICT`, and `E_EXTRACTION_EXHAUSTED` are enough to reveal where the pipeline spends its failure budget. Log the code, attempt number, completion state, latency, schema version, and a hash that correlates the source with a controlled diagnostic record. Don't dump raw documents or model output into general logs by default; extracted text may contain credentials, personal data, or contractual terms.

This is also where benchmarking earns its keep. Test a fixed corpus that includes empty input, long input, Unicode, quoted braces, arrays at the root, missing fields, extra fields, wrong scalar types, and a deliberately interrupted stream. Track syntax-valid rate separately from schema-valid and semantic-acceptance rates. One aggregate “success” metric hides the very boundary the design is supposed to expose.

Measure each boundary.

## What I would change at scale

At higher volume, I would preserve the same decoder and add infrastructure around it: a versioned schema registry, a bounded work queue, redacted diagnostic storage, and a review lane for exhausted or semantically uncertain records. Deploy schema and prompt changes against the fixed corpus before shifting traffic. Keep the old schema reader available long enough to consume work already in flight.

Streaming needs one more deliberate choice. If the UI needs immediate motion, stream human-readable progress separately from the machine object, or show transport status while the structured result is assembled. Parsing partial SSE data as if it were a final object couples presentation latency to data correctness. That coupling is config bloat in disguise: soon every screen owns a slightly different fence stripper, partial parser, and retry timer.

Watch distributions, not anecdotes. Useful signals include attempts per accepted extraction, failures by stage code, output-limit frequency, queue age, review rate, and latency per stage. Avoid publishing a benchmark number without its corpus, schema, model configuration, completion limit, and acceptance test. Otherwise it is decoration.

## Where this pattern is the wrong choice

The catch is that schema-constrained extraction adds latency, validation code, observability, and a review path. It is not suitable when deterministic parsing already describes the input. Stick with a normal JSON parser for a JSON document, a CSV parser for a stable CSV export, or a narrow deterministic rule for a fixed identifier. An LLM is unnecessary surface area in those cases.

It also doesn't remove ambiguity. If the source omits a required value, a strict shape can pressure the pipeline toward a syntactically convenient placeholder. Model the absence explicitly with a nullable field or a tagged result, then make the business rule decide whether absence is acceptable. For high-impact actions, semantic review may be mandatory even when every schema check passes.

Finally, `json_schema` support is an adapter capability, not the architecture. When a runtime can enforce the schema during generation, use that facility at the edge and still validate locally. When it cannot, request one JSON value and apply the same completed-text decoder. The portable asset is the boundary: complete transport, exact parse, explicit validation, bounded retry, then side effects.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://github.com/pgvector/pgvector
