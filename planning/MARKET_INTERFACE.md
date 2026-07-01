# Unified Market Data Interface

Documents the as-built design in `backend/app/market/` for retrieving stock prices from
either the Massive API or the built-in simulator behind one interface. This reflects the
actual implemented code (see `backend/CLAUDE.md` and `planning/MARKET_DATA_SUMMARY.md`
for the higher-level summary); it supersedes the pre-implementation draft in
`planning/archive/MARKET_INTERFACE.md`.

## Shape

```
MarketDataSource (ABC)                  interface.py
├── SimulatorDataSource                 simulator.py   (default, no API key)
└── MassiveDataSource                   massive_client.py (MASSIVE_API_KEY set)
        │
        ▼
   PriceCache                           cache.py
        │
        ├──→ SSE stream (/api/stream/prices)
        ├──→ Portfolio valuation
        └──→ Trade execution
```

Both sources write into the same `PriceCache`; every downstream consumer reads from the
cache and is agnostic to which source is running.

## `PriceUpdate` (models.py)

Immutable, frozen dataclass — one price observation for one ticker:

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float: ...          # price - previous_price
    @property
    def change_percent(self) -> float: ...  # % change vs previous_price
    @property
    def direction(self) -> str: ...         # "up" / "down" / "flat"
    def to_dict(self) -> dict: ...          # JSON-serializable for SSE
```

`previous_price` is the price from the *prior update in the cache* (not a market
"previous close") — see "Future Extension" below for how EOD data could change that.

## `MarketDataSource` (interface.py)

Abstract contract both sources implement:

```python
class MarketDataSource(ABC):
    async def start(self, tickers: list[str]) -> None: ...
    async def stop(self) -> None: ...
    async def add_ticker(self, ticker: str) -> None: ...
    async def remove_ticker(self, ticker: str) -> None: ...
    def get_tickers(self) -> list[str]: ...
```

Lifecycle:
```python
source = create_market_data_source(cache)
await source.start(["AAPL", "GOOGL", ...])
await source.add_ticker("TSLA")
await source.remove_ticker("GOOGL")
await source.stop()
```

- `start()` spawns a background asyncio task that periodically writes to the `PriceCache`.
  Call exactly once.
- `stop()` cancels that task; safe to call multiple times.
- `add_ticker` / `remove_ticker` mutate the active set live — the next update cycle
  picks up the change (simulator: rebuilds correlation matrix; Massive: included in
  next poll).

## `PriceCache` (cache.py)

Thread-safe (uses a plain `Lock`, not asyncio-specific — safe if a source ever runs
sync work in a thread, as `MassiveDataSource` does).

```python
cache = PriceCache()
cache.update(ticker="AAPL", price=191.20)   # -> PriceUpdate, bumps cache.version
cache.get("AAPL")                            # -> PriceUpdate | None
cache.get_price("AAPL")                      # -> float | None
cache.get_all()                              # -> dict[str, PriceUpdate] (shallow copy)
cache.remove("AAPL")
cache.version                                # monotonic int, for SSE change detection
```

`update()` computes `previous_price` from whatever was already in the cache for that
ticker (or `price` itself on first write, giving `direction="flat"`). Both sources call
this the same way — the cache has no knowledge of which source is writing.

## `create_market_data_source()` (factory.py)

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    return SimulatorDataSource(price_cache=price_cache)
```

This is the entire selection logic: non-empty `MASSIVE_API_KEY` -> real data, otherwise
simulator. Returns an *unstarted* source — the caller must `await source.start(tickers)`.

## The Two Implementations

- **`SimulatorDataSource`** (simulator.py) — wraps a `GBMSimulator`; a 500ms asyncio loop
  calls `.step()` and writes every ticker's new price to the cache. See
  `planning/MARKET_SIMULATOR.md` for the math.
- **`MassiveDataSource`** (massive_client.py) — polls `get_snapshot_all()` on an interval
  (default 15s, matching the free-tier rate limit) via `asyncio.to_thread` (the Massive
  `RESTClient` is synchronous), extracting `last_trade.price` and writing to the cache.
  See `planning/MASSIVE_API.md` for the underlying REST API.

Both do an immediate first update inside `start()` so the cache — and therefore SSE —
has data right away instead of waiting for the first tick/poll.

## Future Extension: EOD Data

Nothing in this interface assumes real-time-only data. If a future feature needs
end-of-day context (e.g. "% change since yesterday's close" instead of "% change since
page load"), it fits without changing the interface shape:

- `MassiveDataSource` could fetch `get_grouped_daily_aggs()` once per day (see
  `planning/MASSIVE_API.md`) and store each ticker's previous close alongside the
  existing `PriceCache`, e.g. a small companion cache or an extra field on `PriceUpdate`.
- `SimulatorDataSource` already has an analogous concept in `SEED_PRICES` (seed_prices.py)
  and could snapshot "price at start of day" the same way.

This is speculative — not implemented, and not required by the current plan (`change_pct`
in `PLAN.md` §13 is explicitly "computed vs. simulator seed price," not a market EOD
close), but the cache/source split means adding it later wouldn't require touching the
`MarketDataSource` ABC.
