# Engine-swap spike — proving the EngineAdapter seam (2026-07-19)

## Why

`backtesting.py` is AGPL-3.0 (KTD-1). Replacing it is a hard prerequisite to
any commercial use. The risk to retire was not "can we find another engine" —
it was "is our `EngineAdapter` seam actually engine-agnostic, or does the rest
of the platform silently depend on backtesting.py behaviors?"

## What was built

- `app/engine/simple_engine.py::SimpleEngineAdapter` — a clean-room engine
  (pandas/numpy only, zero AGPL imports) implementing the adapter protocol for
  a bounded subset: long-only, single position, whole-share equities,
  next-bar-open fills, percent/ATR stop and take-profit anchored to the
  signal-bar close, contingent intra-bar exits, exit-rule closes,
  per-side commission+slippage. Everything outside the subset raises a typed
  `EngineError` naming the spike.
- `app/api/execution.py::create_adapter()` — engine selection is now a config
  knob (`engine_backend: backtesting | simple`); routes/demo/portfolio all go
  through the factory.
- `tests/engine/test_simple_engine_parity.py` — both engines run the SAME
  compiled strategy on the SAME bars; trades (bars, sizes, prices) and the
  FULL equity curve must match (rtol 1e-9; commission scenario at 1e-6).

## Result: the seam holds

7/7 parity scenarios pass, including a same-bar stop-out-then-re-enter shape
and a commission run. Nothing outside `engine/backtesting_py.py` needed to
change to run a second engine end-to-end — `compile_strategy` output
(boolean signal series + risk block + ATR series) turned out to be a fully
engine-neutral contract, which is the thing this spike set out to prove.

## Conventions discovered empirically (now encoded in the spike + tests)

1. **Finalized trades close at the last bar's OPEN**, not its close.
   `finalize_trades=True` pushes the close-out order through normal order
   processing on the final bar, and market orders fill at that bar's open.
2. **Commission is a cash fee, not a price adjustment**: trades record RAW
   fill prices; the per-side cost is charged to cash at entry and exit.
   Affordability/sizing, however, IS judged against the cost-adjusted price.
3. **A stop-out does not block same-bar re-entry** (position is already gone
   when signals are evaluated on the close); only a signal-exit does. This is
   the same shape as the 2026-07 pyramiding incident — the parity suite now
   covers it across engines.

## What the spike deliberately does NOT cover

Shorts/two-sided, pyramiding, take-profit tiers, trailing stops (incl.
indicator-level), fractional crypto/forex sizing, and the ratio statistics
(Sharpe/Sortino). These are all mechanical extensions of the same loop —
none of them looked seam-threatening — but they are real work (est. the
trailing/tier logic is the bulk of `backtesting_py.py`'s complexity).

## Third-party engine assessment (for the real swap, later)

- **nautilus-trader**: production-grade, Rust core, permissive license
  (LGPL-3.0 — acceptable for commercial use as a dynamically-linked dep).
  Heavy install and an event-driven API (venues, instruments, data engine)
  that would make the adapter substantially thicker than `_make_runner`.
  Best candidate if live/broker execution is the endgame (roadmap: demo
  tracking is the bridge to broker connect — aligns).
- **zipline-reloaded**: Apache-2.0, but bundle-based data ingestion fights
  our provider/cache design, and Windows support is historically fragile.
  Poor fit.
- **Grow SimpleEngine**: MIT-clean, already parity-tested against the
  incumbent, zero new dependencies. The realistic cheapest path to shedding
  AGPL for the CURRENT feature set; the parity suite doubles as the
  acceptance harness while extending it convention by convention.

**Recommendation:** when commercialization forces the swap, extend
SimpleEngine behind the same parity suite (each convention ported = one new
parity test), and reevaluate nautilus-trader only when broker-connect work
starts — its event model is the right shape for live execution, which the
per-bar spike loop is not.
