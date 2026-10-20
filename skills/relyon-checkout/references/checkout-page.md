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
    if (!paymentSucceeded && (result.state === "closed" || result.state === "not_found")) {
      status.textContent = result.message; // a reload starts a new checkout
      return;
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
      status.textContent = result.message; // declined or refused: nothing was charged
      return;
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
  says the checkout is still `open` (nothing was sent). A lost connection, a
  timeout, a `5xx` or a `pending` answer means the payment may have gone
  through: the page says so, links to the order status page and keeps asking
  `/checkout/confirm`. It never says "didn't go through" unless the server
  answered `closed`.
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
