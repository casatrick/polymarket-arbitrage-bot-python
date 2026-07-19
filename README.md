# Polymarket Arbitrage Bot: 5 Trading Strategies Explained

https://github.com/user-attachments/assets/55df2f87-2aaa-4778-b726-05d759995e84

**A complete breakdown of the five arbitrage strategies used by this Polymarket arbitrage bot** - covering intra-market arbitrage, combinatorial arbitrage, cross-platform arbitrage, endgame arbitrage, and momentum/mean-reversion trading across 5-minute, 15-minute, and 1-hour timeframes.

## now working on **market making bot**

---

## What Is a Polymarket Arbitrage Bot?

A **Polymarket arbitrage bot** is an automated trading system that scans Polymarket's prediction markets for pricing inefficiencies and executes trades to capture the resulting spread. This bot runs **five distinct arbitrage and trading strategies** in parallel, each targeting a different type of market inefficiency, then ranks and executes the highest-quality signals automatically.

This document covers how each strategy works, how signals are scored, and how to configure the bot's parameters.

**For strategy consulting or purchase inquiries, contact me at Telegram:** [@casatrick](https://telegram.me/casatrick)

---

## Table of Contents

- [Strategy 1: Intra-Market Arbitrage](#strategy-1-intra-market-arbitrage)
- [Strategy 2: Combinatorial Arbitrage](#strategy-2-combinatorial-arbitrage)
- [Strategy 3: Cross-Platform Arbitrage](#strategy-3-cross-platform-arbitrage)
- [Strategy 4: Endgame Arbitrage](#strategy-4-endgame-arbitrage)
- [Strategy 5: Momentum / Mean-Reversion](#strategy-5-momentum--mean-reversion)
- [How Signal Ranking Works](#how-signal-ranking-works)
- [Risk Management](#risk-management)
- [How the Strategies Are Combined](#how-strategies-are-combined)
- [Configuration Reference](#configuration)
- [FAQ: Polymarket Arbitrage Bot](#faq-polymarket-arbitrage-bot)

---

## Strategy 1: Intra-Market Arbitrage

**How Polymarket YES/NO arbitrage works:** in binary prediction markets, YES and NO token prices should always sum to $1.00. When combined YES + NO pricing falls below $1.00 after fees, buying both outcomes locks in a guaranteed, risk-free profit - the core mechanic behind most Polymarket arbitrage bot strategies.

### Execution

1. Buy YES and NO tokens separately through the CLOB API
2. Merge tokens into complete sets
3. Redeem complete sets immediately for $1.00 per share
4. Profit is realized immediately - no waiting for market resolution

### Configuration

| Parameter | Description | Default |
|---|---|---|
| `min_spread_pct` | Minimum profit % required | 1.5% |
| `max_position_usd` | Maximum position size | $500 |
| `fee_pct` | Polymarket fee percentage | 2.0% |

---

## Strategy 2: Combinatorial Arbitrage

For multi-outcome Polymarket markets (e.g., price-range or bracket markets), all outcome prices should sum to $1.00. When the total falls below $1.00, buying every outcome guarantees a profit - a scalable extension of classic Polymarket arbitrage bot logic to markets with 3+ outcomes.

### Execution

1. Buy all outcome tokens separately through the CLOB API
2. Merge tokens into complete sets
3. Redeem complete sets immediately for $1.00 per share
4. Profit is realized immediately - no waiting for resolution

### Configuration

| Parameter | Description | Default |
|---|---|---|
| `min_deviation_pct` | Minimum deviation from $1.00 | 2.0% |
| `max_position_usd` | Maximum position size | $300 |
| `min_outcomes` | Minimum number of outcomes required | 3 |

---

## Strategy 3: Cross-Platform Arbitrage

This strategy compares Polymarket prediction prices against real-time spot prices from exchanges like Binance and CoinGecko. When a significant discrepancy appears, the bot trades on the assumption that Polymarket pricing will converge toward fair value - a directional, convergence-based approach rather than risk-free arbitrage.

### Fair Probability Calculation

Fair probability is estimated using:
- Distance of spot price from the market's strike price
- Recent price momentum
- Time remaining until market expiry

### Execution

Buy the underpriced outcome (YES or NO) and hold for market correction.

### Configuration

| Parameter | Description | Default |
|---|---|---|
| `min_price_diff_pct` | Minimum price difference to trigger | 3.0% |
| `max_position_usd` | Maximum position size | $1000 |
| `stale_threshold_sec` | Maximum age of price data | 30s |

---

## Strategy 4: Endgame Arbitrage

When a market approaches resolution and one outcome shows very high probability (>93%), buying that outcome captures a small but near-certain return. This is one of the more capital-efficient strategies in a Polymarket arbitrage bot, since time-to-resolution is short and predictable.

### Execution

Buy the high-probability outcome and hold to resolution.

### Configuration

| Parameter | Description | Default |
|---|---|---|
| `min_probability` | Minimum outcome probability | 0.93 |
| `max_time_to_resolution_hrs` | Maximum hours until resolution | 48 |
| `min_annualized_return_pct` | Minimum annualized return | 100% |
| `max_position_usd` | Maximum position size | $2000 |

---

## Strategy 5: Momentum / Mean-Reversion

This strategy tracks Polymarket YES/NO prices as a time series and applies standard technical indicators: Z-score, RSI, rate of change, and VWAP divergence - the same tools used in crypto and equities momentum trading, adapted for prediction markets.

### Entry Conditions

**Mean Reversion Buy (Oversold):**
- Z-score below -threshold
- RSI < 35
- Rate of change > -0.5%
- Action: buy YES (expect reversion upward)

**Mean Reversion Sell (Overbought):**
- Z-score above +threshold
- RSI > 65
- Rate of change < 0.5%
- Action: buy NO (expect YES price to revert down)

### Timeframe Parameters

| Timeframe | Lookback | Entry Z-score | Take Profit | Stop Loss |
|---|---|---|---|---|
| 5-minute (scalping) | 12 candles (1 hr) | ±2.0σ | 1.5% | 1.0% |
| 15-minute (swing) | 16 candles (4 hrs) | ±1.8σ | 3.0% | 2.0% |
| 1-hour (position) | 24 candles (1 day) | ±1.5σ | 5.0% | 3.0% |

### Configuration

Each timeframe has its own parameter set in `MomentumConfig`:
- `tf_5m_lookback`, `tf_5m_entry_zscore`, `tf_5m_take_profit_pct`, `tf_5m_stop_loss_pct`
- `tf_15m_*` (same pattern)
- `tf_1h_*` (same pattern)

---

## How Signal Ranking Works

The `StrategyAggregator` scores every signal (0.0-1.0) using a weighted composite:

```
score = (profit_score        × 0.30) +
        (confidence_score    × 0.25) +
        (strategy_priority   × 0.20) +
        (urgency_score       × 0.15) +
        (risk_reward_score   × 0.10)
```

| Component | Description |
|---|---|
| `profit_score` | Expected profit % ÷ 10 (capped at 1.0) |
| `confidence_score` | Signal confidence (0.0-1.0) |
| `strategy_priority` | Strategy-type weight (see below) |
| `urgency_score` | HIGH = 1.0, MEDIUM = 0.67, LOW = 0.33 |
| `risk_reward_score` | Risk/reward ratio × 5 (capped at 1.0) |

### Strategy Priority Weights

| Rank | Strategy | Priority | Reliability |
|---|---|---|---|
| 1 | Intra-market | 1.00 | Risk-free arbitrage |
| 2 | Combinatorial | 0.95 | Risk-free arbitrage |
| 3 | Endgame | 0.90 | High probability |
| 4 | Cross-platform | 0.80 | Directional, requires convergence |
| 5 | Momentum / mean-reversion | 0.70 | Technical, less certain |

---

## Risk Management

The bot's risk layer includes:

- Maximum position size per trade
- Maximum portfolio exposure
- Daily loss limits
- Stop-loss and take-profit levels
- Consecutive-loss protection

See `risk_manager.py` for implementation details.

---

## How Strategies Are Combined

The bot runs every applicable strategy on every scanned market simultaneously, then merges and ranks the resulting signals.

### Scanning Process

1. **Market discovery** - the bot discovers all crypto prediction markets for BTC, ETH, XRP, and SOL.
2. **Parallel scanning** - for each market, applicable strategies run concurrently:
   - Intra-market (binary markets)
   - Combinatorial (3+ outcome markets)
   - Cross-platform (where a strike price can be extracted)
   - Endgame (all markets)
   - Momentum/mean-reversion (all binary markets, per timeframe)
3. **Signal collection** - all signals from all strategies are pooled into a single list.
4. **Filtering** - low-quality signals are discarded:
   - Minimum confidence: 0.35 (35%)
   - Minimum profit: 0.5%
5. **Ranking** - remaining signals are ranked by composite score.
6. **Execution** - the top-ranked signals execute (max 3 per scan cycle).

### Example: Multiple Signals on One Market

**Market:** "Will BTC be above $100k by Friday?"

| Signal | Detail | Score |
|---|---|---|
| Endgame | YES at $0.96, resolves in 2 hours | 0.91 |
| Intra-market | YES=$0.45, NO=$0.50 → combined $0.95 | 0.85 |
| Cross-platform | Spot implies 70% probability vs. $0.45 YES | 0.72 |
| Momentum (5m) | Oversold condition detected | 0.58 |

**Result:** the bot executes Endgame first, then Intra-Market if capital allows - ranked purely by composite score.

### Signal Deduplication

Multiple strategies can generate signals for the same market. The risk manager blocks duplicate positions in a single market, so only the highest-ranked signal executes.

### Timeframe Handling

Cross-Platform and Momentum strategies run across 5m, 15m, and 1h timeframes simultaneously, letting the bot capture short-term scalps, swing trades, and longer position trades from the same market data.

---

## Configuration

Strategy parameters live in `config.py`, one configuration class per strategy:

- `IntraMarketConfig`
- `CombinatorialConfig`
- `CrossPlatformConfig`
- `EndgameConfig`
- `MomentumConfig`

Each strategy can be enabled or disabled independently via its `enabled` flag.

### Adjusting Strategy Weights

Modify `STRATEGY_PRIORITY` in the `StrategyAggregator` class:

```python
STRATEGY_PRIORITY = {
    "intra_market": 1.0,        # Increase for more risk-free arb focus
    "combinatorial": 0.95,
    "endgame": 0.90,
    "cross_platform": 0.80,     # Increase for more directional trades
    "momentum_mean_reversion": 0.70,  # Increase for more technical trades
}
```

### Adjusting Composite Score Weights

Modify the weights in `_composite_score()`:

```python
composite = (
    profit_score * 0.30 +      # Increase for profit-focused ranking
    confidence_score * 0.25 +   # Increase for confidence-focused ranking
    strategy_score * 0.20 +    # Increase for strategy-type preference
    urgency_score * 0.15 +     # Increase for time-sensitive trades
    rr_score * 0.10            # Increase for risk/reward focus
)
```

---

## FAQ: Polymarket Arbitrage Bot

**What is Polymarket arbitrage?**
Polymarket arbitrage exploits pricing inefficiencies in prediction markets - most commonly when YES + NO token prices don't sum to exactly $1.00, allowing a risk-free profit by buying both sides.

**Is a Polymarket arbitrage bot risk-free?**
Some strategies (intra-market, combinatorial) are structurally risk-free if executed correctly, since both outcomes are purchased for guaranteed redemption. Others (cross-platform, momentum) are directional and carry market risk.

**What timeframes does this Polymarket arbitrage bot support?**
The bot supports 5-minute, 15-minute, and 1-hour timeframes for its momentum and cross-platform strategies, allowing scalping, swing, and position-style trading from the same infrastructure.

**Which markets does the bot scan?**
Crypto prediction markets for BTC, ETH, XRP, and SOL, across all five strategies described above.

---

*Interested in strategy consulting or licensing this Polymarket arbitrage bot? Contact at Telegram: [@casatrick](https://telegram.me/casatrick)*
