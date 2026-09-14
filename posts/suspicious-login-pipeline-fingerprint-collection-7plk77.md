# Suspicious Login Pipeline: Fingerprint Collection, Event Reporting, and Risk Scoring

**Short answer:** For phone one-time-code login in a media app, collect a small set of coarse signals, report one normalized event per attempt, and let an explicit risk policy choose allow, re-challenge, or deny; a device fingerprint should inform that choice, never become a hidden password.

Every extra challenge protects a session but also interrupts someone who may only want to read or publish. Start with a boring pipeline. Keep collection separate from scoring, keep raw secrets out of the event, and make the final decision explainable. That's enough to ship a defensible first version without turning the login handler into a pile of config.

## How should a suspicious login pipeline collect fingerprints and report risk events?

Treat the pipeline as four boundaries: collect, normalize, score, and enforce. The browser or app collects only signals the service has deliberately approved. The server normalizes them into a versioned event, derives a rotating pseudonymous fingerprint, and evaluates a policy. The login endpoint consumes the decision; it doesn't reconstruct the score.

That separation matters. If a mobile client can submit `riskScore: 0`, the score is theater. If the login handler owns twenty signal-specific branches, every new rule becomes a deployment risk. If analysts receive raw phone numbers, full IP addresses, or one-time codes in a general event stream, an authentication feature has quietly become a data-retention problem.

Keep the fingerprint coarse. An operating-system family, app major version, coarse network prefix, and stable installation nonce can help correlate attempts without pretending to identify a human. Hash the normalized values with a server-held, periodically rotated key. Store the fingerprint version and key version beside the digest so that rotation is an intentional compatibility event rather than a late-night surprise. Don't log the nonce, code, session token, or full phone number.

It is tempting to collect every browser attribute available. Skip it.

More inputs create more drift, more privacy review, and more ways to lock out an ordinary reader after an update. A fingerprint is evidence, not identity.

## The constraint that changed the choice

Phone one-time codes prove control of a delivery channel for that attempt. The suspicious-login decision still has to account for the surrounding context: a new installation, an unusual network region, repeated failed verification, or a sensitive action immediately after login. The OWASP Authentication Cheat Sheet recommends generic authentication responses so an attacker cannot distinguish whether an account exists, and it discusses adaptive or risk-based authentication as a way to add stronger checks when context warrants them. Those two ideas shape the boundary here: return the same public failure shape while preserving a richer internal reason code for authorized operators.

For a media app, session security versus friction is not an abstract slider. A reader opening an article and an editor changing payout details do not carry the same consequence. The first can often proceed after a valid code when risk is low. The second deserves a fresh check or a restricted session when risk is elevated. Put the action context into policy instead of assigning permanent trust to a device.

Consider one concrete pass through the policy. An editor enters a valid code from a newly installed app while requesting a payout change. Collection sends the approved device and request context to the server, but it does not send a score or a requested outcome. Normalization produces a fingerprint version, a policy version, and a pseudonymous subject reference. Scoring adds `NEW_INSTALLATION` and `SENSITIVE_ACTION`; enforcement sees `challenge` and issues the same kind of additional proof it would use for any elevated-risk attempt. Event reporting records the decision once under its `eventId`, even if delivery retries. The client learns that another check is required, not which signal caused it. Now change only the context to reading an article. The same valid code and new installation may remain below the challenge threshold because the action carries less consequence. This example exposes the useful property of the design: the fingerprint did not declare the editor trusted or hostile. It contributed evidence to a versioned decision about one request, and an operator can later reconstruct that decision without opening a bag of raw device attributes.

Short-lived signals also need event-time semantics. Score the facts known at the attempt timestamp, not whatever a later consumer happens to read. Attach a policy version to every decision. Without that, a support engineer can't tell whether two similar attempts diverged because the inputs changed or because the rules did.

This is where I am skeptical of opaque scores. A single `72` looks benchmarkable, but it isn't useful unless the event also records stable reason codes such as `NEW_INSTALLATION` or `RECENT_CODE_FAILURES`. I'm not sure any static threshold survives contact with every audience and traffic mix; a shadow deployment with labeled outcomes is what resolves that uncertainty.

## The smallest working implementation

The first implementation needs one event contract and one pure scoring function. No SDK layer. No rule language. The thresholds below are illustrative policy values, not universal security constants; tune them against your own false-challenge and account-takeover data.

```ts
import { createHmac } from "node:crypto";

type LoginSignalInput = {
  installationNonce: string;
  osFamily: "android" | "ios" | "web" | "other";
  appMajor: number;
  networkPrefix: string;
};

type LoginRiskEvent = {
  eventId: string;
  occurredAt: string;
  subjectRef: string;
  fingerprint: string;
  fingerprintVersion: 1;
  policyVersion: "2026-08-otp-1";
  context: "read" | "publish" | "payout-change";
  validCode: boolean;
  knownFingerprint: boolean;
  recentCodeFailures: number;
  regionChanged: boolean;
};

type RiskDecision = {
  score: number;
  action: "allow" | "challenge" | "deny";
  reasons: string[];
};

function fingerprint(input: LoginSignalInput, rotatingKey: string): string {
  const normalized = [
    input.installationNonce,
    input.osFamily,
    String(input.appMajor),
    input.networkPrefix,
  ].join("|");

  return createHmac("sha256", rotatingKey).update(normalized).digest("hex");
}

function scoreLogin(event: LoginRiskEvent): RiskDecision {
  let score = 0;
  const reasons: string[] = [];

  if (!event.knownFingerprint) {
    score += 25;
    reasons.push("NEW_INSTALLATION");
  }

  if (event.regionChanged) {
    score += 20;
    reasons.push("REGION_CHANGED");
  }

  if (event.recentCodeFailures >= 3) {
    score += 35;
    reasons.push("RECENT_CODE_FAILURES");
  }

  if (event.context === "payout-change") {
    score += 20;
    reasons.push("SENSITIVE_ACTION");
  }

  if (!event.validCode) {
    return { score: 100, action: "deny", reasons: [...reasons, "INVALID_CODE"] };
  }

  const action = score >= 70 ? "deny" : score >= 35 ? "challenge" : "allow";
  return { score, action, reasons };
}
```

The event producer should generate `eventId` server-side and make reporting idempotent on that identifier. Authentication enforcement happens synchronously from the current event; event delivery to analytics or a security queue can be retried without repeating the login decision. That distinction prevents a delayed reporting system from granting access, while still making a dropped report visible through an outbox backlog or delivery metric.

The public response stays generic. Internally, preserve `eventId`, action, policy version, score, and reasons in access-controlled logs. Never put the one-time code into a log line, trace attribute, exception, or event payload. Also avoid returning the internal reason to the client. `INVALID_CODE` and `UNKNOWN_SUBJECT` may require different operational handling, but exposing that distinction can enable account enumeration.

Test the pure function with a table of boundary cases: scores immediately below and at each threshold, a valid code on a new installation, repeated failures on a known installation, and a sensitive action after an otherwise ordinary login. Then test the integration twice with the same `eventId`. Exactly one report should be retained and both calls should produce the same policy result. This is a useful benchmark because it measures behavior, not a synthetic requests-per-second number detached from the security contract.

## What I would change at scale

At scale, I would split synchronous enforcement from asynchronous enrichment. The synchronous path should use signals already available at login and finish within a strict latency budget. Slower correlation -- velocity across subjects, aggregate network reputation, or reviewed abuse labels -- belongs in an event consumer that can update future decisions. It must not retroactively mutate the record of why an earlier session was allowed.

Add shadow mode before adding sharper rules. A candidate policy records the action it would have taken without changing the live decision. Compare challenge rate, deny rate, later confirmed abuse, and successful completion by context. Segment those results by app major version and fingerprint version so a client release doesn't masquerade as an attack wave.

Then add expiry. Fingerprint associations should have a documented retention period, and the rotating key needs a defined overlap window if old and new digests must coexist. Access to raw login events should be narrower than access to aggregate dashboards. These are dull controls. Good. Dull controls are easier to audit.

Operational alerts should track pipeline health separately from user risk: event-production lag, duplicate rate, missing policy versions, and the share of decisions with no reason code. A spike in high-risk logins is a security signal; a spike in events missing required fields is a software-quality signal. Combining them into one red light wastes time during an incident.

## Trade-offs and the stopping rule

This design is not suitable when a fingerprint must serve as legal identity proof, and it is a poor fit for offline authentication because scoring depends on server-side state. For a very small app with no sensitive account actions and little abuse, stick with a valid one-time code, conservative session lifetime, rate limits, and careful logging until measured risk justifies the extra collection. The catch is that a pipeline has ongoing costs: privacy review, rule calibration, support cases, and on-call ownership.

Conversely, don't stretch a score into authorization. A low-risk login should not automatically permit payout changes, mass publishing, or account recovery. Use normal authorization checks and require recent authentication for sensitive operations. Risk can narrow a session or request another proof; it cannot replace permissions.

My stopping rule is simple: collect a signal only when a named policy consumes it, its retention is defined, and a test shows how it changes an action. Delete unused fields. Benchmark the challenge rate and decision latency on every policy revision. If the team cannot explain a denial from the stored event and versioned policy, the pipeline is too clever to operate.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
