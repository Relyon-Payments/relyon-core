# Express server with the store kit

A complete Express 5 app: the kit's routes, a product page, the checkout page
on the store's domain, and the pay, confirm and order status routes.
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
import { findOrderByPayment, getProduct, priceCart } from "./store.js";

const escape = (text) => String(text).replace(/[&<>"']/g, (c) => `&#${c.charCodeAt(0)};`);
const money = (amount, currency) =>
  new Intl.NumberFormat("en-US", { style: "currency", currency }).format(amount / 100);

/** The Web Request the kit's helpers read, built from an Express request. */
export function webRequest(req) {
  const headers = new Headers();
  for (const [name, value] of Object.entries(req.headers)) {
    if (typeof value === "string") headers.set(name, value);
    else if (Array.isArray(value)) for (const item of value) headers.append(name, item);
  }
  return new Request(`https://${req.headers.host ?? "localhost"}${req.originalUrl}`, { method: req.method, headers });
}

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
  // 1. The kit first: it needs the raw body of its own routes.
  app.use(storeKitMiddleware(kit));
  // 2. Then the app's own parsers, static files and routes.
  app.use(express.json({ limit: "64kb" }));
  app.use(express.static("public")); // serves public/checkout.js

  // A product page the app renders from its own data.
  app.get("/products/:id", (req, res) => {
    const product = getProduct(req.params.id);
    kit.recordCrawlerVisit(webRequest(req), product ? 200 : 404, { ip: req.ip });
    if (!product) return res.status(404).send("Not found");
    res.type("html").send(`<!doctype html><meta charset="utf-8"><title>${escape(product.title)}</title>
${kit.renderProductJsonLd(product)}
<h1>${escape(product.title)}</h1>
<p>${escape(product.description)}</p>
<p>${money(product.from_price.amount, product.from_price.currency)}</p>
<a href="/checkout?product=${encodeURIComponent(product.merchant_product_id)}&quantity=1">Buy now</a>`);
  });

  // The checkout page, on the store's own domain.
  app.get("/checkout", async (req, res) => {
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
<form id="pay" data-order-reference="${escape(orderReference)}">
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

Start it and stop cleanly. No catalog sync is needed for source-backed feeds:

```js main.js
import { createRelyonKit } from "./relyon.js";
import { createApp } from "./server.js";

const kit = createRelyonKit();
await kit.start();
// catalog.source reads the store's database. syncCatalog is an optional mirror only.
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
