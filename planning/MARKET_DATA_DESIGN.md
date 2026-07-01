# Market Data Backend — Detailed Design

Implementation-ready design for the FinAlly market data subsystem: the unified
`MarketDataSource` interface, the `PriceCache`, the GBM simulator, the Massive API
client, the SSE streaming endpoint, and how the rest of the backend (portfolio,
watchlist, chat — not yet built) should wire into it.

**Status**: the subsystem described here is fully implemented, tested (73 tests
passing, 84% coverage), and reviewed. This document reflects the actual code in
`backend/app/market/`, not a pre-implementation proposal — every snippet below is
copied from or matches the real source. It consolidates and supersedes:

- `planning/archive/MARKET_DATA_DESIGN.md` — the original pre-implementation draft;
  a few details there (lazy `massive` imports, a private `_tickers` access, an
  unused `DEFAULT_CORR` constant) were changed during implementation and review.
- `planning/MARKET_INTERFACE.md`, `planning/MARKET_SIMULATOR.md`,
  `planning/MASSIVE_API.md` — focused as-built docs for the interface, the
  simulator's math, and the Massive REST API respectively. This document folds
  their content into one place alongside full code and adds the integration
  guidance (§10–§11) needed by whoever builds portfolio/watchlist/chat next.
- `planning/MARKET_DATA_SUMMARY.md` — the short summary; this is the long form.

Everything below lives under `backend/app/market/`.

---

## Table of Contents

1. [File Structure](#1-file-structure)
2. [Data Model — `models.py`](#2-data-model)
3. [Price Cache — `cache.py`](#3-price-cache)
4. [Abstract Interface — `interface.py`](#4-abstract-interface)
5. [Seed Prices & Ticker Parameters — `seed_prices.py`](#5-seed-prices--ticker-parameters)
6. [GBM Simulator — `simulator.py`](#6-gbm-simulator)
7. [Massive API Client — `massive_client.py`](#7-massive-api-client)
8. [Factory — `factory.py`](#8-factory)
9. [SSE Streaming Endpoint — `stream.py`](#9-sse-streaming-endpoint)
10. [FastAPI Lifecycle Integration (for the app not yet built)](#10-fastapi-lifecycle-integration)
11. [Watchlist Coordination (for the app not yet built)](#11-watchlist-coordination)
12. [Testing Strategy (as-built)](#12-testing-strategy)
13. [Error Handling & Edge Cases](#13-error-handling--edge-cases)
14. [Configuration Summary](#14-configuration-summary)

---

## 1. File Structure

```
backend/
  app/
    market/
      __init__.py             # Re-exports: PriceUpdate, PriceCache, MarketDataSource,
                               #             create_market_data_source, create_stream_router
      models.py                # PriceUpdate dataclass
      cache.py                 # PriceCache (thread-safe in-memory store)
      interface.py              # MarketDataSource ABC
      seed_prices.py            # SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS, CORRELATION_GROUPS
      simulator.py               # GBMSimulator + SimulatorDataSource
      massive_client.py          # MassiveDataSource
      factory.py                 # create_market_data_source()
      stream.py                  # SSE endpoint (FastAPI router factory)
  tests/
    market/
      test_models.py
      test_cache.py
      test_simulator.py
      test_simulator_source.py
      test_factory.py
      test_massive.py
  market_data_demo.py           # Rich terminal demo (uv run market_data_demo.py)
```

Each file has a single responsibility. `__init__.py` re-exports the public API so
the rest of the backend imports from `app.market` without reaching into submodules:

```python
from app.market import PriceCache, PriceUpdate, MarketDataSource, create_market_data_source, create_stream_router
```

`massive` and `numpy` are both **core** dependencies in `backend/pyproject.toml`
(not optional / lazily imported) — an early draft of this design imported `massive`
lazily inside `MassiveDataSource.start()` to avoid requiring the package when only
the simulator is used, but that was simplified away during implementation since the
project always ships both. `numpy`, `massive`, `fastapi`, `uvicorn[standard]`, and
`rich` (for the demo) are declared directly in `[project.dependencies]`.

---

## 2. Data Model

**File: `backend/app/market/models.py`**

`PriceUpdate` is the only data structure that leaves the market data layer. Every
downstream consumer — SSE streaming, portfolio valuation, trade execution — works
exclusively with this type.

```python
"""Data models for market data."""

from __future__ import annotations

import time
from dataclasses import dataclass, field


@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""

    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float:
        """Absolute price change from previous update."""
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        """Percentage change from previous update."""
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        """'up', 'down', or 'flat'."""
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        """Serialize for JSON / SSE transmission."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

### Design decisions

- **`frozen=True`**: Price updates are immutable value objects. Once created they
  never change, so they're safe to share across async tasks without copying.
- **`slots=True`**: Minor memory optimization — many of these are created per second.
- **Computed properties** (`change`, `direction`, `change_percent`): Derived from
  `price` and `previous_price` so they can never be inconsistent. No stale
  `direction` field to forget to update.
- **`to_dict()`**: Single serialization point used by both the SSE endpoint and any
  future REST API response that embeds a price.
- **`previous_price` semantics**: this is the price from the *prior update in the
  cache*, not a market "previous close" (e.g. yesterday's closing price). It answers
  "did this tick up or down from the last tick," which is what the flashing
  green/red UI and `direction` need. See §13 in `planning/PLAN.md` — `change_pct` in
  the watchlist is explicitly "computed vs. simulator seed price," a separate,
  UI-level concept computed by the frontend/route layer from `SEED_PRICES`, not by
  `PriceUpdate` itself.

---

## 3. Price Cache

**File: `backend/app/market/cache.py`**

The price cache is the central data hub. Data sources write to it; SSE streaming,
portfolio valuation, and trade execution read from it. It must be thread-safe
because `MassiveDataSource` runs its synchronous REST call inside
`asyncio.to_thread()` — a real OS thread, not just a different coroutine.

```python
"""Thread-safe in-memory price cache."""

from __future__ import annotations

import time
from threading import Lock

from .models import PriceUpdate


class PriceCache:
    """Thread-safe in-memory cache of the latest price for each ticker.

    Writers: SimulatorDataSource or MassiveDataSource (one at a time).
    Readers: SSE streaming endpoint, portfolio valuation, trade execution.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # Monotonically increasing; bumped on every update

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price for a ticker. Returns the created PriceUpdate.

        Automatically computes direction and change from the previous price.
        If this is the first update for the ticker, previous_price == price (direction='flat').
        """
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price

            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        """Get the latest price for a single ticker, or None if unknown."""
        with self._lock:
            return self._prices.get(ticker)

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all current prices. Returns a shallow copy."""
        with self._lock:
            return dict(self._prices)

    def get_price(self, ticker: str) -> float | None:
        """Convenience: get just the price float, or None."""
        update = self.get(ticker)
        return update.price if update else None

    def remove(self, ticker: str) -> None:
        """Remove a ticker from the cache (e.g., when removed from watchlist)."""
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        """Current version counter. Useful for SSE change detection."""
        return self._version

    def __len__(self) -> int:
        with self._lock:
            return len(self._prices)

    def __contains__(self, ticker: str) -> bool:
        with self._lock:
            return ticker in self._prices
```

### Why a version counter?

The SSE streaming loop polls the cache every ~500ms. Without a version counter it
would serialize and send all prices every tick even when nothing changed (e.g. the
Massive source only updates every 15s). The counter lets the SSE loop skip sends
when nothing is new — see `_generate_events` in §9.

### Thread safety rationale

`threading.Lock`, not `asyncio.Lock`, because:

- `MassiveDataSource`'s synchronous `get_snapshot_all()` call runs inside
  `asyncio.to_thread()`, which executes in a real OS thread — `asyncio.Lock` would
  not protect against that thread racing with the event loop.
- `threading.Lock` works correctly whether the caller is a sync thread or an async
  coroutine on the event loop (it just blocks briefly; the critical section is a
  dict read/write, effectively instantaneous).

### Design note: caveat on `update(timestamp=0)`

`ts = timestamp or time.time()` means an explicit `timestamp=0.0` is treated the
same as "not provided" and replaced with the current wall-clock time. This is
intentional and harmless here — no caller ever legitimately means "the Unix
epoch" — but is worth knowing if `PriceCache.update()` is ever reused outside this
subsystem's callers (`SimulatorDataSource._run_loop` and
`MassiveDataSource._poll_once`, neither of which ever passes `0`).

---

## 4. Abstract Interface

**File: `backend/app/market/interface.py`**

```python
"""Abstract interface for market data sources."""

from __future__ import annotations

from abc import ABC, abstractmethod


class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never calls the data source directly for prices —
    it reads from the cache.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])
        # ... app runs ...
        await source.add_ticker("TSLA")
        await source.remove_ticker("GOOGL")
        # ... app shutting down ...
        await source.stop()
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates for the given tickers.

        Starts a background task that periodically writes to the PriceCache.
        Must be called exactly once. Calling start() twice is undefined behavior.
        """

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources.

        Safe to call multiple times. After stop(), the source will not write
        to the cache again.
        """

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present.

        The next update cycle will include this ticker.
        """

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the active set. No-op if not present.

        Also removes the ticker from the PriceCache.
        """

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

### Why the source writes to the cache instead of returning prices

This push model decouples timing. The simulator ticks every 500ms; Massive polls
every 15s; SSE always reads from the cache at its own 500ms cadence. The SSE layer
never needs to know which data source is active or what its update interval is —
`PriceCache.version` is the only signal it watches.

### Both implementations conform to the same contract

```
MarketDataSource (ABC)                  interface.py
├── SimulatorDataSource                 simulator.py       (default, no API key)
└── MassiveDataSource                   massive_client.py  (MASSIVE_API_KEY set)
        │
        ▼
   PriceCache                           cache.py
        │
        ├──→ SSE stream (/api/stream/prices)
        ├──→ Portfolio valuation   (not yet built)
        └──→ Trade execution        (not yet built)
```

---

## 5. Seed Prices & Ticker Parameters

**File: `backend/app/market/seed_prices.py`**

Constants only — no logic, no imports beyond stdlib. Consumed by the simulator for
initial prices and GBM parameters, and available as a reference for any future
route that needs a "seed price" baseline (e.g. `change_pct` since page load, per
`planning/PLAN.md` §13).

```python
"""Seed prices and per-ticker parameters for the market simulator."""

# Realistic starting prices for the default watchlist (as of project creation)
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00,
    "GOOGL": 175.00,
    "MSFT": 420.00,
    "AMZN": 185.00,
    "TSLA": 250.00,
    "NVDA": 800.00,
    "META": 500.00,
    "JPM": 195.00,
    "V": 280.00,
    "NFLX": 600.00,
}

# Per-ticker GBM parameters
# sigma: annualized volatility (higher = more price movement)
# mu: annualized drift / expected return
TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL": {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT": {"sigma": 0.20, "mu": 0.05},
    "AMZN": {"sigma": 0.28, "mu": 0.05},
    "TSLA": {"sigma": 0.50, "mu": 0.03},  # High volatility
    "NVDA": {"sigma": 0.40, "mu": 0.08},  # High volatility, strong drift
    "META": {"sigma": 0.30, "mu": 0.05},
    "JPM": {"sigma": 0.18, "mu": 0.04},  # Low volatility (bank)
    "V": {"sigma": 0.17, "mu": 0.04},  # Low volatility (payments)
    "NFLX": {"sigma": 0.35, "mu": 0.05},
}

# Default parameters for tickers not in the list above (dynamically added)
DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}

# Correlation groups for the simulator's Cholesky decomposition
# Tickers in the same group have higher intra-group correlation
CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

# Correlation coefficients
INTRA_TECH_CORR = 0.6  # Tech stocks move together
INTRA_FINANCE_CORR = 0.5  # Finance stocks move together
CROSS_GROUP_CORR = 0.3  # Between sectors / unknown tickers
TSLA_CORR = 0.3  # TSLA does its own thing
```

Any ticker not in `TICKER_PARAMS` (e.g. added later via the watchlist or by the
LLM) gets `DEFAULT_PARAMS` and, if also missing from `SEED_PRICES`, a random seed
price uniformly drawn from `[50, 300]` (see `_add_ticker_internal` in §6). `TSLA`
is deliberately **excluded** from `CORRELATION_GROUPS["tech"]` even though it's a
tech stock — `_pairwise_correlation()` special-cases it to always use `TSLA_CORR`,
modeling it as an idiosyncratic mover rather than a typical large-cap tech name.

An earlier draft of this file also defined `DEFAULT_CORR = 0.3` as a fourth
constant duplicating `CROSS_GROUP_CORR`'s value; it was unused by
`_pairwise_correlation()` (which falls through to `CROSS_GROUP_CORR` for both
cross-sector and unknown-ticker pairs) and was removed during code review.

---

## 6. GBM Simulator

**File: `backend/app/market/simulator.py`**

Two classes live here:

- `GBMSimulator` — pure math engine. Synchronous, stateful, no asyncio/cache
  knowledge, easy to unit test deterministically (given seeded `numpy.random` and
  stdlib `random` global RNGs — see the determinism caveat below).
- `SimulatorDataSource` — the `MarketDataSource` implementation that wraps
  `GBMSimulator` in an async loop and writes results to the `PriceCache`.

### 6.1 Why GBM

Geometric Brownian Motion is the standard textbook model for a stock price:
log-returns are normally distributed, prices stay strictly positive (the update is
multiplicative via `exp(...)`), and volatility scales with price level. It needs
only two parameters per ticker (drift `mu`, volatility `sigma`) and is cheap to
step every 500ms for a whole watchlist.

```
S(t+dt) = S(t) * exp((mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z)
```

- `S(t)` — current price
- `mu` — annualized drift (expected return)
- `sigma` — annualized volatility
- `dt` — time step, as a fraction of a trading year
- `Z` — a (correlated) standard normal draw

### 6.2 Deriving `dt` from the 500ms tick

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8
```

252 trading days/year, 6.5 trading hours/day. A 500ms tick is an extremely small
slice of a trading year — intentional, since it produces sub-cent moves per tick
that accumulate into realistic-looking intraday paths rather than jumping
wildly every half second.

### 6.3 Correlated moves via Cholesky decomposition

Real markets move together — tech stocks drift together, financials drift
together. The simulator reproduces this by drawing correlated normal variables
instead of independent ones:

1. Build an `n x n` correlation matrix (`n` = number of tracked tickers) from
   sector-based pairwise correlations (`_pairwise_correlation`).
2. Compute its Cholesky decomposition `L` (`numpy.linalg.cholesky`).
3. Each tick, draw `n` independent standard normals `Z_indep`, then
   `Z_correlated = L @ Z_indep`.
4. Feed `Z_correlated[i]` into ticker `i`'s GBM step.

The matrix is rebuilt (`_rebuild_cholesky`) whenever a ticker is added or removed —
`O(n^2)`, fine for the small watchlists this project targets (`n < 50`). With `n
<= 1` there's nothing to correlate, so `_cholesky` is left `None` and `step()` uses
the independent draws directly.

### 6.4 Random shock events

Independent of the GBM step, each ticker has a small per-tick chance of a sudden
2–5% jump, purely for visual drama on the dashboard:

```python
if random.random() < self._event_prob:      # default 0.001 (0.1%) per tick
    shock_magnitude = random.uniform(0.02, 0.05)
    shock_sign = random.choice([-1, 1])
    self._prices[ticker] *= 1 + shock_magnitude * shock_sign
```

With 10 tickers ticking twice a second, that's roughly one shock event every ~50
seconds across the whole watchlist.

### 6.5 Full source

```python
"""GBM-based market simulator."""

from __future__ import annotations

import asyncio
import logging
import math
import random

import numpy as np

from .cache import PriceCache
from .interface import MarketDataSource
from .seed_prices import (
    CORRELATION_GROUPS,
    CROSS_GROUP_CORR,
    DEFAULT_PARAMS,
    INTRA_FINANCE_CORR,
    INTRA_TECH_CORR,
    SEED_PRICES,
    TICKER_PARAMS,
    TSLA_CORR,
)

logger = logging.getLogger(__name__)


class GBMSimulator:
    """Geometric Brownian Motion simulator for correlated stock prices.

    Math:
        S(t+dt) = S(t) * exp((mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z)

    Where:
        S(t)   = current price
        mu     = annualized drift (expected return)
        sigma  = annualized volatility
        dt     = time step as fraction of a trading year
        Z      = correlated standard normal random variable

    The tiny dt (~8.5e-8 for 500ms ticks over 252 trading days * 6.5h/day)
    produces sub-cent moves per tick that accumulate naturally over time.
    """

    # 500ms expressed as a fraction of a trading year
    # 252 trading days * 6.5 hours/day * 3600 seconds/hour = 5,896,800 seconds
    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR  # ~8.48e-8

    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
    ) -> None:
        self._dt = dt
        self._event_prob = event_probability

        # Per-ticker state
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}

        # Cholesky decomposition of the correlation matrix (for correlated moves)
        self._cholesky: np.ndarray | None = None

        # Initialize all starting tickers
        for ticker in tickers:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    # --- Public API ---

    def step(self) -> dict[str, float]:
        """Advance all tickers by one time step. Returns {ticker: new_price}.

        This is the hot path — called every 500ms. Keep it fast.
        """
        n = len(self._tickers)
        if n == 0:
            return {}

        # Generate n independent standard normal draws
        z_independent = np.random.standard_normal(n)

        # Apply Cholesky to get correlated draws
        if self._cholesky is not None:
            z_correlated = self._cholesky @ z_independent
        else:
            z_correlated = z_independent

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            params = self._params[ticker]
            mu = params["mu"]
            sigma = params["sigma"]

            # GBM: S(t+dt) = S(t) * exp((mu - 0.5*sigma^2)*dt + sigma*sqrt(dt)*Z)
            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random event: ~0.1% chance per tick per ticker
            # With 10 tickers at 2 ticks/sec, expect an event ~every 50 seconds
            if random.random() < self._event_prob:
                shock_magnitude = random.uniform(0.02, 0.05)
                shock_sign = random.choice([-1, 1])
                self._prices[ticker] *= 1 + shock_magnitude * shock_sign
                logger.debug(
                    "Random event on %s: %.1f%% %s",
                    ticker,
                    shock_magnitude * 100,
                    "up" if shock_sign > 0 else "down",
                )

            result[ticker] = round(self._prices[ticker], 2)

        return result

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the simulation. Rebuilds the correlation matrix."""
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the simulation. Rebuilds the correlation matrix."""
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        """Current price for a ticker, or None if not tracked."""
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        """Return the list of currently tracked tickers."""
        return list(self._tickers)

    # --- Internals ---

    def _add_ticker_internal(self, ticker: str) -> None:
        """Add a ticker without rebuilding Cholesky (for batch initialization)."""
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        """Rebuild the Cholesky decomposition of the ticker correlation matrix.

        Called whenever tickers are added or removed. O(n^2) but n < 50.
        """
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return

        # Build the correlation matrix
        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = rho
                corr[j, i] = rho

        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float:
        """Determine correlation between two tickers based on sector grouping.

        Correlation structure:
          - Same tech sector:    0.6
          - Same finance sector: 0.5
          - TSLA with anything:  0.3 (it does its own thing)
          - Cross-sector:        0.3
          - Unknown tickers:     0.3
        """
        tech = CORRELATION_GROUPS["tech"]
        finance = CORRELATION_GROUPS["finance"]

        # TSLA is not in any correlation group; always pin to TSLA_CORR
        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR

        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR

        return CROSS_GROUP_CORR


class SimulatorDataSource(MarketDataSource):
    """MarketDataSource backed by the GBM simulator.

    Runs a background asyncio task that calls GBMSimulator.step() every
    `update_interval` seconds and writes results to the PriceCache.
    """

    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,
        event_probability: float = 0.001,
    ) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(
            tickers=tickers,
            event_probability=self._event_prob,
        )
        # Seed the cache with initial prices so SSE has data immediately
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")
        logger.info("Simulator started with %d tickers", len(tickers))

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        logger.info("Simulator stopped")

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            # Seed cache immediately so the ticker has a price right away
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
            logger.info("Simulator: added ticker %s", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)
        logger.info("Simulator: removed ticker %s", ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        """Core loop: step the simulation, write to cache, sleep."""
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

### Key behaviors

- **Immediate seeding**: `start()` populates the cache with seed prices *before*
  the loop begins, and `add_ticker()` does the same for a ticker added later. The
  SSE endpoint therefore always has data to send on its very first tick — no
  blank-screen delay.
- **Public `get_tickers()`**: `SimulatorDataSource.get_tickers()` delegates to
  `GBMSimulator.get_tickers()` rather than reaching into `self._sim._tickers`
  directly — an earlier draft accessed the private attribute; this was fixed
  during review to avoid coupling across the private boundary.
- **Graceful cancellation**: `stop()` cancels the task and awaits it, catching
  `CancelledError`. Clean shutdown during FastAPI lifespan teardown.
- **Exception resilience**: the loop catches exceptions per-step so one bad tick
  doesn't kill the entire data feed.

### Determinism caveat

`GBMSimulator` is deterministic given a seeded RNG — but it draws from **two**
process-global RNGs, not one: `np.random.standard_normal()` for the correlated
moves, and stdlib `random` for unknown-ticker seed prices (`_add_ticker_internal`)
and shock events (`step`). Reproducing a simulator run requires seeding both
`numpy.random.seed(...)` and `random.seed(...)`, and requires that nothing else in
the process consumes either global RNG state in between (e.g. two simulator
instances, or another module calling `random.random()`, will desynchronize the
sequence). Tests that need determinism should seed both, or assert on statistical
properties (positivity, boundedness) rather than exact values.

---

## 7. Massive API Client

**File: `backend/app/market/massive_client.py`**

Polls the Massive (formerly Polygon.io) REST API snapshot endpoint on a
configurable interval. See `planning/MASSIVE_API.md` for the full REST reference
(auth, rate limits, response shapes, EOD endpoints for future use); this section
covers only what `MassiveDataSource` uses today.

```python
"""Massive (Polygon.io) API client for real market data."""

from __future__ import annotations

import asyncio
import logging

from massive import RESTClient
from massive.rest.models import SnapshotMarketType

from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


class MassiveDataSource(MarketDataSource):
    """MarketDataSource backed by the Massive (Polygon.io) REST API.

    Polls GET /v2/snapshot/locale/us/markets/stocks/tickers for all watched
    tickers in a single API call, then writes results to the PriceCache.

    Rate limits:
      - Free tier: 5 req/min → poll every 15s (default)
      - Paid tiers: higher limits → poll every 2-5s
    """

    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,
    ) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)

        # Do an immediate first poll so the cache has data right away
        await self._poll_once()

        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")
        logger.info(
            "Massive poller started: %d tickers, %.1fs interval",
            len(tickers),
            self._interval,
        )

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None
        logger.info("Massive poller stopped")

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)
            logger.info("Massive: added ticker %s (will appear on next poll)", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)
        logger.info("Massive: removed ticker %s", ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # --- Internal ---

    async def _poll_loop(self) -> None:
        """Poll on interval. First poll already happened in start()."""
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        """Execute one poll cycle: fetch snapshots, update cache."""
        if not self._tickers or not self._client:
            return

        try:
            # The Massive RESTClient is synchronous — run in a thread to
            # avoid blocking the event loop.
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            processed = 0
            for snap in snapshots:
                try:
                    price = snap.last_trade.price
                    # Massive timestamps are Unix milliseconds → convert to seconds
                    timestamp = snap.last_trade.timestamp / 1000.0
                    self._cache.update(
                        ticker=snap.ticker,
                        price=price,
                        timestamp=timestamp,
                    )
                    processed += 1
                except (AttributeError, TypeError) as e:
                    logger.warning(
                        "Skipping snapshot for %s: %s",
                        getattr(snap, "ticker", "???"),
                        e,
                    )
            logger.debug("Massive poll: updated %d/%d tickers", processed, len(self._tickers))

        except Exception as e:
            logger.error("Massive poll failed: %s", e)
            # Don't re-raise — the loop will retry on the next interval.
            # Common failures: 401 (bad key), 429 (rate limit), network errors.

    def _fetch_snapshots(self) -> list:
        """Synchronous call to the Massive REST API. Runs in a thread."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

### Imports are top-level, not lazy

`from massive import RESTClient` and `from massive.rest.models import
SnapshotMarketType` happen at module import time. A pre-implementation draft of
this design deferred these imports into `start()` on the theory that students
without a `MASSIVE_API_KEY` shouldn't need the package installed — but `massive`
is a required top-level dependency in `pyproject.toml` regardless of which source
is active (see §1), so the lazy-import indirection was removed; `massive_client.py`
always imports cleanly.

### Error handling philosophy

The Massive poller is intentionally resilient — a single bad tick, a rate limit, or
a transient network failure should never take down the price feed:

| Error | Behavior |
|-------|----------|
| **401 Unauthorized** | Logged as error. Poller keeps running (user might fix `.env` and restart). |
| **429 Rate Limited** | Logged as error. Next poll retries after `poll_interval` seconds. |
| **Network timeout** | Logged as error. Retries automatically on next cycle. |
| **Malformed snapshot** (e.g. missing `last_trade`) | Individual ticker skipped with a warning; other tickers in the same response still processed. |
| **All tickers fail** | Cache retains last-known prices. SSE keeps streaming stale data (better than no data). |

### Timestamp conversion

Massive's `last_trade.timestamp` is Unix **milliseconds**; `PriceCache.update()`
and `PriceUpdate.timestamp` both expect Unix **seconds** (matching
`time.time()`). `_poll_once` divides by `1000.0` before calling `cache.update()`.

---

## 8. Factory

**File: `backend/app/market/factory.py`**

```python
"""Factory for creating market data sources."""

from __future__ import annotations

import logging
import os

from .cache import PriceCache
from .interface import MarketDataSource
from .massive_client import MassiveDataSource
from .simulator import SimulatorDataSource

logger = logging.getLogger(__name__)


def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Create the appropriate market data source based on environment variables.

    - MASSIVE_API_KEY set and non-empty → MassiveDataSource (real market data)
    - Otherwise → SimulatorDataSource (GBM simulation)

    Returns an unstarted source. Caller must await source.start(tickers).
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        logger.info("Market data source: Massive API (real data)")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        logger.info("Market data source: GBM Simulator")
        return SimulatorDataSource(price_cache=price_cache)
```

This is the entire selection logic: a non-empty, whitespace-trimmed
`MASSIVE_API_KEY` selects real data; anything else (unset, empty string,
whitespace-only) selects the simulator. Both `MassiveDataSource` and
`SimulatorDataSource` are imported at module level (both are core dependencies —
see §1), so there's no import-time branching to reason about.

### Usage at app startup

```python
price_cache = PriceCache()
source = create_market_data_source(price_cache)
await source.start(initial_tickers)  # e.g., ["AAPL", "GOOGL", ...]
```

---

## 9. SSE Streaming Endpoint

**File: `backend/app/market/stream.py`**

A FastAPI route that holds open a long-lived HTTP connection and pushes price
updates to the client as `text/event-stream`.

```python
"""SSE streaming endpoint for live price updates."""

from __future__ import annotations

import asyncio
import json
import logging
from collections.abc import AsyncGenerator

from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse

from .cache import PriceCache

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/api/stream", tags=["streaming"])


def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Create the SSE streaming router with a reference to the price cache.

    This factory pattern lets us inject the PriceCache without globals.
    """

    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        """SSE endpoint for live price updates.

        Streams all tracked ticker prices every ~500ms. The client connects
        with EventSource and receives events in the format:

            data: {"AAPL": {"ticker": "AAPL", "price": 190.50, ...}, ...}

        Includes a retry directive so the browser auto-reconnects on
        disconnection (EventSource built-in behavior).
        """
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",  # Disable nginx buffering if proxied
            },
        )

    return router


async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    """Async generator that yields SSE-formatted price events.

    Sends all prices every `interval` seconds. Stops when the client
    disconnects (detected via request.is_disconnected()).
    """
    # Tell the client to retry after 1 second if the connection drops
    yield "retry: 1000\n\n"

    last_version = -1
    client_ip = request.client.host if request.client else "unknown"
    logger.info("SSE client connected: %s", client_ip)

    try:
        while True:
            # Check for client disconnect
            if await request.is_disconnected():
                logger.info("SSE client disconnected: %s", client_ip)
                break

            current_version = price_cache.version
            if current_version != last_version:
                last_version = current_version
                prices = price_cache.get_all()

                if prices:
                    data = {ticker: update.to_dict() for ticker, update in prices.items()}
                    payload = json.dumps(data)
                    yield f"data: {payload}\n\n"

            await asyncio.sleep(interval)
    except asyncio.CancelledError:
        logger.info("SSE stream cancelled for: %s", client_ip)
```

The generator's return type is annotated `AsyncGenerator[str, None]` (not `None`)
— it's an async generator function, and the correct type hint reflects that it
yields `str` chunks, which matters for `mypy`/IDE tooling even though FastAPI
itself only cares that it's iterable.

### SSE wire format

```
data: {"AAPL":{"ticker":"AAPL","price":190.50,"previous_price":190.42,"timestamp":1751328000.5,"change":0.08,"change_percent":0.042,"direction":"up"},"GOOGL":{"ticker":"GOOGL","price":175.12,...}}

```

Client-side:

```javascript
const eventSource = new EventSource('/api/stream/prices');
eventSource.onmessage = (event) => {
    const prices = JSON.parse(event.data);
    // prices is { "AAPL": { ticker, price, previous_price, timestamp, change, change_percent, direction }, ... }
};
```

### Why poll-and-push instead of event-driven?

The SSE endpoint polls the cache on a fixed interval rather than being notified by
the data source. This is simpler and produces predictable, evenly-spaced updates
for the frontend, which matters because the frontend accumulates these into
sparkline charts (per `planning/PLAN.md` §2/§10) — regular spacing keeps that
visualization clean regardless of which data source (500ms simulator ticks vs.
15s Massive polls) is actually driving the cache.

### Multiple tabs, one cache

Per `planning/PLAN.md` §13, multiple browser tabs each open their own `EventSource`
connection and their own `_generate_events` generator/task, but all read from the
same process-wide `PriceCache` — no per-connection state beyond `last_version`, so
this scales fine for the single-user, single-process deployment this project
targets.

---

## 10. FastAPI Lifecycle Integration

**Not yet built** — `backend/app/main.py` doesn't exist yet; this section is
guidance for whoever wires the market data subsystem into the FastAPI app as part
of building the rest of the platform (portfolio, watchlist, chat routes, per
`planning/PLAN.md` §4/§8). It follows directly from the public API in §1 and the
`lifespan` pattern FastAPI recommends for background-task startup/shutdown.

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from app.market import PriceCache, create_market_data_source, create_stream_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Manage startup and shutdown of background services."""

    # --- STARTUP ---

    # 1. Create the shared price cache
    price_cache = PriceCache()
    app.state.price_cache = price_cache

    # 2. Create and start the market data source
    source = create_market_data_source(price_cache)
    app.state.market_source = source

    # 3. Load initial tickers from the database watchlist (lazy-init db first —
    #    see planning/PLAN.md §7 for the lazy schema/seed logic)
    initial_tickers = await load_watchlist_tickers()  # reads from SQLite
    await source.start(initial_tickers)

    # 4. Register the SSE streaming router
    app.include_router(create_stream_router(price_cache))

    yield  # App is running

    # --- SHUTDOWN ---
    await source.stop()


app = FastAPI(title="FinAlly", lifespan=lifespan)


# Dependencies for injecting market data state into route handlers
def get_price_cache() -> PriceCache:
    return app.state.price_cache


def get_market_source():
    return app.state.market_source
```

### Accessing market data from other routes

Portfolio, watchlist, and trade routes reach the cache and the active source via
FastAPI dependency injection — never by importing `SimulatorDataSource` or
`MassiveDataSource` directly, keeping those routes source-agnostic:

```python
from fastapi import APIRouter, Depends, HTTPException

from app.market import MarketDataSource, PriceCache

router = APIRouter(prefix="/api")


@router.post("/portfolio/trade")
async def execute_trade(
    trade: TradeRequest,
    price_cache: PriceCache = Depends(get_price_cache),
):
    current_price = price_cache.get_price(trade.ticker)
    if current_price is None:
        raise HTTPException(404, f"No price available for {trade.ticker}")
    # ... shared trade validation + execution (PLAN.md §8) at current_price ...


@router.post("/watchlist")
async def add_to_watchlist(
    payload: WatchlistAdd,
    source: MarketDataSource = Depends(get_market_source),
):
    # ... insert into watchlist table ...
    await source.add_ticker(payload.ticker.upper())
    # ... return the ticker + its (now cached) current price ...


@router.delete("/watchlist/{ticker}")
async def remove_from_watchlist(
    ticker: str,
    source: MarketDataSource = Depends(get_market_source),
):
    # ... delete from watchlist table ...
    await source.remove_ticker(ticker.upper())
```

---

## 11. Watchlist Coordination

**Not yet built** — describes how the (future) watchlist routes should keep the
market data source's active ticker set in sync with the SQLite `watchlist` table.

### Flow: adding a ticker

```
User (or LLM) → POST /api/watchlist {ticker: "PYPL"}
  → INSERT INTO watchlist (SQLite)
  → await source.add_ticker("PYPL")
      Simulator: adds to GBMSimulator, rebuilds Cholesky, seeds cache immediately
      Massive:   appends to internal ticker list, appears on the next poll (≤15s)
  → Return success (ticker + current price, if already cached)
```

### Flow: removing a ticker

```
User (or LLM) → DELETE /api/watchlist/PYPL
  → DELETE FROM watchlist (SQLite)
  → await source.remove_ticker("PYPL")
      Simulator: removes from GBMSimulator, rebuilds Cholesky, removes from cache
      Massive:   removes from internal ticker list, removes from cache
  → Return success
```

### Edge case: ticker has an open position

Per `planning/PLAN.md` §13 ("SSE scope: watchlist tickers only; removing from
watchlist stops live price updates for that ticker") combined with §8's trade
auto-add-to-watchlist rule, a user could remove a ticker from the watchlist while
still holding shares. If the data source stops tracking it, portfolio valuation
for that position goes stale. The watchlist-removal route should guard against
this by checking for an open position before calling `remove_ticker`:

```python
@router.delete("/watchlist/{ticker}")
async def remove_from_watchlist(
    ticker: str,
    source: MarketDataSource = Depends(get_market_source),
):
    await db.delete_watchlist_entry(ticker)

    position = await db.get_position(ticker)
    if position is None or position.quantity == 0:
        await source.remove_ticker(ticker)
    # else: keep tracking it in the background so /api/portfolio stays accurate,
    # even though it no longer appears in the SSE watchlist stream.

    return {"status": "ok"}
```

This is a design recommendation for the portfolio/watchlist implementation, not
existing code — `app.market` itself has no concept of "positions"; it only tracks
whatever ticker set `add_ticker`/`remove_ticker`/`start` tell it to.

---

## 12. Testing Strategy (as-built)

**73 tests, all passing.** 6 modules under `backend/tests/market/`, run with `uv
run --extra dev pytest -v` (see `backend/CLAUDE.md`).

| Module | Tests | Coverage | What it covers |
|--------|-------|----------|-----------------|
| `test_models.py` | 11 | `models.py`: 100% | `PriceUpdate` properties, `to_dict()`, edge cases (`previous_price == 0`, flat direction) |
| `test_cache.py` | 13 | `cache.py`: 100% | update/get/get_all/get_price/remove, version increments, first-update-is-flat, thread-safety-shaped access patterns |
| `test_simulator.py` | 18 | `simulator.py`: 98% | `GBMSimulator` math: positivity, seed prices, add/remove ticker, Cholesky rebuild triggers, unknown-ticker random seed range, empty-tickers step |
| `test_simulator_source.py` | 10 | (integration) | `SimulatorDataSource` async lifecycle: start populates cache immediately, prices evolve over time, clean/idempotent stop, add/remove ticker propagates to cache |
| `test_factory.py` | 7 | `factory.py`: 100% | env-var selection logic: empty/unset/whitespace `MASSIVE_API_KEY` → simulator; non-empty → Massive; returned source is unstarted |
| `test_massive.py` | 13 | `massive_client.py`: 56% (expected — REST calls are mocked, not live-hit) | poll updates cache, malformed snapshot skipped without crashing the whole poll, API/network errors don't raise out of `_poll_once` |

Overall coverage: **84%**. The gap in `massive_client.py` is intentional — the
real HTTP calls to `RESTClient.get_snapshot_all()` are never exercised in tests
(no live API key in CI); `_fetch_snapshots` is mocked via
`unittest.mock.patch.object(source, "_fetch_snapshots", ...)` and the surrounding
error-handling paths (`_poll_once`'s `try`/`except`) are what's actually verified.

### Representative test shapes

Positivity property (GBM prices can never go negative — the model is
multiplicative via `exp()`):

```python
def test_prices_are_positive(self):
    sim = GBMSimulator(tickers=["AAPL"])
    for _ in range(10_000):
        prices = sim.step()
        assert prices["AAPL"] > 0
```

Cache version semantics:

```python
def test_version_increments(self):
    cache = PriceCache()
    v0 = cache.version
    cache.update("AAPL", 190.00)
    assert cache.version == v0 + 1
```

Massive malformed-snapshot resilience (one bad ticker doesn't sink the poll):

```python
async def test_malformed_snapshot_skipped(self):
    cache = PriceCache()
    source = MassiveDataSource(api_key="test-key", price_cache=cache, poll_interval=60.0)
    source._tickers = ["AAPL", "BAD"]

    good_snap = _make_snapshot("AAPL", 190.50, 1707580800000)
    bad_snap = MagicMock(ticker="BAD", last_trade=None)  # triggers AttributeError

    with patch.object(source, "_fetch_snapshots", return_value=[good_snap, bad_snap]):
        await source._poll_once()

    assert cache.get_price("AAPL") == 190.50
    assert cache.get_price("BAD") is None
```

### Demo (manual/exploratory verification)

`backend/market_data_demo.py` — a Rich terminal dashboard, not part of the
automated suite, useful for visually sanity-checking the simulator:

```bash
cd backend
uv run market_data_demo.py
```

Displays all 10 default tickers live-updating with sparklines, color-coded
direction arrows, and an event log for shock events. Runs 60 seconds or until
Ctrl+C.

---

## 13. Error Handling & Edge Cases

### 13.1 Startup: empty watchlist

If the database has no watchlist entries, `start([])` is called. Both sources
handle this gracefully: `GBMSimulator.step()` on zero tickers returns `{}`
immediately (`n == 0` short-circuit); the Massive poller's `_poll_once()`
early-returns when `self._tickers` is empty. The SSE endpoint sends no `data:`
events (an empty `prices` dict is falsy, so `_generate_events` skips the yield).
When a ticker is later added, tracking starts immediately.

### 13.2 Price cache miss during trade

Only relevant once trade routes exist (§10/§11), but worth designing for now: if a
user tries to trade a ticker with no cached price yet (e.g. Massive hasn't polled
it for the first time), the route should surface a clear 400 rather than treating
`None` as `0`:

```python
price = price_cache.get_price(ticker)
if price is None:
    raise HTTPException(
        status_code=400,
        detail=f"Price not yet available for {ticker}. Please wait a moment and try again.",
    )
```

In practice this window is near-zero for the simulator (seeded synchronously in
`add_ticker`/`start`) and bounded by one HTTP round-trip for Massive (`start()`
does an immediate `_poll_once()` before returning, and `add_ticker()` appends to
the list so the *next* scheduled poll — up to `poll_interval` seconds away —
picks it up; there is no immediate poll on `add_ticker()` today).

### 13.3 Massive API key invalid

First poll fails with 401; the poller logs the error and keeps retrying on its
normal interval (it does not back off or give up). SSE keeps streaming — with an
empty payload if no ticker has ever successfully cached a price, or stale prices
if some had. The connection status indicator (per `planning/PLAN.md` §2) would
correctly show "connected" since the SSE transport itself is healthy; only the
data is missing/stale. Fix: correct `MASSIVE_API_KEY` in `.env` and restart the
container (the key is read once, in `factory.create_market_data_source()`, at
process startup — not hot-reloaded).

### 13.4 Thread safety under load

`PriceCache` uses a plain mutex (`threading.Lock`) — only one thread/coroutine
holds it at a time. Under this project's actual load (≤10s of tickers, ≤2
updates/sec, a handful of SSE readers per the "multiple tabs" case in §9), lock
contention is negligible; the critical section is a dict read/write. If this ever
became a bottleneck (hundreds of tickers, many concurrent readers), the fix would
be a read/write lock — unnecessary complexity for this project's scale.

### 13.5 Simulator numerical precision

GBM with the tiny `dt` derived in §6.2 produces very small per-tick moves.
Floating-point precision is not a practical concern because:

- Prices are `round()`ed to 2 decimals both in `GBMSimulator.step()` and again in
  `PriceCache.update()` (belt-and-suspenders — the cache rounds whatever price a
  source hands it, so a hypothetical future source that doesn't round internally
  still gets consistent 2-decimal storage).
- The multiplicative `exp(drift + diffusion)` formulation is numerically stable
  and keeps prices strictly positive by construction — there's no additive
  formulation that could drift a price to zero or negative.

---

## 14. Configuration Summary

All tunable parameters and their defaults:

| Parameter | Location | Default | Description |
|-----------|----------|---------|-------------|
| `MASSIVE_API_KEY` | Environment variable | `""` (empty) | If set (non-empty after `.strip()`), use Massive API; otherwise use the simulator |
| `update_interval` | `SimulatorDataSource.__init__` | `0.5` (seconds) | Time between simulator ticks |
| `event_probability` | `GBMSimulator.__init__` (passed through from `SimulatorDataSource`) | `0.001` | Chance of a random shock event per ticker per tick |
| `dt` | `GBMSimulator.__init__` | `GBMSimulator.DEFAULT_DT` ≈ `8.48e-8` | GBM time step, as a fraction of a trading year |
| `poll_interval` | `MassiveDataSource.__init__` | `15.0` (seconds) | Time between Massive API polls (matches the free-tier 5 req/min budget) |
| SSE push interval | `_generate_events(interval=...)` | `0.5` (seconds) | Cache-poll cadence inside the SSE generator |
| SSE retry directive | `_generate_events` (`"retry: 1000\n\n"`) | `1000` (ms) | Browser `EventSource` reconnection delay after a dropped connection |

None of these are currently exposed as environment variables beyond
`MASSIVE_API_KEY` — `update_interval`, `poll_interval`, `event_probability`, and
`dt` are constructor defaults, overridable by whoever calls
`SimulatorDataSource(...)` / `MassiveDataSource(...)` directly (as the test suite
does, e.g. `update_interval=0.05` for fast integration tests), but
`create_market_data_source()` always uses the defaults. If a future need arises to
tune these without code changes (e.g. a faster demo mode), the factory would be
the place to read additional environment variables and pass them through.

### Package `__init__.py`

**File: `backend/app/market/__init__.py`**

```python
"""Market data subsystem for FinAlly.

Public API:
    PriceUpdate         - Immutable price snapshot dataclass
    PriceCache          - Thread-safe in-memory price store
    MarketDataSource    - Abstract interface for data providers
    create_market_data_source - Factory that selects simulator or Massive
    create_stream_router - FastAPI router factory for SSE endpoint
"""

from .cache import PriceCache
from .factory import create_market_data_source
from .interface import MarketDataSource
from .models import PriceUpdate
from .stream import create_stream_router

__all__ = [
    "PriceUpdate",
    "PriceCache",
    "MarketDataSource",
    "create_market_data_source",
    "create_stream_router",
]
```
