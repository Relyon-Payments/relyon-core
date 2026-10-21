---
name: relyon-checkout
description: Build checkout on the store's own domain with @relyon/store-kit - the app's priceCart, a checkout page, kit.cardFields for the only Relyon browser element, tokenize() and kit.createPayment charging exactly the store's cart total, plus relyon_cart hand-off landings.
---

# Relyon checkout

<!-- BEGIN relyon-store-first-rules (generated from the shared rules source; edit it there) -->
## Relyon store-first rules

These seven rules are identical in every Relyon skill. They override anything else you read.

1. Install `@relyon/store-kit@0.1.1` from npm and mount its handlers on the app's own server (Express `@relyon/store-kit/express`, Next.js `@relyon/store-kit/next`, Hono `@relyon/store-kit/hono`).
2. Catalog, cart and pricing stay in the app: the app owns `listProducts`, `getProduct`, `priceCart` and `createOrder`. Relyon never prices a cart. It charges exactly the final amount the store sends.
3. Checkout runs on the store's own domain. The only Relyon element in the shopper's browser is the card fields component. Card numbers never touch the app's code, server or logs.
4. The app calls Relyon only from its server, and only for payment, agent verification, fraud and limits, and recording.
5. Agent activity reporting stays on: never stub, disable or throttle-to-zero the kit's event reporting.
6. Test/sandbox only. Live is enabled by Relyon staff, never by the agent.
7. No Relyon-hosted checkout, redirect to a Relyon page, CNAME, proxy, forwarding Worker or `quote.request`/pricing endpoint.
<!-- END relyon-store-first-rules -->

Contract: the approved public v1 contract dated **2026-10-30**, at
`https://core.getrelyon.com/v1/openapi.yaml` (another deployment:
`<service origin>/v1/openapi.yaml`). The kit calls `POST /v1/payment_sessions` and
`POST /v1/payments` for the app.

The canonical API base is `https://core.getrelyon.com/api`; `/v1/payments`
therefore means `https://core.getrelyon.com/api/v1/payments`. The download URL
above and the store's own routes are separate. Retain operator-confirmed API
base overrides for other deployments.

**In the Start here flow** (`relyon-setup`) this is step 4, after the catalog,
and step 10's Test purchase. Shipping regions, rates and tax come from the
merchant: ask, and check what `relyon-store-brief.md` says, but never invent
them.

## The flow

1. The shopper builds a cart in the app. The app stores it server-side.
2. The app's checkout page (default path `/checkout`, on the store's domain)
   prices the cart with the app's own `priceCart` and shows every line,
   discount, shipping, tax and the total.
3. The server calls `kit.cardFields(...)` for exactly that total and puts the
   returned HTML in the page.
4. In the browser, `tokenize()` turns the card into a payment method. The page
   sends the order reference and the payment method, and nothing else, to the
   app's server.
5. The server re-prices the stored cart, checks the total hasn't changed,
   records the attempt and calls `kit.createPayment(...)`. On success the kit
   calls `createOrder` once.
6. If the answer is lost anywhere (store to Relyon, or browser to store), the
   checkout stays "pending": the page keeps Pay disabled, points to the order
   status and asks the server, which only replays the recorded attempt.

The server half is [the checkout flow reference](references/checkout-flow.md)
(`checkout-flow.js`); a complete server is in the `relyon-setup` skill
(`references/express-server.md`); the browser script is in
[the checkout page reference](references/checkout-page.md).

## priceCart: the store prices every cart

The kit calls the app's `priceCart` for every agent checkout step and every
hand-off landing; the app calls it for its own checkout page too.

```js
async function priceCart(lines, address, coupons, context) { /* returns a Cart */ }
```

- `lines`: `[{ product_id, quantity, options? }]`. `product_id` is the app's
  `merchant_product_id`.
- `address`: the shipping address, or `null` before the shopper or agent gave one.
- `coupons`: codes to apply (validate them; refuse unknown ones).
- `context`: `{ protocol, operation, shippingOptionId, agentCheckoutId }`.
  `protocol` is `ucp`, `acp`, `mcp` or `web`.

Return a `Cart` with one line per input line, in the same order:

```json
{
  "currency": "USD",
  "lines": [
    { "id": "line_1", "merchant_product_id": "lavender-candle", "title": "Lavender candle", "quantity": 2, "unit_amount": 2400, "amount": 4800 }
  ],
  "discounts": [{ "code": "WELCOME10", "label": "10% off your first order", "amount": 480 }],
  "shipping_options": [{ "id": "ground", "label": "Ground (3-5 days)", "amount": 599 }],
  "shipping_option_id": "ground",
  "totals": { "currency": "USD", "subtotal": 4800, "discount": 480, "shipping": 599, "tax": 302, "total": 5221 }
}
```

It must add up: the lines' `amount` sum to `totals.subtotal`, `discounts` sum
to `totals.discount`, and `totals.total` is subtotal − discount + shipping +
tax, all integers in minor units, with matching currencies. Relyon checks the
sums and charges exactly `totals.total`; it never changes a price. To refuse a
cart (out of stock, invalid option or coupon), throw
`new StoreRefusal(code, message, status)` from `@relyon/store-kit`: agents get
it in their protocol's error format.

## Card fields on the store's page

```js
const fields = await kit.cardFields({ amount: cart.totals.total, currency: cart.currency, orderReference });
// fields.html = fields.scriptTag + fields.element; fields.session is the payment session
```

- Creates a fresh payment session (`POST /v1/payment_sessions`) for exactly
  that amount. Sessions are single use: call it on every page render and
  every new attempt.
- `orderReference` is the app's own reference for this checkout attempt. Store
  it server-side with the cart.
- Optional: `fieldStyles` (styling passed to the fields), `elementId`,
  `metadata` (no card or personal data).
- `fields.html` is the only Relyon markup in the page. Don't load other Relyon
  scripts, frames or pages in the shopper's browser.
- Throws `PaymentUnavailableError` when Relyon can't be reached. Creating card
  fields never charges, so show "try again shortly"; never fall back to
  collecting card data yourself.

## createPayment: charge the store's total

```js
const result = await kit.createPayment({
  cart,                 // re-priced on the server just now; totals.total is charged
  paymentMethod,        // exactly what tokenize() returned, unchanged
  orderReference,       // same reference the card fields were created for
  channel, agentCheckoutId, // from kit.landing for a handed-off shopper; omit otherwise
  request,              // the shopper's Web Request, so agent rules apply at the edge
});
```

- The `Idempotency-Key` is derived from `orderReference`: retrying with the
  same reference never charges twice and returns the same payment. A new
  attempt after a decline needs a new reference and new card fields.
- Re-price the stored cart before charging. If the total changed since the
  card fields were created, don't charge: show the new total, create new card
  fields and get the shopper's confirmation again.
- `result.payment.status === "succeeded"`: the kit called `createOrder` once
  (`result.orderCreated`, `result.order`). `result.orderError` means the
  payment succeeded but `createOrder` threw: the shopper was charged, so show
  it as pending (never "didn't go through"), keep the attempt (cart and
  payment method) and replay it with the same reference, which charges
  nothing new and retries the order.
- Any other status: nothing to fulfill. Tell the shopper and let them retry.
- `PaymentOutcomeUnknownError`: Relyon answered `202 pending`, timed out,
  failed or sent an unusable reply, so **a charge may have succeeded**. It
  carries `orderReference`, `idempotencyKey` and `reason`. Keep the open
  checkout and the exact attempt (the same cart and payment method), tell the
  shopper the payment is being confirmed and not to pay again, and replay
  that same attempt with the same reference (from the page or a background
  job) until Relyon gives a final answer. The replay reuses the
  `Idempotency-Key`, so it never charges twice. Never start a new reference,
  re-price into a different cart, create an order, or tell the shopper
  nothing was charged on this error.
- Other errors: `AgentRefusedError` (`agent_refused` or `channel_paused` from
  the store's agent rules) and `RelyonProblemError` with a `code`:

| Code | What to do |
| --- | --- |
| `amount_mismatch` | The amount differs from the payment session or the cart doesn't add up. Fix `priceCart`, create new card fields and charge again. Nothing was charged. |
| `processor_not_configured` | No processor account in this environment yet. Relyon staff set it up; report it (see `relyon-go-live`). |
| `card_data_rejected` | A request carried something that looks like card data. Remove it; never rename or encode it. |
| `idempotency_key_reused` | The same order reference was used with a different cart or amount. Use a new reference for a new attempt. |
| `permission_denied` | The key lacks a scope. Report it; don't switch keys. |

**Two separate wire contracts:** agent checkout accepts the contract `Cart`,
including `currency`, line `id`/`merchant_product_id` and shipping fields.
That agent-cart support does not change `POST /v1/payments`. Today's Test
payment parser still requires `lines[].product_id` and rejects contract-only
fields. Keep `paymentCartFormat: "server-legacy"` (as `relyon-setup` shows).
The app's `priceCart` still returns the contract `Cart`; only the kit's
payment request converts it to `{ lines, discounts, totals }`, maps each
`merchant_product_id` to `product_id`, and omits line `id`, top-level
`currency`, `shipping_options`, `shipping_option_id` and `shipping_address`.
Do not change the application cart schema or kit default. Remove the option
only when Relyon explicitly announces contract carts for payments too.

For known charged but unsaved orders show exactly:
"Your payment went through. Your order is still being placed".
Keep Pay disabled and the original attempt; retry order placement with that
same reference, never a second charge. Unknown payment outcomes instead keep
the cautious confirmation message. Neither case permits a completed-order or
fulfillment claim before a saved order exists.

## Lost answers, retries and reloads: never two charges

Use `checkout-flow.js` ([checkout flow reference](references/checkout-flow.md))
or keep the same guarantees in the app's own code:

- Record the attempt (cart and payment method) with one conditional write
  before calling `createPayment`. Only the request that wins it sends.
- After that, every retry, second click, status check, background job or
  reload replays that recorded attempt with the same order reference, so the
  `Idempotency-Key` is the same. A new payment method from the browser is
  ignored.
- The browser treats a lost connection, a timeout, a `5xx` or `pending` as
  "the payment may have gone through": keep Pay disabled, say so, link to the
  order status and keep asking the server. Enable Pay again only when the
  server says nothing was sent (`open`).
- Remember the shopper's order reference (a cookie or local storage). A
  reload while it is pending shows its status, never new card fields.
- Only a final answer closes a checkout: a declined payment, or a refusal of
  the first send. Then a reload starts a new checkout with a new reference.

## Hand-off landings (`relyon_cart`)

Policy setup is explicit, not a checkout-read side effect: follow the complete
print-on-demand example in `relyon-setup`. Saved privacy/terms links and supported
shipping/return links travel in existing protocol `links`; do not invent policy
summary properties. Contact uses `support.url`, `support.email`, `support.phone`.
Only describe `defective_only` as defects/damage-only, including its reporting
window and cost. Zero days promises no reporting period; `none` accepts no
returns. Keep plain-text conditions intact and never insert them as active HTML.

When an agent hands a shopper over to the store to finish paying, the shopper
lands on the store's checkout path with `?relyon_cart=…`.
`await kit.landing(request)` returns:

- `attributed: true`, `cart` (re-priced through `priceCart` just now),
  `inputs`, `agentCheckoutId`, `protocol` and `channel` for a valid link.
  Charge with `channel` and `agentCheckoutId` from it.
- `refusal` when `priceCart` refused the landed cart (show its message).
- `attributed: false` for no, expired or tampered links: a normal checkout
  with the app's own cart. It never throws because of the link.

The kit records the landing as agent activity. Links come from
`kit.handoffUrl(protocol, sessionId)` (see `relyon-agents`).

## Never

- Never take a price, total or discount from the browser or from an agent
  message; always re-price on the server.
- Never enable Pay again, start a new order reference or tell the shopper the
  payment didn't go through after a payment was sent, unless the server has a
  final answer.
- Never fulfill on a thank-you page load alone. A succeeded `createPayment`
  result and `createOrder` are the record of payment.
- Never collect card numbers in the app's own inputs, and never log the
  payment method object.
