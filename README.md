# Relyon Core — run npx @relyon/store-kit@latest init first

Build an AI-ready online store with any coding agent. Start in your app's directory:

```sh
npx @relyon/store-kit@latest init
```

[Relyon](https://core.getrelyon.com) payments and AI-agent commerce for a store app, in one npm
package: [`@relyon/store-kit`](https://www.npmjs.com/package/@relyon/store-kit) version
`0.1.6`, plus six skills that teach a coding agent to build the whole store on the
store's **own** server and domain.

## Start here

Read the copied `relyon/QUICKSTART.md` first, or [QUICKSTART.md](QUICKSTART.md)
here. Open a skill only when a numbered step names its section. The six skills
are references, not mandatory whole-file reading. Do not read dist files,
framework docs, or the full README to start.
The [Recommended build order](AGENTS.md#recommended-build-order) points to the
same seven-stage flow. Package managers may suppress the best-effort install
notice or disable scripts; this instruction and init work without that notice.

Tell your coding agent (Claude Code, Codex, Cursor, Replit Agent or OpenCode) your store idea and
paste this prompt, with your idea in place of `<your store idea>`:

```text
Build my store: <your store idea>. Use Relyon Core: run npx @relyon/store-kit@latest init first, then read relyon/QUICKSTART.md.
```

`npx @relyon/store-kit init` runs in the app's directory and:

- adds `@relyon/store-kit@0.1.6` (exact version) with the app's package manager,
  picked from its lockfile (npm, pnpm, Yarn or Bun), and creates a minimal private `package.json`
  if there is none;
- copies the six skills to `.agents/skills` (Codex, Replit Agent, OpenCode), `.claude/skills`
  (Claude Code) and `.cursor/skills` (Cursor);
- copies `relyon/QUICKSTART.md` and points `AGENTS.md` and `CLAUDE.md` to it;
- with `--store "<your store idea>"`, writes the idea word for word into `relyon-store-brief.md`,
  the store brief the skills read.

It never overwrites a file of yours that differs: it reports each conflict and the line to add by
hand, and writes nothing. A second run changes nothing. `--force` replaces differing skill files and
the quick start, the store brief, and only the marked Relyon block inside `AGENTS.md` and `CLAUDE.md`.
`--skip-install` skips the package install. It never asks for or prints a key.
Follow the quick start and build before requesting credentials.
On Node/POSIX, automatic Test setup requires `RELYON_PRIVATE_DIR` pointing to
an existing owner-only 0700 directory outside the app and every git tree.
Init preserves existing keys; validated Test credentials and the private claim
URL stay in `sandbox.json` (0600), atomically written without overwriting.
There is no generic host secret-manager integration. Import `loadTestConfig`
from `@relyon/store-kit/server-config` on the Node server only, and call it
with `{ project: process.cwd() }`. Use a separate private directory per store,
never in public/static roots. Repeated/concurrent init and unknown outcomes
never automatically retry creation. Claiming ownership does not gate building;
unclaimed sandboxes expire.
If safe persistence or provisioning is unavailable, keep building catalog,
pages, feeds, checkout UI, agent integration and orders with checkout
**waiting for sandbox key**. Record credential-dependent checks as pending.
Only after building, visit https://core.getrelyon.com/get-sandbox-key if still
waiting and configure `RELYON_API_KEY` through the host's secret tooling.
Restart the server and resume verification without rebuilding.
In a pnpm workspace, run from the root, it installs the kit into the one server package, or, when
it can't tell which package that is, skips the install and prints the `pnpm --filter … add`
command to run; the skills still go to the project root.

From there follow the quick start's numbered steps. Ask only for missing merchant
promises; use the centralized Test defaults without blocking the build. Secrets
belong only in host secret tooling, never chat.

| Skill | What it covers |
| --- | --- |
| `relyon-setup` | Reference: Test sandbox key, mounting on Express, Next.js or Hono, durable state and host notes. |
| `relyon-catalog` | Keep products and prices in the app: store-owned catalog callbacks, feeds, product JSON-LD, `llms.txt` and the product sitemap. |
| `relyon-checkout` | The store's own checkout page, card fields, payments that charge exactly the store's total, and agent hand-offs. |
| `relyon-agents` | Serve AI shopping agents (UCP, ACP, MCP) on the store's domain, agent rules, activity reporting and the webhook. |
| `relyon-orders` | Create each order exactly once, report fulfillment and verify Relyon webhooks. |
| `relyon-go-live` | Get a Test store ready for Live without switching to Live yourself. |

**Test only for coding agents.** Agents build and test with Test keys (`rly_test_…`), which never
move money. Live is enabled by Relyon staff after they review the store; the kit then runs Live only
with an explicit double opt-in (`relyon.live: true` and a Live key, see Test and Live below). The
skills tell agents to stop and hand over rather than switch to Live.

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

## Install for each agent

`npx @relyon/store-kit init` (above) installs the kit and the same six skills for every agent below.
The skills are in [`skills/`](skills/). Agents load them on demand when a task involves Relyon. Each
section below also names another way to install the skills.

### Claude Code

`init` writes `.claude/skills` and `CLAUDE.md`. Or install the plugin:

```text
/plugin marketplace add relyon-payments/relyon-core
/plugin install relyon@relyon
```

Or copy the skills into a project: `mkdir -p .claude/skills && cp -R relyon-core/skills/* .claude/skills/`
(use `~/.claude/skills` for every project).

### Codex

`init` writes `.agents/skills`; Codex reads `AGENTS.md` at the project root. Or copy them:

```sh
git clone --depth 1 https://github.com/relyon-payments/relyon-core.git
mkdir -p .agents/skills && cp -R relyon-core/skills/* .agents/skills/
```

### Cursor

`init` writes `.cursor/skills`; Cursor also reads `AGENTS.md` at the project root. Or copy them:

```sh
git clone --depth 1 https://github.com/relyon-payments/relyon-core.git
mkdir -p .cursor/skills && cp -R relyon-core/skills/* .cursor/skills/
```

### Replit Agent

Start with the prompt above in a new Replit App. `init` writes `.agents/skills`. Or, in the Replit
shell, from the project root:

```sh
git clone --depth 1 https://github.com/relyon-payments/relyon-core.git /tmp/relyon-core
mkdir -p .agents/skills && cp -R /tmp/relyon-core/skills/* .agents/skills/
```

### OpenCode

`init` writes `.agents/skills`, which OpenCode reads along with `AGENTS.md`. Or copy them:

```sh
git clone --depth 1 https://github.com/relyon-payments/relyon-core.git
mkdir -p .opencode/skills && cp -R relyon-core/skills/* .opencode/skills/
```

(use `~/.config/opencode/skills` for every project).

### The Relyon CLI

The `relyon` CLI (served by Relyon, not the npm registry) installs skills for every tool above in one
step. It bundles these skills; to install the repository's copy instead, pass `--source`:

```sh
git clone --depth 1 https://github.com/relyon-payments/relyon-core.git
npx -y https://core.getrelyon.com/cli/relyon.tgz agent setup --source relyon-core/skills
```

`--tools claude,cursor,codex,opencode,replit` limits the tools; `--global` installs into your home
directory instead of the project.

### Lovable and Base44

Lovable and Base44 don't load skill folders. Paste the blocks from
[`platforms/lovable/`](https://github.com/relyon-payments/relyon-core/tree/main/platforms/lovable/) or
[`platforms/base44/`](https://github.com/relyon-payments/relyon-core/tree/main/platforms/base44/) in
the Relyon Core repository into the project's knowledge or instructions only when
a quick-start step needs that section. Each block says what works on that platform: card checkout works, but
the kit's agent routes need a server on the store's own domain.

# Store kit developer reference

A server-side kit a store installs in its own app to take payments and serve AI shopping agents
through Relyon. Every agent and shopper request lands on the store's own domain. The store's server
is the only thing that calls Relyon. The only Relyon resource a shopper's browser loads is the card
fields script.

- The store keeps its catalog, cart and pricing. Relyon never prices a cart; it charges exactly the
  total the store sends.
- UCP Test checkout is implemented on the store domain. ACP Test integration is implemented
  with merchant-domain forwarding, ChatGPT credentials, application and channel prerequisites,
  not unrestricted native ChatGPT availability. Store-hosted MCP checkout works in Test by
  handing the shopper off to the store's checkout page (the order gets an `ord_` ID); in Live it
  returns `protocol_not_available`. Separate Commerce MCP supports checkout-link handoff, not direct
  charging. No Live availability or platform approval is promised.
- Card numbers never touch the store's code, server or logs.

Requires Node.js 20 or later. ESM only. Adapters for Express, Next.js (App Router) and Hono, none of
which is a runtime dependency.

## Install

```sh
npm install @relyon/store-kit@0.1.6
```

(or `pnpm add`, `yarn add`, `bun add` with the same specifier; Deno: `npm:@relyon/store-kit@0.1.6`).

In a pnpm workspace, add it to the package that runs the store's server (`pnpm --filter <server
package> add --save-exact @relyon/store-kit@0.1.6`), never the workspace root;
`npx @relyon/store-kit init` does this when the workspace has one server package and otherwise
prints that command. If a minimum release age policy holds back a new version, wait or exempt only
`@relyon/store-kit`.

Relyon's API base URL is `https://core.getrelyon.com/api`. The kit has no default: pass it as
`relyon.baseUrl`. The core kit reads no environment variables itself; the names below are the app's.
The optional Node-only server-config loader and init read `RELYON_PRIVATE_DIR`.

## Quick start

Create one kit per server process, in a server-only module:

```ts
// relyon.ts
import { createStoreKit } from "@relyon/store-kit";
import { priceCart, createOrder, catalogSource } from "./store.js"; // your app's own functions

export const kit = createStoreKit({
  relyon: {
    baseUrl: "https://core.getrelyon.com/api",
    apiKey: process.env.RELYON_API_KEY, // real Test key from server-only configuration, when available
    waitingForSandboxKey: true, // explicit Test-only build mode; a real key activates normal operation
  },
  store: {
    name: "Example Store",
    description: "Hand-made goods.",
    hostnames: ["shop.example.com"], // the store's own public hostnames, primary first
  },
  // The store prices every cart. Return one line per input line, in order,
  // with totals that add up: Relyon charges exactly `totals.total`.
  priceCart: async (lines, address, coupons, context) => priceCart(lines, address, coupons, context.shippingOptionId),
  // Called exactly once for each succeeded payment.
  createOrder: async (cart, payment) => createOrder(cart, payment),
  // The store's own catalog: listProducts + getProduct read the store's database (see "Store-owned catalog reads and feeds" below).
  catalog: { source: catalogSource },
  // Signed webhooks: `agent_rules.updated` refreshes the agent rules right away, and Relyon's
  // doctor challenge is acknowledged on this route.
  rules: { webhookSecret: process.env.RELYON_WEBHOOK_SECRET },
});

await kit.start(); // reads the agent rules, starts event reporting
// On shutdown (for example SIGTERM): await kit.stop();
```

For automatic private storage, import `loadTestConfig` from
`@relyon/store-kit/server-config` in this Node server module and spread
`await loadTestConfig({ project: process.cwd() })` into `relyon`.
Never import the loader into browser code or expose its return value.
Show `kit.checkoutStatus` on the checkout page; while it is
`"waiting for sandbox key"`, disable payment submission and card fields.
Never fabricate a credential, successful payment or order. Credential-dependent
operations refuse locally, without unauthenticated API attempts. Doctor marks
missing-key checks UNVERIFIED, never a verified connection.

Store-owned catalog callbacks (`catalog.source`) need no catalog sync: the kit's discovery routes
and feeds read them directly. `kit.syncCatalog(products)` only fills Relyon's optional dashboard
index mirror; without `catalog.source`, the legacy `llms.txt`, sitemap and MCP browse read that
mirror instead.
Relyon's index takes full URLs only: `syncCatalog` sends each product's own `url` and `images`
(a path such as `/products/mug`, or a URL on the store's hostnames, loopback or `0.0.0.0`) on
`STORE_PUBLIC_URL`, else on `https://` + the first `STORE_HOSTNAMES` entry, the same address the
feeds use. The kit's own catalog keeps your paths. With paths, a loopback-only store needs
`STORE_PUBLIC_URL` (plain http on `localhost` with the dev server's port); without it the sync stops before sending
anything and names the product and the setting to fix.

Then mount it on your server. The kit answers its own paths and passes everything else to your app.

### Express

```ts
import express from "express";
import { storeKitMiddleware } from "@relyon/store-kit/express";
import { kit } from "./relyon.js";

const app = express();
app.use(storeKitMiddleware(kit)); // before express.json() or any other body parser
app.use(express.json());
// ...your routes
app.listen(3000);
```

The kit needs the raw request body to relay signed agent requests; the middleware throws if a body
parser ran first.

### Next.js (App Router)

Create a route file for each kit path (`app/.well-known/ucp/route.ts`,
`app/.well-known/relyon-verification/route.ts`, `app/ucp/[...path]/route.ts`,
`app/acp/[...path]/route.ts`, `app/mcp/route.ts`, `app/llms.txt/route.ts`,
`app/sitemap-products.xml/route.ts` and `app/relyon/webhooks/route.ts`), each containing:

```ts
import { storeKitRouteHandlers } from "@relyon/store-kit/next";
import { kit } from "@/lib/relyon";

export const runtime = "nodejs";
export const { GET, POST, PUT, PATCH, DELETE, HEAD } = storeKitRouteHandlers(kit);
```

Import `kit` from one shared server module so every route uses the same instance.
`storeKitFetch(kit)` returns `Response | null` for code that routes Web requests itself.

### Hono

```ts
import { Hono } from "hono";
import { storeKitMiddleware } from "@relyon/store-kit/hono";
import { kit } from "./relyon.js";

const app = new Hono();
app.use(storeKitMiddleware(kit)); // before your routes
// ...your routes
export default app; // on Node, serve it with @hono/node-server
```

### Any other framework

The core works on Web `Request`/`Response`: `kit.handle(request)` returns a `Response`, or `null` for
paths the kit doesn't serve.

## Routes served on the store's domain

| Path | What it does |
| --- | --- |
| `/.well-known/ucp` | Relyon's UCP profile for this hostname. Cached for the response's `max-age`, and served stale if Relyon is down. |
| `/ucp/*`, `/acp/*` | Rebuilds the agent's exact request on the store's public `https` URL (method, lower-case headers, raw body), prices it with `priceCart` and relays it to Relyon. Returns status, headers and body unchanged. On a paid `complete`, calls `createOrder` once; until that order exists the agent gets [order pending](#order-pending-after-a-charge), never `completed`. |
| `/mcp` | Browse and search use `catalog.source` when configured, otherwise the legacy cached index mirror. Checkout tools are relayed to Relyon. In Test, checkout hands the shopper off to the store's checkout page (`continue_url`); the payment happens there and the order gets an `ord_` ID. In Live Relyon doesn't offer MCP checkout, so the tools fail closed with an in-band JSON-RPC error, before any charge. |
| `/.well-known/relyon-verification` | The store's domain verification value, as `text/plain`. |
| `/llms.txt`, `/sitemap-products.xml`, `/sitemap.xml` | With `catalog.source`, generated directly from visible store products. Without it, only products Relyon published: `upsert` adds to the mirror, `full` replaces it, and failed sync leaves it unchanged. `/sitemap.xml` is a urlset of the same product pages (`sitemapIndex: false` turns it off). llms.txt also lists the UCP profile, MCP, ACP checkout, the agent card, the selected feeds and the policy pages. |
| `/.well-known/agent-card.json` | A JSON agent card for the store's UCP, ACP and MCP endpoints and Relyon payments. AP2 checkout mandates are an optional extension (verified when presented on UCP checkout); it claims no A2A message endpoint, and MCP checkout only in Test. `agentCard: false` turns it off. |
| feeds | Selected formats only (`/feeds/openai.jsonl`, `.jsonl.gz`, `/feeds/google.xml`, `/feeds/microsoft.xml`); see below. |
| `/relyon/webhooks` | A signed `agent_rules.updated` webhook refreshes the rules cache right away. A signed `doctor.challenge` is acknowledged exactly as the contract defines (the `Relyon-Doctor-Acknowledgement` header), so Relyon's doctor can confirm the webhook route works. |

Every link these routes return uses one public origin per request, `kit.publicOrigin(request)`:
`store.publicUrl` (`STORE_PUBLIC_URL`) when set, otherwise the store's own or Test preview hostname
the request arrived on (any other Host is refused). Catalog URLs on the store's own hostnames,
including loopback and preview addresses, are moved onto that origin. `kit.renderDiscoveryLinks(request)`
returns the same endpoints as `<link>` tags for the homepage `<head>`. In Test, `store.publicUrl` must
name one of the store's hostnames (https, or http on loopback).

Paths can be changed with the `paths` option if they clash with your app's routes.

## Checking an install: `npx @relyon/store-kit doctor`

The package ships a doctor that changes nothing in the store. Run it against the store's public URL, with `RELYON_API_KEY`
already in the environment from your secrets tool:

```sh
STORE_PUBLIC_URL=https://shop.example.com npx @relyon/store-kit doctor
```

It fetches what agents and Relyon see on the store's domain, then asks Relyon's integration doctor
(`GET /v1/doctor`) for the server-side view:

| Item | Checks |
| --- | --- |
| `/.well-known/ucp` | A UCP business profile for this host. |
| `/.well-known/relyon-verification` | The domain verification value is served as text. |
| `/llms.txt`, product sitemap | Served, and the sitemap is valid XML. |
| Feeds | Each selected feed (`--feeds` / `RELYON_FEEDS`, default: those `llms.txt` links) is served and parses; a valid feed with no products is UNVERIFIED. No selected feeds is N/A. |
| `/mcp` | An MCP `initialize` answers. |
| `robots.txt` | Doesn't block the store's discovery files for the search and shopping crawlers. A missing file is PASS. |
| Relyon doctor | Each of the five checks from `GET /v1/doctor`. Without `RELYON_API_KEY` they are UNVERIFIED. `pricing` and `checkout` test an older Relyon-side flow that store-first apps don't use: in Test, when this run saw the store's UCP profile, verification file, `llms.txt` and MCP pass, they are N/A instead of FAIL. A store-first blocker Relyon reports under them ("Observed blocker" or "Unverified") still FAILs, and Live keeps every failure. |

Relyon's API base defaults to `https://core.getrelyon.com/api`; `--base-url` or `RELYON_BASE_URL`
changes it for another operator-confirmed deployment. When Relyon can't be asked, its items are
UNVERIFIED with one line naming what happened and the URL tried (without its query string or any
path part that could be a token):
Relyon's server answering with a web page instead of data (an HTML page, or a 3xx pointing at a
sign-in or protection page, with the status and only the origin it points to), a Relyon server error (5xx), a
timeout, a DNS failure, a refused connection or a blocked network. None of these is caused by the
store, and none is fixed by publishing it again. Text from Relyon's answer is printed with control
characters removed.

Every item prints PASS, FAIL, N/A or UNVERIFIED. The exit code is 1 only when an item FAILs (2 for a
usage error). Keys are read from `RELYON_API_KEY` only, never from flags, and are never printed:
anything shaped like a Relyon key is redacted from all output. `--live` (or `RELYON_LIVE=true`)
checks in Live mode with a Live key; there, a `/.well-known/ucp` profile that advertises checkout
must also publish an ES256 signing key (Relyon advertises Live checkout, with its keys, only for a
store approved for Live), or the item FAILs. The doctor never creates a shopper checkout or
submits a payment; Relyon's server diagnostic may create and clean up one temporary system-owned
checkout and sends one webhook challenge.

## Test and Live

The kit runs in **Test** mode unless Live is switched on explicitly. Test never moves money.

| `relyon.apiKey` | `relyon.live` | Result |
| --- | --- | --- |
| `rly_test_…` | absent or `false` | Test mode. |
| `rly_live_…` | `true` | Live mode, if the store URL rules below pass. |
| `rly_live_…` | absent or `false` | Refused with a `TypeError` when the kit is created. |
| `rly_test_…` | `true` | Refused with a `TypeError` when the kit is created. |

The mode never comes from the key prefix alone; `kit.livemode` says which mode the kit runs in.
Live mode also needs `store.publicUrl` (your `STORE_PUBLIC_URL`) to be an `https` URL naming one
of `store.hostnames`, and refuses `localhost`, IP-literal and development preview (`*.replit.dev`)
hostnames. Each refusal is a `TypeError` naming the problem.

```ts
export const kit = createStoreKit({
  relyon: {
    baseUrl: "https://core.getrelyon.com/api",
    apiKey: process.env.RELYON_API_KEY!,      // rly_live_… in production
    live: process.env.RELYON_LIVE === "true", // explicit opt-in
  },
  store: { name: "Example Store", hostnames: ["shop.example.com"], publicUrl: process.env.STORE_PUBLIC_URL },
  // ...
});
```

Live payments still need the store's Relyon account to be ready for Live: Relyon staff verify the
store and attach its processor account. The kit makes no assumption about which accounts can go
Live (a claimed sandbox may become Live later); it relies on Relyon's answer. While an account isn't
ready, Relyon refuses before any charge with one of these codes, and the kit fails closed: it
answers the shopper or agent with Relyon's status and code and a plain explanation, never sends a
`Retry-After`, and never resends the step or the charge.

| Code | Status | Meaning |
| --- | --- | --- |
| `processor_not_configured` | 409 | The account has no processor account for this mode yet. |
| `sandbox_test_only` | 403 | The account is a sandbox, which works in Test mode only for now. |
| `live_not_available` | 403 | Relyon doesn't offer this operation in Live mode for this store yet. |

`cardFields` and `createPayment` throw `RelyonAccountNotReadyError` (`code`, `status`,
`retryable: false`); `payAndConfirm` answers it with Relyon's status and code; relayed UCP, ACP and
MCP steps answer it in the protocol's own error format. In every case nothing was charged.

To get a Test key without an account, create a sandbox:

```sh
curl -X POST https://core.getrelyon.com/api/v1/sandboxes
```

The response carries a `restrictedKey` (`rly_test_…`, server-side only) and a one-time `claimUrl` the
merchant opens to keep the sandbox. Store the key with your platform's secret tooling, for example as
`RELYON_API_KEY`. Never put it in browser code, logs or source control.

## Store-owned catalog reads and feeds (opt in)

Existing installs continue to use the last accepted index mirror for MCP browse, `llms.txt` and
the sitemap. Nothing changes in checkout tools or `syncCatalog`. To read your store database
directly, supply **both** callbacks under `catalog.source`. In this mode the index is only a
dashboard mirror; reads do not call Relyon's index or require `syncCatalog`.

```ts
import { createStoreKit, type StoreCatalogSource } from "@relyon/store-kit";

const source: StoreCatalogSource = {
  async listProducts({ cursor, limit, query }) {
    // Implement opaque pagination in your store. `query` is optional; feeds,
    // llms.txt and sitemap omit it and must receive the COMPLETE catalog.
    const page = await storeDatabase.list({ cursor, limit, query });
    return {
      products: page.products.map(toStoreCatalogProduct),
      nextCursor: page.nextCursor ?? null, // null explicitly means complete
    };
  },
  async getProduct(id) {
    const product = await storeDatabase.get(id);
    return product ? toStoreCatalogProduct(product) : null;
  },
};

const kit = createStoreKit({
  // ... existing relyon, store, priceCart and createOrder configuration ...
  catalog: { source },
  feeds: {
    formats: ["openai", "openaiGzip", "google", "microsoft"],
    cacheSeconds: 60,
    onDiagnostic: ({ productId, format, code }) => reportCatalogIssue(productId, format, code),
  },
});
```

`storeDatabase`, `toStoreCatalogProduct`, and `reportCatalogIssue` above are your own app functions,
not kit exports. Each `StoreCatalogProduct` has the existing `StoreProduct` fields, plus optional
`feed` facts:

```ts
const product = {
  merchant_product_id: "mug-blue",
  title: "Blue mug",
  description: "Glazed ceramic mug.",
  url: "https://shop.example.com/products/mug?color=blue",
  images: ["https://images.example-cdn.com/mug-blue.jpg"],
  brand: "Actual Manufacturer",
  mpn: "MUG-BLUE", // manufacturer-assigned, never an invented identifier
  from_price: { amount: 1800, currency: "USD" }, // display-only
  availability: "in_stock",
  configurable: false, // this row is one concrete selection, not a configurable parent
  agent: { visible: true, purchasable: true },
  group_id: "mugs",
  feed: {
    exactPrice: { amount: 1800, currency: "USD" }, // minor units; explicit exact public item price
    condition: "new", // explicit new | used | refurbished; no default
    variant: { groupId: "mugs", options: { color: "blue" } },
    // Optional GOOGLE-only metadata; never inferred from visibility:
    googleCheckout: { eligible: false, channelStatus: "disconnected" },
  },
};
```

### Read-source contract and visibility

- `listProducts(StoreCatalogQuery)` returns `StoreCatalogPage`: at most the requested 500 products
  and `nextCursor: string | null`. Cursors must be nonempty and never repeat; duplicate IDs,
  malformed/incomplete pagination, callback errors or more than 100 pages fail the whole read.
  No partial feed, saved-index fallback or stale-on-error response is served.
  For a catalog changing during pagination, use a database snapshot/revision in your cursor so
  every page belongs to the same read; the kit cannot create a transaction in your database.
- `getProduct(id)` returns a product or `null`. It must return the exact requested ID.
- Callback-mode MCP browse, `llms.txt`, sitemap and feeds share the same boundary:
  **only `agent.visible === true`**, a valid stable ID, and an absolute HTTPS product URL on one of
  `store.hostnames`. Unknown visibility is hidden, even if legacy mirror entries exist.
  Bad/off-store product URLs and IDs are logged as catalog diagnostics. Feeds apply additional
  required-fact checks and expose `StoreFeedDiagnostic` (`openai`, `google`, `microsoft`).
- Credentials, nondefault ports, fragments, malformed URLs and known Relyon hosts are not valid
  product/seller links. Legitimate external image CDNs are allowed. Seller URL is the configured
  primary store origin. No Relyon product, policy or checkout links are created.
- Search passes `query` to your source, then applies the kit's existing all-words search and limit.
  Keep list/get visibility and product facts consistent in your database.
- `syncCatalog`, MCP checkout tools, pricing, relays, payment authorization, agent rules and order
  creation are unchanged. Callback visibility is a discovery decision, not payment authority.

### Files, freshness and limits

Only selected formats are claimed by the adapters; exact paths are:

| Format | Route | Content type |
| --- | --- | --- |
| `openai` | `/feeds/openai.jsonl` | `application/x-ndjson; charset=utf-8` |
| `openaiGzip` | `/feeds/openai.jsonl.gz` | `application/gzip` |
| `google` | `/feeds/google.xml` | `application/rss+xml; charset=utf-8` |
| `microsoft` | `/feeds/microsoft.xml` | `application/rss+xml; charset=utf-8` |

Feeds require `catalog.source`; mirror-only configuration cannot accidentally publish a misleading
feed. `llms.txt` links only the configured feed URLs. The gzip route is a real `.jsonl.gz` file,
**not** transparent HTTP `Content-Encoding: gzip`. No SFTP, credentials, upload, Merchant API
registration, scheduled delivery or destination activation is implemented.

OpenAI files follow the **Stable file-upload products** reference retrieved 2026-10-20. Stable is
undated; no release version is invented. They are flat records, not ACP 2026-04-17 Product/Variant
objects. All nine required fields are emitted. The existing flat mapping supplies title,
description, identifiers, images, stock and exact decimal money. `is_eligible_checkout` is false
unless Relyon reports the store's ChatGPT channel connected (`GET /v1/channels/chatgpt`, same
mode, re-read at most once a minute), the store has `policies.privacyUrl` and `policies.termsUrl`
(sent as `seller_privacy_policy` and `seller_tos`), and the product is in stock and purchasable
without restrictions or `must_ask`, with the merchant's brand and a valid GTIN or
merchant-supplied MPN. While connected, an ineligible product gets a
`chatgpt_checkout_ineligible:<reason>` diagnostic.
Unknown required facts cause omission with diagnostics; even though OpenAI treats omitted condition
as new, this kit requires explicit condition to avoid that assertion.

RSS uses the shared Google mapping, with Microsoft's documented common attributes and stock
spelling (`in stock`, `out of stock`) and exact two-decimal prices for its file. No
Microsoft-specific UCP/checkout attributes are claimed. Both XML files require known condition and
an assigned valid GTIN or brand+MPN, or an explicit custom-goods attestation (below);
`identifier_exists=false` is never invented. Microsoft price limit is 10 million major units and
brand limit is 10 words. This is a conservative discovery
subset, **not** a guarantee of acceptance for every country/category: configure tax/shipping and
category-specific legal requirements separately before using a destination.

#### Own-brand and custom goods

A store selling goods under its own label (print on demand, for example) is the brand owner, and
its own SKU is the product's MPN. Say so per product with `feed.ownBrand: true`, and set the
product's `brand` and `mpn` yourself: the kit never takes the brand from the store name or the MPN
from a product ID. Made-to-order goods with no GTIN or MPN at all use `feed.customGoods: true`,
the only way the kit emits `identifier_exists=false`.

Own-brand goods made to order (no GTIN, but your own part number) can say both:
`feed.customGoods: true` and `feed.ownBrand: true`, with the product's `brand` and `mpn`. Since
0.1.1 that is an own-brand item: Google and Microsoft get the brand and your MPN, never
`identifier_exists=false`, and there is no conflict. Product JSON-LD and the ChatGPT file carry the
same brand and MPN, so readiness scans see a complete identifier everywhere the kit serves one.

```ts
// A print-on-demand poster sold under the store's own label.
const poster = {
  merchant_product_id: "poster-sunrise-a2",
  title: "Sunrise poster, A2",
  description: "Giclée print on 200 gsm matte paper.",
  url: "https://prints.example.com/products/sunrise?size=a2",
  images: ["https://prints.example.com/img/sunrise-a2.jpg"],
  brand: "Lumen Prints",      // the store's own brand, set explicitly
  mpn: "LP-SUNRISE-A2",       // the store's own SKU
  availability: "in_stock",
  agent: { visible: true, purchasable: true },
  feed: { exactPrice: { amount: 3200, currency: "USD" }, condition: "new", ownBrand: true },
};

// A one-off custom portrait: no GTIN or MPN exists.
const portrait = { ...portraitFacts, feed: { exactPrice, condition: "new", customGoods: true } };
```

Google and Microsoft diagnostics explain what's missing or contradictory:

| Code | Meaning |
| --- | --- |
| `assigned_identifier_missing` | No valid GTIN, no brand+MPN and no custom-goods attestation. |
| `own_brand_mpn_missing` | `ownBrand: true` without the product's `mpn` (your SKU). |
| `identifier_attestation_conflict` | `ownBrand` and `customGoods` together with a GTIN, `customGoods` with a GTIN or MPN (without `ownBrand`), or an attestation that isn't `true`/`false`. |
| `unknown_required_text` | No `brand` on the product (with or without `ownBrand`). |

Brand+MPN without `ownBrand` is still emitted as before, as the manufacturer's identifiers. The
OpenAI file is unchanged by these attestations.

#### Apparel: age group and gender

Google and Microsoft want `age_group` and `gender` on apparel. Set them per product as
`feed.ageGroup` (`newborn`, `infant`, `toddler`, `kids` or `adult`) and `feed.gender` (`male`,
`female` or `unisex`); they are emitted as `age_group` and `gender` in both XML files, for any item
that has them. The ChatGPT file doesn't carry them.

An item is **apparel** only when its product `category` is Google's "Apparel & Accessories": the
category path's first level is `Apparel & Accessories` (any case; `&amp;` is accepted), or it is that
category's Google ID `166` (alone or as `166 > …`). Numeric IDs of subcategories aren't recognized,
so give the full path (for example `Apparel & Accessories > Clothing > Shirts & Tops`). Titles,
brands and other category names never make an item apparel. `isApparelCategory(category)` applies
the same rule.

An apparel item without a value gets a documented default (`APPAREL_DEFAULTS`: `adult` and
`unisex`), with a diagnostic per defaulted field so you can see it. A value that isn't one of the
listed ones is omitted, never guessed or defaulted:

| Code | Meaning |
| --- | --- |
| `age_group_defaulted` | Apparel without `ageGroup`; `adult` was emitted. |
| `gender_defaulted` | Apparel without `gender`; `unisex` was emitted. |
| `invalid_age_group_omitted` | `ageGroup` isn't one of the five values; no `age_group` was emitted. |
| `invalid_gender_omitted` | `gender` isn't one of the three values; no `gender` was emitted. |

These diagnostics are reported (for Google and Microsoft) only for items the XML files include; the
item itself is still emitted.

```ts
const tee = {
  // ...
  category: "Apparel & Accessories > Clothing > Shirts & Tops",
  feed: { exactPrice, condition: "new", ageGroup: "adult", gender: "unisex" },
};
```

`feed.exactPrice` uses safe nonnegative integer **minor units**, not floating point dollars. It must
be positive for these formats. `from_price` alone never becomes exact, and configurable parent rows
are omitted even if given `exactPrice`. A selected variant must be non-configurable, with its own
ID, URL, price and stock; supply `variant.groupId` different from the item ID and nonempty selected
options. A feed group ID is 1–50 letters, digits, `.`, `_`, `:` or `-`: a valid `variant.groupId`
is used unchanged, and a longer or unusual one (a long product handle) becomes a sanitized prefix
of at most 37 characters, `-` and 12 hex digits of its SHA-256, the same on every run and in
every feed (`feedGroupId(groupId)` returns it). Two different store group IDs that would share a
feed group are omitted with a diagnostic, never merged. Group option names and selections must be
consistent and unique. OpenAI accepts arbitrary
selected option names; this XML subset supports `color`, `size`, `material`, `pattern` only, and
omits unsupported variant groups with diagnostics. Preorder/backorder are emitted in OpenAI but
omitted in RSS (availability dates are not supported here). Unknown stock is omitted, not guessed.

Supported currencies and precisions are exported as `currencyPrecision`: USD, EUR, GBP, CAD, AUD,
NZD, CHF, CNY, HKD, SGD, INR, BRL, MXN, ZAR, SEK, NOK, DKK, PLN, AED, SAR, ILS (2 digits);
KRW, JPY, CLP, VND (0); BHD, KWD, OMR, JOD, TND (3). RSS never rounds a three-decimal amount;
it may drop a trailing zero exactly, otherwise it omits the item. Currency conversion is never done.

Google XML may carry the documented `g:native_commerce/g:checkout_eligibility` only with explicit
`feed.googleCheckout` inputs. It reuses the Merchant Center sync decision (eligible + new +
connected), additionally requires in-stock/USD for this subset, and is absent from the Microsoft
file. No checkout URLs, policies, digital-item settings or unsupported metadata are fabricated.
The kit's own contract-gap notes and pinned feed fixtures are kept with Relyon's source and are
not part of this package.

Default internal cache is **60 seconds**, configurable from 0 to 300. A change to price, stock or
visibility is visible on the next request at/after expiry, without `syncCatalog`. Callbacks are
re-read then; errors answer 503 (`no-store`, `Retry-After: 5`), never stale hidden products. Each
process maintains its own finite cache. HTTP responses require revalidation (`max-age=0,
must-revalidate`) so downstream cache time does not extend the internal interval.

GET/HEAD, per-representation ETags, weak `If-None-Match`, `*`, and date-only conditional reads are
supported. ETags take precedence over dates. HEAD has identical metadata and no body; 304 has no
body. Last-Modified is withheld when second-resolution dates could produce a false 304 after a
same-second change. Every request reaching a configured feed route—including cached, HEAD, 304,
refused and failed reads—uses the sitemap's `feed_read` event queue and existing agent rules.
An upstream cache that never contacts your app cannot be observed by the kit.
Callback-backed sitemap/llms reads revalidate on every read; mirror-mode caching is unchanged.

The default text seam accepts **plain text**, removes illegal XML/control characters and enforces
length limits. HTML-like required text is omitted. If your catalog contains HTML, pass a trusted
pure sanitizer as `feeds.normalizeText(value, max)`; its result is still filtered. The API keeps
its existing sanitizer at its own boundary, not in the kit.

Public additive exports: `StoreCatalogSource`, `StoreCatalogQuery`, `StoreCatalogPage`,
`StoreCatalogProduct`, `StoreFeedFacts`, `StoreCatalogReads`, `CatalogReadDiagnostic`,
`StoreFeedOptions`, `StoreFeedFormat`, `StoreFeedMerchant`, `StoreFeedGeneration`,
`StoreFeedDiagnostic`, `generateStoreFeeds`, `StoreFeedReads`, `STORE_FEED_PATHS`, `feedPlainText`,
`storeDomainUrl`; and dependency-neutral `mapFeedRows`, `FeedRow`, `FeedMerchant`, `FeedGeneration`,
`FeedDiagnostic`, `FeedMappingFormat`, `StoreMappingProfile`, `IdentifierAttestation`, `GoogleItem`, `SerializeHooks`,
`googleNativeCommerce`, `FeedText`, `decimalMoney`, `currencyPrecision`, `validGtin`, `publicUrl`,
`feedXml`, `feedTag`, `scriptSafeJson`. Existing helpers/types and their signatures remain valid;
`llmsTxt` only gains an optional fifth argument containing configured feed URLs and an optional
sixth with the [store policies](#store-policies-on-the-stores-own-surfaces).

## Merge-safe robots helpers (opt in)

The kit does **not** claim `/robots.txt` by default. Keep your existing SEO/plugin integration.
Use `aiRobotsSection({ origin, sitemapPath?, llmsPath? })` for a new file or
`mergeAiRobots(existingText, options)` for an existing file. The result is
`{ content, changed, conflicts }`; inspect `RobotsConflict` entries (line, code, action message).
On **any** conflict the original text is returned byte-for-byte, without partial additions.
`RobotsOptions`, `RobotsMerge` and `AI_DISCOVERY_CRAWLERS` are also exported.

```ts
const result = mergeAiRobots(await readExistingRobots(), {
  origin: "https://shop.example.com",
});
if (result.conflicts.length) notifyMerchant(result.conflicts);
else if (result.changed) await saveRobots(result.content); // only your app chooses to persist
```

The section allows vendor-verified **search/shopping/user-fetch** tokens: OAI-SearchBot,
ChatGPT-User, Claude-SearchBot, Claude-User, PerplexityBot, Perplexity-User, Googlebot,
Storebot-Google and bingbot. It does not add GPTBot, ClaudeBot or Google-Extended. It adds a product
`Sitemap:` and a **comment** linking `llms.txt`, not a falsely standardized robots directive.

Merging preserves existing text, mixed case, user-agent groups, wildcard private-path rules,
allow exceptions and crawl delays. Any newly introduced specific-agent group inherits all wildcard
rules, preventing accidental private-path access. Explicit crawler disallows, wildcard blocks on
discovery, unknown directives or unfinished groups are reported for merchant resolution rather
than overridden. Repeated merges are idempotent; sitemap/link additions are deduplicated.
No new checkout disallow is created; existing restrictions are never removed.

If you intentionally want the kit to serve the route, configure
`robots: { existing: () => readExistingRobots(), onConflict: conflicts => notifyMerchant(conflicts) }`.
Even an empty existing file must be explicitly provided. On conflict it serves that original file
with status 200 and `X-Relyon-Robots-Conflicts`, logs/calls the conflict hook, and makes no changes.
It does not return a 4xx robots response that crawlers might interpret as permission to crawl.

Robots names are not verified identities. User-triggered fetches (notably ChatGPT-User and
Perplexity-User) may ignore robots. These helpers never change payment authorization, Relyon
agent rules, signature verification or WAF configuration and grant no payment permissions.

## Self-serve setup: domain and store policies

Agent checkout on a hostname needs that hostname verified as the store's domain, and agent channels
need complete store policies. Both can be done with the store's own Test key, with no Relyon admin
step. A sandbox key has every scope it needs. A restricted key needs `catalog:read` for the
verification file and the reads below, and `catalog:write` to add or verify a domain or replace the
policies. Relyon doesn't offer domain operations to Live keys yet: they answer `live_not_available`
(see [Test and Live](#test-and-live)).

```ts
// Once, from a setup script or an admin action in the store (server side only).
const domain = await kit.ensureDomain();          // first of store.hostnames; adds it if missing
// The kit now serves domain.verification.record_value at /.well-known/relyon-verification.
const checked = await kit.verifyDomain();         // Relyon fetches that file (or the TXT record)
if (checked.status !== "verified") console.warn("not verified yet:", checked.failure_reason);

const policies = await kit.replaceStorePolicies({
  returns: { window_days: 30, cost: "free", policy_url: "https://candles.example/returns" },
  support: { url: "https://candles.example/help" },
});
if (!policies.complete) console.warn("store policies still missing:", policies.missing);
```

| Method | What it does |
| --- | --- |
| `ensureDomain(hostname?)` | Returns this store's domain for `hostname` (default: the first of `store.hostnames`), adding it when Relyon doesn't have it. Retries never add a second domain. After adding, the verification file serves the new value at once. |
| `verifyDomain(hostname?)` | Ensures the domain, then asks Relyon to check it now. A domain that is already verified is returned without a new check. A `failed` domain stays served, so fixing the problem and calling this again can pass. |
| `storePolicies()` | Reads the store policies, with `complete` and `missing`. |
| `replaceStorePolicies(input)` | Replaces the whole policy document (fields you leave out are cleared). |

The domain helpers only accept the store's own configured `store.hostnames` and throw a `TypeError`
for anything else. Ownership is proven only by Relyon's own check of the file or the
`_relyon.<hostname>` TXT record (`verification.record_name`); a hostname another store has already verified answers
`hostname_in_use` and is never taken over. Adding and checking domains is rate limited (a `429`
with `Retry-After`), and a key can add at most 10 domains to a store. The returned domain never
includes the legacy routing secret.

`ensureDomain`, `verifyDomain`, `storePolicies` and `replaceStorePolicies` ship in 0.1.0.

## Store policies on the store's own surfaces

The `policies` option tells agents where the store's own policy pages are and what its returns
terms are:

```ts
createStoreKit({
  // ...
  policies: {
    shippingUrl: "https://shop.example.com/shipping",
    privacyUrl: "https://shop.example.com/privacy",
    termsUrl: "https://shop.example.com/terms",
    contactUrl: "https://shop.example.com/contact",
    returns: { windowDays: 30, scope: "defective_only", policyUrl: "https://shop.example.com/returns" },
  },
});
```

All URLs must be absolute `https` URLs without credentials; `returns.windowDays` is 1–365 for
`returns.scope: "all"`, 0–365 for `"defective_only"` (only items that arrive defective or
damaged), or zero for `"none"` (no returns). A zero-day defective-only policy does not promise
a reporting period; the shopper must consult the linked policy and conditions.
Optional `returns.conditions` is plain text, at most 2000 characters, without markup.
A bad value is a `TypeError` when the kit is created. The kit publishes them
only where the protocols already carry policies:

- **UCP and ACP checkout `links`.** UCP gets `privacy_policy`, `terms_of_service`, `refund_policy`
  (the returns page) and `shipping_policy`; native ACP (with `capabilities.payment`) gets
  `terms_of_use`, `privacy_policy`, `return_policy`, `shipping_policy` and `contact_us`.
  The existing legacy relay shape retains `seller_shop_policies` (the returns page, else the
  shipping page) for compatibility. A link Relyon already sent for
  a type is kept. Refusals and bodies Relyon signed are never changed.
- **`llms.txt`**, in a `## Policies` section.
- **Product JSON-LD** (`kit.productJsonLd` / `kit.renderProductJsonLd`): `offers.hasMerchantReturnPolicy`.
  `scope: "all"` is a finite return window of `windowDays`; `"defective_only"` is
  `MerchantReturnNotPermitted` with the defect window in its description.

The kit never states a return cost in local discovery, and a defective-only policy is never
phrased as an ordinary "N-day returns" window. `"none"` is never a finite return window.
Construction and reads never synchronize policies. Explicitly call
`kit.replaceStorePolicies(input)` to save them to Relyon:

```ts
await kit.replaceStorePolicies({
  returns: { window_days: 30, cost: "buyer_pays" },
  support: { email: "help@example.com" },
  consumer_notices: [{ text: "Made to order." }],
});
```

On that explicit replacement, local `shippingUrl`, `privacyUrl` and `termsUrl` map to
`shipping_policy_url`, `privacy_policy_url` and `terms_url`; `contactUrl` maps to `support.url`.
The local return scope, conditions and policy URL fill omitted input fields. Caller values win,
including the required `window_days` and `cost`; support email/phone, fixed-cost amounts and
notices are preserved. If local returns exist but the input has no return cost, the kit throws
before sending anything: it never assumes free returns. For fixed costs, include `cost_amount`.
Without a local `policies` option the existing method sends the input unchanged. This remains
a whole-document replacement: fields absent from both input and local options are cleared.

## Production: durable state

The defaults keep checkout state, agent events and the catalog in memory, in one process. That is
fine for a local Test run, not for production or more than one server process. Plug in:

- `state`: a `CheckoutStateStore` (`get`, `byAgentCheckoutId`, `put`, `claimOrder`, `releaseOrder`,
  `completeOrder`, `orderClaimStatus`). Its order claim makes `createOrder` run once per payment
  across processes:
  - `claimOrder(paymentId, { claimId, now, leaseMs })` must be one atomic step. It returns true
    (this caller creates the order) when the payment has no claim, or when its claim was never
    marked created and is at least `leaseMs` old: the server that held it may have died while
    creating the order. It returns false while another claim holds, and always once the claim is
    marked created.
  - `completeOrder(paymentId)` marks the claim done after `createOrder` returned. A claim marked
    done never expires and is never given back.
  - `releaseOrder(paymentId, claimId)` gives the claim back after `createOrder` threw, only while
    `claimId` still holds it.
  - `orderClaimStatus(paymentId, now)` says where the claim stands: `"created"`, `"creating"` (an
    unexpired claim), or `null` (no claim, or its lease passed).

  The lease is `orderClaimLeaseMs` (default 2 minutes); make it longer than `createOrder` ever
  takes. `MemoryCheckoutStateStore` implements all of this for one process. A store written
  before `completeOrder` and `orderClaimStatus` existed still works, but its claims never expire
  and a WebMCP checkout whose order another request is creating stays order pending (below).
  Without `orderClaimStatus` the kit also can't tell an order another request is creating from one
  that exists, so UCP, ACP and `payAndConfirm` answer order pending only when `createOrder` threw
  (see [Order pending after a charge](#order-pending-after-a-charge)). Implement it for honest
  order-pending answers.

  `put` must save every field of the `CheckoutRecord` and `get` must return it unchanged,
  including `orderPending` (set while a checkout's payment succeeded and its order isn't created
  yet). A store that drops it shows Relyon's `completed` on reads of an order-pending checkout,
  and its refusals on that checkout are only charge-neutral instead of order pending. The same
  goes for the optional `handoffToken` (the `relyon_cart` token of the latest hand-off link Relyon
  issued for the checkout): save and return it unchanged. A store that drops it lands Relyon's
  hand-off links as normal, unattributed checkouts.

  A store running more than one server process should also implement
  `claimHandoffToken(tokenDigest, { now, expiresAt })`: one atomic step that returns true exactly
  once per digest until `expiresAt` (a unique insert in a shared table is enough). The kit passes
  a SHA-256 digest, never the token. Without it each process lets a Relyon hand-off token land
  attributed once, so a replay on another server can still be attributed.
  `MemoryCheckoutStateStore` implements it.

  Also implement `setHandoffToken(protocol, sessionId, token, { expectedToken, expectedInputs, now })`:
  one atomic compare-and-set that sets the record's `handoffToken` (and `handoffTokenAt` and
  `updatedAt`, both to `now`) only while its
  `handoffToken` and `inputs` (as canonical JSON) still equal the expected ones, returning whether it
  wrote. The kit uses it for a newer link that a read or cancel answer carries, so a cart update
  saved meanwhile is never overwritten by the read's older snapshot. Without it the kit records
  links only from creates and updates, and such a link lands unattributed (with a warning).
  `MemoryCheckoutStateStore` implements it.

  Optionally implement `getStorePayment(key)` and `putStorePayment(key, record)`: the
  store-payment memory. `payAndConfirm` saves `state: "charging"` right before it sends the
  payment, `"order_pending"` (with the payment ID) as soon as it succeeded and before `createOrder`
  runs, then `"order_created"`, or `"declined"`. Each record is saved under the payment's
  Idempotency-Key (one per order reference and attempt) and also under `order_reference:<ref>`
  and, for a hand-off, `agent_checkout:<id>`, so a new attempt or another order reference for an
  already-paid checkout is recognized. Records carry `orderReference`, `attempt` and `handoff`;
  save them unchanged. What `payAndConfirm` answers from them is in
  [Order pending after a charge](#order-pending-after-a-charge). `MemoryCheckoutStateStore`
  implements it (in one process, bounded). Without it the kit remembers in the process only.

  The memory never forgets a charge it knows about. The kit keeps its own copy of every record it
  wrote in the process and merges it with the shared one by rank (`declined` < `charging` <
  `order_pending` < `order_created`, and a record with a payment ID over one without): a failed
  shared read or save, or a stale shared record, never turns a known `order_pending` back into
  `charging` or into "no record". Only the same attempt's own outcome replaces its `charging`
  record, so a later decline can't hide an earlier attempt's charge.

  Make `createOrder` idempotent on the payment id (for example a unique constraint on it). The kit
  calls it once more for the same payment when:
  - it throws after its order was saved (say, a later step failed): the claim is given back, so
    the next replay or `complete` calls it again right away;
  - a server dies after `createOrder` saved the order and before the claim was marked created, or
    `completeOrder` fails and the checkout's completion isn't stored either: another request
    takes the claim over once the lease passes;
  - it runs longer than the lease: another request may take the claim over while it still runs.
- `events: { store }`: an `AgentEventStore` (`append`, `peek`, `remove`, `size`), so queued agent
  events survive a restart.
- `catalog: { persistence }`: a `CatalogPersistence`, so discovery files survive a restart.
- `webmcp: { store }`: a `WebMcpCheckoutStore` with the atomic approval claim, so an approval pays
  at most once across servers. See [Running on several servers](#running-on-several-servers).

## Checkout helpers

- **`kit.cardFields({ amount, currency, orderReference })`:** creates a payment session and returns
  the `<relyon-card-fields>` script tag and element for the store's own checkout page.
- **`kit.createPayment({ cart, paymentMethod, orderReference, agentCheckoutId? })`:** charges
  `cart.totals.total` with an `Idempotency-Key` derived from the order reference, then creates the
  order once.
  - A `202 pending` reply, timeout, network failure, server error or unusable reply throws
    **`PaymentOutcomeUnknownError`**: a charge may have succeeded. Save its `orderReference` and
    `idempotencyKey`, keep the original payment request, and reconcile or retry that same request.
    Never create a replacement order or key, and never tell the shopper nothing was charged.
  - The error's `reason` distinguishes pending from transport failures; `payment` holds the pending
    reply when there is one. No order is created until a confirmed success.
  - **`PaymentUnavailableError`** is only for card-field session creation, which cannot charge.
  - For a succeeded payment, the result's `orderStatus` says what happened to the order:
    `created` (by this call), `already_created` (by an earlier call), `in_progress` (another
    request is creating it, or the state store can't say) or `failed` (`createOrder` threw).
    `orderCreated` is true only for `created`.
  - On `already_created` the result has the existing order when the kit was given the optional
    **`findOrderByPaymentId(paymentId)`**: a read-only lookup by the payment ID `createOrder`
    received, returning the order or `null`. It is called only then; if it throws, the error is
    logged and `order` is left out. Without it, look the order up in the app by `payment.id`.
- **`kit.payAndConfirm(input)`:** for the store's own pay endpoint. Runs `createPayment` and
  answers the order confirmation (`200`), the decline (`402`), a refusal (`403`/`503` for the
  kit's rule gate, or Relyon's own status and code), an unknown outcome (`202`, `payment_outcome:
  "unknown"`) or order pending (`202`, below). It never throws: an unexpected error after the
  payment was sent answers the unknown outcome; one before it was sent answers `500`
  `internal_error` with charge-neutral wording. The charge rules are in
  [Order pending after a charge](#order-pending-after-a-charge).
- **`kit.landing(request)`:** verifies a `relyon_cart` token, re-prices the cart through `priceCart`
  and returns the attribution. It accepts the kit's own tokens (`kit.handoffUrl`, or your
  `cartTokens` resolver) and the hand-off links Relyon issues in Test
  (`https://<verified host>/checkout?relyon_cart=rc_test_…`, in ACP and UCP `continue_url`, messages
  and MCP tool results), and, in Live mode, the ones Relyon issues in Live (`rc_live_…`, see
  [Live hand-off links](#live-hand-off-links)). Test links carry no signature the kit can check, so it verifies them
  locally: a Relyon-issued token is accepted only when it is exactly the latest one the kit saw
  Relyon issue for that agent checkout in a relayed answer (a link in the answer counts only when
  its token names that checkout), names the request's store hostname and hasn't expired on the
  kit's clock, and only the first time it lands: a replayed Relyon token, even a simultaneous one,
  is a normal checkout. The request's store hostname is its URL host when that is one of
  `store.hostnames`, else the first `X-Forwarded-Host` value when that is; a request that names
  none (for example an internal `127.0.0.1` URL behind a front server that rewrites `Host` and sends no
  `X-Forwarded-Host`) lands Relyon tokens unattributed and logs a warning. The cart and the
  attribution always come from the kit's own record of the checkout, never from the token. A
  tampered, forged, expired, superseded (the cart changed since), replayed or unknown token, or
  one for another hostname, lands as a normal, unattributed checkout. A Relyon-shaped token the
  kit doesn't recognize (not one it recorded, or for a hostname it doesn't serve) still reaches a
  custom `cartTokens` resolver; one it refused (replayed, wrong request host, expired) never does.
  The kit's own sealed tokens stay reusable until they expire. The `handoff_landing` event names
  the agent checkout and never carries the token (no `cart_token`, and agent events drop
  `relyon_cart` from their URLs); kit log lines redact cart tokens. When this store's
  `payAndConfirm` already paid (or may have paid) that agent checkout, a link Relyon issues for it
  later lands with `alreadyPaid: { orderPending, paymentId }`, and any refusal message it returns
  never says nothing was charged (`LANDING_PAID_NOTE` is used instead). Show the order status
  rather than a new card form.
- <a id="live-hand-off-links"></a>**Live hand-off links (0.1.1).** In Live mode (a Live key and
  `relyon.live: true`), Relyon's hand-off links carry `rc_live_` tokens: compact ES256 JWS tokens
  (`typ: "rc-live+jws"`) whose audience is the store's verified Live hostname. The kit accepts one
  only when all of these hold: its header and payload have exactly the expected fields (and it is
  at most 2000 characters); it names one of `store.hostnames` and the request's store hostname; it
  hasn't expired; it is exactly (compared in constant time) the latest token the kit saw in a
  relayed Live answer for that agent checkout; its signature verifies against the Live signing keys
  in the store's Relyon UCP profile (cached for the profile's lifetime, refetched once when the key
  ID is unknown, never taken from an expired copy); and it lands for the first time. The cart and
  attribution always come from the kit's own record. Which environment a token belongs to comes
  from the kit's mode, never from the token's prefix: a Relyon Test link (`rc_test_<payload>.<MAC>`)
  in a Live kit, an `rc_live_` link in a Test kit, and a tampered, unsigned, expired, replayed or
  other-store token are all refused (an unattributed checkout, never handed to a `cartTokens`
  resolver). The kit's own sealed links from `kit.handoffUrl` share the `rc_test_` prefix but have
  no dot; they still reach your `cartTokens` resolver in either mode. They carry no environment, so
  don't give a Test kit and a Live kit the same explicit `cartTokens.secret` (the default secret
  comes from the API key, which differs). If the Live token is superseded by a cart update, or
  expires, while its keys are being read, it is refused. Test behaviour is unchanged. Reading the
  same checkout again reuses its hand-off link rather than minting a new one, in Test and Live. A
  Live kit whose state store has no `claimHandoffToken` logs a warning at start: single use then
  holds per process only.
- **`kit.handoffUrl(protocol, sessionId)`:** mints a signed, expiring link that hands an agent's
  checkout to the store's own checkout page.
- **`kit.renderProductJsonLd(product: StoreProduct, request?: Request): string`:** a `<script type="application/ld+json">`
  tag with schema.org `Product` JSON-LD for product pages (name, price, currency, availability, brand,
  `sku`, and `gtin` or `mpn` only when supplied), escaped for HTML, using the kit's configured
  `policies`. With the page's `request`, its URLs and images use `kit.publicOrigin(request)`.
  `kit.productJsonLd(product, request)` returns the object instead.
- **`kit.renderProductListJsonLd(products, request?)`** / **`kit.productListJsonLd(products, request?)`:**
  the homepage product list as one schema.org `ItemList` of `Product` entries (at most 100).
- **`kit.renderDiscoveryLinks(request)`** / **`kit.discoveryLinks(request)`:** `<link>` tags (or objects)
  for the homepage `<head>`: llms.txt, the UCP profile, MCP, ACP checkout, the agent card, the product
  sitemap and every selected feed, on the public origin.
- **`kit.publicOrigin(request)`:** the store's public origin for this request (see the routes table).
- **`kit.redirect(request, path, { headers? })`:** a `303 See Other` to `path` on the store's own
  origin for this request, for the app's own routes (a cart form, "Buy now", after a POST). `path`
  starts with a single `/`, such as `/checkout?product=mug`; a full URL or `//host` throws a
  TypeError. Build every Location with it, never from `request.url`: on a dev server bound to `0.0.0.0`
  or `[::]` (`next dev --hostname 0.0.0.0`), `request.url` names that bind address, which no
  browser can open. In Test the kit then reads the browser's Host header, but only when it names
  loopback or one of the store's hostnames; an unknown Host falls back to the primary hostname.
- **`kit.syncStorePolicies({ confirm?, returns? })`:** compares the `policies` option with Relyon's
  stored store policies and returns `{ changes, needs, applied }`. Without `confirm: true` it changes
  nothing; show the plan to the merchant and call again with `confirm: true` once they agree. It never
  infers free returns or a return cost. The package also
  exports the stand-alone helpers `renderProductJsonLd(product: StoreProduct, policies?:
  StorePolicyOptions): string` and `productJsonLd(product, policies?)`; without `policies` they
  leave out the return-policy data.
- **`kit.reportFulfillment(orderId, update)`:** reports shipped, delivered and canceled status.

Throw `new StoreRefusal(code, message, status)` from `priceCart` to turn an agent away; it gets the
refusal in its protocol's error format. Word the message about the cart only ("Cedar candle is out
of stock"), and never say whether anything was charged: the same `priceCart` runs on steps for
checkouts that are already paid. The kit removes "nothing was charged"-style sentences on those
steps (see [Order pending after a charge](#order-pending-after-a-charge)), but that removal is a
safety net for common English wording, not a guarantee for every phrasing or language.

## Order pending after a charge

When a payment succeeded but the store's order isn't created yet (`createOrder` threw, or another
request is still creating it), the answer says the payment went through and the order is being
placed. It never says `completed`, carries no order or order confirmation, and never says nothing
was charged. Sending the same request again finishes the order through the order claim, exactly
once, without a new charge.

- **UCP and ACP `complete`:** Relyon's checkout body with `status: "complete_in_progress"`, no
  `order`, and an info message saying the payment went through and the order is being placed:
  resend `complete` or read the checkout, and don't start a new checkout or re-enter payment.
  UCP's message has `code: "order_pending"`. ACP's `MessageInfo` has no code, so on both
  protocols the response headers carry it: `Order-Status: pending` and `Payment-Status:
  succeeded`. HTTP `200` with `Retry-After: 5`. Relyon's headers that describe its body bytes
  (`Content-Length`, `Content-Digest`, `Repr-Digest`, `Digest`, `Content-MD5`, `ETag`, and a
  signature covering a digest) are dropped, and so is UCP's `ap2.merchant_authorization` (a
  detached signature over Relyon's body, which the kit can't re-sign). ACP's published complete response
  (`CheckoutSessionWithOrder`) requires an `order`; this answer has none, so it is a
  `CheckoutSession`.
- **Reads:** the kit remembers per checkout that its order is pending (`CheckoutRecord.orderPending`).
  A `GET` of that checkout shows the same order-pending answer instead of Relyon's `completed`, until
  the order claim is marked created, from any server sharing the state store. A read never creates
  the order. If the state store refuses to save the marker, the server that answered keeps it in
  memory (its reads stay order pending) and saves it on a later read once the store accepts writes;
  other servers see the marker only after that save.
- **Resending `complete`:** goes through Relyon as usual (its principal binding and signature checks
  apply, and it replays the payment rather than charging again), then the kit creates the order
  through the order claim and relays Relyon's `completed` answer. When resends race, one creates the
  order and the others answer order pending until it's done.
- **When the kit refuses a step on an order-pending checkout itself** (agent checkout paused, the
  agent refused by a rule, Relyon unreachable or answering with a Problem, `priceCart` throwing,
  `cart_changed` on a resend, or an unexpected error such as the state store failing), it has no
  checkout body from Relyon to rewrite. UCP and ACP then answer an `order_pending` error, built
  like the unknown-outcome answer: HTTP `503` with `Retry-After: 5`, `Payment-Status: succeeded`,
  `Order-Status: pending` and `Link: <store's own checkout URL>; rel="status"`. UCP's body is
  `{ code: "order_pending", content, payment_status: "succeeded", status_url }`; ACP's is the
  Error `{ type: "processing_error", code: "order_pending", message }`. MCP answers the in-band
  `order_pending` JSON-RPC error below. The kit never calls Relyon's payment endpoint or
  `createOrder` while it refuses; once the pause lifts or Relyon is back, resending `complete`
  finishes the order once with no new charge. If the store's prices changed since the charge,
  Relyon answers the resend `cart_changed` and the kit keeps answering order pending: the order
  can't be finished through the agent until the cart is priced as it was charged.
- **Other existing checkouts:** a refusal or unexpected error on a step that names a checkout (get,
  update, complete or cancel with its ID) that the kit can't show is order pending (completed with
  its order, say, or a state store without the marker) uses charge-neutral wording: "Read this
  checkout for its status before trying again; don't start a new checkout" (MCP: "Call
  get_checkout…"). A refusal's own message (a `StoreRefusal` from `priceCart`, say) on such a step
  keeps its text without any "nothing was charged" sentence, and the note is appended. Only
  `create` says nothing was charged. An unexpected failure on a UCP, ACP or
  MCP checkout route answers in the protocol's own error format (MCP in-band, HTTP 200), never as
  plain text. Relyon's own refusals are relayed unchanged; on a paid or processing session they
  report the existing charge instead of saying nothing was charged.
- **What the sanitizer removes:** any sentence with a "not charged" claim, in any case and with
  any run of spaces, tabs, newlines or non-breaking spaces between the words: "nothing was / is /
  has been / will be charged", "Nothing charged", "charged nothing" (but not "charged nothing
  more"), "wasn't / hasn't been / haven't been / won't be / not (yet) been charged" (straight or
  curly apostrophes), "no charge has been made", "no charges were made", "no payment was taken",
  "didn't charge". Sentences end at `.`, `!` or `?` followed by whitespace, so "$12.50" never
  splits one; an abbreviation such as "e.g." may end a sentence early, which only narrows what is
  removed. A claim without that ending is removed together with the text after it, up to the next
  sentence end; if a claim would still be left, the whole message is replaced by the kit's note. It doesn't understand other languages or paraphrases ("your card is safe"): keep refusal
  messages about the cart.
- **Checkouts paid on the store's own page (hand-off):** Relyon's session for that agent checkout
  stays unpaid, so the kit decides. While this store's `payAndConfirm` remembers a payment for the
  agent checkout (or marked it order pending), the agent's UCP `complete` is answered by the kit
  and never relayed: order pending while the store's order isn't created, otherwise `409`
  `{ code: "checkout_paid_on_store", content }` ("This checkout was paid on the store's checkout
  page; this request charged nothing more…"). Relyon itself also refuses that charge, so a store
  server without the memory is covered too: the complete answers `requires_escalation` with a
  `checkout_paid_on_store` message, and a store payment after the agent's own wallet charge is
  refused `409 checkout_already_final`. A relayed Relyon refusal on such a checkout (a pause or a
  rule the kit hasn't seen yet) answers order pending while the order is pending, and is
  otherwise relayed with any "nothing was charged" sentence removed. ACP and MCP completes never
  charge and are relayed as usual.
- **MCP `complete_checkout`:** Relyon refuses MCP checkout before any charge today. Should a relayed
  MCP complete ever carry a succeeded payment without the store's order, the agent gets an in-band
  JSON-RPC error with code `-32045` (`data.error: "order_pending"`, `payment_status: "succeeded"`,
  `order_status: "pending"`), never a completed result. The kit's own refusals on that checkout
  (`get_checkout` or `complete_checkout` while paused, refused, or Relyon is unreachable) answer
  the same error.
- **`kit.payAndConfirm`:** answers `202` with `{ payment_outcome: "succeeded", order_status:
  "pending", order_reference, payment_id, message }` and `Retry-After: 5`, and returns
  `orderPending: true` with no `confirmation`. Show the shopper that the payment went through and
  the order is being placed, then call `payAndConfirm` again with the same input (order reference,
  attempt and the original `paymentMethod`): it replays the payment with the same Idempotency-Key,
  creates the order once and answers the `200` confirmation. Never ask for the card again.
  - Any refusal or failure on a key, order reference or agent checkout whose payment is recorded
    `order_pending` answers this same `202`: the kit's rule gate, Relyon's own refusals (an
    inactive agent, `checkout_already_final`, the in-flight or idempotency `409`, the attempt
    limit, a pause), and unexpected errors such as the state store failing. When the pending
    payment is another attempt's or another order reference's, the `202` adds
    `paid_order_reference` and `paid_attempt`: resend that request, not this one.
  - Otherwise, when the kit's records say the key, order reference or agent checkout may have
    been charged (`charging`, `order_pending`, `order_created`, or, for the kit's own gate, no
    record at all), the refusal keeps its status and code but its message (and `rule` label) is
    charge-neutral: no "nothing was charged", plus "Check the order status before paying again…".
  - "Nothing was charged" reaches the store only for a fresh request Relyon refused (it replays a
    key it charged before refusing anything) or after a recorded decline.
- **State stores without `orderClaimStatus`:** an order claim held elsewhere can't be told from a
  created order. Those answers keep today's behavior (Relyon's answer relayed, `payAndConfirm`
  confirms) and log a warning, so an order that exists is never shown as pending forever.

## Agent events and rules

- **Events:** every agent interaction is queued with a stable event ID and sent to Relyon in batches
  of up to 500. Failed batches are retried with backoff; Relyon dedupes on event ID. The queue is
  bounded: overflow is counted and never blocks a request. Keep reporting on; it is how the merchant
  sees agent activity.
- **Rules:** the merchant's agent rules are cached, refreshed every `refresh_seconds` and immediately
  on the signed webhook. At the edge, the kit refuses blocked agents, and checkout while the store or
  the agent's channel is paused. Refusals use each protocol's own error format.
- **No bot heuristics:** the kit adds no CAPTCHA, challenge, bot score or user-agent block of its own.

## Browser agents on the store's own checkout

Browser agents buy through the store's normal checkout page and type a virtual card into
`<relyon-card-fields>` like any shopper.

- **`kit.identifyAgent(request)`** names the agent. It is **verified** only when an RFC 9421 signature
  checks out locally (Web Bot Auth, Visa Trusted Agent Protocol or Mastercard Agent Pay), and
  **claimed** for everything else. A failed check falls back to claimed; it never errors the page.
- **`kit.verifiedAgent(request)`** returns `{ verified, skipChallenge, name, headers }`.
  `skipChallenge` is true only for a verified agent no Relyon rule refuses. If you run your own bot
  protection, skip it when `skipChallenge` is true; claimed agents and people go through your normal
  flow.
- **`createPayment`** sends Relyon only the shopper's `user-agent`, `signature-agent`, `signature-input`,
  `signature` and `ucp-agent` headers (never cookies or authorization), so Relyon can name
  the agent.
- **WebMCP:** `kit.createWebMcpCheckout({ lines })` returns the checkout, its card fields, a
  `<script>` tag and a `sessionCookie` the checkout page must set. In a browser with
  `navigator.modelContext`, the script registers `get_checkout`, `update_checkout` and
  `complete_checkout`. No tool accepts card data, and `complete_checkout` pays only after the shopper
  approves the total in a panel the server checks.
- **Order confirmation:** `kit.payAndConfirm(input)` and `complete_checkout` return the same
  `order_confirmation` JSON; `kit.renderOrderConfirmation` renders it with schema.org `Order`
  JSON-LD.

## Running on several servers

An approval token pays at most once, even when the store runs the kit on several servers or
processes. A multi-server deployment needs:

- **One shared WebMCP checkout store with the atomic claim** (`webmcp.store`). The default
  `MemoryWebMcpCheckoutStore` only works within one process. A `WebMcpCheckoutStore` must
  implement:
  - `claimApproval(id, claim)`: consumes the approval with `claim.jti` and records the pending
    payment in one atomic step. It succeeds only while that approval is outstanding and nothing
    is pending; `applyApprovalClaim` is the rule.
  - `compareAndSet(checkout, expectedVersion)`: writes only if the stored `version` is unchanged.
    Every write after creation (update, approve, recording the outcome) uses it, so a stale
    server can't overwrite newer state.
- **One shared checkout state store** (`state`) with the leased order claim and its done marker
  (`claimOrder`, `completeOrder`, `releaseOrder`, `orderClaimStatus`; see
  [Production: durable state](#production-durable-state)). It makes sure only one server creates
  the store's order for a payment, and that a server dying mid-creation doesn't strand it.
  Give it the store-payment memory too (`getStorePayment`, `putStorePayment`): a `payAndConfirm`
  retry that lands on another server after a `202` order pending is then answered order pending
  when the agent rules or Relyon refuse it, instead of only charge-neutrally, and an agent's UCP
  `complete` on a checkout paid on another server's page is answered by the kit. Without it,
  Relyon still refuses a second charge for one agent checkout. The shared record never
  overrides a stronger record this server holds (see
  [Production: durable state](#production-durable-state)). Concurrent requests for the same key
  on different servers can both reach Relyon: Relyon's per-key replay charges once, and the one
  it refuses as in flight answers charge-neutrally.

The kit ships `SharedWebMcpCheckoutStore`, which works over any backend with a conditional write.
Implement `AtomicKeyValueStore` (`get` returns the value and its version; `setIfVersion` writes
only if the version still matches, or only if the key is absent when the version is `null`).
`MemoryAtomicKeyValueStore` is the in-memory reference implementation. A Postgres example:

```ts
import { SharedWebMcpCheckoutStore, type AtomicKeyValueStore } from "@relyon/store-kit";

// CREATE TABLE webmcp_checkouts (key text PRIMARY KEY, value text NOT NULL, version bigint NOT NULL);
const kv: AtomicKeyValueStore = {
  async get(key) {
    const { rows } = await pool.query("SELECT value, version FROM webmcp_checkouts WHERE key = $1", [key]);
    return rows[0] ? { value: rows[0].value, version: Number(rows[0].version) } : null;
  },
  async setIfVersion(key, value, version) {
    const { rowCount } = version === null
      ? await pool.query("INSERT INTO webmcp_checkouts (key, value, version) VALUES ($1, $2, 1) ON CONFLICT DO NOTHING", [key, value])
      : await pool.query("UPDATE webmcp_checkouts SET value = $2, version = version + 1 WHERE key = $1 AND version = $3", [key, value, version]);
    return rowCount === 1;
  },
};

const kit = createStoreKit({ /* ... */ webmcp: { store: new SharedWebMcpCheckoutStore(kv) }, state: sharedStateStore });
```

With Redis, use a Lua script (or `WATCH`/`MULTI`) that compares a version field before `SET`.

How a race resolves:

- Only the server whose claim succeeds sends the payment. The other answers `409
  approval_already_used`: this request charged nothing, and the agent should read the checkout
  (`get_checkout`) for its status. It never changes the winner's state or creates the order.
- A `complete` that arrives while another server is still sending the payment gets the same
  answer. A pending payment counts as "being sent" for twice the Relyon timeout plus 5 seconds.
  After that, it is treated as an unknown outcome (the sending server may have died) and the
  next `complete` reconciles it.
- A succeeded payment is first stored as **order pending** (`status: "order_pending"`): the
  succeeded payment and its confirmation are kept, and the pending payment is cleared, so nothing
  can send it again. Then the store's order is created through the order claim, and only after
  that is the checkout stored as `completed` and the completion reported. If the store can't
  record the payment's outcome at all (the checkout changed under the write), the answer is the
  unknown outcome and a later `complete` reconciles the same payment.
- While the order is pending, `complete_checkout` (`202`) and `get_checkout` show
  `complete_in_progress` with an `order_pending` message: the payment went through and the order
  is being placed. It never says nothing was charged, shows no order confirmation or card-field
  token, and asks for no new approval. Updates and approvals are refused (`409 order_pending`),
  like on a placed order, and so is the approval claim. A Relyon rule refusing a step there says
  the payment already went through. When the kit can't establish that a checkout is unpaid (no
  matching page-session cookie, a payment pending or done, or the checkout store can't be read),
  refusals don't mention charges at all and show nothing about the checkout. The same goes for an
  unexpected failure on any WebMCP checkout step (`500 internal_error`), for shoppers and agents.
- Calling `complete_checkout` again on an order-pending checkout finishes it. It needs no
  approval token or card and never calls Relyon's payment endpoint: it creates the order through
  the order claim and then stores `completed`. When `createOrder` threw, or the server died after
  storing order pending, the next `complete` (on any server) creates the order. When several
  requests race, exactly one `createOrder` runs; the others answer order pending until the winner
  stored `completed`, and every request after that returns the completed checkout. If the server
  holding the claim died mid-creation, the order stays pending until the claim's lease passes.
- Relyon is the backstop if a custom store's `claimApproval` isn't truly atomic (for example a
  read-then-write without a conditional write). The built-in `MemoryWebMcpCheckoutStore` and
  `SharedWebMcpCheckoutStore` over a correct `setIfVersion` are atomic, so this doesn't arise
  with them. Every send for one approval uses the same Idempotency-Key and order reference, so
  Relyon charges once. A second first-time send that Relyon refuses (`idempotency_key_reused`,
  `checkout_already_final`), or answers with a replay of a payment another server already
  recorded, is answered `approval_already_used`. If Relyon is still processing the first send, it
  may answer the second "in progress"; that server answers the unknown outcome (never a second
  charge), and the next `complete` returns the placed order.

Failing closed:

- The approval is consumed only after the pre-charge checks pass, including the agent-rules
  decision read from Relyon. If Relyon can't be reached for them, `complete` is refused (`503`),
  nothing is charged, and the approval stays usable.
- A store without `claimApproval` and `compareAndSet` refuses `complete`, updates and approvals
  with `503 checkout_store_unavailable`; it never falls back to an unguarded write. A claim that
  throws gets the same answer, and nothing is charged. A checkout another server already paid,
  placed or has a payment pending for is answered as such there (order pending, already placed,
  or the unknown outcome), never with that refusal; such a server never sends a payment.
- If Relyon becomes unreachable during the charge, the approval stays consumed and the outcome
  stays unknown. No new approval or Idempotency-Key is issued until Relyon gives a final answer
  (see the unknown-outcome rules above).

## Failing closed

- If Relyon is unreachable, or answers with a 5xx or a malformed reply, agent checkout steps answer
  `503` and payment calls throw `PaymentOutcomeUnknownError` (Relyon may have charged before the
  response was lost). On an order-pending checkout the step answers order pending instead, and on
  any other existing checkout the wording is charge-neutral (see
  [Order pending after a charge](#order-pending-after-a-charge)).
- For an identified agent, payments and checkout steps also fail closed before any charge when the
  agent rules can't be read (`agents.failClosed`, default true). A person's browser is never affected.
- Discovery routes keep working from cache, and their events stay queued until Relyon is back.

## Compatibility notes

- **`paymentCartFormat: "server-legacy"`** sends the cart shape Relyon's Test API accepts today
  (lines keyed by `product_id`, no top-level `currency`). The default, `contract`, sends the
  contract's `Cart`. Set `"server-legacy"` until Relyon announces otherwise.
- A pending payment (`202`, `status: pending`) is treated only as an unknown outcome, never as a final
  success or failure.
- **`order_pending` isn't in the published error sets.** Neither UCP's error codes nor ACP's
  `Error.code` values list `order_pending` (nor `payment_outcome_unknown`); agents that don't know
  it still get HTTP `503` with `Retry-After`, the `Payment-Status`, `Order-Status` and `Link`
  headers, and a message saying the payment went through.
- **ACP's `CheckoutSessionWithOrder` requires `order`.** An order-pending ACP complete answers a
  plain `CheckoutSession` (`status: "complete_in_progress"`, no `order`) instead.
- **Custom WebMCP checkout stores** must implement `compareAndSet` and `claimApproval` as well as
  `get` and `put`, and keep the checkout's `version`. A store written for `get`/`put` only is
  refused (`503 checkout_store_unavailable`) rather than used without the single-use guarantee.
  The default memory store needs no change.
- **Relyon's hand-off links are verified locally.** The v1 contract has no operation that resolves
  or verifies a `relyon_cart` token, so the kit accepts a Relyon-issued token only when it saw that
  exact token in a relayed answer for the agent checkout (above). Live links also need a valid
  signature from the store's published Live keys. A link opened on another server
  works only when the servers share a `CheckoutStateStore`, and is single use across servers only
  when that store implements `claimHandoffToken`. The kit can't tell a link Relyon wrote from a
  link-shaped string an agent got Relyon to echo for its own checkout, so it records one only
  when it names that checkout, preferring the answer's `continue_url` (or its MCP equivalent), then
  its messages; two different links at the same place record none.
- **Relyon's hand-off links always use `/checkout`.** Relyon builds them as
  `https://<host>/checkout?relyon_cart=…` whatever `paths.checkout` says; a store with a custom
  `paths.checkout` must also serve `/checkout` (or send shoppers on from it to the custom path), or the shopper never reaches its page.
- **An unattributed landing has no cart.** A store page that turns an empty cart into an error
  (for example a 404) shows that error for any hand-off link the kit doesn't accept; show a normal
  empty checkout instead.
## Reference

- Public API base: https://core.getrelyon.com/api (for example, `/v1/products` → `https://core.getrelyon.com/api/v1/products`).
- API contract (OpenAPI 3.1): https://core.getrelyon.com/v1/openapi.yaml
- The contract download and CLI downloads use the service origin, not the API base.
- Relyon: https://core.getrelyon.com
- Source and issues: https://github.com/relyon-payments/relyon-core. Changes are listed in the npm
  package's `CHANGELOG.md`.

## License

MIT. See [LICENSE](LICENSE).
