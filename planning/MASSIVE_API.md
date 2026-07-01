# Massive API Reference (formerly Polygon.io)

Research notes and code examples for the Massive REST API, covering both real-time
and end-of-day (EOD) price retrieval for multiple tickers. This supersedes
`planning/archive/MASSIVE_API.md`, which was written before the Polygon.io -> Massive
rebrand completed.

## Overview

- On 2025-10-30, Polygon.io rebranded to **Massive**. Docs now live at `massive.com/docs`.
- **Base URL**: `https://api.massive.com` — legacy `https://api.polygon.io` still works
  (existing API keys and integrations continue unmodified).
- **Python package**: `massive` (was `polygon-api-client`). Install with `pip install -U massive`
  or `uv add massive`.
- **Min Python version**: 3.9+
- **Auth**: API key via `MASSIVE_API_KEY` env var or `RESTClient(api_key=...)`. The client
  sends it as a bearer token automatically.
- **GitHub**: `massive-com/client-python` (the `polygon-io/client-python` repo now redirects here).

```python
from massive import RESTClient

client = RESTClient(api_key="your_key_here")
```

## Rate Limits

| Tier | Limit |
|------|-------|
| Free | 5 requests/minute |
| Paid (all tiers) | Unlimited request count (fair-use expected) |

Confirmed current via Massive's own knowledge base (see Sources). Matches what this
project already assumes: free tier polls every 15s, paid tiers poll every 2-15s.

## Real-Time: Snapshot — All Tickers (Multiple Tickers, One Call)

This is the endpoint this project's `MassiveDataSource` already uses
(`backend/app/market/massive_client.py`).

**REST**: `GET /v2/snapshot/locale/us/markets/stocks/tickers`

**Query params**:
- `tickers` — comma-separated, case-sensitive list, e.g. `AAPL,GOOGL,MSFT`. Empty/omitted
  defaults to *all* tickers (10,000+) — always pass an explicit list for a watchlist.
- `include_otc` — bool, default `false`.

**Python client**:
```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient()

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],
)

for snap in snapshots:
    print(f"{snap.ticker}: ${snap.last_trade.price}")
    print(f"  Day change: {snap.todays_change_percent}%")
```

**Response shape** (per ticker — field names per current docs; the Python client exposes
these as snake_case attributes, e.g. `last_trade`, `prev_day`, `todays_change_percent`):
```json
{
  "ticker": "AAPL",
  "day": { "o": 129.61, "h": 130.15, "l": 125.07, "c": 125.07, "v": 111237700, "vw": 127.35 },
  "prevDay": { "o": ..., "h": ..., "l": ..., "c": ..., "v": ..., "vw": ... },
  "lastTrade": { "p": 125.07, "s": 100, "t": 1675190399000 },
  "lastQuote": { "p": 125.06, "P": 125.08, "s": 500, "S": 1000, "t": 1675190399500 },
  "min": { "...": "most recent minute bar" },
  "fmv": 125.05,
  "todaysChange": -4.54,
  "todaysChangePerc": -3.5,
  "updated": 1675190399000
}
```

**Notes**:
- Snapshot data resets daily at ~3:30 AM ET and repopulates from ~4:00 AM ET as exchanges
  report.
- 15-minute delayed on Starter/Developer plans; real-time on Advanced/Business plans.
- `fmv` (fair market value) is a Business-plan-only field.

### Single Ticker Snapshot

For a detail view of one ticker:
```python
snapshot = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)
print(f"Price: ${snapshot.last_trade.price}")
```

## End-of-Day: Grouped Daily Bars (Multiple Tickers, One Call)

This is the efficient way to get EOD prices for many tickers — **one API call returns
every U.S. stock's OHLCV for a given date**, rather than one `prev` call per ticker.
The current implementation does not use this endpoint yet; it's documented here for a
future EOD/historical feature (e.g. seeding "change since yesterday's close" or a daily
recap).

**REST**: `GET /v2/aggs/grouped/locale/us/market/stocks/{date}`

**Query params**:
- `date` — `YYYY-MM-DD` (required)
- `adjusted` — bool, default `true` (split-adjusted)
- `include_otc` — bool, default `false`

**Python client**:
```python
grouped = client.get_grouped_daily_aggs(
    date="2026-06-30",
    adjusted=True,
)

for bar in grouped:
    print(f"{bar.ticker}: open={bar.open} close={bar.close} volume={bar.volume}")
```

Signature: `get_grouped_daily_aggs(date, adjusted=None, locale='us', market_type='stocks', include_otc=False, params=None, raw=False)`

**Response** (raw JSON — the client maps these onto an `Agg` object with `ticker`, `open`,
`high`, `low`, `close`, `volume`, `vwap`, `transactions`, `timestamp`):
```json
{
  "adjusted": true,
  "queryCount": 3,
  "resultsCount": 3,
  "status": "OK",
  "results": [
    { "T": "AAPL", "o": 190.1, "h": 192.4, "l": 189.0, "c": 191.8, "v": 54000000, "vw": 190.9, "n": 512000, "t": 1751241600000 }
  ]
}
```
Field key: `T`=ticker, `o`/`h`/`l`/`c`=OHLC, `v`=volume, `vw`=VWAP, `n`=transaction count,
`t`=Unix ms timestamp for the end of the aggregate window.

Historical coverage goes back to 2003-09-10; how far back you can query depends on plan
tier (2 years on Basic, full history on Advanced/Business).

**Why this over per-ticker previous-close**: for a 10-ticker watchlist the difference is
minor, but this endpoint costs the *same one request* whether you need 1 ticker's EOD
data or all 10,000 — so it's the right choice the moment EOD is needed for more than a
handful of tickers, and it avoids burning the free tier's 5 req/min budget on N separate
calls.

## End-of-Day: Previous Close (Single Ticker)

Simpler when you only need one ticker's prior close (e.g. a seed price).

**REST**: `GET /v2/aggs/ticker/{stocksTicker}/prev`

**Python client**:
```python
prev = client.get_previous_close_agg(ticker="AAPL", adjusted=True)
for agg in prev:
    print(f"Previous close: ${agg.close}")
```

**Response**:
```json
{
  "adjusted": true,
  "ticker": "AAPL",
  "queryCount": 1,
  "resultsCount": 1,
  "status": "OK",
  "results": [
    { "T": "AAPL", "c": 115.97, "h": 117.59, "l": 114.13, "o": 115.55, "t": 1605042000000, "v": 131704427, "vw": 116.3058 }
  ]
}
```

## Historical Aggregates (Bars)

Date-range OHLCV bars for one ticker — useful for chart history, not for live polling.

**REST**: `GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}`

```python
aggs = []
for a in client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2026-01-01",
    to="2026-01-31",
    limit=50000,
):
    aggs.append(a)
```

## Last Trade / Last Quote (Single Ticker)

```python
trade = client.get_last_trade(ticker="AAPL")
quote = client.get_last_quote(ticker="AAPL")
```

## Error Handling

- **401**: Invalid API key
- **403**: Plan doesn't include this endpoint/data recency tier
- **429**: Rate limit exceeded (free tier: 5 req/min)
- **5xx**: Server errors — client retries a few times by default

## What's Verified vs. Carried Over

- **Verified against current `massive.com/docs`**: full-market snapshot endpoint/params/response
  shape, grouped-daily endpoint/params/response shape, previous-close endpoint/response shape,
  package rename (`polygon-api-client` -> `massive`), free-tier rate limit (5 req/min).
- **Verified against this repo's working code** (`backend/app/market/massive_client.py`,
  tested and passing): `get_snapshot_all(market_type=SnapshotMarketType.STOCKS, tickers=[...])`
  is a real, working method call today.
- **Carried over from the legacy Polygon.io docs, not independently re-verified**: exact
  `list_aggs`, `get_last_trade`, `get_last_quote`, `get_snapshot_ticker` signatures and the
  historical-aggregates response shape. These are long-standing, stable endpoints unlikely
  to have changed in the rebrand, but treat field-level details as best-effort.
- **Not found / inconclusive**: no official maximum-tickers-per-snapshot-request number
  was locatable in the docs surfaced during this research; if that matters, batch requests
  defensively or confirm directly against the current plan's docs before relying on it.

## Sources

- [Overview | Stocks REST API - Massive](https://massive.com/docs/rest/stocks/overview)
- [Full Market Snapshot | Stocks REST API - Massive](https://massive.com/docs/rest/stocks/snapshots/full-market-snapshot)
- [Daily Market Summary (OHLC) | Stocks REST API - Massive](https://massive.com/docs/rest/stocks/aggregates/daily-market-summary)
- [Previous Day Bar (OHLC) | Stocks REST API - Massive](https://massive.com/docs/rest/stocks/aggregates/previous-day-bar)
- [What is the request limit for Massive's RESTful APIs?](https://massive.com/knowledge-base/article/what-is-the-request-limit-for-massives-restful-apis)
- [GitHub - massive-com/client-python](https://github.com/massive-com/client-python)
- [client-python/docs/source/Aggs.rst](https://github.com/polygon-io/client-python/blob/master/docs/source/Aggs.rst)
- [PyPI - polygon-api-client](https://pypi.org/project/polygon-api-client/) (predecessor package, for migration context)
