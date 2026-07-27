# Landing page — backend/product scope (not frontend design)

Everything below is deliberately **not** part of the frontend design pass on the new marketing landing page (`/`) and `/prompt` archive. Listed here so it doesn't get silently assumed done.

## 1. Waitlist capture (the hero's primary CTA)

The landing page's main CTA collects an email into a waitlist — nothing else exists to send it to yet.

- New endpoint: `POST /api/waitlist` — body `{email: str, source?: str}`.
- New SQLite table `waitlist_signups` (id, email UNIQUE, created_at, source), same guarded-`ALTER TABLE`/`CREATE TABLE IF NOT EXISTS` convention as the rest of `storage/db.py`.
- Resubmitting an already-registered email should be a quiet 200 (idempotent), not a 409 — a landing page shouldn't surface backend uniqueness errors to a visitor.
- No confirmation/welcome email is implied by "capture" — decide separately if you want transactional email at all.
- Needs basic abuse protection before a public deploy (honeypot field is enough to start; this app has never had a publicly reachable write endpoint before).

## 2. Pricing tiers

The landing page's pricing table is **illustrative only** per your call — no real tier names, limits, or prices are final. Before they can be real:

- Decide actual plan names/limits (backtests per month? symbols? API access?).
- Payment processor integration (Stripe or similar) — none exists.
- Tier-gating logic in the backend (today every capability is unconditionally available — there is no concept of a gated feature anywhere in `backend/app/`).
- Subscription lifecycle (upgrade/downgrade/cancel, dunning) — none exists.
- This depends on accounts/auth existing first (see #3) — you can't gate a tier for an anonymous local user.

## 3. Accounts / auth

Unchanged from the existing roadmap (`CLAUDE.md` tier B/C: explicitly lowest priority). The landing page and its waitlist capture do not require this — flagging only so pricing (#2) doesn't get assumed as unblocked.

## 4. Legal / compliance copy

A public page describing an AI-generated "interpretation" of a trading strategy's performance likely needs a visible disclaimer (not financial advice, past performance ≠ future results) beyond what the in-app interpreter already carries (`interpreter.py`'s constant disclaimer, shown only after a run, inside the app). Nothing has been added to the marketing page for this — recommend a compliance/legal pass before public launch, not something I'm positioned to write myself.

## 5. Public deployment target

Today the entire product is local-only, no-auth, single-user by design (`CLAUDE.md`). A live waitlist endpoint is the **first** thing this product would ever expose publicly. Needs its own decision, independent of the existing tier B/C roadmap:

- Static hosting for the landing page SPA.
- Where `POST /api/waitlist` actually runs (a minimal serverless function, or a narrowly-scoped deployment of the existing FastAPI app with everything else firewalled off — the rest of the backend should almost certainly **not** go public alongside it, since it has no auth).

## 6. Analytics / conversion tracking

Not wired up anywhere. Decide if you want basic, privacy-respecting analytics (e.g. Plausible) on the public landing page before launch — currently zero visibility into traffic or conversion.

---

Not listed here because they're part of the frontend build itself, not backend scope: SEO meta tags, sitemap/robots.txt, page performance budget (sub-2s load), accessibility — those are covered directly in the landing page implementation.

---

## Unrelated product note (not landing-page scope, logged here per request 2026-07-27)

Investigated a reported bug: a strategy's "Backtested markets" list showed several AAPL/1d rows with identical return/trade-count numbers. **Confirmed not a bug** — 8 distinct DB rows, timestamps 70s–5min apart, config changing (start date) and reverting mid-sequence: genuine repeated manual runs. The engine is deterministic, so identical config always produces identical numbers; that's correct behavior, not duplication.

One real UX improvement fell out of the investigation, not urgent, worth doing whenever the strategy-detail view gets touched: `get_strategy()`/`list_strategies()` (`storage/db.py`) could collapse consecutive runs that share an identical `evidence_key` (symbol/asset_class/timeframe/start/end/cash/commission/slippage — already computed in `_strategy_robustness`) into one "ran N× with these results" row in the Backtested markets table, instead of listing every literal DB row. Purely a display grouping — no schema or engine change.

---

## 7. What still doesn't work or look its best (2026-07-27)

Requested directly: real gaps in features / engine reliability / backtest functionality / TA coverage, independent of the visual redesign. Grouped by what kind of work each is, roughly ordered by leverage (biggest "simple, understandable, inviting, easy to use" payoff first).

### Onboarding / first-run experience — the single biggest simplicity gap
A brand-new visitor lands on a blank describe box with no strategy in mind. The 7 templates exist (`templates/library.py`) but live one click away on the describe page, easy to miss. There's no guided "try this first" path, no example prompts inline in the textbox placeholder rotation, no "what makes a good description" micro-example next to the input. For a product whose whole pitch is "describe a strategy in plain English," the blank-page problem is the first thing a new user hits, before any TA vocabulary or engine quality matters at all. Cheapest win in this whole list: 3-4 rotating example prompts as clickable chips right above/below the describe textarea (not buried in a template library), so a first-time user never faces true blank-page paralysis.

### Clarification loop friction
`RecapCard`'s clarification Q&A is solid mechanically (structured options, free text, blocks acceptance until resolved) but multi-round clarification (ask → answer → ask again) has no visible progress signal — a user answering a 3rd round of questions has no sense of whether they're almost done or stuck in a loop. A simple "question 2 of ~3" or even just a visible history of prior Q&A pairs (currently: does the old Q&A stay visible, or does it collapse away once answered?) would reduce the "is this ever going to finish" anxiety a multi-turn clarification creates. Worth an explicit UX check next design pass, not just a code check.

### Translation reliability — the ladder works, but its failure mode is invisible
The three-rung cost ladder (Haiku → Sonnet → Opus) is architecturally solid, but from the user's side, a translation that escalates through all three rungs and still fails just... fails, with "deterministic retry guidance." There's no visible signal to the user about *why* it struggled (ambiguous wording? unsupported indicator? conflicting conditions?) beyond generic retry copy. `GET /api/translation-issues/report`'s n-gram ranking exists for operators on the dashboard — none of that diagnostic value reaches the end user in the moment they hit a failure. A future pass could surface the *specific* clarification-shaped reason a parse failed, not just "please rephrase."

### Engine reliability — two known-incident classes worth a standing regression checklist, not just the one fix each got
Both the pyramiding re-entry bug and the interpretation-sharing `skipped`-flag bug (`CLAUDE.md` Known Incidents) share a root cause shape: **a rewrite of a stateful per-bar/per-run loop was correct for every existing test fixture and wrong for an uncovered edge shape that only showed up against real accumulated data.** That's a process gap, not just two fixed bugs — every future engine change to `Runner.next()` or any per-trade state machine should get a deliberately adversarial fixture (overlapping entry/exit conditions, same-bar multi-signal conflicts, pyramided positions closing and reopening same bar) as a standing checklist item before merge, not discovered after the fact by a review pass. Worth codifying as an actual checklist in `engine/backtesting_py.py`'s test file header, not just tribal knowledge in `CLAUDE.md`.

### TA vocabulary — real, named gaps (already flagged as deliberately deferred, listed here for prioritization)
From the roadmap's own deferred list: OBV, CCI, further single-bar candle patterns beyond engulfing (doji/hammer/pin bar — currently the translator is instructed to raise a clarification rather than guess, which is correct but means a fairly common request like "buy on a hammer at support" always stalls into a clarification today). General multi-timeframe conditions (e.g. "daily trend filter on an hourly entry") remain unsupported outside the fixed `4h` resample — this is probably the single most-requested-sounding capability for anyone with real technical-analysis background, worth moving up from "deferred" to "next" if TA vocabulary continues as the standing top priority. Economic calendar / earnings-date awareness ("don't enter within 2 days of earnings") is a very commonly desired real-world filter with zero support today.

### Backtest functionality — realism/robustness gaps beyond what's already "done"
Liquidity-aware fills (flagged as the one remaining realism gap after commission+slippage shipped) — no volume-based fill constraints or partial-fill modeling, meaning a strategy that would move the market on illiquid symbols backtests as if it never would. A true blind-holdout mode (lock the strategy, reveal a final untested slice of history) is listed as a "possible later layer" for overfitting defense — this is a materially different and stronger form of validation than the subperiod-consistency check that shipped, worth promoting once vocabulary growth slows down. Portfolio backtesting has no rebalancing-frequency concept — `ideal_weights` is a static point-in-time suggestion, not a rule that could itself be backtested (buy-and-hold-the-weights vs. periodic-rebalance would show materially different results for correlated strategies).

### Small trust/legibility gaps in already-shipped features
`RatingBadge`'s composite score (Sharpe/Sortino/Calmar/drawdown/profit-factor weighted blend) is genuinely sophisticated but entirely opaque to a first-time user beyond the info popover — a first-time user has no intuition for what a "B+" means in absolute terms (is a B+ a strategy worth trading? worth ignoring?). Similarly Monte Carlo and subperiod consistency are both real, valuable robustness signals that currently read as "more numbers" rather than a clear verdict — worth an eventual single rolled-up "how much should I trust this" indicator that surfaces the rating/Monte Carlo/consistency/live-demo divergence as one legible signal rather than four separate panels a user has to synthesize themselves. This is squarely a design-pass item (tier B), flagging here because it's also a functionality clarity gap, not just visual polish.
