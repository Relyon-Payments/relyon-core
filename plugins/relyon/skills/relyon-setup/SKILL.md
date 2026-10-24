---
name: relyon-setup
description: Add Relyon payments and agent commerce to a store app by installing @relyon/store-kit on the app's own Express, Next.js or Hono server, with a Test sandbox key, durable state and the bundled Relyon skills.
---

# Relyon setup

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

Reference only. Read [the quick start](references/QUICKSTART.md) first.
Open only the section a numbered quick-start step names.

## Recommended build order

Run `npx @relyon/store-kit@latest init` first, then read `relyon/QUICKSTART.md`.
Open a skill only when a numbered quick-start step names its section, not all
six full files in sequence. The package and public repository also carry
`QUICKSTART.md`; CLI-only installations carry the same guide in
`relyon-setup/references/QUICKSTART.md`.

Follow its seven stages: init, mount, catalog, checkout, doctor, Test purchase,
handover. Verify dependencies before exercising purchases; missing credentials
defer credential-dependent checks, not local building. Pending is not passed.
Agents never enable Live. No particular task tool is required.

## Facts at a glance

**Relyon's API base URL** is `https://core.getrelyon.com/api`. Every
operation path is appended to it (`/v1/payments` is
`https://core.getrelyon.com/api/v1/payments`).

- The kit has no built-in default: the app passes the base as
  `relyon.baseUrl` when it creates the kit. The examples read
  `RELYON_API_BASE_URL` and fall back to `https://core.getrelyon.com/api`.
- The kit's doctor (`npx @relyon/store-kit doctor`) takes `--base-url` or
  `RELYON_BASE_URL`, with the same API base as its default.
- The optional `relyon` CLI also reads `RELYON_BASE_URL`, but there it is the
  service origin `https://core.getrelyon.com`, without `/api`.
- Override any of them only with another operator-confirmed Relyon
  deployment. None of them is ever the store's own URL.

**Every setting the kit and the CLI read.** The kit library reads no
environment variables itself; the app reads them and passes options. These are
all the names involved:

| Name | Read by | Secret | What it is |
| --- | --- | --- | --- |
| `RELYON_API_KEY` | the app (`relyon.apiKey`), the kit's doctor, the `relyon` CLI | yes | The sandbox's `restrictedKey` (`rly_test_…`). |
| `RELYON_PRIVATE_DIR` | init, the Node server-config loader, the kit's doctor | private path | Existing owner-only 0700 POSIX directory outside the app and every git tree. Optional safe automatic Test setup. |
| `RELYON_WEBHOOK_SECRET` | the app (`rules.webhookSecret`) | yes | The one-time `secret` from registering the store's webhook endpoint. |
| `RELYON_CART_TOKEN_SECRET` | the app (`cartTokens.secret`) | yes | A store-generated secret for hand-off cart tokens; stable across processes and key rotation. |
| `DATABASE_URL` | the app (durable state, orders) | yes | The store's database connection string. |
| `RELYON_API_BASE_URL` | the app (`relyon.baseUrl`) | no | Relyon's API base (above). |
| `RELYON_LIVE` | the app (`relyon.live`), the kit's doctor (`--live`) | no | `true` only after Relyon staff approved Live (`relyon-go-live`). |
| `STORE_HOSTNAMES`, `STORE_HOSTNAME` | the app (`store.hostnames`) | no | The store's own hostnames, primary first. |
| `STORE_PUBLIC_URL` | the app (`store.publicUrl`), the kit's doctor (`--url`) | no | The store's public HTTPS origin. When set, every agent-facing link uses it; otherwise links use the store's own hostname the request arrived on. |
| `PORT` | the app | no | The port the host gives the server. |
| `RELYON_BASE_URL` | the kit's doctor (`--base-url`), the `relyon` CLI | no | Relyon's API base (doctor) or service origin (CLI). |
| `RELYON_FEEDS` | the kit's doctor (`--feeds`) | no | Which store feeds the doctor checks, comma-separated. |
| `RELYON_API_VERSION` | the `relyon` CLI | no | Pins the API version the CLI sends. |
| `RELYON_SKILLS_SOURCE` | the `relyon` CLI | no | Where the CLI copies skills from, instead of its bundled copy. |
| `RELYON_NO_UPDATE_CHECK` | the `relyon` CLI | no | `1` turns off its new-version notice. |
| `XDG_CACHE_HOME` | the `relyon` CLI | no | Where it caches that new-version check (the usual cache folder by default). |

`npx @relyon/store-kit init` preserves an existing `RELYON_API_KEY`. Automatic
Test provisioning is supported only on Node/POSIX with `RELYON_PRIVATE_DIR`
pointing to an existing owner-only (0700) directory outside the app and every
git tree. It stores Test credentials and the claim URL in `sandbox.json` (0600),
never prints them, and removes every `RELYON_*` variable from the package
manager environment. No platform secret-manager service is assumed.
The operator chooses a separate private directory per store; init never creates
one under the app or a public/static directory. A persistent attempt marker
prevents concurrent, repeated and ambiguous-outcome provisioning retries.
Existing dotenv files also skip automatic provisioning rather than guessing
whether they already contain credentials. Keep the app's existing configuration.
Do not delete it to retry blindly. If unavailable, keep building keylessly.

**pnpm and Replit workspaces.** In a pnpm workspace (a `pnpm-workspace.yaml`
at or above the app), the kit belongs in the package that runs the store's
server, never the workspace root. Run `init` from the project root: when the
workspace has exactly one server package it installs there; otherwise it
skips the install and prints the exact command to run, such as
`pnpm --filter <server package> add --save-exact @relyon/store-kit@0.1.6`.
It still copies the skills to the project root. Running `init` inside the
server package installs into that package.

**A brand-new kit version may be held back.** If the package manager refuses
the pinned version because of a minimum release age policy (pnpm's
`minimumReleaseAge`, for example), either wait until the version is old
enough, or exempt only `@relyon/store-kit` from that policy. Don't turn the
policy off for every package and don't install an older kit version instead.

**Why `init` refuses to replace a file.** It never overwrites a file that
differs from what it would write: a skill copy someone added or edited by
hand, an older copy from another channel, or a `relyon-store-brief.md` with
other text. If any file conflicts, it writes nothing at all and exits with an
error that names the file. Compare the file with the bundled skill first.
Use `--force` only when the differing copies are older or hand-made copies
of these same skills and nothing in them needs to be kept.

- `--force` replaces the six skills' files in `.agents/skills`,
  `.claude/skills` and `.cursor/skills`, the store brief, and only the marked
  Relyon block in `AGENTS.md` and `CLAUDE.md` (it adds the block if there is
  none).
- It never touches anything outside that block, other skills, or extra files
  in a skill folder that the pack doesn't have.
- It still refuses a symlink, a file that isn't UTF-8 text, or a damaged,
  repeated or nested Relyon block; fix those by hand.

## Start here: from the store idea to a working Test store

Run `npx @relyon/store-kit@latest init` first, then read the copied
`relyon/QUICKSTART.md`. For CLI/plugin-only installations read
[the same quick start](references/QUICKSTART.md). Open a skill only when a
numbered step names its section; do not read all six full files in sequence.
The shared rules above own Test defaults and ask-versus-proceed decisions.
Missing credentials defer credential-dependent verification, never local
building. Never expose unfinished checkout or synchronize placeholder policies.

UCP Test checkout is implemented. ACP Test integration is implemented with
merchant-domain forwarding, ChatGPT credentials, application and channel prerequisites.
Store-hosted MCP checkout works in Test by handing off to the store page;
Live returns `protocol_not_available`. Separate Commerce MCP supports checkout-link handoff, not direct charging.
No platform approval is promised.

### Doctor merchant questions are not technical failures

In Test, the kit doctor labels missing merchant return terms, a confirmed returns
page and support contact as `PENDING merchant`. An exact missing-webhook finding
is pending only with positive preview or loopback evidence. These are questions
for the merchant, not permission to invent a window, return cost, email or address.
List them separately from technical `FAIL` items in the hand-over. Agent checkout
remains unavailable until the merchant confirms the facts and policies are set;
the server still enforces every checkout gate.

Default Test doctor exits zero when there is no `FAIL`, even if merchant items are
pending or checks are `UNVERIFIED`. That does not prove checkout or Live readiness.
Use `doctor --strict` to treat pending merchant requirements as failures. Every
Live run keeps them as failures without needing `--strict`. Fix technical failures
normally; never hide mixed catalog blockers or an ordinary public-store webhook fault.

Policy pages may visibly say **"to confirm with the merchant"** while answers are
missing. Keep this display copy separate from kit policy options, synchronization,
feeds, JSON-LD and API payloads. Never synchronize placeholder text or pretend that
a page containing it is a confirmed policy. Only synchronize real merchant-approved
facts; an absent answer stays absent. Do not submit the Merchant to-do questions as data.

## After you deploy: reachable, images, then the scan

Do these after every publish, on the public address (not the preview):

1. **Publicly reachable.** Some hosts publish privately or invite-only by
   default, or put a login in front of the app. From a shell with no cookies,
   `curl -sS -o /dev/null -w '%{http_code}\n' https://<host>/` must answer
   `200` with the store's own page, not a sign-in page; check `/llms.txt`
   and `/.well-known/ucp` the same way. If not, ask the merchant to make the
   deployment public in their host's settings. Agents and the scan can't
   read a private store.
2. **Images load from the public address.** Open a product page and fetch
   each `image` URL from its Product data and from `/feeds/google.xml`: each
   must answer `200` with an image type, on the store's public hostname. A
   localhost, preview or dev-only image URL means the catalog stored an
   address from build time: store image paths on the store's own hostname
   (the kit moves those onto the public address) and serve the files from
   the deployed app.
3. **The free scan.** Run it on the public URL and read every check:

   ```sh
   curl -sS -X POST https://core.getrelyon.com/api/scan \
     -H 'Content-Type: application/json' -d '{"url":"https://<host>"}'
   ```

   It is also at `https://core.getrelyon.com/scan`. It reads only public
   pages, needs no key and can't reach `localhost` or a private address.
   Fix every check the store's own code or content fails, publish again
   and rerun it. Record checks only the merchant or Relyon can fix as open
   items in the hand-over.

**Avoid a page reload loop.** A storefront that keeps reloading itself on a
preview or tunnel is almost always the dev server's live reload: its
websocket can't connect through the host's front end, so the client reloads to
retry, forever. Run the built app (production mode, for example
`NODE_ENV=production node main.js`) for agent and scan testing, or turn live
reload off for the preview. The store's own pages must never reload
themselves either: the checkout page polls `/checkout/confirm` with `fetch`
and stops on a final answer, and the status page never sends the browser
back to itself.

**Done, blocked or open.** A step is done only when its check passed. Fix every
failure the store's own code, content or setup causes. A failure only Relyon or
the merchant can resolve (a staff-only setting, a contracted operation that
  answers 404, 405 or 501, the scan being unavailable, a fact the merchant hasn't
given yet) is an open item: record the step as blocked, with the code or
message, in the `relyon-go-live` hand-over. Never describe a blocked step as
passed.

## Platform notes

- **Replit Agent.** The merchant starts with a prompt in a new Replit App
  (prompt-first: "Build my store: … Use Relyon Core: run
  `npx @relyon/store-kit init` and follow the Relyon skills"). Run `init` in
  the Shell; Replit Agent reads `.agents/skills` and `AGENTS.md`. Keys go in
  the **Secrets** pane only. Use Replit's built-in Postgres database
  (`DATABASE_URL`) for products, orders and the kit's durable state. Publish
  as a **Reserved VM** deployment so the server stays on (see
  [Replit hosting](#replit-hosting-and-hostname-rotation)). Who does what:
  you create the database, run the schema migration, set the run command
  (the server's start script) and make the server listen on `PORT`; the
  merchant runs the sandbox and webhook commands in the Shell, enters
  secrets in the Secrets pane and presses Publish. After publishing, read
  the deployment's hostname, set `STORE_HOSTNAMES` and `STORE_PUBLIC_URL`
  for the deployment, and publish again.
- **Claude Code.** `init` writes `.claude/skills` and `CLAUDE.md`, which Claude
  Code reads. Or install the plugin:
  `/plugin marketplace add relyon-payments/relyon-core`, then
  `/plugin install relyon@relyon`. Put keys in the shell environment or the
  host's secret store, never in `CLAUDE.md` or chat.
- **Codex.** Reads `.agents/skills` and `AGENTS.md` at the project root.
- **Cursor.** Reads `.cursor/skills`, and `AGENTS.md` at the project root.
- **OpenCode.** Reads `.agents/skills` (also `.opencode/skills`) and
  `AGENTS.md` at the project root.
- **Any other platform or host** (Bolt, Vercel, Netlify, Codespaces, a
  local machine and others): keys go in your platform's secrets or
  environment settings, never in a committed file or chat. For Test agent
  checkout on the preview address, set `RELYON_PREVIEW_HOSTS` to the app's
  preview hostname or tunnel when the kit doesn't pick it up itself (see
  [Test agent checkout on the preview address](#test-agent-checkout-on-the-preview-address)).
- **Lovable and Base44** don't load skill folders or run `init`; use the
  Relyon Core repository's paste blocks for those platforms.

## The six Relyon skills

| Skill | Use it for |
| --- | --- |
| `relyon-setup` (this one) | The Start here flow, sandbox key, installing and mounting the kit, durable state, policies, diagnostics |
| `relyon-catalog` | The app's own product data and importer, `catalog.source`, JSON-LD, `llms.txt`, the sitemap and feeds |
| `relyon-checkout` | `priceCart`, the checkout page on the store's domain, card fields and `createPayment` |
| `relyon-agents` | The UCP, ACP and MCP routes the kit serves, agent rules and agent activity reporting |
| `relyon-orders` | `createOrder`, fulfillment updates and verifying Relyon webhooks |
| `relyon-go-live` | What Test proves, domain verification and the hand-over to Relyon staff for Live |

**Reference lookup only.** Open only the section named by the quick start:

| Stage | Open |
| --- | --- |
| Install the kit, the Test key, durable state, policies, the doctor | `relyon-setup` |
| Products, variants, prices, stock, images, product pages, `llms.txt`, sitemaps, feeds | `relyon-catalog` |
| Cart pricing, the store's checkout page, card fields, Test payment | `relyon-checkout` |
| UCP, ACP and MCP routes, hand-off links, agent rules, crawler rules, Test agent checkout on the preview address | `relyon-agents` |
| Saving orders exactly once, fulfillment updates, Relyon webhooks | `relyon-orders` |
| Publishing, verifying the domain, final checks and the Live hand-over | `relyon-go-live` |

## Test agent checkout on the preview address

With a Test key (`rly_test_…`), agent checkout works on the app's preview
address right away, before the store is published or its domain verified,
on any coding platform or host: Replit, Lovable, Bolt, Vercel, Netlify,
Cursor, Codespaces, a tunnel, a local machine and others. Relyon keeps no
list of platforms: with a Test key it accepts any https address (a DNS name,
not an IP address) and `localhost` or a loopback IP as the key's own Test
store, without a verified domain.

- In Test mode the kit answers on these hostnames as well as on
  `store.hostnames`:
  - `localhost`, `127.0.0.1` and `[::1]`;
  - every address listed in `RELYON_PREVIEW_HOSTS` (comma separated
    hostnames or URLs), set in your platform's secrets or environment
    settings: the way to name a tunnel or any preview URL, on any platform;
  - `store.previewHosts`, when you pass a list of hostnames;
  - the app's own address when its platform names it in a well-known
    variable (Replit, Vercel, Netlify, Codespaces, Render and Railway do).
    That is only a shortcut for setting `RELYON_PREVIEW_HOSTS`.

  Any other host header is still refused, so a forged Host can't make the
  store answer for an address it was never given. Set
  `store.previewHosts: false` to turn all of this off. Live mode never adds
  them.
- On a preview address the store serves `/.well-known/ucp`, the UCP and ACP
  checkout routes, `/mcp` browsing and `llms.txt` as the key's own Test
  store. An agent can quote, hand off to the store's checkout page on that
  same address, pay with Test payment methods and get a Test order.
  Everything is marked Test (`livemode: false`); no real money moves.
  Store-hosted MCP checkout works in Test: it hands the shopper off to the
  store's checkout page (`continue_url`), the payment happens there and the
  order gets Relyon's `ord_…` ID. In Live it still answers
  `protocol_not_available`.
- Plain `http` and a port work only for loopback (`localhost` on port 3000, for example).
  Every other preview address is `https` on the default port.
- The preview address belongs to the key's own Relyon account only. Another
  account using the same address gets its own Test data, never this store's.
  A store may use up to 20 preview addresses; one unused for a day makes
  room for a new one.
- The kit's doctor reports such a store as **Test on preview address**, not
  as a failure. Channel setup (Google, Merchant Center, ChatGPT) still
  needs a verified domain.

**Going live later.** A Live key is refused on any preview or unverified
address. The steps, in order:

1. Publish the app on your platform.
2. Add its domain in Relyon and verify it (`kit.ensureDomain()`,
   `kit.verifyDomain()`; see `relyon-go-live`).
3. Switch to a Live key, only with the merchant's approval. Relyon staff
   enable Live; agents never ask for a Live key or set `relyon.live`.

The approved public Relyon v1 contract is dated **2026-10-30**. It is published,
without authentication, at `https://core.getrelyon.com/v1/openapi.yaml` (for
another deployment, `<service origin>/v1/openapi.yaml`). It is not the Relyon
dashboard's API. The kit already speaks it; read it only to check an error
code or a field.

The canonical API base is `https://core.getrelyon.com/api`; append operation
paths such as `/v1/products` to get `https://core.getrelyon.com/api/v1/products`.
The OpenAPI download above uses the service origin, not this API base. For
another deployment, retain its operator-confirmed API base.

## Get a Test sandbox

When init could not save a Test key, keep building. Only after building, if
checkout is still waiting, send the
merchant to https://core.getrelyon.com/get-sandbox-key and their host's
secret tooling.
Ownership claiming is optional for continuing the build; unclaimed sandboxes
expire. Automatic setup retains the claim URL privately, never in output.
After setting a real Test key, restart the server and resume pending verification
without rebuilding. Do not remove an attempt marker after an unknown outcome
or silently issue a second sandbox.

Start in **Test**. `POST https://core.getrelyon.com/api/v1/sandboxes` needs no key;
the body `{"name":"Example shop"}` is optional. Give the merchant this command to
run in their own interactive shell (not through your tools, so the keys never
reach the conversation):

```sh
curl -sS -X POST https://core.getrelyon.com/api/v1/sandboxes \
  -H 'Content-Type: application/json' -d '{"name":"Example shop"}'
```

It returns `201` with exactly these fields (values shortened here):

```json
{
  "object": "sandbox",
  "id": "org_…",
  "name": "Example shop",
  "livemode": false,
  "publishableKey": "pk_test_…",
  "restrictedKey": "rly_test_…",
  "claimUrl": "https://core.getrelyon.com/claim-sandbox#token=ONE-TIME-TOKEN",
  "expiresAt": "2026-10-31T12:00:00.000Z"
}
```

- Keys and the claim URL are shown once. Display one-time secrets only in the
  merchant's interactive **Shell**, never in chat, captured logs, source control
  or browser code. The merchant privately opens the claim URL to keep the sandbox
  and transfers credentials directly to the **Secrets pane**; do not read them
  back or ask the merchant to send them in chat.
- Store `restrictedKey` with the platform's secret tooling as
  `RELYON_API_KEY`. It is a server-side restricted key and carries every
  merchant scope the kit needs. Restricted keys (`rly_test_…`, `rly_live_…`)
  stay server-side. Publishable keys (`pk_test_…`) are not interchangeable with
  them, and a store-first app doesn't need one: the card fields get a
  short-lived client token from the kit instead.
- By default the kit refuses `rly_live_…` keys: Live needs both a Live key and
  `relyon.live: true`, an https `STORE_PUBLIC_URL`, and Relyon staff approval.
  Never ask for a Live key or set that option (see `relyon-go-live`).

## Install the kit

`npx @relyon/store-kit init` (see [Start here](#start-here-from-the-store-idea-to-a-working-test-store))
installs the pinned kit and these skills in one step. The manual way follows.

### Replit: install the public skills manually

Use the public GitHub **relyon-core** repository, not a private workspace or
copied kit source. From the app's project root in the Replit shell:

```sh
git clone --depth 1 https://github.com/relyon-payments/relyon-core.git /tmp/relyon-core
mkdir -p .agents/skills
cp -R /tmp/relyon-core/skills/* .agents/skills/
npm install @relyon/store-kit@0.1.6
```

In channels without a repository URL in the clone command, set
`RELYON_CORE_REPOSITORY` to the confirmed public GitHub relyon-core URL from
its README; do not guess an owner. The public repository copy includes the
clone URL directly. The CLI below is optional; these public skills and the
pinned npm kit are enough.

Use **Node.js 20 or newer**, ES modules (`"type": "module"` in package.json),
and a server on the store's own domain, not static-only hosting. The Express
example also needs `npm install express@5`; the Postgres example needs
`npm install pg`. Extract the named JavaScript files from the references,
put browser `checkout.js` in `public/checkout.js`, and run `node main.js`.
Serve the app's real product images at the URLs in its catalog.

Add the dependency with the project's package manager: `npm install @relyon/store-kit@0.1.6`
(or the pnpm/yarn/bun equivalent, with the same pinned version). This is the intended install path. If the
registry can't find the package, stop and tell the merchant. Don't recreate the
kit by hand, copy its source into the app, or call Relyon's agent routes
directly in its place.

### Server environment

Set credentials using Replit's Secrets pane only, never by pasting values
into chat, source files, logs or browser code. Non-secret settings can use the
server environment too.

| Setting | Required use and provenance |
| --- | --- |
| `RELYON_API_KEY` | Optional during build; required for credential-dependent operations. The `restrictedKey` from unauthenticated `POST https://core.getrelyon.com/api/v1/sandboxes`, stored server-only. Not the publishable key. |
| `STORE_HOSTNAMES` | Comma-separated bare hostnames, primary first; no scheme, port or path. For example `shop.example.com,store.example.com`. It takes precedence over `STORE_HOSTNAME`. |
| `STORE_HOSTNAME` | Compatibility setting for a single hostname when `STORE_HOSTNAMES` is absent. The example defaults to `shop.example.com` for demonstration only. |
| `STORE_PUBLIC_URL` | Optional explicit HTTPS store origin, with no path. Its hostname must match the primary configured hostname; the catalog example otherwise derives its origin from that primary hostname. |
| `PORT` | The listening port supplied by the host. The example defaults to 3000 locally and listens on `0.0.0.0`. |
| `RELYON_WEBHOOK_SECRET` | Required to accept signed webhooks, optional for timer-only rule refresh. Save the one-time `secret` returned by registering this store's `/relyon/webhooks` endpoint through `POST /v1/webhook_endpoints` (see `relyon-agents`). Do not generate it yourself or reuse another endpoint's secret. Without it, the kit refuses webhook deliveries. |
| `RELYON_CART_TOKEN_SECRET` | Optional for local Test; required as a stable store-managed server-only secret before shared/restarted deployments. Generate it securely with the host's secret tooling, share it across server processes, and retain it across API-key rotation. Without it, kit tokens derive from the API key. |
| `RELYON_API_BASE_URL` | Optional. Defaults to `https://core.getrelyon.com/api`; use only another operator-confirmed API base. This is Relyon's API origin/path, never the store URL. |
| `DATABASE_URL` | Required for the Postgres durable adapters and durable orders/checkouts in a deployed store. Obtain it from the store's database provider via secret tooling. A connection string alone does not install adapters or migrations. Local in-memory Test needs no database. |

The runnable starter deliberately uses memory; it is not a durable deployed
store. Wire in the database adapters below, migrate once before serving,
and replace the example orders map with a transactional database table unique
on payment ID. Do not reset checkout attempts or order claims on restart.

### Replit hosting and hostname rotation

Use a **Reserved VM** for always-on serving, or a deployment with **Postgres-backed
state** that restores state across process restarts; a Reserved VM alone does not
make memory or deployment files durable. Wire the adapters in the
[durable-state reference](references/durable-state.md) for checkout attempts,
order claims and agent events. Published files can reset on republish.

Verify ownership on the actual published `*.replit.app` host (or the custom
domain), not a sleeping/changing development URL. Confirm the published host
serves the kit routes over HTTPS before requesting verification.
Relyon schedules daily ownership rechecks using wall time. The often-described
**three-failed-daily-recheck rule** applies only to consecutive lookup failures:
a missing or mismatched proof fails immediately. Inconclusive lookup failures
leave the domain due, so retries can occur sooner than three separate days;
a successful proof resets the counter.

Rotation checklist, **dev → .replit.app → custom-domain**:

1. Use development hosts for Test only (Test agent checkout works there); never advertise them as durable public URLs.
2. Publish, set `STORE_HOSTNAMES` to the confirmed `.replit.app` hostname first,
   update `STORE_PUBLIC_URL` if set, restart the store, then add and verify that hostname.
3. Connect the custom domain, put it first in `STORE_HOSTNAMES`, retaining the
   published hostname only while it still serves the store. Update `STORE_PUBLIC_URL`
   and product, policy and webhook URLs to the new primary origin.
4. Add and verify the custom hostname, check discovery and a Test checkout, then
   retire old links/proofs only after migration. Domain ownership never changes traffic routing.

## Create the kit on the server

Create one kit per server process, in a server-only module. The full working
example is in [the Express server reference](references/express-server.md);
the core is:

```js relyon.js
// Server-only module. The kit is the only code that talks to Relyon.
import { createStoreKit } from "@relyon/store-kit";
import { priceCart, createOrder, catalogSource } from "./store.js";

export function createRelyonKit({ env = process.env, fetch, state, events, catalog,
  source = catalogSource, formats = ["openai", "openaiGzip", "google", "microsoft"],
  cacheSeconds = 60, onDiagnostic, robots } = {}) {
  const hostnames = (env.STORE_HOSTNAMES ?? env.STORE_HOSTNAME ?? "shop.example.com")
    .split(",").map(host => host.trim()).filter(Boolean);
  return createStoreKit({
    relyon: {
      baseUrl: env.RELYON_API_BASE_URL ?? "https://core.getrelyon.com/api",
      apiKey: env.RELYON_API_KEY, // rly_test_… from the server environment
      waitingForSandboxKey: true, // Test-only local build; a real key activates normal operation
      ...(fetch ? { fetch } : {}),
    },
    store: {
      name: "Example Candle Co",
      description: "Hand-poured soy candles.",
      hostnames, // primary first; STORE_HOSTNAME is the single-host compatibility fallback
      // Optional. Without it, links use the store's own hostname the request came in on.
      ...(env.STORE_PUBLIC_URL ? { publicUrl: env.STORE_PUBLIC_URL } : {}),
    },
    // The policy pages the homepage footer links to (server.js). Returns are
    // added only with the merchant's confirmed terms (Store policies, below).
    policies: {
      shippingUrl: `https://${hostnames[0]}/shipping`,
      privacyUrl: `https://${hostnames[0]}/privacy`,
      termsUrl: `https://${hostnames[0]}/terms`,
    },
    // The app's own functions (relyon-checkout, relyon-orders).
    priceCart,
    createOrder,
    // Signed agent_rules.updated webhooks refresh rules right away (relyon-agents).
    rules: { webhookSecret: env.RELYON_WEBHOOK_SECRET },
    // Only POST /v1/payments needs this conversion; agent carts stay contract Cart.
    paymentCartFormat: "server-legacy",
    // Production: shared, durable stores (references/durable-state.md).
    ...(state ? { state } : {}),
    ...(events ? { events: { store: events } } : {}),
    catalog: { source, ...(catalog ? { persistence: catalog } : {}) },
    feeds: { formats, cacheSeconds, ...(onDiagnostic ? { onDiagnostic } : {}) },
    // Opt-in only; existing() must read the store's original robots policy.
    ...(robots ? { robots } : {}),
    ...(env.RELYON_CART_TOKEN_SECRET ? { cartTokens: { secret: env.RELYON_CART_TOKEN_SECRET } } : {}),
  });
}
```

- `await kit.start()` once at boot: it restores the saved catalog, reads the
  agent rules and starts the event reporting timer. Calling it again is safe.
- Automatic storage users: on the Node server only, import `loadTestConfig`
  from `@relyon/store-kit/server-config`, then load with
  `await loadTestConfig({ project: process.cwd() })` and pass its `apiKey`
  into `createRelyonKit`'s server environment. Never return it to a client.
  Without a key, the explicit waiting mode starts store-owned discovery without
  API calls, and `kit.checkoutStatus` is `"waiting for sandbox key"`.
  Show that status on checkout and disable Pay/card fields. Add a real Test key
  through server-only configuration and restart; resume checks, do not rebuild.
- `await kit.stop()` on shutdown (for example on `SIGTERM`): it stops the
  timers and flushes queued agent events. Don't call it per request.
- `store.hostnames` lists the store's own public hostnames, the first being
  the primary one.
- **The public address.** Every agent-facing link (llms.txt, the UCP
  profile, the MCP link, both sitemaps, Product data, product and image
  links, feeds and the agent card) uses one origin per request,
  `kit.publicOrigin(request)`: `STORE_PUBLIC_URL` when set, otherwise the
  store's own or Test preview hostname the request arrived on. Any other Host
  header is refused, never trusted. Catalog URLs on the store's own
  hostnames (including loopback and preview addresses) are moved onto that
  origin, so an address from build or dev time never leaks into what agents
  read. Pass the page's request to `kit.renderProductJsonLd(product, request)`
  and `kit.renderProductListJsonLd(products, request)` for the same result.

## Mount it on the app's server

The kit answers its own paths and passes everything else to the app.

- **Express:** `app.use(storeKitMiddleware(kit))` from
  `@relyon/store-kit/express`, **before** `express.json()` or any other body
  parser. The kit needs the raw request body to relay signed agent requests,
  and the middleware throws if a parser ran first.
- **Next.js App Router:** `storeKitRouteHandlers(kit)` from
  `@relyon/store-kit/next` returns `GET`, `POST`, `PUT`, `PATCH`, `DELETE` and
  `HEAD`. Export them from a route file for each kit path:
  `app/.well-known/ucp/route.js`, `app/.well-known/relyon-verification/route.js`,
  `app/ucp/[...path]/route.js`, `app/acp/[...path]/route.js`,
  `app/mcp/route.js`, `app/llms.txt/route.js`,
  `app/sitemap-products.xml/route.js`, `app/feeds/[file]/route.js` for selected
  feeds, `app/robots.txt/route.js` only when opting in, and
  `app/relyon/webhooks/route.js`, each
  containing `export const { GET, POST, PUT, PATCH, DELETE, HEAD } = storeKitRouteHandlers(kit);`
  plus `export const runtime = "nodejs";`. Import `kit` from one shared
  server module so every route uses the same instance. `storeKitFetch(kit)`
  returns `Response | null` for code that routes Web requests itself.
- **Hono:** `app.use(storeKitMiddleware(kit))` from
  `@relyon/store-kit/hono`, before the app's routes. On Node, serve it with
  `@hono/node-server`.

Paths the kit serves by default: `/.well-known/ucp`,
`/.well-known/relyon-verification`, `/.well-known/agent-card.json`, `/ucp/*`,
`/acp/*`, `/mcp`, `/llms.txt`, `/sitemap-products.xml`, the root sitemap
`/sitemap.xml` and `/relyon/webhooks`. `/checkout` is the app's own
page (see `relyon-checkout`); the kit only uses its path in agent links. Change
paths with the `paths` option only if they clash with the app's routes.
Configured feeds additionally claim `/feeds/openai.jsonl`,
`/feeds/openai.jsonl.gz`, `/feeds/google.xml` and `/feeds/microsoft.xml`;
unselected feed paths pass to the app. `/robots.txt` is claimed only with the
explicit `robots` option. Express and Hono middleware handle these selected
paths automatically; Next.js needs the route files listed above (plus
`app/sitemap.xml/route.js` for the root sitemap and
`app/.well-known/agent-card.json/route.js` for the agent card).

The root `/sitemap.xml` is a urlset that lists the product pages, so crawlers
that only look at `/sitemap.xml` find the products. If the app already serves
its own `/sitemap.xml`, pass `sitemapIndex: false` to the kit and list
`/sitemap-products.xml` in the app's own sitemap index instead. With the
`robots` option, the kit's robots merge adds a `Sitemap:` line for each:

```text
Sitemap: https://shop.example.com/sitemap.xml
Sitemap: https://shop.example.com/sitemap-products.xml
```

## Production: durable state

The defaults keep checkout state, agent events and the catalog **in memory, in
one process**. That is fine for a local Test run, not for production or for
more than one server process. Before the store takes real traffic, plug in:

- `state`: a `CheckoutStateStore` (`get`, `byAgentCheckoutId`, `put`,
  `claimOrder`, `releaseOrder`). `claimOrder` must be an atomic insert, so
  `createOrder` runs once per payment across every process.
- `events.store`: an `AgentEventStore` (`append`, `peek`, `remove`, `size`), so
  queued agent events survive a restart instead of being lost.
- `catalog.persistence`: `{ load, save }` for the synced index (agent checkout's product list).
  Source-backed feeds/discovery read `catalog.source`, not that snapshot;
  the app's product database must itself be durable and available.
- `cartTokens: { secret }`: the app's own secret for hand-off links, so links
  keep working when the API key is rotated.

A Postgres example of the first two is in
[the durable state reference](references/durable-state.md).

## Store policies: explicit synchronization

For a print-on-demand store, configure defects/damage-only returns truthfully.
These local options affect discovery and supported checkout links; creating the
kit or reading discovery never saves policy information to Relyon.

```js
const policies = {
  shippingUrl: "https://shop.example.com/shipping",
  privacyUrl: "https://shop.example.com/privacy",
  termsUrl: "https://shop.example.com/terms",
  contactUrl: "https://shop.example.com/contact",
  returns: {
    windowDays: 30,
    scope: "defective_only",
    policyUrl: "https://shop.example.com/returns",
    conditions: "Made to order. Only items arriving defective or damaged qualify. Report damage with a photograph.",
  },
};
const kit = createStoreKit({
  relyon: { baseUrl: process.env.RELYON_API_BASE_URL, apiKey: process.env.RELYON_API_KEY },
  store: { name: "Example Print Shop", hostnames: ["shop.example.com"] },
  policies,
  priceCart, createOrder,
});

// An explicit setup action, never a constructor or read-request side effect.
await kit.replaceStorePolicies({
  returns: { window_days: 30, cost: "buyer_pays" },
  support: { email: "help@example.com", phone: "+16145550142" },
  consumer_notices: [{ text: "Items are made to order." }],
});
const saved = await kit.storePolicies();
// Check saved.complete and saved.missing; do not assume completeness.
```

**Keep the pages and Relyon's store policies in sync.** Checkout fails closed
while Relyon's store policies are incomplete or don't match what the store's
pages promise, and the kit's doctor names the exact field and the fix (for
example `returns.cost` or `support.email`). `kit.syncStorePolicies()` compares
the `policies` option with what Relyon has stored and returns a plan without
changing anything: `changes` (each field it would set, old and new) and
`needs` (facts only the merchant can supply, such as the return window or who
pays return shipping). Show the plan to the merchant. Only after they
confirm, apply exactly that:

```js
const plan = await kit.syncStorePolicies(); // read-only
// Show plan.changes and plan.needs to the merchant; wait for their answer.
await kit.syncStorePolicies({ confirm: true, returns: { window_days: 30, cost: "buyer_pays" } });
// No contact page? Pass the support email the merchant gave you:
// syncStorePolicies({ confirm: true, ..., support: { email: "help@example.com" } })
```

It never infers free returns or a return cost: `returns.cost` comes only from
the merchant's answer or a value Relyon already stores. Run it again whenever
a policy page changes, so the page and Relyon say the same thing.

Choose `cost` from the store's actual terms, never from this example by default.
Use `free` only when the store pays; `fixed` requires the actual minor-unit
`cost_amount`. A local return option contains no cost, so explicit replacement
requires caller-supplied `returns.window_days` and `returns.cost`. Caller values
win; support email/phone, notices and fixed costs are not cleared by the mapping.
Local contact maps to existing `support.url`, not a duplicate contact field.
The three page links map to `shipping_policy_url`, `privacy_policy_url`, `terms_url`.
Scope, conditions and the return policy URL fill omitted return fields.

`defective_only` qualifies both the reporting window and cost to defective or
damaged items only. Never call this "30-day returns" or "30-day free returns".
`none` requires zero days and accepts no returns. Zero-day `defective_only`
promises no reporting period: refer to conditions and the linked policy.
The API still accepts legacy documents without scope; zero days still means no
returns. The local `all` option retains its existing 1–365 day range.
Conditions are plain text of at most 2000 characters; links are absolute HTTPS,
without credentials, at most 4096 characters. No markup or scripts.

Replacement clears fields absent from both the explicit input and local options.
Without local options, the existing `replaceStorePolicies(input)` sends the
contract input unchanged. No new fields are required; completeness still checks
only returns, its policy URL and support email or URL. Do not synchronize from a
constructor, product read, checkout read or discovery request.

## Keys, scopes and requests

- Only the server uses `RELYON_API_KEY`. Never put it in a URL, a frontend
  bundle, an agent prompt, an example or telemetry.
- Test and Live have separate keys, resources, webhook secrets and idempotency
  records. Check `livemode: false` on what Relyon returns.
- The kit sends `Relyon-Version: 2026-10-30`, Bearer authentication and its
  own `Idempotency-Key` values. Don't add a second client beside it for the
  same calls.
- Never put card numbers, CVV/CVC, expiry dates or raw card payloads in
  product data, metadata, logs or requests. If Relyon answers
  `card_data_rejected`, remove the field; never rename or encode it to get
  past the check.

## Install these skills and the CLI

`npx @relyon/store-kit init` installs the kit and these skills for Codex,
Replit Agent and OpenCode (`.agents/skills`), Claude Code (`.claude/skills`)
and Cursor (`.cursor/skills`); it needs nothing else. The optional `relyon`
CLI is served by Relyon, not the npm registry: `npm install -g https://core.getrelyon.com/cli/relyon.tgz` (or project-local
`npm i -D <that URL>` and `npx relyon …`). One-shot:
`npx -y https://core.getrelyon.com/cli/relyon.tgz agent setup`. The SHA-256 is
at `https://core.getrelyon.com/cli/relyon.tgz.sha256`. Commands use
`https://core.getrelyon.com`, the CLI's default base URL. Set
`RELYON_BASE_URL` (or `--base-url`) only for another deployment its operator
confirmed.

`relyon agent setup` installs the six bundled skills offline, with no key, for
Claude Code (`.claude/skills`), Cursor (`.cursor/skills`), Codex and Replit
Agent (`.agents/skills`) and OpenCode (`.opencode/skills`). Claude Code users
can install the Relyon plugin instead.

## Diagnose without overclaiming

Run the kit's doctor against the published store, with `RELYON_API_KEY`
already in the server environment from the secrets tool (never on the command
line):

```sh
STORE_PUBLIC_URL=https://shop.example.com npx @relyon/store-kit doctor
```

It checks what agents see on the store's public URL (the UCP profile, the
verification file, `llms.txt`, the product sitemap, selected feeds, MCP and
`robots.txt`), then runs Relyon's integration doctor (`GET /v1/doctor`). It
prints PASS, FAIL, N/A or UNVERIFIED per item, never prints the key, and exits
nonzero only when an item FAILs. It never creates a shopper checkout or submits
a payment; Relyon's server diagnostic may create and clean up one temporary
system-owned checkout and sends one webhook challenge.
`--feeds openai,google` names the selected feeds. It checks Test unless given
`--live` (or `RELYON_LIVE=true`) with a Live key; never pass either (see
`relyon-go-live`).

`relyon doctor` (the CLI) runs only `GET /v1/doctor`. HTTP 200 can still
contain `status: fail`; read every failure. For a store-first app, only these
checks are goals:

- `keys`: a Test restricted key exists and has the scopes it needs.
- `catalog`: with store-owned `catalog.source` callbacks, Relyon doesn't need
  its index; once it has observed the store's own discovery, an empty mirror
  is not a failure. Relyon counts the app as store-first when the store's
  domain is verified (within step 1) and it has seen the kit's UCP profile, MCP or
  `llms.txt` on that domain, or received the kit's agent events; run the
  doctor again after setup's domain verification. If the store's own discovery passes and
  `catalog` still fails only because the index is empty, run
  `kit.syncCatalog`: agent checkout needs the products there anyway. Without `catalog.source`, it
  passes once at least one valid, in-stock product is published to the index.
- `webhook`: Relyon challenges one active webhook endpoint. The kit's
  `/relyon/webhooks` route answers the challenge itself once
  `RELYON_WEBHOOK_SECRET` holds that endpoint's secret. Any other endpoint the
  app registers must answer it too (see `relyon-orders`).

The doctor's `pricing` and `checkout` checks test an older Relyon-side flow
that store-first apps don't use. The kit's doctor shows them as N/A when, in
Test, the same run saw the store's UCP profile, verification file, `llms.txt`
and MCP on its domain; `relyon doctor` may still list them. Don't build
anything to make them pass. A store-first blocker Relyon reports under those
checks (its explanation says "Observed blocker" or "Unverified") stays a FAIL:
fix it.

A contracted operation that answers 404, 405 or 501 is not available on that
deployment. Report it; never call an internal or dashboard route instead.
