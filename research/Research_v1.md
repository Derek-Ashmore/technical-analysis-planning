# Technical Analysis Application Research v1

## Executive Summary

This research addresses the requirements for an application that analyzes historical daily spot price data using technical analysis techniques to identify the most profitable buy/sell signals. The research covers four domains: (1) individual indicator analysis, (2) backtesting methodology, (3) indicator combination strategies, and (4) data engineering and architecture.

**Key findings:**

- **25 technical indicators** were evaluated across trend-following, momentum, volatility, volume, and combined system categories
- **6 Tier-1 indicators** emerged as must-implement: EMA, RSI, MACD, ATR, ADX, and Bollinger Bands
- **Market regime detection** (trending vs ranging) is the single most important factor -- no indicator works well in all conditions; ADX is the recommended regime filter
- **Walk-forward analysis** is the gold standard for backtesting validation; simple backtesting should be used for initial screening
- **2-4 indicators from different categories** should be combined to reduce false signals while avoiding redundancy
- **Composite scoring** with weighted metrics (Sharpe, Sortino, Calmar, Profit Factor, Expectancy) is recommended for ranking indicators objectively
- **TA-Lib + vectorbt** is the recommended technology stack (C-speed indicators + vectorized backtesting)
- **CSV input, pipeline architecture, YAML configuration** provide the best balance of simplicity and extensibility

Detailed findings for each research area are documented in supplementary files (referenced below) and summarized in this master document.

---

## Table of Contents

1. [Technical Indicators Analysis](#1-technical-indicators-analysis)
2. [Backtesting Methodology](#2-backtesting-methodology)
3. [Indicator Combination Strategies](#3-indicator-combination-strategies)
4. [Data Engineering and Architecture](#4-data-engineering-and-architecture)
5. [Consolidated Recommendations](#5-consolidated-recommendations)
6. [Supplementary Documents](#6-supplementary-documents)

---

## 1. Technical Indicators Analysis

> Full details: [research/technical-indicators-research.md](technical-indicators-research.md)

### 1.1 Indicators Evaluated

25 indicators were researched across five categories:

| Category | Indicators |
|----------|-----------|
| **Trend-Following** | SMA, EMA, MA Crossover (Golden/Death Cross), MACD, ADX, Parabolic SAR, Ichimoku Cloud |
| **Momentum/Oscillator** | RSI, Stochastic Oscillator, Williams %R, CCI, Rate of Change, Momentum |
| **Volatility** | Bollinger Bands, ATR, Keltner Channels, Standard Deviation |
| **Volume-Based** | OBV, Volume-Price Trend, Accumulation/Distribution, Chaikin Money Flow |
| **Combined/Advanced** | Moving Average Ribbon, Elder's Triple Screen, Donchian Channels (Turtle Trading), Supertrend |

### 1.2 Implementation Priority (Tiered Ranking)

**Tier 1 -- Must Implement:**

| Indicator | Key Finding | Best Signal Type |
|-----------|------------|-----------------|
| **EMA** | Outperforms SMA across all asset classes; 394% profit in comparative study; 1.92 profit factor | Standalone trend signal |
| **RSI (short-period, 2-6 days)** | Up to 91% win rate on specific setups; strongest mean-reversion results | Standalone mean-reversion |
| **MACD (12, 26, 9)** | 70-80% success in trending markets; 73% win rate combined with RSI | Standalone + confirmation |
| **ATR** | Essential for position sizing and stop-loss; foundation of Turtle Trading risk management | Risk management (not signal) |
| **ADX** | Critical regime filter; eliminates whipsaw signals when < 25; enables trend-following when > 25 | Trend strength filter |
| **Bollinger Bands (20, 2)** | Dual-purpose: mean reversion + breakout; +0.8% avg gain within 6 days below lower band | Mean reversion + breakout |

**Tier 2 -- High Priority:**

| Indicator | Key Finding |
|-----------|------------|
| **Williams %R** | Outperforms RSI and Stochastic in broad backtests; 81% win rate on S&P 500 |
| **Donchian Channels** | Foundation of Turtle Trading; historically proven for commodities; fully systematic |
| **Supertrend** | 67% win rate with 11.07% avg profit per trade when optimized; requires per-asset tuning |
| **MA Crossover Systems** | Golden Cross averaged 2.12% at 3 months; Death Cross 71% accuracy predicting further declines |

**Tier 3 -- Valuable Additions:** Keltner Channels, Stochastic, CCI, Parabolic SAR, Ichimoku Cloud

**Tier 4 -- Supplementary:** OBV, CMF, SMA, ROC, VPT, A/D Line, Standard Deviation, MA Ribbon, Momentum

**System-Level:** Elder's Triple Screen (implements as a complete multi-timeframe framework using Tier 1-2 indicators)

### 1.3 Performance by Market Regime

| Indicator Type | Trending Market | Ranging Market |
|---------------|:-:|:-:|
| MA Crossovers, MACD, Parabolic SAR | Strong (70-80% accuracy) | Very Poor (whipsaws) |
| RSI, Stochastic, Williams %R, Bollinger (mean reversion) | Fair (divergences) | Excellent (70-90% win rate) |
| ADX | Confirms trend strength | Correctly identifies no-trend |
| ATR | Wider stops needed | Tighter stops possible |
| Donchian/Turtle | Excellent (captures trends) | Accepts small losses |

**Critical insight:** ADX > 25 = use trend-following signals; ADX < 25 = use mean-reversion signals or stand aside.

### 1.4 Performance by Asset Class

| Indicator | Commodities | Forex | Crypto | Equities |
|-----------|:-:|:-:|:-:|:-:|
| EMA/SMA Crossovers | Strong | Strong | Strong | Strong |
| RSI | Moderate | Moderate | Strong (adjust thresholds: 80/25) | Strong |
| MACD | Strong | Strong | Strong | Strong |
| ADX | Very Strong | Strong | Moderate | Strong |
| Bollinger Bands | Strong | Moderate | Strong | Strong |
| Volume indicators | Limited (no vol) | Not applicable | Strong | Very Strong |
| Donchian/Turtle | Very Strong | High | High | Moderate |
| Ichimoku | Moderate | Strong | Very Strong (78% CAGR BTC) | Strong |

---

## 2. Backtesting Methodology

> Backtesting methodology details are consolidated in this master document. Additional context in [data-considerations-and-architecture.md](data-considerations-and-architecture.md) Sections 4 and 6.

### 2.1 Framework Design

**Recommended approach:** Hybrid -- simple backtesting for initial indicator screening, walk-forward analysis (WFA) for validation of top performers.

| Parameter | Recommended Value |
|-----------|------------------|
| In-sample window | 252 trading days (1 year) |
| Out-of-sample window | 63 trading days (1 quarter) |
| Execution assumption | Next-bar open + configurable slippage |
| Minimum data | 3-5 years (750-1,250 trading days) |
| Transaction costs | Configurable; default 0.1% round-trip |

**Critical bias prevention:**
- **Look-ahead bias:** Signal on bar T executes at bar T+1 open. Structurally enforce this.
- **Survivorship bias:** Acknowledge that tested indicators are pre-selected for historical popularity.
- **Data snooping:** Apply multiple-testing correction (Bonferroni/Holm-Bonferroni) when screening many indicators.
- **Overfitting:** Require trade-to-parameter ratio of at least 10:1; compare IS vs OOS performance.

### 2.2 Profitability Metrics

Report all of the following for each indicator:

| Metric | Formula/Description | Interpretation |
|--------|-------------------|---------------|
| **Sharpe Ratio** | (R_p - R_f) / sigma_p | > 1.0 = good; > 2.0 = very good |
| **Sortino Ratio** | (R_p - R_f) / sigma_downside | > 2.0 = strong |
| **Calmar Ratio** | Annualized Return / Max Drawdown | > 1.0 = return exceeds worst drawdown |
| **Max Drawdown** | Largest peak-to-trough decline | Report %, dates, and recovery time |
| **Win Rate** | Profitable trades / Total trades | Meaningless alone; pair with payoff ratio |
| **Profit Factor** | Gross Profit / Gross Loss | > 1.5 = good; > 2.0 = very good |
| **Payoff Ratio** | Average Win / Average Loss | Combined with win rate gives expectancy |
| **Expectancy** | (Win% x Avg Win) - (Loss% x Avg Loss) | Average $ expected per trade |

**Always compare against buy-and-hold benchmark.** Many indicators fail to beat passive holding after costs.

### 2.3 Trade Statistics

- Average holding period (trading days and calendar days)
- Average time between trades
- Trades per year, per month
- Maximum consecutive wins/losses
- Distribution of trade outcomes (histogram, percentiles, skewness)
- Average P&L per trade (absolute and %)
- Minimum 30 trades for statistical significance; 100+ preferred

### 2.4 Indicator Ranking

**Composite scoring approach with configurable weights:**

| Metric | Default Weight |
|--------|:---:|
| Sharpe Ratio | 0.15 |
| Sortino Ratio | 0.15 |
| Calmar Ratio | 0.15 |
| Profit Factor | 0.15 |
| Expectancy | 0.10 |
| Max Drawdown (penalty) | 0.15 |
| Number of Trades | 0.05 |
| OOS Performance Consistency | 0.10 |

Normalize each metric to [0,1] across all indicators, apply weights, rank by composite score.

### 2.5 Pattern Recognition (Configurable Windows)

- **Default 30-day rolling window** for pattern detection
- Multi-scale analysis recommended (15/30/60 days) for confirmation
- Detect: trend direction, indicator signal clustering, volatility expansion/contraction, higher highs/lows structure
- **Regime detection** via ADX thresholds (primary) + volatility ratio (confirming)
- Adaptive window option: shorten when volatility is high, lengthen when low

### 2.6 Academic Evidence

| Period | Finding |
|--------|---------|
| Pre-1990 | Generally positive evidence for TA profitability |
| 1990-2005 | Declining evidence, especially in developed equities |
| Post-2005 | Little evidence in developed stock markets after cost/snooping controls |
| FX & Commodities | More persistent evidence of profitability |
| Crypto & Emerging | Positive evidence (less efficient markets) |

**Adaptive Market Hypothesis (Lo, 2004):** Markets are not always efficient; strategies have life cycles. The most useful output is "which indicators work in which conditions" rather than "which indicator is permanently best."

---

## 3. Indicator Combination Strategies

### 3.1 Principles

- **Combine 2-4 indicators from different categories** (trend + momentum + volatility + volume) to reduce false signals
- **Avoid redundancy:** RSI, Stochastic, and Williams %R all measure the same thing -- pick one per category
- **Confirmation signals** across dimensions are stronger than same-dimension confirmation
- **Composite scoring** with weighted voting is the most systematic approach

### 3.2 Best Indicator Combinations Identified

| Combination | Expected Performance | Best For |
|------------|---------------------|----------|
| **MACD + RSI + ADX filter** | 73% win rate, 0.88% avg gain, 65% false signal reduction | Trending markets |
| **Donchian Breakout + ATR sizing + Supertrend trail** | 35-45% win rate, large winners | Commodities, trend-following |
| **Elder's Triple Screen (Weekly MACD + Daily RSI + ATR entry)** | 31% annual return on EURUSD | Multi-timeframe systems |
| **RSI + Bollinger Bands (mean reversion)** | 70-90% win rate, small per-trade gains | Ranging markets |
| **MACD + Bollinger Bands** | 78% win rate reported | Momentum at volatility extremes |

### 3.3 Signal Generation Methods

1. **Threshold-based:** RSI < 30 = buy, RSI > 70 = sell (best in ranging markets)
2. **Crossover-based:** MACD signal line cross, MA crossovers (most reliable with volume confirmation)
3. **Divergence-based:** Price vs indicator disagreement (powerful but hard to automate)
4. **Breakout:** Donchian/Bollinger Band breakouts (best after squeeze/consolidation)
5. **Composite scoring:** Multiple indicators vote with weights; act when score exceeds threshold

### 3.4 Position Management

- **Entry:** Composite score > threshold; confirm with trend direction
- **Stop-loss:** ATR-based (2-3x ATR for daily data); automatically widens in volatile markets
- **Trailing stop:** Supertrend, Parabolic SAR, or ATR-based Chandelier Exit
- **Position sizing:** Risk 1-2% of capital per trade; size = Account Risk / (ATR x Multiplier)
- **Exit:** Indicator-based (RSI reversal, opposite MA cross), time-based (max holding period), or target-based (reward/risk ratio)

### 3.5 Adaptive Approaches

- **Regime switching:** Use ADX/volatility ratio to detect trending vs ranging; switch indicator sets accordingly
- **Adaptive parameters:** Shorten lookback periods when volatility is high; lengthen when low
- **Walk-forward re-optimization:** Re-optimize parameters quarterly using the most recent 1-year window
- **Ensemble methods:** Majority voting or weighted ensemble of multiple indicator strategies

---

## 4. Data Engineering and Architecture

> Full details: [research/data-considerations-and-architecture.md](data-considerations-and-architecture.md)

### 4.1 Spot Price Data

- Spot prices are the simplest price representation -- no contract rolls, no time decay
- Standard format: OHLCV (Open, High, Low, Close, Volume)
- Validation rules: High >= max(Open, Close), Low <= min(Open, Close), Volume >= 0
- For commodities, "spot" often means front-month futures proxy -- user should clarify

### 4.2 Asset Class Considerations

| Asset Class | Volatility (Ann.) | Volume Data | Key TA Consideration |
|-------------|:-:|:-:|---|
| Commodities | 12-60% | Limited for spot | Trend-following dominates; Donchian/Turtle ideal |
| Forex | 6-25% | None (tick vol only) | Disable volume indicators; MA crossovers work well |
| Crypto | 50-300% | Available | Adjust RSI thresholds (80/25); wider ATR stops (3x) |
| Equities | 12-50% | Best available | All indicators work; require adjusted close data |

### 4.3 Data Requirements

| Purpose | Minimum | Recommended |
|---------|:---:|:---:|
| Indicator calculation | 1 year | 2 years |
| Basic backtesting | 2 years | 5 years |
| Walk-forward analysis | 3 years | 5-10 years |
| Regime analysis | 5 years | 10+ years |

Daily data is trivially small: 50 years of data for one asset is < 1 MB in CSV.

### 4.4 Technology Stack

| Component | Recommended | Rationale |
|-----------|-----------|-----------|
| Indicator computation | **TA-Lib** | C speed, 150+ indicators, 60+ candlestick patterns, 20+ years battle-tested |
| Backtesting | **vectorbt** | 10-100x faster than event-driven; ideal for parameter sweeps |
| Static charts | **mplfinance** | Purpose-built for financial charts; PDF-ready |
| Interactive charts | **Plotly** | Best interactivity; HTML output |
| Reports | **Jinja2 + HTML** | Flexible, professional |
| Configuration | **YAML** | Human-readable, supports nested config |
| Data format | **CSV (input), Parquet (cache)** | Universal input, fast caching |

### 4.5 Application Architecture

Pipeline architecture: **Load -> Validate -> Calculate -> Signal -> Backtest -> Report**

```
src/
  config.py                 # Configuration loading
  data/
    loader.py               # CSV/API data loading
    validator.py             # OHLCV validation, cleaning
  indicators/
    base.py                 # Indicator base class + registry
    trend.py                # SMA, EMA, MACD, Ichimoku, SAR
    momentum.py             # RSI, Stochastic, Williams %R, CCI
    volatility.py           # Bollinger, ATR, Keltner, Donchian
    volume.py               # OBV, A/D, CMF
  signals/
    generator.py            # Signal generation from indicators
    regime.py               # ADX-based regime detection
    composite.py            # Multi-indicator scoring
  backtest/
    engine.py               # vectorbt wrapper
    walk_forward.py         # WFA orchestration
    metrics.py              # Performance metrics
    scoring.py              # Composite ranking
  reporting/
    charts.py               # mplfinance + Plotly
    html_report.py          # Jinja2 HTML generation
    export.py               # CSV export
  pipeline.py               # Main orchestration
```

### 4.6 Configuration (YAML)

Key configurable elements:
- Data source (file path or API)
- Asset class (commodity, forex, crypto, equity) -- drives default indicator selection
- Lookback window (default 30 days for pattern detection, independent of indicator parameters)
- Indicators to test (with per-indicator parameters)
- Backtest settings (capital, costs, slippage, long-only/long-short)
- Ranking weights
- Regime detection settings
- Output format (HTML, PDF, CSV)

---

## 5. Consolidated Recommendations

### 5.1 Architecture

1. **Pipeline architecture:** Load -> Validate -> Calculate -> Signal -> Backtest -> Report
2. **Regime detection first:** ADX-based trending/ranging classification should be built into the core pipeline
3. **Next-bar-open execution:** Signals on bar T execute at bar T+1 open with configurable slippage
4. **Look-ahead bias prevention:** Structurally enforce at the architecture level
5. **Configurable transaction costs:** Default 0.1% round-trip; always show results with and without costs

### 5.2 Indicator Selection

6. **Start with Tier 1:** EMA, RSI, MACD, ATR, ADX, Bollinger Bands
7. **Add Tier 2:** Williams %R, Donchian Channels, Supertrend, MA Crossovers
8. **Implement Elder's Triple Screen** as a complete system framework
9. **Test combinations:** MACD+RSI+ADX and RSI+Bollinger as default combinations
10. **Asset-class-aware defaults:** Disable volume indicators for forex; adjust crypto thresholds

### 5.3 Evaluation and Reporting

11. **Always compare against buy-and-hold** benchmark
12. **Report comprehensive metrics:** Sharpe, Sortino, Calmar, Max Drawdown, Win Rate, Profit Factor, Expectancy
13. **Rank using composite score** with configurable weights
14. **Report by market regime:** Which indicators work in trending vs ranging conditions
15. **Flag potential overfitting:** High profit factor + few trades + unstable parameters = red flag
16. **Monte Carlo simulation** (1,000+ iterations) for confidence intervals

### 5.4 Pattern Recognition

17. **Default 30-day rolling window** with multi-scale analysis (15/30/60)
18. **Detect regime within each window** (ADX + volatility ratio)
19. **Load warmup data** beyond the analysis window for indicator initialization
20. **Auto-scale indicator parameters** when window size changes

### 5.5 Statistical Rigor

21. **Minimum 30 trades** for basic significance; 100+ preferred
22. **Multiple-testing correction** when screening many indicators
23. **Report temporal stability:** Rolling annual performance to detect decaying edge
24. **Walk-forward analysis** for all top-performing indicators
25. **Parameter sensitivity heatmaps** to verify robustness

---

## 6. Supplementary Documents

### Research Details

| Document | Contents |
|----------|---------|
| [research/technical-indicators-research.md](technical-indicators-research.md) | Detailed analysis of 25 indicators: calculations, parameters, signal generation, strengths/weaknesses, profitability, 90+ source citations |
| [research/data-considerations-and-architecture.md](data-considerations-and-architecture.md) | Spot price data characteristics, asset class analysis, file formats, Python libraries, visualization, application architecture, YAML config schema |

### Thinking and Methodology

| Document | Contents |
|----------|---------|
| [notes/Research_Thinking_v1.md](../notes/Research_Thinking_v1.md) | Reasoning behind key decisions, confidence levels, architectural implications |

### Open Questions

| Document | Contents |
|----------|---------|
| [notes/Research_Questions_v1.md](../notes/Research_Questions_v1.md) | 33 questions organized by category that would refine implementation |

---

## Sources

### Academic Papers
1. Brock, Lakonishok & LeBaron (1992) -- Simple Technical Trading Rules and Stock Returns
2. Park & Irwin (2007) -- What Do We Know About the Profitability of Technical Analysis?
3. Lo (2004) -- The Adaptive Markets Hypothesis
4. Sullivan, Timmermann & White (1999) -- Data-Snooping and Technical Trading Rule Performance
5. Bajgrowicz & Scaillet (2012) -- Technical Trading Revisited: False Discoveries
6. Lo, Mamaysky & Wang (2000) -- Foundations of Technical Analysis
7. Menkhoff (2010) -- The Use of Technical Analysis by Fund Managers
8. de Prado (2018) -- Advances in Financial Machine Learning (CPCV methodology)

### Backtesting and Strategy Studies
- QuantifiedStrategies.com -- Extensive backtested results for RSI, MACD, Williams %R, Bollinger Bands, Donchian, Supertrend, Keltner, CCI, Ichimoku, Elder Triple Screen, and combination strategies
- LiberatedStockTrader -- Large-scale backtesting (15,000+ trades for Ichimoku, 4,000+ for Supertrend, 2,880 years for Parabolic SAR)
- FXSSI -- RSI vs CCI vs Williams %R comparative performance study

### Additional Sources
- Full citation lists available in supplementary documents
