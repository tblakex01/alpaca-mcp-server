# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Alpaca MCP Server v2 — auto-generates MCP tools from Alpaca's bundled OpenAPI specs using FastMCP's `from_openapi()`. Exposes 61 tools for trading, market data, and account management.

## Commands

```bash
# Install (use uv, not pip)
uv pip install -e ".[dev]"

# Core tests — no network, no credentials needed
pytest tests/test_integrity.py tests/test_server_construction.py -v

# Integration tests — requires paper API credentials
ALPACA_API_KEY=... ALPACA_SECRET_KEY=... pytest tests/ -m integration -v

# Run a single test
pytest tests/test_integrity.py::test_all_tool_names_unique -v

# Lint
ruff check .

# Type check
mypy .

# Sync OpenAPI specs from Alpaca
./scripts/sync-specs.sh
```

## Architecture

**Tool generation pipeline:**
1. OpenAPI specs (`src/alpaca_mcp_server/specs/`) are loaded at process init
2. `toolsets.py` defines which operationIds are exposed, grouped into 8 toolsets (account, trading, watchlists, assets, stock-data, crypto-data, options-data, corporate-actions)
3. `names.py` maps operationIds to snake_case tool names and curated descriptions
4. `server.py::build_server()` constructs the FastMCP server, mounting sub-servers per spec
5. Complex endpoints bypass auto-generation via `OVERRIDE_OPERATION_IDS` in `toolsets.py` and use hand-written tools in `overrides.py` (order placement) and `market_data_overrides.py` (historical bars/quotes/trades with relative time params)

**Key flow:** `cli.py` (Click entry point) → `build_server()` → FastMCP with two mounted sub-servers (Trading API + Market Data API), each backed by an `httpx.AsyncClient` with Alpaca auth headers.

**Toolset filtering:** Users set `ALPACA_TOOLSETS` env var (comma-separated) to load only specific toolsets. When empty/unset, all toolsets load.

## Spec Sync Workflow

When updating OpenAPI specs, follow the full process documented in `AGENTS.md`: sync specs → diff → classify changes (modified/new/removed endpoints) → update `toolsets.py` and `names.py` → validate with integrity tests → update README → add integration tests → commit.

## Test Layers

- **Integrity tests** (`test_integrity.py`): Validate spec ↔ toolset ↔ names consistency. Self-updating — they read source files at runtime. All 7 must pass before any commit.
- **Server construction** (`test_server_construction.py`): Verify server builds and exposes exactly 61 tools. Uses mock credentials, no network.
- **Paper integration** (`test_paper_integration.py`): Real API calls against Alpaca paper trading. Tests use `server.call_tool()` and parse results with `_to_dict`/`_parse` helpers. Auto-cleans stale orders/positions at session start.

## Key Conventions

- `postOrder` is never exposed directly — it's replaced by three override tools: `place_stock_order`, `place_crypto_order`, `place_option_order`
- Historical data endpoints (`StockBars`, `StockQuotes`, `StockTrades`, `CryptoBars`, `CryptoQuotes`, `CryptoTrades`) are in `OVERRIDE_OPERATION_IDS` and replaced by override tools with relative-time convenience parameters
- Tool names must be unique across all toolsets (enforced by integrity tests)
- Every operationId in `toolsets.py` must have a corresponding `ToolOverride` in `names.py`
- Python 3.10+, ruff line-length 100, mypy strict mode
- asyncio_mode = "auto" for pytest (no need for `@pytest.mark.asyncio`)

## CI

Two GitHub Actions jobs in `.github/workflows/ci.yml`:
- `test-core`: Runs on every PR (integrity + server construction tests)
- `test-integration`: Runs on pushes to main and same-repo PRs (requires `ALPACA_API_KEY`/`ALPACA_SECRET_KEY` secrets)
