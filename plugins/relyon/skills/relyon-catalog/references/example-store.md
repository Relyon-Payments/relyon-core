# Example store module

The app's own catalog, cart pricing and orders for a small store that sells candles. Every
price, stock level, coupon, shipping rate and tax rate lives here, in the app.
The kit calls `priceCart` and `createOrder`; the app's own pages call
`listProducts`, `getProduct` and `priceCart`. Replace the in-memory data with
the app's database.

These candle records are synthetic teaching data, not supplier or merchant
facts. Do not copy their brand, MPN, condition, stock or prices into a real
POD/dropship store. Replace them with verified supplier records first.

Amounts are integers in minor units (cents). Rounding is done once per value
with integer math, so the totals always add up exactly.

```js store.js
// The app's own store logic. Relyon never prices a cart; this module does.
import { StoreRefusal } from "@relyon/store-kit";

const PRIMARY_HOSTNAME = (process.env.STORE_HOSTNAMES ?? process.env.STORE_HOSTNAME ?? "shop.example.com")
  .split(",").map(host => host.trim()).filter(Boolean)[0];
if (!PRIMARY_HOSTNAME) throw new Error("Configure at least one store hostname");
const ORIGIN = process.env.STORE_PUBLIC_URL ?? `https://${PRIMARY_HOSTNAME}`;
if (new URL(ORIGIN).origin !== ORIGIN || new URL(ORIGIN).protocol !== "https:"
  || new URL(ORIGIN).hostname !== PRIMARY_HOSTNAME) {
  throw new Error("STORE_PUBLIC_URL must be the HTTPS origin of the primary store hostname");
}
const CURRENCY = "USD";
const PRODUCTS = [
  { id: "lavender-candle", title: "Lavender candle", description: "Hand-poured soy candle with lavender oil. 40-hour burn.", price: 2400, stock: 25 },
  { id: "cedar-candle", title: "Cedar candle", description: "Hand-poured soy candle with cedarwood oil. 40-hour burn.", price: 2600, stock: 0 },
  { id: "candle-trio", title: "Candle trio", description: "Three small candles: lavender, cedar and citrus.", price: 5300, stock: 10 },
];
const SHIPPING_OPTIONS = [
  { id: "ground", label: "Ground (3-5 days)", amount: 599 },
  { id: "pickup", label: "Pick up in store", amount: 0, fulfillment_type: "pickup" },
];
const COUPONS = { WELCOME10: { label: "10% off your first order", basisPoints: 1000 } };
const TAX_BASIS_POINTS = 700; // a flat example rate; real stores compute tax from the address
const orders = new Map(); // the app's orders table, unique on the Relyon payment ID

/** A product as the app's pages and Relyon's discovery index see it. */
function indexProduct(product) {
  return {
    merchant_product_id: product.id,
    title: product.title,
    description: product.description,
    url: `${ORIGIN}/products/${product.id}`,
    images: [`${ORIGIN}/images/${product.id}.jpg`],
    from_price: { amount: product.price, currency: CURRENCY }, // display only
    availability: product.stock > 0 ? "in_stock" : "out_of_stock",
  };
}

/** Full source-backed product; feed facts are separate from the display minimum. */
function sourceProduct(product) {
  return {
    ...indexProduct(product),
    // Synthetic fixture only: this example attests the store owns its candle brand.
    // Real stores must supply their own verified brand and SKU/MPN; never infer them.
    brand: "Example Candle Co",
    mpn: product.id,
    agent: { visible: true, purchasable: true },
    feed: { exactPrice: { amount: product.price, currency: CURRENCY }, condition: "new", ownBrand: true },
  };
}

/** The kit's paginated source, independent of syncCatalog and mirror persistence. */
export const catalogSource = {
  async listProducts({ cursor, limit, query }) {
    const offset = cursor === undefined ? 0 : Number(cursor);
    if (!Number.isSafeInteger(offset) || offset < 0) throw new Error("Invalid catalog cursor");
    if (!Number.isSafeInteger(limit) || limit < 1 || limit > 500) throw new Error("Invalid catalog limit");
    const words = (query ?? "").toLowerCase().split(/\s+/).filter(Boolean);
    const matches = PRODUCTS.filter(product =>
      words.every(word => `${product.title} ${product.description}`.toLowerCase().includes(word)));
    const end = offset + limit;
    return {
      products: matches.slice(offset, end).map(sourceProduct),
      nextCursor: end < matches.length ? String(end) : null,
    };
  },
  async getProduct(id) {
    const product = PRODUCTS.find(item => item.id === id);
    return product ? sourceProduct(product) : null;
  },
};

/**
 * Products for Relyon's index, which agent checkout buys from: the same facts
 * as the source, including the merchant-supplied brand and MPN that Google and
 * ChatGPT checkout require, without the kit-only feed block.
 */
export async function listProducts() {
  return PRODUCTS.map((product) => {
    const { feed, ...indexed } = sourceProduct(product);
    return indexed;
  });
}

/** One product for the app's product page: the same source-backed facts the feeds carry, so its JSON-LD has the same brand and MPN. */
export function getProduct(id) {
  const product = PRODUCTS.find((item) => item.id === id);
  return product ? sourceProduct(product) : null;
}

/**
 * Prices a cart: one cart line per input line, in the same order, and totals
 * that add up. Relyon charges exactly `totals.total`. Throw a StoreRefusal to
 * turn the cart away; agents get it in their protocol's error format.
 */
export async function priceCart(lines, address, coupons, context) {
  if (!Array.isArray(lines) || lines.length === 0) throw new StoreRefusal("empty_cart", "The cart is empty.");
  const wanted = new Map();
  const priced = lines.map((line, index) => {
    const product = PRODUCTS.find((item) => item.id === line.product_id);
    if (!product) throw new StoreRefusal("unknown_product", "That product isn't sold here.", 404);
    if (!Number.isSafeInteger(line.quantity) || line.quantity < 1 || line.quantity > 99) {
      throw new StoreRefusal("invalid_quantity", "Quantity must be between 1 and 99.");
    }
    if (line.options && Object.keys(line.options).length > 0) {
      throw new StoreRefusal("unsupported_options", `${product.title} has no options to choose.`);
    }
    wanted.set(product.id, (wanted.get(product.id) ?? 0) + line.quantity);
    if (wanted.get(product.id) > product.stock) {
      throw new StoreRefusal("out_of_stock", `${product.title} is out of stock.`, 409);
    }
    return {
      id: `line_${index + 1}`,
      merchant_product_id: product.id,
      title: product.title,
      quantity: line.quantity,
      unit_amount: product.price,
      amount: product.price * line.quantity,
    };
  });
  const subtotal = priced.reduce((sum, line) => sum + line.amount, 0);

  const discounts = [];
  for (const code of new Set(coupons ?? [])) {
    const coupon = COUPONS[String(code).toUpperCase()];
    if (!coupon) throw new StoreRefusal("invalid_coupon", `The code ${code} isn't valid.`);
    discounts.push({ code: String(code).toUpperCase(), label: coupon.label, amount: Math.floor((subtotal * coupon.basisPoints) / 10000) });
  }
  const discount = discounts.reduce((sum, item) => sum + item.amount, 0);
  if (discount > subtotal) throw new StoreRefusal("invalid_coupon", "Those codes can't be combined.");

  const optionId = context?.shippingOptionId ?? null;
  const option = optionId === null ? null : SHIPPING_OPTIONS.find((item) => item.id === optionId);
  if (optionId !== null && !option) throw new StoreRefusal("invalid_shipping_option", "Choose one of the offered shipping options.");
  const shipping = option ? option.amount : 0;
  const tax = Math.round(((subtotal - discount) * TAX_BASIS_POINTS) / 10000);
  const total = subtotal - discount + shipping + tax;

  return {
    currency: CURRENCY,
    lines: priced,
    discounts,
    shipping_options: SHIPPING_OPTIONS,
    ...(option ? { shipping_option_id: option.id } : {}),
    ...(address ? { shipping_address: address } : {}),
    totals: { currency: CURRENCY, subtotal, discount, shipping, tax, total },
  };
}

/**
 * Called by the kit exactly once for each succeeded payment. Keyed by the
 * payment ID as well, so the app's own records can never hold one payment twice.
 */
export async function createOrder(cart, payment) {
  const existing = orders.get(payment.id);
  if (existing) return existing;
  for (const line of cart.lines) {
    const product = PRODUCTS.find((item) => item.id === line.merchant_product_id);
    if (product) product.stock = Math.max(0, product.stock - line.quantity);
  }
  const order = {
    id: `order_${orders.size + 1}`,
    paymentId: payment.id,
    // Relyon's order ID (ord_…), for agent checkouts and the store's own checkout page, used to report fulfillment.
    relyonOrderId: typeof payment.order_id === "string" ? payment.order_id
      : payment.id.startsWith("ord_") ? payment.id : null,
    total: cart.totals.total,
    currency: cart.currency,
    lines: cart.lines.map(({ merchant_product_id, title, quantity, amount }) => ({ merchant_product_id, title, quantity, amount })),
    status: "paid",
  };
  orders.set(payment.id, order);
  return order;
}

export function findOrderByPayment(paymentId) {
  return orders.get(paymentId) ?? null;
}

export function findOrder(id) {
  return [...orders.values()].find((order) => order.id === id) ?? null;
}
```

A cart this module returns for two lavender candles, ground shipping and the
`WELCOME10` code:

```json
{
  "currency": "USD",
  "lines": [
    { "id": "line_1", "merchant_product_id": "lavender-candle", "title": "Lavender candle", "quantity": 2, "unit_amount": 2400, "amount": 4800 }
  ],
  "discounts": [{ "code": "WELCOME10", "label": "10% off your first order", "amount": 480 }],
  "shipping_options": [
    { "id": "ground", "label": "Ground (3-5 days)", "amount": 599 },
    { "id": "pickup", "label": "Pick up in store", "amount": 0, "fulfillment_type": "pickup" }
  ],
  "shipping_option_id": "ground",
  "totals": { "currency": "USD", "subtotal": 4800, "discount": 480, "shipping": 599, "tax": 302, "total": 5221 }
}
```
