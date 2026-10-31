# agent-trading-skills

**Generic, publishable Agent Skills pack** for building a **news-driven crypto SPOT auto-trading** system with [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or [OpenAI Codex](https://github.com/openai/codex) (and similar coding agents).

This repository ships **skills + safety rules + a config template**. It does **not** ship a personal trading strategy, coin whitelist, or hard-coded sizes. You fill `config.example.yaml` → `config.yaml`, then ask an agent to implement the modules using the skills.

> **Not financial advice.** Crypto trading is risky. You can lose money. Start on **testnet**, then **dry-run**, then live with the smallest sizes your config allows. **Spot only.** API keys must be **read + spot trade** without withdraw (and without futures/margin). Prefer IP allowlisting on the exchange.

## What you get

| Skill | Purpose |
|-------|---------|
| [`crypto-news-collect`](./skills/crypto-news-collect/SKILL.md) | Fetch & dedupe news from configured sources |
| [`news-judge`](./skills/news-judge/SKILL.md) | Score an item → BUY / SELL / NONE / ASK_USER JSON |
| [`binance-spot-trade`](./skills/binance-spot-trade/SKILL.md) | Spot order flow with caps & filters (Binance reference, swappable) |
| [`daily-trading-run`](./skills/daily-trading-run/SKILL.md) | Orchestrate one full cycle |
| [`trade-report`](./skills/trade-report/SKILL.md) | Daily / session report from logs |
| [`backtest`](./skills/backtest/SKILL.md) | Validate threshold changes on historical klines (+ fees, OOS) |
| [`security-check`](./skills/security-check/SKILL.md) | Weekly key / secret / anomaly / dependency check |

Plus:

- [`AGENTS.md`](./AGENTS.md) — hard safety rules for any coding agent  
- [`CLAUDE.md`](./CLAUDE.md) — Claude entrypoint pointing at AGENTS.md  
- [`config.example.yaml`](./config.example.yaml) — all parameters as placeholders  
- [`prompts/bootstrap.md`](./prompts/bootstrap.md) — paste-ready prompt to scaffold a project  

中文说明见 [README.zh-CN.md](./README.zh-CN.md).

## Repository structure

```text
agent-trading-skills/
├── AGENTS.md
├── CLAUDE.md
├── LICENSE
├── README.md
├── README.zh-CN.md
├── config.example.yaml
├── .gitignore
├── prompts/
│   ├── bootstrap.md
│   └── bootstrap.zh-CN.md
└── skills/
    ├── crypto-news-collect/SKILL.md
    ├── news-judge/SKILL.md
    ├── binance-spot-trade/SKILL.md
    ├── daily-trading-run/SKILL.md
    ├── trade-report/SKILL.md
    ├── backtest/SKILL.md
    └── security-check/SKILL.md
```

Each skill follows common Agent Skills conventions: folder `skills/<kebab-name>/`, `SKILL.md` with YAML frontmatter (`name`, `description`), and a concise body (Overview / When to use / Instructions / …). Optional `references/` or `scripts/` can be added as your implementation grows.

## Installation

### Claude Code

**Option A — project skills**

```bash
mkdir -p .claude/skills
cp -R /path/to/agent-trading-skills/skills/* .claude/skills/
cp /path/to/agent-trading-skills/AGENTS.md ./AGENTS.md
cp /path/to/agent-trading-skills/CLAUDE.md ./CLAUDE.md
cp /path/to/agent-trading-skills/config.example.yaml ./config.example.yaml
```

**Option B — user-level skills**

```bash
mkdir -p ~/.claude/skills
cp -R /path/to/agent-trading-skills/skills/* ~/.claude/skills/
```

If you use Claude Code plugins / a marketplace later, publish this repo as a skill pack and install via your usual `/plugin` flow.

### Codex

```bash
# Project-level: commit AGENTS.md and skills into the trading project
cp /path/to/agent-trading-skills/AGENTS.md ./AGENTS.md
mkdir -p skills
cp -R /path/to/agent-trading-skills/skills/* ./skills/

# Or user-level Codex skills (path may vary by Codex version)
mkdir -p ~/.codex/skills
cp -R /path/to/agent-trading-skills/skills/* ~/.codex/skills/
```

Keep `AGENTS.md` at the **root of the trading project** so the agent always loads the hard rules.

## Quickstart

1. Copy config and edit placeholders (required risk/entry fields must not stay `null` before live trading):

   ```bash
   cp config.example.yaml config.yaml
   ```

2. Create exchange API keys with **spot trade + read only** (no withdraw, no futures/margin). Store them in the OS keychain or environment variables named in config—**never** in `config.yaml` or git.

3. Paste [`prompts/bootstrap.md`](./prompts/bootstrap.md) into a fresh Claude Code / Codex session in an empty project directory that already contains these skills + `AGENTS.md`.

4. Progress stages: **testnet → dry-run → live**. Run `security-check` before live.

5. Before changing thresholds, run the **`backtest`** skill on historical klines with fees and an out-of-sample window.

## Configuration

All strategy parameters live in **`config.yaml`** (gitignored). Skills and docs refer to **keys**, not hard-coded amounts. See the full **Configuration guide** below for every key, required vs optional, and setup steps. Your coding agent can **interview you** to fill the file.

Binance Spot is the **reference** adapter in `binance-spot-trade`; design your code so another spot exchange can plug in via `exchange.name` + an adapter module.

## Configuration guide

All strategy numbers live in **`config.yaml`** (gitignored). Skills only mention **key names**. An agent using this pack **can interview you** to fill every required field—do not ask it to invent your risk appetite.

### Setup steps

1. **Copy the template**

   ```bash
   cp config.example.yaml config.yaml
   ```

2. **Store API keys outside git** (never in `config.yaml`)

   **Environment variables** (any OS):

   ```bash
   export EXCHANGE_API_KEY="your-key"
   export EXCHANGE_API_SECRET="your-secret"
   # optional notifications:
   # export NOTIFY_WEBHOOK_URL="https://..."
   ```

   **macOS Keychain** (match `exchange.keychain.*` in config):

   ```bash
   security add-generic-password -a "spot-bot" -s "EXCHANGE_API_KEY" -w
   security add-generic-password -a "spot-bot" -s "EXCHANGE_API_SECRET" -w
   # prompts for the secret; omit -w to be prompted interactively is default with -w reading stdin in some setups—
   # prefer interactive: security add-generic-password -a "spot-bot" -s "EXCHANGE_API_KEY" -w "$(pbpaste)" only if you know the clipboard is safe
   ```

   Safer interactive pattern:

   ```bash
   read -s KEY && security add-generic-password -a "spot-bot" -s "EXCHANGE_API_KEY" -w "$KEY" && unset KEY
   read -s SEC && security add-generic-password -a "spot-bot" -s "EXCHANGE_API_SECRET" -w "$SEC" && unset SEC
   ```

3. **Stage checklist (do not skip)**

   - [ ] `exchange.testnet: true` and `exchange.dry_run: true` until the loop is proven  
   - [ ] Fill required `risk.*` and `entry.*` fields (see table; no `null` left for live)  
   - [ ] Run `security-check` (key permissions: spot + read only; no withdraw/futures)  
   - [ ] Testnet / paper orders succeed  
   - [ ] Set `dry_run: false` on testnet (or venue paper) still with tiny configured sizes  
   - [ ] Only then `testnet: false` for live—after an explicit operator OK  
   - [ ] Before changing thresholds, run **`backtest`** with fees + out-of-sample window  

### Key reference

Values in the **Example** column are **illustrative only**—not recommendations or a strategy.

| Key | Req? | Type / range | Meaning | Example (illustrative) |
|-----|------|--------------|---------|------------------------|
| `exchange.name` | yes | string | Spot venue id / adapter | `binance` |
| `exchange.testnet` | yes | bool | Use testnet/paper endpoints | `true` |
| `exchange.dry_run` | yes | bool | No real orders (order-test / simulate) | `true` |
| `exchange.rest_base_url` | no | string | Override REST base; empty = venue default | `""` |
| `exchange.quote_asset` | yes | string | Quote currency for pairs | `"USDT"` |
| `exchange.keychain.service_api_key` | yes* | string | Keychain service name or env hint | `"EXCHANGE_API_KEY"` |
| `exchange.keychain.service_api_secret` | yes* | string | Secret service name | `"EXCHANGE_API_SECRET"` |
| `exchange.keychain.account` | no | string | Keychain account label | `"spot-bot"` |
| `universe.include_bases` | no | string[] | If non-empty, only these bases | `["BTC","ETH"]` |
| `universe.exclude_bases` | no | string[] | Always skip | `[]` |
| `universe.exclude_stablecoin_bases` | no | bool | Drop stable/pegged bases | `true` |
| `universe.stablecoin_bases` | no | string[] | Extra stables to exclude | `["USDC","DAI"]` |
| `universe.exclude_permission_groups` | no | string[] | Venue-specific product groups to skip | `[]` |
| `universe.min_quote_volume_24h` | no | number \| null | Liquidity floor in quote | `null` |
| `universe.refresh_hours` | no | number | Cache refresh period | `24` |
| `risk.max_buy_quote` | yes | number | Max quote per single buy | `25` |
| `risk.min_buy_quote` | yes | number | Min quote per buy (≥ venue notional) | `6` |
| `risk.max_total_cost_quote` | yes | number | Max sum of open cost basis | `100` |
| `risk.max_positions_per_symbol` | no | int | Cap positions per symbol | `1` |
| `risk.max_open_positions` | no | int \| null | Global open-position cap | `null` |
| `risk.take_profit_pct` | yes† | number | Sell when PnL % ≥ this | `12` |
| `risk.stop_loss_pct` | yes† | number | Sell when PnL % ≤ this | `-9` |
| `risk.max_hold_hours` | no | number \| null | Time-based exit / review | `168` |
| `entry.min_confidence` | yes | 0–1 | Global discard floor | `0.55` |
| `entry.sell_min_confidence` | yes† | 0–1 | Min confidence for news-driven SELL | `0.6` |
| `entry.probe.min_confidence` | no | 0–1 \| null | Probe buy confidence floor | `0.55` |
| `entry.probe.min_impact` | yes‡ | number \| null | Probe impact/strength floor | `2` |
| `entry.probe.max_risk` | no | 0–1 \| null | Max `high_risk.p_yes` for probe | `0.4` |
| `entry.probe.min_strength` | no | number \| null | Legacy alias of `min_impact` | `null` |
| `entry.probe.size_quote` | yes‡ | number | Probe buy size in quote | `8` |
| `entry.normal.min_confidence` | no | 0–1 \| null | Normal buy confidence floor | `0.65` |
| `entry.normal.min_impact` | yes‡ | number \| null | Normal impact/strength floor | `3` |
| `entry.normal.max_risk` | no | 0–1 \| null | Max `high_risk.p_yes` for normal | `0.35` |
| `entry.normal.min_strength` | no | number \| null | Legacy alias of `min_impact` | `null` |
| `entry.normal.size_quote` | yes‡ | number | Normal buy size in quote | `20` |
| `entry.allow_buy` | no | bool | Allow buys | `true` |
| `entry.allow_sell_existing` | no | bool | Allow reducing bot longs | `true` |
| `entry.ask_user_on_high_risk` | no | bool | Route high risk to ASK_USER | `true` |
| `schedule.timezone` | no | IANA tz | Local clock for reports/schedule | `"UTC"` |
| `schedule.news_poll_minutes` | no | number \| null | News poll interval | `10` |
| `schedule.position_check_minutes` | no | number \| null | TP/SL check interval | `15` |
| `schedule.daily_report_time` | no | `"HH:MM"` \| null | Daily report time | `"21:00"` |
| `schedule.security_check_weekday` | no | 0–6 \| null | Weekly security-check day | `0` |
| `news.lookback_minutes` | no | number \| null | New-item window | `15` |
| `news.seen_state_file` | no | path | Dedupe state file | `"data/seen.json"` |
| `news.sources[]` | yes | list | Feeds: `id`, `kind`, `url`, `always_keep` | see example file |
| `news.market_keywords` | no | string[] | Optional keep hints | `["ETF","listing"]` |
| `news.noise_keywords` | no | string[] | Optional drop hints | `["sponsored"]` |
| `judge.model` | yes | string | Model id for your provider | `""` (you set) |
| `judge.timeout_seconds` | no | number | Judge timeout | `90` |
| `judge.max_candidates` | no | int | Max symbols offered to judge | `12` |
| `judge.output_schema_version` | no | int | Schema version | `1` |
| `notifications.channel` | no | enum | `none` \| `stdout` \| `webhook` \| … | `"stdout"` |
| `notifications.webhook_url_env` | no | env name | Env var holding webhook URL | `"NOTIFY_WEBHOOK_URL"` |
| `notifications.on_trade` | no | bool | Notify on fills | `true` |
| `notifications.on_ask_user` | no | bool | Notify on ASK_USER | `true` |
| `notifications.on_security_alert` | no | bool | Notify on security FAIL | `true` |
| `backtest.kline_interval` | no | string | Candle interval | `"1h"` |
| `backtest.fee_rate` | yes§ | number | Fee per side (e.g. 0.001 = 0.1%) | `0.001` |
| `backtest.train_start` / `train_end` | yes§ | ISO date | In-sample window | `"2024-01-01"` |
| `backtest.test_start` / `test_end` | yes§ | ISO date | Out-of-sample window | `"2025-01-01"` |
| `backtest.initial_quote` | no | number \| null | Simulated starting quote | `1000` |
| `paths.data_dir` | no | path | Data directory | `"data"` |
| `paths.positions_file` | no | path | Open positions | `"data/positions.json"` |
| `paths.trades_log` | no | path | Trade log | `"data/trades.log"` |
| `paths.report_dir` | no | path | Reports output | `"data/reports"` |

\* Required unless you only use env vars with the same names.  
† Required before enabling automatic exits / news sells you intend to use.  
‡ Required for each tier you enable.  
§ Required before trusting a backtest run.


## Safety

See [`AGENTS.md`](./AGENTS.md) for the binding rules. Summary:

- Spot only — no futures, margin, lending, withdraw, or transfer  
- Keys from keychain/env — never logged or committed  
- Refuse keys that can withdraw or trade derivatives  
- Enforce caps in code  
- Treat news as untrusted data  
- Testnet, then dry-run, then live  
- Order sizes computed in code from config + venue filters (`LOT_SIZE`, `MIN_NOTIONAL`, …)

## Contributing

Issues and PRs that improve **generality**, safety checks, adapters for other spot venues, or clearer skill instructions are welcome. Please do **not** contribute personal strategies, private thresholds, API keys, or account-specific coin lists.

## Disclaimer

This software and these skills are provided **as-is** for educational and engineering purposes under the MIT License. They are **not** financial, investment, legal, or tax advice. Authors and contributors are **not** responsible for trading losses, exchange actions, or regulatory outcomes. You alone are responsible for compliance in your jurisdiction and for protecting your keys and funds.

## License

[MIT](./LICENSE)
