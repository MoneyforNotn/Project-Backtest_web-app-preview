# Alpha Vantage data-provider assessment

Date: 2026-07-23

## Decision

Do not add Alpha Vantage as a free fallback. Keep it as the leading paid-history
candidate for a dedicated equity-history provider once a premium key is
available for a pilot.

ETFs and cash indexes do not require a new asset class. Both already travel
through the existing `equity` provider and engine path:

- ETFs use their normal exchange tickers, such as `SPY`, `QQQ`, and `TLT`.
- yfinance cash indexes use symbols such as `^GSPC`, `^IXIC`, and `^VIX`.
- Cash-index runs are benchmark simulations, not directly tradable instruments.
  A corresponding ETF is the better choice when the intent is tradable exposure.

The symbol picker now exposes Stocks, ETFs, and Indexes separately while keeping
their common backend asset class stable.

## What Alpha Vantage would add

Alpha Vantage documents 20+ years of stock/ETF intraday OHLCV at 1, 5, 15, 30,
and 60 minute intervals. Historical intraday data is requested one calendar
month at a time with `month=YYYY-MM`; without a month, a full response covers
only the trailing 30 days.

That would directly address the current yfinance limits of roughly 730 days for
hourly bars and 60 days for 15/30-minute bars. It would be most useful as an
older-history source for stocks and ETFs, not as a routine fallback for recent
requests.

## Cost and practical constraints

- Historical intraday is a premium endpoint.
- The free allowance is 25 requests per day, so a free key does not solve this
  product's lower-timeframe history goal.
- The entry premium plan is currently $49.99/month for 75 requests/minute, with
  no daily request limit and premium endpoints unlocked.
- A ten-year request needs about 120 monthly API calls per symbol before cache
  reuse. Local month-level caching is therefore essential.
- Historical index data is also premium and requires the provider's entitlement
  setup. Index symbol conventions differ from yfinance and need an explicit map.

Current official references:

- <https://www.alphavantage.co/documentation/#intraday>
- <https://www.alphavantage.co/premium/>

## Integration shape for a pilot

1. Add Alpha Vantage as an optional, equity-only history provider enabled by
   `ALPHAVANTAGE_API_KEY`.
2. Route only requests older than the free-provider window to it; do not place it
   behind yfinance as a blind fallback.
3. Cache immutable monthly responses independently, then assemble and clip the
   requested range locally.
4. Request adjusted data and regular market hours by default so split history and
   session semantics remain explicit and consistent.
5. Replace the global intraday lookback rejection with provider-capability-aware
   validation. The current guard intentionally rejects old requests before any
   provider call, so merely adding another fallback would not extend coverage.
6. Add golden provider tests for timezone conversion, month boundaries,
   adjustment behavior, rate-limit/error payloads, and index-symbol mapping.
7. Surface the active source and coverage in the confirmation/setup flow before
   letting a paid provider silently change the data contract.

The pilot should validate several representative stocks and ETFs against the
existing source around splits, dividends, daylight-saving transitions, and
regular-session boundaries before the longer window is exposed to users.
