# Massive API Reference (formerly Polygon.io)

How FinAlly retrieves real-time and end-of-day stock prices for multiple tickers from Massive. Supersedes `archive/MASSIVE_API.md`, which contained errors (see "Corrections").

Verification legend: **[docs]** read from the official docs at massive.com; **[src]** confirmed in the installed `massive` Python package source (v1.x); **[unverified]** not tested against the live API. No live calls were made while writing this: the research sandbox could not complete TLS to `api.massive.com`. Re-verify the **[unverified]** items with a real key before relying on them.

## 1. Basics

| Item | Value |
|---|---|
| Base URL | `https://api.massive.com` (legacy `https://api.polygon.io` still works in parallel) [docs] |
| Python package | `massive` (`uv add massive`), Python 3.10+ [docs] |
| Auth | `Authorization: Bearer <key>` header, or `?apiKey=<key>` query param [docs] |
| Env var | The client reads `MASSIVE_API_KEY` when `api_key` is not passed [src] |
| Timestamps | **Mixed units** - see section 5 |
| Retries | Client retries 3 times with backoff on 413, 429, 499, 500, 502, 503, 504 [src] |
| Client is sync | `RESTClient` uses urllib3; call it via `asyncio.to_thread` from async code |

## 2. Plans, Recency and Rate Limits

| Plan | Data recency | History | Snapshots | Last trade endpoint |
|---|---|---|---|---|
| Stocks Basic (free) | End-of-day | 2 years | **Not included** | Not included |
| Stocks Starter | 15-min delayed | 5 years | Included | Not included |
| Stocks Developer | 15-min delayed | 10 years | Included | Included (delayed) |
| Stocks Advanced ($199/mo) | Real-time | All | Included | Included |
| Stocks Business | Real-time | All | Included | Not included (Business + Expansion only) |

All aggregate endpoints (bars, grouped daily, daily open/close, previous day) are included on every plan, with the recency above [docs].

**Rate limits** [docs]: Basic allows **5 requests/minute per asset class**; paid plans have no request limit for the subscribed asset class. Exceeding it returns HTTP 429 `{"status":"ERROR","error":"You've exceeded the maximum requests per minute..."}`. HTTP 403 `NOT_AUTHORIZED` means the plan does not cover the endpoint or the data recency requested.

**Consequence for FinAlly:** on the free tier there is no live price feed at all. The best available is the previous session close. Real (delayed or live) polling needs Starter or above. See `MARKET_INTERFACE.md` for how the design handles both.

## 3. Endpoints for Multiple Tickers

### 3.1 Full Market Snapshot - live prices, many tickers, one call (primary)

`GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT,TSLA` [docs]

- `tickers`: comma-separated, **case-sensitive**; omit for all US stocks. `include_otc` defaults to false.
- Access: Starter and above (delayed on Starter/Developer, real-time on Advanced/Business).

REST response:

```json
{
  "status": "OK",
  "count": 1,
  "tickers": [{
    "ticker": "AAPL",
    "day":     {"o": 119.62, "h": 120.53, "l": 118.81, "c": 120.42, "v": 28727868, "vw": 119.72},
    "prevDay": {"o": 117.19, "h": 119.63, "l": 116.44, "c": 119.49, "v": 110597265, "vw": 118.50},
    "min":     {"o": 120.43, "h": 120.46, "l": 120.37, "c": 120.42, "v": 270796, "av": 28724441, "n": 762, "t": 1684428720000},
    "lastTrade": {"p": 120.47, "s": 236, "x": 10, "c": [14, 41], "t": 1605195918306274000},
    "lastQuote": {"p": 120.46, "s": 8, "P": 120.47, "S": 4, "t": 1605195918507251700},
    "todaysChange": 0.98,
    "todaysChangePerc": 0.82,
    "updated": 1605195918306274000
  }]
}
```

`lastTrade` and `lastQuote` appear only if the plan includes trades/quotes, so code must not assume they exist. Fallback order for "current price": `lastTrade.p`, then `min.c`, then `day.c`, then `prevDay.c`.

Python client [src]:

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key="...")          # or rely on MASSIVE_API_KEY
snaps = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "MSFT", "TSLA"],
)
for s in snaps:                              # list[TickerSnapshot]
    price = s.last_trade.price if s.last_trade else s.min.close
    prev_close = s.prev_day.close
    print(s.ticker, price, s.todays_change_percent)
```

### 3.2 Unified Snapshot (alternative)

`GET /v3/snapshot?ticker.any_of=AAPL,MSFT&limit=250` [docs]. Up to 250 tickers, paginated via `next_url`, response has `results[]` with `session` (`price`, `change`, `change_percent`, `previous_close`, ...), `last_trade`, `last_quote`, `last_minute`, `market_status` (`open`/`closed`/`early_trading`/`late_trading`), and per-ticker `error: "NOT_FOUND"` entries for unknown symbols. Same plan access as 3.1. Client: `client.list_universal_snapshots(ticker_any_of=[...])` [src]. Its per-ticker `NOT_FOUND` errors make it the better choice for **validating unknown tickers**, if 3.1 is found to silently drop them [unverified].

### 3.3 Grouped Daily Bars - end-of-day, every ticker, one call (free-tier path)

`GET /v2/aggs/grouped/locale/us/market/stocks/{date}` [docs]

- One request returns OHLCV for **all** US stocks for that date (thousands of rows). Filter client-side.
- `date` is `YYYY-MM-DD`. Non-trading days are expected to return `resultsCount: 0` [unverified].
- Included on all plans. On Basic, whether today's date is queryable before the session closes is [unverified]; ask for the previous day or earlier to be safe.

```json
{"status": "OK", "adjusted": true, "resultsCount": 3, "queryCount": 3,
 "results": [{"T": "AAPL", "o": 115.55, "h": 117.59, "l": 114.13, "c": 115.97,
              "v": 131704427, "vw": 116.3058, "n": 4966, "t": 1605042000000}]}
```

`t` is the Unix **millisecond** timestamp for the end of the window. Client [src]:

```python
bars = client.get_grouped_daily_aggs(date="2026-10-28")   # list[GroupedDailyAgg]
closes = {b.ticker: b.close for b in bars if b.ticker in {"AAPL", "MSFT"}}
```

### 3.4 Previous Day Bar - one ticker

`GET /v2/aggs/ticker/{ticker}/prev` [docs]. Returns the last completed session's OHLCV in `results[0]` (`o h l c v vw t`). Included on all plans, but costs one request per ticker, so on the free tier 10 tickers consume 2 minutes of quota.

```python
prev = client.get_previous_close_agg(ticker="AAPL")   # list[PreviousCloseAgg]
print(prev[0].close)
```

### 3.5 Custom Bars (historical OHLCV, charts)

`GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}` [docs]

- `timespan`: `minute`, `hour`, `day`, `week`, `month`, `quarter`, `year`. `from`/`to`: `YYYY-MM-DD` or millisecond timestamps. `adjusted` (default true), `sort` (`asc`/`desc`), `limit` (default 5000, max 50000). Bars are in Eastern Time; intervals with no trades return no bar. Paginated via `next_url`. Bar `t` is the window **start**, in milliseconds.

```python
bars = list(client.list_aggs("AAPL", 1, "day", "2026-09-01", "2026-10-28", limit=50000))
for b in bars:
    print(b.timestamp, b.open, b.high, b.low, b.close, b.volume)
```

`list_*` methods auto-paginate (`RESTClient(pagination=False)` disables it) [docs].

### 3.6 Daily Ticker Summary

`GET /v1/open-close/{ticker}/{date}` [docs]. Returns `open high low close volume preMarket afterHours` for one ticker and day. Useful for after-hours close; one call per ticker.

### 3.7 Last Trade

`GET /v2/last/trade/{ticker}` [docs]: `results.p` price, `results.t` SIP timestamp in **nanoseconds**. Developer and Advanced plans only. Not used by FinAlly: one call per ticker, and the snapshot already carries the last trade.

### 3.8 Not used

WebSocket (`wss://socket.massive.com/stocks`, Advanced/Business only) gives true streaming but contradicts the plan's REST-polling choice. Revisit only if sub-second real data becomes a requirement.

## 4. Raw HTTP Example (no SDK)

```python
import httpx

r = httpx.get(
    "https://api.massive.com/v2/snapshot/locale/us/markets/stocks/tickers",
    params={"tickers": "AAPL,MSFT"},
    headers={"Authorization": "Bearer " + api_key},
    timeout=10,
)
r.raise_for_status()
for t in r.json()["tickers"]:
    print(t["ticker"], t.get("lastTrade", {}).get("p") or t["min"]["c"])
```

## 5. Field and Timestamp Cheat Sheet

| REST key | `massive` model attribute | Unit / note |
|---|---|---|
| `lastTrade.p` | `snap.last_trade.price` | dollars |
| `lastTrade.t` | `snap.last_trade.sip_timestamp` | **nanoseconds** [src] |
| `updated` | `snap.updated` | nanoseconds |
| `min.t`, aggregate `t` | `.timestamp` | milliseconds |
| `day.c` / `prevDay.c` | `snap.day.close` / `snap.prev_day.close` | dollars |
| `todaysChange` / `todaysChangePerc` | `snap.todays_change` / `snap.todays_change_percent` | vs previous close |
| grouped agg `T` | `bar.ticker` | symbol |

## 6. Errors

| Status | Meaning | FinAlly response |
|---|---|---|
| 401 | Missing/invalid key | Log clearly at startup; no retry |
| 403 `NOT_AUTHORIZED` | Plan lacks endpoint or recency | Fall back to the end-of-day path (see `MARKET_INTERFACE.md`) |
| 404 | Unknown resource | Treat the ticker as unknown |
| 429 | Basic tier rate limit | Lengthen the poll interval; client already retried 3 times |
| 5xx | Server error | Keep last cached prices; retry next cycle |

## 7. Corrections to the Archived Doc

1. `day.previous_close` does not exist. Previous close is `prev_day.close` (REST: `prevDay.c`) [src].
2. `last_trade.timestamp` does not exist on the snapshot model. It is `sip_timestamp`, in nanoseconds, not milliseconds [src].
3. The snapshot endpoint is **not** available on the free tier, contrary to the archived "free tier: poll every 15s" guidance [docs].
4. Rate limits are per asset class and apply only to the free tier; paid tiers are unlimited [docs].
## Sources

- [Full Market Snapshot](https://massive.com/docs/rest/stocks/snapshots/full-market-snapshot)
- [Single Ticker Snapshot](https://massive.com/docs/rest/stocks/snapshots/single-ticker-snapshot)
- [Unified Snapshot](https://massive.com/docs/rest/stocks/snapshots/unified-snapshot)
- [Daily Market Summary](https://massive.com/docs/rest/stocks/aggregates/daily-market-summary)
- [Custom Bars](https://massive.com/docs/rest/stocks/aggregates/custom-bars)
- [Previous Day Bar](https://massive.com/docs/rest/stocks/aggregates/previous-day-bar)
- [Daily Ticker Summary](https://massive.com/docs/rest/stocks/aggregates/daily-ticker-summary)
- [Last Trade](https://massive.com/docs/rest/stocks/trades-quotes/last-trade)
- [REST quickstart (auth)](https://massive.com/docs/rest/quickstart)
- [Request limits](https://massive.com/knowledge-base/article/what-is-the-request-limit-for-massives-restful-apis)
- [Python client](https://github.com/massive-com/client-python)
- [Pricing](https://massive.com/pricing)
