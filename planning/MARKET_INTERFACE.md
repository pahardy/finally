# Market Data Interface

The unified Python API for stock prices in FinAlly. One interface, two implementations: the Massive REST poller when `MASSIVE_API_KEY` is set and non-empty, otherwise the built-in simulator. Supersedes `archive/MARKET_INTERFACE.md`.

Related: `MASSIVE_API.md` (provider facts and corrections), `MARKET_SIMULATOR.md` (simulator), `MARKET_DATA_SUMMARY.md` (what is built today).

## 1. Design in One Paragraph

A single background task (simulator or Massive poller) **writes** to an in-memory `PriceCache`. Everything else - SSE, trades, portfolio valuation, watchlist - **reads only from the cache** and never calls a provider. Downstream code is therefore source-agnostic, and swapping sources changes nothing outside `app/market/`. This is already built and tested; this document adds the changes needed to make the Massive path correct (section 6).

```
create_market_data_source(cache)
  MASSIVE_API_KEY set?  yes -> MassiveDataSource   (REST polling)
                        no  -> SimulatorDataSource (GBM)
            |
            v  writes
        PriceCache  <--- reads --- SSE stream, /api/portfolio, trade(), watchlist
```

## 2. Public API

Import from `app.market`:

```python
from app.market import PriceCache, PriceUpdate, MarketDataSource, create_market_data_source
```

### 2.1 Startup and shutdown

```python
cache = PriceCache()
source = create_market_data_source(cache)     # reads MASSIVE_API_KEY
await source.start(["AAPL", "GOOGL", ...])    # seeds the cache before returning
...
await source.stop()
```

### 2.2 Reading prices (the only way downstream code gets a price)

```python
cache.get("AAPL")        # PriceUpdate | None
cache.get_price("AAPL")  # float | None  - used by trade execution
cache.get_all()          # dict[str, PriceUpdate] - used by SSE and valuation
cache.version            # int, bumped on every update - SSE change detection
```

A `None` result means the ticker is not tracked (or has no price yet). Trade execution treats `None` as an unknown ticker.

### 2.3 Changing the tracked set

```python
await source.add_ticker("PYPL")     # raises UnknownTickerError if it cannot be priced
await source.remove_ticker("PYPL")  # also removes it from the cache
source.get_tickers()                # list[str]
source.mode                         # "simulated" | "live" | "eod"
```

The tracked set is the **union of watchlist and position tickers** (PLAN section 6). The caller (watchlist/portfolio service) owns that union: it calls `add_ticker` when a ticker enters either set and `remove_ticker` only when it leaves both.

`UnknownTickerError(ValueError)` is defined in `interface.py`. The API layer maps it to `404 {"error": "Unknown ticker: XYZ"}` and adds nothing to the watchlist. Both sources must raise it, per PLAN:

| Source | Rule |
|---|---|
| Simulator | Accepts any `^[A-Z]{1,5}$`; otherwise raises |
| Massive | Raises if a one-ticker lookup returns no price (section 5.4) |

### 2.4 Data model

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float     # prior cache value; drives the flash animation
    timestamp: float          # Unix seconds
    open_price: float         # first price seen since backend start (see 6, item 4)

    change, change_percent    # vs previous_price (tick-to-tick)
    day_change_percent        # vs open_price - the "daily change %" shown in the watchlist
    direction                 # "up" | "down" | "flat"
    to_dict()                 # SSE / REST payload
```

`PriceCache.update(ticker, price, timestamp=None)` computes `previous_price` and `open_price` itself; sources only supply price and timestamp. All timestamps in the cache are Unix **seconds** (convert at the provider boundary - Massive uses ms and ns).

### 2.5 Interface contract

```python
class MarketDataSource(ABC):
    mode: str
    async def start(self, tickers: list[str]) -> None: ...   # once; cache populated on return
    async def stop(self) -> None: ...                        # idempotent
    async def add_ticker(self, ticker: str) -> None: ...     # idempotent; may raise UnknownTickerError
    async def remove_ticker(self, ticker: str) -> None: ...  # idempotent
    def get_tickers(self) -> list[str]: ...
```

## 3. Source Selection

```python
def create_market_data_source(price_cache):
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache,
                                 poll_interval=float(os.environ.get("MASSIVE_POLL_INTERVAL", 15)))
    return SimulatorDataSource(price_cache=price_cache)
```

Optional env var `MASSIVE_POLL_INTERVAL` (seconds). Defaults: 15s. Guidance: free tier 15s or more (5 req/min), Starter/Developer 5s (data is 15-min delayed, so faster adds nothing), Advanced 2s.

## 4. Simulator Source

Unchanged except `add_ticker` validation and `mode = "simulated"`. Full detail in `MARKET_SIMULATOR.md`.

## 5. Massive Source

### 5.1 Modes

`MASSIVE_API_KEY` alone does not tell us the plan, and the snapshot endpoint is not on the free plan (`MASSIVE_API.md` section 2). The source therefore detects its mode on `start()`:

1. Try one snapshot call for the tracked tickers. Success -> `mode = "live"`.
2. If the call fails with a `BadResponse` whose message contains `NOT_AUTHORIZED` -> `mode = "eod"`, log one warning naming the plan limitation. (The client's `BadResponse` carries only the response body, not the status code, so match on the body text.)
3. Any other failure at startup (401, network) -> `start()` raises, so a bad key fails loudly instead of silently serving nothing.

| Mode | Endpoint | Cadence | Prices |
|---|---|---|---|
| `live` | Snapshot, all tickers in one call | `poll_interval` | Last trade, delayed 15 min on Starter/Developer, real-time on Advanced/Business |
| `eod` | Grouped daily bars, one call for all tickers | every 15 minutes | Previous session close; static during the day |

`mode` is exposed on `/api/health` so the UI can show a "delayed" or "end-of-day" badge instead of implying live data.

### 5.2 Live polling (snapshot)

```python
def _fetch_live(self) -> list:
    return self._client.get_snapshot_all(
        market_type=SnapshotMarketType.STOCKS, tickers=self._tickers)

@staticmethod
def _snapshot_price(snap) -> tuple[float, float] | None:
    """Return (price, unix_seconds) using the first available source."""
    if snap.last_trade and snap.last_trade.price:
        return snap.last_trade.price, snap.last_trade.sip_timestamp / 1e9
    if snap.min and snap.min.close:
        return snap.min.close, snap.min.timestamp / 1e3
    if snap.day and snap.day.close:
        return snap.day.close, snap.updated / 1e9
    if snap.prev_day and snap.prev_day.close:
        return snap.prev_day.close, time.time()
    return None
```

The fallback chain exists because `last_trade` is returned only when the plan includes trades (`MASSIVE_API.md` 3.1). Tickers returning `None` are skipped and keep their last cached price.

### 5.3 End-of-day polling (free tier)

```python
def _fetch_eod(self) -> dict[str, float]:
    """Closing prices for the most recent trading day (one API call per day tried)."""
    day = date.today()
    for _ in range(7):                      # skip weekends and holidays
        day -= timedelta(days=1)
        bars = self._client.get_grouped_daily_aggs(date=day.isoformat())
        if bars:
            wanted = set(self._tickers)
            return {b.ticker: b.close for b in bars if b.ticker in wanted}
    return {}
```

Worst case 7 calls on a cold start (a long holiday run), which fits inside the free 5/min budget only if spread out; the loop sleeps 15s between failed days. After a hit the source remembers the date and refreshes every 15 minutes. Every API call runs in `asyncio.to_thread`.

### 5.4 Ticker validation

`add_ticker` on a Massive source does an immediate one-ticker lookup (`get_snapshot_ticker` in live mode, `get_previous_close_agg` in eod mode) and raises `UnknownTickerError` when no price comes back. Until verified with a real key, assume the failure may surface either as an empty result or as a `BadResponse` containing `NOT_FOUND`; treat both as unknown. Unified Snapshot (`MASSIVE_API.md` 3.2) returns an explicit per-ticker `NOT_FOUND` and is the fallback if the simple lookup proves ambiguous.

### 5.5 Failure behavior

- Poll errors (429, 5xx, network) are logged and swallowed; the cache keeps the last price and the next cycle retries. The client has already retried 3 times with backoff.
- On 429 in live mode, double the interval up to 60s, then restore on the next success.
- Never write a zero or `None` price to the cache.

## 6. Required Changes to Existing Code

Found while researching; none are made yet. Items 1-2 are real defects hidden by mocked tests.

| # | File | Problem | Fix |
|---|---|---|---|
| 1 | `massive_client.py` `_poll_once` | Reads `snap.last_trade.timestamp`, which does not exist (it is `sip_timestamp`). The `AttributeError` is caught by `except (AttributeError, TypeError)`, so **every ticker is skipped and the cache stays empty**. Tests pass because they mock the attribute. | Use `_snapshot_price` (5.2) |
| 2 | `massive_client.py` | Divides the trade timestamp by 1000 (treats it as ms). It is nanoseconds. | Divide by `1e9` |
| 3 | `massive_client.py` | Assumes snapshots work on every plan; on the free tier every poll fails with 403 and the app shows no prices. | Mode detection (5.1) and eod path (5.3) |
| 4 | `cache.py`, `models.py` | No `open_price`; PLAN requires daily change vs first price seen since startup. | Store `open_price` on first update, keep it across updates, add `day_change_percent` |
| 5 | `interface.py` and both sources | No ticker validation; PLAN requires 404 for unknown tickers. | `UnknownTickerError` (2.3) |
| 6 | `massive_client.py` tests | Mock objects expose attributes the real model lacks. | Build test snapshots from the real `massive.rest.models.TickerSnapshot.from_dict` using the REST JSON in `MASSIVE_API.md` |
| 7 | `factory.py` | Poll interval hard-coded. | `MASSIVE_POLL_INTERVAL` (section 3) |
| 8 | `/api/health` | No indication of data quality. | Include `market_mode` |

## 7. Integration Points

| Consumer | Uses |
|---|---|
| `GET /api/stream/prices` | `cache.version` to detect change, `cache.get_all()`, `PriceUpdate.to_dict()` every ~500ms. In `live` and `eod` modes updates arrive every 15s or less; the stream simply repeats unchanged values. |
| `trade()` | `cache.get_price(ticker)`; `None` -> 404 unknown ticker |
| `GET /api/portfolio` | `get_price` per position for total value |
| Watchlist add | `await source.add_ticker(t)` first; on `UnknownTickerError` return 404 and do not insert |
| Watchlist remove | `remove_ticker` only if the ticker has no open position |
| App lifespan | create cache and source, `start()` with watchlist plus position tickers, `stop()` on shutdown |

## 8. Testing

- Simulator: existing tests plus `UnknownTickerError` for invalid symbols.
- Massive: parse the REST JSON fixtures from `MASSIVE_API.md` through `TickerSnapshot.from_dict`, assert the price chain (5.2), ns-to-seconds conversion, missing `lastTrade`, mode fallback on a `BadResponse("... NOT_AUTHORIZED ...")`, and eod date walk-back across a weekend.
- Contract: one parametrized test running the same lifecycle (`start`, `add_ticker`, `remove_ticker`, `stop`, double `stop`) against both sources with the provider client stubbed.
- Manual, with a real key before declaring Massive done: confirm the plan's mode, an unknown ticker's failure shape, and a grouped-daily call for a recent date.
