# Payment troubleshooting by error code

Every payment failure in the store's checkout carries one stable code. The same
code appears wherever the failure surfaces:

- the card fields: `err.code` when `tokenize()` rejects, and `event.detail.code`
  on a `relyon-error` event;
- the kit: `PaymentMethodShapeError.code`, the `code` in the kit routes' JSON
  answers (`order_refusal.code`, or `error.code` for the payments-only routes)
  and the kit's log lines;
- Relyon's API: the `code` of a field error in a `400 invalid_request` answer
  (`errors[].code`), while the top-level `code` stays `invalid_request`.

The kit exports the list as `PAYMENT_ERROR_CODES` (with a shopper message for
each in `PAYMENT_ERRORS`), and `paymentErrorCode(error)` reads the code from
any of the errors above. Look up the code below. Each section says what you
see, the cause, how to confirm it, the fix, and a prompt you can paste into
your coding agent.

Messages from the card fields are written for the shopper: show them as they
are. `err.details` lists the processor's own error codes for your logs and
never holds card data. Never log the payment method itself.

The store kit doctor's `checkout` item (`npx @relyon/store-kit doctor`) uses
the same reference. When it isn't PASS, its line ends with "Troubleshooting
code: CODE", and the Relyon dashboard shows the same code for the run the kit
reported. The doctor's own codes (`checkout_page_unavailable` through
`card_fields_session_failed`) follow the payment codes below; it also reports
`session_expired`, which has one section for both.

## card_field_required

**What you see:** `tokenize()` rejects with "Enter the card number." (or the
expiry date, the CVV, or several of them in one sentence), "The CVV isn't
valid. Check it and try again." or "Check the card details and try again.";
`err.code` is `card_field_required`.

**Cause:** a card field is empty or incomplete, the CVV was refused, or the
processor gave a reason the card fields don't classify. Nothing was sent.

**How to confirm:** log `err.code` and `err.details` in the Pay handler's
`catch`. `details` names the field (`fieldType`) and the processor's reason.

**Fix:** show `err.message` next to the fields and enable Pay again. The fields
stay usable; the shopper fixes the card and presses Pay again. Don't send
anything to the server for this attempt.

**Prompt for your coding agent:**

```text
In the checkout page script, the Pay handler calls fields.tokenize(). When it rejects with err.code "card_field_required", show err.message to the shopper, log err.code and err.details (never the card or payment method), re-enable the Pay button and don't call the server. Keep the card fields on the page so the shopper can fix the card and press Pay again.
```

## card_number_invalid

**What you see:** `tokenize()` rejects with "The card number isn't valid.
Check it and try again."; `err.code` is `card_number_invalid`.

**Cause:** the processor's card fields refused the number (wrong length, a
failed check digit or an unsupported format). Nothing was sent.

**How to confirm:** `err.details` has an entry whose `fieldType` is the card
number field.

**Fix:** show `err.message` and enable Pay again so the shopper can correct
the number. Never read or validate the number in your own code: it stays
inside the card fields.

**Prompt for your coding agent:**

```text
When fields.tokenize() rejects with err.code "card_number_invalid", show err.message under the card fields, re-enable Pay and send nothing to the server. Don't add any card-number validation of our own; the number never leaves the card fields.
```

## expiry_invalid

**What you see:** `tokenize()` rejects with "The expiry date isn't valid or
has passed. Check it and try again."; `err.code` is `expiry_invalid`.

**Cause:** the expiry date is malformed or in the past. Nothing was sent.

**How to confirm:** `err.details` has an entry whose `fieldType` is the
expiration date field.

**Fix:** show `err.message` and enable Pay again.

**Prompt for your coding agent:**

```text
When fields.tokenize() rejects with err.code "expiry_invalid", show err.message to the shopper, re-enable Pay and send nothing to the server, so the shopper can correct the expiry date and press Pay again.
```

## session_expired

**What you see:** "Payment session expired, refresh the page." from
`tokenize()`, or a `relyon-error` event whose `detail.code` is
`session_expired` (the element's `client-token` is missing or unreadable).
The doctor's `checkout` item reports it as FAIL when the served
`<relyon-card-fields>` has a `client-token` the card fields can't read, or its
no-charge self-test got back a session that isn't usable.

**Cause:** the payment session behind the card fields expired or was never
loaded: the page sat open too long, the `client-token` attribute is empty or
wrong, or the processor no longer accepts the session's key. Nothing was sent.

**How to confirm:** check that the page's `<relyon-card-fields>` has the
`client-token` from a fresh `kit.cardFields` (or `payments.cardFields`) call
for this checkout, and how long ago that call ran.

**Fix:** reload checkout so the server creates a new payment session and
renders fresh card fields. Never reuse an old `client-token`.

**Prompt for your coding agent:**

```text
Our checkout shows "Payment session expired, refresh the page." (code session_expired). Make sure every checkout page render creates a fresh payment session with kit.cardFields (or payments.cardFields) and puts its client-token on <relyon-card-fields>. When the code is session_expired, show the message and offer a reload of the checkout; never reuse a stored client-token.
```

## fields_timeout

**What you see:** "Card fields took too long, try again." from `tokenize()`,
or a `relyon-error` event whose `detail.code` is `fields_timeout` with "The
card fields could not be loaded." (or the script's own load message).

**Cause:** the processor's card fields didn't load or didn't answer in time:
a slow or blocked network, a content security policy that blocks the card
fields script or its frames, or an ad blocker. Nothing was sent.

**How to confirm:** open the browser console and network panel on the
checkout page. Look for blocked requests to the card fields script or the
processor's frames, and for a content security policy error.

**Fix:** allow the card fields script (`script_url` / `cardFieldsScriptUrl`)
and the processor's frames in the page's content security policy, then let
the shopper press Pay again.

**Prompt for your coding agent:**

```text
The checkout card fields fail with code fields_timeout. Check the checkout page's Content-Security-Policy and any script loading: the Relyon card fields script and the payment processor's iframes must be allowed (script-src and frame-src). Keep the "Card fields took too long, try again." message for the shopper and re-enable Pay after the error.
```

## wallet_payload_wrapped

**What you see:** a Google Pay payment refused before any charge. Relyon
answers `400 invalid_request` with a field error on `payment_method` whose
code is `wallet_payload_wrapped` and whose message is exactly
"payment_method must be event.detail.paymentMethod from the relyon-payment
event, not the whole event detail." A kit that unwraps the payload never
sends it; an older kit or the app's own API call does.

**Cause:** the page sent the whole `relyon-payment` event detail
(`{ paymentMethod, wallet: "google_pay" }`) as the payment method instead of
`event.detail.paymentMethod`.

**How to confirm:** in the page's `relyon-payment` listener, check what it
passes on. It must be `event.detail.paymentMethod`, an object with
`type: "wallet_token"`.

**Fix:** submit `event.detail.paymentMethod` straight away (the wallet data
expires about 60 seconds after the sheet closes). With a bundler, use the
kit's browser helper, imported from the package the store installed:

```text
import { attachRelyonWallet } from "@relyon/kit/browser";
attachRelyonWallet(fields, (paymentMethod) => pay(paymentMethod));
```

A store installed from `@relyon/store-kit` imports the same helper from
`@relyon/store-kit/browser` instead:

```text
import { attachRelyonWallet } from "@relyon/store-kit/browser";
```

Without a bundler, the `relyon-payment` listener in
`references/checkout-page.md` does the same.

**Prompt for your coding agent:**

```text
Google Pay payments fail with code wallet_payload_wrapped. In the checkout page script, the "relyon-payment" listener on <relyon-card-fields> must send event.detail.paymentMethod (an object with type "wallet_token") to our pay route, not event.detail. Submit it immediately, once per checkout, because the wallet data expires about 60 seconds after the Google Pay sheet closes. Use attachRelyonWallet from @relyon/kit/browser (or @relyon/store-kit/browser if the store installed @relyon/store-kit) if the page is bundled.
```

## payment_method_missing_type

**What you see:** the kit throws `PaymentMethodShapeError` with code
`payment_method_missing_type` before sending anything, the kit route answers
with that code, or Relyon answers `400 invalid_request` with a field error on
`payment_method` with that code (beside `required_field_missing` on
`payment_method.type`).

**Cause:** the payment method sent to the server isn't what the card fields
produced: an empty object, a value rebuilt by the app, a JSON string that was
never parsed, or a field picked out of it.

**How to confirm:** log the keys (never the values) of the payment method the
server receives. It must have `type` (`hosted_card` from `tokenize()`, or
`wallet_token` from the `relyon-payment` event).

**Fix:** send exactly what `tokenize()` resolved to, unchanged, as JSON in the
Pay request body, and pass it to the kit as it arrived.

**Prompt for your coding agent:**

```text
Payments fail with code payment_method_missing_type. Trace the payment method from the checkout page to the server: the browser must send exactly what fields.tokenize() resolved to (or event.detail.paymentMethod for Google Pay) as a JSON object in the Pay request, and the server must pass that object unchanged to kit.createPayment / payments.charge. Don't rebuild, rename, stringify or pick fields out of it. Log only its keys, never its values.
```

## processor_declined

**What you see:** the payment was sent and the processor declined it. The
payment's `status` is `failed`; the kit route answers `402` with
`code: "processor_declined"` and Relyon's own `failure_code` (such as
`card_declined`) alongside.

**Cause:** the card issuer or the processor refused the charge (insufficient
funds, a suspected fraud check, a wrong CVV or postal code, and so on).
Nothing was charged for this attempt.

**How to confirm:** look up the payment in the Relyon dashboard or with
`GET /v1/payments/{id}`; its `failure_code` and `failure_message` say why.

**Fix:** tell the shopper the card was declined and offer another card. The
next attempt uses a new attempt number (or a new checkout), never a replay of
the declined one.

**Prompt for your coding agent:**

```text
When our pay route answers code "processor_declined", show the shopper that the card was declined and let them try another card as a new attempt (attempt + 1, or a new checkout), never a replay of the declined request. Log failure_code and the payment id for support; never log card data or the payment method.
```

## checkout_page_unavailable

**What you see:** the doctor's `checkout` item is UNVERIFIED with this code. The checkout page
item above it is not PASS.

**Cause:** the doctor couldn't read the store's checkout page as a checkout page (an
error status, a page on another origin, or a page that isn't HTML), so
it couldn't look for the card fields.

**How to confirm:** read the doctor's checkout page line; it names the path it fetched and what
came back. Open that path on the store's public URL.

**Fix:** fix what the checkout page line reports, then run the doctor again. This
item can't pass until the checkout page loads.

**Prompt for your coding agent:**

```text
The Relyon store-kit doctor reports checkout_page_unavailable. Read its checkout page line, make the store's checkout page load on the public URL as an HTML page with status 200 on the store's own origin, then run npx @relyon/store-kit doctor again.
```

## checkout_cart_empty

**What you see:** the doctor's `checkout` item is UNVERIFIED with this code: it only saw the
checkout page with an empty cart.

**Cause:** no Test cart transfer landed a filled cart on the checkout page (the doctor
ran without a Test key, or the transfer failed), and the plain checkout page
had nothing in the cart. Card fields are only expected once the cart has
items.

**How to confirm:** check the doctor's cart transfer line, and whether `RELYON_API_KEY` held a
Test key when the doctor ran.

**Fix:** run the doctor in Test with the store's Test key so it can transfer an item
into the cart, or fix what the cart transfer line reports.

**Prompt for your coding agent:**

```text
The Relyon store-kit doctor reports checkout_cart_empty for the checkout item. Make sure RELYON_API_KEY holds the store's Test key where the doctor runs, fix anything its cart transfer line reports, then run npx @relyon/store-kit doctor again so it checks the checkout page with an item in the cart.
```

## card_fields_missing

**What you see:** the doctor's `checkout` item is FAIL with this code: the checkout page has
items in the cart but no `<relyon-card-fields>` element.

**Cause:** the checkout page doesn't render the card fields from `kit.cardFields(...)`
(or `payments.cardFields`), so shoppers have nowhere to enter a card.

**How to confirm:** open the checkout page with an item in the cart and view the page source.
Search for `<relyon-card-fields`.

**Fix:** render `kit.cardFields(...).html` (or `payments.cardFields`) into the
checkout page on the server, exactly once, next to the Pay button.

**Prompt for your coding agent:**

```text
The Relyon store-kit doctor reports card_fields_missing: our checkout page has no <relyon-card-fields> element. On every checkout page render, call kit.cardFields (or payments.cardFields) on the server and put its html into the page exactly once, then run npx @relyon/store-kit doctor again.
```

## card_fields_mounted_more_than_once

**What you see:** the doctor's `checkout` item is FAIL with this code and says how many times
the page mounts `<relyon-card-fields>`.

**Cause:** the checkout page renders the card fields more than once (a layout and a
page component both render them, or a component renders once per cart line).
Each mount loads its own card fields, and the Pay button can only tokenize
one of them.

**How to confirm:** view the checkout page's source with an item in the cart and count the
`<relyon-card-fields` elements.

**Fix:** mount the card fields once per checkout page, from one `kit.cardFields` (or
`payments.cardFields`) call.

**Prompt for your coding agent:**

```text
The Relyon store-kit doctor reports card_fields_mounted_more_than_once. Find every place our checkout renders kit.cardFields (or payments.cardFields) html or a <relyon-card-fields> element, keep exactly one mount per checkout page, then run npx @relyon/store-kit doctor again.
```

## card_fields_unverified

**What you see:** the doctor's `checkout` item is UNVERIFIED with this code: the served page
has no `<relyon-card-fields>`, and its scripts either create one in the
browser or couldn't all be read.

**Cause:** the doctor doesn't run a browser. A page that mounts the card fields from
client-side code (or loads scripts the doctor can't read, such as another
origin's bundle) can't be counted from the served HTML.

**How to confirm:** open the checkout page with an item in the cart in a browser and check that
one `<relyon-card-fields>` element appears.

**Fix:** render the card fields' html on the server, so the served page carries the
element. If the page must mount them in the browser, check the mount by hand
and record this item as open.

**Prompt for your coding agent:**

```text
The Relyon store-kit doctor reports card_fields_unverified for the checkout item. Render the html from kit.cardFields (or payments.cardFields) into the checkout page on the server so the served HTML contains exactly one <relyon-card-fields> element, then run npx @relyon/store-kit doctor again.
```

## wallet_listener_missing

**What you see:** the doctor's `checkout` item is FAIL with this code: Google Pay is on for
the store, and none of the checkout page's scripts listens for `relyon-
payment` or calls `attachRelyonWallet`.

**Cause:** the card fields show the Google Pay button and dispatch a `relyon-payment`
event when the shopper completes the sheet, but nothing on the page handles
it, so the payment never reaches the store's pay route.

**How to confirm:** search the checkout page's scripts for `relyon-payment` and
`attachRelyonWallet`.

**Fix:** add a `relyon-payment` listener on the card fields element that sends
`event.detail.paymentMethod` to the store's pay route, or call the kit's
`attachRelyonWallet` helper on the page.

**Prompt for your coding agent:**

```text
The Relyon store-kit doctor reports wallet_listener_missing: Google Pay is on but our checkout page never handles the relyon-payment event. In the checkout page's script, listen for relyon-payment on the <relyon-card-fields> element (or call attachRelyonWallet) and send event.detail.paymentMethod unchanged to our pay route, then run npx @relyon/store-kit doctor again.
```

## wallet_listener_unverified

**What you see:** the doctor's `checkout` item is UNVERIFIED with this code: Google Pay is on,
and either some of the checkout page's scripts couldn't be read, or they mention
`relyon-payment` without a call the doctor recognizes as registering a listener.

**Cause:** the doctor reads the page's inline scripts and same-origin scripts (and the
same-origin modules they import or re-export) within a bounded count and size.
A script on another origin, a package import, one that is too large, or one
past the bound can't be read, so a missing listener can't be told apart from
one in that script. It counts only `addEventListener("relyon-payment", …)` or a
call to `attachRelyonWallet(…)`; the event name held in a constant, or the
helper defined but never called, isn't proof of a listener.

**How to confirm:** complete a Google Pay sheet in Test on the checkout page and check that the
store's pay route receives the payment method.

**Fix:** keep the `relyon-payment` listener (or the `attachRelyonWallet` call) in an
inline script or a small same-origin script, or check it by hand and record
this item as open.

**Prompt for your coding agent:**

```text
The Relyon store-kit doctor reports wallet_listener_unverified for the checkout item. Make sure our checkout page calls attachRelyonWallet(...) on its relyon-card-fields element, or calls addEventListener("relyon-payment", ...) with the event name written literally, in an inline script or a same-origin script file the page loads directly, then run npx @relyon/store-kit doctor again.
```

## google_pay_state_unknown

**What you see:** the doctor's `checkout` item is UNVERIFIED with this code: it found no
`relyon-payment` listener, and it couldn't tell whether Google Pay is on.

**Cause:** the served card fields have no readable `client-token`, and the no-charge
self-test didn't run, so the doctor can't tell whether a listener is needed.

**How to confirm:** check the doctor's line for why the self-test didn't run (no Test key, Live,
or Relyon couldn't be asked).

**Fix:** run the doctor in Test with the store's Test key. If the page only adds the
`client-token` in the browser, render it on the server.

**Prompt for your coding agent:**

```text
The Relyon store-kit doctor reports google_pay_state_unknown for the checkout item. Make sure RELYON_API_KEY holds the store's Test key where the doctor runs and that the checkout page renders the client-token from kit.cardFields on the server, then run npx @relyon/store-kit doctor again.
```

## card_fields_session_failed

**What you see:** the doctor's `checkout` item is FAIL with this code. The line names Relyon's
refusal (status and code).

**Cause:** the no-charge self-test asked Relyon for a card-fields session the way the
checkout page does, and Relyon refused, so the page's card fields can't load
either. Common causes are a processor that isn't set up for the store, or a
key without payment access. Nothing was tokenized or charged.

**How to confirm:** call `kit.cardFields` from the store's server with its key and read the
error it throws.

**Fix:** resolve the refusal the line names (set up the processor, or use a key with
payment access), then run the doctor again.

**Prompt for your coding agent:**

```text
The Relyon store-kit doctor reports card_fields_session_failed with Relyon's refusal code. Find out why Relyon refuses to open a card-fields session for our store with our RELYON_API_KEY (processor setup, key scopes), fix it, then run npx @relyon/store-kit doctor again.
```
