# AGENTS.md

## What this is

AI strategy backtesting platform: user describes a trading strategy in plain English → LLM translates to a constrained schema → deterministic recap confirmed by user → wrapped `backtesting.py` engine runs it on free market data → metrics passthrough + chart + LLM interpretation. Local-only, single-user, no auth (deliberate). FastAPI backend (`backend/`, Python 3.13 + uv) + React/Vite/TS frontend (`frontend/`, lightweight-charts v5). Branch: `feat/backtest-platform-core` — never commit to main directly.

## Architecture / trust spine

The product's trust guarantee is two invariants — protect them in every change:

1. **What you confirm is what runs**: the recap (`backend/app/schema/recap.py`) is rendered deterministically from the parsed Pydantic schema, NEVER written by the LLM. Every executable field must appear in the recap text (property test enforces this). The LLM only *proposes* a schema; low-confidence parses populate `clarifications_needed[]` instead of guessing.
2. **What you see is what the engine computed**: metrics are passed through field-for-field from `backtesting.py`'s own stats (KTD-5). The compile/serialize layers add ZERO arithmetic. Raw stats include pandas Timestamps — `api/serialize.py` handles that; don't recompute anything.

Pipeline: `POST /api/translate` (Haiku, escalate-once to Sonnet on low confidence/validation failure) → recap + clarifications → user confirms → `POST /api/backtests` (compile → EngineAdapter → stats passthrough, synchronous) → interpretation (Sonnet, structured strengths/weaknesses/forward_looking + constant non-LLM disclaimer; interpretation outage never loses a run) → SQLite persist. Strategy fingerprinting (`schema/fingerprint.py`) dedupes runs of the "same" strategy into a `strategies` table for aggregate track records.

**AGPL license gate (KTD-1)**: `backtesting.py` is AGPL-3.0. `backend/app/engine/backtesting_py.py` is the ONLY module allowed to import the `backtesting` package. Everything else goes through `engine/adapter.py::EngineAdapter`. Swapping the engine is a hard documented prerequisite to any paying user. Do not leak engine imports elsewhere.

## Repo map

Backend (`backend/app/`):
- `schema/strategy.py` — the central contract: Pydantic strategy schema. 13 indicators (sma, ema, rsi, macd, bollinger, atr, stochastic, volume_sma, vwap, donchian, session_level, adx, roc), condition types crossover/comparison/session/day_of_week/streak, `all|any` groups (one nesting level), risk block (percent/ATR stop, ATR- or percent-take-profit, trailing stop, fixed-fraction sizing). **Bidirectional strategies**: `entry_short`/`exit_short` let one strategy trade both directions (e.g. breakout either way) — canonical shape is always long-primary + optional short-secondary, never the reverse, so fingerprints stay stable. Structured-outputs-safe: `additionalProperties: false`, no recursion.
- `schema/recap.py` — deterministic recap renderer (trust mechanism, never LLM).
- `schema/fingerprint.py` — sha256 of strategy dict minus top-level name/confidence/clarifications_needed, None-stripped recursively. Exclusion is top-level ONLY (nested `IndicatorRef.name` is identity). Changing this breaks existing strategy identities in the DB.
- `indicators/library.py` — hand-rolled, golden-tested pandas indicators (no TA lib dependency). Wilder RSI/ATR, SMA-seeded EMA, shift(1)-lagged Donchian. Grows ONLY with golden tests.
- `engine/adapter.py` — `EngineAdapter` interface, `EngineSettings` (has `commission`, default 0.0, not yet surfaced in UI), `BacktestResult`.
- `engine/backtesting_py.py` — AGPL isolate. Picks `FractionalBacktest` for crypto/forex (`_FRACTIONAL_UNITS`) vs whole-share `Backtest` for equities. Gotcha: precomputed ATR is original-scale and must be rescaled by `fractional_unit` (`price_scale`) before combining with the engine's internally-scaled Close in stop/trailing math.
- `engine/compiler.py` — schema → runnable strategy. Refuses drafts with pending clarifications unless `allow_pending_clarifications` (the `force` flag); the exit-mechanism invariant stays unconditional (truly-incomplete still 422s); skipped clarifications surface as `result.warnings`. Compiles `entry_short`/`exit_short` into separate signal series alongside primary entry/exit; engine routes long vs short off `compiled.direction` and per-trade `trade.is_long` (not a strategy-wide bool) — same-bar long+short signal conflict enters neither side.
- `data/` — `provider.py` protocol, yfinance (equity/forex) + ccxt (crypto), parquet cache, 1h→4h resample, `sessions.py` (asian 00-09 / london 08-17 / new_york 13-22 UTC, overlap by design). Free hourly history ~730 days — typed error, mapped to 422.
- `llm/` — `client.py` (routing, prompt cache, cost log), `translator.py` (Haiku→Sonnet escalation), `interpreter.py` (digest-based; zero-trade runs skip the LLM call; disclaimer is a constant).
- `analysis/` — `montecarlo.py` (trade-resampling bootstrap, `POST /api/backtests/{id}/montecarlo`, NOT persisted), `rating.py` (deterministic Sharpe→F..A+ grade; <100 trades caps at C, <500 caps at A).
- `api/routes.py` — all endpoints: translate, backtests CRUD, montecarlo, strategies, usage, capabilities, logs, `POST /api/backtests/batch` (run one strategy across up to 10 symbols in one request, interpretation skipped per-symbol for cost, per-symbol failure isolation, returns a `comparison` sorted by return), `POST /api/backtests/{id}/interpret` (fetch interpretation for one run on demand — used by the batch comparison view). `_execute_one()` is the extracted single-symbol run body shared by both the solo and batch paths. Typed data errors → 422 envelopes.
- `storage/db.py` — SQLite, WAL + busy_timeout. **No migration framework**: grow schema via guarded `ALTER TABLE` (check `PRAGMA table_info` first, then ALTER + backfill) — see `_migrate_runs_strategy_id` as the pattern.
- `api/capabilities.py` + `scripts/generate_capabilities_doc.py` → `docs/TA_CAPABILITIES.md` is a **generated artifact** with a byte-for-byte freshness test (`tests/api/test_capabilities_doc.py`). Rerun the script after ANY vocab change or pytest fails.

Frontend (`frontend/src/`):
- `App.tsx` — state-machine SPA: describe → recap → setup → running → results | batch-results; pages `backtest` | `library` | `dashboard`. `handleRerun()` re-enters the flow at `setup` with a synthetic `TranslateResponse` built from a saved strategy, skipping translation entirely — used by "Backtest again". `handleRunBatch()` takes the same confirmed strategy across multiple symbols and lands on `batch-results` instead.
- `pages/BatchResultsPage.tsx` — comparison table (return/Sharpe/max-DD/trades/rating per symbol), per-row on-demand "Interpret" button, click a symbol to drill into its full `ResultsPage`. Reached from `SetupPage`'s optional "Also backtest on" market chips.
- `pages/LibraryPage.tsx` — consumer-facing: `StrategiesPanel` + `CapabilitiesPanel` only, no cost/usage/system content. `pages/DashboardPage.tsx` is the internal operator view (usage/cost/logs/system) — NOT the end-user product; its tagline says so explicitly.
- `chartData.ts` — result → chart series mapping; `cssColor()` resolves oklch CSS tokens via a probe `<span>` (lightweight-charts can't parse oklch — always go through this); `classifyIndicator()` routes oscillator-shaped indicator labels (RSI/Stochastic/MACD/ATR/volume SMA) to their own chart pane + price scale, separate from price-shaped overlays — prevents a 0-100 RSI series from crushing the candlesticks' scale.
- `components/Chart.tsx` — price + overlays + markers + sessions + oscillator pane only. `components/EquityChart.tsx` — a FULLY SEPARATE `createChart` instance for the equity curve (not a second pane of Chart.tsx — that was tried and reverted because it didn't read as "a separate chart"; the two chart's x-axes are intentionally NOT synced).
- `components/SessionShading.ts` — hand-rolled v5 series primitive (v5 has no background-band API).
- `components/StrategyContextPanel.tsx` — this-run vs consolidated-across-all-runs metrics + "Backtest again" button, shown above `MonteCarloPanel` on the results page.
- `hooks/useSpeechToText.ts` + `speech.d.ts` — mic-to-text on the describe page via the browser-native Web Speech API ($0, Chrome/Edge only; no backend). A paid fallback (Groq Whisper-Large-v3-Turbo, ~$0.0006/min) is a documented future option, not built.
- `components/dashboard/` — operator dashboard panels (usage, run history, strategies, capabilities, system) — `StrategiesPanel` and `CapabilitiesPanel` are reused as-is by `LibraryPage` (already consumer-safe, no cost/version info in them).

Docs: `docs/2026-07-13-001-...plan copy.md` (original plan, KTD-1..8), `docs/2026-07-14-002-...` (operator dashboard plan), `PRODUCT.md` (design register: WCAG AA, anti-refs crypto-casino/generic-SaaS).

## Run & test (verified 2026-07-16)

```
cd backend  && uv run uvicorn app.main:app --reload   # http://localhost:8000
cd frontend && npm run dev                            # http://localhost:5173

cd backend  && uv run pytest           # offline suite — 277 passed, 4 deselected, ~7s
cd backend  && uv run pytest -m live   # real yfinance/ccxt fetches
cd backend  && uv run pytest -m eval   # translation eval, live API, costs cents, ~2min (28 cases)
cd backend  && uv run pytest tests/e2e # full loop, mocked LLM/data
cd frontend && npx vitest run          # 99 passed / 19 files
cd frontend && npm run build           # tsc -b + vite build
```

## Conventions & gotchas

- **ANTHROPIC_API_KEY** lives in gitignored `backend/.env`. It was once pasted into tracked `.env.example` — never let it into git.
- **Engine conventions asserted by tests**: orders fill next bar open; percent/ATR stops AND take-profits anchor to signal-bar close (not fill price); trailing stop re-anchors from highest high since entry, per-trade via `trade.is_long`; `EngineError` raised when cash can't afford one whole unit (engine would otherwise silently cancel and report 0 trades). Same-bar long+short entry signal conflict enters neither side.
- **Anchored indicators** (`vwap`, `session_level`) take NO `period` field — validators reject one. `session_level` requires `session`.
- **tsconfig has `erasableSyntaxOnly`** — constructor parameter properties (`constructor(private x: T)`) are forbidden; assign fields in the body (matters for chart primitive classes).
- Frontend tests stub `lightweight-charts` (canvas + jsdom) and `matchMedia` — copy the existing test-file patterns. Missing the `matchMedia` stub doesn't just fail Chart's own test: `Chart`/`EquityChart` mount inside `App.tsx`'s single top-level `ErrorBoundary`, so an uncaught `matchMedia` TypeError there blanks the ENTIRE backtest view in any test that reaches the results page — stub it in every test file that renders that far.
- Windows checkout: git prints LF→CRLF warnings on diff; harmless.
- SQLite migrations: ad-hoc guarded ALTER TABLE only (pattern in `storage/db.py`), no framework — keep it that way.
- New indicators/conditions require: schema enum + validators, `indicators/library.py` fn with golden tests, compiler support, recap sentences (+ recap-completeness), translator prompt vocab, eval cases, capabilities entry, regenerate `docs/TA_CAPABILITIES.md`.
- Deterministic vs LLM: rating (`analysis/rating.py`), recap, disclaimer are deterministic by design — do not "improve" them with LLM output.

## Roadmap (ranked)

1. **Grow the TA vocabulary** (standing top priority — "translations always work seamlessly"): 13 indicators / 5 condition types today (added ADX, ROC, day_of_week, streak; plus bidirectional entry_short/exit_short and ATR-multiple take-profit — closed the acid-test gap where a two-sided session-breakout strategy couldn't be translated at all). Still missing: OBV, CCI, Keltner, Ichimoku, pivot points, candle patterns, percent-change-over-N-bars, multi-timeframe conditions. Each addition follows the checklist below.
2. **Realism controls**: surface `commission` (already in `EngineSettings`, default 0) in the setup UI + add slippage/spread modeling — the interpreter's own top caveat is "costs not modeled".
3. **Overfitting defenses**: walk-forward / out-of-sample split validation to complement Monte Carlo — core to the "would I size a real position on this" trust bar.
4. **Multi-asset runs — done**: `POST /api/backtests/batch` runs one strategy across up to 10 symbols, per-symbol failure isolation, comparison view (`BatchResultsPage`), on-demand per-symbol interpretation. Next increment would be persisting a named "batch" concept (today each symbol just lands as its own run, tied together only in the response).
5. **Persist Monte Carlo results + feed them into the interpretation digest** (currently POST-only, ephemeral, invisible to the interpreter).
6. **Grow the translation eval set with the vocabulary** (28 cases now, including the acid-test two-sided breakout and ATR take-profit) and track structural accuracy over time — the ≥90% bar is only meaningful if the set covers the real vocab.
7. **Conversational/partial strategy refinement**: "just tighten the stop to 5%" without full re-translation (deferred plan item; today edit = full re-translate).
8. **Engine-swap spike**: prototype a second `EngineAdapter` implementation (e.g. nautilus-trader/zipline-reloaded) to prove the seam — the AGPL gate blocks any commercialization.
9. **Data resilience**: fallback provider / paid-provider option for the ~730-day hourly limit and yfinance flakiness.
10. **Broaden speech-to-text browser support**: Web Speech API is Chrome/Edge-only; a Groq Whisper-Large-v3-Turbo backend proxy (~$0.0006/min) is the documented cheap paid fallback if Safari/Firefox support or accuracy matters later.
11. Visual/graphical design polish (explicitly lowest priority).
12. Authentication (explicitly lowest priority).
13. User accounts / multi-tenancy (explicitly lowest priority).
14. Hosted deployment (explicitly lowest priority).
