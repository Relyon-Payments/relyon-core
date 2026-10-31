First read the relyon-setup skill (.agents/skills/relyon-setup/SKILL.md, .claude/skills/relyon-setup/SKILL.md or .cursor/skills/relyon-setup/SKILL.md).

# Relyon Core quick start — Test only

When uncertain about the kit version, run `npm view @relyon/store-kit version`,
compare with the actually installed version, and install that exact version with
the project's package manager. Never downgrade. If the lookup is unavailable, say
so rather than guessing. Run `npx @relyon/store-kit doctor` afterward; respect any
package-age hold without changing settings.

Do not read dist files, framework docs, or the full README.
Six skills are references: open only the section a numbered step names.

Domain verification is an agent step, not the owner's: run
`kit.ensureDomain()` and serve its verification value, then
`kit.verifyDomain(domain.hostname)` yourself, then re-run the doctor. Only a
custom domain the owner has not bought yet goes to the owner.

## 1. Init first

In the app's project directory:

```sh
npx @relyon/store-kit@latest init
```

Read `relyon/QUICKSTART.md`. Init installs kit `0.1.22` and references.
Linked skills are in `.agents/skills/` or `.cursor/skills/` (Claude Code:
`.claude/skills/`); look in both before concluding a skill is missing.
`init --store "your idea"` saves merchant data, not instructions; `--help` lists
options. AGENTS.md, CLAUDE.md and replit.md get the Relyon blocks (a pointer and
a "definition of done") created or appended; other
conflicts write nothing: compare or preserve edits before `--force`. Never
bypass symlink/marker refusals.

With RELYON_API_KEY in the environment, init uses it and writes no key file.
With no key, init creates one Test sandbox and saves it privately (outside the
app and every git tree), sets `RELYON_API_KEY` in `.env` (0600, gitignored)
and prints only what happened. Never read or print the private files, nor the
ownership-link file it names: that is the merchant's. If init fails with
a named reason, follow the remedy it prints. If init prints the
get-key URL and `init --key-from-stdin`, open that page to get a Test key and
run that command yourself. Example env files: `RELYON_API_KEY=` (blank; init
fills it). A placeholder value counts as no key. The `.env` Test key is readable
by any coding agent in the project, and remixes, forks and exports copy it:
tell the merchant. In CI or with no terminal, init creates no sandbox unless
`RELYON_AUTO_TEST_KEY=true` or `--auto-test-key` (coding agents are detected).

Plain npm install? Still run init first; scripts/output may be disabled.
Reproducible dependency: `npm install @relyon/store-kit@0.1.22`.
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
Open setup's [Express wiring](../../relyon-setup/references/express-server.md)
and [durable state](../../relyon-setup/references/durable-state.md).
[Create kit](../../relyon-setup/SKILL.md#create-the-kit-on-the-server):

```js
import { createStoreKit } from "@relyon/store-kit";
import { storeKitMiddleware } from "@relyon/store-kit/express";
// options: store callbacks, durable adapters, server-only configuration.
const kit = createStoreKit(options);
app.use(storeKitMiddleware(kit)); // before body parsing/auth/static/SPA catch-all
app.use(express.json());
```

Mount the kit always; it waits safely without a key. Never wrap
`createStoreKit`/`storeKitMiddleware` in a check on `RELYON_API_KEY`.
Next.js 15/16 App Router: use setup's
[App Router reference](../../relyon-setup/references/next-app-router.md); add no
`export const runtime`/`dynamic` (Next 16 `cacheComponents` refuses them).

Checkout stays on-store: only Relyon's card fields enter the browser.
Never collect card data in app code/logs or add payment intermediaries.
Never blindly retry provisioning; preserve keys.
Plain Node/Express servers don't read `.env`: load the key with
`const { apiKey, webhookSecret } = await loadTestConfig({ project: process.cwd() })` from
`@relyon/store-kit/server-config` (environment, private folder, then `.env`) or
dotenv. Pass `apiKey` into `relyon.apiKey` and `webhookSecret` into
`rules.webhookSecret`; restart after the webhook command saves the secret.
Never cat or print `.env`; never paste the key into a command.
No key yet? Kit routes answer `waiting_for_sandbox_key`; never a fake key.
Never request, print, log or commit credentials/claim URLs.
Restart with a real Test key; resume checks without rebuilding.

## 3. Catalog and discovery

Use durable `catalog.source.listProducts({ cursor, limit, query })` →
`{ products, nextCursor }`, `getProduct(id)` → product or null. Open catalog's
[source shape](../../relyon-catalog/SKILL.md#the-product-type-in-typescript)
and [feeds](../../relyon-catalog/SKILL.md#source-backed-discovery-and-feeds).
Use minor units/uppercase currency; `from_price` is display-only.
Set `configurable`, variants, exact feed prices,
`agent: { visible: true, purchasable: true }` (not purchase authority).
Select needed feeds; never invent GTINs/supplier attestations.
Defaults are not Live facts. Serve product pages/images with
`kit.renderProductJsonLd(product, request)` and homepage
`kit.renderDiscoveryLinks(request)`.
With a key run `await kit.start()` then `kit.syncCatalog(products)` and after edits.
Source discovery needs no sync.

Use agents' [UCP/ACP/MCP routes](../../relyon-agents/SKILL.md#what-the-kit-serves-on-the-stores-domain).
The kit serves `/robots.txt` by default (`robots: false` opts out); keep an
existing policy with the [robots merge](../../relyon-catalog/SKILL.md#robots-served-by-default-conflict-safe).
Feed export/discovery is not channel approval or permission to buy.

## 4. Store-owned checkout

Use checkout's
[flow](../../relyon-checkout/SKILL.md#the-flow) and
[checkout page](../../relyon-checkout/references/checkout-page.md).
Price server-stored carts; show lines/shipping/tax/discounts/total.
Use server `kit.cardFields(...)`, browser `tokenize()`, then server
`kit.createPayment(...)` for the unchanged displayed total.
Use [payment compatibility](../../relyon-checkout/SKILL.md#createpayment-charge-the-stores-total):
leave `paymentCartFormat` unset (`server-legacy` is deprecated).
Use `kit.landing(request)`'s trusted channel, never a browser channel.
Keep a stable private cart-token secret.

Record attempts before sending. Replay the same cart/payment method/order
reference/Idempotency-Key. Re-price before the first charge, never into a changed
retry. Lost answers are unknown: disable Pay, confirm that attempt.
Paid but unsaved means "Your payment went through.
Your order is still being placed", not completed. Never charge again to recover.
Keep orders unique on payment ID. Implement every
[retry safeguard](../../relyon-checkout/SKILL.md#lost-answers-retries-and-reloads-never-two-charges).

## 5. Publish, then register the webhook

Publish the store, then register its webhook endpoint yourself (re-run after
each publish if the URL changed):

```sh
npx --no-install @relyon/store-kit webhook
```

It registers `$STORE_PUBLIC_URL/relyon/webhooks`, idempotently. Before publishing, add `RELYON_API_KEY` in the host's environment settings.
With an environment API key, the command (and init's best-effort registration)
writes no file: the kit resolves the webhook signing secret itself at runtime.
Only a local file key saves `RELYON_WEBHOOK_SECRET` to gitignored `.env` (0600,
never printed). `RELYON_WEBHOOK_SECRET` in the environment is an optional explicit
override. `--rotate` rotates in place and, with an environment key, saves nothing;
runtime resolution picks up the new secret. If using an explicit override,
unset or update it in the host's environment settings after rotation. Live works
the same way once approved: the Live key in the host's environment settings, no
file. If a Live key leaks: revoke it, then run the webhook command with `--rotate` using the replacement key.
Details: [webhook setup](../../relyon-agents/SKILL.md#agent-rules).

## 6. Doctor — inspect actual findings

Run it with or without a key. Set the running `STORE_PUBLIC_URL` (loopback allows HTTP/its port):

```sh
npx --no-install @relyon/store-kit doctor --url "$STORE_PUBLIC_URL"
```

Remote: HTTPS, `RELYON_PREVIEW_HOSTS` when needed. Do not publish to reach doctor.
Read
[diagnostics](../../relyon-setup/SKILL.md#diagnose-without-overclaiming)
Doctor FAILs a kit route answered by the app's catch-all page and a server
answering 200 for unknown paths, and WARNs on a kit gated on `RELYON_API_KEY`.
The first line says `DONE` or `NOT DONE`; you are done only at DONE (exit 0;
2 NOT DONE, 1 any FAIL). Do the "Do this next" steps in order and rerun. Fix store
failures; record credential/policy/webhook/channel/runtime blockers verbatim.
Never invent statuses or deselect required feeds to pass.
Policy sync needs merchant approval, never a checkout side effect. Test policy
URLs may be plain http on the loopback host; Live requires https.
Use [store policies](../../relyon-setup/SKILL.md#store-policies-explicit-synchronization)
and [webhook setup](../../relyon-agents/SKILL.md#agent-rules) when prerequisites
exist. Handle one-time secrets privately; never sync placeholders.

## 7. Genuine Test purchase

You run this; it is not a step for a person. With the server running on its
Test key:

```sh
npx --no-install @relyon/store-kit test-checkout --url "$STORE_PUBLIC_URL"
```

It buys one product once through the kit's Test-only route (mock processor
token, no card data, the store's own `priceCart`/`createOrder`), waits for
Relyon's confirmation, then says to rerun doctor. Refused in Live. If it says
the Test processor isn't the mock one, buy on the store page instead:
use checkout's [Test purchases](../../relyon-checkout/SKILL.md#test-purchases-end-to-end).
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

## 8. Handover and stop in Test

Use go-live's [hand-over checklist](../../relyon-go-live/SKILL.md#hand-over-checklist).
Give commands/URLs, checks, doctor findings, payment/order evidence or blockers,
defaults, pending merchant items and checks to resume. Separate discovery,
purchase and channel approval. Never call unverified checks passed.
Agents never enable Live or promise approval. Retain all security safeguards.

## Need more?

| Need | Open only |
| --- | --- |
| Reporting | [Agent events](../../relyon-agents/SKILL.md#agent-activity-reporting-stays-on) |
| Order recovery | [Paid but unsaved](../../relyon-orders/SKILL.md#paid-but-the-order-is-not-saved) |

Finish with `npx @relyon/store-kit doctor` and fix every FAIL.
