# Direct Browser Avatar Uploads — Presigned URLs Without Making Storage Your Authorization Layer

Short answer: keep the bucket private, authorize every upload intent in the application, and issue a short-lived signed capability for one server-generated object key. The browser can send bytes straight to storage. Possessing that upload URL must not create durable application permission.

| Choice | Application control | Delivery path | Best fit | Main cost |
| --- | --- | --- | --- | --- |
| Signed direct upload | App authorizes; storage enforces a narrow capability | Browser to storage | Routine avatars and private files | CORS, expiry, and finalization state |
| Application proxy | App authorizes and carries bytes | Browser to app to storage | Inspection before storage | Streaming, limits, and timeouts |

**Decision:** a signed direct upload plus a server-side finalize step has the smaller delivery path for a routine B2B SaaS avatar. It keeps access decisions in the application without routing every byte through the application server. A proxy fits the opposite constraint: inspection must happen before storage.

Price is secondary. A cheap path with a vague authorization boundary is still a bad system.

No magic here.

## How should a browser upload an avatar with a presigned URL?

A private bucket blocks anonymous object access. It does not know that `user_42` belongs to `tenant_acme`, may replace only their own avatar, or has been suspended. The endpoint that creates a signed URL must evaluate those application facts before signing.

Treat the URL as a bearer capability. Anyone holding it can exercise the encoded operation until expiry, subject to its signed conditions. Narrow it to one method, one generated key, and a short lifetime suited to realistic clients. Never sign a raw key supplied by the browser. An object key is an address, not an authorization rule.

OWASP recommends allow-listing extensions, validating file type rather than trusting the `Content-Type` header, changing filenames, limiting size, authorizing uploaders, and keeping files outside the webroot or on another host. Direct upload does not erase those controls. It changes where they run. Checks required before storage favor a proxy; checks that may run after arrival belong in finalization or quarantine.

For avatars, allow only formats the image pipeline can decode, constrain byte size where the signing protocol supports it, generate the key on the server, and record the upload as `pending`. Do not attach it to the profile yet.

## Two criteria decide the design

The first is **authorization locality**. The application answers whether this principal may create this object for this tenant now. Storage enforces the resulting narrow capability. Encoding roles into bucket paths looks convenient, but it turns naming into policy and makes tenant changes hazardous. The hard trade-off is that direct upload splits one user action across three systems: the browser, the application, and storage. That split produces better data-plane isolation, but it also creates pending state and more failure edges. It isn't suitable when policy requires synchronous inspection before persistence. In that case, the proxy is the honest design, even though it carries more operational work.

The second is **frontend protocol surface**. Direct upload needs a small state machine: request intent, upload, finalize, then render. It also needs exact CORS configuration because the browser contacts another origin. A proxy simplifies the browser-facing origin story, but the backend inherits streaming, body limits, cancellation, timeouts, and retries. Complexity moved.

Benchmark the useful path. Measure file selection to confirmed finalization under realistic file sizes and networks, then separate signing, transfer, finalization, rejection, and abandoned-object metrics. Use percentiles. One average conceals the slow path.

Keep configuration tight: the actual web origin, the required method, and only headers the client sends. CORS is a browser permission check. It isn't tenant authorization.

Config expands fast.

## A narrow TypeScript contract

The frontend needs no storage SDK. Two application calls and one standards-based upload are enough; the server owns the key and pending record.

```ts
type UploadIntent = {
  uploadId: string;
  uploadUrl: string;
  method: "PUT";
  headers: Record<string, string>;
};

type AvatarResult = {
  avatarId: string;
  downloadUrl: string;
  downloadExpiresAt: string;
};

export async function uploadAvatar(file: File): Promise<AvatarResult> {
  const intentResponse = await fetch("/api/avatar-uploads", {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ contentType: file.type, size: file.size }),
  });
  if (!intentResponse.ok) throw new Error("Upload authorization failed");
  const intent = (await intentResponse.json()) as UploadIntent;

  const uploadResponse = await fetch(intent.uploadUrl, {
    method: intent.method,
    headers: intent.headers,
    body: file,
  });
  if (!uploadResponse.ok) throw new Error("Object upload failed");

  const finalizeResponse = await fetch(
    `/api/avatar-uploads/${encodeURIComponent(intent.uploadId)}/finalize`,
    { method: "POST" },
  );
  if (!finalizeResponse.ok) throw new Error("Finalization failed");
  return (await finalizeResponse.json()) as AvatarResult;
}
```

Send the exact headers returned with the intent. Signed requests can bind header values, so a fetch wrapper that casually adds or alters one can invalidate the request. Transport success means bytes arrived. Finalization must verify the pending record, inspect object metadata and size, run required validation, and atomically attach the accepted object.

Make finalization idempotent. Browsers retry. Tabs close. Networks lie. Repeating finalization for one accepted upload should return the same logical result; missing, expired, oversized, or mismatched objects remain unattached. A scheduled job can remove stale pending objects under a documented retention rule.

Mint download URLs only after an authenticated read check, keep them short-lived, and exclude them from logs and analytics. Review caching deliberately. MDN documents that `private` permits browser caching while preventing shared-cache storage, while `no-store` asks caches not to store the response. They are different controls.

## Failure handling belongs in the interface

Separate intent rejection, transport failure, finalization rejection, and download denial. Give each a stable application error category. Do not expose raw storage errors to users or collapse every case into one telemetry label.

Retry within boundaries. A replayable upload may retry with an unexpired capability. After expiry, request a new intent. If finalization times out, retry the same upload ID; never create a second attachment because a response was lost.

Test that one tenant cannot finalize another tenant's upload, a user cannot choose another user's key, an expired capability fails, unexpected media is rejected, an oversized object stays unattached, and a download capability cannot write. Test CORS from the real application origin in a browser. Server-side requests do not exercise browser preflight.

Trace the upload ID through intent, metadata checks, finalization, and cleanup. Redact signed query strings and authorization material. Count issued intents, client-reported transfer outcomes, validation reasons, finalization outcomes, and pending-object age.

## When the proxy is the better runner-up

The main limitation of direct upload is loss of inline control over the byte stream. Use a proxy when policy requires bytes to be inspected before they enter storage, synchronous malware or content checks must precede persistence, or storage cannot expose a sufficiently narrow browser capability. It can also reduce browser protocol work for tiny, infrequent uploads.

Pay for that choice consciously. The app must stream instead of buffering whole files, enforce limits, propagate cancellation, handle slow clients, and define partial-state behavior. Load tests need concurrent slow uploads, not only fast local requests.

For routine private SaaS avatars, direct upload keeps the data plane out of the application while leaving the control plane where it belongs: authorize intent in the app, constrain the capability, validate before attachment, and issue a separate expiring capability for each private read.

## Further reading

- MDN, `Cache-Control` response header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- OWASP, File Upload Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
