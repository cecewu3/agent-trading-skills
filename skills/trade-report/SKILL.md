---
name: trade-report
description: >-
  Generate a human-readable session or daily report from positions and trade
  logs (fills, PnL, open risk, skips). Use when the user asks for a daily
  summary, PnL report, what happened today, or implementing report.py.
  Trigger phrases: "trading report", "daily PnL", "summarize trades",
  "what did the bot do".
---

# Trade report

## Overview

Read bot state (`paths.positions_file`, `paths.trades_log`) and produce a markdown or JSON report for a date (default: today in `schedule.timezone`). No API keys required for log-only reports; optional live marks via public ticker.

## When to use

- End of day / session
- After a manual cycle
- Auditing behavior before going live

## Instructions

1. Parse `data/trades.log` (or configured path) for the target date.
2. Load open positions; mark with last price if available.
3. Summarize:
   - Buys / sells count and quote volume
   - Realized PnL (fees if logged)
   - Unrealized PnL for open positions
   - Skips / `ASK_USER` counts if logged
   - Cap utilization vs `risk.max_total_cost_quote` (show config key, not a hardcoded number)
4. Write under `paths.report_dir` and print a short summary.
5. Never include API keys, secrets, or full account balances beyond bot-tracked positions unless the user asks.

### Suggested CLI

```bash
python report.py                  # today in schedule.timezone
python report.py YYYY-MM-DD
python report.py --json
```

## Failure handling

- Missing log file → empty report with warning
- Corrupt line → skip line, count errors

## Guidelines

- Reports are for the operator; keep tone factual
- Do not give investment advice in the report footer beyond restating that past cycle results are not predictive
