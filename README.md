# 📊 NAT Funds — Institutional-Grade Mutual Fund Analytics

A full-stack Node.js platform that pulls live data from AMFI, NSE/BSE, and RBI — then computes the same risk and performance metrics used by professional fund analysts. Built for India's 9,100+ mutual fund schemes. No paid APIs. No dashboards-as-a-service. Just raw data feeds, real math, and a fast no-build-step frontend.

![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat-square&logo=node.js&logoColor=white)
![Express](https://img.shields.io/badge/Express-000000?style=flat-square&logo=express&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=flat-square&logo=javascript&logoColor=black)
![Chart.js](https://img.shields.io/badge/Chart.js-FF6384?style=flat-square&logo=chartdotjs&logoColor=white)
![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=flat-square&logo=html5&logoColor=white)
![CSS3](https://img.shields.io/badge/CSS3-1572B6?style=flat-square&logo=css3&logoColor=white)

---

## What this project does

Most retail mutual fund tools in India either show you a star rating and call it a day, or hide real metrics behind a paywall. I wanted something closer to what a CFA analyst would actually look at — drawdown recovery dates, rolling return distributions, benchmark-relative alpha, Sortino, Calmar — all in one place, without a Bloomberg terminal.

NAT Funds is that tool.

It aggregates live data from six sources, runs 12+ quantitative calculations per fund, and serves everything through a paginated REST API with a three-tier caching system — so it's fast on cold boot and near-instant on warm restarts.

---

## At a Glance

| Stat | Value |
|---|---|
| AMFI schemes tracked | **9,100+** |
| Quantitative metrics computed per fund | **12+** |
| Real data sources integrated | **6** (AMFI · mfapi.in · NSE · BSE · RBI · AMFI TER XLSX) |
| Caching tiers | **3** — persistent JSON · per-scheme disk · processed snapshot |
| Funds with full NAV history + all metrics | **~3,000** |
| Rate limit | **200 requests / min / IP** |
| Smoke test suite runtime | **~0.5s** (9 tests, zero network calls) |
| Build steps required | **0** — vanilla HTML + JS modules, runs straight from clone |

---

## Data Sources

| Source | What It Provides | Refresh |
|---|---|---|
| `amfiindia.com` NAV feed | Live NAV for ~9,100 schemes | Boot / on-demand |
| `mfapi.in` | Full per-scheme NAV history | Per-scheme disk cache (24h TTL) |
| AMFI Fund Performance API | AUM, benchmark map, AMFI IR (1Y/3Y/5Y/10Y), SEBI Riskometer | Daily cron at **09:30 IST** |
| AMFI TER XLSX | Total Expense Ratio per scheme | Daily cron at **09:00 IST** |
| NSE / BSE Indices API | Benchmark TRI history — used for Beta, Alpha, IR, Capture Ratios | Daily cron at **09:31 IST** |
| CCIL India (RBI 91-day T-bill) | Risk-free rate for Sharpe / Sortino / Jensen's Alpha | 24h disk cache |

---

## What It Computes

Every fund goes through `services/metricsCalculator.js` — roughly **1,200 lines of pure functions**, no shared state, no side effects.

### Returns

| Metric | Methodology |
|---|---|
| CAGR (1Y / 3Y / 5Y / Since Inception) | Closest-previous NAV lookup; tolerates up to a 7-day gap; N.A. if history is too short |
| Rolling Returns (1Y) | Daily-stepped windows across full NAV history; surfaces min / max |
| Rolling Returns (3Y) | Monthly-stepped (one per calendar month); p10 / p25 / p75 / p90 distribution |

### Risk

| Metric | Methodology |
|---|---|
| Standard Deviation | Annualised sample StdDev of monthly returns; 36-month lookback (min 12) |
| Max Drawdown | Two-pass O(n) scan — returns peak date, trough date, **and recovery date** |
| SEBI Riskometer | Official AMFI label when available; falls back to within-category StdDev percentile |

### Risk-Adjusted Returns

| Metric | Methodology |
|---|---|
| Sharpe Ratio | `(avg_excess_return / σ_fund) × √12` · excess return uses geometric monthly RFR from RBI T-bill |
| Sortino Ratio | Same as Sharpe but denominator = downside deviation only |
| Calmar Ratio | `3Y CAGR% / |Max Drawdown%|` · null-safe if either input is missing |

### Benchmark-Relative

| Metric | Methodology |
|---|---|
| Beta | Covariance / variance · 36 aligned monthly fund vs TRI returns; requires real TRI data |
| Jensen's Alpha | `AnnFundReturn − [Rf + β × (AnnBenchReturn − Rf)]` · geometric annualisation |
| Information Ratio | `(AnnFundReturn − AnnBenchReturn) / Tracking Error` · requires ≥36 aligned months |
| R² | Pearson correlation² between fund and benchmark monthly returns (0–100 scale) |
| Upside / Downside Capture | Morningstar method · requires ≥10 up-months and ≥6 down-months |

### Peer Scoring

| Metric | Methodology |
|---|---|
| Consistency Score (0–10) | 30% median 3Y rolling · 20% rolling positive % · 20% downside capture (inverted) · 20% Sortino · 10% 3Y CAGR — percentile-normalised **within sub-category** |

### AMFI-Published (Sourced Directly — Not Calculated)

| Field | Source |
|---|---|
| AMFI Information Ratios (1Y / 3Y / 5Y / 10Y, Direct + Regular) | AMFI Fund Performance API → `ir-data.json` |
| AUM (₹ Crore) | AMFI Fund Performance API → `aum-data.json` |
| Total Expense Ratio | AMFI TER XLSX parsed with ExcelJS → `ter-data.json` |
| Benchmark Name | AMFI Fund Performance API → `benchmark-data.json` (90-day TTL) |

---

## Pipeline Overview

```
AMFI NAV feed (~9,100 schemes)
        ↓
  amfiParser.js — selects top funds
        ↓
  dataFetcher.js — batch NAV history (10 concurrent, disk-cached)
        ↓
  dataHelpers.js — enriches each fund: TER · AUM · AMFI IR · Benchmark · Riskometer
        ↓
  metricsCalculator.js — pure functions; ~1,200 lines
  ┌─────────────────────────────────────────────────────────────┐
  │  CAGR 1Y / 3Y / 5Y / Since Inception                      │
  │  Rolling Returns (1Y daily · 3Y monthly p10–p90)           │
  │  StdDev · Max Drawdown (peak → trough → recovery)          │
  │  Sharpe · Sortino · Calmar                                 │
  │  Beta · Jensen's Alpha · IR · R²                           │
  │  Upside/Downside Capture · Consistency Score (0–10)        │
  └─────────────────────────────────────────────────────────────┘
        ↓
  cache/processed_funds.json (24h TTL) — skips re-compute on warm restart
        ↓
  Express REST API → Vanilla JS SPA (no build step)
```

---

## Architecture

```
server.js                         ← Express setup, security headers, route mounting, boot trigger
│
├── shared/
│   ├── appState.js               ← Single source of truth for in-memory state
│   └── logger.js                 ← Pino structured logger (level from LOG_LEVEL env var)
│
├── boot/
│   ├── startup.js                ← Full boot() sequence: parse → fetch → compute
│   └── dataHelpers.js            ← Per-fund enrichment: TER, AUM, AMFI IR, Benchmark, Riskometer
│
├── routes/
│   ├── api.js                    ← /api/* REST endpoints
│   ├── admin.js                  ← /admin/* sync endpoints (auth-protected)
│   └── ter.js                    ← /ter/:schemeCode TER lookup
│
├── services/
│   ├── amfiParser.js             ← Parses live AMFI NAV text feed; selects top funds
│   ├── dataFetcher.js            ← NAV history fetch (mfapi.in) + per-scheme disk cache
│   ├── metricsCalculator.js      ← All quantitative metrics (~1,200 lines, pure functions)
│   ├── fundPerformanceService.js ← AUM, Benchmark, AMFI IR, SEBI Riskometer
│   ├── terService.js             ← TER index from AMFI XLSX via ExcelJS
│   ├── triService.js             ← Benchmark TRI history from NSE/BSE Indices APIs
│   └── riskFreeRate.js           ← RBI 91-day T-bill; weekly in-memory cache
│
├── data/                         ← Persistent JSON stores (committed to repo)
│   ├── ter-data.json
│   ├── aum-data.json
│   ├── benchmark-data.json       ← 90-day TTL (benchmarks are semi-permanent per SEBI)
│   ├── ir-data.json
│   ├── riskometer-data.json
│   └── tri-data.json
│
├── cache/                        ← Git-ignored; auto-populated at runtime
│   ├── nav_<schemeCode>.json     ← Per-scheme NAV history (24h TTL)
│   ├── processed_funds.json      ← Full parsed + computed fund list (24h TTL)
│   └── ter-parsed.json           ← Pre-parsed TER; avoids re-downloading AMFI XLSX
│
├── tests/
│   └── smoke.test.js             ← 9 import/shape tests · node:test · ~0.5s · no network
│
├── dev-tools/                    ← 19 standalone diagnostic scripts; never imported by server
├── scripts/                      ← AUM normalisation audit tools (read-only)
│
└── public/                       ← Frontend SPA — no build step
    ├── index.html                ← SPA shell; modules loaded in strict dependency order
    ├── styles.css
    └── js/
        ├── constants.js          ← CATEGORY_META, METRIC_TOOLTIPS
        ├── state.js              ← Frontend state object
        ├── api.js                ← fetch() wrapper around /api/*
        ├── formatters.js         ← fmt, fmtNav, fmtAUM, fmtScore, etc.
        ├── router.js             ← Hash-based routing helpers
        ├── ui.js, search.js, compare.js, charts.js, init.js
        └── views/
            ├── home.js           ← Dashboard / category cards
            ├── explore.js        ← Fund table + filters + pagination
            ├── fund-detail.js    ← Full per-fund analytics page
            └── compare.js        ← Side-by-side comparison (up to 4 funds)
```

---

## Caching Strategy

Three tiers work together so the server never makes redundant network calls:

**Tier 1 — `data/*.json` (committed to repo)**
Persistent stores for TER, AUM, TRI, benchmarks, AMFI IR, and riskometers. Refreshed by cron jobs and admin endpoints. Benchmark data carries a **90-day TTL** — SEBI mandates fund benchmarks as semi-permanent, so daily refreshes would just be noise.

**Tier 2 — `cache/nav_<schemeCode>.json` (git-ignored)**
Per-scheme NAV history from mfapi.in. **24-hour TTL**. Populated on first fetch; no re-download within the window.

**Tier 3 — `cache/processed_funds.json` (git-ignored)**
The full parsed + computed fund list with all 12+ metrics applied. **24-hour TTL**. On a warm restart within 24 hours, boot skips the entire parse → fetch → compute cycle and loads this file directly. Cold boot to warm boot is the difference between minutes and seconds.

**Bonus — `cache/ter-parsed.json` (git-ignored)**
Pre-parsed TER data to avoid re-downloading the large AMFI XLSX on every restart.

---

## Boot Sequence

When the server starts, here's exactly what happens under the hood:

```
1. Load TER index from data/ter-data.json (or sync from AMFI if missing)
2. Load AUM, Benchmark, AMFI IR, Riskometer from data/ (or sync if stale)
3. Fetch risk-free rate from RBI (91-day T-bill) — weekly in-memory cache
4. Check cache/processed_funds.json — if < 24h old, load it and skip to step 8 ✓
5. Parse live AMFI NAV feed (~9,100 schemes)
6. Batch-fetch historical NAV for top ~3,000 schemes — 10 concurrent, disk-cached
7. Compute all metrics in memory → save processed_funds.json
8. Set dataReady = true → serve the SPA
```

The frontend shows a **live progress bar** (polling `/api/status`) during steps 5–7.

---

## API Reference

### Public Endpoints (`/api/*`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/status` | Server readiness + boot progress |
| `GET` | `/api/categories` | Category/sub-category counts; supports `?planType=` and `?optionType=` |
| `GET` | `/api/funds` | Paginated, filtered, sorted fund list |
| `GET` | `/api/fund/:schemeCode` | Single fund with live NAV sync + on-the-fly metric recalculation |
| `GET` | `/api/fund/:schemeCode/nav-history` | Sampled monthly NAV for charts; `?period=1y\|3y\|5y\|max` |
| `GET` | `/api/compare?codes=...` | Side-by-side comparison for 2–4 scheme codes |
| `GET` | `/api/search?q=...` | Full-text search across scheme name + AMC (top 20 results) |
| `GET` | `/ter/:schemeCode` | TER lookup for a single scheme |

### `/api/funds` Query Parameters

| Param | Example | Description |
|---|---|---|
| `type` | `Equity` | Filter by fund type |
| `subCategory` | `Large Cap` | Sub-category filter (comma-separated for multiple) |
| `planType` | `Direct` | `Direct` or `Regular` |
| `optionType` | `Growth` | `Growth` or `IDCW` |
| `search` | `HDFC` | Full-text across scheme name and AMC |
| `sortBy` | `cagr3y` | Any metric field including nested rolling return fields |
| `order` | `desc` | `asc` or `desc` |
| `page` | `1` | 1-indexed |
| `limit` | `20` | Max 100 |

### Admin Endpoints (`/admin/*`)

Require `X-Admin-Secret` header when `ADMIN_SECRET` env var is set.

```
GET /admin/sync-ter   → Re-download AMFI TER XLSX; rebuild ter-data.json
GET /admin/sync-aum   → Refresh AUM + AMFI IR + SEBI Riskometer
GET /admin/sync-tri   → Refresh all benchmark TRI histories from NSE/BSE
```

---

## How to Run

**Prerequisites:** Node.js 18+, internet access on first boot.

```bash
git clone https://github.com/harleenkhanuja/NAT-Funds.git
cd NAT-Funds
npm install

npm start        # production  — node server.js
npm run dev      # development — nodemon (auto-restart on changes)
npm test         # smoke tests — 9 tests, ~0.5s, no network
```

**Environment variables (`.env`):**

| Variable | Default | Description |
|---|---|---|
| `PORT` | `3001` | HTTP port |
| `ADMIN_SECRET` | *(empty)* | If set, `/admin/*` requires `X-Admin-Secret` header |
| `CORS_ORIGIN` | *(empty)* | Adds `Access-Control-Allow-Origin` — leave empty; frontend is served by this same server |
| `LOG_LEVEL` | `info` | Pino log level: `trace` · `debug` · `info` · `warn` · `error` · `fatal` |

---

## Dev Tools & Audit Scripts

**`dev-tools/`** — 19 standalone diagnostic runners. Never imported by the server. Safe to run at any time.

```bash
node dev-tools/test-api.js        # Basic API smoke test
node dev-tools/test-metrics.js    # Metrics calculator unit checks
node dev-tools/diagnose.js        # Data-coverage diagnostic report
node dev-tools/test-benchmark.js  # TRI benchmark lookup check
```

**`scripts/`** — Read-only data-quality audit tools.

```bash
node scripts/audit_aum_norm.js    # AUM match-rate audit across all 9,100+ funds
node scripts/test_normalise.js    # Unit tests for normaliseName() + false-positive check
```

---

## A Few Design Decisions Worth Explaining

**No build step.** The frontend is plain HTML + vanilla JS split into modules loaded in strict dependency order. No webpack, no bundler, no compile-time surprises. Clone, run, done.

**Pure functions for all math.** Every metric in `metricsCalculator.js` is a pure function — no shared state, no mutation. Easy to unit test, easy to reason about when a number looks wrong.

**Benchmark TTL is 90 days, not 24h.** SEBI mandates fund benchmarks as semi-permanent. Refreshing daily would be noise. The benchmark endpoint re-syncs only when `benchmark-data.json` is older than 90 days.

**IDCW methodology.** Rolling returns for IDCW plans reflect NAV movement only — dividends paid out are excluded. This matches Tickertape and INDmoney methodology, so comparisons between Direct and Regular plans stay apples-to-apples.

**The processed_funds cache matters more than it sounds.** On cold boot, parsing 9,100 schemes + fetching NAV history for ~3,000 of them + computing 12+ metrics per fund takes a few minutes. The 24h cache turns every subsequent restart into a near-instant load. Deliberate architectural choice, not an afterthought.

---

## What I Learned Building This

The hardest part wasn't the math — it was data plumbing. Getting clean, aligned monthly returns for a fund *and* its benchmark TRI on the same dates, with no gaps, is genuinely tricky when you're pulling from three different APIs with different response formats, different date conventions, and different update schedules.

The AUM normalisation problem was a close second. AMFI publishes AUM with scheme names that don't always match the NAV feed names exactly. I ended up writing a `normaliseName()` function and a dedicated audit script (`scripts/audit_aum_norm.js`) to measure and improve the match rate across all 9,100+ funds.

The three-tier caching design came from watching the cold boot time and deciding that a multi-minute wait on every restart was not acceptable for a tool I actually wanted to use day to day.

---

## Tech Stack

| Layer | Tech |
|---|---|
| Runtime | Node.js 18+ |
| Framework | Express |
| Logging | Pino (structured, level-configurable) |
| Excel parsing | ExcelJS (AMFI TER XLSX) |
| Frontend | Vanilla HTML + JS modules + Chart.js |
| Testing | node:test (built-in, zero extra dependencies) |
| Dev runner | nodemon |

**Languages:** JavaScript (89.1%) · HTML (5.4%) · Python (3.9%) · CSS (1.6%)

---

## Notes

- `dev-tools/app_original.js` is the original monolithic frontend (~88 KB) kept for reference only. The live app loads exclusively from `public/js/` modules.
- The `cache/` directory is git-ignored. On a fresh clone it is empty; it fills automatically during first boot.
- `data/*.json` files can optionally be git-ignored — the app re-syncs them from AMFI/NSE/BSE on first boot.
- All financial calculations are strictly read-only. No metric computation mutates external state.
- Rate limiting is enforced at the Express layer: **200 requests per minute per IP** (configurable in `server.js`).

---

**Made by [Harleen Khanuja](https://www.linkedin.com/in/harleenkhanuja/)**
