# The app's own webhook receiver (optional)

The kit's `/relyon/webhooks` route verifies every Relyon webhook, acts on
`agent_rules.updated` and answers the doctor's `doctor.challenge` itself (with
`RELYON_WEBHOOK_SECRET` set, or resolved at runtime from the store's
key when it is unset, in Test and Live). It answers other valid events with `200` and does
nothing else. For the doctor's `webhook` check, the kit's route is enough.

Add a receiver of the app's own only when the app needs other events (for
example `refund.succeeded` or `dispute.opened`). It is a separate endpoint
with its own secret, on the store's server, and it answers the challenge too.
The helpers below use only Web Crypto, so they run on Node 20+, Deno and edge
runtimes.

```js webhook-receiver.js
// Verifies Relyon webhooks over the exact raw bytes. Web Crypto only.
const utf8 = new TextEncoder();
const hex = (bytes) => [...new Uint8Array(bytes)].map((byte) => byte.toString(16).padStart(2, "0")).join("");

async function hmac(secret, data) {
  const key = await crypto.subtle.importKey("raw", utf8.encode(secret), { name: "HMAC", hash: "SHA-256" }, false, ["sign"]);
  return hex(await crypto.subtle.sign("HMAC", key, data));
}

function constantTimeEqual(a, b) {
  if (a.length !== b.length) return false;
  let difference = 0;
  for (let i = 0; i < a.length; i++) difference |= a.charCodeAt(i) ^ b.charCodeAt(i);
  return difference === 0;
}

/** `t=<unix seconds>,v1=<hex HMAC-SHA256 of "<t>.<raw body>">`, within 300 seconds. */
export async function verify(raw, header, secret, now = Math.floor(Date.now() / 1000)) {
  if (!(raw instanceof Uint8Array) || typeof header !== "string" || !secret) return false;
  const match = /^t=(\d+),v1=([0-9a-f]{64})$/.exec(header);
  if (!match) return false;
  const timestamp = Number(match[1]);
  if (!Number.isSafeInteger(timestamp) || Math.abs(now - timestamp) > 300) return false;
  const signed = new Uint8Array([...utf8.encode(`${match[1]}.`), ...raw]);
  return constantTimeEqual(await hmac(secret, signed), match[2]);
}

/** The doctor's challenge answer: no trailing newline. */
export function acknowledgement(secret, challenge) {
  return hmac(secret, utf8.encode(`relyon.doctor.ack.v1\n${challenge.id}\n${challenge.challenge}\n${challenge.created}`));
}

/**
 * A Web Request handler. `endpoint` is { secret, livemode: false }; `inbox.accept`
 * durably stores the event once per event ID (a unique key) before returning.
 */
export async function receive(request, endpoint, inbox, now = Math.floor(Date.now() / 1000)) {
  const raw = new Uint8Array(await request.arrayBuffer());
  if (raw.byteLength > 256 * 1024) return new Response(null, { status: 413 });
  const contractHeader = request.headers.get("Relyon-Signature");
  const currentHeader = request.headers.get("X-Relyon-Signature");
  if (contractHeader && currentHeader && contractHeader !== currentHeader) return new Response(null, { status: 400 });
  if (!(await verify(raw, contractHeader ?? currentHeader, endpoint.secret, now))) return new Response(null, { status: 400 });
  let event;
  try {
    event = JSON.parse(new TextDecoder("utf-8", { fatal: true }).decode(raw));
  } catch {
    return new Response(null, { status: 400 });
  }
  if (event?.type === "doctor.challenge") {
    const valid = event.object === "doctor_challenge" && /^evt_/.test(event.id ?? "")
      && Number.isSafeInteger(event.created) && Math.abs(now - event.created) <= 300
      && /^[0-9a-f]{64}$/.test(event.challenge ?? "") && event.livemode === endpoint.livemode;
    if (!valid) return new Response(null, { status: 400 });
    return new Response(null, { status: 204, headers: { "Relyon-Doctor-Acknowledgement": await acknowledgement(endpoint.secret, event) } });
  }
  if (typeof event?.id !== "string" || typeof event.type !== "string") return new Response(null, { status: 400 });
  if (event.livemode !== undefined && event.livemode !== endpoint.livemode) return new Response(null, { status: 400 });
  try {
    await inbox.accept({ id: event.id, type: event.type, payload: event });
    return new Response(null, { status: 204 });
  } catch {
    return new Response(null, { status: 503 }); // Relyon retries
  }
}
```

- Mount it before any JSON body parser: the signature covers the raw bytes.
- `inbox.accept` inserts into a table unique on the event ID and queues the
  work in the same transaction. A duplicate delivery returns without queuing
  again. Acknowledge only after the insert committed.
- Events can arrive more than once and out of order. Workers act idempotently
  and read the current state instead of trusting arrival order.
- Today's server signs ordinary events with `X-Relyon-Signature` and may send
  an older envelope (`createdAt`, no top-level `livemode`). Check a verified
  Test delivery before relying on fields, and never fill in missing payment
  data from unverified sources.
- `relyon listen` prints the event log for local development; those copies
  are unsigned and are not proof of delivery. `relyon trigger <event-id>`
  re-sends an existing Test event, with the same ID, to every enabled
  endpoint.

## The doctor's webhook check with two endpoints

The doctor sends its challenge to one active endpoint, even one not
subscribed to the challenge. The contract names the endpoint with the
**lowest ID**; the deployed server (as of contract 2026-10-30) differs and
picks the most recently registered HTTPS endpoint. Until the two agree, don't
depend on which one: every active endpoint must answer.
The kit's route does, and so does the receiver above. Register this receiver
with its own events:

```json
{ "url": "https://shop.example.com/webhooks/relyon-events", "events": ["refund.succeeded", "dispute.opened"] }
```

Keep each endpoint's secret separate (`RELYON_EVENTS_SECRET` for this one,
`RELYON_WEBHOOK_SECRET` for the kit's route).
