# agent-trading-skills

面向 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) / [Codex](https://github.com/openai/codex) 等编程代理的**通用、可公开发布**的 Agent Skills 包，用于搭建**新闻驱动的加密货币现货（SPOT）自动交易**系统。

本仓库提供 **技能说明 + 安全规则 + 配置模板**，**不包含**个人交易策略、币种白名单或写死的下单金额。请将 `config.example.yaml` 复制为 `config.yaml` 并自行填写，再让代理按技能实现代码模块。

> **不构成投资建议。** 交易有风险，可能亏损。务必先走 **测试网 → dry-run 模拟 → 小额实盘**。仅限**现货**。API Key 只需**现货交易 + 只读**，**关闭提现**，并关闭合约/杠杆权限；建议在交易所配置 **IP 白名单**。

## 技能一览

| 技能 | 作用 |
|------|------|
| [`crypto-news-collect`](./skills/crypto-news-collect/SKILL.md) | 按配置采集并去重新闻 |
| [`news-judge`](./skills/news-judge/SKILL.md) | 将新闻判分为 BUY / SELL / NONE / ASK_USER |
| [`binance-spot-trade`](./skills/binance-spot-trade/SKILL.md) | 现货下单与风控（以 Binance 为参考实现，可替换交易所） |
| [`daily-trading-run`](./skills/daily-trading-run/SKILL.md) | 编排单次完整交易循环 |
| [`trade-report`](./skills/trade-report/SKILL.md) | 根据日志生成日报/会话报告 |
| [`backtest`](./skills/backtest/SKILL.md) | 改门槛前用历史 K 线（含手续费、样本外）验证 |
| [`security-check`](./skills/security-check/SKILL.md) | 每周密钥权限、密钥泄露、异常订单、依赖自查 |

另含 [`AGENTS.md`](./AGENTS.md) 硬性安全规则、[`config.example.yaml`](./config.example.yaml) 占位配置，以及 [`prompts/bootstrap.zh-CN.md`](./prompts/bootstrap.zh-CN.md) 一键脚手架提示词。

英文版：[README.md](./README.md)。

## 安装

### Claude Code

```bash
mkdir -p .claude/skills
cp -R /path/to/agent-trading-skills/skills/* .claude/skills/
cp /path/to/agent-trading-skills/AGENTS.md ./AGENTS.md
cp /path/to/agent-trading-skills/CLAUDE.md ./CLAUDE.md
cp /path/to/agent-trading-skills/config.example.yaml ./config.example.yaml
```

或复制到 `~/.claude/skills/` 作为用户级技能。

### Codex

```bash
cp /path/to/agent-trading-skills/AGENTS.md ./AGENTS.md
mkdir -p skills
cp -R /path/to/agent-trading-skills/skills/* ./skills/
# 或 ~/.codex/skills/
```

请将 `AGENTS.md` 放在**交易项目根目录**，确保代理始终加载安全规则。

## 快速开始

1. `cp config.example.yaml config.yaml`，填好风控与入场等必填项（不要留 `null` 就实盘）。
2. 创建**仅现货、无提现**的 API Key，放入系统钥匙串或环境变量，**不要**写入配置文件或 Git。
3. 在已放入本技能包的空项目中，粘贴 [`prompts/bootstrap.zh-CN.md`](./prompts/bootstrap.zh-CN.md)。
4. 阶段：`testnet` → `dry_run` → live；实盘前运行 `security-check`。
5. 修改止盈止损/门槛前，先跑 **`backtest`**。

## 配置原则

一切数值参数只写在 **`config.yaml`**。技能文档只引用**配置键名**，不写死金额、百分比、币单或具体交易所账号策略。`binance-spot-trade` 以 Binance Spot 为参考，可通过 `exchange.name` + 适配层扩展其他现货所。代理可以**逐项询问你**来填写配置。

## 配置指南

所有策略数字只写在 **`config.yaml`**（已加入 `.gitignore`）。技能文档只引用**配置键名**。使用本包的代理可以**逐项询问你**来填写必填项——不要让它替你编造风险偏好。

### 配置步骤

1. **复制模板**

   ```bash
   cp config.example.yaml config.yaml
   ```

2. **密钥放在 Git 之外**（不要写进 `config.yaml`）

   **环境变量**（通用）：

   ```bash
   export EXCHANGE_API_KEY="your-key"
   export EXCHANGE_API_SECRET="your-secret"
   # export NOTIFY_WEBHOOK_URL="https://..."
   ```

   **macOS 钥匙串**（与 `exchange.keychain.*` 对应）：

   ```bash
   read -s KEY && security add-generic-password -a "spot-bot" -s "EXCHANGE_API_KEY" -w "$KEY" && unset KEY
   read -s SEC && security add-generic-password -a "spot-bot" -s "EXCHANGE_API_SECRET" -w "$SEC" && unset SEC
   ```

3. **上线阶段清单（不要跳步）**

   - [ ] 保持 `exchange.testnet: true` 与 `exchange.dry_run: true` 直到流程跑通
   - [ ] 填完必填的 `risk.*` / `entry.*`（见表；实盘前不要留 `null`）
   - [ ] 运行 `security-check`（密钥仅现货+读取；无提现/合约）
   - [ ] 测试网/模拟下单成功
   - [ ] 在测试网将 `dry_run: false`（仍用配置里的小额）
   - [ ] 经你明确同意后再设 `testnet: false` 实盘
   - [ ] 改门槛前先跑带手续费与样本外窗口的 **`backtest`**

### 配置项一览

**示例**列仅为**示意**，不是推荐策略或投资建议。

| 键 | 必填? | 类型/范围 | 含义 | 示例（仅示意） |
|----|------|-----------|------|----------------|
| `exchange.name` | 是 | string | 现货所/适配器 | `binance` |
| `exchange.testnet` | 是 | bool | 测试网/纸交易 | `true` |
| `exchange.dry_run` | 是 | bool | 不下真实单 | `true` |
| `exchange.rest_base_url` | 否 | string | REST 覆盖；空=默认 | `""` |
| `exchange.quote_asset` | 是 | string | 计价资产 | `"USDT"` |
| `exchange.keychain.service_api_key` | 是* | string | 钥匙串/环境变量名 | `"EXCHANGE_API_KEY"` |
| `exchange.keychain.service_api_secret` | 是* | string | 密钥名 | `"EXCHANGE_API_SECRET"` |
| `exchange.keychain.account` | 否 | string | 钥匙串 account | `"spot-bot"` |
| `universe.include_bases` | 否 | string[] | 非空则仅这些 base | `["BTC","ETH"]` |
| `universe.exclude_bases` | 否 | string[] | 永远排除 | `[]` |
| `universe.exclude_stablecoin_bases` | 否 | bool | 排除稳定币 base | `true` |
| `universe.stablecoin_bases` | 否 | string[] | 额外稳定币列表 | `["USDC","DAI"]` |
| `universe.exclude_permission_groups` | 否 | string[] | 排除的权限组 | `[]` |
| `universe.min_quote_volume_24h` | 否 | number 或 null | 流动性下限 | `null` |
| `universe.refresh_hours` | 否 | number | 宇宙缓存刷新小时 | `24` |
| `risk.max_buy_quote` | 是 | number | 单笔买入报价上限 | `25` |
| `risk.min_buy_quote` | 是 | number | 单笔下限（≥交易所最小名义） | `6` |
| `risk.max_total_cost_quote` | 是 | number | 持仓成本合计上限 | `100` |
| `risk.max_positions_per_symbol` | 否 | int | 每币持仓数上限 | `1` |
| `risk.max_open_positions` | 否 | int 或 null | 全局持仓数上限 | `null` |
| `risk.take_profit_pct` | 是† | number | 止盈（PnL%≥） | `12` |
| `risk.stop_loss_pct` | 是† | number | 止损（PnL%≤） | `-9` |
| `risk.max_hold_hours` | 否 | number 或 null | 最长持有小时 | `168` |
| `entry.min_confidence` | 是 | 0–1 | 全局置信度丢弃线 | `0.55` |
| `entry.sell_min_confidence` | 是† | 0–1 | 新闻驱动卖出最低置信度 | `0.6` |
| `entry.probe.min_confidence` | 否 | 0–1 或 null | 试探档买入置信度 | `0.55` |
| `entry.probe.min_impact` | 是‡ | number 或 null | 试探档影响力下限 | `2` |
| `entry.probe.max_risk` | 否 | 0–1 或 null | 试探档最大 high_risk | `0.4` |
| `entry.probe.min_strength` | 否 | number 或 null | `min_impact` 旧名 | `null` |
| `entry.probe.size_quote` | 是‡ | number | 试探档买入金额 | `8` |
| `entry.normal.min_confidence` | 否 | 0–1 或 null | 常规档买入置信度 | `0.65` |
| `entry.normal.min_impact` | 是‡ | number 或 null | 常规档影响力下限 | `3` |
| `entry.normal.max_risk` | 否 | 0–1 或 null | 常规档最大 high_risk | `0.35` |
| `entry.normal.min_strength` | 否 | number 或 null | `min_impact` 旧名 | `null` |
| `entry.normal.size_quote` | 是‡ | number | 常规档买入金额 | `20` |
| `entry.allow_buy` | 否 | bool | 允许买入 | `true` |
| `entry.allow_sell_existing` | 否 | bool | 允许减持机器人多单 | `true` |
| `entry.ask_user_on_high_risk` | 否 | bool | 高风险转人工 | `true` |
| `schedule.timezone` | 否 | IANA | 报表/调度时区 | `"UTC"` |
| `schedule.news_poll_minutes` | 否 | number 或 null | 新闻轮询间隔 | `10` |
| `schedule.position_check_minutes` | 否 | number 或 null | 持仓检查间隔 | `15` |
| `schedule.daily_report_time` | 否 | HH:MM 或 null | 日报时间 | `"21:00"` |
| `schedule.security_check_weekday` | 否 | 0–6 或 null | 安全自查星期 | `0` |
| `news.lookback_minutes` | 否 | number 或 null | 新消息窗口 | `15` |
| `news.seen_state_file` | 否 | path | 去重状态文件 | `"data/seen.json"` |
| `news.sources[]` | 是 | list | 源列表 | 见示例文件 |
| `news.market_keywords` | 否 | string[] | 可选保留关键词 | `["ETF"]` |
| `news.noise_keywords` | 否 | string[] | 可选噪声词 | `["sponsored"]` |
| `judge.model` | 是 | string | 模型 ID | `""`（自填） |
| `judge.timeout_seconds` | 否 | number | 超时秒数 | `90` |
| `judge.max_candidates` | 否 | int | 候选币上限 | `12` |
| `judge.output_schema_version` | 否 | int | schema 版本 | `1` |
| `notifications.channel` | 否 | enum | 通知渠道 | `"stdout"` |
| `notifications.webhook_url_env` | 否 | env 名 | Webhook 环境变量名 | `"NOTIFY_WEBHOOK_URL"` |
| `notifications.on_trade` | 否 | bool | 成交通知 | `true` |
| `notifications.on_ask_user` | 否 | bool | 待确认通知 | `true` |
| `notifications.on_security_alert` | 否 | bool | 安全告警 | `true` |
| `backtest.kline_interval` | 否 | string | K 线周期 | `"1h"` |
| `backtest.fee_rate` | 是§ | number | 单边手续费 | `0.001` |
| `backtest.train_start` / `train_end` | 是§ | ISO 日期 | 样本内窗口 | `"2024-01-01"` |
| `backtest.test_start` / `test_end` | 是§ | ISO 日期 | 样本外窗口 | `"2025-01-01"` |
| `backtest.initial_quote` | 否 | number 或 null | 回测初始资金 | `1000` |
| `paths.*` | 否 | path | 数据/日志/报告路径 | 见示例文件 |

\* 若改用同名环境变量则可不再依赖钥匙串。  
† 在启用对应自动卖出/出场前必填。  
‡ 对你启用的每一档必填。  
§ 认真做回测前必填。


## 安全摘要

详见 [`AGENTS.md`](./AGENTS.md)：仅现货；密钥不落盘不打印；拒绝带提现/合约权限的 Key；限额代码强制执行；新闻为不可信数据；金额由代码按配置与交易所过滤器计算。

## 免责声明

本项目按 MIT 许可证「按现状」提供，仅供工程与学习用途，**不是**财务或投资建议。作者与贡献者不对交易亏损或合规后果负责。

## License

[MIT](./LICENSE)
