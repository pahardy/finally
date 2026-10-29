# Market Simulator

How FinAlly simulates stock prices when `MASSIVE_API_KEY` is not set. Implemented in `backend/app/market/simulator.py` and `seed_prices.py`. Supersedes `archive/MARKET_SIMULATOR.md`.

Goals, in priority order: believable (prices wander like stocks, tech moves together), lively (visible moves every tick plus occasional drama), cheap (in-process, no dependencies beyond numpy), deterministic enough to test.

## 1. Approach

Geometric Brownian Motion (GBM) per ticker, with correlated shocks across tickers and a small chance of a sudden jump.

```
S(t+dt) = S(t) * exp( (mu - sigma^2/2) * dt  +  sigma * sqrt(dt) * Z )
```

| Symbol | Meaning |
|---|---|
| `S` | price; GBM keeps it positive and makes moves proportional to price |
| `mu` | annualized drift (expected return), 3-8% per ticker |
| `sigma` | annualized volatility, 17% (V) to 50% (TSLA) |
| `dt` | tick length as a fraction of a trading year |
| `Z` | standard normal draw, **correlated across tickers** (section 3) |

**Time step.** One tick is 500ms. A trading year is `252 * 6.5 * 3600 = 5,896,800` seconds, so `dt = 0.5 / 5,896,800 = 8.48e-8`. For AAPL (`sigma = 0.22`) the per-tick standard deviation is `0.22 * sqrt(dt) = 0.0064%`, about 1.2 cents on $190: smooth, never jumpy.

**Pace.** The simulator ticks around the clock (PLAN: no market hours), but `dt` is calibrated to *trading* seconds, so it covers about 2.7% of daily volatility per wall-clock day versus about 1.4% for a real session. That roughly doubled speed is intentional: it keeps the demo lively, and no parameter needs to change.

## 2. Seed Data and Parameters (`seed_prices.py`)

| Constant | Content |
|---|---|
| `SEED_PRICES` | Starting price for the 10 default tickers (AAPL 190, GOOGL 175, MSFT 420, AMZN 185, TSLA 250, NVDA 800, META 500, JPM 195, V 280, NFLX 600) |
| `TICKER_PARAMS` | `{"sigma", "mu"}` per default ticker. TSLA 0.50, NVDA 0.40, NFLX 0.35 are the most volatile; V 0.17, JPM 0.18 the calmest |
| `DEFAULT_PARAMS` | `sigma 0.25, mu 0.05` for any ticker not listed |
| `CORRELATION_GROUPS` | `tech` = AAPL GOOGL MSFT AMZN META NVDA NFLX; `finance` = JPM V |
| Correlation constants | `INTRA_TECH_CORR 0.6`, `INTRA_FINANCE_CORR 0.5`, `CROSS_GROUP_CORR 0.3`, `TSLA_CORR 0.3` |

**Dynamically added tickers.** The simulator accepts any symbol matching `^[A-Z]{1,5}$` (anything else raises `UnknownTickerError`, per `MARKET_INTERFACE.md`). An unlisted symbol is seeded at `random.uniform(50, 300)` with `DEFAULT_PARAMS` and correlation 0.3 against everything. Its start price is fixed for the life of the process: the cache's `open_price` records it, which is what "daily change %" is measured against (PLAN section 6).

## 3. Correlated Moves

Independent noise would make every stock wander alone. Instead, each tick:

1. Draw `z` = `n` independent standard normals (`np.random.standard_normal(n)`).
2. Multiply by the Cholesky factor `L` of the correlation matrix `C` (`C = L L^T`), giving correlated draws `L @ z`.
3. Use element `i` as `Z` for ticker `i` in the GBM formula.

`C` is built pairwise by `_pairwise_correlation(t1, t2)`:

| Pair | rho |
|---|---|
| Either ticker is TSLA | 0.3 |
| Both in `tech` | 0.6 |
| Both in `finance` | 0.5 |
| Anything else (cross-sector, unknown) | 0.3 |

`L` is rebuilt whenever a ticker is added or removed: O(n^2) to build and O(n^3) to factor, negligible for n under 50. The matrix must stay positive definite; the shipped values are, but `np.linalg.cholesky` raises `LinAlgError` if a future edit makes it otherwise (for example, raising `INTRA_TECH_CORR` toward 1 while `CROSS_GROUP_CORR` stays low). A test must cover the shipped matrix at the maximum realistic ticker count.

## 4. Random Events

After each ticker's GBM step, with probability `event_probability = 0.001` (per ticker, per tick) the price is multiplied by `1 +/- U(0.02, 0.05)`: a sudden 2-5% move in a random direction. With 10 tickers at 2 ticks/sec that is about one event every 50 seconds somewhere on the board, enough drama to make the heatmap and P&L chart interesting without making prices unusable. Events apply to the internal unrounded price, so they persist.

## 5. Code Structure

```
backend/app/market/
  seed_prices.py   constants only (section 2)
  simulator.py
    GBMSimulator          pure math, synchronous, no I/O
      __init__(tickers, dt, event_probability)
      step() -> {ticker: price}          advance one tick, prices rounded to 2 dp
      add_ticker / remove_ticker         rebuild Cholesky
      get_price / get_tickers
      _add_ticker_internal               seed price + params
      _rebuild_cholesky                  build C, factor
      _pairwise_correlation (static)
    SimulatorDataSource(MarketDataSource) asyncio wrapper
      start(tickers)        build GBMSimulator, seed cache, spawn _run_loop task
      _run_loop             every update_interval (0.5s): step() then cache.update() per ticker
      add_ticker            sim.add_ticker, then seed cache immediately
      remove_ticker         sim.remove_ticker, cache.remove
      stop                  cancel task, await it, tolerate repeat calls
```

Separation of concerns: `GBMSimulator` knows nothing about asyncio or the cache, so it is unit-testable by calling `step()` in a loop. `SimulatorDataSource` knows nothing about the math; it only moves numbers from `step()` into `PriceCache`. `_run_loop` catches and logs per-step exceptions so one bad tick never kills the feed.

## 6. Behavior Guarantees

- Prices are always positive and rounded to 2 decimals in the cache; the internal state is unrounded so tiny moves still accumulate.
- Every tracked ticker gets a price at `start()` and at `add_ticker()` before either returns, so the first SSE event and the first trade never see a missing price.
- `previous_price` in the cache is the prior tick, so most ticks flash up or down, and a ticker is "flat" only when a move rounds to zero cents.
- `remove_ticker` removes the cache entry; re-adding a ticker later starts from a **new** seed price (default tickers: the original seed price; others: a new random price).

## 7. Testing

Current suite: `tests/market/test_simulator.py` (17) and `test_simulator_source.py` (10). Additions:

- **Seeding.** The simulator uses the global `random` and `np.random`. Tests that assert exact sequences must call `random.seed(n)` and `np.random.seed(n)` first; otherwise assert statistical properties only.
- **Statistics (loose bounds).** After 10,000 ticks of one ticker at known sigma, the std-dev of log returns is within 10% of `sigma * sqrt(dt)`; sample correlation of two tech tickers over 20,000 ticks is within 0.1 of 0.6.
- **Events.** `event_probability=1.0` moves every price by 2-5%; `0.0` produces none.
- **Validation.** Invalid symbols (`"aapl"`, `"TOOLONG"`, `"A1"`) raise `UnknownTickerError`.
- **Positive definiteness.** Build the matrix for all default tickers plus 20 unknown ones and assert `cholesky` succeeds.
- **Demo.** `uv run market_data_demo.py` in `backend/` shows a live terminal dashboard for eyeballing realism.

## 8. Tuning Cheat Sheet

| Want | Change |
|---|---|
| Calmer or wilder overall | Scale all `sigma` in `TICKER_PARAMS` |
| Slower or faster pace | `update_interval` (keep `dt` consistent: `dt = interval / 5,896,800`) |
| More or fewer jumps | `event_probability` |
| Sectors move more in lockstep | Raise `INTRA_*_CORR` (check positive definiteness) |
| New default ticker | Add to `SEED_PRICES` and `TICKER_PARAMS`, and to a `CORRELATION_GROUPS` set if sector-correlated |
