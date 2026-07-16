<!-- GENERATED FILE — do not edit by hand.
     Regenerate with: uv run python scripts/generate_capabilities_doc.py
     (run from backend/). A pytest freshness check fails if this file
     drifts from the live /api/capabilities vocabulary. -->

# TA capabilities

The complete, currently-supported technical-analysis vocabulary — the same set the strategy translator is told about. If it isn't listed here, the translator cannot encode it and will ask for clarification instead of guessing.

## Indicators

| name | description | default period | components |
|---|---|---|---|
| `sma` | Simple moving average of the close price | 20 | — |
| `ema` | Exponential moving average of the close price | 20 | — |
| `rsi` | Relative strength index (Wilder), 0-100 | 14 | — |
| `macd` | MACD of the close price (fast/slow/signal EMAs) | 12 | line, signal, histogram |
| `bollinger` | Bollinger Bands (moving average +/- standard deviations) | 20 | upper, middle, lower |
| `atr` | Average true range (Wilder) — volatility measure | 14 | — |
| `stochastic` | Fast stochastic oscillator, 0-100 | 14 | k, d |
| `volume_sma` | Simple moving average of volume | 20 | — |
| `vwap` | Volume-weighted average price, anchored to each UTC trading day (intraday only) | — | — |
| `donchian` | Donchian Channel: highest high / lowest low over the prior N bars (breakout channel) | 20 | upper, middle, lower |
| `session_level` | High/low accumulated during one trading session per day, held forward after the session ends (intraday only) | — | high, low |
| `adx` | Wilder Average Directional Index — trend strength (0-100) plus +DI/-DI components | 14 | adx, plus_di, minus_di |
| `roc` | Rate of change — percent change in close over the prior N bars | 10 | — |

## Operands

| kind | description |
|---|---|
| `indicator` | One line of a supported indicator |
| `price` | The bar's close price |
| `volume` | The bar's raw volume |
| `value` | A constant number |

## Conditions

| type | description | example |
|---|---|---|
| `crossover` | One series crosses above or below another | the 20 day moving average crosses above the 50 day |
| `comparison` | A series is above/below a value or another series | RSI is below 30 while price is above the 200 day average |
| `session` | The bar falls within (or outside) a fixed-UTC-hours trading session | only enter during the New York session |
| `day_of_week` | The bar's UTC date falls on (or outside) one or more weekdays | only trade on Mondays and Tuesdays |
| `streak` | Each of the last N bars closed higher (or lower) than the one before it | after three consecutive lower closes |

Grouping: Conditions combine with ALL (and) or ANY (or); entries can be long, short, or both (long on one trigger, short on the opposite trigger)

## Risk rules

| rule | forms |
|---|---|
| `stop_loss` | percent below/above entry, ATR multiple |
| `take_profit` | percent target, ATR multiple |
| `trailing_stop` | percent from peak, ATR multiple from peak, indicator level (chandelier-style, chases a moving indicator bar-by-bar) |
| `position_sizing` | fraction of capital per trade (0-100%) |
| `risk_short` | a full risk block overriding stop/target/trailing/sizing for the short side of a two-sided strategy only |
| `max_open_positions` | opt-in pyramiding: 1-10 same-direction positions open at once (default 1) |

## Trading sessions (fixed UTC hours)

| name | UTC hours |
|---|---|
| `asian` | 00:00-09:00 |
| `london` | 08:00-17:00 |
| `new_york` | 13:00-22:00 |

## Markets

| asset class | label | provider | examples |
|---|---|---|---|
| `equity` | Equities / ETFs | yfinance | AAPL, MSFT, NVDA, SPY, QQQ |
| `crypto` | Crypto | ccxt (Binance public endpoints) | BTC/USDT, ETH/USDT, SOL/USDT |
| `forex` | Forex | yfinance | EURUSD=X, GBPUSD=X, USDJPY=X |

## Timeframes

| id | label | max lookback (days) |
|---|---|---|
| `1d` | Daily | unlimited |
| `1w` | Weekly | unlimited |
| `4h` | 4 hour | 730 |
| `1h` | Hourly | 730 |
