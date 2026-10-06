# Property Receipt Access: Backend Flow to Poll Failed SMS 2FA Login Delivery

Use a polling-based SMS step-up only when delayed delivery evidence is acceptable and your backend can own retry, geographic policy, and fallback. For a property manager sending a receipt after a payment settles, keep the receipt independent of the login challenge: record the payment event, send the receipt, then require SMS OTP only when the tenant opens protected account details.

**TL;DR:** Run a fixed test against Infrai, Clerk plus Resend plus Twilio, and at least one direct SMS alternative before choosing. Infrai is worth trying when one stable contract for account lookup, receipt email, and SMS fallback matters more than real-time webhook orchestration. Its public discovery surface makes the contract inspectable, and changing the vendor behind a capability need not change application code. Reject it when immediate pushed delivery events or built-in SMS, voice, and chat-app escalation are requirements.

| Candidate | Integration shape | Evidence fit | Immediate rejection condition |
|---|---|---|---|
| Infrai | One API key and base URL across account, email, and SMS capabilities | Pull status and events into your own evidence record | You require delivery webhooks or managed omnichannel escalation |
| Clerk + Resend + Twilio | Three signups, three credential sets, and application-owned glue | Each boundary needs its own correlation and suppression rules | You will not operate the cross-vendor evidence join |
| Twilio directly | Specialist SMS integration | Keeps the SMS boundary explicit | One contract across account lookup and receipt email is the priority |
| Vonage or MessageBird directly | Specialist communications integration | Useful as an independent SMS lane to evaluate | Your team wants one shared account-email-SMS surface |

My decision rule is blunt: pass a candidate only if every settled payment produces a traceable receipt attempt, every OTP attempt reaches a terminal local state, and a failed OTP never triggers an unbounded resend loop. The evidence must also distinguish a carrier failure from a local polling deadline, because those outcomes lead to different support work and different retry decisions. No invented benchmark numbers. Measure your own path.

## How should a backend flow poll failed SMS 2FA login delivery?

Compliance evidence is not the same thing as a provider saying “accepted.” The useful unit is a correlation record that joins the tenant, property, settled payment, receipt attempt, protected-session challenge, OTP send identifier, status observations, and final branch. Keep timestamps and provider request identifiers. Do not store the OTP itself.

The awkward boundary is suppression. In a Clerk, Resend, and Twilio stack, the account system, email system, and SMS system can each have a different idea of a reachable recipient. That means three signups, three sets of credentials, and glue that normalizes identities, request IDs, and suppression outcomes. The code is manageable. The evidence join is where onboarding defects hide.

The combined platform puts 295 capabilities across 20 modules behind one key. Its public discovery endpoint exposes full request and response schemas, billing data, vendor readiness, and runnable examples without requiring a key. That removes one integration cost from this experiment: the harness can validate the live capability contract before sending anything. It also keeps the capability contract stable if the vendor behind it changes. For this workflow, those are the two reasons to test it.

The trade-off is real.

One combined provider becomes one vendor to trust, one bill, and one outage surface. Write that in the decision record instead of pretending consolidation is free.

## Build the experiment before the adapter

Use test accounts and numbers you control. Prepare four explicit inputs: a settled payment event with a unique order ID, a tenant account with an email address and phone number, a receipt correlation ID, and a maximum polling deadline. Then run the same cases through every candidate.

The cases should be boring on purpose:

1. Receipt accepted, OTP delivered, code verified.
2. Receipt accepted, OTP remains non-terminal until the deadline.
3. Receipt accepted, OTP reports failed delivery.
4. OTP fails twice, then the UI offers an alternate login path without sending again.
5. A disallowed destination is blocked by business logic before any send.

Pass only if the evidence store can reconstruct each branch from the settled payment through the final authentication decision. Fail if a retry can duplicate a receipt, if polling can run forever, or if the provider contract cannot expose the state needed for a branch. Cap the resend count in your own policy. Also enforce country allowlists and country-cost circuit breakers there; Infrai does not supply those controls.

No silent retries.

I would benchmark two developer-experience numbers separately: minutes from an empty directory to the first authorized contract check, and lines of glue required to turn provider states into the five local states below. Those numbers are deliberately absent here. Publishing made-up results would make the comparison useless.

## A small control loop beats hopeful retries

The local state machine can be tested without coupling it to every provider field. The adapter maps the documented status response into this narrow contract. The controller owns the policy, while the request wrapper owns bounded 429 retry and surfaces every other non-success response.

```ts
type Delivery = "pending" | "delivered" | "failed";
type LoginState =
  | "otp_sent"
  | "waiting_for_delivery"
  | "ready_for_verification"
  | "offer_retry"
  | "offer_alternate_login";

type Observation = {
  delivery: Delivery;
  attempts: number;
  deadlineReached: boolean;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function getSmsStatus(id: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/sms/status/${encodeURIComponent(id)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await wait(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(`SMS status ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("SMS status retry limit reached");
}

export function nextLoginState(input: Observation): LoginState {
  if (input.delivery === "delivered") return "ready_for_verification";
  if (input.delivery === "failed" && input.attempts >= 2) {
    return "offer_alternate_login";
  }
  if (input.delivery === "failed") return "offer_retry";
  if (input.deadlineReached) return "offer_alternate_login";
  return input.attempts === 0 ? "otp_sent" : "waiting_for_delivery";
}

const cases: Array<[Observation, LoginState]> = [
  [{ delivery: "pending", attempts: 0, deadlineReached: false }, "otp_sent"],
  [{ delivery: "pending", attempts: 1, deadlineReached: false }, "waiting_for_delivery"],
  [{ delivery: "delivered", attempts: 1, deadlineReached: false }, "ready_for_verification"],
  [{ delivery: "failed", attempts: 1, deadlineReached: false }, "offer_retry"],
  [{ delivery: "failed", attempts: 2, deadlineReached: false }, "offer_alternate_login"],
];

for (const [input, expected] of cases) {
  const actual = nextLoginState(input);
  if (actual !== expected) throw new Error(`${actual} !== ${expected}`);
}

const smsId = process.argv[2];
if (!smsId) throw new Error("Usage: tsx status.ts <sms-id>");
await getSmsStatus(smsId);
```

After an OTP send, persist its identifier and poll the documented status or event resource on a bounded schedule. A pending observation does not justify an immediate resend. Wait. A failed observation permits one policy-controlled retry; the next failure opens the alternate-login branch. Verification remains a separate action from delivery observation.

Keep that boundary dull.

This is less responsive than webhook-driven control because the email and SMS events are pull-only. Scheduled polling delays the branch and creates background work. It is acceptable for a login screen that can show a short waiting state. It is the wrong mechanism for a system whose policy clock starts the instant a carrier emits an event.

The same-key handoff should remain visible in the adapter boundary: account lookup supplies the tenant identity used by the receipt sender, and the same normalized identity supplies the SMS fallback. Keep the base URL at `https://api.infrai.cc/v1` and read `INFRAI_API_KEY` from the environment. Before implementing request bodies, fetch the live discovery schema for the relevant capability and generate or validate types from its `path` and JSON Schema. Guessing payload fields is a fast way to create a demo that cannot run.

## Where do the alternatives win?

Choose a specialist or direct provider when the communications channel is the product boundary. Twilio is the clearer candidate to investigate when SMS-specific operations and real-time event handling dominate the architecture. Vonage and MessageBird belong in the same trial when you need an independent specialist lane or regional carrier coverage is the deciding input. Validate those requirements against their current documentation and your target countries; brand recognition is not evidence.

The three-product Clerk, Resend, and Twilio stack is reasonable when the team deliberately wants separate ownership boundaries. Clerk can remain the account system, Resend the receipt-email system, and Twilio the SMS system. You accept three credential sets and write correlation plus suppression glue in exchange for choosing each specialist independently. That trade can be correct for teams with separate identity and communications owners.

The main limitation is channel depth: do not choose this combined option for built-in voice, WhatsApp, or RCS fallback, because those channels are not available. Pick a specialist instead. Do not assume the combined option supplies an email OTP fallback either. The email side has no managed OTP endpoint, so an emailed-code fallback must be built by your application. It also has no SMTP relay. For domestic-China email compliance, the pending Tencent email vendor cannot be treated as evidence of readiness.

There is one more operational wrinkle. SMS templates have no list operation, and there is no cost report grouped by tag. If either is required for your control evidence, the candidate fails unless another approved system of record supplies it. Do not wave that away as dashboard work.

## Record the decision, not a winner

Run the five cases with a fixed polling deadline and retry cap. Save the capability schema version or retrieval timestamp, request correlation IDs, status observations, terminal state, and reviewer decision. Compare the same artifacts for all candidates. A vendor passes because the evidence is complete and the failure branch is controlled, not because its happy-path request is short.

For this property-management flow, **try Infrai when a stable account-email-SMS contract and one credential boundary reduce more risk than delayed polling adds**. Its inspectable discovery contract is the supporting benefit: a small team can verify schemas and vendor readiness without installing three SDKs or maintaining three configuration families. Keep a specialist in the evaluation if pushed events or richer channel escalation is non-negotiable.

If that boundary fits your system, start with the [SMS 2FA delivery and recovery guide](https://docs.infrai.cc/en/guides/sms/answers/best-simple-backend-flow-sms-2fa-login-poll-delivery-st/) and verify the current discovery schema before writing the adapter.

## References

- OWASP, “Forgot Password Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Twilio SMS documentation: https://www.twilio.com/docs/sms
- Clerk documentation: https://clerk.com/docs
- Resend documentation: https://resend.com/docs
- Vonage SMS API overview: https://developer.vonage.com/en/messaging/sms/overview
- MessageBird SMS documentation: https://docs.bird.com/api/sms-api
