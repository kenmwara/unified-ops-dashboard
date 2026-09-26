# Unified Ops Dashboard

> One dashboard, every product, a single Cloudflare-native data plane. The platform every other product in my portfolio reports into.

🌐 **Live:** ops.tbot.trade *(behind Cloudflare Access — operator only; see screenshots below)*
🏗️ **Stack:** Cloudflare D1 + Workers + Pages + Email Routing + Resend
🔒 **Source:** private — this README + screenshots are the showcase

---

## Screenshots

<p>
  <img src="https://tbot.trade/portfolio/img/ops.jpg" width="400" alt="ops.tbot.trade — fleet overview: health pills, flags, scheduler queue, one tile per product, per-project System health cards">
  &nbsp;&nbsp;
  <img src="https://tbot.trade/portfolio/img/ops-money.jpg" width="400" alt="ops.tbot.trade — Money & funnels: one revenue card per surface; deposits are never P&L">
</p>

*Left — the fleet overview behind Cloudflare Access: health pills, flags, the scheduler queue, one tile per product, then per-project System health cards. Right — Money & funnels: one revenue card per surface, one owner per fact, deposits never counted as P&L.*

## What this is

Every product generates different event types — Stripe charges, Resend opens, T BOT trade fills, YouTube views, lead form submissions, Kalshi resolutions. Before this dashboard existed, I had a tab open per product and no idea which product was actually working in any given week.

The Unified Ops Dashboard is a single web app that ingests events from every product, stores them in one D1 table, and renders unified rollups: 7-day revenue, leads, email opens, T BOT P&L, per-funnel conversion. One screen, full picture.

## Architecture

```
                   ┌──── T BOT (trading) ───────┐
                   ├──── CPR  (Canadian PR)  ───┤
  every product ── ┤──── LBB  (Lean Body)    ───┼─→  POST /ingest  (HMAC-signed)
                   ├──── RBP  (Reinvention)  ───┤         │
                   ├──── Maasai (YouTube)    ───┤         ▼
                   └──── Resend webhooks     ───┘   ┌─────────────────────┐
                                                    │  ingest-worker      │
                                                    │  (Cloudflare Worker)│
                                                    └──────────┬──────────┘
                                                               │
                                                               ▼
                                                    ┌─────────────────────┐
                                                    │  D1: unified-events │
                                                    │  ONE table, never   │
                                                    │  to be migrated.    │
                                                    └──────────┬──────────┘
                                                               │
                                                               ▼
                                                    ┌─────────────────────┐
                                                    │  read-api worker    │
                                                    │  GET /api/events    │
                                                    │  GET /api/stats     │
                                                    └──────────┬──────────┘
                                                               │
                                                               ▼
                                                    ┌─────────────────────┐
                                                    │  Pages site         │
                                                    │  ops.tbot.trade     │
                                                    └─────────────────────┘
```

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

### 2. HMAC-signed ingest, separate write/read workers

Each product holds a per-project secret. POST `/ingest` rejects requests without matching `x-project-secret`. The ingest worker has D1 write permission; the read-api worker has D1 read-only — failure on one doesn't compromise the other.

### 3. Append-only by design

Events are never updated, never deleted. Corrections are new events. Total source-of-truth for "what happened" across all products. The dashboard derives every visualization from rollups over this table.

### 4. Cloudflare-native end to end

| Concern | Cloudflare service |
|---|---|
| Domain + DNS | `tbot.trade` zone |
| Edge routing | Workers (ingest, read-api) |
| Database | D1 (SQLite at the edge) |
| Static site | Pages (the dashboard itself) |
| Webhooks in | Email Routing → ingest worker (e.g. Resend opens) |
| Auth | Cloudflare Access |

**One vendor. One CLI (`wrangler`). One bill. Operational simplicity at multi-product scale beats best-of-breed sprawl for a small team.**

## What you see on the dashboard

*(screenshots)*

- **Headline cards:** 7-day revenue · 7-day leads · 7-day email opens · T BOT 7-day P&L
- **Per-project rollups:** CPR conversion funnel (leads → opened → clicked → purchased), RBP audience growth, T BOT win-rate trend, LBB sales velocity
- **Event stream:** chronological log of every event across all products, filterable by project and type
- **Project filters:** ALL · TBOT · CPR · LBB · RBP — instant pivot

## Tech stack details

- **Workers:** TypeScript, ~150 lines per worker, Zod for schema validation
- **D1:** Single migration (`0001_events.sql`), never altered in production
- **Pages:** vanilla HTML/CSS/JS (no framework — render is fast and the markup is auditable)
- **Email webhook bridges:** Cloudflare Email Routing → custom address → Worker that parses Resend digest emails into events
- **Local dev:** `wrangler dev` for both workers; `pnpm db:migrate:local` mirrors prod schema; dev secrets in `.dev.vars`

## What was hard

- **Event-shape ambiguity across products.** "What counts as a lead?" was different per product. Solved by keeping the `event_type` enum small (~6 values) and pushing all per-product specifics into the `metadata` JSON.
- **Deduplication of webhook deliveries.** Resend retries; Stripe retries; my own product retries. Solved with the `(project, source_id)` unique index — duplicate POSTs are silent no-ops.
- **One database, many consumers.** Solved by splitting write (ingest) and read (read-api) into separate workers with different D1 bindings. The read-api can be rate-limited or cached aggressively without affecting ingestion.

## Security operations on the same event store

The same append-only table is also T BOT's SIEM. Intrusion detection on the trading server, audit trails from both APIs, and the exchange-side reconciliation that catches a misused API key all write security events into it, and a separate SOAR page on the dashboard turns them into a verdict, a timeline and six response playbooks. The full write-up, with screenshots: **[kenmwara/tbot-security](https://github.com/kenmwara/tbot-security)**.

<p>
  <a href="https://github.com/kenmwara/tbot-security"><img src="https://raw.githubusercontent.com/kenmwara/tbot-security/main/docs/soar-desktop.png" width="400" alt="The SOAR page in demo mode: posture tiles, an incident timeline, the event feed and response playbooks"></a>
</p>

*The SOAR page in its synthetic demo mode. The live page sits behind Cloudflare Access.*

## What I'd build next

1. **Revenue alerting** — a summary when revenue drops more than X% week-over-week (security alerting already pages on Telegram)
2. **Per-product P&L view** with attribution (which ad campaign drove which lead drove which purchase)
3. **OpenTelemetry-style trace IDs** across products → dashboard, so a `lead → email_open → clicked → purchased` chain is visible as a single timeline

## Why this is interesting

Most "ops dashboards" are bolted onto a single product. This one ingests from every product I run. The trick wasn't the dashboard — it was deciding to ingest *into one D1 table* instead of running per-product analytics stacks. That decision means every new product I ship gets dashboard support by writing one HTTP call instead of standing up new infrastructure.

It's the most leverage I've ever gotten out of 200 lines of Cloudflare Workers code.

---

*Source code is private. The architecture, schema, and operational reasoning above are the showcase. If you'd like to discuss the design decisions, [hit me up](https://linkedin.com/in/kenmwara).*
