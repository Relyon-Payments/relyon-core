# Checkout page in the browser

The checkout page is the app's own page on the store's domain (the
`relyon-setup` skill's `references/express-server.md` renders one, with
`checkout-flow.js` from `references/checkout-flow.md` on the server). The server
puts `fields.html` from `kit.cardFields` into the page: the card fields
script tag and one `<relyon-card-fields>` element. That element is the only
Relyon code in the browser. The card number is typed into fields the
processor hosts, and the app only ever sees the tokenized payment method.

The card fields script loads `async`, and in Test the fields announce
`relyon-ready` as soon as the element upgrades. That can happen before the
page's own script has attached its listener (for example when the script
comes from the browser cache), so the page records readiness itself. Put
this ready latch in the page **before** `fields.html`; it runs before the
card fields script can, and marks the element once it is ready:

```html
<script>
  addEventListener("relyon-ready", (event) => { event.target.dataset.relyonReady = "true"; }, true);
  addEventListener("relyon-error", (event) => { delete event.target.dataset.relyonReady; }, true);
</script>
```

## Sending the shopper to checkout from the server

A "Buy now" or cart form that posts to the app answers with
`kit.redirect(request, path)`: a `303 See Other` to `path` (starting with a
single `/`) on this request's validated store address. Never build that
Location or a link from `request.url` (for example
`new URL("/checkout", request.url)`): a dev server bound to every interface
(`next dev --hostname 0.0.0.0`) hands the app a `request.url` on `0.0.0.0`,
which no browser can open, so the pay form never appears. In Test the kit
reads the browser's Host header instead, and only when it names loopback or one
of the store's hostnames; an unknown Host falls back to the primary hostname.
A full URL or `//host` as `path` throws.

```ts
import type { StoreKit } from "@relyon/store-kit";
declare const kit: StoreKit; // the app's kit (relyon-setup)

// A route that takes a Web Request, such as a Next.js route handler (app/cart/route.ts).
export async function POST(request: Request): Promise<Response> {
  const form = await request.formData();
  const product = String(form.get("product") ?? "");
  const quantity = Math.min(99, Math.max(1, Math.trunc(Number(form.get("quantity")) || 1)));
  const query = new URLSearchParams({ product, quantity: String(quantity) });
  return kit.redirect(request, `/checkout?${query}`);
}
```

With Express, pass `webRequest(req)` (see `express-server.md`) and send the
answer's status and `Location`.

The page's own script (served by the app as `/checkout.js`):

```js checkout.js
// Browser code. It never reads, stores or sends card numbers.
const form = document.getElementById("pay");
const page = form ?? document.getElementById("checkout-status");
const status = document.getElementById("status");
const button = form?.querySelector("button[type=submit]");
const fields = form?.querySelector("relyon-card-fields");
const reference = page.dataset.orderReference;
const statusUrl = `/checkout/status?order_reference=${encodeURIComponent(reference)}`;
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
const orderPending = "Your payment went through. Your order is still being placed";
// Payment success is evidence, not a polling status: once known, never forget it.
let paymentSucceeded = page.dataset.paymentSucceeded === "true";
let attemptSent = !form;
// Set once the server says this checkout definitely charged nothing.
let newCheckoutHref = null;
function remember(result) {
  if (result.payment_succeeded === true || result.state === "paid") paymentSucceeded = true;
}

/** One call to the store's server. No answer, a timeout or an unreadable one is "unknown". */
async function call(path, body) {
  try {
    const response = await fetch(path, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
      signal: AbortSignal.timeout(30_000),
    });
    const result = await response.json();
    return typeof result?.state === "string" ? result : { state: "unknown" };
  } catch {
    return { state: "unknown" };
  }
}

function paid(result) {
  window.location.assign(`/orders/thanks?order=${encodeURIComponent(result.order ?? "")}`);
}

/**
 * A definite non-charge: `closed`, or `not_found` answering this page's own
 * Pay request (the server had no checkout, so it sent nothing). This checkout
 * is over, so Pay is never left stuck. It becomes "Try again", which opens a
 * new checkout (new order reference) instead of paying here.
 */
function offerNewCheckout(message) {
  status.textContent = message;
  newCheckoutHref = form?.dataset.newCheckout || "/checkout?new=1";
  if (button) {
    button.textContent = "Try again";
    button.disabled = false;
  }
}

/** Keep known success even when a later answer is lost or unreadable. */
function showPending(message) {
  if (button) button.disabled = true;
  const link = document.createElement("a");
  link.href = statusUrl;
  link.textContent = "Check your order status";
  status.replaceChildren(`${paymentSucceeded ? orderPending : message} `, link);
}

/**
 * After a payment was sent (or may have been): ask the server until the
 * outcome is final. The server only replays the attempt it already has, with
 * the same Idempotency-Key, so this never charges again.
 */
async function confirmOutcome() {
  for (let attempt = 0; attempt < 30; attempt += 1) {
    const result = await call("/checkout/confirm", { orderReference: reference });
    remember(result);
    if (result.state === "paid" && result.order) return paid(result);
    if (!paymentSucceeded && result.state === "open") {
      // The server never received the payment: nothing was sent, so Pay is safe again.
      status.textContent = result.message;
      attemptSent = false;
      if (button) button.disabled = false;
      return;
    }
    if (!paymentSucceeded && result.state === "closed") {
      return offerNewCheckout(result.message); // nothing was charged
    }
    if (result.state === "not_found") {
      // A payment may already have been sent: a missing record is not proof
      // that nothing was charged. Never offer a new checkout from here.
      return showPending("We can't find this payment to confirm it. Don't pay again: check your order status or contact the store.");
    }
    showPending(result.state === "pending" ? result.message : "We can't confirm your payment yet. Don't pay again.");
    await sleep(Math.min(30_000, 2_000 * 2 ** Math.min(attempt, 4)));
  }
  // Still not final: the server keeps checking. Pay stays disabled.
  showPending("We're still confirming your payment. Don't pay again.");
}

if (form) {
  fields.addEventListener("relyon-ready", () => { if (!attemptSent) button.disabled = false; });
  fields.addEventListener("relyon-error", (event) => { if (!attemptSent) status.textContent = event.detail.message; });
  // The card fields script loads async, so the fields can be ready before
  // this script runs. The page's ready latch (above fields.html) recorded it.
  button.disabled = fields.dataset.relyonReady !== "true";

  form.addEventListener("submit", async (event) => {
    event.preventDefault();
    if (newCheckoutHref) return window.location.assign(newCheckoutHref); // "Try again": a new checkout
    if (attemptSent) return; // late ready events or duplicate submissions cannot start a new attempt
    attemptSent = true;
    button.disabled = true;
    let paymentMethod;
    try {
      // Resolves to the payment_method object; the card data stays in the fields.
      paymentMethod = await fields.tokenize();
    } catch (error) {
      status.textContent = error.message; // incomplete card details; nothing was sent
      attemptSent = false;
      button.disabled = false;
      return;
    }
    status.textContent = "Paying…";
    // Only the order reference and the token. Never totals: the server re-prices.
    const result = await call("/checkout/pay", { orderReference: reference, paymentMethod });
    remember(result);
    if (result.state === "paid" && result.order) return paid(result);
    if (!paymentSucceeded && result.state === "open") {
      status.textContent = result.message; // the server sent nothing
      attemptSent = false;
      button.disabled = false;
      return;
    }
    if (!paymentSucceeded && (result.state === "closed" || result.state === "not_found")) {
      return offerNewCheckout(result.message); // declined or refused: nothing was charged
    }
    // "pending" or no usable answer: the payment may have gone through.
    showPending(result.state === "pending" ? result.message : "We can't confirm your payment yet. Don't pay again.");
    await confirmOutcome();
  });
} else if (page.dataset.state === "pending") {
  // Render the saved evidence before the first poll, including after a reload.
  if (paymentSucceeded) showPending(orderPending);
  void confirmOutcome();
}
```

- `relyon-ready` fires when the fields can take a card; `relyon-error`
  carries a message to show the shopper.
- Pay is enabled by whichever comes first: the `relyon-ready` listener, or
  the ready latch's `data-relyon-ready` mark when the fields were ready before
  `/checkout.js` ran. Without the latch, Pay can stay disabled forever.
- `tokenize()` rejects when the card details are incomplete. Nothing was sent,
  so the page lets the shopper fix them and press Pay again.
- Once `/checkout/pay` was called, Pay is enabled again only when the server
  says the checkout is still `open` (nothing was sent). After a definite
  non-charge (`closed`: declined or refused; or `not_found` as the answer to
  the Pay request itself, so nothing was sent) the button is never left
  stuck: it becomes **Try again**, which opens a new checkout
  (`data-new-checkout` on the form, such as `/checkout?…&new=1`) with a new
  order reference. It never pays again under the closed reference, and it is
  never offered while a payment may have gone through. A lost connection, a
  timeout, a `5xx` or a `pending` answer means the payment may have gone
  through: the page says so, links to the order status page and keeps asking
  `/checkout/confirm`. It never says "didn't go through" unless the server
  answered `closed`. A `not_found` from `/checkout/confirm` (after a payment
  may have been sent, for example when the server lost its record) is not
  proof of a non-charge: Pay stays disabled with the status link and no
  **Try again**.
- The page never reloads itself. It polls `/checkout/confirm` with `fetch`
  and stops on a final answer; calling `location.reload()` from a status
  answer (or a status page that sends the browser back to itself) is how a page
  reload loop starts.
- A reload while a payment is being confirmed shows the order status page
  (the server remembers the shopper's order reference), not new card fields,
  so it can't start a second payment.
- For `payment_succeeded: true`, show exactly "Your payment went through. Your
  order is still being placed" immediately, while polling and after polling
  stops. Keep the status link and Pay disabled. The status page's
  `data-payment-succeeded` restores that evidence on reload. Lost or unreadable
  answers never erase it; genuinely unknown outcomes keep their cautious wording.
- Ignore late card-field events once an attempt starts, and only navigate to
  the thank-you page when the server returns a saved store order.
- The thank-you page is the app's own. Showing it is not proof of payment;
  the server's `createPayment` result and `createOrder` are.
- In Test without a processor sandbox account, the fields show a Test notice
  and `tokenize()` returns a mock payment method the Test processor accepts.
  Never type real card numbers in Test.
