# Next.js App Router (Next.js 15 and 16)

The kit runs inside the app's own route handlers. These files build on
`create-next-app@latest` (Next.js 16, Turbopack, `cacheComponents`) and on
Next.js 15. `npx @relyon/store-kit init --template next` creates a complete
starter that already does all of this.

## No route segment config

Add **no** `export const runtime`, `export const dynamic` or
`export const revalidate` to the kit's route files. Route handlers run on
Node.js by default, and they read the request, so they are dynamic on both
versions. Next.js 16 with `cacheComponents` refuses that segment config and the
build fails.

## One shared kit

Create the kit once, on the server only. Next.js loads `.env` itself; the Test
key init saved privately is read by `loadTestConfig()`. Never cat or print
`.env`, and never paste the key into a command.

```ts lib/relyon.ts
import "server-only"; // optional: npm install server-only
import { createStoreKit, type StoreKit } from "@relyon/store-kit";
import { loadTestConfig } from "@relyon/store-kit/server-config";
import { priceCart, createOrder, catalogSource } from "./store";

// One instance per server process, created on first use (never at build time).
const shared = globalThis as typeof globalThis & { relyonKit?: Promise<StoreKit> };

export function getKit(): Promise<StoreKit> {
  return (shared.relyonKit ??= (async () => {
    const { apiKey } = await loadTestConfig({ project: process.cwd() });
    const hostnames = (process.env.STORE_HOSTNAMES ?? "localhost").split(",").map((h) => h.trim()).filter(Boolean);
    const kit = createStoreKit({
      relyon: { baseUrl: "https://core.getrelyon.com/api", apiKey, waitingForSandboxKey: true },
      store: { name: "Example Store", description: "What the store sells.", hostnames },
      priceCart,
      createOrder,
      catalog: { source: catalogSource },
      feeds: { formats: ["openai", "openaiGzip", "google", "microsoft"] },
      // robots.txt is served by default; robots: false opts out.
    });
    await kit.start();
    return kit;
  })());
}
```

## Route files

```ts lib/relyon-routes.ts
import { storeKitRouteHandlers } from "@relyon/store-kit/next";
import { getKit } from "./relyon";

type Method = "GET" | "POST" | "PUT" | "PATCH" | "DELETE" | "HEAD";
const route = (method: Method) => async (request: Request) => {
  const handler = storeKitRouteHandlers(await getKit())[method];
  return handler(request);
};

export const GET = route("GET"), POST = route("POST"), PUT = route("PUT"),
  PATCH = route("PATCH"), DELETE = route("DELETE"), HEAD = route("HEAD");
```

Each of these files contains only one line,
`export { GET, POST, PUT, PATCH, DELETE, HEAD } from "@/lib/relyon-routes";`:

| File | Kit path |
| --- | --- |
| `app/.well-known/ucp/route.ts` | UCP profile |
| `app/.well-known/ucp/[...path]/route.ts` | UCP 2026-04-08 profile (`/.well-known/ucp/2026-04-08`, needed for cart transfer) |
| `app/.well-known/relyon-verification/route.ts` | domain verification |
| `app/.well-known/agent-card.json/route.ts` | agent card |
| `app/ucp/[...path]/route.ts`, `app/acp/[...path]/route.ts` | agent checkout; UCP cart transfer (`POST /ucp/carts`) runs under `app/ucp/[...path]` |
| `app/mcp/route.ts` | store MCP |
| `app/llms.txt/route.ts`, `app/sitemap-products.xml/route.ts`, `app/sitemap.xml/route.ts` | discovery |
| `app/feeds/[file]/route.ts` | selected feeds |
| `app/robots.txt/route.ts` | robots (skip with `robots: false`) |
| `app/relyon/[...path]/route.ts` | webhooks and the Test checkout |

A Next.js `app/robots.ts` or `app/sitemap.ts` metadata file conflicts with the
kit's route of the same path: keep one. To keep the app's own robots policy,
pass `robots: { existing: () => itsText }` and delete `app/robots.ts`.

## Product pages (Next.js 16)

A product page builds on Next.js 16 (`cacheComponents`) when its dynamic
params come from `generateStaticParams`, the data read sits inside a
`Suspense` boundary, and the product JSON-LD is rendered on the server.
`productJsonLd` is a pure function: it needs no kit instance, so nothing talks
to Relyon at build time.

```tsx app/products/[id]/page.tsx
import { Suspense } from "react";
import { notFound } from "next/navigation";
import { productJsonLd } from "@relyon/store-kit";
import { catalogSource } from "@/lib/store";
import { AddToCart } from "./add-to-cart";

type Params = Promise<{ id: string }>;

export async function generateStaticParams() {
  const { products } = await catalogSource.listProducts({ limit: 100 });
  return products.map((product) => ({ id: product.merchant_product_id }));
}

export default function ProductPage({ params }: { params: Params }) {
  return (
    <Suspense fallback={<p>Loading…</p>}>
      <Product params={params} />
    </Suspense>
  );
}

async function Product({ params }: { params: Params }) {
  const { id } = await params;
  const product = await catalogSource.getProduct(id);
  if (!product) notFound();
  const jsonLd = JSON.stringify(productJsonLd(product)).replace(/</g, "\\u003c");
  return (
    <main>
      <h1>{product.title}</h1>
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: jsonLd }} />
      <AddToCart productId={product.merchant_product_id} />
    </main>
  );
}
```

```tsx app/products/[id]/add-to-cart.tsx
"use client";
// A client component: plain props only. Never import @relyon/store-kit here.
export function AddToCart({ productId }: { productId: string }) {
  return (
    <form method="post" action="/cart">
      <input type="hidden" name="product" value={productId} />
      <button type="submit">Add to cart</button>
    </form>
  );
}
```

Keep every `@relyon/store-kit` import (and `lib/relyon.ts`) out of client
components (`"use client"` files and anything they import): the kit is server
code that reads the store's keys. Pass plain values to client components as
props.

For a product with options, list its variants in `feed.variants`
(relyon-catalog skill) and post the chosen options with the cart form. The
checkout page shows a landed cart with `kit.landing()`, which also takes the
page's `searchParams` (see "Transferred carts and hand-offs" in the
relyon-checkout skill's `references/checkout-page.md`).

## Previews on 127.0.0.1

`next dev` refuses its development resources to an origin it doesn't know.
When the preview opens the app on `127.0.0.1` (doctor and test-checkout use
that address too), add it to `next.config.mjs`:

```js next.config.mjs
const nextConfig = { allowedDevOrigins: ["127.0.0.1"] };
export default nextConfig;
```

The `--template next` starter sets it.

## Imports between server files

With `"moduleResolution": "bundler"` (the create-next-app default), import
server files without an extension (`./store`). If the project's server code
uses NodeNext-style `.js` imports of `.ts` files, set `"module"` and
`"moduleResolution"` to `"NodeNext"` in `tsconfig.json` so Turbopack (Next 16)
resolves them, and keep `"skipLibCheck": true` (Next.js 15's own type files
don't resolve under NodeNext otherwise). For Next.js 15 (webpack) also add
`webpack(config) { config.resolve.extensionAlias = { ".js": [".ts", ".tsx", ".js"] }; return config; }`
and `turbopack: {}` to `next.config.mjs`. The `--template next` starter does
exactly this.

## Check it

Start the app (with `STORE_PUBLIC_URL` set to its address), then:

```sh
npx --no-install @relyon/store-kit doctor --url "$STORE_PUBLIC_URL"
npx --no-install @relyon/store-kit test-checkout --url "$STORE_PUBLIC_URL"
npx --no-install @relyon/store-kit doctor --url "$STORE_PUBLIC_URL"
```
