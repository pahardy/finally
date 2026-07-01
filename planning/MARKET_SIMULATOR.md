# Market Simulator: GBM Approach & Code Structure

Documents the as-built default price source — `backend/app/market/simulator.py` and
`seed_prices.py` — used when `MASSIVE_API_KEY` is not set. Supersedes the
pre-implementation draft at `planning/archive/MARKET_SIMULATOR.md`.

## Why GBM

Geometric Brownian Motion is the standard textbook model for a stock price: log-returns
are normally distributed, prices stay positive, and volatility scales with price level.
It needs only two parameters per ticker (drift `mu`, volatility `sigma`) and is cheap to
step every 500ms for a whole watchlist.

## The Formula

```
S(t+dt) = S(t) * exp((mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z)
```

- `S(t)` — current price
- `mu` — annualized drift (expected return)
- `sigma` — annualized volatility
- `dt` — time step, as a fraction of a trading year
- `Z` — a (correlated) standard normal draw

## Deriving `dt` from the 500ms Tick

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR  # ~8.48e-8
```

252 trading days/year, 6.5 trading hours/day. A 500ms tick is therefore an extremely
small slice of a trading year, which is intentional: it produces sub-cent moves per tick
that accumulate into realistic-looking intraday paths rather than jumping around wildly
every half second.

## Correlated Moves via Cholesky Decomposition

Real markets move together — tech stocks drift together, financials drift together. The
simulator reproduces this by drawing correlated normal variables instead of independent
ones:

1. Build an `n x n` correlation matrix where `n` = number of tracked tickers, using
   sector-based pairwise correlations.
2. Compute its Cholesky decomposition `L` (`numpy.linalg.cholesky`).
3. Each tick, draw `n` independent standard normals `Z_indep`, then `Z_correlated = L @ Z_indep`.
4. Feed `Z_correlated[i]` into ticker `i`'s GBM step.

```python
def step(self) -> dict[str, float]:
    z_independent = np.random.standard_normal(n)
    z_correlated = self._cholesky @ z_independent if self._cholesky is not None else z_independent
    ...
```

The correlation matrix is rebuilt (`_rebuild_cholesky`) whenever a ticker is added or
removed — O(n^2), fine for the small watchlists this project targets (n < 50).

### Sector Correlation Groups (`seed_prices.py`)

```python
CORRELATION_GROUPS = {
    "tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}
INTRA_TECH_CORR = 0.6      # tech stocks move together
INTRA_FINANCE_CORR = 0.5   # finance stocks move together
CROSS_GROUP_CORR = 0.3     # different sectors, or unknown tickers
TSLA_CORR = 0.3            # TSLA is in the "tech" set but excluded from that correlation
```

`_pairwise_correlation(t1, t2)` special-cases TSLA to always use `TSLA_CORR` regardless
of its sector membership — modeling it as an idiosyncratic mover rather than a typical
tech stock.

## Random Shock Events

Independent of the GBM step, each ticker has a small per-tick chance of a sudden 2-5%
jump, for visual drama on the dashboard:

```python
if random.random() < self._event_prob:      # default 0.001 (0.1%) per tick
    shock_magnitude = random.uniform(0.02, 0.05)
    shock_sign = random.choice([-1, 1])
    self._prices[ticker] *= 1 + shock_magnitude * shock_sign
```

With 10 tickers ticking twice a second, that's roughly one shock event every ~50 seconds
across the whole watchlist.

## Seed Prices and Per-Ticker Parameters (`seed_prices.py`)

Realistic starting prices and annualized `(sigma, mu)` per default ticker:

```python
SEED_PRICES = {"AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00, ...}
TICKER_PARAMS = {
    "TSLA": {"sigma": 0.50, "mu": 0.03},   # high volatility
    "NVDA": {"sigma": 0.40, "mu": 0.08},   # high volatility, strong drift
    "JPM":  {"sigma": 0.18, "mu": 0.04},   # low volatility (bank)
    ...
}
DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}  # used for dynamically added tickers
```

Any ticker not in `TICKER_PARAMS` (e.g. added later via the watchlist or by the LLM) gets
`DEFAULT_PARAMS` and a random seed price in `[50, 300)` if not in `SEED_PRICES` either.

## Code Structure

```
GBMSimulator            # pure simulation state + math, no asyncio/cache knowledge
├── step()              # advance all tickers one tick, return {ticker: new_price}
├── add_ticker() / remove_ticker()   # mutate tracked set, rebuild Cholesky
└── get_price() / get_tickers()

SimulatorDataSource(MarketDataSource)   # asyncio wrapper implementing the shared interface
├── start(tickers)      # construct GBMSimulator, seed the PriceCache, spawn _run_loop task
├── _run_loop()          # every `update_interval` (default 0.5s): step() -> cache.update() per ticker
├── add_ticker/remove_ticker   # delegate to GBMSimulator, keep PriceCache in sync
└── stop()              # cancel the background task
```

The split matters: `GBMSimulator` is a plain, synchronous, easily-unit-testable class
(deterministic given a seeded RNG, no I/O); `SimulatorDataSource` is the thin async
adapter that fits it into the shared `MarketDataSource` lifecycle described in
`planning/MARKET_INTERFACE.md`, mirroring how `MassiveDataSource` adapts a synchronous
REST client the same way.

## Test Coverage (as-built)

Per `planning/MARKET_DATA_SUMMARY.md`: `test_simulator.py` (17 tests, 98% coverage of
`simulator.py`) plus `test_simulator_source.py` (10 integration tests against the async
wrapper).
