<!-- banner --><a href="https://github.com/kenmwara"><img src="https://raw.githubusercontent.com/kenmwara/kenmwara/main/assets/brand/card-ops.png?v=37d0891735" width="100%" alt="Unified Ops Dashboard: One dashboard every product reports into: an append-only event store on Cloudflare D1, Workers and Pages. LIVE behind Cloudflare Access."></a>

# Unified Ops Dashboard

> One dashboard for every product: sales, sign-ups, replies and errors from each app land in one place, so it's easy to see what's working each week.

🌐 **Live:** ops.tbot.trade *(behind Cloudflare Access — operator only; see screenshots below)*
🏗️ **Stack:** Cloudflare D1 + Workers + Pages + Email Routing + Email Sending
🔒 **Source:** private — this README + screenshots are the showcase

---

<img src="images/flow.png" width="100%" alt="How it works, in five steps. 1. Products send updates: Sales, sign-ups, replies and errors from every product. 2. Each update is checked: It must carry the right product's key, and payments must be signed. 3. Stored once, never edited: One database; duplicates are dropped. 4. Summaries are built: Totals, a timeline of incidents, and questions you can ask in plain English. 5. The owner sees it all: A private dashboard behind a login wall; alerts to a phone. Safeguards: Who can write: Only a product with its own key can send updates; Trust: Nothing is changed after the fact; Who can read: Every page needs a sign-in.">

## At a glance

<img src="images/event-flow.png" width="100%" alt="Flow diagram of one update, from a product to the screen. A product sends an update: a sale, a sign-up, a reply or an error. It goes to the intake program, carrying that product's own secret key. Decision: does the key match a known product? No: turned away, nothing saved; the intake program is the only way into the database. Yes: the key decides which product it is, and the update cannot claim to be another. Decision: is the update in the right shape? No: turned away, with the problems listed; unknown kinds of update are refused. Yes: saved once, never edited, and a repeat of the same update is ignored. A phone alert goes out for each saved update, unless the product sent its own. The owner sees it on a private dashboard, behind a sign-in. Side notes: payments and forms arrive by their own doors, each signature-checked, and join at the same save; retries are safe, so a payment sent twice is still saved once.">

<sub>Editable diagram: [images/event-flow.drawio](images/event-flow.drawio) (opens in diagrams.net).</sub>

## Screenshots

<img src="images/ops-screens.png" width="100%" alt="Every product on one screen, and the money in one place. Fleet overview: Status pills, flags, the job queue, one tile per product, then each product's health. Money & funnels: One revenue card per product. Deposits are never counted as profit.">

## What this is

Every product generates different event types — Stripe charges, support emails, T BOT trade fills, YouTube views, lead form submissions, market settlements. Before consolidation, each product was monitored in its own console, with no single view of which product was performing in a given week.

The Unified Ops Dashboard is a single web app that ingests events from every product, stores them in one D1 table, and renders unified rollups: 7-day revenue, leads, email opens, T BOT P&L, per-funnel conversion. One screen, full picture.

## Architecture


<img src="images/architecture.png" width="100%" alt="Architecture diagram of the Unified Ops Dashboard. Products (T BOT, Canadian PR Mastery, Lean Body Blueprint, Reinvention Blueprint, JobScout, the web studio), webhooks (Stripe, signature-checked; Tally) and support email via Cloudflare Email Routing write to one Cloudflare ingest worker. Guard 1: the project comes from each product's own secret, never the body; Stripe needs an HMAC signature within a 5-minute window; every body is schema-validated. Guard 2: events land in one can't-be-edited D1 table, unified-events, de-duplicated on project and source id and never migrated in production; it is also T BOT's SIEM. The ops-read-api serves events, stats, incidents with a daily sweep, address reputation and Ask the SOAR, where Claude plans a search from an allow-list and never writes SQL. Guard 3: ops.tbot.trade on Pages sits behind Cloudflare Access with its default hostname redirected. Dev twins run on their own D1 with no routes or crons; GitHub runs the typecheck and the rule, schema and hunt checks before every deploy.">

## Key design decisions

### 1. One event table, designed to never migrate

```sql
CREATE TABLE events (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  project     TEXT NOT NULL,    -- 'tbot' | 'cpr' | 'lbb' | 'rbp' | 'maasai'
  event_type  TEXT NOT NULL,    -- 'daily_rollup' | 'lead' | 'purchase' | 'open' | ...
  title       TEXT NOT NULL,
  amount      REAL,
  currency    TEXT,
  source_id   TEXT,             -- product-side ID for dedup
  metadata    TEXT,             -- JSON blob — schema-less by design
  created_at  TEXT DEFAULT (datetime('now'))
);
CREATE UNIQUE INDEX idx_dedup ON events(project, source_id);
```

Schema is intentionally minimal. Every product writes through the same shape. The `metadata` JSON blob absorbs all per-product variation without requiring schema changes. **Every product. One schema. Zero migrations in production.**

### 2. Per-project ingest secrets, signed webhooks, separate write/read workers

Each product holds a per-project secret. POST `/ingest` rejects requests without matching `x-project-secret`. Stripe webhooks must also carry a valid HMAC signature inside a 5-minute window. Writing events goes through the ingest worker only; the read API serves the dashboard behind Cloudflare Access, so a failure on one side doesn't take down the other.

### 3. Append-only by design

Events are never updated, never deleted. Corrections are new events. Total source-of-truth for "what happened" across all products. The dashboard derives every visualization from rollups over this table.

### 4. Cloudflare-native end to end

<img src="images/ops-cloudflare.png" width="100%" alt="Everything runs on Cloudflare, one account and one bill. One vendor and one command-line tool keep a multi-product platform simple for a small team. The domain: tbot.trade and its DNS records. The two small programs: Cloudflare Workers: one takes events in, one serves the dashboard's data. The database: Cloudflare D1, a small SQL database at the edge. The dashboard itself: Cloudflare Pages, a plain static site. Email in and out: Cloudflare Email Routing passes support emails to the intake program; Cloudflare Email Sending sends the mail out. Who can see it: Cloudflare Access: a sign-in in front of every page.">

<details><summary>The services as a table</summary>

| Concern | Cloudflare service |
|---|---|
| Domain + DNS | `tbot.trade` zone |
| Edge routing | Workers (ingest, read-api) |
| Database | D1 (SQLite at the edge) |
| Static site | Pages (the dashboard itself) |
| Webhooks in | Email Routing → ingest worker (support email) |
| Auth | Cloudflare Access |

</details>

**One vendor. One CLI (`wrangler`). One bill. Operational simplicity at multi-product scale beats best-of-breed sprawl for a small team.**

## What you see on the dashboard

- **Headline cards:** 7-day revenue · 7-day leads · 7-day email opens · T BOT 7-day P&L
- **Per-project rollups:** CPR conversion funnel (leads → opened → clicked → purchased), RBP audience growth, T BOT win-rate trend, LBB sales velocity
- **Event stream:** chronological log of every event across all products, filterable by project and type
- **Project filters:** ALL · TBOT · CPR · LBB · RBP — instant pivot

## Tech stack details

- **Workers:** TypeScript, with every incoming event checked against a fixed format (Zod)
- **D1:** Single migration (`0001_events.sql`), never altered in production
- **Pages:** vanilla HTML/CSS/JS (no framework — render is fast and the markup is auditable)
- **Email webhook bridges:** Cloudflare Email Routing → custom address → Worker that turns each support email into an event and forwards it to Gmail
- **Local dev:** `wrangler dev` for both workers; `pnpm db:migrate:local` mirrors prod schema; dev secrets in `.dev.vars`

## What was hard

- **Event-shape ambiguity across products.** "What counts as a lead?" was different per product. Solved by keeping the `event_type` enum small (~6 values) and pushing all per-product specifics into the `metadata` JSON.
- **Deduplication of webhook deliveries.** Stripe and the internal products both retry. Solved with the `(project, source_id)` unique index — duplicate POSTs are silent no-ops.
- **One database, many consumers.** Solved by splitting write (ingest) and read (read-api) into separate workers with different D1 bindings. The read-api can be rate-limited or cached aggressively without affecting ingestion.

## Security operations on the same event store

The same table, which can't be edited, is also T BOT's security log (SIEM). Intrusion detection on the trading server, audit trails from both APIs, and the exchange-side reconciliation that catches a misused API key all write security events into it, and a separate SOAR page on the dashboard turns them into a verdict, a timeline and six response playbooks. The full write-up, with screenshots: **[kenmwara/tbot-security](https://github.com/kenmwara/tbot-security)**.

<a href="https://github.com/kenmwara/tbot-security"><img src="https://raw.githubusercontent.com/kenmwara/tbot-security/main/images/tsec-console.png" width="100%" alt="The response console, shown with made-up demo data. Every address, email and event here is invented, from ranges reserved for examples. The live page shows the same views with real data, behind a sign-in. Incidents: Related warnings grouped into one incident: open, acknowledged or resolved, with a button to ban an address. Live logs: Login, firewall and web logs streaming in, newest first, with filters and search. Overview: Health tiles, a seven-day timeline, the event feed and six ready-made responses. On a phone: The verdict and health tiles come first."></a>

## Planned enhancements

1. **Revenue alerting** — a summary when revenue drops sharply from one week to the next (security alerting already pages on Telegram)
2. **Per-product P&L view** with attribution (which ad campaign drove which lead drove which purchase)
3. **OpenTelemetry-style trace IDs** across products → dashboard, so a `lead → email_open → clicked → purchased` chain is visible as a single timeline

## Design rationale

Most operations dashboards are attached to a single product. This platform ingests from every product into *one D1
table* instead of running per-product analytics stacks, so onboarding a new product requires one HTTP call rather than
new infrastructure. Writing and reading run as two small Cloudflare Workers.

---

*Source code is private. The architecture, schema, and operational reasoning above are the showcase. Design discussions: [LinkedIn](https://linkedin.com/in/kenmwara).*
