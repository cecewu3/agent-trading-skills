---
name: daily-trading-run
description: >-
  Orchestrate one end-to-end news-driven spot trading cycle: collect news,
  judge items, execute allowed spot actions, run position checks, and notify.
  Use when the user says run the bot, start the daily loop, poll-and-trade,
  or wire the scheduler. Trigger phrases: "run daily trading", "start the
  cycle", "poll and trade", "run the bot once".
---

# Daily trading run

## Overview

Single orchestration skill that composes `crypto-news-collect` → `news-judge` → `binance-spot-trade` (spot adapter) → optional notify, then position `check`. All intervals and sizes come from `config.yaml` (`schedule.*`, `entry.*`, `risk.*`, `notifications.*`).

## When to use

- Implementing `run.py` / cron / agent schedule entrypoint
- Manual "run one cycle now"
- Documenting the production loop for operators

## Instructions

1. **Preflight**
   - Load `config.yaml`; abort if required `risk.*` / `entry.min_confidence` / tier `size_quote` & `min_impact` (or `min_strength`) fields needed for live are `null`
   - If live (`testnet: false` and `dry_run: false`), confirm permission check passed recently (or run it)
   - Acquire lock so two cycles do not overlap
2. **Collect** — run news collector for `news.lookback_minutes`
3. **Judge** — for each new item, run judge; collect actions
4. **Act**
   - `NONE` → log skip
   - `ASK_USER` → notify; do not trade
   - `BUY` → call trade module with `tier` (`probe`|`normal`); size from `entry.<tier>.size_quote`; respect dry-run
   - `SELL` → only if `entry.allow_sell_existing` and confidence ≥ `entry.sell_min_confidence` (when set)
5. **Manage** — run position check (TP / SL / max-hold)
6. **Report** — optional short stdout summary; full report via `trade-report` at `schedule.daily_report_time`
7. **Release lock**

### Suggested cycle script

```bash
python run.py once              # one cycle
python run.py loop              # sleep using schedule.* minutes
```

### Ordering guarantees

- Never trade before judge JSON validates
- Never size from model prose—only from config tiers + code
- News bodies never reach the shell as commands

## Failure handling

- Collector partial failure → continue with items received
- Judge failure on one item → skip that item (`ASK_USER` / log), continue
- Trade refuse → notify `on_security_alert` or trade failure channel; stop buying for this cycle if permissions failed

## Guidelines

- Default new deployments to `testnet: true` and `dry_run: true`
- Keep orchestration thin; business rules stay in trade/judge modules
- Exchange-agnostic: call the configured spot adapter, not a hardcoded venue name in the loop
