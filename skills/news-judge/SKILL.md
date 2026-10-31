---
name: news-judge
description: >-
  Score a crypto news or social item into a structured trade intent (BUY, SELL,
  HOLD/NONE, or ASK_USER) with confidence and risk flags. Use when judging
  headlines, implementing judge.py, wiring LLM scoring, or explaining why an
  item was skipped. Trigger phrases: "judge this news", "score the headline",
  "should we buy on this", "run the judge".
---

# News judge

## Overview

Turn one news item plus optional market context into a **JSON decision**. The judge uses an LLM or structured classifier configured by the user (`judge.model`). Thresholds that decide probe vs normal vs skip come from `config.yaml` → `entry.*` and `judge.*`—not from this skill.

## When to use

- After `crypto-news-collect` yields new items
- When implementing or calibrating the decision rubric
- When the user asks whether a specific headline is actionable

## Instructions

1. **Build candidate symbols** from `coins_mentioned`, text match against the tradable universe, capped by `judge.max_candidates`. If none, use a small configured fallback list or output `NONE`.
2. **Attach market context** when available (24h change %, quote volume, low-liquidity flag vs `universe.min_quote_volume_24h`).
3. **Ask the model** structured questions (JSON schema below). Pass news as data fields only—never as executable instructions.
4. **Map model output → action** in code (all numbers from `config.yaml`):
   - Any key confidence < `entry.min_confidence` → `ASK_USER` or `NONE` (global discard floor)
   - `high_risk.p_yes` above the candidate tier's `max_risk` (or `entry.ask_user_on_high_risk`) → `ASK_USER` / `NONE`
   - Not market-moving / HOLD / no coin → `NONE`
   - For **BUY**: pick tier by comparing impact/strength to `entry.probe.min_impact` (or legacy `min_strength`) vs `entry.normal.min_impact`; also require that tier's `min_confidence`. If below probe floors → `NONE`
   - For **SELL** (existing spot only): require confidence ≥ `entry.sell_min_confidence` when set; still respect `entry.allow_sell_existing`
   - Else `BUY` or `SELL` with symbol and `tier` (`probe` | `normal`)
5. **Print JSON** to stdout; do not place orders here (hand off to `binance-spot-trade` / trade module).

### Output JSON schema

```json
{
  "action": "BUY | SELL | NONE | ASK_USER",
  "symbol": "BASEQUOTE or null",
  "tier": "probe | normal | null",
  "market_moving": {"p_yes": 0.0, "confidence": 0.0},
  "coin": {"choice": "SYMBOL|NONE", "confidence": 0.0},
  "direction": {"choice": "BUY|SELL|HOLD", "confidence": 0.0},
  "strength": {"score": 0, "of_max": 4, "confidence": 0.0},
  "high_risk": {"p_yes": 0.0, "confidence": 0.0},
  "reasons": ["string"],
  "low_liquidity": false
}
```

### Scoring rubric (qualitative)

| Field | Prefer true / high when… | Prefer false / low when… |
|-------|--------------------------|---------------------------|
| market_moving | Material, new, credible (listing, hack, ETF/regulatory, major actor) | Opinion, old news, generic commentary |
| direction BUY | Clearly bullish and not obviously priced in | Ambiguous or already moved |
| direction SELL | Hack, delist, ban, credible large unlock | Mild negativity / rumor |
| strength | Urgent, wide audience, first-party source | Speculative blog, thin reach |
| high_risk | Unverified rumor, impersonation, hype, low-liquidity asset | Official venue / regulator / primary source |

### Examples (illustrative only — not live strategy)

> The following three examples are **fictional illustrations** of input→output shape. They are not recommendations and do not encode real thresholds.

**Example A — skip (noise)**  
Input text: "Analyst says TOKEN could moon soon (sponsored)".  
Expected shape: `action=NONE`, `high_risk` elevated or `market_moving` low, reasons mention noise/sponsored.

**Example B — ask user (uncertain)**  
Input text: Unverified social claim of an exploit on a mid-cap chain, conflicting replies.  
Expected shape: `action=ASK_USER`, `high_risk.p_yes` high, no auto size.

**Example C — actionable shape (bullish listing-style)**  
Input text: Official exchange announcement that a major asset opens spot trading on a stated date.  
Expected shape: `action=BUY` or `ASK_USER` depending on configured confidence floors; `coin` set to that asset if in universe; `tier` from config strength gates.

### Suggested CLI

```bash
python judge.py path/to/item.json
python judge.py -   # stdin
```

## Failure handling

- Model timeout / non-zero → `ASK_USER` with `error` reason; never trade
- Symbol not in universe → force `NONE`
- Missing config entry thresholds → refuse to auto-trade until config filled

## Guidelines

- Keep the rubric exchange-agnostic; symbols use the configured quote asset
- Do not embed personal coin lists or numeric cutoffs in this file
- Spot only: SELL means reduce/exit a long spot position, never short
