# Review

Base reviewed: `HEAD` (`02f990c` - `add files`)

Scope reviewed: untracked change `planning/MARKET_SIMULATOR.md`.

## Findings

### Low: simulator test count is stale

- `planning/MARKET_SIMULATOR.md:133` says `test_simulator.py` has 17 tests.
- `backend/tests/market/test_simulator.py` currently contains 18 `test_*` methods.
- The tracked summary doc has the same stale count at `planning/MARKET_DATA_SUMMARY.md:53`, but the new simulator doc repeats it in an "as-built" section.

Impact: this makes the new documentation inaccurate on arrival and can mislead future coverage/test-suite summaries. Update the simulator doc to say 18 tests, and consider correcting the existing summary doc in the same follow-up.

### Low: TSLA correlation group description contradicts the implementation

- `planning/MARKET_SIMULATOR.md:70` says `TSLA_CORR` applies because TSLA is in the `"tech"` set but excluded from that correlation.
- The actual seed data excludes TSLA from the tech group: `backend/app/market/seed_prices.py:38-40` lists `"tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"}`.
- The new doc also shows the same TSLA-free group at `planning/MARKET_SIMULATOR.md:63-66`, so the comment is internally inconsistent.

Impact: future readers may incorrectly think TSLA is a member of `CORRELATION_GROUPS["tech"]`, or may change the seed data to match the prose rather than the current simulator behavior. The doc should instead say that TSLA is not in a sector correlation group and `_pairwise_correlation()` pins any TSLA pair to `TSLA_CORR`.

### Low: Determinism claim omits the global RNG dependency

- `planning/MARKET_SIMULATOR.md:125-126` describes `GBMSimulator` as deterministic given a seeded RNG.
- The implementation uses two process-global RNGs: `np.random.standard_normal()` in `backend/app/market/simulator.py:84` and Python `random` for unknown seed prices and shock events in `backend/app/market/simulator.py:105-107` and `backend/app/market/simulator.py:151`.

Impact: seeding only one RNG, or sharing either global RNG with other code/tests, will not make simulator paths reproducible. The doc should qualify this as deterministic only when both `numpy.random` and `random` are seeded and no interleaved consumers advance those global RNG states.

## Notes

- `planning/MARKET_SIMULATOR.md:108` says unknown ticker seed prices are in `[50, 300)`, while `random.uniform(50.0, 300.0)` in `backend/app/market/simulator.py:151` is better documented as approximately `[50, 300]` for practical purposes. This is minor, but worth tightening while editing the doc.

## Validation

- Ran `git status --porcelain=v1 -uall`: only `planning/MARKET_SIMULATOR.md` was untracked before this review file was updated.
- Ran `git diff --stat HEAD`: no tracked code or doc modifications before this review file was updated.
- Reviewed `planning/MARKET_SIMULATOR.md` against `backend/app/market/simulator.py`, `backend/app/market/seed_prices.py`, `backend/tests/market/test_simulator.py`, `backend/tests/market/test_simulator_source.py`, and `planning/MARKET_DATA_SUMMARY.md`.
- Counted test methods with `rg -n "def test_" backend/tests/market/test_simulator.py backend/tests/market/test_simulator_source.py`: 18 simulator unit tests and 10 async wrapper tests.
- Test execution did not complete: `python3 -m pytest ...` failed because `pytest` is not installed in the system Python, and `UV_CACHE_DIR=/Users/patrickhardy/Documents/projects/finally/.uv-cache uv run pytest ...` could not resolve missing packages because network access is blocked.
