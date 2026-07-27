# Operator Dashboard — Plan

Goal: a dashboard view inside the app where the operator (you) sees usage costs, run history, what the platform can and cannot backtest, which external services are in use, and recent activity/logs. Local, single-user, no auth — revenue and multi-user stay out of scope until there are paying users (per the product plan's commercialization gate).

## Principles

- **Generated, not hand-written.** The capabilities documentation is derived from the code that enforces it (the indicator enum, the schema, provider constants) so it can never drift from what actually runs.
- **The ledger is the source of truth for cost.** Every LLM call is persisted the moment it happens — including translate calls that never become a run.
- **Reuse the run store.** "My backtests" is a view over the existing SQLite runs table.

## Implementation units

### D1. LLM usage ledger (backend)
- New `llm_calls` table: id, timestamp, purpose (translate/interpret), model, input/output/cache tokens, cost_usd, run_id (nullable — translate calls precede runs).
- Translator and interpreter write to it at call time.
- `GET /api/usage` — totals, breakdown by model and purpose, per-day series, cost of last N runs.
- Tests: ledger written per call; aggregation math on fixtures.

### D2. Capabilities endpoint (backend)
- `GET /api/capabilities` — generated from code:
  - Indicators: the 8-indicator vocabulary with params + defaults (from `IndicatorName` + `DEFAULT_PERIODS`).
  - Condition/rule types: crossover, comparison, all/any groups, risk rules (stop/tp/trailing, sizing) — what a strategy can express, with plain-English examples.
  - Asset classes + data providers + example symbols; timeframes with their range limits (hourly ~730 days, daily/weekly unrestricted).
  - Services in use: Anthropic models + per-MTok pricing, yfinance, ccxt/binance, backtesting.py version, storage paths.
- Friendly `/` root: app name + links to `/docs`, `/api/capabilities`, `/health`.

### D3. Run history hardening (backend)
- Unify GET-run payload with the POST-response shape (review finding — the frontend type only matches POST today).
- List endpoint gains summary fields: symbol, timeframe, return %, trades, total cost.
- `DELETE /api/backtests/{id}`.

### D4. Activity log (backend)
- Middleware appends API events (method, path, status, duration ms, error detail) to a capped `events` table; `GET /api/logs?limit=100`.
- Engine warnings and escalation errors surface here too.

### D5. Dashboard view (frontend)
- Top-level nav: **Backtest | Dashboard** (state switch, no router yet).
- Sections:
  1. **Usage & costs** — total spend, per-model/per-purpose breakdown, per-day bars, cost per backtest.
  2. **My backtests** — history table (date, symbol, strategy name, return, trades, cost) → click opens the stored run in the existing results view (needs D3).
  3. **What you can backtest** — rendered from `/api/capabilities`: indicator table, rule types with examples, symbols/asset classes, timeframe limits.
  4. **System** — services + models + pricing in use, storage locations, recent activity log, latest engine warnings.

### D6. Deferred-review cleanups (ride along)
- Chart re-colors on OS theme change (advisory from Phase C review).
- App favicon/title polish.

## Sequencing

D1 → D2 → D3 → D4 (backend, each independently testable) → D5 (consumes all four) → D6.

## Explicitly later

Revenue tracking (needs a paid tier), auth/accounts, hosted deployment (AGPL engine gate), strategy library/versioning (plan already defers), alerting.
