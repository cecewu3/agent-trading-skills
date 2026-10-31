---
name: security-check
description: >-
  Run a weekly (or on-demand) security self-check: API key permissions, secret
  leakage in logs/repo, anomalous orders, and dependency advisories. Use when
  the user asks for a security audit of the trading bot, weekly hygiene, or
  before enabling live trading. Trigger phrases: "security check", "audit
  keys", "scan for secrets", "weekly safety review", "anomalous orders".
---

# Security check

## Overview

Operator hygiene checklist implemented as a script and/or agent walkthrough. Schedule hint: `schedule.security_check_weekday`. Alerts use `notifications.on_security_alert`.

## When to use

- Weekly cadence
- Before flipping `dry_run: false` or `testnet: false`
- After any credential rotation or dependency upgrade

## Instructions

Run each section; record PASS / FAIL / WARN in a report under `data/reports/`.

### 1. Key permissions

- Load key the same way `trade.py` does (keychain/env)—do not print it
- Query venue API key permissions / account flags
- **FAIL** if withdraw, transfer, futures, margin, or universal transfer is allowed
- **PASS** only for spot trade + read as needed

### 2. Secret scan (logs + repo)

- Scan `data/`, `*.log`, and tracked source for patterns: API key shapes, `secret`, PEM blocks, webhook URLs with tokens
- Ensure `config.yaml`, `.env`, and state files are gitignored
- `git grep` / `rg` for accidental credential strings; **FAIL** on hits
- Confirm skills and docs contain no live secrets

### 3. Anomalous orders

- Read `data/trades.log` / positions for the lookback window
- Flag: orders larger than `risk.max_buy_quote`, symbols outside universe, trades when `dry_run` should have been on, duplicate client-order bursts, sells exceeding position
- Compare to exchange trade history if a signed read is available (still spot-only endpoints)

### 4. Dependency check

- If Python: `pip audit` or `uv pip audit` when available; else note manual review
- Pin or lockfile present? WARN if not
- No post-install scripts from untrusted paths

### 5. Config / stage sanity

- Live mode requires explicit `testnet: false` and `dry_run: false`
- Required risk fields must not be `null`
- Notification secrets only via env keys named in config

### Suggested CLI

```bash
python security_check.py
python security_check.py --json
```

## Failure handling

- Any FAIL on permissions or secret leak → notify immediately; refuse further live trades until fixed
- Audit tool missing → WARN with install hint; do not skip permission check

## Guidelines

- Read-only by default; never "fix" by widening key permissions
- Keep reports free of raw secrets (redact mid-string)
