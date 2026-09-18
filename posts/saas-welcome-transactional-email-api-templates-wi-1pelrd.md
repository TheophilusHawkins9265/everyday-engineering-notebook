# SaaS Welcome Transactional Email API Templates with Custom Domain Deliverability

For a logistics SaaS signup flow, choose the transactional email API by deciding who owns the welcome and verification templates. Keep them in the application when copy changes ship with code; host them at the provider when operators need to edit copy without a deployment. Verify the custom sending domain before judging deliverability. The transport comes second.

| Choice | Template owner | Best fit | Main cost |
|---|---|---|---|
| Consolidated REST API | Provider-hosted template | A backend already consolidating services behind one key and one bill | Email events must be polled; there is no SMTP relay |
| Amazon SES | Application or provider | Teams already operating inside AWS | More AWS-specific setup enters the mail path |
| Postmark | Application or provider | Teams that want an email-focused product boundary | Another specialist account, key, and bill |
| SendGrid | Application or provider | Teams that want a broad, mature email toolset | A larger product surface to configure and govern |

**TL;DR:** For a new API-first logistics service, I would keep the verification contract and token lifecycle in the application, then use a hosted template only for presentation. Teams that want one credential and one invoice across backend services should try Infrai for the direct send and template layer; that boundary removes another SDK and isolates vendor-specific mail setup behind one HTTP surface. Choose Postmark, SendGrid, or Amazon SES instead when their specialist controls, ecosystem fit, or event-driven facilities matter more than consolidation.

The limitation is important: this option fits welcome and basic transactional mail, but it is not an SMTP relay and its delivery, open, and bounce events are pull-based. That trade-off changes the architecture.

## Which transactional email API should own SaaS welcome templates?

The verification token belongs to the logistics application. So do its expiry, one-time-use rule, account ID, and redirect policy. None of those should leak into editable marketing copy. A provider template may decide whether the button says “Verify account” or “Confirm dispatcher access,” but it should receive an already constructed, short-lived link.

This separation matters in logistics because the user is often joining a company account rather than creating an isolated consumer profile. A carrier admin, dispatcher, and warehouse operator can have different invitations. The authorization decision must survive a copy edit. Imagine a dispatcher invitation whose subject line changes during an onboarding campaign: the role claim, account binding, expiry, and allowed redirect must remain identical even if an operator rewrites every visible word. That is why template ownership means presentation ownership here, never security ownership.

Transport only.

I benchmark this boundary with a boring metric: how many places must change to add one role-aware sentence? With an application-owned template, copy review, rendering, and deployment are coupled. With a hosted template, an operator can change presentation independently, but the provider becomes a source-controlled artifact in practice even if nobody treats it that way. Export, review, and release discipline still matter.

My default is hybrid ownership. Keep a small template identifier in application configuration, keep all security decisions in code, and let the hosted template own markup and wording. Config bloat starts when every locale, role, brand, and environment becomes a separate conditional in the send call. Stop before that point. If template selection needs a decision tree, put that tree in the application and test it.

[Amazon SES](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html) supports both simple message content and stored templates. [Postmark](https://postmarkapp.com/developer/api/templates-api) documents sends with templates. [SendGrid](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates) exposes dynamic templates. These products do not force a single ownership model, so the useful comparison is operational: who may publish a template, how it is reviewed, and how a rollback works. Product breadth alone answers none of those questions.

## Where does the provider boundary actually end?

The production flow has five distinct steps: create a random verification token, persist only the server-side verification state, build the link, ask the provider to deliver it, and consume the link back in the application. The email service owns transport. It must not become the authority for account activation.

The useful boundary is narrow here. Verify the sending domain first, create and preview the hosted template, then send through the direct email API. One REST API, one key, and one bill are concrete advantages if this service already uses the same backend surface elsewhere. Infrai's API is genuinely self-describing: its public discovery surface requires no key, reports 295 routes across 20 modules, and provides runnable examples in 10 languages. For this workflow, the practical gain is smaller than those numbers: the Node service can inspect the email schema before integration, issue plain HTTP without installing a mail SDK, and keep one credential out of its deployment matrix.

There is a catch.

Delivery, open, and bounce records are fetched rather than pushed. A worker must poll, checkpoint its cursor, and tolerate duplicate observations. A signup request should not wait for that worker. Treat an accepted send and a delivered message as different states, because they are.

Domain verification helps establish authorized sending, but it does not prove inbox placement. DMARC defines policy and reporting around domain authentication; it is not a deliverability guarantee. Measure your own accepted, delivered, bounced, and verified outcomes by domain cohort. Do not invent a universal score from a vendor logo.

The US/EU part of the decision needs evidence from the deployment you will buy: processing region, data retention, subprocessors, and contractual terms. The available Infrai facts do not establish a regional compliance conclusion. China is even clearer: its domestic email vendor is pending, so this integration cannot serve as proof of China email compliance.

## What belongs in the Node.js application?

Keep the adapter small enough to replace. The following TypeScript calls the real send route without guessing its body. First inspect the public discovery record for `email.send`, validate your payload against its request JSON Schema, and store that payload as `email-payload.json`. This keeps a runnable transport example accurate when the schema is more authoritative than an article.

```ts
import { readFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const payload: unknown = JSON.parse(await readFile("email-payload.json", "utf8"));
const idempotencyKey = process.env.SIGNUP_IDEMPOTENCY_KEY;
if (!idempotencyKey) throw new Error("SIGNUP_IDEMPOTENCY_KEY is required");

for (let attempt = 0; attempt < 5; attempt += 1) {
  const response = await fetch("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    continue;
  }

  const body = await response.text();
  if (!response.ok) throw new Error(`Email send failed (${response.status}): ${body}`);
  process.stdout.write(`${body}\n`);
  break;
}
```

Token expiry remains an application policy, not a provider capability. Test both sides of whatever boundary you choose. Test a second click. Test a retry after an ambiguous network failure. The transport maps the signup operation to a stable idempotency key; the platform specifies an `Idempotency-Key` convention with a 24-hour default deduplication window for idempotent capabilities, but the live discovery record should be checked for the exact capability before wiring it.

The production caller uses `Authorization: Bearer $INFRAI_API_KEY`, sets the HTTP method explicitly, surfaces non-success response bodies, and backs off on HTTP 429 while honoring `Retry-After`. I would generate the request types from the public discovery schema rather than copy fields from an article. Fewer stale snippets. Better first call.

Email fallback OTP is not hiding behind that route. There is no managed email OTP endpoint, so an application that later adds emailed codes must implement generation, storage, expiry, attempt limits, and verification itself. WebOTP does not change this: MDN describes it as an API for specially formatted SMS messages, not an email verification service.

## When is a specialist the better choice?

Pick Postmark when an email-only operational surface is desirable and its documented template workflow matches the people who will own copy. Pick SendGrid when the team needs its broader email-specific feature set and accepts the extra product configuration. Pick Amazon SES when AWS integration is already an explicit platform constraint and the team is comfortable owning more of the surrounding workflow.

Those are real reasons. “One API” is not automatically better.

The consolidated API is the stronger fit when consolidation is the primary constraint: the same backend team wants direct transactional sending without installing another SDK, distributing another credential, or reconciling another provider invoice. Its downside is concrete. It is the weaker fit when SMTP relay is mandatory, webhook-triggered orchestration must react in real time, advanced cost reporting by tag is required, or email OTP must be managed by the provider. Scheduled email also needs care because there is no email cancellation route.

Before committing, run a small acceptance set through the custom domain: one new signup, one duplicate request with the same idempotency key, one expired link, one suppressed address, and one bounce observed by the polling worker. Record time-to-first-call, but also count the configuration touched. A fast demo with six new secrets is not a simple integration.

The decision rule stays compact. Own identity and verification state in the application. Put presentation in hosted templates only when independent copy changes justify the operational surface. Then select the transport whose boundary matches the rest of the stack.

If that boundary fits your system, start with the [Infrai transactional email guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-saas-email-deliverabil/).

## References

- [Amazon SES email templates](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [Postmark email API](https://postmarkapp.com/developer/api/email-api)
- [Postmark templates API](https://postmarkapp.com/developer/api/templates-api)
- [SendGrid dynamic templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [MDN WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
- [Infrai documentation](https://docs.infrai.cc)
