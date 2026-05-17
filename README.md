# Unified Ops Dashboard

> One dashboard, six products, single Cloudflare-native data plane. The platform every other product in my portfolio reports into.

🌐 **Live:** ops.tbot.trade *(behind Cloudflare Access — operator only; see screenshots below)*
🏗️ **Stack:** Cloudflare D1 + Workers + Pages + Email Routing + Resend
🔒 **Source:** private — this README + screenshots are the showcase

---

## What this is

Six products. Each generating different event types — Stripe charges, Resend opens, T BOT trade fills, YouTube views, lead form submissions, Kalshi resolutions. Before this dashboard existed, I had six tabs open and no idea which product was actually working in any given week.

The Unified Ops Dashboard is a single web app that ingests events from every product, stores them in one D1 table, and renders unified rollups: 7-day revenue, leads, email opens, T BOT P&L, per-funnel conversion. One screen, full picture.

## Architecture

```
                   ┌──── T BOT (trading) ────────┐
                   ├──── CPR  (Canadian PR)  ───┤
   six products ──┤──── LBB  (Lean Body)    ───┼─→  POST /ingest  (HMAC-signed)
                   ├──── RBP  (Reinvention)  ───┤         │
                   ├──── Maasai (YouTube)    ───┤         ▼
                   └──── Resend webhooks     ───┘    ┌─────────────────────┐
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

Schema is intentionally minimal. Every product writes through the same shape. The `metadata` JSON blob absorbs all per-product variation without requiring schema changes. **Six products. One schema. Zero migrations in production.**

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
| Auth (later) | Cloudflare Access |

**One vendor. One CLI (`wrangler`). One bill. Operational simplicity at six-product scale beats best-of-breed sprawl for a solo operator.**

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

## What I'd build next

1. **Cloudflare Access in front of `/api/events` and the dashboard** (currently in front but ingest stays public-with-HMAC)
2. **Alerting** — daily summary email via Resend when revenue drops more than X% week-over-week
3. **Per-product P&L view** with attribution (which ad campaign drove which lead drove which purchase)
4. **OpenTelemetry-style trace IDs** across products → dashboard, so a `lead → email_open → clicked → purchased` chain is visible as a single timeline

## Why this is interesting

Most "ops dashboards" are bolted onto a single product. This one ingests from six. The trick wasn't the dashboard — it was deciding to ingest *into one D1 table* instead of running per-product analytics stacks. That decision means every new product I ship gets dashboard support by writing one HTTP call instead of standing up new infrastructure.

It's the most leverage I've ever gotten out of 200 lines of Cloudflare Workers code.

---

*Source code is private. The architecture, schema, and operational reasoning above are the showcase. If you'd like to discuss the design decisions, [hit me up](https://linkedin.com/in/kenmwara).*
