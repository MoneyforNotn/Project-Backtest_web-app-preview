# AI Strategy Backtesting Platform

Describe a trading strategy in plain English, get a trustworthy backtest: faithful schema translation with a confirm-before-run recap, engine-computed metrics passed through unmodified, an interactive chart of trades and indicators, and an honest AI interpretation.

## Layout

- `backend/` — FastAPI + backtest engine (Python 3.11+, uv)
- `frontend/` — React SPA (Vite, TypeScript, lightweight-charts)

## Prerequisites

- Python ≥ 3.11 with [uv](https://docs.astral.sh/uv/)
- Node.js ≥ 20 with npm
- An Anthropic API key (translation + interpretation)

## Setup

```bash
# 1. API key
cp .env.example backend/.env    # then paste your real ANTHROPIC_API_KEY

# 2. Backend
cd backend
uv sync
uv run uvicorn app.main:app --reload       # serves http://localhost:8000

# 3. Frontend (separate terminal)
cd frontend
npm install
npm run dev                                 # serves http://localhost:5173
```

Open http://localhost:5173, describe a strategy, confirm the recap, review the results.

## Tests

```bash
cd backend
uv run pytest              # offline suite (providers + LLM mocked)
uv run pytest -m live      # live data smokes (real yfinance/ccxt fetches)
uv run pytest -m eval      # translation eval harness (live API, costs cents)
uv run pytest tests/e2e    # full translate->run->get loop (mocked LLM/data)

cd frontend
npm test                   # vitest
npm run build              # type-check + production build
```

The translation eval must score ≥ 90% structural accuracy on the labeled case set (`backend/tests/eval/translation_cases.json`).

## How trust is enforced

- The recap you confirm is rendered deterministically from the parsed strategy schema — the LLM cannot describe one thing and encode another.
- Metrics are the engine's own stats passed through field-for-field; the platform adds zero arithmetic.
- A strategy with unresolved clarification questions cannot execute.
- The AI disclaimer is a constant attached in code, never model-generated.

## License notice

The backtest engine dependency ([backtesting.py](https://github.com/kernc/backtesting.py)) is **AGPL-3.0**. It is acceptable for private personal use, but a commercial network service linking it must offer its backend source. The engine is isolated behind `backend/app/engine/adapter.py`; only `backend/app/engine/backtesting_py.py` may import it. **Replacing the engine (or complying with AGPL) is a hard prerequisite before any commercial or hosted use.**

Free data sources: yfinance (equities/ETFs, forex) and ccxt public endpoints (crypto). Hourly/4h history reaches back roughly 730 days on free sources; daily/weekly is unrestricted.
