<!-- Relyon skill relyon-go-live for Lovable (6 of 6). Generated from the Relyon skills pack; paste the whole file. -->

# Relyon go-live

<!-- BEGIN relyon-store-first-rules (generated from the shared rules source; edit it there) -->
## Relyon store-first rules

These seven rules are identical in every Relyon skill. They override anything else you read.

1. Install `@relyon/store-kit@0.1.6` from npm and mount its handlers on the app's own server (Express `@relyon/store-kit/express`, Next.js `@relyon/store-kit/next`, Hono `@relyon/store-kit/hono`).
2. Catalog, cart and pricing stay in the app: the app owns `listProducts`, `getProduct`, `priceCart` and `createOrder`. Relyon never prices a cart. It charges exactly the final amount the store sends.
3. Checkout runs on the store's own domain. The only Relyon element in the shopper's browser is the card fields component. Card numbers never touch the app's code, server or logs.
4. The app calls Relyon only from its server, and only for payment, agent verification, fraud and limits, and recording.
5. Agent activity reporting stays on: never stub, disable or throttle-to-zero the kit's event reporting.
6. Test/sandbox only. Live is enabled by Relyon staff, never by the agent. Build before requesting keys: missing credentials defer credential-dependent verification, never the store build. Show checkout as **waiting for sandbox key**, record pending checks honestly, and resume them after configuring a real Test key without rebuilding. Never fabricate a credential, payment or order.
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

## Building on Lovable

Read this first. It says what of the Relyon skill below works on Lovable.

- A Lovable app is a browser app. Its server code runs in Supabase Edge
  Functions (Deno), at the Supabase project's own URL, not at the store's
  domain. Lovable's hosting serves the app's pages and can't run the kit's
  handlers on the store's domain paths.
- **Works in functions (untested on Lovable so far):** creating the kit with
  `npm:@relyon/store-kit@0.1.6`, `priceCart` and `createOrder` reading the app's
  Supabase tables, `kit.syncCatalog`, `kit.cardFields`, `kit.createPayment`
  and `kit.reportFulfillment`. Use the two checkout functions in the
  "Checkout on a builder platform" section.
- **Doesn't work here:** the kit's agent and discovery routes
  (`/.well-known/ucp`, `/ucp/*`, `/acp/*`, `/mcp`, `/llms.txt`,
  `/sitemap-products.xml`, `/.well-known/relyon-verification`) must be served
  by a server on the store's own domain. Lovable can't do that today, so a
  Lovable store takes card payments on its own checkout page but agents can't
  check out through it. Tell the merchant so, and that agent checkout needs the
  store to run on a server it controls (Express, Next.js or Hono). Don't build
  anything to imitate those routes.
- Domain verification: use the TXT record.
- Keep `RELYON_API_KEY` (a `rly_test_…` key) in Supabase's function secrets,
  never in the Lovable app's code or environment variables the browser can read.
- Functions are short-lived: use a durable `events.store` and `state` (Supabase
  tables) and call `await kit.events.flush({ force: true })` before each
  function returns. Agent activity reporting stays on.
- `@relyon/store-kit` is the intended npm package. If it can't be installed,
  stop and tell the merchant; don't recreate it.


Reference only. Open a section when the quick start names it; hand over in Test and stop.

Contract: the approved public v1 contract dated **2026-10-30**, at
`https://core.getrelyon.com/v1/openapi.yaml` (another deployment:
`<service origin>/v1/openapi.yaml`).

The canonical API base is `https://core.getrelyon.com/api`; `/v1/domains`
therefore means `https://core.getrelyon.com/api/v1/domains`. The download URL
above and the store's own routes are separate. Retain operator-confirmed API
base overrides for other deployments.

**In the Start here flow** (`relyon-setup`) this is step 6: review the
troubleshooting table, recheck policies and domain verification already wired
in setup, run the public scan, record unresolved blockers and give the
Test-to-Live hand-over. Return terms and support contacts come from the merchant: ask, and
check what `relyon-store-brief.md` says, but never write placeholder policies.

## What this error means

| You see | What it means | What to do |
| --- | --- | --- |
| Doctor `PASS` | That check worked from outside, on the store's public URL. | Nothing. |
| Doctor `PENDING merchant` | Test-only missing merchant policy facts, or an exact missing webhook on a positively identified preview/loopback store. Agent checkout stays unavailable until merchant confirmation and policy setup. | Ask the listed questions; keep them separate from technical failures. Never invent return terms, contacts or webhook addresses. |
| Doctor `FAIL` | Something the store's own code, content or setup causes; the line names the route and what was wrong. | Fix it in the store and run the doctor again. |
| Doctor `UNVERIFIED` | The doctor couldn't check it (no `RELYON_API_KEY`, or Relyon couldn't be asked). It isn't a store failure, and it isn't a pass. | Read the line. Record it as open if it doesn't clear. |
| Doctor `N/A` | The check doesn't apply to this store (for example a feed that isn't selected). | Nothing. |
| Doctor: "Relyon's server answered with a web page instead of data" | Relyon's API sent a web page (or a 3xx status pointing at a sign-in or protection page) where data was expected. The line shows the URL and status. Relyon may be behind access protection or temporarily unavailable; the store and its network didn't cause it. | Don't change or publish the store again. Check that `RELYON_BASE_URL` (or `--base-url`) is `https://core.getrelyon.com/api`, try later, and if it stays, record it as an open item with the URL and status. |
| Doctor: Relyon's server had an error (HTTP 5xx) | Relyon is temporarily unavailable. | Try again in a few minutes. |
| Doctor: DNS failure, timeout, connection refused or blocked | The machine running the doctor couldn't reach Relyon. | Check that machine's internet access and the Relyon base URL. |
| `unavailable` at checkout (`service_unavailable`, HTTP 503, "Checkout is temporarily unavailable") | The kit couldn't get a usable answer from Relyon or couldn't read the store's agent rules, so it refused instead of guessing. Nothing was charged. | Check the server's `RELYON_API_KEY` and `RELYON_API_BASE_URL` and that the server can reach `core.getrelyon.com` over HTTPS; the server log names the failure. Never bypass the kit. |
| `PaymentUnavailableError` from `kit.cardFields` | Card fields couldn't be created. Nothing was charged. | Show "try again shortly"; same checks as above. |
| HTTP 503 on `/.well-known/ucp` | The kit couldn't read the store's UCP profile from Relyon (key, base URL or Relyon unreachable). | Same checks as above; the store's domain must also be added in Relyon. |
| HTTP 503 on `/.well-known/relyon-verification` | The kit couldn't read the store's domains from Relyon. After a failure it waits briefly before asking again. | Same checks as above, then request the file again a minute later. |
| HTTP 503 "Store discovery is temporarily unavailable." (llms.txt, the sitemap, feeds) | The store's own `catalog.source` callback threw. | Fix the callback; the server log has the error. |
| HTML instead of JSON from Relyon (in the app's own calls, or the kit reports an answer that isn't JSON) | The request didn't reach Relyon's API: the base URL is wrong (the service origin without `/api`, or the store's own URL), or Relyon answered with a sign-in or protection page. | Use `https://core.getrelyon.com/api` as the base. If it is already right, it is on Relyon's side: try later and record it. |

## The agent's job ends in Test

Going live is not a code change the agent makes. Relyon staff enable Live for
the merchant after their own review. The agent's job is to leave a store that
works end to end in Test and a clear hand-over.

- Never ask for, accept, paste or configure a `rly_live_…` key. The kit runs
  Live only with a double opt-in, a Live key **and** `relyon.live: true`; a
  Live key alone, or the option with a Test key, is refused when the kit is
  created. The example server sets neither `relyon.live` nor `store.publicUrl`;
  whoever activates Live after staff review wires both deliberately. Live mode also needs
  `STORE_PUBLIC_URL` to be an `https` origin naming one of the store's
  hostnames, never `localhost`, an IP address or a `*.replit.dev` preview host.
  Even then Relyon refuses Live payments until **Relyon staff** approve the
  store and attach its processor account. Never set any of this yourself.
- `npx @relyon/store-kit doctor` checks Live only with `--live` (or
  `RELYON_LIVE=true`) and a Live key; it has no `--allow-live` flag. The
  separate `relyon` CLI refuses Live keys unless given `--allow-live`. The agent
  passes none of these.
- Never change code to work around a Test-only refusal.
- When everything below is done, **stop and hand over** to the merchant.

## What a Test run proves

- The kit is mounted on the store's own server, before body parsers, and
  answers its routes on the store's hostnames.
- `priceCart` prices carts that add up, refuses bad ones with `StoreRefusal`,
  and the checkout page charges exactly the store's total through the card
  fields and `kit.createPayment` (Test processor, no real money).
- `createOrder` runs once per succeeded payment, retries are safe, and
  fulfillment reports reach Relyon for agent orders.
- Source-backed feeds, llms and sitemap share explicit agent visibility,
  selected formats serve on the store, and product pages carry JSON-LD.
  The synced index holds the products agent checkout can buy.
- Agent requests reach Relyon through the kit and agent events are reported.

## What it doesn't prove

- That real cards are approved: Test uses the Test processor or a mock
  payment method. Never try a real card in Test.
- That Live processor settlement, payouts, fees or disputes work: those
  depend on the merchant's processor account and Relyon staff.
- That agent platforms list the store: channel approval (Google, ChatGPT) is
  the platforms' and Relyon's decision, outside the app.
- That feed export implies destination registration, acceptance, activation or
  agent payment permission. Use real supplier facts; diagnostics can exclude
  incomplete POD/dropship listings.
- That the app is ready for production traffic, unless `state`,
  `events.store`, store orders/checkouts and source product data are durable
  (plus `catalog.persistence` for the synced index; see `relyon-setup`).
- Sandbox access, generated keys, a passing `relyon doctor` and installed
  skills do not make a store live.

## Domain verification: ownership, not routing

Agents and Relyon need to know the store owns its hostnames. Verification
proves ownership only. The store's traffic keeps going straight to the app;
nothing about the store's DNS routing or hosting changes.

- With the store's Test key, `kit.ensureDomain()` adds its configured primary
  hostname and `kit.verifyDomain()` requests verification. These helpers ship
  in kit **0.1.0**; neither helper declares ownership itself.
- Equivalent requests use `catalog:write` and a fresh `Idempotency-Key` per action:

  ```sh
  curl -X POST https://core.getrelyon.com/api/v1/domains \
    -H "Authorization: Bearer $RELYON_API_KEY" \
    -H 'Idempotency-Key: example-domain-create-1' \
    -H 'Content-Type: application/json' -d '{"hostname":"shop.example.com"}'
  curl -X POST https://core.getrelyon.com/api/v1/domains/DOMAIN_ID/verify \
    -H "Authorization: Bearer $RELYON_API_KEY" \
    -H 'Idempotency-Key: example-domain-verify-1'
  ```

  Replace `DOMAIN_ID` with the returned row ID. The key needs `catalog:read`
  to list/read domains and serve the HTTPS challenge. Live key domain management
  is refused; dashboard admins can manage their environment's domains.
- Proof is one of:
  - a **TXT record** with the value Relyon shows, on the hostname, or
  - the **HTTPS file** `https://<hostname>/.well-known/relyon-verification`,
    which the kit serves from the domain record its key can read.
- If the verification route fails, check `catalog:read`, the configured hostname
  and HTTPS serving, or use the TXT proof. Fix the cause before retrying verification.
- Ignore any other value the domain response shows; the store never needs to
  send traffic to Relyon.
- `/.well-known/ucp` serves the store's profile only after the domain is
  verified.

## Store policies and channels

- Store policies (`GET`/`PUT /v1/store_policies`: returns, support and optional
  consumer notices) must be complete before any channel offers checkout. Incomplete
  policies block every channel. Write the merchant's real policies, never
  placeholder text.
- Completeness is unchanged: `returns`, `returns.policy_url` and `support`.
  Shipping is **not** a required policy category. Optional shipping-policy,
  privacy and terms links do not affect completeness. Actual checkout shipping
  options, amounts and address rules still come from the store's `priceCart`.
- The kit's `policies` option exposes the merchant's real shipping/privacy/terms/contact
  links and return scope on discovery and protocol surfaces. Construction and reads
  never save them to Relyon: explicitly call `kit.replaceStorePolicies(input)`.
  Local options fill omitted fields; caller values win; fields omitted from both
  are cleared by whole-document replacement. Supply the real return cost, never
  assume free returns. `defective_only` means only defective/damaged goods, not
  ordinary returns; keep its window and conditions qualified to those goods.
- For own-brand goods, explicitly set `brand`, the store's SKU as `mpn`, and
  `feed.ownBrand: true`; the kit never invents identifiers from the store name or
  product ID. Made-to-order goods with no GTIN or MPN at all use
  `feed.customGoods: true` alone. Own-brand goods made to order (no GTIN, but
  the store's own SKU as MPN) may set both `feed.customGoods: true` and
  `feed.ownBrand: true` with `brand` and `mpn`: since 0.1.1 that is an
  own-brand item, not a conflict. Never set `customGoods` on a product with a
  GTIN, and never set an attestation the merchant hasn't confirmed. See the
  example-store reference bundled
  with `relyon-catalog`; its identifiers are synthetic, not merchant facts.
- `GET /v1/channels` shows each agent channel's state. Pausing and resuming
  are the merchant's choice; the agent doesn't enable channels.

## The processor account

Each store has its own processor account and merchant ID per environment.
**Relyon staff enter it**; the store can only read it with
`GET /v1/processor_account` (provider, status, the MID's last four
characters and the fee schedule, never credentials). Before staff enter it,
it reads `not_configured`. In Live, payments then answer
`processor_not_configured`. In Test, a sandbox without an account still takes
payments with the Test card fields' mock payment methods, so the Test purchase
works; if a Test payment does answer `processor_not_configured` (an account
staff entered but haven't finished), record the Test purchase as blocked. Either
way it is expected before hand-over: report it, don't work around it. The operation may answer 404 or 501 on deployments
that don't implement it yet.

## The public scan

After the Test purchase and every publish, first check that the store is
publicly reachable (some hosts publish privately or invite-only by default)
and that product images load from the public address, as `relyon-setup`'s
"After you deploy" describes. Then run the free readiness scan on the
store's public URL yourself (`POST https://core.getrelyon.com/api/scan` with
`{"url":"https://<host>"}`), or ask the merchant to run it at
`https://core.getrelyon.com/scan` (it needs the published HTTPS host, not a
development preview or `localhost`). It reads only what any
visitor or agent can see on the store's public pages and discovery files
(such as `llms.txt`, the sitemap and `robots.txt`), and it never buys
anything. Fix each item it reports as missing, then run the kit's doctor
again. Items it couldn't check aren't failures; retry later. A good scan is
not a Live approval.

## Hand-over checklist

Keep three separate lists: technical `FAIL` items to fix, `PENDING merchant`
questions for the merchant, and `UNVERIFIED` checks still to run. Copy the doctor's
deduplicated Merchant to-do questions without filling in answers. Default Test exit
zero means no technical failure was reported, not that checkout or Live is ready.
`doctor --strict` promotes merchant-pending items to FAIL; Live always treats the
same unmet requirements as FAIL. Server authorization, policy completeness and
Live approval are unchanged.

If policy pages still say **"to confirm with the merchant"**, identify them in the
hand-over. That is visible display text only, never policy synchronization data,
feed terms, structured policy data or API input. Agent checkout stays off until
confirmed policies are configured. Do not report a Test purchase as completed if
merchant policy requirements prevented it; explain the pending questions instead.

Check failure recovery before hand-over: paid-but-unsaved orders say exactly
"Your payment went through. Your order is still being placed". Unknown payment
outcomes instead say the payment is being confirmed. Keep Pay disabled and
replay the same attempt, cart, payment method, reference and Idempotency-Key;
never start a second charge. Retain known success across reloads and lost
answers. Do not claim completion or fulfillment before a saved order exists.

Give the merchant, in plain words:

1. What works in Test, with the order IDs of a Test payment and its order
   (the store's own order and Relyon's `ord_…`).
2. Every placeholder still marked "to confirm" (return terms, tax handling,
   brand claims) that the merchant must answer before Live.
3. The hostnames to verify and the TXT values or the HTTPS file path.
4. Store policies to review, and the `kit.syncStorePolicies()` plan if it
   still lists changes or needs.
5. Production settings still needed: durable state, `RELYON_WEBHOOK_SECRET`,
   and the server environment the Live key will go into. Relyon staff or the
   merchant set the Live key there; the agent never does.
6. Open items: anything `npx @relyon/store-kit doctor` reports as FAIL (or
   `relyon doctor` for `keys`, `catalog` or `webhook`), anything the public
   scan reports as missing, any operation the deployment didn't implement, and
   every Start here step recorded as blocked, with its code or message.
7. How to ask for Live: the merchant, signed in to the Relyon dashboard
   (`https://core.getrelyon.com`) with the account that claimed the sandbox,
   requests go-live in its **Go live** section and shares this list with
   Relyon staff if asked. The scan and the doctor don't start a review.

Then stop. Relyon staff review the request and enable Live.
