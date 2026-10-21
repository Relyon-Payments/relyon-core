# Checkout flow on the server: one attempt, never two charges

`checkout-flow.js` is the server half of the store's own checkout. The Express
server (`relyon-setup`, `references/express-server.md`) and the builder-platform
functions both use it. It keeps one record per order reference and guarantees:

- **A payment is recorded before it is sent.** One conditional write moves the
  checkout from `open` to `confirming` and stores the exact attempt (the
  re-priced cart and the payment method). Only the request that wins that write
  sends it.
- **Anything already sent is only ever replayed.** A retry, a second click, a
  status check or a reload replays the stored attempt with the same order
  reference, so `kit.createPayment` reuses the same `Idempotency-Key`. A new
  payment method from the browser is ignored once an attempt exists.
- **An unknown outcome stays unknown.** `PaymentOutcomeUnknownError`, a crash
  or any unexpected error leaves the checkout `confirming`. It is never shown
  as "didn't go through" and never closed until Relyon gives a final answer.
- **Paid but not saved is still pending.** When the payment succeeded but the
  app's `createOrder` threw, the checkout stays `confirming` (marked paid) and
  a replay charges nothing new and retries the order.
- **An order an earlier call already saved is found, not waited for.** When a
  replay's payment succeeded and the kit answers `orderStatus: "already_created"`
  (an earlier call's `createOrder` saved it, for example before a crash), the
  flow looks the order up by payment ID with the app's `findOrder` and finishes
  the checkout. `in_progress` (another request is still creating it) stays
  pending.
- **A reload never starts a second payment while one may be in flight.** A
  checkout that is `confirming` (or `paid`) is shown as its status, not as new
  card fields. Only an `open` checkout, where nothing was sent, is replaced.

States the browser sees (`state` in every JSON answer):

| `state` | HTTP | Meaning | The page |
| --- | --- | --- | --- |
| `open` | 200 | Nothing was sent for this checkout. | Pay may be enabled. |
| `pending` | 202 | A payment was sent; the outcome isn't final, or it's paid and the order is still being saved. | Pay stays disabled; keep checking the order status. |
| `paid` | 200 | Charged and the order saved. | Show the thank-you page. |
| `closed` | 409 | Declined or refused; nothing was charged. | A reload starts a new checkout. |
| `not_found` | 404 | No such checkout. | A reload starts a new checkout. |

The starter's `pending` answer also carries `payment_succeeded`: `true` means
the payment went through but the store order is not saved yet. This is the
store's own response field, not a Relyon API field. Keep that evidence in the
rendered status page so a reload or a lost polling answer cannot turn a known
successful payment into an uncertain one.

```js checkout-flow.js
// Server-only. The store's own checkout records, so a payment attempt is sent
// under one order reference and only ever replayed, never sent twice.
import {
  AgentRefusedError, PaymentOutcomeUnknownError, RelyonProblemError, StoreRefusal,
} from "@relyon/store-kit";

/**
 * Checkout records for one process (development, tests). In production use
 * the app's database (`postgresStoreCheckouts` in relyon-setup's
 * durable-state.js): `update` must be one conditional write there.
 */
export function memoryCheckouts() {
  const rows = new Map();
  return {
    async get(ref) { return rows.get(ref) ?? null; },
    async insert(ref, row) { rows.set(ref, row); },
    // Writes `row` only while the stored state is still `expected`.
    async update(ref, expected, row) {
      const current = rows.get(ref) ?? null;
      if (current?.state !== expected) return { updated: false, row: current };
      rows.set(ref, row);
      return { updated: true, row };
    },
    async confirming() { return [...rows].filter(([, row]) => row.state === "confirming").map(([ref]) => ref); },
  };
}

// The request is replayed for agent rules; credentials and cookies are not kept.
const PRIVATE_HEADER = /^cookie$|authorization$/;
const snapshot = (request) => request
  ? { url: request.url, method: request.method, headers: [...request.headers].filter(([name]) => !PRIVATE_HEADER.test(name)) }
  : null;
const rebuild = (saved) => saved ? new Request(saved.url, { method: saved.method, headers: saved.headers }) : undefined;

const MESSAGES = {
  open: "No payment was sent for this checkout. You can pay now.",
  pending: "We can't confirm your payment yet. Don't pay again: we're checking whether it went through.",
  paidUnsaved: "Your payment went through. Your order is still being placed",
  paid: "Your order is paid.",
  notFound: "We can't find this checkout. Reload to start again.",
};

/** What the browser is told about a checkout record. */
export function checkoutView(ref, row) {
  if (!row) return { httpStatus: 404, body: { state: "not_found", order_reference: ref ?? null, message: MESSAGES.notFound } };
  const body = { order_reference: ref };
  switch (row.state) {
    case "open": return { httpStatus: 200, body: { ...body, state: "open", message: MESSAGES.open } };
    case "paid": return { httpStatus: 200, body: { ...body, state: "paid", order: row.order ?? null, message: MESSAGES.paid } };
    case "closed": return { httpStatus: 409, body: { ...body, state: "closed", error: row.error, message: row.message } };
    default: return { httpStatus: 202, body: { ...body, state: "pending", payment_succeeded: row.paid === true, message: row.paid ? MESSAGES.paidUnsaved : MESSAGES.pending } };
  }
}

/**
 * `priceCart` is the app's own (the same function the kit calls for agents).
 * `checkouts` is `memoryCheckouts()` or a durable store with the same methods.
 * `findOrder(paymentId)` returns the app's saved order for a payment, or null
 * (`findOrderByPayment` in the example store; a lookup by the orders table's
 * unique payment ID in production).
 */
export function createCheckoutFlow(kit, { priceCart, checkouts, findOrder = async () => null }) {
  const price = (inputs) => priceCart(inputs.lines, inputs.address, inputs.coupons, {
    protocol: "web", operation: "landing", shippingOptionId: inputs.shipping_option_id, agentCheckoutId: null,
  });

  async function close(ref, expected, row, error, message) {
    const { attempt, ...rest } = row;
    const saved = await checkouts.update(ref, expected, { ...rest, state: "closed", error, message });
    return checkoutView(ref, saved.row);
  }

  /** Sends (or replays) the stored attempt. Same order reference, so the same Idempotency-Key. */
  async function send(ref, row, { first }) {
    const { attempt } = row;
    let result;
    try {
      result = await kit.createPayment({
        cart: attempt.cart, paymentMethod: attempt.paymentMethod, orderReference: ref,
        channel: row.channel ?? undefined, agentCheckoutId: row.agentCheckoutId ?? null,
        request: rebuild(attempt.request), info: attempt.info ?? undefined,
      });
    } catch (error) {
      // A first send that is refused never reached a charge.
      if (first && error instanceof AgentRefusedError) {
        return close(ref, "confirming", row, error.code, error.message);
      }
      if (first && error instanceof RelyonProblemError) {
        return close(ref, "confirming", row, error.code, "The payment couldn't be made and nothing was charged. Reload checkout to try again.");
      }
      // Anything else may have charged: keep the attempt and answer "pending".
      if (!(error instanceof PaymentOutcomeUnknownError)) {
        console.error(`payment outcome unknown for ${ref}:`, error instanceof Error ? error.message : String(error));
      }
      return checkoutView(ref, (await checkouts.get(ref)) ?? row);
    }
    if (result.payment.status !== "succeeded") {
      return close(ref, "confirming", row, "payment_failed", "The payment didn't go through and nothing was charged. Reload checkout to try again.");
    }
    let orderId = result.order?.id ?? null;
    if (!result.orderError && !orderId && result.orderStatus === "already_created") {
      // An earlier call saved the order: find it by the payment.
      orderId = (await findOrder(result.payment.id))?.id ?? null;
    }
    if (result.orderError || !orderId) {
      // Paid, but the order isn't saved yet (createOrder failed, or another
      // request is still creating it). Keep the attempt: a replay charges
      // nothing new and retries the order.
      const saved = await checkouts.update(ref, "confirming", { ...row, paid: true, paymentId: result.payment.id });
      return checkoutView(ref, saved.row);
    }
    const { attempt: sent, ...rest } = row;
    const saved = await checkouts.update(ref, "confirming", {
      ...rest, state: "paid", paid: true, paymentId: result.payment.id, order: orderId,
    });
    return checkoutView(ref, saved.row);
  }

  /** The current state; replays the attempt when its outcome isn't final. Never a new charge. */
  async function confirm(ref) {
    const row = typeof ref === "string" ? await checkouts.get(ref) : null;
    if (row?.state !== "confirming") return checkoutView(ref, row);
    return send(ref, row, { first: false });
  }

  return {
    price,

    /**
     * Opens a checkout for the shopper's cart. `previous` is the shopper's
     * last order reference (cookie or local storage). When it is still
     * being confirmed (or is paid and `startNew` isn't set), its status is
     * returned instead of new card fields.
     */
    async open({ inputs, cart, previous, startNew = false, channel, agentCheckoutId }) {
      if (typeof previous === "string" && previous) {
        let prior = await checkouts.get(previous);
        if (prior?.state === "confirming") return { orderReference: previous, view: await confirm(previous) };
        if (prior?.state === "paid" && !startNew) return { orderReference: previous, view: checkoutView(previous, prior) };
        if (prior?.state === "open") {
          // Nothing was sent for it; retire it unless a payment claims it first.
          const { attempt, ...rest } = prior;
          const retired = await checkouts.update(previous, "open", { ...rest, state: "closed", error: "replaced", message: "A newer checkout replaced this one." });
          prior = retired.row;
          if (prior?.state === "confirming") return { orderReference: previous, view: await confirm(previous) };
        }
      }
      const priced = cart ?? await price(inputs); // throws StoreRefusal
      const orderReference = `order_${crypto.randomUUID()}`;
      // One payment session for exactly the store's total; throws PaymentUnavailableError.
      const fields = await kit.cardFields({ amount: priced.totals.total, currency: priced.currency, orderReference });
      await checkouts.insert(orderReference, {
        state: "open", inputs, total: priced.totals.total, currency: priced.currency,
        channel: channel ?? null, agentCheckoutId: agentCheckoutId ?? null,
      });
      return { orderReference, cart: priced, fields };
    },

    /** The browser's Pay: only an order reference and the tokenized payment method. */
    async pay({ orderReference, paymentMethod, request, info }) {
      const row = typeof orderReference === "string" ? await checkouts.get(orderReference) : null;
      if (!row) return checkoutView(orderReference, null);
      if (row.state !== "open") return confirm(orderReference);
      if (!paymentMethod || typeof paymentMethod !== "object") {
        return { httpStatus: 400, body: { ...checkoutView(orderReference, row).body, error: "payment_method_required" } };
      }
      // Re-price on the server. Never take a total from the browser.
      let cart;
      try {
        cart = await price(row.inputs);
      } catch (error) {
        if (!(error instanceof StoreRefusal)) throw error;
        return close(orderReference, "open", row, error.code, error.message);
      }
      if (cart.totals.total !== row.total || cart.currency !== row.currency) {
        return close(orderReference, "open", row, "cart_changed", "The total changed. Reload checkout to see it.");
      }
      // Record the attempt before sending it; this one conditional write
      // decides which request sends it.
      const attempt = { cart, paymentMethod, request: snapshot(request), info: info ?? null, at: Date.now() };
      const claim = await checkouts.update(orderReference, "open", { ...row, state: "confirming", attempt });
      if (!claim.updated) return confirm(orderReference);
      return send(orderReference, claim.row, { first: true });
    },

    confirm,

    /** Read-only: what the order status page shows. */
    async status(ref) {
      return checkoutView(ref, typeof ref === "string" ? await checkouts.get(ref) : null);
    },

    /** Run every minute or so: replays attempts whose outcome still isn't final. */
    async reconcile({ minAgeMs = 60_000 } = {}) {
      for (const ref of await checkouts.confirming()) {
        const row = await checkouts.get(ref);
        if (row?.state === "confirming" && Date.now() - (row.attempt?.at ?? 0) >= minAgeMs) await confirm(ref);
      }
    },
  };
}
```

- Keep the checkout record (and its attempt) until the state is final. Delete
  the stored payment method once a checkout is `paid` or `closed`; the flow
  already drops it from those records.
- Run `reconcile()` on a timer (or a scheduled job) so a shopper who closed
  the tab still gets their order saved.
- Order references are random and unguessable; the status answer shows only
  the state and the store's order ID.
