# Research Thinking v1

This document describes the reasoning process and thinking behind the technical analysis research.

---

## Research Approach

### How We Structured the Research

We decomposed the research into three parallel workstreams to maximize coverage and depth:

1. **Indicators Research** -- A comprehensive survey of all major technical analysis indicator categories (trend, momentum, volatility, volume, pattern-based). The goal was not just to list indicators but to document concrete, implementable signal rules for each.

2. **Backtesting & Profitability Research** -- The methodology layer. Understanding how to rigorously evaluate whether an indicator actually works, what metrics to measure, and how to avoid the many pitfalls that plague backtesting.

3. **Combinations & Architecture Research** -- The integration layer. How indicators work together, what the academic literature says about their effectiveness, what tools exist, and how the application should be structured.

This decomposition ensures we cover both the "what" (indicators), the "how" (methodology), and the "why" (evidence and architecture).

### Why We Organized Indicators by Category

We classified indicators into five categories (trend, momentum, volatility, volume, pattern-based) because this classification directly maps to how they should be used in a multi-indicator system:

- **Trend indicators** answer: "Which direction should I trade?"
- **Momentum indicators** answer: "Is it time to enter/exit?"
- **Volatility indicators** answer: "How much risk should I take?"
- **Volume indicators** answer: "Is this move genuine?"
- **Pattern-based systems** answer: "What's the complete picture?"

A well-designed application uses indicators from multiple categories in a hierarchical filtering approach, not indicators from the same category (which tend to be correlated and redundant).

---

## Key Decisions and Reasoning

### Why Walk-Forward Analysis Over Simple Backtesting

Simple backtesting optimizes and tests on the same data, which virtually guarantees overfitting. Walk-forward analysis is the industry standard because it:
- Always tests on unseen data (out-of-sample)
- Simulates the realistic scenario of adapting parameters over time
- Produces a sequence of OOS results that can be statistically evaluated
- The "Walk-Forward Efficiency" metric directly measures robustness

We recommend supporting both anchored (growing IS window) and rolling (fixed IS window) variants because they suit different strategy timeframes.

### Why ADX as the Primary Regime Detector

We identified a critical insight: the single most impactful design decision is detecting whether the market is trending or ranging, then selecting the appropriate indicator set. ADX is uniquely suited for this because:
- It directly measures trend strength (no other indicator does this)
- Clear, well-researched thresholds (< 20 ranging, > 25 trending)
- Simple to implement and interpret
- Can be enhanced later with HMM-based detection without changing the architecture

This avoids the fundamental problem of applying trend-following indicators in ranging markets (whipsaws) or mean-reversion indicators in trending markets (premature exits).

### Why Multi-Indicator Over Single-Indicator

Academic research and practitioner experience consistently show:
- No single indicator is sufficient (all have failure modes)
- Multi-indicator strategies outperform by ~23%
- Signal confirmation reduces false signals by up to 65% (MACD+RSI vs MACD alone)
- Volume confirmation increases Golden Cross accuracy from 54% to 72%

The hierarchical filtering approach (trend filter -> confirmation -> entry) is the best balance of complexity and effectiveness.

### Why Composite Scoring for "Best Fit"

The requirement to identify which indicators "best fit" the data requires a multi-criteria ranking system because:
- No single metric captures all aspects of performance
- Sharpe ratio alone doesn't capture drawdown risk
- Win rate alone is misleading (30% win rate can be profitable)
- The weighted composite approach allows the user to express their priorities

We recommend Sharpe (30%), MaxDD (25%), Profit Factor (20%), Expectancy (15%), Win Rate (10%) as defaults, but these should be configurable because different users have different risk tolerances.

### Why 30-Day Default Lookback Window

The 30-day default was chosen because:
- Long enough to filter daily noise
- Short enough to remain responsive to regime changes
- Captures most common chart patterns (H&S typically forms over 20-40 bars)
- Rule of thumb: window = half cycle length; monthly patterns = ~15 days, quarterly = ~30-45 days
- Widely used as a standard in financial analysis

However, this must be configurable because different assets have different characteristic cycle lengths.

### Library Choices

We recommend TA-Lib + vectorbt as the primary stack because:
- **TA-Lib**: C implementation is 2-4x faster, 150+ battle-tested indicators, most comprehensive candlestick pattern library (60+), active maintenance
- **vectorbt**: Fastest Python backtesting framework (NumPy + Numba), supports complex operations like trailing stops, excellent for parameter optimization
- **pandas-ta** is a good secondary option but has sustainability concerns (potential archive by July 2026)
- **backtrader** is no longer maintained (since 2018)

---

## What Surprised Us

### The Overfitting Problem Is Worse Than Expected

The backtesting research revealed how pervasive overfitting is:
- Knight Capital lost $440 million in 45 minutes from a deployed overfitted algorithm
- A 2023 study of 6,000+ trading rules found results are "very sensitive to the introduction of moderate transaction costs"
- Recently best-performing rules "perform significantly worse than buy-and-hold in the future"

This reinforces the need for:
- Strict walk-forward analysis (never test on training data)
- Parameter limits (3-5 core parameters max)
- Statistical validation (Monte Carlo, minimum trade counts)
- Transaction cost modeling from the start

### Volume Confirmation Is More Impactful Than Expected

Volume confirmation dramatically improved signal accuracy:
- Golden Cross with 40%+ volume increase: 72% accuracy
- Golden Cross without volume confirmation: only 54% accuracy
- This 18 percentage point improvement is one of the largest single improvements available

### Academic Evidence Is Mixed But Useful

The academic literature doesn't provide a clear "these indicators work" answer, but it does provide:
- Evidence that combining indicators outperforms single indicators
- Evidence that ML-enhanced TA approaches capture real effects
- Evidence that behavioral finance creates exploitable patterns
- Strong warnings about transaction costs eroding edge
- The EMH debate is less relevant than the practical question of "can we extract edge after costs?"

---

## Risks and Concerns

### pandas-ta Sustainability

pandas-ta has announced potential archival by July 2026 unless significant community support materializes. This is a risk for any application depending on it. Mitigation: use TA-Lib as primary, pandas-ta as optional secondary.

### Complexity vs Usability Tradeoff

The research reveals a tension between:
- **Completeness**: 27+ indicators, multiple combination strategies, regime detection, pattern recognition
- **Usability**: Users need clear, actionable results, not an overwhelming dashboard

The application should present a clear "best fit" ranking with supporting evidence, not dump all indicator results on the user.

### The "Best Fit" Problem

Determining which indicators "best fit" a specific dataset is inherently a moving target because:
- Market regimes change over time
- What worked in the past may not work in the future
- Overfitting to historical data is the constant danger

The application should be honest about this uncertainty and present confidence intervals, not point estimates.

---

## Research Gaps

Several areas warrant deeper investigation (documented in Research_Questions_v1.md):
- Specific asset class behavior (commodities vs equities vs forex)
- Real-world slippage and liquidity impact for the specific assets being traded
- Optimal rebalancing frequency for the regime detection system
- User experience design for presenting complex backtesting results
- Whether machine learning regime detection (HMM) provides enough improvement over simple ADX thresholds to justify the complexity
