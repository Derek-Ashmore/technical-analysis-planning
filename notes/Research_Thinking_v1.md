# Research Thinking v1: Consolidated Methodology and Reasoning

This document describes the reasoning process, key decisions, and confidence assessments behind all four research streams: individual indicators, backtesting methodology, indicator combinations, and data/architecture.

---

## Research Approach

The research was divided into four parallel workstreams to cover all dimensions of the requirements:

1. **Individual Indicators** -- What are the 25+ indicators, how do they work, and which perform best?
2. **Backtesting Methodology** -- How to properly evaluate indicator performance with statistical rigor?
3. **Indicator Combinations** -- How to combine indicators into effective trading systems?
4. **Data Engineering & Architecture** -- What are the data, tooling, and design considerations?

Each workstream conducted web research, synthesized academic and practitioner sources, and produced findings with source citations.

---

## Key Decisions and Reasoning

### On Indicator Selection and Tiering

The tiered ranking (Tier 1-4) was driven by a combination of factors:

- **Backtest evidence:** Indicators with published backtest results showing strong win rates, profit factors, and Sharpe ratios ranked higher. EMA, RSI, and MACD have the deepest published backtest evidence.
- **Universality:** Indicators that work across asset classes (commodities, forex, crypto, equities) ranked higher than those effective in only one class.
- **Complementarity:** The Tier 1 set was chosen to cover all four signal dimensions: trend (EMA, MACD), momentum (RSI), volatility (Bollinger Bands, ATR), and trend strength/filter (ADX).
- **Simplicity:** Indicators with fewer parameters and more straightforward signal generation were preferred, reducing overfitting risk.

ATR was included in Tier 1 despite not being a signal generator because it is essential for risk management. Every backtest that includes proper stop-loss and position sizing depends on ATR.

### On Market Regime Detection

The single most important finding across all four workstreams is that **no indicator works well in all market conditions**. This finding was consistent across academic literature, practitioner guides, and backtest results.

Trend-following indicators (MACD, EMA, MA crossovers) perform 2-3x better in trending markets. Mean-reversion indicators (RSI, Stochastic, Bollinger Bands) perform well in ranging markets but generate false signals in strong trends.

ADX was selected as the primary regime detector because:
- It is simple and well-understood
- ADX > 25 = trending, ADX < 20 = ranging is a widely validated threshold
- It does not require complex statistical modeling (unlike Hidden Markov Models or Hurst Exponent)
- It can be computed with standard TA-Lib functions

The regime detection recommendation directly shapes the architecture: the application should compute regime first, then select the appropriate indicator set.

### On Backtesting Framework Design

The central tension is between thoroughness and practicality.

**Walk-forward analysis (WFA)** is the gold standard but computationally expensive. **Simple backtesting** is fast but unreliable due to overfitting risk.

The hybrid approach was chosen: simple backtesting for initial screening of 25+ indicators (fast, broad), followed by WFA for validation of the top 5-10 (rigorous, targeted). This is practical because:
- Screening 25 indicators with WFA would be slow and unnecessary for Tier 3-4 indicators
- The top indicators need rigorous validation before being presented as "best fitting"
- Both results (IS and OOS) are reported for transparency

**Next-bar-open execution** was chosen because with daily bars, there is no way to know the intraday price path. Using the close of the signal bar as the execution price introduces look-ahead bias. Using the next open is conservative, realistic, and unambiguous.

### On Profitability Metrics

All three major risk-adjusted ratios (Sharpe, Sortino, Calmar) were recommended because each captures a different dimension:
- **Sharpe:** Total volatility (penalizes upside and downside equally)
- **Sortino:** Downside volatility only (more relevant for asymmetric strategies with stop-losses)
- **Calmar:** Maximum drawdown (most intuitive for practitioners -- "how much pain for how much gain?")

The composite scoring approach for ranking was chosen because no single metric captures all aspects. A strategy might have the best Sharpe but the worst drawdown, or the best win rate but too few trades for statistical significance.

### On Indicator Combinations

The recommendation to combine 2-4 indicators from different categories was driven by:
- **Multicollinearity avoidance:** Using RSI + Stochastic + Williams %R gives triple-weight to the same information (momentum). Using RSI + EMA + ADX covers momentum + trend + trend strength.
- **Diminishing returns:** Research converges on 2-4 as optimal. Beyond 4, conflicting signals become frequent and the system becomes over-optimized.
- **Practical evidence:** MACD + RSI + ADX filter shows 73% win rate with 65% false signal reduction -- this is the strongest published combination result.

Elder's Triple Screen was highlighted as a system-level recommendation because it elegantly solves the multi-timeframe problem: long-term trend filter + medium-term entry timing + short-term precise entry. It can be approximated on daily data using long-period, medium-period, and short-period indicators.

### On Technology Stack

**TA-Lib over pandas-ta:** Despite pandas-ta having a cleaner API, TA-Lib was chosen because:
- It is the reference implementation (20+ years in production)
- 2-5x faster (C implementation), which matters during parameter optimization
- Most comprehensive candlestick pattern library (60+ patterns)
- pandas-ta has sustainability concerns (potential archival by July 2026)

**vectorbt over Backtrader/Zipline:** The application's core use case is testing 25+ indicators across multiple parameter combinations. This is a parameter sweep problem:
- vectorbt: 10,000 parameter combos in seconds (vectorized)
- Backtrader: Same in minutes-hours (event-driven), and unmaintained since 2018
- Zipline: Installation complexity, US-equity focus

**CSV as primary input:** Universal compatibility, human-readable, trivially small for daily data. Parquet for internal caching only.

**YAML for configuration:** Supports comments (unlike JSON), handles nested structures cleanly (unlike TOML), human-readable.

### On the 30-Day Window

The 30-day default window aligns with ~1.5 calendar months (~22 trading days). This is:
- A natural swing trading period
- Long enough for most indicators to produce meaningful signals
- Short enough to be responsive to recent conditions
- **Independent of indicator parameters** -- the 30 days is for pattern detection, not for setting RSI to 30-period

The multi-scale analysis recommendation (15/30/60 days) addresses the limitation that patterns operate at different timescales. Agreement across scales increases confidence.

The warmup period problem is critical: with only 30 bars, a 50-day MA cannot be computed. The application must load additional data beyond the display window for warmup, or auto-scale indicator parameters to fit the window.

### On Asset Class Modes

The research identified significant differences across asset classes that cannot be ignored:
- Forex has no volume data -- volume indicators are useless
- Crypto needs adjusted RSI thresholds (80/25 not 70/30) and wider ATR-based stops
- Commodities favor trend-following more strongly than mean-reversion
- Equities require adjusted close data and have the richest volume information

Rather than building separate code paths, the recommendation is asset-class-aware default configurations. The same pipeline processes all asset classes, but defaults change.

---

## What Surprised Us

### Daily Data is Trivially Small
Even 50 years of data for one asset is under 1 MB in CSV. This means file format performance is irrelevant, everything fits in memory, and no database infrastructure is needed. The bottleneck is computation (parameter optimization), not I/O.

### Forex Volume Does Not Exist
Forex has no centralized exchange, so "volume" data from brokers is just tick count -- a weak proxy. An entire category of indicators (OBV, CMF, A/D, MFI) is unusable for forex. This was not addressed in the requirements and is a significant design consideration.

### Crypto "Daily Close" is Ambiguous
Unlike traditional markets with fixed closing times, crypto trades 24/7. The "daily close" depends on the data source (usually midnight UTC, but varies). The application should standardize on midnight UTC.

### Academic Evidence is Weaker Than Expected
While 87% of fund managers use technical analysis (Menkhoff, 2010), academic evidence for profitability is surprisingly weak after proper controls. The most robust evidence is for trend-following in commodities and FX -- exactly the kind of spot price data the requirements describe.

### The Indicator Survivorship Problem
The indicators we test (RSI, MACD, Bollinger Bands) are popular precisely because they appeared to work in past data. Hundreds of failed indicators were never published. This selection bias inflates expected performance. The application should acknowledge this.

---

## Confidence Levels

| Recommendation | Confidence | Notes |
|---------------|:-:|---|
| Tier 1 indicator selection | High | Supported by multiple independent backtest studies |
| Regime detection via ADX | Medium-High | ADX is one of several valid approaches; HMMs may be superior but more complex |
| Walk-forward analysis for validation | High | Well-established gold standard |
| Next-bar-open execution | High | Universally recommended for daily data |
| Composite scoring for ranking | Medium | Weights are subjective; no consensus on ideal weighting |
| 30-day default window | Medium | Reasonable default but optimal depends on instrument |
| Monte Carlo for confidence intervals | High | Standard practice; adds significant value |
| TA-Lib as primary library | High | Reference implementation, fastest, most comprehensive |
| vectorbt as backtesting engine | High | Best fit for parameter sweep use case |
| Pipeline architecture | High | Simplest architecture that meets all requirements |
| 2-4 indicators per combination | High | Converging evidence from research and practice |
| MACD + RSI + ADX as top combination | Medium-High | 73% win rate published, but results are market-dependent |
| Adaptive window approach | Low-Medium | Theoretically appealing but adds complexity and overfitting risk |
| Asset-class-aware defaults | High | Clear evidence of significant cross-asset differences |

---

## Connections Between Research Streams

The four workstreams reinforce each other:

1. **Individual indicator research** identified that regime matters most -- **backtesting methodology** confirmed this via academic evidence (AMH) -- **combinations research** provided the solution (ADX as regime filter) -- **architecture** implements this as a pipeline stage.

2. **Individual indicator research** ranked indicators by evidence quality -- **backtesting methodology** defined how to measure evidence quality (Sharpe, WFA, statistical significance) -- **combinations research** showed how top indicators combine -- **architecture** structures this as indicator calculation -> signal generation -> backtesting -> ranking.

3. **Backtesting methodology** identified overfitting as the primary risk -- **combinations research** proposed composite scoring to avoid single-metric optimization -- **architecture** implements configurable weights and mandatory buy-and-hold benchmarking.

4. **Data/architecture research** identified forex volume absence and crypto threshold adjustments -- **individual indicator research** confirmed asset-class-dependent performance -- **combinations research** showed which combinations work per asset class -- **architecture** implements asset-class modes in configuration.
