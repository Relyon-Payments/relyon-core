# Relyon Core quick start — Test only

Do not read dist files, framework docs, or the full README.
Six skills are references: open only the section a numbered step names.

## 1. Init first

In the app's project directory:

```sh
npx @relyon/store-kit@latest init
```

Read `relyon/QUICKSTART.md`. Init installs kit `0.1.6` and references.
`init --store "your idea"` saves merchant data, not instructions; `--help` lists
options. Conflicts write nothing: compare/add the manual pointer or preserve
edits before `--force`. Never bypass symlink/marker refusals.

Plain npm install? Still run init first; scripts/output may be disabled.
Reproducible dependency: `npm install @relyon/store-kit@0.1.6`.
Unpublished validation: install the local tarball, then run
`./node_modules/.bin/store-kit init --skip-install`, never `@latest`.
Workspaces: follow init's server-package command.

### Test defaults — not merchant policy

| Item | Test default |
| --- | --- |
| Brand | Store name, unless the merchant supplies another brand. |
| MPN | SKU, unless the merchant supplies another MPN. |
| Stock | Available unless told otherwise; no invented inventory count. |
| Returns/support | Pending merchant items. Never invent them or submit placeholders as real policy data. |

Explicit facts override defaults. Ask before asserting missing tax/supplier/
policy promises or syncing them; block dependent checks, not building.

## 2. Mount kit routes (implementation, not a CLI command)

Use Node 20+, ESM, one kit per process, `priceCart`, transactional `createOrder`,
read-only `findOrderByPaymentId`. Deployed/multiprocess stores need shared durable
checkout/payment/claim/event storage. Never disable reporting.
Open setup's [Express wiring](skills/relyon-setup/references/express-server.md)
and [durable state](skills/relyon-setup/references/durable-state.md).
[Create kit](skills/relyon-setup/SKILL.md#create-the-kit-on-the-server):

```js
import { createStoreKit } from "@relyon/store-kit";
import { storeKitMiddleware } from "@relyon/store-kit/express";
// options: store callbacks, durable adapters, server-only configuration.
const kit = createStoreKit(options);
app.use(storeKitMiddleware(kit)); // before body parsing/auth/static fallback
app.use(express.json());
```

Checkout stays on-store: only Relyon's card fields enter the browser.
Never collect card data in app code/logs or add payment intermediaries.
Automatic Test setup needs `RELYON_PRIVATE_DIR`: existing owner-only 0700 POSIX
directory outside the app/every git tree, never public/static.
Never blindly retry provisioning; preserve keys.
Node servers can use `loadTestConfig({ project: process.cwd() })` from
`@relyon/store-kit/server-config`.
No safe setup? Build with **waiting for sandbox key**, never a fake key.
Only after building, if still waiting, send the merchant to
https://core.getrelyon.com/get-sandbox-key and host secret tooling.
Never request, print, log or commit credentials/claim URLs.
Restart with a real Test key; resume checks without rebuilding.

## 3. Catalog and discovery

Use durable `catalog.source.listProducts({ cursor, limit, query })` →
`{ products, nextCursor }`, `getProduct(id)` → product or null. Open catalog's
[source shape](skills/relyon-catalog/SKILL.md#the-product-type-in-typescript)
and [feeds](skills/relyon-catalog/SKILL.md#source-backed-discovery-and-feeds).
Use minor units/uppercase currency; `from_price` is display-only.
Set `configurable`, variants, exact feed prices,
`agent: { visible: true, purchasable: true }` (not purchase authority).
Select needed feeds; never invent GTINs/supplier attestations.
Defaults are not Live facts. Serve product pages/images with
`kit.renderProductJsonLd(product, request)` and homepage
`kit.renderDiscoveryLinks(request)`.
With a key run `await kit.start()` then `kit.syncCatalog(products)` and after edits.
Source discovery needs no sync.

Use agents' [UCP/ACP/MCP routes](skills/relyon-agents/SKILL.md#what-the-kit-serves-on-the-stores-domain).
Preserve merchant restrictions with
[robots merge](skills/relyon-catalog/SKILL.md#robots-explicit-conflict-safe-opt-in).
Feed export/discovery is not channel approval or permission to buy.

## 4. Store-owned checkout

Use checkout's
[flow](skills/relyon-checkout/SKILL.md#the-flow) and
[checkout page](skills/relyon-checkout/references/checkout-page.md).
Price server-stored carts; show lines/shipping/tax/discounts/total.
Use server `kit.cardFields(...)`, browser `tokenize()`, then server
`kit.createPayment(...)` for the unchanged displayed total.
Use [payment compatibility](skills/relyon-checkout/SKILL.md#createpayment-charge-the-stores-total):
retain `paymentCartFormat: "server-legacy"`.
Use `kit.landing(request)`'s trusted channel, never a browser channel.
Keep a stable private cart-token secret.

Record attempts before sending. Replay the same cart/payment method/order
reference/Idempotency-Key. Re-price before the first charge, never into a changed
retry. Lost answers are unknown: disable Pay, confirm that attempt.
Paid but unsaved means "Your payment went through.
Your order is still being placed", not completed. Never charge again to recover.
Keep orders unique on payment ID. Implement every
[retry safeguard](skills/relyon-checkout/SKILL.md#lost-answers-retries-and-reloads-never-two-charges).

## 5. Doctor — inspect actual findings

Set the running `STORE_PUBLIC_URL` (loopback allows HTTP/its port):

```sh
npx --no-install @relyon/store-kit doctor --url "$STORE_PUBLIC_URL"
```

Remote: HTTPS, `RELYON_PREVIEW_HOSTS` when needed. Do not publish to reach doctor.
Read
[diagnostics](skills/relyon-setup/SKILL.md#diagnose-without-overclaiming)
Read every line: reaching doctor/exit zero is not all checks passed. Fix store
failures; record credential/policy/webhook/channel/runtime blockers verbatim.
Never invent statuses or deselect required feeds to pass.
Policy sync needs merchant approval, never a checkout side effect.
Use [store policies](skills/relyon-setup/SKILL.md#store-policies-explicit-synchronization)
and [webhook setup](skills/relyon-agents/SKILL.md#agent-rules) when prerequisites
exist. Handle one-time secrets privately; never sync placeholders.

## 6. Genuine Test purchase

Use checkout's [Test purchases](skills/relyon-checkout/SKILL.md#test-purchases-end-to-end).
Buy on the store page, never a real card. Mock fields collect no card data;
processor-backed Test uses that processor's published test cards only.
Record displayed total, actual `pay_` ID, `livemode: false`, succeeded status
and exactly one saved order. Reload/retry recovers that payment/order.
Missing credentials/`processor_not_configured`? Record the blocker, never success.
Save actual `payment.order_id` (`ord_`) for fulfillment; no ID means no claim.

UCP Test checkout is implemented. ACP needs channel setup, ChatGPT credentials,
application/channel prerequisites, not promised native ChatGPT availability.
Store-hosted MCP Test hands off to the store page; Live returns
`protocol_not_available`. Separate Commerce MCP supports
checkout-link handoff, not direct charging. Relyon verifies agents and enforces
consent, restrictions and limits; never bypass refusals or change identity.

## 7. Handover and stop in Test

Use go-live's [hand-over checklist](skills/relyon-go-live/SKILL.md#hand-over-checklist).
Give commands/URLs, checks, doctor findings, payment/order evidence or blockers,
defaults, pending merchant items and checks to resume. Separate discovery,
purchase and channel approval. Never call unverified checks passed.
Agents never enable Live or promise approval. Retain all security safeguards.

## Need more?

| Need | Open only |
| --- | --- |
| Reporting | [Agent events](skills/relyon-agents/SKILL.md#agent-activity-reporting-stays-on) |
| Order recovery | [Paid but unsaved](skills/relyon-orders/SKILL.md#paid-but-the-order-is-not-saved) |
