# QuantDinger conversation handoff

This file summarizes the discussion so another Codex session can continue without rereading the whole chat.

## Project

This checkout is QuantDinger `5.4.1`, an open-source, self-hosted trading platform. It includes:

- Python/Flask API and PostgreSQL persistence
- Redis plus background workers for strategies, backtests, schedules, and reconciliation
- Strategy API V2 for Python strategies
- chart indicators and AI research
- signal mode, paper workflows, Quick Trade, and live exchange/broker adapters
- optional Agent Gateway/MCP access
- optional JEV pre-trade entry filter

The frontend is supplied as a prebuilt container image; its source is not in this checkout.

## Local runtime status

The stack was started with:

```bash
docker compose -f docker-compose.ghcr.yml up -d --quiet-pull frontend trading-worker scheduler-worker celery-worker celery-beat
```

Current local URLs:

- Web: <http://127.0.0.1:8888>
- API health: <http://127.0.0.1:5000/api/health>

The database, backend, frontend, Redis services, and workers were verified healthy. The health endpoint returned HTTP 200.

Local configuration files were generated and are intentionally uncommitted:

- `.env` — Compose settings and generated infrastructure secrets
- `backend.env` — backend settings, generated admin credentials, and API-provider settings

Never print or commit either file. The local admin username is `mizan`. If the password is needed, read it locally with:

```bash
sed -n 's/^ADMIN_PASSWORD=//p' backend.env
```

Live trading is not configured. The database currently has zero saved exchange credentials.

## Token issue

The UI showed `Token invalid or expired`.

Relevant implementation:

- JWTs are created in `backend_api_python/app/utils/auth.py`.
- Tokens expire after seven days.
- Every login increments the user `token_version`; this invalidates earlier tokens.
- The middleware returns the same generic 401 message for expiry, invalid signatures, and token-version mismatch.

Observed behavior: a login from Safari was followed by 401 requests from the Codex browser, then a later successful login. The likely cause was logging in from a second browser, which replaced the first token. Sign in again and continue in one browser tab.

## SuperTrend chart in the conversation

The screenshot showed `[Sample] SuperTrend Trend-Following` on `BTC/USDT · SPOT`, with green `L` and red `S` markers.

The sample indicator defaults to:

- ATR period: `10` candles
- multiplier: `3.0`

Calculation:

1. True range is `max(high-low, abs(high-previous_close), abs(low-previous_close))`.
2. ATR uses Wilder-style smoothing: `ewm(alpha=1 / period, adjust=False)`.
3. Basic bands are `HL2 + multiplier * ATR` and `HL2 - multiplier * ATR`, where `HL2 = (high + low) / 2`.
4. Final bands are carried forward and only tighten in the current trend direction.
5. A close above the previous final upper band changes direction to bullish (`1`). A close below the previous final lower band changes direction to bearish (`-1`).
6. `L` is emitted only on a `-1 -> 1` flip. `S` is emitted only on a `1 -> -1` flip.

Source: `backend_api_python/app/services/builtin_indicators.py`, around `_SUPERTREND_CODE`.

Important boundary: the indicator's `signals` array is chart-only. It does not place orders. On spot, the normal interpretation is `L = open/increase a long` and `S = close a long`; an `S` marker does not automatically open a short.

## Strategy implementation caveat

The executable Strategy API has a canonical SuperTrend series:

```python
trend = indicator(
    "supertrend",
    g.symbol,
    period=10,
    multiplier=3.0,
    output="direction",
    frequency="1h",
)
```

Use completed bars and compare the previous/current direction after dropping warmup `NaN` values. The runtime implementation is in `backend_api_python/app/services/strategy_v2/runtime.py`; the factor implementation is in `backend_api_python/app/services/factors/registry.py`.

The chart sample and executable factor seed ATR and initial direction differently. Before automating the chart, compare their flip timestamps on the same BTC/USDT data. Do not assume equal parameters produce identical trades.

## How trades are implemented

There are three distinct paths:

1. **Indicator:** visual lines and L/S markers only.
2. **Strategy API V2:** a Python source declares its universe, timeframe, signals, sizing, exits, protections, and direction mode. Verify it, backtest it, create a stopped deployment, and use `signal` mode first.
3. **Quick Trade:** a human can submit a manual order after adding an exchange credential and selecting the exact market/product. Live execution uses the pending-order queue, exchange adapter, reconciliation, position ownership, and risk limits.

For a BTC spot SuperTrend strategy, the first executable version should be long-only:

- bullish flip: `open_long` or a bounded target allocation
- bearish flip: `close_long` / target allocation `0`
- no short orders on spot
- explicit quantity/notional cap, stop-loss, and maximum drawdown policy

For actual shorts, the product must be a supported swap/derivative account and the strategy must declare an explicit `direction_mode` plus independent short entry/exit and risk rules. Do not convert a bearish long exit into a short automatically.

## JEV decision filter

The screenshot's AI Decision Filter switch was off.

When enabled for a regular live strategy or Quick Trade, JEV checks prepared context such as signal alignment, market regime, account risk, and execution quality before an entry. Exits, stop-loss, take-profit, and emergency reductions bypass it.

JEV is not the trading strategy and is not a guaranteed safety stop. If JEV and the fallback LLM are unavailable, the current implementation fails open and records the provider failure. Configuration keys are `JEV_API_KEY`, `JEV_BASE_URL`, `JEV_MODEL`, `JEV_TIMEOUT_SECONDS`, and `JEV_MIN_CONFIDENCE`.

## Safe continuation plan

1. Fix the browser session by logging in once and staying in one tab.
2. Inspect the exact BTC/USDT timeframe and compare chart-sample versus Strategy API SuperTrend flips.
3. Write a long-only Strategy API V2 conversion.
4. Verify and backtest with fees, slippage, warmup, drawdown, and execution ledger review.
5. Run signal mode and inspect notifications/state restoration.
6. Only after that, add a restricted exchange API credential with withdrawals disabled and consider a small paper/live test.

Do not place a real order from this handoff. A live trade requires an explicit instrument, venue, account credential, order size, and user confirmation at the final order action.

## Useful files

- `README.md` — project overview and deployment
- `docs/getting-started/FIRST_RUN.md` — first validated workflow
- `docs/product/STRATEGY_WORKFLOW.md` — idea to deployment lifecycle
- `docs/product/BACKTEST_CENTER.md` — how to read backtest results
- `docs/trading/INDICATOR_DEV_GUIDE.md` — indicator versus strategy boundary
- `docs/trading/STRATEGY_DEV_GUIDE.md` — Strategy API V2 contract and live boundaries
- `docs/trading/LIVE_TRADING_SAFETY.md` — preflight and incident checklist
- `backend_api_python/app/services/builtin_indicators.py` — sample SuperTrend code
- `backend_api_python/app/services/ai_decision_filter.py` — JEV/LLM entry filter
- `backend_api_python/app/services/strategy_v2/live_execution.py` — strategy order submission path

## Suggested opening prompt for the next Codex session

> Read `README_HANDOFF.md`. Keep the local stack running. First compare the sample SuperTrend and Strategy API SuperTrend on BTC/USDT for the chart timeframe, then propose a long-only spot strategy conversion and a backtest plan. Do not add exchange credentials or place live orders.
