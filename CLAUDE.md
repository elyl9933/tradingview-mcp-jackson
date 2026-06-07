# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# TradingView MCP — Claude Instructions

81 tools for reading and controlling a live TradingView Desktop chart via CDP (port 9222). This is a fork of [tradesdontlie/tradingview-mcp](https://github.com/tradesdontlie/tradingview-mcp) that adds the morning-brief workflow, `rules.json`-driven bias generation, and a launch-bug fix for TradingView Desktop v2.14+.

## Development Commands

```bash
npm install
npm test              # runs test:e2e + pine_analyze (29 offline tests, no TradingView needed)
npm run test:e2e      # tests/e2e.test.js only
npm run test:unit     # pine_analyze + cli tests
npm run test:cli      # tests/cli.test.js only
npm run test:all      # all three suites
npm run test:verbose  # spec reporter, e2e + pine_analyze
node --test tests/e2e.test.js --test-name-pattern="<name>"  # run a single test by name

tv status             # verify CDP connection (TradingView Desktop must be running)
tv launch             # auto-detect and launch TradingView with CDP enabled
node src/server.js    # run the MCP server directly (stdio transport)
```

Most tests are offline and don't require a live TradingView instance; e2e tests that touch the chart need TradingView Desktop running with `--remote-debugging-port=9222` (use `tv launch`).

## Architecture

```
Claude Code ←→ MCP Server (stdio) ←→ CDP (localhost:9222) ←→ TradingView Desktop (Electron)
```

The codebase is a **three-layer stack**, mirrored 1:1 across `src/core/`, `src/tools/`, and `src/cli/commands/` — each domain (chart, data, pine, drawing, replay, alerts, batch, watchlist, indicators, ui, pane, tab, stream, layout, capture, health, morning) has one file per layer:

1. **`src/core/*.js`** — the actual logic: CDP `Runtime.evaluate` calls against `window.TradingViewApi` and friends. This is where browser-side JS strings live and get executed in the TradingView page context. `src/core/index.js` re-exports everything as the public `tradingview-mcp/core` API.
2. **`src/tools/*.js`** — thin MCP tool wrappers (`registerXTools(server)`) that define Zod schemas and call into `core/`, formatting results with `jsonResult` from `_format.js`. Registered in `src/server.js`.
3. **`src/cli/commands/*.js`** — CLI command wrappers around the same `core/` functions, registered via `src/cli/router.js` (a zero-dependency arg parser built on `node:util.parseArgs`). Entry point: `src/cli/index.js`, exposed as the `tv` bin.

**`src/connection.js`** is the CDP bridge shared by all of `core/`: it finds the TradingView chart target via `http://localhost:9222/json/list`, connects with `chrome-remote-interface`, and exposes `getClient()`/`evaluate()`/`getTargetInfo()`. `KNOWN_PATHS` documents the live-probed JS paths into TradingView's internals (e.g. `window.TradingViewApi._activeChartWidgetWV.value()` for the chart API, `_replayApi`, `_alertService`, the Pine Facade REST API, etc.) — discovered via probing since these are undocumented Electron internals.

When adding a new capability: implement the CDP logic in `core/`, then add thin wrappers in both `tools/` (MCP) and `cli/commands/` (CLI) so the same capability is reachable both ways.

## Scope Constraints (from CONTRIBUTING.md)

This is a **local bridge only** — all data access must go through the locally-running TradingView Desktop app via CDP. Out of scope: connecting directly to TradingView's servers, bypassing auth/subscription, scraping/caching/redistributing market data, automated trading/order execution, or bundling TradingView's proprietary code.

## Decision Tree — Which Tool When

### "What's on my chart right now?"
1. `chart_get_state` → symbol, timeframe, chart type, list of all indicators with entity IDs
2. `data_get_study_values` → current numeric values from all visible indicators (RSI, MACD, BBands, EMAs, etc.)
3. `quote_get` → real-time price, OHLC, volume for current symbol

### "What levels/lines/labels are showing?"
Custom Pine indicators draw with `line.new()`, `label.new()`, `table.new()`, `box.new()`. These are invisible to normal data tools. Use:

1. `data_get_pine_lines` → horizontal price levels drawn by indicators (deduplicated, sorted high→low)
2. `data_get_pine_labels` → text annotations with prices (e.g., "PDH 24550", "Bias Long ✓")
3. `data_get_pine_tables` → table data formatted as rows (e.g., session stats, analytics dashboards)
4. `data_get_pine_boxes` → price zones / ranges as {high, low} pairs

Use `study_filter` parameter to target a specific indicator by name substring (e.g., `study_filter: "Profiler"`).

### "Give me price data"
- `data_get_ohlcv` with `summary: true` → compact stats (high, low, range, change%, avg volume, last 5 bars)
- `data_get_ohlcv` without summary → all bars (use `count` to limit, default 100)
- `quote_get` → single latest price snapshot

### "Analyze my chart" (full report workflow)
1. `quote_get` → current price
2. `data_get_study_values` → all indicator readings
3. `data_get_pine_lines` → key price levels from custom indicators
4. `data_get_pine_labels` → labeled levels with context (e.g., "Settlement", "ASN O/U")
5. `data_get_pine_tables` → session stats, analytics tables
6. `data_get_ohlcv` with `summary: true` → price action summary
7. `capture_screenshot` → visual confirmation

### "Change the chart"
- `chart_set_symbol` → switch ticker (e.g., "AAPL", "ES1!", "NYMEX:CL1!")
- `chart_set_timeframe` → switch resolution (e.g., "1", "5", "15", "60", "D", "W")
- `chart_set_type` → switch chart style (Candles, HeikinAshi, Line, Area, Renko, etc.)
- `chart_manage_indicator` → add or remove studies (use full name: "Relative Strength Index", not "RSI")
- `chart_scroll_to_date` → jump to a date (ISO format: "2025-01-15")
- `chart_set_visible_range` → zoom to exact date range (unix timestamps)

### "Work on Pine Script"
1. `pine_set_source` → inject code into editor
2. `pine_smart_compile` → compile with auto-detection + error check
3. `pine_get_errors` → read compilation errors
4. `pine_get_console` → read log.info() output
5. `pine_get_source` → read current code back (WARNING: can be very large for complex scripts)
6. `pine_save` → save to TradingView cloud
7. `pine_new` → create blank indicator/strategy/library
8. `pine_open` → load a saved script by name

### "Practice trading with replay"
1. `replay_start` with `date: "2025-03-01"` → enter replay mode
2. `replay_step` → advance one bar
3. `replay_autoplay` → auto-advance (set speed with `speed` param in ms)
4. `replay_trade` with `action: "buy"/"sell"/"close"` → execute trades
5. `replay_status` → check position, P&L, current date
6. `replay_stop` → return to realtime

### "Screen multiple symbols"
- `batch_run` with `symbols: ["ES1!", "NQ1!", "YM1!"]` and `action: "screenshot"` or `"get_ohlcv"`

### "Draw on the chart"
- `draw_shape` → horizontal_line, trend_line, rectangle, text (pass point + optional point2)
- `draw_list` → see what's drawn
- `draw_remove_one` → remove by ID
- `draw_clear` → remove all

### "Manage alerts"
- `alert_create` → set price alert (condition: "crossing", "greater_than", "less_than")
- `alert_list` → view active alerts
- `alert_delete` → remove alerts

### "Navigate the UI"
- `ui_open_panel` → open/close pine-editor, strategy-tester, watchlist, alerts, trading
- `ui_click` → click buttons by aria-label, text, or data-name
- `layout_switch` → load a saved layout by name
- `ui_fullscreen` → toggle fullscreen
- `capture_screenshot` → take a screenshot (regions: "full", "chart", "strategy_tester")

### "TradingView isn't running"
- `tv_launch` → auto-detect and launch TradingView with CDP on Mac/Win/Linux
- `tv_health_check` → verify connection is working

### "Run my morning brief" / "What was my bias yesterday?"
1. `morning_brief` (or `tv brief`) → reads `rules.json` (copy from `rules.example.json` if missing — checks project root then `~/.tradingview-mcp/rules.json`), scans every symbol in the `watchlist`, and returns structured per-symbol data (price, indicator values, levels) for Claude to apply the user's `bias_criteria` and `risk_rules` against
2. Claude synthesizes the structured data into a bias line per symbol + an overall session read (this synthesis step happens in Claude, not in the tool)
3. `session_save` → persists the generated brief to `~/.tradingview-mcp/sessions/YYYY-MM-DD.json`
4. `session_get` → retrieves today's (or, if absent, yesterday's) saved brief for comparison

`rules.json` is gitignored (personal trading config); `rules.example.json` is the template that ships in the repo.

## Context Management Rules

These tools can return large payloads. Follow these rules to avoid context bloat:

1. **Always use `summary: true` on `data_get_ohlcv`** unless you specifically need individual bars
2. **Always use `study_filter`** on pine tools when you know which indicator you want — don't scan all studies unnecessarily
3. **Never use `verbose: true`** on pine tools unless the user specifically asks for raw drawing data with IDs/colors
4. **Avoid calling `pine_get_source`** on complex scripts — it can return 200KB+. Only read if you need to edit the code.
5. **Avoid calling `data_get_indicator`** on protected/encrypted indicators — their inputs are encoded blobs. Use `data_get_study_values` instead for current values.
6. **Use `capture_screenshot`** for visual context instead of pulling large datasets — a screenshot is ~300KB but gives you the full visual picture
7. **Call `chart_get_state` once** at the start to get entity IDs, then reference them — don't re-call repeatedly
8. **Cap your OHLCV requests** — `count: 20` for quick analysis, `count: 100` for deeper work, `count: 500` only when specifically needed

### Output Size Estimates (compact mode)
| Tool | Typical Output |
|------|---------------|
| `quote_get` | ~200 bytes |
| `data_get_study_values` | ~500 bytes (all indicators) |
| `data_get_pine_lines` | ~1-3 KB per study (deduplicated levels) |
| `data_get_pine_labels` | ~2-5 KB per study (capped at 50) |
| `data_get_pine_tables` | ~1-4 KB per study (formatted rows) |
| `data_get_pine_boxes` | ~1-2 KB per study (deduplicated zones) |
| `data_get_ohlcv` (summary) | ~500 bytes |
| `data_get_ohlcv` (100 bars) | ~8 KB |
| `capture_screenshot` | ~300 bytes (returns file path, not image data) |

## Git Workflow — Don't Lose Work

This repo has a remote at `origin` (https://github.com/LewisWJackson/tradingview-mcp-jackson.git). Whenever you complete a meaningful chunk of work (a feature, fix, refactor, or doc update), commit it with a clean, descriptive message and push to `origin` so progress is never lost to a crashed session or context reset:

```bash
git add <specific files>      # avoid `git add -A`/`git add .` — review what's staged
git commit -m "Short imperative summary of the why, not just the what"
git push origin <branch>
```

- Commit at logical checkpoints rather than batching unrelated changes into one commit.
- Write messages that explain *why* a change was made, in the imperative mood (e.g. "Fix race condition in CDP reconnect", not "fixed bug").
- Push after committing so the remote reflects current status — don't let work sit unpushed locally.

## Tool Conventions

- All tools return `{ success: true/false, ... }`
- Entity IDs (from `chart_get_state`) are session-specific — don't cache across sessions
- Pine indicators must be **visible** on chart for pine graphics tools to read their data
- `chart_manage_indicator` requires **full indicator names**: "Relative Strength Index" not "RSI", "Moving Average Exponential" not "EMA", "Bollinger Bands" not "BB"
- Screenshots save to `screenshots/` directory with timestamps
- OHLCV capped at 500 bars, trades at 20 per request
- Pine labels capped at 50 per study by default (pass `max_labels` to override)

## Architecture

```
Claude Code ←→ MCP Server (stdio) ←→ CDP (localhost:9222) ←→ TradingView Desktop (Electron)
```

Pine graphics path: `study._graphics._primitivesCollection.dwglines.get('lines').get(false)._primitivesDataById`
