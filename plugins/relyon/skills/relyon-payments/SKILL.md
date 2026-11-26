---
name: relyon-payments
description: Accept card payments in your own app with payment sessions and hosted fields, without a Relyon catalog or kit-built shopping pages.
---

# Relyon payments

<!-- BEGIN relyon-store-first-rules (generated from the shared rules source; edit it there) -->
Before building, ask the user: 'Do you want to (1) accept card payments in your own app — no Relyon product catalog or shopping pages; (2) also list products so AI shopping agents can find and buy them; or (3) also have Relyon's kit build the shopping pages for you?' Don't choose for them. Then follow the matching quickstart.

## Choose the integration before applying kit guidance

Payments: https://core.getrelyon.com/docs/payments-quickstart and `relyon-payments`.
Products and agent selling: https://core.getrelyon.com/docs (catalog,
agent checkout and AI visibility are optional additions).
Shopping pages: https://core.getrelyon.com/docs/quickstart and `relyon-setup`.
Never install the kit or build catalog, /cart or shopping pages for payments only.
The kit-specific rules below apply only after the user chooses shopping pages.
Live payments support human buyers and agents in every integration after staff
approval and processor onboarding, with existing payment safety checks intact.
Use one secret key, kept server-side as RELYON_API_KEY; browser code never gets a key.

When using the optional kit and uncertain about its version, run `npm view @relyon/store-kit version`.
Compare it with the store project's actually installed version; never downgrade.
Install the exact returned version with the project's package manager (npm install
--save-exact, pnpm add --save-exact, yarn add --exact, or bun add --exact), then run
`npx @relyon/store-kit doctor` afterward. If the lookup fails, say it is unavailable;
do not guess a latest version. Respect package-age holds without changing settings.

## Test setup and hand-over

- Start the store as a separate detached background process, using the host's
  process manager or `setsid nohup npm run start >store.log 2>&1 < /dev/null &`.
  Never run it in the agent's own foreground session. Stop a recorded PID;
  never use a `pkill` pattern that can match the shell issuing it.
- On a public Test URL, call `kit.ensureDomain(hostname)` and then
  `kit.verifyDomain(domain.hostname)` after the verification route is reachable.
  Domain verification is not only for Live.
- Say exactly “Test mode: no real money moves” when handing over a Test store.
  Never say shoppers can buy or pay for real while the store is in Test.
- Server-only `loadTestConfig` supplies both `apiKey` and `webhookSecret`.
  Wire them into `relyon.apiKey` and `rules.webhookSecret`; never expose either
  in browser code or agent prompts.
- With an environment API key, `webhook` never writes a secret file.
  A set `RELYON_WEBHOOK_SECRET` in environment or dotenv overrides key-only
  runtime recovery. Unset it, or update it in host settings, then restart.
  Rotation exits 1 if an override differs from the current secret. Do not
  treat that exit as a failed rotation: the endpoint may already be rotated.
  Never read or print the override value; never ask for it in an agent prompt.

## Relyon store-first rules

These seven kit rules are identical in every Relyon skill. They apply only to the shopping-pages integration, never to payments-only apps.

1. Install `@relyon/store-kit@0.1.58` from npm and mount its handlers on the app's own server (Express `@relyon/store-kit/express`, Next.js `@relyon/store-kit/next`, Hono `@relyon/store-kit/hono`).
2. Catalog, cart and pricing stay in the app: the app owns `listProducts`, `getProduct`, `priceCart` and `createOrder`. Relyon never prices a cart. It charges exactly the final amount the store sends.
3. Checkout runs on the store's own domain. The only Relyon element in the shopper's browser is the card fields component. Card numbers never touch the app's code, server or logs.
4. The app calls Relyon only from its server, and only for payment, agent verification, fraud and limits, and recording.
5. Agent activity reporting stays on: never stub, disable or throttle-to-zero the kit's event reporting.
6. Start in Test. Live is enabled by Relyon staff, never approved by the agent; after staff approval, the agent activates it at the merchant's explicit instruction (set the env vars by name, redeploy, run the Live doctor for shopping pages) and never asks for or pastes a Live secret key. Build before requesting keys: missing credentials defer credential-dependent verification, never the local build. Show checkout as **waiting for a Test secret key**, record pending checks honestly, and resume them after configuring a real Test secret key without rebuilding. Never fabricate a credential, payment or order.
7. No Relyon-hosted checkout, redirect to a Relyon page, CNAME, proxy, forwarding Worker or `quote.request`/pricing endpoint.

### Test defaults — not merchant policy

| Item | Test default |
| --- | --- |
| Brand | Store name, unless the merchant supplies another brand. |
| MPN | SKU, unless the merchant supplies another MPN. |
| Stock | Available unless told otherwise; no invented inventory count. |
| Returns/support | Pending merchant items. Never invent them or submit placeholders as real policy data. |

Explicit merchant facts override these Test defaults. Proceed with local Test
building; ask before asserting missing merchant promises or syncing policies.
Never invent supplier claims, GTINs, tax treatment or policy terms. Pending
merchant information blocks only the checks that require it, not the build.
<!-- END relyon-store-first-rules -->

For payments only, follow https://core.getrelyon.com/docs/payments-quickstart.md.
That is the shared, maintained quickstart; do not duplicate its implementation
from an old example. No kit, catalog, feed, doctor or /cart page is required.
Products and agent selling and kit-built shopping pages are optional additions,
not payment prerequisites.

Read the public contract at https://core.getrelyon.com/v1/openapi.yaml
(or `<service origin>/v1/openapi.yaml`). Use the currently served contract,
not a copied historical version. Versions are dated: send
`Relyon-Version: 2026-11-21`. A key is pinned to the version current when it was
created; the previous version, `2026-10-30`, is still accepted during the
project-space rename window.

1. Use one secret key, kept server-side as `RELYON_API_KEY`; browser code gets
   only the payment session's client token, never a key. Never request secret
   values in an agent prompt.
   Sign in at https://core.getrelyon.com/get-sandbox-key to create a sandbox.
   Init does not create one automatically.
2. Use https://core.getrelyon.com/api/v1/payment_sessions with an idempotency key
   and the amount, currency and your order reference.
3. Pass the client token to hosted card fields. Card numbers go directly to
   the processor, never through your server, Relyon's API or logs.
4. Send the encrypted result to your server; call `/v1/payments` with
   `payment_method.type: "hosted_card"`, `payment_session`, `encrypted_data`
   and `key_id`. Include your order's cart breakdown and final amount; the
   cart body is order data, not a Relyon catalog or a shopping-page requirement.
5. Verify webhooks before trusting a payment. Register an https endpoint with
   `POST /v1/webhook_endpoints` (`url`, `events`, an `Idempotency-Key`); its
   one-time `secret` goes in server-side secret settings as
   `RELYON_WEBHOOK_SECRET`, never printed, logged or committed. Read the
   `Relyon-Signature` header, falling back to `X-Relyon-Signature` (what Relyon
   sends today): `t=<unix seconds>,v1=<hex>`, where `v1` is the HMAC-SHA256 of
   `<t>.<raw body>` keyed with the whole secret. Verify the raw request body
   before parsing it, compare in constant time, refuse timestamps more than
   300 seconds off, answer 2xx quickly and de-duplicate on the event `id`.
   Mark an order paid only on a verified `payment.succeeded`, never because
   the shopper's browser returned to your page. Verifier code, events and
   retries:
   https://core.getrelyon.com/docs/webhooks.md
6. Retry unknown payment outcomes using the original idempotency key and
   identical request, never a new charge. Confirm recorded outcomes before
   saying payment succeeded.

Each workspace has one project space, selected automatically for sessions,
payments and refunds; to run another store, create another workspace. Only an
older workspace with more than one project space needs
`Relyon-Project-Space: prj_…` or
`Relyon-Project-Space-Hostname: your verified hostname`; otherwise it gets
`400 project_space_required`. A key limited to one project space always uses it.
Optional product-selling policies use `/v1/project_space_policies`.

Live payments work for payments-only human-buyer apps and agents alike.
Staff approval, an active onboarded processor, Live keys, payment limits and
existing safety checks still apply. Never ask for or display Live secret values;
the user sets them in deployment secrets. Activate Live only on explicit instruction.
Test mode: no real money moves.
