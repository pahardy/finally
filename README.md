# FinAlly

AI-powered trading workstation: live streaming prices, a simulated $10,000 portfolio, and an LLM chat assistant that can analyze positions and execute trades.

Built entirely by coding agents as the capstone of an agentic AI coding course.

## Features

- Live prices over SSE with green/red flash animations and sparklines
- Market-order trading with a simulated portfolio, heatmap, P&L chart and positions table
- Watchlist management, manual or via chat
- AI assistant (LiteLLM, OpenRouter, Cerebras) with structured outputs and auto-executed trades

## Architecture

Single Docker container on port 8000.

- **Frontend:** Next.js + TypeScript, static export served by FastAPI
- **Backend:** FastAPI managed with `uv`
- **Database:** SQLite, persisted in a named Docker volume
- **Market data:** built-in GBM simulator by default, Massive API if `MASSIVE_API_KEY` is set

## Configuration

Copy `.env.example` to `.env`:

| Variable | Purpose |
|---|---|
| `OPENROUTER_API_KEY` | LLM chat (not needed when `LLM_MOCK=true`) |
| `MASSIVE_API_KEY` | Optional real market data; simulator used if empty |
| `LLM_MOCK` | `true` for deterministic mock chat responses |

## Status

| Component | State |
|---|---|
| Market data (simulator, Massive client, price cache, SSE) | Done, see `backend/` |
| Portfolio, watchlist, chat API | Planned |
| Frontend | Planned |
| Docker, start/stop scripts, E2E tests | Planned |

## Development

```bash
cd backend
uv sync
uv run pytest
uv run market_data_demo.py
```

## Docs

The full spec is in [`planning/PLAN.md`](planning/PLAN.md). Market data details are in [`planning/MARKET_DATA_SUMMARY.md`](planning/MARKET_DATA_SUMMARY.md).

## License

See [LICENSE](LICENSE).
