---
name: backtest
description: >-
  Validate proposed entry/exit threshold or size changes against historical
  klines with fees and an out-of-sample window before going live. Use when the
  user wants to change take-profit, stop-loss, strength floors, or position
  sizes, or asks to backtest a rule. Trigger phrases: "backtest", "validate
  thresholds", "simulate on klines", "out-of-sample test", "before changing
  TP/SL".
---

# Backtest

## Overview

Before changing `entry.*` (including per-tier `min_confidence` / `min_impact` / `max_risk` and `sell_min_confidence`) or `risk.*` thresholds, simulate the rule set on historical candles. Include fees (`backtest.fee_rate`), split **train** vs **out-of-sample test** windows, and report metrics. This skill does not promise profitability—only disciplined validation.

## When to use

- User proposes new TP / SL / max-hold / strength / size values
- Calibrating probe vs normal tiers
- Regression check after judge rubric changes (replay saved news+labels if available)

## Instructions

1. **Freeze candidate config** — copy proposed values into a temporary config or CLI overrides; do not write live `config.yaml` until results reviewed.
2. **Download klines** for symbols under test (interval `backtest.kline_interval`) from the venue's public market data API (Binance reference: `/api/v3/klines` or data mirror).
3. **Simulate**
   - Apply entry rules when using a historical signal file, or a simplified proxy if only OHLC (document the proxy clearly)
   - Honor `entry.min_confidence`, per-tier `min_confidence` / `min_impact` / `max_risk`, and `entry.sell_min_confidence` when signals include those fields
   - Apply exits: `risk.take_profit_pct`, `risk.stop_loss_pct`, `risk.max_hold_hours`
   - Deduct fees each side using `backtest.fee_rate`
   - Enforce caps: `risk.max_buy_quote`, `risk.max_total_cost_quote`, and tier `size_quote` values
4. **Split**
   - Train: `backtest.train_start` → `train_end` (tuning only)
   - Test: `backtest.test_start` → `test_end` (**report this as primary**)
5. **Report** (JSON + short markdown): trade count, win rate, profit factor, max drawdown, fee drag, exposure time, vs buy-and-hold baseline on same window. Flag if test window is too short or sample size tiny.
6. **Gate** — do not recommend promoting thresholds when out-of-sample is weak or statistically empty; ask user to decide.

### Suggested CLI

```bash
python backtest.py --config config.candidate.yaml
python backtest.py --override take_profit_pct=X stop_loss_pct=Y
```

### Project paths

| Path | Role |
|------|------|
| `config.yaml` / candidate file | thresholds under test |
| `backtest.py` | simulator |
| `data/reports/backtest-*.md` | artifacts |

## Failure handling

- Missing fee_rate → refuse to run (silent zero-fee backtests are misleading)
- Incomplete klines → shrink window or abort with reason
- No trades triggered → report "no signals"; do not invent fills

## Guidelines

- Never hardcode personal threshold numbers in this skill; always read config
- Prefer out-of-sample honesty over flattering train metrics
- Spot-only simulation; no leverage
