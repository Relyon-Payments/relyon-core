# Durable state for production

The kit's defaults keep everything in one process's memory. In production, and
whenever the app runs more than one server process, give the kit shared,
durable stores. This example uses Postgres through any client with
`query(text, params)` returning `{ rows, rowCount }` (a `pg` `Pool` works).
Adapt the SQL to the app's own database; keep the guarantees:

- `claimOrder` is a single atomic statement. Exactly one caller at a time gets
  `true` for a payment ID, across every process, so `createOrder` runs once per
  payment. A claim holds for the kit's lease (`orderClaimLeaseMs`, default 2
  minutes): a claim that was never marked created may be taken over once its
  lease has passed, because the server that held it may have died while creating
  the order. A claim marked created never expires.
- `completeOrder` marks the claim created after `createOrder` returned.
- `releaseOrder` deletes the claim after `createOrder` threw, only while that
  attempt (`claimId`) still holds it, so a retry can create the order.
- `orderClaimStatus` answers `"created"`, `"creating"` (an unexpired claim) or
  `null`, so the kit can tell an order another request is creating from one that
  exists. Implement all four; without the last two, claims never expire and
  checkouts can stay order pending (see `store-kit-reference.md`, "Production
  stores").
- Agent events are kept until Relyon accepted them (`remove` is called only
  then), in the order they were appended, and duplicates by `id` are ignored.
  Never make `append` a no-op or cap the table to zero rows; reporting must
  stay on.
- The catalog store keeps the last good catalog the kit published.
- Payment records (`getStorePayment`/`putStorePayment`) and hand-off token
  claims (`claimHandoffToken`) are optional members of the same store; the
  `relyon-checkout` skill has a typed version with all of them and the two
  extra tables. Add them before running more than one server process.
- `postgresStoreCheckouts` holds the app's own checkouts for
  `checkout-flow.js` (`relyon-checkout`). Its `update` is one conditional
  statement, so only one request sends a payment attempt and a checkout is
  never closed while its payment may have gone through.

```js durable-state.js
// Postgres-backed stores for @relyon/store-kit. Run SCHEMA once (a migration).
export const SCHEMA = `
create table if not exists relyon_checkouts (
  protocol text not null,
  session_id text not null,
  agent_checkout_id text,
  record jsonb not null,
  updated_at bigint not null,
  primary key (protocol, session_id)
);
create index if not exists relyon_checkouts_agent on relyon_checkouts (agent_checkout_id);
create table if not exists relyon_order_claims (
  payment_id text primary key,
  claim_id text,
  created boolean not null default false,
  expires_at bigint -- the kit's clock (ms); null once created
);
create table if not exists relyon_agent_events (
  seq bigserial primary key,
  id text not null unique,
  event jsonb not null
);
create table if not exists relyon_catalog (
  singleton boolean primary key default true check (singleton),
  products jsonb not null
);
create table if not exists store_checkouts (
  order_reference text primary key,
  state text not null,
  record jsonb not null,
  updated_at timestamptz not null default now()
);
create index if not exists store_checkouts_confirming on store_checkouts (updated_at) where state = 'confirming';
`;

/** CheckoutStateStore: per-checkout memory and the exactly-once order claim. */
export function postgresCheckoutState(db) {
  return {
    async get(protocol, sessionId) {
      const { rows } = await db.query(
        "select record from relyon_checkouts where protocol = $1 and session_id = $2", [protocol, sessionId]);
      return rows[0]?.record ?? null;
    },
    async byAgentCheckoutId(agentCheckoutId) {
      const { rows } = await db.query(
        "select record from relyon_checkouts where agent_checkout_id = $1 order by updated_at desc limit 1", [agentCheckoutId]);
      return rows[0]?.record ?? null;
    },
    async put(record) {
      await db.query(
        `insert into relyon_checkouts (protocol, session_id, agent_checkout_id, record, updated_at)
         values ($1, $2, $3, $4, $5)
         on conflict (protocol, session_id) do update
           set agent_checkout_id = excluded.agent_checkout_id, record = excluded.record, updated_at = excluded.updated_at`,
        [record.protocol, record.sessionId, record.agentCheckoutId, JSON.stringify(record), record.updatedAt]);
    },
    // One statement: a new claim, or a takeover of an unfinished claim whose lease passed.
    async claimOrder(paymentId, { claimId = null, now = Date.now(), leaseMs = 120_000 } = {}) {
      const { rowCount } = await db.query(
        `insert into relyon_order_claims (payment_id, claim_id, expires_at) values ($1, $2, $3)
         on conflict (payment_id) do update set claim_id = excluded.claim_id, expires_at = excluded.expires_at
           where not relyon_order_claims.created and relyon_order_claims.expires_at <= $4`,
        [paymentId, claimId, now + leaseMs, now]);
      return rowCount === 1;
    },
    async completeOrder(paymentId) {
      await db.query(
        `insert into relyon_order_claims (payment_id, created, expires_at) values ($1, true, null)
         on conflict (payment_id) do update set created = true, expires_at = null`, [paymentId]);
    },
    async releaseOrder(paymentId, claimId) {
      await db.query(
        `delete from relyon_order_claims
         where payment_id = $1 and not created and ($2::text is null or claim_id is null or claim_id = $2)`,
        [paymentId, claimId ?? null]);
    },
    async orderClaimStatus(paymentId, now) {
      const { rows } = await db.query(
        "select created, expires_at > $2 as holding from relyon_order_claims where payment_id = $1", [paymentId, now]);
      if (!rows[0]) return null;
      if (rows[0].created) return "created";
      return rows[0].holding ? "creating" : null;
    },
  };
}

/** AgentEventStore: queued agent events survive restarts until Relyon accepts them. */
export function postgresAgentEvents(db) {
  return {
    async append(event) {
      await db.query(
        "insert into relyon_agent_events (id, event) values ($1, $2) on conflict (id) do nothing",
        [event.id, JSON.stringify(event)]);
    },
    async peek(limit) {
      const { rows } = await db.query("select event from relyon_agent_events order by seq limit $1", [limit]);
      return rows.map((row) => row.event);
    },
    async remove(ids) {
      if (ids.length > 0) await db.query("delete from relyon_agent_events where id = any($1)", [ids]);
    },
    async size() {
      const { rows } = await db.query("select count(*)::int as n from relyon_agent_events");
      return rows[0].n;
    },
  };
}

/** CatalogPersistence: the last good catalog, served after a restart. */
export function postgresCatalog(db) {
  return {
    async load() {
      const { rows } = await db.query("select products from relyon_catalog");
      return rows[0]?.products ?? null;
    },
    async save(products) {
      await db.query(
        `insert into relyon_catalog (singleton, products) values (true, $1)
         on conflict (singleton) do update set products = excluded.products`,
        [JSON.stringify(products)]);
    },
  };
}

/** The app's own checkout records for checkout-flow.js (relyon-checkout). */
export function postgresStoreCheckouts(db) {
  const get = async (ref) => {
    const { rows } = await db.query("select record from store_checkouts where order_reference = $1", [ref]);
    return rows[0]?.record ?? null;
  };
  return {
    get,
    async insert(ref, record) {
      await db.query("insert into store_checkouts (order_reference, state, record) values ($1, $2, $3)",
        [ref, record.state, JSON.stringify(record)]);
    },
    // One conditional statement: only one request moves a checkout out of a state.
    async update(ref, expected, record) {
      const { rows } = await db.query(
        `update store_checkouts set state = $3, record = $4, updated_at = now()
         where order_reference = $1 and state = $2 returning record`,
        [ref, expected, record.state, JSON.stringify(record)]);
      return rows[0] ? { updated: true, row: rows[0].record } : { updated: false, row: await get(ref) };
    },
    async confirming() {
      const { rows } = await db.query(
        "select order_reference from store_checkouts where state = 'confirming' order by updated_at limit 100");
      return rows.map((row) => row.order_reference);
    },
  };
}
```

Wire them in when creating the kit (see `SKILL.md`):

```js
import pg from "pg";
import { postgresAgentEvents, postgresCatalog, postgresCheckoutState, postgresStoreCheckouts } from "./durable-state.js";
import { createRelyonKit } from "./relyon.js";

const db = new pg.Pool({ connectionString: process.env.DATABASE_URL });
export const kit = createRelyonKit({
  state: postgresCheckoutState(db),
  events: postgresAgentEvents(db),
  catalog: postgresCatalog(db),
});
// The app's own checkouts (server.js): createApp(kit, { checkouts: postgresStoreCheckouts(db) }).
export const checkouts = postgresStoreCheckouts(db);
```

The app's own orders table should also be unique on the Relyon payment ID, so
even a bug elsewhere can't record one payment twice (see `relyon-orders`).
