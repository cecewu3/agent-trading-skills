---
name: binance-spot-trade
description: >-
  Place and manage crypto SPOT orders with hard risk caps, exchange filters, and
  dry-run/testnet support. Reference implementation targets Binance Spot REST;
  keep adapters swappable for other spot venues. Use when buying/selling spot,
  checking balances or positions, implementing trade.py, or wiring order flow
  after a judge decision. Trigger phrases: "spot buy", "spot sell", "place
  order", "check positions", "dry-run trade", "trade.py".
---

# Spot trade (Binance reference)

## Overview

Implement a **spot-only** trading module that reads keys from keychain/env, enforces `risk.*` and `entry.*` from `config.yaml`, respects venue filters, and records positions/logs. Wording below is venue-agnostic where possible; Binance-specific endpoints and filters are called out as the reference adapter.

## When to use

- Executing a `BUY` / `SELL` from `news-judge`
- Periodic TP / SL / max-hold checks
- Implementing or auditing the order path

## Instructions

### 1. Safety gates (before any signed call)

1. Confirm `exchange.testnet` / `exchange.dry_run` per config and AGENTS.md stage rules.
2. Load API key + secret from OS keychain or env (`exchange.keychain` / documented env names). Never print them.
3. **Permission check**: query API key / account permissions. If withdraw, transfer, futures, or margin are enabled → **refuse**, notify via `notifications`, stop.
4. Allow-list HTTP methods+paths in code. Reference Binance Spot allow-list pattern:
   - `GET /api/v3/time`
   - `GET /api/v3/ticker/price`
   - `GET /api/v3/exchangeInfo`
   - `GET /api/v3/account`
   - `POST /api/v3/order` (live)
   - `POST /api/v3/order/test` (dry-run)
   - Reject everything else (no withdraw, margin, futures, transfer).

### 2. Symbol + filters

1. Resolve `SYMBOL` = base + `exchange.quote_asset` (e.g. `BTC` + `USDT`).
2. Verify symbol status is tradable for **spot** (Binance: `status=TRADING`, spot permission).
3. Read filters (Binance names; map equivalents on other venues):
   - `LOT_SIZE` / `MARKET_LOT_SIZE` — step size, min/max qty
   - `MIN_NOTIONAL` or `NOTIONAL` — min quote
   - price tick size if using limit orders
4. Round quantities **down** to step size; ensure quote notional ≥ max(config `risk.min_buy_quote`, venue min).

### 3. Risk caps (code)

Before buy:

- Quote amount ≤ `risk.max_buy_quote`
- Amount equals `entry.probe.size_quote` or `entry.normal.size_quote` for the judge `tier` (or explicit CLI amount that still respects caps)
- Judge confidence already met `entry.min_confidence` and the tier's `min_confidence`; impact met tier `min_impact`; risk ≤ tier `max_risk` (trade module may re-check)
- Open bot cost basis + new buy ≤ `risk.max_total_cost_quote`
- Positions on symbol < `risk.max_positions_per_symbol`
- Symbol in universe (include/exclude/stables rules)

Before sell:

- Only sell quantity tracked (or available) for that spot position; no naked short
- News-driven sells: confidence ≥ `entry.sell_min_confidence` when that key is set; `entry.allow_sell_existing` must be true
- TP/SL/max-hold exits use `risk.*` and do not require `sell_min_confidence`

### 4. Order flow

```text
judge/CLI intent
  → validate config + permissions + universe
  → compute quote/qty in code
  → apply filters (LOT_SIZE, MIN_NOTIONAL)
  → acquire process lock
  → dry_run ? order-test / simulate : create market (or configured) order
  → update positions file + append trades log
  → release lock → JSON result on stdout
  → optional notification (no secrets)
```

### 5. Exit checks

On a schedule (`schedule.position_check_minutes`) or CLI `check`:

- If PnL % ≥ `risk.take_profit_pct` → sell (per rules)
- If PnL % ≤ `risk.stop_loss_pct` → sell
- If held ≥ `risk.max_hold_hours` → sell or alert (per your explicit policy in config comments)

### Suggested CLI

```bash
python trade.py balance
python trade.py price SYMBOL
python trade.py buy SYMBOL --tier probe|normal [--dry-run]
python trade.py sell SYMBOL [all|QTY] [--dry-run]
python trade.py positions
python trade.py check [--dry-run]
```

### Project paths

| Path | Role |
|------|------|
| `config.yaml` | exchange, risk, entry |
| `trade.py` | orders + checks |
| `data/positions.json` | open bot positions |
| `data/trades.log` | append-only log |

## Input / output schema

**Buy input:** symbol, tier or quote amount, dry-run flag.  
**Output:** JSON with `ok`, `symbol`, `side`, `qty`, `quote`, `order_id` (or `dry_run`), `ts`.  
**Refuse:** JSON with `ok: false`, `error`, non-zero exit.

## Failure handling

- Insufficient balance / filter reject → no retry loop that escalates size
- Lock busy → exit cleanly
- Network error mid-order → query open orders / fills before retry; use client order id for idempotency

## Guidelines

- Spot only; refuse futures/margin wording in CLI
- Prefer marketable spot orders for news-driven flow unless config says otherwise
- Other venues: keep the same caps and lock/log pattern behind an adapter interface
