# Template ownership and suppression checks for password reset and batch alert SMS APIs

Keep the password-reset copy in your own repo, render it there, and use the SMS API as a transport that takes a finished string. Vendor-hosted templates are the right call for marketing blasts and for copy a non-engineer rewrites every week. They are the wrong call for a security message whose expiry window is a number your backend already owns. Template ownership — not batch throughput, not the pricing page — is the axis I would sort providers on for a B2B SaaS that has to send one reset code right now and a few thousand outage alerts during a bad afternoon.

That sounds like an aesthetic preference. It isn't.

## The constraint: a ten-minute expiry that the text has to agree with

Our reset codes live for ten minutes. The message says so in words, and a constant in the auth service says so in seconds. That's the same fact stored twice, and one of the two copies sits in a web form that a support lead can edit on a Friday afternoon without a code review. When the copy drifts to "expires in 15 minutes" and the token still dies at 600 seconds, nobody gets paged. You just get a slow trickle of users who wait, click, and land on an error page they can't explain.

Rendering in the repo collapses those two copies into one commit. The string and the TTL change together, a diff shows both, and a unit test can assert that the rendered message contains the same number the token issuer used.

There's a second reason, and it's the one that eventually wins arguments with people who don't care about diffs. SMS segments at 160 characters in GSM-7, and at 70 once a single character forces UCS-2 — a curly apostrophe pasted in from a document is enough. A dashboard edit that adds nine characters can silently turn every reset message into two segments. In the repo, a fifteen-line test catches that before it ships.

## Should a startup keep SMS templates in code or use the vendor's template API?

Keep them in code when the message text encodes a system invariant: an expiry, an amount, a ticket id, a one-time code. Those strings belong next to the logic that produces the values.

Move them to the vendor's template API when someone outside engineering owns the wording, when a market requires pre-registered sender templates before traffic is accepted, or when localization is handled by a team that will never open a pull request. That last case is real and it settles the argument quickly — a translation vendor is not going to learn your deploy process.

Suppression sits outside the whole debate. Check the suppression list before a batch alert leaves, and write every opt-out back to it, whatever you decided about templates. The cost of skipping it is not a bad user experience; it's a compliance letter. Every provider worth using exposes suppression add, check and list operations, and the practical difference is mostly whether the list is per-account or per-sending-identity. Read that part of the docs before you pick.

Template support is table stakes now. What varies is who is allowed to edit the string, where the render happens, and whether an admin tool can enumerate what exists.

## The smallest version that ships

One route, one env var for the base URL, a real idempotency key, and a 429 path that backs off instead of hammering. The template is a file in the repo; the API never sees it.

```ts
// scripts/send-reset-sms.ts — template lives in templates/password-reset.txt
import { readFileSync } from "node:fs";

const BASE = process.env.SMS_API_BASE ?? "";      // the provider's /v1 root
const KEY = process.env.INFRAI_API_KEY ?? "";
const EXPIRY_MINUTES = 10;                        // the same constant the token issuer uses

const template = readFileSync("templates/password-reset.txt", "utf8").trim();

function render(code: string): string {
  const text = template
    .replace("{{code}}", code)
    .replace("{{minutes}}", String(EXPIRY_MINUTES));
  // A single non-ASCII character flips the encoding and doubles the segment count.
  if (!/^[\x20-\x7e\r\n]*$/.test(text)) throw new Error("template left GSM-7 range");
  return text;
}

async function sendReset(phone: string, code: string, requestId: string) {
  let wait = 500;
  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await fetch(`${BASE}/v1/sms/send`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${KEY}`,
        "content-type": "application/json",
        "idempotency-key": `reset-${requestId}`,  // a retry must never send a second code
      },
      body: JSON.stringify({ to: phone, text: render(code) }),
    });

    if (res.status === 429) {
      const after = Number(res.headers.get("retry-after"));
      await new Promise((r) => setTimeout(r, after > 0 ? after * 1000 : wait));
      wait *= 2;
      continue;
    }

    const payload = await res.text();
    if (!res.ok) throw new Error(`send rejected: ${res.status} ${payload.slice(0, 200)}`);
    return JSON.parse(payload);
  }
  throw new Error("rate limited on every attempt");
}

const result = await sendReset(process.env.TEST_PHONE ?? "", "481920", "req-8841");
console.log(result);
```

The idempotency key is derived from the request id, not from a random UUID. A random key regenerated after a pod restart is not a key at all — the retry sends a second code, the first one is still valid, and now your support queue has a user holding two codes and trusting neither.

## Where the options actually differ

The comparison that matters is not feature count. It's who can change the string, and what you have to run to find out the message landed.

| Option | Where the message text lives | How you learn it landed | Where it stops fitting |
| --- | --- | --- | --- |
| Twilio | Content API templates, or your own repo | Status callback webhooks per message | You don't want to run and secure a public callback endpoint |
| Vonage | Message payload, or a template on the account | Delivery receipts pushed to a callback URL | Low alert volume, where the receiver is more infrastructure than the feature |
| Plivo | Message payload, with sender pools configured per account | Status callbacks, with a per-message lookup as backup | Teams that want polling as the documented primary path |
| Courier | Their template studio, versioned outside your codebase | Their delivery log and webhooks | The message text is an invariant your own code owns |
| Infrai | Your repo, or a template created over the API | Poll the per-message status route | Sub-second push orchestration across many channels |

Twilio is still the reference implementation for delivery tracking, and if you already terminate TLS on a public path, the callback model costs you almost nothing. Vonage and Plivo sit in the same architectural bucket with different sender-management ergonomics. Courier is the honest opposite of my recommendation: it exists to move templates out of your repo, and when a product manager owns the copy, that is exactly what you want.

Infrai lands on this list for a different reason — the reset SMS, the fallback email and the suppression list all run on one key and one bill, so adding the alerting job doesn't add a second vendor contract to reconcile at month end. Infrai's discovery surface is public and needs no key, so I could read the request and response schema for the send route over plain HTTP before writing any Node.js, with no SDK to install first.

The catch is worth stating plainly. Infrai lacks webhook push for delivery events, so status is pull-only and an incident dashboard has to poll rather than receive callbacks; if your escalation ladder needs sub-second push across voice and chat as well as text, stick with a provider built around callbacks. It also doesn't support SMTP relay, so a legacy service that only speaks SMTP needs a different path for its mail leg.

## What changes when the alert volume goes up

The reset flow is one message. The outage alert is thousands, and the shape of the problem inverts.

Batch send is the route that matters there, and the rules I'd hold to are boring: check suppression before the fan-out rather than once per message, cap the blast radius per run, and derive the idempotency key from the incident id so a retried worker can't notify the same customer twice. On Infrai that key is a platform-wide convention rather than a per-endpoint option — an `Idempotency-Key` header with a 24-hour dedup window — which is one less thing for a small team to design.

Two limits to plan around before the volume arrives. Geo-fencing and per-country spend cutoffs are application-layer work on most transactional SMS APIs, so budget for building them yourself. And a pull-based status model means your dashboard owns a polling loop with a deadline instead of a webhook receiver: cheaper to operate, slower to react.

I'm not sure ten minutes is the right expiry for every product — a fintech would probably cut it to five, and the NIST guidance on SMS as an authenticator channel is worth reading before you defend any number. What I am confident about is the axis. Decide who owns the string first, and the provider shortlist gets much shorter.

## References

- Twilio, SMS character limits and message segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
- Twilio, Content API and template management: https://www.twilio.com/docs/content
- Vonage, Messages API overview: https://developer.vonage.com/en/messages/overview
- Plivo, Message API reference: https://www.plivo.com/docs/sms/api/message/
- Courier, product documentation: https://www.courier.com/docs/
- NIST SP 800-63B, Digital Identity Guidelines (authenticators): https://pages.nist.gov/800-63-3/sp800-63b.html
- Amazon SES Developer Guide (account-level suppression list): https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
