# Review of changes since HEAD

Reviewed the tracked changes in `README.md` and `planning/PLAN.md`, plus the new marketplace manifest, plugin manifest, and stop hook in `.claude-plugin/` and `independent-reviewer/`. The existing untracked review report was checked against the source and refreshed. Findings are ordered by priority. Application implementation files are unchanged.

## Findings

### [P2] Install the development extra before running pytest

**Location:** `README.md:46–47`

The documented sequence runs `uv sync` followed by `uv run pytest`, but pytest and pytest-asyncio are declared in `[project.optional-dependencies].dev` in `backend/pyproject.toml`. Neither command requests that extra. On a clean environment the test runner and async plugin are therefore unavailable, so the advertised test command fails or depends on unrelated globally installed tools. Use `uv sync --extra dev` and `uv run --extra dev pytest`, or move these dependencies into a default development group.

### [P2] Schedule the cache changes required for daily change percentages

**Location:** `planning/PLAN.md:168`

The new rule says the cache stores `open_price` and supports change relative to the first observed price, but `PriceCache` and `PriceUpdate` retain only the latest and previous prices. The SSE payload's `change_percent` is calculated against the previous tick. For prices 100 → 110 → 121, that calculation yields 10%, whereas the new baseline rule requires 21%. The build order calls market data complete and does not schedule this extension, leaving the frontend without the promised baseline. Add explicit cache/model/SSE work and distinguish daily change from tick change, or revise the rule to match the existing contract.

### [P2] Plan validation before promising rejection of unknown Massive symbols

**Location:** `planning/PLAN.md:167`

The new rule promises that the Massive client rejects unpriceable symbols and nothing is added. However, `MassiveDataSource.add_ticker()` appends a symbol without fetching or validating it. Prices arrive on a later poll; polling failures are logged and suppressed. The API cannot rely on this method to distinguish an unknown symbol from a pending price or an upstream failure. The final review decisions also say to keep the Massive client as is. Schedule explicit symbol validation and error propagation, distinguishing upstream failures from unknown symbols, or revise this guarantee.

## Verification and limits

- Inspected the complete diff against HEAD (no staged changes), all three new plugin files, dependency declarations, demo entry point, and market cache/model/provider/SSE implementation.
- `git diff --check` passed. All three plugin files parsed successfully as JSON.
- Inspected local `codex exec --help`; did not run the stop hook or launch a nested review.
- The percentage and ticker-validation findings are based on source inspection; no application code was executed.
- Did not install dependencies, run the full test suite, execute the demo, or call live providers. These changes modify documentation and review automation rather than application implementation.
- The README's reference to a missing `.env.example` already existed in HEAD, so it is not reported as a newly introduced issue.
