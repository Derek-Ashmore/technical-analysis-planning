# Research Thinking v2: Methodology, Reasoning, and Decision-Making

This document describes the reasoning process, key decisions, confidence assessments, and trade-offs behind the v2 research revision. It is the companion "thinking" document to `research/Research_v2.md`.

---

## 1. Revision Context

### What Triggered the Revision

Research v1 was conducted without knowing the answers to several critical scope questions (documented in `notes/Research_Questions_v1.md`). The five most important unknowns were:

1. **Target asset class** (Q1.1) -- v1 covered all four asset classes generically
2. **Long-only or long-short** (Q2.1) -- v1 presented both without preference
3. **Combination testing scope** (Q3.1) -- v1 deferred the decision
4. **What constitutes a "pattern"** (Q5.1) -- v1 offered four interpretations
5. **Single or multiple instruments** (Q7.1) -- v1 designed for both

The user's answers to these questions eliminated significant ambiguity and invalidated several v1 assumptions. The revision was not triggered by errors in v1 but by the scope collapsing from "general-purpose technical analysis tool" to "equities-focused, long-only, single-instrument backtest and signal system."

### What Changed Between v1 and v2 Requirements

| Dimension | v1 Assumption | v2 Confirmed Scope | Impact |
|-----------|---------------|-------------------|--------|
| Asset class | All four (commodities, forex, crypto, equities) | **Equities only** | Removes forex volume workaround, crypto threshold adjustments, commodity contract roll logic |
| Trading direction | Long-only and long-short both considered | **Long-only** | Simplifies backtesting, removes short-selling cost modeling, changes indicator evaluation criteria |
| Position sizing | Risk-based sizing recommended (1-2% risk per trade) | **100% in/out** (single instrument) | Eliminates position sizing calculations, simplifies to binary signal evaluation |
| Instruments | Potentially multi-instrument | **Single instrument at a time** | Removes correlation analysis, portfolio-level metrics, data alignment |
| Trade frequency | No explicit target | **~30 calendar days average** | Adds frequency constraint/preference to scoring and parameter selection |
| Transaction costs | 0.1% round-trip default | **Negligible** (but we use 0.02% token cost) | Removes cost sensitivity analysis, but token cost prevents degenerate solutions |
| Max drawdown | Not constrained | **30% hard limit** | Adds circuit breaker to backtest engine, changes from preference to constraint |
| Combinations | 5-6 proven combinations suggested | **5 curated combinations** confirmed | Prevents combinatorial explosion, mandates pre-selection based on evidence |
| Data format | Generic CSV/API | **Nasdaq.com CSV format** specifically | Enables targeted parser, known column structure |
| Output format | HTML with optional PDF | **HTML primary** confirmed | Removes PDF dependency (WeasyPrint), simplifies reporting |
| Re-analysis frequency | Not specified | **Two modes: full analysis + daily signal check** | Adds architectural distinction between optimization runs and operational runs |
| Future extension | Not discussed | **Options analysis readiness** | Adds Instrument and PositionLeg model stubs |

The net effect is a dramatic scope reduction. v1 was designing for a tool that could analyze gold futures, EUR/USD, Bitcoin, and NVDA. v2 is designing for a tool that analyzes NVDA (and similar equities). This is not a limitation -- it is a focusing decision that allows deeper, more specific recommendations.

---

## 2. Research Approach

The v2 research was conducted through four parallel workstreams, matching the v1 structure but with narrowed scope:

### Workstream 1: Indicator Re-Evaluation (Equities + Long-Only Focus)

Re-examined all 25 indicators through the lens of:
- How does this indicator perform specifically on equities?
- Does long-only constraint help or hurt performance?
- Is volume data available and informative for this indicator on equities?
- Does the indicator naturally produce signals at approximately 30-day frequency?

This workstream produced the most significant tier changes from v1.

### Workstream 2: Backtesting Framework Refinement

Revisited all backtesting parameters with:
- Known trade frequency target (~30 days) informing window sizing
- Known drawdown constraint (30%) requiring hard filter implementation
- Known position sizing (100% in/out) simplifying the backtest engine
- Known cost structure (negligible) removing cost sensitivity analysis
- Individual equity volatility patterns (noisier than indices or commodities)

### Workstream 3: Data and Architecture Revision

Simplified architecture for:
- Single asset class (no mode switching)
- Known data format (Nasdaq CSV)
- Two operational modes (full analysis vs. daily signal)
- Options extension readiness without current implementation

### Workstream 4: Gap Analysis

Identified what v1 missed or could not address without scope answers:
- Specific drawdown behavior of individual equities (NVDA's 66% drawdown in 2022)
- Equity-specific volume analysis capabilities
- Benchmark comparison methodology (buy-and-hold of the stock AND S&P 500)
- New library recommendations enabled by narrower scope (quantstats, exchange_calendars)

---

## 3. Key Decisions and Reasoning

### 3.1 Indicator Re-Ranking Reasoning

#### Williams %R Upgrade: Tier 2 to Tier 1

The published evidence shows an 81% win rate on the S&P 500. This alone would justify Tier 1 consideration, but the critical insight is subtler.

The backtests specifically noted that short-side Williams %R trades "didn't work well" on equities. In a long-short context, this is a weakness -- you are leaving performance on the table during downtrends. In v1, this was treated as a mild negative.

In a long-only context, this finding inverts. If short signals are weak, that means long signals carry the entire 81% win rate. The indicator is not splitting its performance across two trade directions; all of its edge is concentrated in the direction the user actually trades. This is a rare case where an external constraint (long-only) does not reduce performance but actually improves signal quality by eliminating the weaker side.

The reasoning chain: long-only is the constraint -> Williams %R long signals are specifically strong -> short signal weakness becomes irrelevant -> performance estimate improves -> Tier 1 justified.

#### OBV and CMF Upgrade: Tier 4 to Tier 2

In v1, all volume indicators were placed in Tier 4. The reasoning was: volume data is unavailable for forex (the largest FX market), unreliable for commodities (spot has no exchange volume), and of uncertain quality for crypto. This penalized volume indicators across the board.

With equities confirmed as the sole asset class, this penalty is removed entirely. Equities have the highest-quality, most informative volume data of any asset class. Institutional order flow constitutes 70-80% of equity volume, making accumulation/distribution patterns genuinely detectable. When a large fund is building a position in NVDA over weeks, OBV and CMF can identify the accumulation before price moves significantly.

The v1 tier ranking was biased by multi-asset considerations. Volume indicators were universally penalized because they are useless for forex and questionable for commodities. Removing those asset classes from scope immediately elevates the entire volume category. The upgrade to Tier 2 (not Tier 1) reflects that volume is a confirming signal, not a standalone signal generator.

#### Donchian Channels Downgrade: Tier 2 to Tier 3

v1 itself documented that Donchian/Turtle Trading "works well for commodities and currencies, but not for stocks." The Turtle Trading system was designed for commodity futures with high leverage and portfolio diversification across 20+ markets. Its 35-45% win rate depends on rare, large trend captures that are more common in commodities.

Individual equities have different dynamics. They spend 60-70% of their time ranging, which produces frequent small losses for breakout systems. The large trend captures that make Donchian profitable in commodities are less frequent and less extreme in equities (a stock rarely makes a sustained multi-month trend without fundamental catalysts).

However, Donchian is not dropped to Tier 4 because inverted Donchian (mean-reversion -- buy near the lower channel, sell near the upper) is viable for equities in ranging markets. The indicator retains some utility in the opposite direction from its original design.

#### Volume Indicators Overall

The principle: indicator quality depends on the quality of the data it consumes. Volume indicators consume volume data. Volume data quality ranking:

1. **Equities** -- centralized exchanges, mandatory reporting, institutional flow visible
2. **Crypto** -- available but subject to wash trading and fragmentation
3. **Commodities** -- futures volume is reliable, spot volume often unavailable
4. **Forex** -- no centralized volume, tick count is a weak proxy

v1 evaluated indicators against the worst-case data quality (forex). v2 evaluates against the best-case (equities). The entire volume category shifts upward.

### 3.2 Backtesting Framework Reasoning

#### 504-Day In-Sample Window (vs. 252 in v1)

The v1 recommendation of 252 trading days (1 year) was based on general practice. With the trade frequency target now known (~30 calendar days, or ~21 trading days), we can calculate the expected number of trades per window:

- 252-day IS window / 21-day trade frequency = ~12 trades per window
- With some indicators producing fewer signals, this drops to 8-10

A parameter optimization on 8-10 trades is marginal. The trade-to-parameter ratio of 10:1 recommended in v1 would allow optimizing only 1 parameter with 10 trades. For a system with 2-3 parameters (e.g., RSI period, overbought threshold, oversold threshold), this is insufficient.

504 trading days (~2 years) produces 16-24 trades per window. This is still not large-sample territory, but it is sufficient for 2-3 parameter optimization without severe overfitting. The cost is that the walk-forward analysis has fewer steps over a fixed data history (e.g., 10 years of data = ~5 WFA steps with 504/126 windows, vs. ~10 steps with 252/63 windows).

Additionally, individual equities are noisier than commodity or forex prices. A single earnings report can create a 10-20% gap that distorts indicator signals. More data smooths out the impact of individual events.

#### 126-Day Out-of-Sample Window (vs. 63 in v1)

The same trade frequency logic applies. With 63-day OOS windows and ~21-day trade frequency, each OOS period contains only 2-3 trades. Evaluating a strategy's out-of-sample performance on 2-3 trades is essentially meaningless -- the variance is too high.

126 days yields 4-6 trades per OOS period. This is still minimal but allows at least a rough assessment of whether the strategy continues to function. The alternative (keeping 63-day OOS) would require aggregating multiple OOS periods before drawing conclusions, which the walk-forward framework does anyway, but individual period performance would be noise.

#### Rolling Windows (Not Anchored)

For individual equities, a company's fundamental characteristics can change dramatically over a decade. NVDA in 2016 was a gaming GPU company trading at $30. NVDA in 2024 is an AI infrastructure company trading at $800+ (split-adjusted). The price dynamics, volatility patterns, trading volume, and institutional ownership are fundamentally different.

Anchored walk-forward analysis (where the IS window grows to include all prior data) would give ever-increasing weight to early data. A 2016 ranging period for NVDA would permanently influence parameter optimization even when the 2023-2024 trend data is far more relevant. Rolling windows ensure the optimization always uses the most recent N days, naturally adapting to the company's evolution.

For broad market indices or commodity prices, anchored windows are more defensible because the underlying dynamics are more stable. But for individual equities, rolling is strongly preferred.

#### Token Transaction Cost (0.02%) Instead of Zero

The user stated that transaction costs are negligible. Setting costs to exactly zero is tempting but creates a subtle problem: it fails to penalize excessive trading. A strategy that trades every day and earns $0.01 per trade would rank identically to one that trades monthly and earns $0.30 per trade, all else being equal. In reality, every trade carries execution risk, slippage, and opportunity cost that are not captured in a zero-cost model.

A token cost of 0.02% (0.01% per side) serves as a tiebreaker. It is small enough that it will not materially change the performance metrics of any reasonable strategy. But if two strategies have similar gross returns, the one that trades less frequently will rank slightly higher. If moving from 0% to 0.02% costs materially changes a strategy's ranking, that strategy is fragile and probably not worth implementing.

The value 0.02% was chosen as approximately half of the minimum commission on a discount broker for a reasonably sized equity trade. It is a lower bound, not an estimate.

### 3.3 Drawdown Constraint Reasoning

#### Hard Filter (Reject), Not Soft Penalty

The user specified 30% as the "maximum acceptable drawdown." The word "maximum" implies a constraint, not a preference. There is a meaningful difference:

- **Soft penalty (scoring):** A strategy with 35% max drawdown gets a lower composite score but could still rank first if other metrics are excellent.
- **Hard filter (reject):** A strategy with 35% max drawdown is eliminated from consideration regardless of other metrics.

The user's language suggests the second interpretation. "Maximum acceptable" means strategies exceeding it are not acceptable. The scoring system still differentiates among passing strategies (a 15% drawdown scores higher than a 25% drawdown), but the 30% threshold is a pass/fail gate.

#### Circuit Breaker in Backtest

Without a circuit breaker, the backtest might report a strategy with 45% max drawdown but excellent recovery and total return. The user's constraint says this is unacceptable. But more importantly, without simulating the exit at 30%, the reported metrics are misleading.

If a strategy hits 30% drawdown and the user would exit in practice, then the backtest should model that exit. Otherwise, the recovery period (which the user would never experience because they already exited) inflates the total return. The circuit breaker ensures that reported metrics accurately reflect what would have happened under the user's risk management rules.

#### 30% is Aggressive for Individual Stocks

This was one of the most significant findings. NVDA specifically experienced:
- 56% drawdown in Q4 2018 (crypto mining bust)
- 66% drawdown in 2022 (broader tech selloff)
- Multiple 30%+ drawdowns throughout its history

Buy-and-hold NVDA would be **rejected** by the 30% drawdown constraint in multiple periods. This is not a flaw -- it is the system working as designed. The constraint forces the system to find strategies that would have exited before the worst of these declines. A strategy that stays fully invested through a 66% drawdown is not viable under the user's risk parameters, even if NVDA recovered to new highs afterward.

This also means that any strategy presented as "passing" has demonstrated an ability to sidestep or reduce at least some of the worst equity declines. That is a much stronger claim than simply beating buy-and-hold on total return.

### 3.4 Trade Frequency Reasoning

#### Soft Constraint (Scoring Penalty), Not Hard Filter

The user specified 30 days as a "default," which is meaningfully different language from "maximum acceptable." A default implies a starting point or preference, not a hard requirement. A strategy that averages 45 days between trades but produces excellent risk-adjusted returns should not be eliminated. Similarly, a strategy that averages 15 days between trades might be acceptable if it does not generate excessive costs.

The implementation is a scoring penalty: strategies near 30 calendar days (21 trading days) get the full frequency score. Strategies deviating from this target lose points proportionally. A strategy averaging 60 days between trades loses some frequency score but can still rank highly if its other metrics are strong.

#### 21 Trading Days Equals 30 Calendar Days

The conversion factor is 252 trading days / 365 calendar days = 0.69. So 30 calendar days x 0.69 = 20.7, rounded to 21 trading days. This accounts for weekends (104 days/year) and NYSE holidays (~9 days/year). Using 30 trading days instead of 21 would correspond to approximately 43 calendar days, significantly longer than the user's target.

#### Standard Parameters Naturally Fit

This was a reassuring finding. The most widely-used indicator parameters were designed for swing and position trading on daily data:

- Standard MACD(12,26,9) produces signal line crossovers roughly every 20-40 trading days on typical equities
- Standard RSI(14) with 30/70 thresholds produces overbought/oversold signals roughly every 15-35 trading days
- Bollinger Bands(20,2) produce band touches roughly every 15-25 trading days

These frequencies align naturally with the ~21 trading day target. This means the application does not need aggressive parameter re-tuning to hit the frequency target. Standard textbook parameters are a reasonable starting point, and the walk-forward optimization can fine-tune from there.

### 3.5 Combination Strategy Reasoning

#### 5 Curated Combinations, Not Algorithmically Generated

With 15 indicators (Tier 1 + Tier 2) and combinations of 2-4, the number of possible combinations is:

- 2-indicator: C(15,2) = 105
- 3-indicator: C(15,3) = 455
- 4-indicator: C(15,4) = 1,365
- Total: 1,925 combinations

Testing 1,925 combinations with walk-forward analysis, each with 3+ parameters, would produce millions of parameter configurations. With only 5 slots for the final selection, the overfitting risk from this search is enormous. The winning combination would likely be the one that happened to fit the specific in-sample data best, not the one with genuine predictive power.

Pre-defined combinations based on published research evidence sidestep this problem. The 5 combinations are selected before seeing the data, based on theoretical complementarity and published performance. This is analogous to pre-registering hypotheses in scientific research -- it does not prevent overfitting within each combination's parameter space, but it prevents selection bias across combinations.

#### Coverage of Strategy Archetypes

The 5 combinations are designed to cover distinct strategy archetypes:

1. **MACD + RSI + ADX filter** -- Classic trend-following with momentum confirmation and regime filter. The "workhorse" combination.
2. **RSI + Bollinger Bands** -- Pure mean-reversion. Works in ranging markets where combination 1 fails.
3. **Williams %R + OBV + EMA** -- Momentum timing with volume confirmation. Unique angle: volume adds information not present in other combinations.
4. **EMA crossover + Supertrend trailing stop** -- Trend-following with systematic exit. Focuses on trade management rather than entry timing.
5. **ADX + Bollinger Bands + ATR** -- Volatility breakout with trend confirmation. Captures regime transitions.

Each combination uses indicators from different categories, avoiding the multicollinearity problem (e.g., not combining RSI + Stochastic + Williams %R, which all measure the same momentum dimension).

#### Individual Indicators Tested First

The 5-combination limit applies to multi-indicator strategies. All Tier 1 and Tier 2 indicators are first tested individually as a screening step. This serves two purposes: (1) understanding each indicator's standalone performance on the specific equity, and (2) identifying which individual indicators show the most promise for the specific asset, which informs interpretation of combination results.

### 3.6 Composite Scoring Reasoning

#### Changes from v1 Weights

| Metric | v1 Weight | v2 Weight | Reasoning for Change |
|--------|:-:|:-:|---|
| Sharpe Ratio | 0.15 | 0.15 | Unchanged -- primary risk-adjusted return measure |
| Sortino Ratio | 0.15 | 0.15 | Unchanged -- captures downside risk specifically |
| Calmar Ratio | 0.15 | 0.15 | Unchanged -- relates return to worst drawdown |
| Profit Factor | 0.15 | 0.15 | Unchanged -- intuitive gross profit/loss ratio |
| Expectancy | 0.10 | **0.05** | Reduced -- less independent once Sharpe and Profit Factor are included |
| Max Drawdown (penalty) | 0.15 | **0.10** | Reduced -- now a hard constraint (pass/fail at 30%), so scoring only differentiates among passing strategies |
| Number of Trades | 0.05 | 0.05 | Unchanged -- penalizes insufficient sample size |
| OOS Consistency | 0.10 | 0.10 | Unchanged -- guards against overfitting |
| Trade Frequency Alignment | -- | **0.05** | New -- penalizes strategies far from 30-day target |
| Benchmark Outperformance | -- | **0.05** | New -- rewards beating both buy-and-hold and S&P 500 |

**Expectancy reduction (0.10 to 0.05):** Expectancy = (Win% x AvgWin) - (Loss% x AvgLoss). This is mathematically related to both Sharpe (via mean return) and Profit Factor (via win/loss ratio). Giving it 10% weight when Sharpe and Profit Factor together get 30% overweights the same underlying information. Reducing to 5% treats it as a tiebreaker rather than a primary criterion.

**Max Drawdown reduction (0.15 to 0.10):** In v1, max drawdown was both a scoring component and an implicit preference. In v2, it is explicitly a hard filter at 30%. Among strategies that pass the filter (all have max drawdown < 30%), the scoring component differentiates a 10% drawdown from a 25% drawdown. But since the worst cases are already eliminated, the scoring component does less work and deserves less weight.

**Trade Frequency Alignment (new, 0.05):** The user's 30-day preference was not part of v1 scoring. A small weight (5%) creates a gentle pull toward the target frequency without dominating the score. A strategy averaging 45 days between trades loses ~0.025 points (half of 0.05), which is unlikely to change its overall ranking unless it is in a tight race with another strategy near 30 days.

**Benchmark Outperformance (new, 0.05):** v1 recommended comparing against buy-and-hold but did not include it in the composite score. Adding it as a scored metric rewards strategies that actually beat the simplest alternative. The weight is small because beating buy-and-hold is necessary but not sufficient -- a strategy that barely beats buy-and-hold with twice the volatility is not desirable.

#### Minimum Trades Reduced: 30 to 20 per WFA Window

v1 recommended 30 trades for statistical significance, which remains the global minimum. However, requiring 30 trades per WFA in-sample window is now too restrictive. With ~12 trades/year and 2-year IS windows, each window produces ~24 trades. A 30-trade minimum would cause most windows to fail, effectively preventing walk-forward analysis.

Reducing to 20 trades per window is a pragmatic compromise. The global minimum across all OOS periods combined remains 30 (aggregated), but individual windows can contribute with 20+ trades. This allows WFA to function while still filtering out indicators that produce too few signals for meaningful evaluation.

### 3.7 Architecture Reasoning

#### Equities-Only Simplification

Removing multi-asset support eliminates the following v1 components:

- **Asset class mode selector** in configuration (commodity/forex/crypto/equity)
- **Forex volume workaround** (disabling volume indicators, substituting tick volume)
- **Crypto threshold adjustments** (RSI 80/25 instead of 70/30, wider ATR multipliers)
- **Commodity contract roll logic** (detecting and handling front-month futures rolls)
- **Asset-class-aware default configurations** (four separate default profiles)

This is significant complexity reduction. Each eliminated component also eliminates its corresponding tests, documentation, edge cases, and maintenance burden. The resulting codebase is narrower but can be deeper -- more sophisticated handling of equity-specific concerns (earnings gaps, dividend adjustments, corporate actions) instead of surface-level handling of four asset classes.

#### Two Run Modes: Full Analysis and Daily Signal

Full re-analysis daily is wasteful and potentially harmful. The "most profitable indicator" can change day-to-day based on a single new data point. Running the full optimization pipeline daily creates "indicator chasing" -- switching strategies frequently based on noise.

The two-mode design addresses this:

1. **Full analysis mode:** Runs the complete pipeline (optimize parameters, rank indicators, select strategy). Run quarterly or when the user wants to re-evaluate.
2. **Daily signal mode:** Takes the locked-in strategy from the most recent full analysis and generates today's signal. Fast, deterministic, no optimization.

This mirrors professional practice. Fund managers re-optimize models quarterly or semi-annually, then run the locked model daily for signal generation. They do not re-derive their entire strategy every morning.

#### Quarterly Re-Optimization

The 126-day OOS window (approximately 6 months) suggests a natural re-optimization cycle of 6 months. However, quarterly re-optimization (approximately 63 trading days) is a reasonable compromise:

- **More frequent than OOS window:** Catches regime changes faster
- **Less frequent than monthly:** Avoids parameter instability
- **Industry standard:** Most institutional systematic strategies re-evaluate parameters quarterly
- **Calendar-aligned:** Easy to schedule (January, April, July, October)

The tension is between stability (less frequent = more stable) and adaptiveness (more frequent = more responsive). Quarterly balances these for equities, which can undergo regime changes in 3-6 months (e.g., a stock transitioning from growth to value status after an earnings disappointment).

#### Options Extension Readiness

Adding two stub models now (Instrument with asset type field, PositionLeg with leg type and options fields) costs almost nothing in complexity. These models have nullable options-specific fields that are simply unused for equity analysis. The data pipeline ignores them.

The payoff comes when the user eventually wants to add options analysis. Without these stubs, adding options support requires refactoring the core Instrument model, which touches every part of the pipeline. With the stubs, options support extends existing models rather than replacing them.

The key insight is that options analysis uses the same underlying equity indicators for directional signals. An options trader using this system would still use MACD + RSI + ADX to determine direction; the options-specific logic only affects position construction (which strike, which expiry, which strategy). The indicator pipeline is unchanged.

### 3.8 Data Format Reasoning

#### Nasdaq Format as Primary Parser

The user's data comes from Nasdaq.com. Building a generic parser that handles arbitrary CSV formats adds complexity with no immediate benefit. The Nasdaq format has known columns (Date, Close/Last, Volume, Open, High, Low) and known formatting (dates as MM/DD/YYYY, prices with dollar signs).

However, the parser includes a column mapping YAML configuration that maps logical column names (date, open, high, low, close, volume) to physical column names in the CSV. This provides an escape hatch: if the user later switches to Yahoo Finance, Alpha Vantage, or any other CSV provider, they change the mapping configuration rather than the code.

This is a classic YAGNI (You Aren't Gonna Need It) decision with a low-cost safety valve. Build for the known case, but make the unknown case configurable.

#### Split-Adjusted Data Acceptance

The sample NVDA data was verified against known stock split dates:

- **4:1 split on July 20, 2021:** Pre-split prices in the data are already divided by 4
- **10:1 split on June 10, 2024:** Pre-split prices are already divided by 10

This confirms the Nasdaq data is split-adjusted. It is NOT dividend-adjusted (adjusted close would differ from close for dividend-paying stocks). For NVDA specifically, the dividend impact is negligible: approximately $0.04/quarter on a $100+ stock, representing less than 0.04% per quarter. For higher-dividend equities, this could matter, but the v2 scope note acknowledges this as a known limitation.

Full CSV reload daily is the chosen approach. With approximately 2,500 rows (10 years of daily data), loading takes single-digit milliseconds. Incremental loading (appending new rows, detecting changes) adds state management complexity for zero performance benefit. The entire file fits in a single memory read operation.

---

## 4. What Changed from v1 and Why

### 4.1 Indicator Tier Rankings

| Change | v1 Tier | v2 Tier | Why |
|--------|:-:|:-:|---|
| Williams %R | 2 | **1** | 81% win rate on equities; long-only removes weak short signals, concentrating edge |
| OBV | 4 | **2** | Equities have highest-quality volume data; institutional flow detectable |
| CMF (Chaikin Money Flow) | 4 | **2** | Same reasoning as OBV; money flow analysis is most informative on equities |
| Donchian Channels | 2 | **3** | v1 noted poor stock performance; designed for commodities; inverted (mean-reversion) use still viable |
| Ichimoku Cloud | 3 | 3 | Unchanged, but noted as more complex than alternatives for same information |
| Parabolic SAR | 3 | 3 | Unchanged; better as trailing stop than entry signal |
| All Tier 1 indicators | 1 | 1 | EMA, RSI, MACD, ATR, ADX, Bollinger Bands remain the core; equities focus reinforces all of them |

### 4.2 Backtesting Parameters

| Parameter | v1 Value | v2 Value | Why |
|-----------|---------|---------|-----|
| IS window | 252 days (1 year) | **504 days (2 years)** | 30-day trade frequency produces only 8-12 trades in 1 year; need 16-24 for robust optimization |
| OOS window | 63 days (1 quarter) | **126 days (6 months)** | 30-day trades produce only 2-3 OOS trades in 1 quarter; need 4-6 minimum |
| Transaction costs | 0.1% round-trip | **0.02% round-trip (token)** | User confirmed negligible costs; token prevents degenerate solutions |
| WFA type | Not specified | **Rolling** | Individual equities change fundamentally over 10 years; anchored overweights stale data |
| Max drawdown | Scoring component | **Hard filter at 30% + circuit breaker** | User language ("maximum acceptable") implies constraint, not preference |
| Trade frequency | Not scored | **Soft scoring penalty** | User language ("default") implies preference, not constraint |
| Min trades/window | 30 | **20** | 504-day window with 30-day frequency yields ~24 trades; 30 minimum too restrictive |

### 4.3 Composite Scoring Weights

| Change | v1 | v2 | Why |
|--------|---|---|-----|
| Expectancy | 0.10 | 0.05 | Redundant with Sharpe and Profit Factor |
| Max Drawdown | 0.15 | 0.10 | Now a hard filter; scoring differentiates only among passing strategies |
| Trade Frequency | -- | 0.05 | New: penalizes deviation from 30-day target |
| Benchmark Outperformance | -- | 0.05 | New: rewards beating buy-and-hold and S&P 500 |

### 4.4 Technology Stack

| Change | v1 | v2 | Why |
|--------|---|---|-----|
| quantstats | Not included | **Added** | Professional tearsheets (200+ metrics, Sharpe, drawdown plots) with minimal code |
| exchange_calendars | Not included | **Added** | Correct NYSE holiday handling; avoids manual holiday maintenance |
| PDF generation | WeasyPrint (optional) | **Removed** | HTML confirmed as primary output; removes C library dependency |
| Asset class modes | 4 mode configs | **Removed** | Equities-only; no mode switching needed |

### 4.5 Architecture

| Change | v1 | v2 | Why |
|--------|---|---|-----|
| Pipeline stages | 6 (Load -> Validate -> Calculate -> Signal -> Backtest -> Report) | **6 (same stages, simplified internals)** | Same pipeline, but each stage is simpler without multi-asset branching |
| Run modes | 1 (full analysis) | **2 (full analysis + daily signal)** | Prevents "indicator chasing" from daily re-optimization |
| Options readiness | Not considered | **Instrument + PositionLeg stubs** | Minimal cost now, major refactoring savings later |
| Data parser | Generic CSV | **Nasdaq-specific with YAML column mapping** | Targeted for known format, configurable for future formats |
| Benchmark | Buy-and-hold of instrument | **Buy-and-hold + S&P 500** | Dual benchmark provides both absolute and relative performance context |

---

## 5. What Surprised Us

### Williams %R is BETTER for Long-Only Than Long-Short

Usually, constraining a strategy (removing the short side) reduces its opportunity set and therefore its expected performance. Williams %R is an exception. The published backtests show that short-side Williams %R trades on equities "didn't work well," meaning the short signals actively drag down the system's performance. Removing them does not just eliminate neutral trades -- it eliminates negative-expectancy trades. The long-only constraint acts as a filter that improves performance, not one that limits it.

This is genuinely unusual. Most indicators have roughly symmetric performance on long and short sides, or at least both sides are positive. Williams %R's asymmetry on equities is a specific, exploitable property.

### 30% Drawdown Limit Rejects Buy-and-Hold NVDA in Multiple Periods

NVDA experienced drawdowns exceeding 30% in at least three distinct periods: 2018 Q4 (56%), 2022 (66%), and several smaller instances. A simple buy-and-hold strategy applied to NVDA would be rejected by the 30% maximum drawdown constraint in multiple walk-forward windows.

This has a profound implication: the system MUST find strategies that actively avoid or reduce major drawdowns. Buy-and-hold is not just a benchmark to beat on returns -- it is a strategy that literally fails the risk constraint. Any strategy that passes the 30% filter has demonstrated the ability to sidestep at least some of the worst equity declines, which is a much stronger claim than simply outperforming on total return.

### Standard MACD/RSI Parameters Naturally Produce ~30-Day Trade Frequency

The user's trade frequency target (30 calendar days / 21 trading days) aligns surprisingly well with the standard parameters of the most common indicators:

- MACD(12,26,9) signal line crossovers on typical equities: every 20-40 trading days
- RSI(14) with 30/70 thresholds: every 15-35 trading days
- Bollinger Bands(20,2) band touches: every 15-25 trading days

This means the application does not need to aggressively re-tune parameters away from their standard values to hit the frequency target. Walk-forward optimization will fine-tune, but the starting point is already in the right neighborhood. This reduces overfitting risk because the parameter search space does not need to explore extreme values.

### Equities Spend 60-70% of Time Ranging

This is a well-known fact in quantitative finance, but its implications for indicator selection are under-appreciated. If the market is ranging 60-70% of the time, then mean-reversion indicators (RSI, Bollinger Bands, Williams %R) have more opportunity to generate signals than trend-following indicators (MACD, EMA crossovers).

ADX as a regime filter becomes even more valuable in this context than in v1's multi-asset framework. In commodities and forex, trends are more common and more persistent, so trend-following indicators have more runway. In equities, the ADX filter prevents trend-following strategies from churning during the dominant ranging regime, which is the majority of the time.

### Daily Data is So Small That Infrastructure Concerns are Irrelevant

Ten years of daily data for one equity is approximately 2,500 rows and under 500 KB in CSV. All of the v1 discussion about Parquet caching, database infrastructure, and incremental loading was solving a problem that does not exist at this scale. Loading 2,500 rows from CSV takes single-digit milliseconds. Even the parameter optimization (thousands of backtests) completes in seconds with vectorbt.

The only place where computational cost matters is Monte Carlo simulation (1,000+ iterations x walk-forward windows), which might take minutes. Everything else is effectively instant.

---

## 6. Confidence Levels

| Recommendation | Confidence | Notes |
|---------------|:-:|---|
| Williams %R upgrade to Tier 1 | High | 81% win rate on S&P 500, long-only explicitly optimal; published evidence directly supports use case |
| OBV/CMF upgrade to Tier 2 | High | Clear evidence once forex/commodity concern removed; equities have best volume data |
| Donchian downgrade to Tier 3 | Medium-High | v1 itself noted poor stock performance; designed for commodities; inverted use still viable adds uncertainty |
| 504-day IS / 126-day OOS | Medium | Trade-off between robustness (more trades per window) and data usage (fewer WFA steps); reasonable but not the only valid choice |
| Rolling (not anchored) WFA | High | Individual stocks change fundamentally over 10 years; anchored windows would overweight obsolete data |
| 30% drawdown as hard filter | High | "Maximum acceptable" is unambiguous constraint language; circuit breaker ensures metric accuracy |
| Trade frequency as soft constraint | Medium-High | "Default" implies preference, not hard constraint; small scoring weight is proportionate |
| 5 curated combinations | High | Algorithmic search with 5 slots would massively overfit; pre-registration is scientifically sound |
| MACD+RSI as primary combination | Medium-High | 73% win rate published, but results are market-dependent and period-specific |
| Token transaction cost (0.02%) | Medium | Principled (prevents degenerate solutions) but the specific value is somewhat arbitrary; 0.01% or 0.05% would serve the same purpose |
| Quarterly re-optimization | Medium | Balance between stability and freshness; no strong evidence that quarterly is better than semi-annual or monthly |
| Total return as primary ranking | Medium | User said "most profitable" which maps to total return, but composite scoring may better serve their actual goal |
| 100% in/out position sizing | Medium-High | Simplest for single-instrument analysis; user confirmed this approach; scales linearly |
| Nasdaq format as primary parser | High | Matches user's current data source; YAML column mapping provides escape hatch |
| Full CSV reload (not incremental) | High | 2,500 rows makes optimization pointless; complexity cost of incremental loading far exceeds any benefit |
| quantstats addition to stack | High | Professional tearsheets with minimal code; widely used in quantitative finance; reduces custom reporting effort |
| exchange_calendars addition | High | Correct NYSE holiday handling; eliminates a class of subtle date-alignment bugs |

---

## 7. Connections Between Research Streams

The four workstreams reinforce each other more tightly in v2 than in v1, because the narrower scope creates stronger logical dependencies.

### Indicator Re-Ranking Depends on Backtesting Parameters

The decision to upgrade Williams %R to Tier 1 depends on the long-only constraint. The decision to upgrade OBV/CMF to Tier 2 depends on the equities-only scope. Both of these are backtesting/scope parameters that flow into indicator evaluation. In v1, the indicator rankings were more independent because they were evaluated across all asset classes.

### Backtesting Windows Depend on Trade Frequency, Which Depends on Indicator Selection

The 504/126-day window sizing is driven by the ~30-day trade frequency. But the ~30-day frequency is partly a property of the selected indicators with standard parameters. If the indicator set included very fast indicators (e.g., 2-period RSI, which can trigger every few days), the windows could be shorter. The indicator selection and backtesting parameters co-determine each other, which is why both workstreams needed to iterate.

### Drawdown Constraint Connects Backtesting to Architecture

The 30% hard filter is a backtesting parameter, but it has architectural implications. The backtest engine must implement a circuit breaker (not just report max drawdown after the fact). The daily signal mode must track current drawdown from the last equity high and potentially generate an emergency exit signal. This is a case where a risk management decision directly shapes the codebase.

### Volume Indicator Upgrade Connects Indicator Research to Data Architecture

Upgrading OBV and CMF to Tier 2 only makes sense because the data architecture confirms high-quality volume data from Nasdaq. If the data source provided unreliable volume (as tick-based forex data does), the upgrade would not be justified. The data workstream's confirmation of volume data quality is a prerequisite for the indicator workstream's re-ranking.

### Options Readiness Connects Architecture to Future Indicator Use

The Instrument and PositionLeg model stubs in the architecture are motivated by future options analysis. But options analysis depends on the same directional indicators (MACD, RSI, ADX) for underlying direction. The indicator pipeline's design must be clean enough that options logic can consume its output without modification. This means the signal generation stage must output directional conviction (bullish/bearish/neutral with strength) rather than specific position instructions (buy 100 shares).

---

## 8. Open Tensions and Trade-offs

### Tension 1: Robustness vs. Adaptiveness

**504-day IS windows** produce more robust parameter estimates but are slower to adapt to regime changes. **252-day IS windows** adapt faster but risk overfitting to recent noise. There is no objectively correct answer; it depends on whether the equity's dynamics are more stable (favoring 504) or more variable (favoring 252).

The v2 recommendation of 504 days leans toward robustness, which is the conservative choice. If the user finds that strategies degrade significantly between quarterly re-optimizations, shortening to 378 days (1.5 years) would be a reasonable adjustment.

### Tension 2: Hard Drawdown Filter vs. Missing Good Strategies

A strategy that hits 31% drawdown once in 10 years but otherwise has excellent performance would be rejected by the hard 30% filter. This is by design (the user set the constraint), but it may eliminate strategies that are "close enough." The alternative -- a soft penalty -- would keep these strategies in the running but violate the user's stated maximum.

This tension cannot be resolved technically; it is a user preference. The implementation follows the user's language ("maximum acceptable" = hard filter) but the 30% threshold is configurable.

### Tension 3: Token Transaction Cost -- Principled but Arbitrary

The 0.02% token cost is designed to prevent degenerate solutions (excessive trading for minimal edge). But the specific value is somewhat arbitrary. Setting it at 0.01% or 0.05% would serve the same purpose with slightly different tiebreaking behavior. There is no empirical basis for choosing 0.02% specifically.

The recommendation is to run sensitivity analysis: if the final rankings change when varying costs from 0% to 0.05%, the affected strategies are fragile regardless of the specific cost assumption.

### Tension 4: Composite Score vs. "Most Profitable"

The user asked for the "most profitable" indicators. Taken literally, this means ranking by total return. But a strategy with 200% total return and 80% max drawdown is "most profitable" while being arguably unusable. The composite score captures what the user likely means (best risk-adjusted profitability), but it introduces subjective weight choices.

v2 addresses this by reporting both: total return ranking AND composite score ranking. If they agree, there is no tension. If they disagree, the report explains why and lets the user decide.

### Tension 5: Pre-Defined Combinations vs. Data-Driven Discovery

The 5 curated combinations are chosen based on published research, not the user's specific data. It is possible that a different combination (not in the curated list) would work better for the specific equity. By pre-defining combinations, we avoid overfitting but may miss the optimal solution.

This is the classic bias-variance trade-off applied to strategy selection. Pre-defined combinations have lower variance (more likely to work out-of-sample) but higher bias (may not be optimal). Algorithmically-searched combinations have lower bias but higher variance (may not generalize). With only 5 slots and limited data, the high-bias/low-variance choice is appropriate.

### Tension 6: Daily Signal Mode Assumes Stationarity

The daily signal mode uses locked-in parameters from the most recent full analysis. This assumes that the indicator's performance characteristics remain stable between re-optimizations (quarterly). For equities experiencing fundamental shifts (e.g., a major acquisition, a product line pivot, a sector rotation), this assumption may break down between quarterly reviews.

The mitigation is the circuit breaker: if drawdown approaches 30%, the system exits regardless of the locked-in strategy's signal. This provides a safety valve against stationarity violations.

### Tension 7: Walk-Forward Windows vs. Market Regime Alignment

The rolling WFA windows are calendar-based (504/126 days), not regime-based. A window boundary might split a major trend in half, causing the IS period to optimize for trending conditions while the OOS period is ranging (or vice versa). Regime-aligned windows would be theoretically superior but are impractical because regime boundaries are only identifiable in hindsight.

The mitigation is running enough WFA steps that regime misalignment averages out across windows. With 10 years of data and 504/126 windows, approximately 12-14 OOS periods will include a mix of regimes.

---

## Appendix: Methodology Notes

### How Confidence Ratings Were Assigned

- **High:** Multiple independent sources agree, the logic is straightforward, and the decision is robust to reasonable alternative assumptions.
- **Medium-High:** Strong evidence supports the decision, but reasonable people could disagree on the specific implementation. Alternative approaches would be defensible.
- **Medium:** The decision is reasonable and well-motivated, but the specific parameter values are somewhat arbitrary. Sensitivity analysis is recommended.
- **Low-Medium:** Theoretically appealing but lacks strong empirical support, or introduces complexity that may not pay off.

### Relationship to v1 Thinking Document

This document supersedes `notes/Research_Thinking_v1.md` for all topics that changed. v1 thinking remains the authoritative reference for decisions that did not change (e.g., TA-Lib over pandas-ta, vectorbt over Backtrader, CSV as primary input, YAML for configuration, pipeline architecture, regime detection via ADX). Where v2 modifies a v1 decision, this document explains the specific change and reasoning.
