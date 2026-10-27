# Express server with the store kit

A complete Express 5 app: the kit's routes, a homepage with the product list,
a product page, the policy pages its footer links to, the checkout page on the
store's domain, and the pay, confirm and order status routes.
`store.js` is in the `relyon-catalog` skill (`references/example-store.md`),
`relyon.js` is in this skill's `SKILL.md`, and `checkout-flow.js` (the
server's checkout records) and `checkout.js` (the browser script) are in the
`relyon-checkout` skill (`references/checkout-flow.md` and
`references/checkout-page.md`).

The order of middleware matters: the kit is mounted **before** `express.json()`.

```js server.js
import express from "express";
import { PaymentUnavailableError, StoreRefusal } from "@relyon/store-kit";
import { storeKitMiddleware } from "@relyon/store-kit/express";
import { createCheckoutFlow, memoryCheckouts } from "./checkout-flow.js";
import { catalogSource, findOrderByPayment, getProduct, priceCart } from "./store.js";

const escape = (text) => String(text).replace(/[&<>"']/g, (c) => `&#${c.charCodeAt(0)};`);
const money = (amount, currency) =>
  new Intl.NumberFormat("en-US", { style: "currency", currency }).format(amount / 100);

/**
 * The Web Request the kit's helpers read, built from an Express request.
 * The kit only trusts its Host when it is one of the store's own or Test
 * preview hostnames; plain http is used for loopback (localhost) only.
 */
export function webRequest(req) {
  const headers = new Headers();
  for (const [name, value] of Object.entries(req.headers)) {
    if (typeof value === "string") headers.set(name, value);
    else if (Array.isArray(value)) for (const item of value) headers.append(name, item);
  }
  const host = req.headers.host ?? "localhost";
  const loopback = /^(localhost|127\.0\.0\.1|\[::1\])(:\d+)?$/i.test(host);
  const scheme = loopback && !req.socket?.encrypted ? "http" : "https";
  return new Request(`${scheme}://${host}${req.originalUrl}`, { method: req.method, headers });
}

/**
 * The policy pages the homepage footer links to. Write them from the
 * merchant's own answers (relyon-setup: "Ask before publishing claims").
 * Until the merchant confirms a fact, the page says "to confirm with the merchant" rather than
 * inventing a return window, a return cost, tax handling or a brand claim.
 * Display copy only: never pass POLICY_PAGES to policy sync, kit policies,
 * feeds, structured policy data or API payloads.
 */
export const POLICY_PAGES = {
  shipping: { title: "Shipping", text: "Where we ship, delivery times and costs: to confirm with the merchant." },
  returns: { title: "Returns", text: "Return window, which items qualify and who pays return shipping: to confirm with the merchant." },
  privacy: { title: "Privacy", text: "How this store handles your personal data: to confirm with the merchant." },
  terms: { title: "Terms", text: "Terms of sale, including how tax is handled: to confirm with the merchant." },
};

/** The footer every page shows: links to the four policy pages. */
const footer = () => `<footer><nav aria-label="Store policies">${Object.entries(POLICY_PAGES)
  .map(([path, page]) => `<a href="/${path}">${escape(page.title)}</a>`).join(" · ")}</nav></footer>`;

/** One cookie value from the request, or null. */
const cookie = (req, name) => {
  for (const part of (req.headers.cookie ?? "").split(";")) {
    const [key, ...value] = part.trim().split("=");
    if (key === name) return decodeURIComponent(value.join("="));
  }
  return null;
};

/** The order status page: no card fields, Pay can't be pressed here. */
function statusPage(view, newCheckoutHref) {
  const { state, order_reference: ref, message, payment_succeeded: paymentSucceeded } = view.body;
  const statusHref = `/checkout/status?order_reference=${encodeURIComponent(ref ?? "")}`;
  return `<!doctype html><meta charset="utf-8"><title>Order status</title>
<h1>Order status</h1>
<div id="checkout-status" data-order-reference="${escape(ref ?? "")}" data-state="${escape(state)}" data-payment-succeeded="${paymentSucceeded === true}">
<p id="status" role="status">${escape(message)}${state === "pending" ? ` <a href="${escape(statusHref)}">Check your order status</a>` : ""}</p>
<p>Order reference: ${escape(ref ?? "unknown")}</p>
</div>
${newCheckoutHref && state !== "pending" ? `<p><a href="${escape(newCheckoutHref)}">Start a new order</a></p>` : ""}
<script type="module" src="/checkout.js"></script>`;
}

/**
 * `checkouts` holds each checkout by order reference (checkout-flow.js in
 * relyon-checkout). Use the app's database in production
 * (`postgresStoreCheckouts` in durable-state.js).
 */
export function createApp(kit, { checkouts = memoryCheckouts() } = {}) {
  const flow = createCheckoutFlow(kit, { priceCart, checkouts, findOrder: async (paymentId) => findOrderByPayment(paymentId) });
  const app = express();
  // Run app.locals.reconcile() on a timer: replays payments still being confirmed.
  app.locals.reconcile = (options) => flow.reconcile(options);
  // 1. The kit first: it needs the raw body of its own routes. It serves the
  // discovery files too: /.well-known/ucp, /.well-known/agent-card.json,
  // /llms.txt, /sitemap-products.xml, the selected feeds and the root sitemap
  // /sitemap.xml, a urlset of the product pages (sitemapIndex: false turns
  // that off when the app has its own /sitemap.xml).
  app.use(storeKitMiddleware(kit));
  // 2. Then the app's own parsers, static files and routes.
  app.use(express.json({ limit: "64kb" }));
  app.use(express.static("public")); // serves public/checkout.js

  // The homepage: the product list with its Product data, the discovery links
  // agents look for in <head> (llms.txt, UCP, MCP, ACP, the agent card, the
  // sitemap and the feeds) and the policy footer. Every link uses the store's
  // public address for this request (kit.publicOrigin), never a dev address.
  app.get("/", async (req, res) => {
    const request = webRequest(req);
    kit.recordCrawlerVisit(request, 200, { ip: req.ip });
    const { products } = await catalogSource.listProducts({ limit: 100 });
    res.type("html").send(`<!doctype html><html lang="en"><head><meta charset="utf-8"><title>Example Candle Co</title>
${kit.renderDiscoveryLinks(request)}
${kit.renderProductListJsonLd(products, request)}
</head><body>
<h1>Example Candle Co</h1>
<ul>${products.map((product) => `<li><a href="/products/${encodeURIComponent(product.merchant_product_id)}">${escape(product.title)}</a>
 ${money(product.from_price.amount, product.from_price.currency)}${product.availability === "in_stock" ? "" : " (out of stock)"}</li>`).join("")}</ul>
${footer()}
</body></html>`);
  });

  // A product page the app renders from its own data. Pass the request so the
  // Product data's links and images use the public address.
  app.get("/products/:id", (req, res) => {
    const request = webRequest(req);
    const product = getProduct(req.params.id);
    kit.recordCrawlerVisit(request, product ? 200 : 404, { ip: req.ip });
    if (!product) return res.status(404).send("Not found");
    const image = kit.productJsonLd(product, request).image?.[0];
    res.type("html").send(`<!doctype html><html lang="en"><head><meta charset="utf-8"><title>${escape(product.title)}</title>
${kit.renderProductJsonLd(product, request)}
</head><body>
<h1>${escape(product.title)}</h1>
${image ? `<img src="${escape(image)}" alt="${escape(product.title)}" width="480">` : ""}
<p>${escape(product.description)}</p>
<p>${money(product.from_price.amount, product.from_price.currency)}</p>
<form method="post" action="/cart"><input type="hidden" name="product" value="${escape(product.merchant_product_id)}">
<input type="number" name="quantity" value="1" min="1" max="99" aria-label="Quantity"> <button type="submit">Buy now</button></form>
${footer()}
</body></html>`);
  });

  // "Buy now": the form posts here, and a 303 sends the shopper to checkout.
  // kit.redirect(request, path) builds the Location on this request's validated store
  // address. Never build a Location from req.url or a Web request.url: a dev
  // server bound to 0.0.0.0 can name that bind address, which no browser opens.
  app.post("/cart", express.urlencoded({ extended: false, limit: "8kb" }), (req, res) => {
    const product = getProduct(String(req.body?.product ?? ""));
    if (!product) return res.status(404).send("Not found");
    const quantity = Math.min(99, Math.max(1, Math.trunc(Number(req.body?.quantity) || 1)));
    const query = new URLSearchParams({ product: product.merchant_product_id, quantity: String(quantity) });
    const seeOther = kit.redirect(webRequest(req), `/checkout?${query}`);
    res.status(seeOther.status).set("cache-control", "no-store").location(seeOther.headers.get("location")).end();
  });

  // The policy pages the footer links to.
  for (const [path, page] of Object.entries(POLICY_PAGES)) {
    app.get(`/${path}`, (req, res) => {
      res.type("html").send(`<!doctype html><html lang="en"><head><meta charset="utf-8"><title>${escape(page.title)}</title></head><body>
<h1>${escape(page.title)}</h1><p>${escape(page.text)}</p>
${footer()}
</body></html>`);
    });
  }

  // The checkout page, on the store's own domain.
  app.get("/checkout", async (req, res) => {
    if (kit.checkoutStatus === "waiting for sandbox key") {
      return res.status(503).type("html").send(`<!doctype html><meta charset="utf-8"><title>Checkout</title>
<h1>Checkout</h1><p role="status">Checkout is waiting for sandbox key.</p><button disabled>Pay</button>
<a href="/products/lavender-candle">Continue shopping</a>`);
    }
    const previous = cookie(req, "checkout_ref");
    const startNew = req.query.new === "1";
    const again = new URLSearchParams({ ...req.query, new: "1" });
    // Check saved payment evidence even when there is no usable cart on reload.
    const prior = await flow.status(previous);
    if (prior.body.state === "pending" || (prior.body.state === "paid" && !startNew)) {
      const view = prior.body.state === "pending" ? await flow.confirm(previous) : prior;
      return res.status(view.httpStatus).type("html").send(statusPage(view, `/checkout?${again}`));
    }
    // A shopper an agent handed over arrives with ?relyon_cart=…; anyone else
    // gets the app's own cart. Only an unattributed or invalid link falls back,
    // never a handed-over cart the store refused, a failure pricing that
    // independent cart, or a failure obtaining its payment fields.
    let landing;
    try {
      landing = await kit.landing(webRequest(req), { ip: req.ip });
    } catch {
      landing = null;
    }
    if (landing?.alreadyPaid) {
      return res.status(202).type("html").send(`<!doctype html><meta charset="utf-8"><title>Order status</title>
<h1>Order status</h1><p role="status">${landing.alreadyPaid.orderPending
  ? "Your payment went through. Your order is still being placed"
  : "This order may already have been paid. Check your order status before paying again."}</p>`);
    }
    if (landing?.attributed && landing.refusal) {
      // The store refused the cart the agent handed over: say why, never swap in another cart.
      return res.status(409).type("html").send(`<!doctype html><meta charset="utf-8"><title>Checkout</title>
<h1>Checkout</h1><p role="alert">${escape(landing.refusal.message)}</p><p><a href="/products/lavender-candle">Start shopping</a></p>`);
    }
    const accepted = landing?.attributed && !landing.refusal
      && landing.cart?.lines?.length > 0 && landing.inputs?.lines?.length > 0;
    // This example's independent cart source is the product/quantity query.
    // A real store can read a shopper session instead.
    const product = String(req.query.product ?? "");
    const inputs = accepted ? landing.inputs : {
      lines: product ? [{ product_id: product, quantity: Number(req.query.quantity ?? 1) }] : [],
      address: null, coupons: [], shipping_option_id: "ground",
    };
    if (!inputs.lines.length) {
      return res.type("html").send(`<!doctype html><meta charset="utf-8"><title>Checkout</title>
<h1>Checkout</h1><p>Your cart is empty.</p><p><a href="/products/lavender-candle">Start shopping</a></p>`);
    }
    let opened;
    try {
      opened = await flow.open({
        inputs, cart: accepted ? landing.cart : null,
        channel: accepted ? landing.channel : "web", agentCheckoutId: accepted ? landing.agentCheckoutId : null,
        // A reload while a payment is being confirmed shows its status, never new card fields.
        previous, startNew,
      });
    } catch (error) {
      if (error instanceof StoreRefusal) return res.status(error.status).send(escape(error.message));
      if (error instanceof PaymentUnavailableError) return res.status(503).send("Payments are unavailable right now. Try again shortly.");
      throw error;
    }
    const { orderReference } = opened;
    res.cookie("checkout_ref", orderReference, { httpOnly: true, secure: true, sameSite: "lax", path: "/" });
    if (opened.view) {
      return res.status(opened.view.httpStatus).type("html").send(statusPage(opened.view, `/checkout?${again}`));
    }
    const { cart, fields } = opened;
    const t = cart.totals;
    res.type("html").send(`<!doctype html><meta charset="utf-8"><title>Checkout</title>
<h1>Checkout</h1>
<ul>${cart.lines.map((line) => `<li>${escape(line.title)} × ${line.quantity}: ${money(line.amount, cart.currency)}</li>`).join("")}</ul>
<p>Subtotal ${money(t.subtotal, t.currency)} · Discount ${money(t.discount, t.currency)} · Shipping ${money(t.shipping, t.currency)} · Tax ${money(t.tax, t.currency)}</p>
<p><strong>Total ${money(t.total, t.currency)}</strong></p>
<form id="pay" data-order-reference="${escape(orderReference)}" data-new-checkout="${escape(`/checkout?${again}`)}">
<script>
  // Ready latch (relyon-checkout's checkout-page.md): before the card fields, so it never misses "relyon-ready".
  addEventListener("relyon-ready", (event) => { event.target.dataset.relyonReady = "true"; }, true);
  addEventListener("relyon-error", (event) => { delete event.target.dataset.relyonReady; }, true);
</script>
${fields.html}
<button type="submit">Pay ${money(t.total, t.currency)}</button>
<p id="status" role="status"></p>
</form>
<script type="module" src="/checkout.js"></script>`);
  });

  const answer = (res, view) => res.status(view.httpStatus).json(view.body);

  // The browser sends only the order reference and the tokenized payment method.
  app.post("/checkout/pay", async (req, res) => {
    if (kit.checkoutStatus === "waiting for sandbox key") return res.status(503).json({ code: "sandbox_key_pending", message: "Checkout is waiting for sandbox key." });
    const { orderReference, paymentMethod } = req.body ?? {};
    answer(res, await flow.pay({ orderReference, paymentMethod, request: webRequest(req), info: { ip: req.ip } }));
  });

  // After a lost answer: replays the attempt already sent, never a new one.
  app.post("/checkout/confirm", async (req, res) => {
    answer(res, await flow.confirm(req.body?.orderReference));
  });

  // The order status page a shopper can come back to.
  app.get("/checkout/status", async (req, res) => {
    const view = await flow.status(String(req.query.order_reference ?? ""));
    res.status(view.httpStatus).type("html").send(statusPage(view, null));
  });

  return app;
}
```

Start it and stop cleanly. Source-backed discovery and feeds need no sync,
but Relyon's agent checkout (MCP, UCP and ACP) only buys products it has in
the store's Relyon catalog, so the starter sends them once a key is set.
A plain Node/Express server doesn't read `.env` by itself: `loadTestConfig()`
below loads the Test key init saved (or use `dotenv`). Never cat or print
`.env`, and never paste the key into a command.

```js main.js
import { createRelyonKit } from "./relyon.js";
import { createApp } from "./server.js";
import { loadTestConfig } from "@relyon/store-kit/server-config";
import { listProducts } from "./store.js";

const privateConfig = await loadTestConfig({ project: process.cwd() });
const kit = createRelyonKit({
  env: { ...process.env, ...(privateConfig.apiKey ? { RELYON_API_KEY: privateConfig.apiKey } : {}) },
  // robots.txt is served by default (AI search and shopping crawlers allowed,
  // both sitemaps named). An app with its own robots.txt passes
  // robots: { existing: () => itsText } so every line is kept; robots: false opts out.
});
await kit.start();
// catalog.source serves discovery and feeds from the store's database. Agent
// checkout buys only products in the store's Relyon catalog: send them now and
// again whenever the merchant edits products. A failed sync changes nothing.
if (kit.checkoutStatus === "ready") {
  await kit.syncCatalog(await listProducts()).catch((error) => console.error("catalog sync failed", error.message));
}
const app = createApp(kit);
const server = app.listen(Number(process.env.PORT ?? 3000), "0.0.0.0");
// Payments whose outcome wasn't final are replayed (same Idempotency-Key) until it is.
const reconcile = setInterval(() => {
  app.locals.reconcile().catch((error) => console.error("reconcile failed", error.message));
}, 60_000);
process.on("SIGTERM", () => {
  clearInterval(reconcile);
  server.close();
  void kit.stop(); // flushes queued agent events
});
```

## Sitemaps and robots.txt

The kit answers the root `/sitemap.xml` with a urlset of the product pages
(the same entries as `/sitemap-products.xml`), on the store's public address:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url><loc>https://shop.example.com/products/lavender-candle</loc></url>
  <url><loc>https://shop.example.com/products/candle-trio</loc></url>
</urlset>
```

The kit serves `/robots.txt` by default. With `robots: { existing }`
(`existing()` returns the app's own robots.txt), the merged file keeps every
original line and adds the AI discovery section. Either way it ends with a
`Sitemap:` line for the root sitemap and one for the product sitemap:

```text
Sitemap: https://shop.example.com/sitemap.xml
Sitemap: https://shop.example.com/sitemap-products.xml
# llms.txt: https://shop.example.com/llms.txt
```

A new store has no robots.txt of its own, so the starter's `main.js` uses the
default. An Express app with its own robots route mounted after the kit must
pass `robots: { existing }` or `robots: false`, or the kit's answer shadows it.
Some hosts and tunnels add
their own robots.txt rules in front of the app's (blocking AI crawlers by
default); the kit's explicit `Allow: /` groups for AI search and shopping
crawlers still apply, but check the host's bot settings too. With
`robots: false`, add those two `Sitemap:` lines to the app's own robots.txt by
hand.

## On the preview address

In Test mode the same server works on the app's preview address, on any
platform: an https preview URL or tunnel, or `localhost` on its port over
plain http. The kit adds loopback, the hostnames in `RELYON_PREVIEW_HOSTS`
(set it in your platform's secrets or environment settings) and the app's own
address when its platform names it in a well-known variable to
`store.hostnames` itself, so agents can run a whole Test checkout there, and
the hand-off link comes back to this same address. A Live key is refused there: publish, add
and verify the domain, then switch to a Live key only with the merchant's
approval (`relyon-go-live`).
