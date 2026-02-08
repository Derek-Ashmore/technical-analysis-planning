# Technical Analysis Application Research v2

## Scope: US Equities, Long-Only Trading

---

## 1. Executive Summary

### What Changed from v1

Research v1 addressed a broad multi-asset application (commodities, forex, crypto, equities). Based on user requirements clarification, v2 narrows scope to **US equities only** with **long-only trading**. This focus enables significant improvements in indicator selection, backtesting rigor, and practical applicability.

**Key changes:**

- **Scope narrowed:** US equities only (individual stocks). Multi-asset support removed from core design, retained as future extension.
- **Direction constrained:** Long-only. Short-selling logic removed. Exit signals = sell to cash (flat), not reverse to short.
- **Volume indicators promoted:** OBV and CMF upgraded from Tier 4 to Tier 2. Volume data is fully available and highly informative for equities.
- **Williams %R upgraded:** Promoted from Tier 2 to Tier 1. Long-only is its optimal use case (81% win rate on S&P 500).
- **Donchian Channels downgraded:** Moved from Tier 2 to Tier 3. Designed for commodities trend-following; less effective for equities.
- **Tier 1 expanded:** 7 indicators (up from 6). RSI, Williams %R, EMA, MACD, ATR, ADX, Bollinger Bands.
- **Walk-forward analysis revised:** 504-day in-sample / 126-day out-of-sample (up from 252/63) for more robust optimization on equities.
- **Dual benchmark:** Buy-and-hold of the asset AND S&P 500 comparison.
- **Drawdown constraint:** 30% maximum drawdown as a hard rejection filter.
- **Trade frequency target:** ~30-day average holding period as a soft scoring constraint.
- **Daily run mode added:** Evening signal generation for next-day execution alongside full weekly analysis.
- **Verbose mode added:** Detailed indicator evaluation with rejection reasoning.
- **Data format specified:** Nasdaq-style CSV with known quirks (dollar signs, Close/Last column, MM/DD/YYYY, reverse chronological).
- **Earnings gap detection added:** Flags gaps exceeding 3x rolling ATR.
- **NYSE calendar integration:** Via `exchange_calendars` library for proper trading day handling.
- **Options readiness:** Architecture designed with extension points for future options integration.

**Key findings (unchanged from v1):**

- Market regime detection (ADX) remains the single most important factor.
- MACD + RSI + ADX filter remains the strongest combination (73% win rate).
- TA-Lib + vectorbt remains the recommended technology stack.
- Pipeline architecture remains the recommended approach.

**Key findings (new in v2):**

- Equities exhibit stronger mean-reversion at short time horizons than other asset classes, favoring RSI and Bollinger Bands strategies.
- Volume confirmation via OBV/CMF adds significant value for equity signals.
- The self-fulfilling prophecy effect is strongest for equities (50/200 EMA, Golden Cross) because the most market participants watch these levels.
- 30% maximum drawdown is aggressive for individual stocks (NVDA beta 1.5-2.0). Circuit breaker logic is essential.
- Academic evidence for TA profitability in developed equities has weakened since 1990, but the Adaptive Market Hypothesis suggests strategies have life cycles and context-dependent value.

---

## 2. Revised Indicator Tier Rankings

### 2.1 Tier 1 -- Must Implement (7 indicators)

| # | Indicator | Change from v1 | Key Evidence | Primary Use |
|---|-----------|:-:|---|---|
| 1 | **RSI** | Strengthened | Up to 91% win rate on equities; short-period RSI(2-6) excels at mean-reversion; equities are inherently mean-reversive at short horizons | Standalone mean-reversion + confirmation |
| 2 | **Williams %R** | UPGRADED from Tier 2 | 81% win rate on S&P 500; long-only is the optimal use case (buy on leaving oversold); mathematically inverse of Fast Stochastic but superior in equity backtests | Standalone mean-reversion + timing |
| 3 | **EMA** | Unchanged | Superior MA for daily equities; 50/200 EMA widely watched; 394% profit in comparative study; 1.92 profit factor | Standalone trend signal |
| 4 | **MACD** | Unchanged | 73-86% win rate on equity indices; 70-80% success in trending markets; combines trend and momentum in one indicator | Standalone + confirmation |
| 5 | **ATR** | Unchanged | Essential for position sizing and stop-loss; foundation of risk management; critical for enforcing 30% drawdown constraint | Risk management (not signal) |
| 6 | **ADX** | Unchanged | Equities spend 60-70% of time ranging; ADX > 25 as trend filter eliminates whipsaw losses; only indicator that quantifies trend strength | Trend strength filter |
| 7 | **Bollinger Bands** | Strengthened | Mean-reversion aligns naturally with equities; +0.8% avg gain within 6 days below lower band; dual-purpose (mean reversion + breakout) | Mean reversion + breakout |

**Justification for tier changes:**

- **Williams %R upgrade:** In v1, Williams %R was Tier 2 because the application was multi-asset. For equities specifically, the 81% win rate on S&P 500 is directly relevant. Long-only trading is the natural use case for oversold bounces -- you only need the buy signal (crossing above -80), and the sell signal (-20 crossover) serves as exit. The short-selling weakness (crossing below -20 to initiate shorts) is irrelevant in long-only mode. Williams %R also makes Stochastic redundant (mathematically identical signal, different scale), consolidating two indicators into one.
- **RSI strengthened:** Equities exhibit stronger mean-reversion at short horizons than commodities, forex, or crypto. The 91% win rate was specifically measured on equity setups. Short-period RSI(2-6) is particularly powerful for individual stocks.
- **Bollinger Bands strengthened:** The inherent mean-reversive nature of equities (especially large-cap) aligns directly with Bollinger Band mean-reversion strategies. The +0.8% average gain below the lower band is most reliable for equities.

### 2.2 Tier 2 -- High Priority (5 indicators)

| # | Indicator | Change from v1 | Key Evidence | Primary Use |
|---|-----------|:-:|---|---|
| 8 | **OBV** | UPGRADED from Tier 4 | Volume data fully available for equities; leading indicator for institutional accumulation/distribution; no parameters to optimize; divergences precede price moves | Volume confirmation (primary) |
| 9 | **CMF** | UPGRADED from Tier 4 | Bounded volume oscillator (-1 to +1); effective confirmation filter; uses close position within daily range; works as zero-line crossover filter | Volume confirmation (secondary) |
| 10 | **MA Crossover** | Unchanged | Golden Cross/Death Cross: strongest self-fulfilling prophecy effect in equities; 50/200 SMA is the most widely watched level by institutional investors; 2.12% avg gain at 3 months | Trend change detection |
| 11 | **Supertrend** | Unchanged | 65.79% win rate on S&P 500 long-only; combines trend direction and trailing stop; ATR-based adaptation to volatility | Trend + trailing stop |
| 12 | **Elder's Triple Screen** | System-level | Framework integrating Tier 1 indicators across timeframes; 31% annual return reported; inherently filters for high-probability trades | Complete multi-timeframe system |

**Justification for volume indicator upgrades:**

In v1, OBV and CMF were Tier 4 because the multi-asset scope included forex (no volume) and commodities (limited volume). For US equities:
- Exchange-reported volume is the highest quality volume data of any asset class.
- Institutional order flow creates detectable accumulation/distribution patterns via OBV.
- CMF provides a bounded, systematic way to confirm signals from other indicators.
- Volume confirmation reduces false signals for trend-following and breakout strategies.

### 2.3 Tier 3 -- Valuable Additions (9 indicators)

| # | Indicator | Change from v1 | Notes |
|---|-----------|:-:|---|
| 13 | **Keltner Channels** | Unchanged | ATR-based channel; Keltner-Bollinger Squeeze setup valuable; 77-80% win rate on S&P 500 but declining post-2016 |
| 14 | **CCI** | Unchanged | Short lookback (5-9 days) effective for daily equity mean-reversion; 8% annual return vs 6% buy-and-hold |
| 15 | **Stochastic Oscillator** | Unchanged | Redundant with Williams %R (Tier 1); retained for completeness; use only if Williams %R is not available |
| 16 | **Parabolic SAR** | Unchanged | Trailing stop mechanism only; 19% win rate as standalone across 2,880 years of DJIA backtesting; never use as entry signal |
| 17 | **Donchian Channels** | DOWNGRADED from Tier 2 | Designed for commodities trend-following (Turtle Trading); works poorly for equities; low win rate (35-45%) depends on large winners from sustained trends, which equities provide less reliably than commodities |
| 18 | **A/D Line** | UPGRADED from Tier 4 | Supplementary volume indicator; more granular than OBV (uses close position within range); institutional accumulation/distribution detection |
| 19 | **VPT** | UPGRADED from Tier 4 | Weights volume by percentage price change; more nuanced than OBV; supplementary volume confirmation |
| 20 | **SMA** | Slightly upgraded | Long-term trend reference (50/200 day); the 200-day SMA is the most widely watched institutional benchmark; Golden Cross uses SMA specifically |
| 21 | **Ichimoku Cloud** | Slightly downgraded | Complex; settings designed for Japanese equities in the 1960s; on S&P 500 only 5.2% CAGR vs 6.9% buy-and-hold; better for crypto and forex |

### 2.4 Tier 4 -- Supplementary (4 indicators)

| # | Indicator | Notes |
|---|-----------|---|
| 22 | **ROC** | Momentum confirmation; 66% win rate on DJIA but inferior to RSI, Stochastic, Williams %R |
| 23 | **Standard Deviation** | Building block for other indicators; not a standalone signal |
| 24 | **MA Ribbon** | Visual trend strength tool; lagging; computationally heavier; bear markets are fast and defeat ribbon twist signals |
| 25 | **Momentum (raw)** | Simplest momentum; unbounded and not normalized; inferior to ROC and RSI for systematic trading |

### 2.5 Change Summary: Tier Rankings v1 to v2

| Indicator | v1 Tier | v2 Tier | Direction | Reason |
|-----------|:---:|:---:|:-:|---|
| Williams %R | 2 | 1 | UP | 81% WR on S&P 500; long-only is optimal use case |
| OBV | 4 | 2 | UP | Volume data fully available for equities |
| CMF | 4 | 2 | UP | Bounded volume oscillator; effective equity filter |
| A/D Line | 4 | 3 | UP | Supplementary volume indicator for equities |
| VPT | 4 | 3 | UP | Supplementary volume confirmation |
| SMA | 4 | 3 | UP | 200-day widely watched; Golden Cross uses SMA |
| Donchian Channels | 2 | 3 | DOWN | Designed for commodities; poor for equities |
| Ichimoku Cloud | 3 | 3 (lower) | DOWN | Underperforms buy-and-hold on S&P 500 |
| All others | -- | -- | -- | Unchanged |

---

## 3. Volume Indicators for Equities

### 3.1 Why Volume Matters for Equities

In v1, volume indicators were Tier 4 because the multi-asset scope included forex (no volume) and commodities (limited volume). For US equities, volume data is the highest quality of any asset class:

- **Exchange-reported:** NYSE/NASDAQ report official trade volume; no proxy or estimate needed.
- **Institutional signal:** Large institutional orders create detectable patterns in volume before price reflects the intent.
- **Confirmation value:** A price breakout on high volume is far more reliable than one on low volume.
- **Leading indicator:** Volume changes often precede price changes by 1-3 days.

### 3.2 Recommended Volume Framework

**Primary:** OBV (On-Balance Volume)

| Attribute | Detail |
|-----------|--------|
| Calculation | Cumulative: add volume on up days, subtract on down days |
| Parameters | None (parameter-free) |
| Signal type | Divergence (OBV direction vs price direction) and trend confirmation |
| Strengths | Simplest; leading indicator; no parameters to optimize; detects institutional accumulation/distribution |
| Weaknesses | Binary classification (entire day's volume) is crude; one large-volume day skews disproportionately; absolute value meaningless |
| Best use | Confirm breakouts; detect divergences; filter false signals from other indicators |
| Signal line | 20-period SMA of OBV for crossover signals |

**Secondary:** CMF (Chaikin Money Flow)

| Attribute | Detail |
|-----------|--------|
| Calculation | Sum(CLV x Volume, 20) / Sum(Volume, 20), where CLV = [(Close - Low) - (High - Close)] / (High - Low) |
| Parameters | Period (default 20) |
| Signal type | Zero-line crossover; extreme readings (+/-0.25) |
| Strengths | Bounded (-1 to +1); uses close position within daily range; effective as confirmation filter; fixed lookback (more responsive than cumulative OBV) |
| Weaknesses | Less reliable during gaps; zero-line whipsaws in ranging markets |
| Best use | Confirm trend direction; filter entry signals; CMF > 0 confirms bullish signals from other indicators |

**Supplementary:** A/D Line (Accumulation/Distribution)

| Attribute | Detail |
|-----------|--------|
| Calculation | Cumulative: A/D = A/D(prev) + CLV x Volume |
| Parameters | None |
| Signal type | Divergence; trend confirmation |
| Strengths | More granular than OBV (considers where close falls within range); detects institutional activity |
| Weaknesses | Ignores gap-to-gap moves; cumulative absolute value meaningless |
| Best use | Supplementary divergence detection; confirm OBV readings |

**Supplementary:** VPT (Volume-Price Trend)

| Attribute | Detail |
|-----------|--------|
| Calculation | Cumulative: VPT = VPT(prev) + Volume x [(Close - Close(prev)) / Close(prev)] |
| Parameters | None for indicator; 14-21 period EMA for signal line |
| Signal type | Signal line crossover; divergence |
| Strengths | Weights volume by percentage price change (more precise than OBV); proportional money flow |
| Weaknesses | Subtle movements on small percentage changes; cumulative nature limits short-term use |
| Best use | Supplementary confirmation; signal line crossovers for systematic entry/exit |

### 3.3 Volume Confirmation Framework

For any primary signal from a Tier 1 indicator, volume confirmation adds reliability:

```
Primary Signal (e.g., MACD bullish cross)
    + OBV rising (OBV > OBV SMA(20))
    + CMF > 0
    = Confirmed signal (higher probability)

Primary Signal
    + OBV falling (OBV < OBV SMA(20))
    + CMF < 0
    = Unconfirmed signal (lower probability, smaller position or skip)
```

Volume confirmation is a **filter**, not a signal generator. It does not produce entry/exit signals on its own but upgrades or downgrades the confidence of signals from other indicators.

---

## 4. Long-Only Trading Implications

### 4.1 Asymmetric Signal Design

Long-only trading fundamentally changes signal design:

| Signal Type | Multi-directional (v1) | Long-only (v2) |
|-------------|----------------------|----------------|
| Buy | Enter long position | Enter long position |
| Sell (to flat) | Exit long, enter short | Exit long position (go to cash) |
| Short sell | Enter short position | N/A -- not available |
| Cover (buy to close) | Exit short position | N/A -- not available |

**Implications:**

- **Entry signals matter more:** In long-only, you can only profit when price rises. Entry timing is critical.
- **Exit signals are asymmetric:** Sell signals do not need to predict downside -- they only need to detect when upside momentum is exhausted. A "neutral" signal (ADX < 20, no clear direction) is sufficient reason to exit.
- **Cash position is the default:** When no signal is active, the position is cash (earning nothing). This creates opportunity cost that must be measured.
- **Bearish indicators become exit-only:** Death Cross, RSI overbought, MACD bearish cross all serve as exit signals, not entry signals.

### 4.2 Indicator Behavior in Long-Only Context

| Indicator | Entry Signal (buy) | Exit Signal (sell to flat) |
|-----------|-------------------|--------------------------|
| RSI | RSI crosses above 30 from below (leaving oversold) | RSI crosses below 70 from above (leaving overbought) |
| Williams %R | %R crosses above -80 from below | %R crosses below -20 from above |
| EMA | Price crosses above EMA; short EMA crosses above long EMA | Price crosses below EMA; short EMA crosses below long EMA |
| MACD | MACD crosses above signal line; histogram turns positive | MACD crosses below signal line; histogram turns negative |
| Bollinger Bands | Price touches/crosses below lower band | Price touches/crosses above upper band or returns to middle band |
| ADX | ADX rises above 25 (enables trend-following entries) | ADX falls below 20 (trend exhaustion) |
| Supertrend | Supertrend flips bullish (moves below price) | Supertrend flips bearish (moves above price) |

### 4.3 Effect on Performance Metrics

Long-only trading affects several metrics:

- **Time in market:** Long-only systems are typically in the market 40-60% of the time (vs 100% for always-in systems). Cash periods must be tracked.
- **Benchmark comparison:** Buy-and-hold is always 100% invested. Long-only systems must overcome the opportunity cost of cash periods to outperform.
- **Sharpe ratio impact:** Cash periods reduce both return and volatility. The net effect on Sharpe depends on whether the system successfully avoids drawdowns during cash periods.
- **Max drawdown:** Long-only systems should have lower max drawdown than buy-and-hold because cash periods protect capital during downturns. This is a primary advantage.
- **Win rate vs payoff ratio:** Long-only mean-reversion strategies typically have high win rates (70-90%) with modest per-trade gains. Long-only trend-following has lower win rates (40-60%) with larger per-trade gains.

### 4.4 Cash Period Tracking

The application must track:
- Total time in position vs total time in cash (as percentage)
- Return during in-position periods vs buy-and-hold return during same periods
- Return foregone during cash periods (what buy-and-hold earned while system was flat)
- Drawdown avoided during cash periods (what buy-and-hold lost while system was flat)

---

## 5. Trade Frequency Alignment

### 5.1 Target: ~30-Day Average Holding Period

The target holding period of approximately 30 trading days (~6 calendar weeks) aligns with swing trading. This affects parameter selection for all indicators.

### 5.2 Parameter Tuning for 30-Day Trades

| Indicator | Default Parameters | 30-Day Aligned Parameters | Rationale |
|-----------|-------------------|--------------------------|-----------|
| RSI | 14-period, 30/70 | 14-period, 30/70 (unchanged) | RSI(14) naturally produces signals at swing-trade frequency |
| RSI (short-period) | 2-6 period, 10/90 | 5-period, 20/80 | Short-period RSI produces too-frequent signals; lengthen slightly |
| Williams %R | 14-period | 14-period (unchanged) | Natural frequency aligns with target |
| EMA Crossover | 12/26 or 50/200 | 20/50 | 50/200 is too slow (~100+ day trades); 12/26 is too fast (~10-15 day); 20/50 targets 25-40 days |
| MACD | (12, 26, 9) | (12, 26, 9) (unchanged) | Standard MACD produces signals at approximately monthly frequency |
| Bollinger Bands | (20, 2) | (20, 2) (unchanged) | 20-day period aligns well with monthly cycle |
| Supertrend | ATR(10), mult 3 | ATR(14), mult 3 | Slightly longer ATR smooths out noise for swing-trade holding periods |
| ADX | 14-period | 14-period (unchanged) | Standard; no tuning needed for trade frequency |

### 5.3 Frequency as Soft Constraint

Trade frequency is a **soft constraint** -- it affects scoring but does not reject indicators outright.

- **Target:** 15-25 trades per year (approximately one per month, each held ~30 days)
- **Acceptable range:** 8-40 trades per year
- **Scoring penalty:** Indicators producing fewer than 8 or more than 40 trades per year receive a downward adjustment in the composite score
- **Hard minimum:** At least 30 total trades across the full backtest period (for statistical significance)
- **Metric:** Average time between trades and average holding period are reported but not used as rejection criteria

---

## 6. Drawdown Analysis

### 6.1 The 30% Maximum Drawdown Constraint

Maximum drawdown of 30% serves as a **hard rejection filter**. Any indicator or combination that produces a drawdown exceeding 30% in either in-sample or out-of-sample testing is rejected.

### 6.2 Context for Individual Stocks

The 30% max drawdown constraint is aggressive for individual US equities:

| Context | Typical Max Drawdown |
|---------|:---:|
| S&P 500 index (2000-2024) | 55% (2008-2009), 34% (2020 COVID) |
| Large-cap individual stocks (typical) | 40-60% during bear markets |
| NVDA specifically (2022) | ~66% peak-to-trough |
| NVDA beta | 1.5-2.0 (amplifies market moves) |
| A well-managed active strategy | 15-25% target |
| Buy-and-hold on most individual stocks | Will exceed 30% at some point |

**Implication:** The 30% constraint will reject many strategies that are otherwise profitable. This is by design -- the constraint prioritizes capital preservation. However, the user should understand that:

1. Few individual-stock strategies will stay below 30% during severe bear markets.
2. The constraint effectively forces the system to exit positions early in drawdowns, which can reduce total return.
3. High-beta stocks like NVDA will be particularly challenging to keep within 30%.

### 6.3 Historical Drawdown Profiles by Strategy Type

| Strategy Type | Typical Max DD | Recovery Time | Frequency |
|---------------|:-:|:-:|:-:|
| Buy-and-hold (S&P 500) | 30-55% | 1-5 years | Every 7-12 years |
| Buy-and-hold (individual stock) | 40-70% | 1-7 years | Variable |
| Trend-following (long-only) | 15-30% | 3-12 months | 1-3 per decade |
| Mean-reversion (long-only) | 10-25% | 1-6 months | 2-5 per decade |
| ADX-filtered (exit on no-trend) | 10-20% | 1-4 months | More frequent small DDs |

### 6.4 Drawdown Mitigation Strategies

1. **ATR-based position sizing:** Size positions inversely proportional to ATR. Higher volatility = smaller position = limited loss potential per trade.

2. **Circuit breaker in backtest:** When cumulative drawdown from equity peak reaches a threshold (e.g., 20%), halt all new entries and close existing positions. Resume only when conditions improve. The circuit breaker prevents a 20% drawdown from becoming 30%.

   ```
   Circuit Breaker Logic:
   - Level 1 (15% DD): Reduce position size by 50%
   - Level 2 (20% DD): No new entries; maintain existing positions with tight stops
   - Level 3 (25% DD): Close all positions; go to cash
   - Resume: After 10 consecutive trading days with equity above Level 2 threshold
   ```

3. **Regime filter (ADX < 20 = exit):** The simplest drawdown mitigation. When ADX indicates no trend, exit to cash. Most large drawdowns occur when the system holds positions through trendless, declining markets.

4. **Trailing stop via ATR:** 2x ATR trailing stop from highest close since entry. This mechanically limits per-trade drawdown while allowing profits to run.

5. **Maximum holding period:** Force exit after N days (e.g., 60 trading days) regardless of signal. This prevents positions from drifting into extended drawdowns.

### 6.5 Drawdown Reporting

For each indicator and combination, report:
- Maximum drawdown (percentage and absolute dollar amount)
- Drawdown start date, trough date, recovery date
- Drawdown duration (peak to trough) and recovery duration (trough to new high)
- Number of drawdowns exceeding 10%, 15%, 20%
- Average drawdown depth and duration
- Drawdown during each OOS window

---

## 7. Recommended Indicator Combinations

### 7.1 Five Curated Combinations

Each combination specifies entry logic, exit logic, filters, expected trade frequency, and expected performance characteristics.

#### Combination 1: MACD + RSI Confirmation

| Attribute | Detail |
|-----------|--------|
| Category | Trend-following with momentum confirmation |
| Entry | MACD line crosses above signal line AND RSI > 40 (not oversold, confirming momentum) |
| Exit | MACD line crosses below signal line OR RSI < 30 (whichever comes first) |
| ADX filter | ADX > 20 (minimum trend presence) |
| Volume filter | OBV > OBV SMA(20) preferred but not required |
| Stop-loss | 2x ATR(14) trailing stop from highest close |
| Expected frequency | 15-25 day average holding period; 12-20 trades per year |
| Expected win rate | 60-73% |
| Target market regime | Trending (ADX > 25) and transitional (ADX 20-25) |
| Academic basis | MACD + RSI reduces false signals by 65% vs MACD alone; 73% win rate over 235 trades |

#### Combination 2: ADX-Filtered EMA Crossover

| Attribute | Detail |
|-----------|--------|
| Category | Trend-following with regime filter |
| Entry | 20-period EMA crosses above 50-period EMA AND ADX > 25 |
| Exit | 20 EMA crosses below 50 EMA OR ADX falls below 20 |
| Volume filter | CMF > 0 at time of entry |
| Stop-loss | 2.5x ATR(14) from entry price (wider due to slower signals) |
| Expected frequency | 25-40 day average holding period; 8-15 trades per year |
| Expected win rate | 55-65% |
| Target market regime | Strong trending (ADX > 25) only |
| Notes | Fewer trades but higher quality; ADX filter eliminates ranging-market whipsaws; may underperform in choppy markets due to infrequent entries |

#### Combination 3: RSI + Bollinger Mean-Reversion

| Attribute | Detail |
|-----------|--------|
| Category | Mean-reversion |
| Entry | RSI(14) < 30 AND price below lower Bollinger Band(20, 2) AND ADX < 30 (not in strong trend) |
| Exit | RSI > 70 OR price touches upper Bollinger Band OR price returns to middle Bollinger Band (20 SMA) -- whichever comes first |
| Volume filter | OBV divergence (OBV rising while price falling) strengthens the signal |
| Stop-loss | 2x ATR(14) from entry; also exit if ADX rises above 35 (trend emerging against position) |
| Expected frequency | 10-20 day average holding period; 15-25 trades per year |
| Expected win rate | 70-85% |
| Target market regime | Ranging (ADX < 25-30) |
| Notes | High win rate, small per-trade gains; depends on equities' mean-reversive nature; will underperform in prolonged bear markets; the ADX < 30 filter prevents buying dips in strong downtrends |

#### Combination 4: Supertrend + MACD

| Attribute | Detail |
|-----------|--------|
| Category | Trend-following with momentum |
| Entry | Supertrend flips bullish (moves below price) AND MACD histogram > 0 |
| Exit | Supertrend flips bearish (moves above price) |
| ADX filter | No additional ADX filter (Supertrend provides built-in trend detection via ATR) |
| Volume filter | OBV confirming uptrend preferred |
| Stop-loss | Supertrend line itself acts as trailing stop |
| Expected frequency | 20-35 day average holding period; 10-18 trades per year |
| Expected win rate | 55-67% |
| Target market regime | Trending markets; rides trends via Supertrend trailing stop |
| Notes | Supertrend at ATR(14), multiplier 3; MACD confirmation reduces false Supertrend flips; clean exit logic (Supertrend flip only) |

#### Combination 5: Williams %R + EMA Trend

| Attribute | Detail |
|-----------|--------|
| Category | Mean-reversion within trend |
| Entry | Williams %R(14) crosses above -80 from below (leaving oversold) AND price > 50 EMA (confirming uptrend) |
| Exit | Williams %R crosses below -20 from above (leaving overbought) OR price closes below 50 EMA |
| ADX filter | Optional: ADX > 20 for additional trend confirmation |
| Volume filter | CMF > 0 at entry |
| Stop-loss | 2x ATR(14) trailing stop |
| Expected frequency | 15-25 day average holding period; 12-20 trades per year |
| Expected win rate | 65-80% |
| Target market regime | Trending markets with pullback entries |
| Notes | Buys dips within uptrends; 50 EMA acts as trend guard preventing buys in downtrends; Williams %R's fast movement provides timely oversold detection |

### 7.2 Combination Selection Logic

The application should test all 5 combinations in backtest and rank by composite score. The user may also define custom combinations via YAML configuration.

The 5 combinations cover:
- 2 pure trend-following (Combinations 2 and 4)
- 1 pure mean-reversion (Combination 3)
- 2 hybrid trend+momentum (Combinations 1 and 5)

This ensures at least one combination should perform well regardless of the prevailing market regime.

---

## 8. Backtesting Framework

### 8.1 Walk-Forward Analysis Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| In-sample (IS) window | 504 trading days (~2 years) | Captures at least one full market cycle for equities; sufficient data for parameter optimization |
| Out-of-sample (OOS) window | 126 trading days (~6 months) | Long enough to assess performance across multiple trades; short enough to adapt to changing conditions |
| Step size | 126 trading days | Roll forward by one OOS window each step; maximizes the number of OOS windows |
| Warmup period | 200 trading days | Ensures 200-day SMA/EMA have valid values before IS window begins |
| Method | Rolling windows (preferred) | For individual equities, rolling is preferred over anchored because equity characteristics change over time; anchored window is an alternative for indices |
| Minimum data required | 200 (warmup) + 504 (first IS) + 126 (first OOS) = 830 trading days (~3.3 years) |

**Walk-forward sequence example (10 years of data):**

```
|--warmup--|----IS (504)-----|--OOS (126)--|
                |--warmup--|----IS (504)-----|--OOS (126)--|
                                |--warmup--|----IS (504)-----|--OOS (126)--|
                                ...
```

Each OOS window produces an independent, unbiased performance estimate. The final performance assessment uses only OOS results.

### 8.2 Execution Model

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Execution timing | Next-bar-open | Signal on bar T executes at bar T+1 open. Structurally prevents look-ahead bias. |
| Transaction cost | 0.02% per trade (0.04% round trip) | Reflects modern discount brokerage (near-zero commission) plus market impact for typical retail size. Lower than v1's 0.1% because US equities have the lowest transaction costs of any asset class. |
| Slippage | Included in transaction cost | For daily bar execution on liquid US equities, slippage is negligible at retail size. Absorbed into the 0.02% per-side cost. |
| Position sizing | 100% of capital per trade (full allocation to single stock) | Since the application analyzes one stock at a time, position sizing within the stock is binary -- either in or out. ATR-based sizing applies only if extended to portfolio mode. |
| Direction | Long-only | Enter by buying; exit by selling to cash. No short selling. |

### 8.3 Drawdown Circuit Breaker in Backtest

The circuit breaker is implemented within the backtest engine:

```
DURING BACKTEST:
  if current_drawdown >= 25%:
      force_exit_all_positions()
      enter_cooldown_mode()

  if in_cooldown_mode:
      if equity > 0.85 * peak_equity for 10 consecutive days:
          exit_cooldown_mode()
      else:
          skip_all_entry_signals()
```

The circuit breaker ensures no strategy can produce a drawdown exceeding ~28-30% in backtest (25% trigger + potential gap on exit). Any strategy that triggers the circuit breaker more than twice in OOS testing receives a significant scoring penalty.

### 8.4 Trade Frequency as Soft Constraint

Trade frequency is evaluated in composite scoring, not as a hard filter:

- **Target range:** 8-40 trades per year (approximately monthly)
- **Scoring:** Full score if 12-24 trades/year; linear penalty scaling to 0 below 8 or above 40
- **Hard minimum:** 30 total trades across the full backtest period (all IS + OOS windows combined) for statistical significance
- **Rejection:** Strategies with fewer than 30 total trades are rejected regardless of other metrics

### 8.5 Composite Scoring Weights

| Metric | Weight | Change from v1 | Rationale |
|--------|:---:|:-:|---|
| Sharpe Ratio | 0.15 | 0.15 (unchanged) | Risk-adjusted return; primary metric |
| Sortino Ratio | 0.15 | new (was in Sharpe) | Penalizes downside volatility only; more relevant for long-only |
| Calmar Ratio | 0.15 | 0.15 (unchanged) | Return relative to worst drawdown; directly relevant to 30% DD constraint |
| Profit Factor | 0.15 | 0.15 (unchanged) | Gross profit / gross loss; robust measure of edge |
| Expectancy | 0.05 | 0.10 -> 0.05 | Average profit per trade; useful but less discriminating |
| Max Drawdown (penalty) | 0.10 | 0.15 -> 0.10 | Reduced because 30% DD hard filter already handles extreme cases |
| Number of Trades | 0.05 | 0.05 (unchanged) | More trades = more statistical confidence |
| OOS Performance Consistency | 0.10 | 0.10 (unchanged) | Low variance of OOS Sharpe across windows |
| Trade Frequency Alignment | 0.05 | new | Proximity to 30-day target holding period |
| Benchmark Outperformance | 0.05 | new | Excess return vs buy-and-hold |
| **Total** | **1.00** | | |

**Normalization:** Each metric is normalized to [0, 1] across all indicators/combinations being evaluated. Max Drawdown is inverted (lower DD = higher score). Composite score = sum of (normalized metric x weight).

### 8.6 Minimum Trade Requirements

| Requirement | Threshold | Consequence |
|-------------|:-:|---|
| Minimum trades (total, all windows) | 30 | Below 30: strategy rejected |
| Minimum trades (per OOS window) | 3 | Below 3 in any window: window flagged as low-confidence |
| Preferred trades (total) | 100+ | Reported confidence level: 30-99 = "moderate", 100+ = "high" |

---

## 9. Profitability Metrics and Composite Scoring

### 9.1 Full Metric Set

Report all of the following for each indicator and combination:

| Metric | Formula/Description | Good Threshold |
|--------|-------------------|:-:|
| **Sharpe Ratio** | (Annualized Return - Risk-Free Rate) / Annualized Volatility | > 1.0 |
| **Sortino Ratio** | (Annualized Return - Risk-Free Rate) / Downside Deviation | > 2.0 |
| **Calmar Ratio** | Annualized Return / Maximum Drawdown | > 1.0 |
| **Profit Factor** | Gross Profit / Gross Loss | > 1.5 |
| **Expectancy** | (Win% x Avg Win) - (Loss% x Avg Loss) | > 0 |
| **Win Rate** | Profitable Trades / Total Trades | Depends on payoff ratio |
| **Payoff Ratio** | Average Win / Average Loss | > 1.0 for trend; < 1.0 OK for mean-reversion |
| **Max Drawdown** | Largest peak-to-trough decline (%, dates, recovery time) | < 30% (hard filter) |
| **Avg Holding Period** | Mean number of trading days per trade | ~30 days target |
| **Time in Market** | Percentage of trading days with open position | Report only |
| **Trades per Year** | Annualized trade count | 8-40 target |
| **Max Consecutive Losses** | Longest losing streak | Report only |
| **Return (annualized)** | CAGR of equity curve | > buy-and-hold |
| **Excess Return vs B&H** | Strategy CAGR - Buy-and-hold CAGR | > 0 |
| **Excess Return vs S&P** | Strategy CAGR - S&P 500 CAGR over same period | Report only |

### 9.2 Risk-Free Rate

Use the 3-month US Treasury yield as the risk-free rate, obtained from:
- User-provided value in YAML config (simplest)
- FRED API (`DGS3MO`) if available
- Default: 0% if not specified (conservative -- makes Sharpe slightly lower)

### 9.3 Rejection Criteria

A strategy is **rejected** (not ranked) if any of the following are true:

| Criterion | Threshold |
|-----------|-----------|
| Max drawdown exceeds limit | > 30% |
| Below buy-and-hold return | Strategy return < Buy-and-hold return (annualized) |
| Insufficient trades | < 30 total trades |
| Negative expectancy | Expectancy <= 0 |
| OOS degradation | OOS Sharpe < 50% of IS Sharpe (average across windows) |

---

## 10. Dual Benchmark Comparison

### 10.1 Two Benchmarks

**Benchmark 1: Buy-and-Hold of the Asset**

- Always-in position: buy at first signal date, hold until end of analysis period
- No transaction costs (single entry)
- Represents the "do nothing" alternative for this specific stock
- This is the primary benchmark: the system must outperform passive holding of the same asset

**Benchmark 2: S&P 500 (SPY)**

- Buy-and-hold of the S&P 500 over the same period
- Represents the "invest in the market instead" alternative
- Requires separate S&P 500 data:
  - **Preferred:** User provides SPY CSV in same format as the asset CSV
  - **Fallback:** Download via `yfinance` (requires internet; cached after first download)
  - **If unavailable:** Skip S&P 500 comparison; report only asset buy-and-hold benchmark
- Configuration: path to S&P 500 CSV or `auto` (yfinance download)

### 10.2 Benchmark Metrics

| Metric | Description |
|--------|-------------|
| **Excess Return (vs B&H)** | Strategy annualized return minus asset buy-and-hold annualized return |
| **Excess Return (vs S&P)** | Strategy annualized return minus S&P 500 annualized return over same period |
| **Sharpe Differential** | Strategy Sharpe minus benchmark Sharpe |
| **Alpha** | Jensen's alpha: strategy return unexplained by market beta |
| **Beta** | Sensitivity of strategy returns to market returns (regression slope) |
| **Information Ratio** | Excess return / Tracking error (active return per unit of active risk) |
| **Tracking Error** | Standard deviation of (strategy returns - benchmark returns) |

### 10.3 Three Levels of "Beating" the Benchmark

1. **Absolute return outperformance:** Strategy return > benchmark return. Simple but ignores risk.
2. **Risk-adjusted outperformance:** Strategy Sharpe > benchmark Sharpe. Accounts for volatility.
3. **Statistically significant outperformance:** Alpha is positive with t-statistic > 2.0 (95% confidence). The most rigorous test.

The application reports all three levels. Rejection criterion uses Level 1 only (strategy must beat asset buy-and-hold on absolute return). Levels 2 and 3 are reported for information.

---

## 11. Verbose Mode and Indicator Rejection

### 11.1 Purpose

Verbose mode provides detailed output explaining why each indicator was accepted, rejected, or skipped. This supports transparency, debugging, and user education.

### 11.2 Data Model

```
IndicatorEvaluation:
    name: str                          # e.g., "RSI(14)"
    category: str                      # e.g., "momentum"
    status: enum [ACCEPTED, REJECTED, SKIPPED]
    rank: int | null                   # null if rejected/skipped
    composite_score: float | null      # [0, 1]

    # Performance metrics (null if skipped)
    metrics:
        sharpe_ratio: float
        sortino_ratio: float
        calmar_ratio: float
        profit_factor: float
        expectancy: float
        max_drawdown: float
        max_drawdown_date: date
        win_rate: float
        payoff_ratio: float
        total_trades: int
        avg_holding_period: float
        trades_per_year: float
        time_in_market: float
        annualized_return: float
        excess_return_bh: float
        excess_return_sp500: float | null

    # WFA details
    wfa_results:
        is_sharpe: float               # Average in-sample Sharpe
        oos_sharpe: float              # Average out-of-sample Sharpe
        oos_sharpe_std: float          # Std dev of OOS Sharpe across windows
        oos_degradation: float         # oos_sharpe / is_sharpe
        num_windows: int
        per_window: list[WindowResult]

    # Rejection details (null if accepted)
    rejection:
        reason: str                    # Primary rejection reason
        details: str                   # Detailed explanation
        failing_metric: str            # Which metric triggered rejection
        failing_value: float           # The actual value
        threshold: float               # The threshold it failed

    # Skip details (null if not skipped)
    skip_reason: str | null            # e.g., "Volume data not available"
```

### 11.3 Rejection Reason Codes

| Code | Description | Example |
|------|-------------|---------|
| `EXCEEDS_MAX_DRAWDOWN` | Max drawdown exceeded 30% | "Max DD was 34.2% on 2022-06-13" |
| `BELOW_BENCHMARK` | Failed to beat buy-and-hold | "CAGR 8.2% vs buy-and-hold 12.1%" |
| `INSUFFICIENT_TRADES` | Fewer than 30 total trades | "Only 18 trades in 5 years" |
| `NEGATIVE_EXPECTANCY` | Expected value per trade <= 0 | "Expectancy -$12.50 per trade" |
| `OOS_DEGRADATION` | OOS performance < 50% of IS | "OOS Sharpe 0.3 vs IS Sharpe 1.2 (25%)" |
| `SKIPPED_NO_VOLUME` | Volume data required but not available | "OBV requires volume; none in dataset" |
| `SKIPPED_INSUFFICIENT_DATA` | Not enough bars for indicator warmup | "Ichimoku needs 78 bars; only 50 available" |

### 11.4 Verbose Output Structure

```
=== INDICATOR EVALUATION REPORT ===

--- EXECUTIVE SUMMARY ---
Total indicators evaluated: 25
  Accepted: 8
  Rejected: 12
  Skipped: 5
Best indicator: MACD + RSI (Combination 1), composite score 0.847

--- ACCEPTED (ranked by composite score) ---
1. MACD + RSI Confirmation     Score: 0.847  Sharpe: 1.42  MaxDD: 18.3%  Trades: 87
2. Williams %R + EMA Trend     Score: 0.791  Sharpe: 1.28  MaxDD: 21.7%  Trades: 104
3. ADX-Filtered EMA Crossover  Score: 0.753  Sharpe: 1.15  MaxDD: 16.5%  Trades: 52
   ...

--- REJECTED (with reasons) ---
1. Donchian Breakout
   Reason: EXCEEDS_MAX_DRAWDOWN
   Detail: Max drawdown 38.7% on 2022-01-03 to 2022-06-16 (115 days)
   Other metrics: Sharpe 0.89, Win Rate 42%, 34 trades

2. Stochastic Oscillator
   Reason: BELOW_BENCHMARK
   Detail: Strategy CAGR 7.3% vs buy-and-hold 11.8%
   Other metrics: Sharpe 0.65, MaxDD 22.1%, 156 trades

3. Ichimoku Cloud
   Reason: OOS_DEGRADATION
   Detail: OOS Sharpe 0.21 vs IS Sharpe 1.05 (20% ratio, threshold 50%)
   ...

--- SKIPPED ---
1. MFI: Requires volume data (not in dataset)
   ...

--- COMBINATIONS ---
[Same format as accepted/rejected, for the 5 curated combinations]

--- CORRELATION MATRIX ---
[Pairwise correlation of accepted indicator returns, flagging pairs > 0.7]
```

---

## 12. Statistical Considerations for Equities

### 12.1 Mean-Reversion Dominance at Short Horizons

US equities exhibit stronger mean-reversion at daily to weekly horizons than other asset classes. This is well-documented in academic literature:

- **Short-term (1-5 days):** Strong mean-reversion. Stocks that fall tend to bounce. RSI and Bollinger Band strategies exploit this.
- **Medium-term (1-6 months):** Momentum effect. Stocks that have risen tend to continue rising. EMA crossover and MACD strategies exploit this.
- **Long-term (3-5 years):** Mean-reversion returns. Extreme winners underperform; extreme losers recover.

This dual regime (short-term reversion, medium-term momentum) is why combining mean-reversion entry (RSI, Bollinger) with trend-following exit (EMA, MACD) can be particularly effective for equities.

### 12.2 Earnings Seasonality

Approximately 80% of S&P 500 companies report earnings in January, April, July, and October. Earnings announcements create:

- **Pre-earnings:** Reduced volume, narrowing Bollinger Bands, ADX decline. Indicators may produce false quiet-market signals.
- **Post-earnings gap:** 5-20% gap (either direction) that distorts indicator readings. RSI, MACD, and oscillators jump to extremes.
- **Indicator recovery:** 3-5 days for indicators to normalize after an earnings gap.

**Application handling:**
- Detect earnings gaps as price moves exceeding 3x rolling ATR(20).
- Flag the gap date in output.
- Do not generate signals on the gap bar or the following 2 bars (indicator recovery period).
- In backtest, count earnings-gap trades separately from normal trades.

### 12.3 Survivorship Bias

Testing on currently successful stocks (e.g., NVDA) introduces survivorship bias:

- NVDA's extraordinary post-2023 performance is not representative of "typical" individual stock behavior.
- Results on NVDA may overstate what the strategy would achieve on a randomly selected stock.
- The application cannot correct for survivorship bias on a single stock, but should:
  - Report this warning in the output.
  - Note that backtest results on any individual stock that has significantly outperformed the market may be optimistically biased.
  - Recommend testing on multiple stocks, including some underperformers, for robustness.

### 12.4 Self-Fulfilling Prophecy

The self-fulfilling prophecy effect is strongest for equities because equities have the largest number of active participants watching the same technical levels:

| Level | Strength of Self-Fulfilling Effect |
|-------|------------------------------------|
| 200-day SMA/EMA | Very strong. Institutional investors use this as a trend filter. |
| 50/200 SMA Golden Cross | Strong. Media coverage amplifies the signal. |
| RSI 30/70 | Strong. Widely watched overbought/oversold levels. |
| Bollinger Band touches | Moderate. Common but less universally watched. |
| MACD signal cross | Moderate to strong. Very popular among retail traders. |
| Williams %R -20/-80 | Moderate. Less popular than RSI but used by active traders. |

**Implication:** Popular indicators on popular stocks may work partly because many traders act on them simultaneously. This is a feature, not a bug -- self-fulfilling prophecy is a legitimate source of short-term profitability.

### 12.5 Academic Evidence

| Period | Finding for US Equities |
|--------|------------------------|
| Pre-1990 | Generally positive evidence for TA profitability |
| 1990-2005 | Declining evidence after controlling for transaction costs and data snooping |
| Post-2005 | Little evidence of consistent profitability in developed stock markets after robust statistical controls |
| Exception | Short-term mean-reversion (RSI, Bollinger) retains some evidence |
| Exception | Momentum factor (medium-term) is well-documented and persistent |

**Adaptive Market Hypothesis (Lo, 2004):** Markets cycle between efficiency and inefficiency. Technical strategies have life cycles -- they work, attract capital, become crowded, stop working, then work again when capital leaves. The application's walk-forward analysis captures this naturally by re-optimizing parameters over time.

### 12.6 NVDA-Specific Considerations

If the primary test asset is NVDA:

- **High beta (1.5-2.0):** NVDA amplifies S&P 500 moves by 1.5-2x. A 20% market decline becomes a 30-40% NVDA decline. The 30% drawdown limit will be tested frequently.
- **Post-2023 AI narrative:** NVDA's rise from ~$15 (split-adjusted) to ~$140+ is an extreme outlier. Any trend-following strategy would show extraordinary returns. This performance is not replicable or predictive.
- **Liquidity:** NVDA is among the most liquid US stocks. Transaction costs and slippage are minimal.
- **Options activity:** Heavy options activity can affect price around monthly expirations and may create short-term support/resistance at round numbers.

---

## 13. Data Format and Handling

### 13.1 Nasdaq CSV Format

The user's data comes from Nasdaq's historical data download. Known characteristics:

| Attribute | Nasdaq CSV Behavior |
|-----------|-------------------|
| Column names | `Date`, `Close/Last`, `Volume`, `Open`, `High`, `Low` |
| Price format | Dollar signs in price fields: `$141.28` |
| Date format | MM/DD/YYYY (e.g., `02/07/2025`) |
| Sort order | Reverse chronological (most recent first) |
| Delimiter | Comma |
| Header | First row |
| Encoding | UTF-8 |

**Auto-detection logic:**

```
1. Read first row as header
2. Detect "Close/Last" column -> rename to "Close"
3. Detect dollar signs in any price column -> strip "$" and convert to float
4. Detect date format by parsing first data row:
   - If "/" present and first segment <= 12: MM/DD/YYYY
   - If "-" present: YYYY-MM-DD (standard)
5. After parsing dates, check sort order:
   - If first date > last date: reverse chronological -> sort ascending
   - If first date < last date: already ascending
6. Set date as DatetimeIndex
```

### 13.2 Split-Adjusted Data

Nasdaq historical download provides split-adjusted prices but NOT dividend-adjusted prices.

**Implications:**

- Price data is correct for indicator calculation (no false signals at split dates).
- **Return calculations over long periods will be understated** because dividends are not added back.
- For NVDA (low dividend yield ~0.03%), this is negligible.
- For high-dividend stocks, the understatement could be material.
- The application should note in output whether data appears to be dividend-adjusted or not.

**Detection heuristic for splits:**

```
If day-over-day price change is approximately -50%, -67%, -75%, or +100%, +200%, +300%
    AND the corresponding volume change is the inverse ratio
    AND no broader market crash on that date
THEN: likely unadjusted split detected -> warn user
```

Since Nasdaq data is split-adjusted, this check serves as a validation rather than a correction.

### 13.3 Earnings Gap Detection

An earnings gap is defined as a price move exceeding 3x the 20-day rolling ATR:

```
gap = abs(Open(today) - Close(yesterday))
if gap > 3 * ATR(20, yesterday):
    flag as earnings_gap
```

When an earnings gap is detected:
- Flag the date in data quality output.
- Exclude the gap bar and the following 2 bars from signal generation (indicator recovery period).
- In backtest, if a position is open and an earnings gap occurs, record it as a gap trade with a note.
- Report gap trades separately from normal trades.
- Do not use the gap move to update trailing stops (the stop should be based on normal-range ATR, not gap-inflated ATR).

### 13.4 NYSE Trading Calendar

Use the `exchange_calendars` library to determine valid trading days:

```python
import exchange_calendars as xcals

nyse = xcals.get_calendar("XNYS")
trading_days = nyse.sessions_in_range(start_date, end_date)
```

This handles:
- NYSE holidays (New Year, MLK Day, Presidents Day, Good Friday, Memorial Day, Juneteenth, Independence Day, Labor Day, Thanksgiving, Christmas)
- Early close days (day before Independence Day, Black Friday, Christmas Eve)
- Special closures (e.g., weather, mourning days)

Use the NYSE calendar to:
1. Identify legitimate gaps (holidays) vs unexpected gaps (data errors) in the input data.
2. Calculate accurate trading-day-based metrics (252 trading days per year).
3. Determine the correct number of bars for indicator warmup.

---

## 14. Daily Run Mode

### 14.1 Two Operating Modes

| Mode | When to Run | What It Does | Output |
|------|-------------|-------------|--------|
| `full` | Weekly or on-demand | Complete backtest, walk-forward analysis, indicator ranking, full report | HTML/PDF report, rankings, trade log |
| `daily` | Evening after market close | Load today's data, compute indicators for best strategy, generate signal for tomorrow | Console signal, updated state |

### 14.2 Full Mode

Full mode runs the complete pipeline:

1. Load all historical data from CSV.
2. Validate data quality.
3. Run walk-forward analysis for all configured indicators and combinations.
4. Rank by composite score.
5. Generate full report.
6. Save the winning strategy configuration to state.

**Frequency:** Weekly (e.g., Sunday evening) or on-demand.

### 14.3 Daily Mode

Daily mode runs a lightweight signal generation:

1. Load the most recent CSV data (user must update the file with today's bar).
2. Load saved state (current position, best strategy from last full run).
3. Compute indicators for the best strategy only.
4. Generate signal for tomorrow's open.
5. Update state (position, trade log).
6. Print to console.

**Console output example:**

```
=== DAILY SIGNAL REPORT ===
Date: 2025-02-07 (Friday)
Asset: NVDA
Strategy: MACD + RSI Confirmation (rank #1 from 2025-02-02 full run)

Current Position: LONG since 2025-01-15 @ $131.50
  Unrealized P/L: +$9.78 (+7.4%)
  Days held: 17
  Trailing stop: $128.20 (2x ATR)

Today's Close: $141.28
RSI(14): 62.3
MACD: 1.47 (above signal 0.89)
ADX: 28.5 (trending)

Signal for Monday 2025-02-10: HOLD
  Reason: MACD bullish, RSI neutral, ADX trending
  Stop update: $133.50 (2x ATR from new high)

Key Levels:
  50 EMA: $134.20
  Upper BB: $145.60
  Lower BB: $127.80
  ATR(14): $3.85

Drawdown Status:
  Current DD from peak: -2.1%
  Circuit breaker: CLEAR (threshold 25%)

Next re-optimization: 2025-02-09 (Sunday full run)
```

### 14.4 State Persistence

State is stored in the `state/` directory:

| File | Contents |
|------|----------|
| `state/position.json` | Current position (direction, entry date, entry price, trailing stop, strategy name) |
| `state/trades.csv` | Completed trade log (entry/exit date, price, P/L, strategy, holding period) |
| `state/last_run.json` | Last full run metadata (date, best strategy, composite score, parameters) |
| `state/cache/` | Cached indicator values and intermediate computations from last full run |

**position.json example:**

```json
{
  "asset": "NVDA",
  "direction": "long",
  "entry_date": "2025-01-15",
  "entry_price": 131.50,
  "current_stop": 133.50,
  "strategy": "macd_rsi_confirmation",
  "strategy_params": {
    "macd": [12, 26, 9],
    "rsi_period": 14,
    "rsi_entry_threshold": 40,
    "rsi_exit_threshold": 30,
    "adx_threshold": 20,
    "atr_stop_multiplier": 2.0
  }
}
```

### 14.5 Re-optimization Schedule

- **Full re-optimization:** Every 63 trading days (quarterly), triggered by comparing `last_run.json` date to current date.
- **Intermediate check:** Weekly full run updates the ranking but uses the same WFA windows.
- **Emergency re-optimization:** If daily mode detects circuit breaker activation, flag for immediate full re-optimization.

---

## 15. Application Architecture

### 15.1 Revised Pipeline

```
Configuration (YAML)
    |
    v
Data Loading ---------> Data Validation ---------> Data Quality Report
    |                        |
    |                   Earnings Gap Detection
    |                   NYSE Calendar Validation
    v
Indicator Calculation (all configured indicators, Tier 1-3)
    |
    v
Signal Generation (entry/exit per indicator, long-only)
    |
    v
Walk-Forward Analysis (504/126 rolling windows)
    |                   |
    |              Circuit Breaker
    |              Drawdown Monitoring
    v
Performance Metrics (full set + dual benchmark)
    |
    v
Composite Scoring & Ranking
    |
    v
Verbose Evaluation Report (accepted/rejected/skipped with reasons)
    |
    v
Report Generation (HTML/PDF/CSV + interactive charts)
    |
    v
State Persistence (for daily mode)
```

### 15.2 Revised File Structure

```
src/
  __init__.py
  config/
    __init__.py
    loader.py              # YAML config loading and validation
    defaults.py            # Default parameter values
  cli/
    __init__.py
    main.py                # CLI entry point (click)
    commands.py            # full, daily, report commands
  data/
    __init__.py
    loader.py              # DataLoader ABC + NasdaqCSVLoader
    validator.py           # OHLCV validation, gap detection
    calendar.py            # NYSE calendar integration
    quality.py             # Data quality report
  indicators/
    __init__.py
    base.py                # Indicator ABC + IndicatorRegistry
    trend.py               # EMA, MACD, MA Crossover, Supertrend, Ichimoku
    momentum.py            # RSI, Williams %R, Stochastic, CCI, ROC
    volatility.py          # Bollinger Bands, ATR, Keltner, Donchian, Std Dev
    volume.py              # OBV, CMF, A/D Line, VPT
    combinations.py        # The 5 curated combinations
  signals/
    __init__.py
    generator.py           # Signal generation (long-only)
    regime.py              # ADX-based regime detection
    composite.py           # Multi-indicator composite scoring
    earnings.py            # Earnings gap signal suppression
  backtest/
    __init__.py
    engine.py              # vectorbt wrapper (long-only)
    walk_forward.py        # WFA orchestration (504/126)
    metrics.py             # Performance metrics (full set)
    scoring.py             # Composite scoring and ranking
    circuit_breaker.py     # Drawdown circuit breaker
    benchmark.py           # Dual benchmark (B&H + S&P 500)
  reporting/
    __init__.py
    charts.py              # mplfinance + Plotly
    html_report.py         # Jinja2 HTML generation
    verbose.py             # Verbose evaluation output
    export.py              # CSV/trade-log export
  portfolio/
    __init__.py
    position.py            # Position tracking (state persistence)
    daily_mode.py          # Daily signal generation
    state.py               # State save/load (JSON/CSV)
  pipeline.py              # Main orchestration (full mode)

tests/
  test_data_loader.py
  test_validator.py
  test_calendar.py
  test_indicators/
    test_trend.py
    test_momentum.py
    test_volatility.py
    test_volume.py
    test_combinations.py
  test_signals.py
  test_backtest.py
  test_walk_forward.py
  test_metrics.py
  test_scoring.py
  test_circuit_breaker.py
  test_benchmark.py
  test_verbose.py
  test_daily_mode.py
  test_pipeline.py

config/
  default_config.yaml      # Default configuration
  example_nvda.yaml        # NVDA example configuration

state/                     # Runtime state (gitignored)
  position.json
  trades.csv
  last_run.json
  cache/

output/                    # Generated reports (gitignored)
  {asset}_{date}/
    report.html
    charts/
    data/
```

### 15.3 Technology Stack

| Component | Recommended | Version | Rationale |
|-----------|------------|:---:|---|
| Language | Python | 3.11+ | Type hints, performance improvements, match statements |
| Data manipulation | pandas | 2.x | Universal standard; all libraries expect pandas DataFrames |
| Indicator computation | TA-Lib | 0.4.28+ | C speed, 150+ indicators, battle-tested; requires C library installation |
| Backtesting | vectorbt | 0.26+ | 10-100x faster than event-driven; long-only mode supported; built-in metrics |
| Performance analytics | quantstats | latest | One-line tearsheet generation; Sharpe, Sortino, Calmar, drawdown analysis |
| Static charts | mplfinance | 0.12.10+ | Purpose-built for financial candlestick charts |
| Interactive charts | Plotly | 5.18+ | HTML output, zoom, hover, range selection |
| Report generation | Jinja2 | 3.1+ | HTML templating for professional reports |
| Trading calendar | exchange_calendars | latest | NYSE/NASDAQ holiday and session data |
| S&P 500 data | yfinance | latest | Benchmark download fallback (cached) |
| Configuration | PyYAML | 6.0+ | YAML config loading |
| CLI | click | 8.x | Command-line interface |
| Caching | pyarrow | 14+ | Parquet intermediate storage |
| Testing | pytest | 7.4+ | Test framework |

### 15.4 Revised YAML Configuration Template

```yaml
# analysis_config.yaml -- US Equities Long-Only

asset:
  name: "NVDA"
  class: equity               # Fixed: equity only

data:
  source: file
  path: data/nvda_daily.csv
  format: nasdaq_csv          # Auto-detects: dollar signs, Close/Last, MM/DD/YYYY, reverse chrono
  # Column mapping (auto-detected for Nasdaq format, override if needed):
  # columns:
  #   date: Date
  #   close: "Close/Last"
  #   volume: Volume
  #   open: Open
  #   high: High
  #   low: Low

benchmark:
  sp500:
    source: file              # file or auto (yfinance download)
    path: data/spy_daily.csv  # Required if source: file
    # source: auto            # Alternative: download via yfinance

analysis:
  direction: long_only        # Fixed: long-only
  start_date: null            # null = use all available data
  end_date: null

indicators:
  # Tier 1
  tier1:
    - name: rsi
      params:
        period: [7, 14]
        overbought: 70
        oversold: 30
    - name: rsi_short
      params:
        period: [2, 3, 5]
        overbought: 90
        oversold: 10
    - name: williams_r
      params:
        period: [10, 14]
    - name: ema
      params:
        periods: [20, 50, 200]
    - name: macd
      params:
        fast: 12
        slow: 26
        signal: 9
    - name: bollinger_bands
      params:
        period: 20
        std_dev: 2.0
    - name: adx
      params:
        period: 14
        trend_threshold: 25
        no_trend_threshold: 20
    - name: atr
      params:
        period: 14

  # Tier 2
  tier2:
    - name: obv
    - name: cmf
      params:
        period: 20
    - name: ma_crossover
      params:
        fast: [20, 50]
        slow: [50, 200]
    - name: supertrend
      params:
        atr_period: [10, 14]
        multiplier: [2.5, 3.0]

  # Tier 3 (optional, enable selectively)
  tier3:
    enabled: false            # Set to true to include Tier 3
    indicators:
      - name: keltner_channels
      - name: cci
      - name: stochastic
      - name: parabolic_sar
      - name: donchian_channels
      - name: ad_line
      - name: vpt
      - name: sma
      - name: ichimoku

combinations:
  - name: macd_rsi_confirmation
    entry: "macd_cross_up AND rsi > 40"
    exit: "macd_cross_down OR rsi < 30"
    adx_filter: 20
    atr_stop: 2.0

  - name: adx_ema_crossover
    entry: "ema_20_above_50 AND adx > 25"
    exit: "ema_20_below_50 OR adx < 20"
    volume_filter: "cmf > 0"
    atr_stop: 2.5

  - name: rsi_bollinger_reversion
    entry: "rsi < 30 AND price < lower_bb AND adx < 30"
    exit: "rsi > 70 OR price > upper_bb OR price > middle_bb"
    atr_stop: 2.0

  - name: supertrend_macd
    entry: "supertrend_bullish AND macd_hist > 0"
    exit: "supertrend_bearish"

  - name: williams_r_ema_trend
    entry: "williams_r_cross_above_neg80 AND price > ema_50"
    exit: "williams_r_cross_below_neg20 OR price < ema_50"
    atr_stop: 2.0

backtest:
  initial_capital: 10000
  transaction_cost: 0.0002    # 0.02% per trade
  walk_forward:
    in_sample_days: 504       # ~2 years
    out_of_sample_days: 126   # ~6 months
    step_days: 126
    warmup_days: 200
    method: rolling           # rolling or anchored
  circuit_breaker:
    level1_pct: 15            # Reduce position size
    level2_pct: 20            # No new entries
    level3_pct: 25            # Force close all
    resume_days: 10           # Days above L2 to resume
  max_drawdown: 30            # Hard rejection filter (%)
  min_trades: 30              # Minimum total trades for significance

scoring:
  weights:
    sharpe_ratio: 0.15
    sortino_ratio: 0.15
    calmar_ratio: 0.15
    profit_factor: 0.15
    expectancy: 0.05
    max_drawdown: 0.10
    num_trades: 0.05
    oos_consistency: 0.10
    trade_frequency: 0.05
    benchmark_outperformance: 0.05
  trade_frequency:
    target_days: 30           # Target average holding period
    min_trades_year: 8
    max_trades_year: 40

output:
  directory: output
  formats: [html, csv]
  interactive_charts: true
  verbose: true               # Enable verbose evaluation output

daily:
  enabled: true
  reoptimize_interval: 63     # Trading days between full re-optimizations
```

### 15.5 Options Extension Readiness

The architecture supports future options integration through extension points:

**Instrument ABC:**

```
Instrument (ABC)
    |
    +-- Stock (current implementation)
    +-- Option (future)
         - strike, expiry, type (call/put)
         - greeks (delta, gamma, theta, vega)
```

**PositionLeg model:**

```
Position
    |
    +-- legs: List[PositionLeg]
         - instrument: Instrument
         - quantity: int
         - direction: long/short
         - entry_price: float
         - entry_date: date
```

**DataLoader ABC:**

```
DataLoader (ABC)
    |
    +-- NasdaqCSVLoader (current)
    +-- YFinanceLoader (benchmark)
    +-- OptionsChainLoader (future)
    +-- APILoader (future)
```

These extension points do not add complexity to the current implementation but provide clean seams for future enhancement.

---

## 16. Future Extensions

### 16.1 Options Integration

- **Covered calls:** Enhance long-only positions with income generation.
- **Protective puts:** Enforce 30% drawdown limit mechanically.
- **Options chain data:** Different data format, different indicators (implied volatility, put/call ratio).
- **Greeks-based signals:** Delta-neutral strategies, gamma scalping.
- **Data sources:** CBOE, Polygon.io options chain, or user-provided CSV.

### 16.2 API Data Source Patterns

Replace or supplement CSV input with programmatic data access:

| API Pattern | Implementation |
|-------------|---------------|
| Real-time data | Polygon.io WebSocket, Alpaca API |
| End-of-day | yfinance, Alpha Vantage |
| Options chain | CBOE DataShop, Polygon.io |
| Fundamentals | Financial Modeling Prep, Alpha Vantage |
| News/sentiment | Benzinga, NewsAPI |

The `DataLoader` ABC pattern supports adding any of these without modifying existing code.

### 16.3 Multi-Stock Portfolio Mode

- Test the same strategy across multiple stocks simultaneously.
- Portfolio-level position sizing (Kelly criterion, risk parity).
- Correlation-aware position limits.
- Sector exposure constraints.

### 16.4 Machine Learning Regime Detection

- Replace ADX-based regime detection with Hidden Markov Models (`hmmlearn`).
- Feature set: returns, volatility, volume, VIX (if available).
- Train on historical data to identify 2-4 hidden states.
- Use predicted state to select indicator strategy.

---

## 17. Summary of All Changes from v1

### 17.1 Comprehensive Change Table

| Area | v1 | v2 | Impact |
|------|----|----|--------|
| **Scope** | Multi-asset (commodities, forex, crypto, equities) | US equities only | Simplified data handling, focused indicator selection |
| **Direction** | Long and short | Long-only | Asymmetric signals; exit = sell to cash |
| **Tier 1 count** | 6 indicators | 7 indicators | Williams %R promoted |
| **Williams %R** | Tier 2 | Tier 1 | 81% WR on S&P 500; long-only optimal |
| **OBV** | Tier 4 | Tier 2 | Volume fully available for equities |
| **CMF** | Tier 4 | Tier 2 | Bounded volume oscillator; equity filter |
| **A/D Line** | Tier 4 | Tier 3 | Supplementary volume |
| **VPT** | Tier 4 | Tier 3 | Supplementary volume |
| **SMA** | Tier 4 | Tier 3 | 200-day widely watched; Golden Cross |
| **Donchian** | Tier 2 | Tier 3 | Designed for commodities, not equities |
| **Ichimoku** | Tier 3 | Tier 3 (lower) | Underperforms B&H on S&P 500 |
| **WFA in-sample** | 252 days (1 year) | 504 days (2 years) | More robust optimization |
| **WFA out-of-sample** | 63 days (quarter) | 126 days (6 months) | Better OOS assessment |
| **Transaction cost** | 0.1% round trip | 0.04% round trip | Reflects modern equity brokerage |
| **Benchmark** | Buy-and-hold only | Dual: B&H + S&P 500 | More rigorous evaluation |
| **Drawdown limit** | None (reported only) | 30% hard rejection | Capital preservation priority |
| **Trade frequency** | Not constrained | ~30-day target (soft) | Aligns with swing trading |
| **Circuit breaker** | Not present | 15/20/25% staged | Prevents catastrophic drawdown |
| **Composite weights** | 8 metrics | 10 metrics | Added trade frequency, benchmark outperformance |
| **Expectancy weight** | 0.10 | 0.05 | Less discriminating for equities |
| **Max DD weight** | 0.15 | 0.10 | Hard filter handles extreme cases |
| **Sortino** | Not separate | 0.15 weight | More relevant for long-only |
| **Data format** | Generic CSV | Nasdaq CSV with quirks | Auto-detection of format |
| **Calendar** | Not specified | NYSE via exchange_calendars | Proper trading day handling |
| **Earnings gaps** | Not addressed | 3x ATR detection + suppression | Prevents false signals |
| **Split detection** | Not addressed | Heuristic validation | Data quality |
| **Daily mode** | Not present | Evening signal generation | Practical daily use |
| **State persistence** | Not present | position.json, trades.csv | Continuity between runs |
| **Verbose mode** | Not present | Detailed accept/reject output | Transparency |
| **Asset class config** | commodity/forex/crypto/equity | equity (fixed) | Simplified |
| **Volume indicators** | Conditional on asset class | Always available | Simplified logic |
| **Forex mode** | Disable volume, tick vol proxy | Removed | Not applicable |
| **Crypto mode** | Adjusted thresholds, wider stops | Removed | Not applicable |
| **Commodity mode** | Trend-following emphasis | Removed | Not applicable |
| **Options readiness** | Not addressed | Extension points in architecture | Future-proofed |
| **Combinations** | 5 generic | 5 equity-specific with full entry/exit logic | Actionable |
| **S&P 500 data** | Not needed | Required (CSV or yfinance) | Dual benchmark |
| **Re-optimization** | Quarterly WFA | 63-day interval with emergency trigger | Adaptive |
| **Alpha/beta** | Not computed | Part of benchmark metrics | Risk decomposition |
| **Information ratio** | Not computed | Part of benchmark metrics | Active return assessment |

### 17.2 What Was Removed

- Multi-asset support (commodities, forex, crypto)
- Asset class detection and asset-class-aware defaults
- Short-selling logic
- Forex tick volume handling
- Crypto threshold adjustments
- Commodity seasonality considerations
- Multi-asset YAML configuration options

### 17.3 What Was Added

- Nasdaq CSV auto-detection and parsing
- NYSE trading calendar integration
- Earnings gap detection and signal suppression
- Split detection heuristic
- Dual benchmark comparison (B&H + S&P 500)
- Alpha, beta, information ratio, tracking error
- 30% maximum drawdown hard rejection filter
- Staged circuit breaker (15/20/25%)
- ~30-day trade frequency scoring
- Long-only signal asymmetry (entry/exit distinction)
- Cash period tracking and opportunity cost reporting
- Daily run mode with console output
- State persistence (position.json, trades.csv, last_run.json)
- Verbose evaluation with rejection reasons and codes
- Indicator correlation matrix
- Volume confirmation framework (OBV + CMF)
- Options extension readiness (Instrument ABC, PositionLeg)
- Sortino ratio as separate weighted metric
- OOS degradation rejection criterion
- Survivorship bias warning
- Self-fulfilling prophecy analysis for equities
- NVDA-specific risk considerations

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
8. de Prado (2018) -- Advances in Financial Machine Learning

### Backtesting and Strategy Studies
- [MACD and RSI Strategy: 73% Win Rate (QuantifiedStrategies)](https://www.quantifiedstrategies.com/macd-and-rsi-strategy/)
- [RSI Trading Strategy: 91% Win Rate (QuantifiedStrategies)](https://www.quantifiedstrategies.com/rsi-trading-strategy/)
- [Williams %R Trading Strategy: 81% Win Rate (QuantifiedStrategies)](https://www.quantifiedstrategies.com/williams-r-trading-strategy/)
- [Bollinger Band Squeeze Strategy (QuantifiedStrategies)](https://www.quantifiedstrategies.com/bollinger-band-squeeze-strategy/)
- [Bollinger Bands Trading Strategies (QuantifiedStrategies)](https://www.quantifiedstrategies.com/bollinger-bands-trading-strategy/)
- [CCI Trading Strategy (QuantifiedStrategies)](https://www.quantifiedstrategies.com/cci-trading-strategy/)
- [ROC Trading Strategy (QuantifiedStrategies)](https://www.quantifiedstrategies.com/rate-of-change-trading-strategy/)
- [Supertrend Indicator Strategy (QuantifiedStrategies)](https://www.quantifiedstrategies.com/supertrend-indicator-trading-strategy/)
- [Keltner Channel Trading Strategy (QuantifiedStrategies)](https://www.quantifiedstrategies.com/keltner-bands-trading-strategies/)
- [Parabolic SAR Trading Strategy (QuantifiedStrategies)](https://www.quantifiedstrategies.com/parabolic-sar-trading-strategy/)
- [Ichimoku Cloud Trading Strategy (QuantifiedStrategies)](https://www.quantifiedstrategies.com/ichimoku-strategy/)
- [Alexander Elder Triple Screen Strategy (QuantifiedStrategies)](https://www.quantifiedstrategies.com/alexander-elder-triple-screen-strategy/)
- [Donchian Channels Trading Strategy (QuantifiedStrategies)](https://www.quantifiedstrategies.com/donchian-channel/)
- [MACD and Bollinger Bands Strategy: 78% Win Rate (QuantifiedStrategies)](https://www.quantifiedstrategies.com/macd-and-bollinger-bands-strategy/)

### Large-Scale Backtesting
- [Ichimoku Cloud: 15,024 Trades Tested (LiberatedStockTrader)](https://www.liberatedstocktrader.com/ichimoku-cloud/)
- [Parabolic SAR Test Results: 2,880 years (LiberatedStockTrader)](https://www.liberatedstocktrader.com/parabolic-sar/)
- [Supertrend: 4,052 Trades Tested (LiberatedStockTrader)](https://www.liberatedstocktrader.com/supertrend-indicator/)
- [Rate of Change: 66% Win Rate on DJIA (LiberatedStockTrader)](https://www.liberatedstocktrader.com/rate-of-change-indicator/)

### Indicator Comparison Studies
- [RSI vs CCI vs Williams %R Performance (FXSSI)](https://fxssi.com/test-rsi-cci-and-williams-r-indicators)
- [EMA vs SMA Backtested Results (HorizonAI)](https://www.horizontrading.ai/learn/ema-vs-sma-comparison)
- [MACD Crossovers in Trending vs Ranging Markets (LuxAlgo)](https://www.luxalgo.com/blog/macd-crossovers-in-trending-vs-ranging-markets/)

### Golden Cross / Death Cross Studies
- [Testing the Golden Cross and Death Cross on SPY (Cabot Wealth)](https://www.cabotwealth.com/daily/how-to-invest/testing-the-golden-cross-and-death-cross-on-the-spy)
- [Golden Cross vs Death Cross (XS)](https://www.xs.com/en/blog/golden-cross-vs-death-cross/)

### Additional Research
- [Technical Analysis Meets Machine Learning: Bitcoin (arXiv 2024)](https://arxiv.org/pdf/2511.00665)
- [Unlocking Trading Insights: RSI and MA Indicators (Sage Journals, 2025)](https://journals.sagepub.com/doi/10.1177/09726225241310978)
- [Effectiveness of RSI and MACD in Addressing Stock Price Volatility (ResearchGate)](https://www.researchgate.net/publication/392317792)
- [Bollinger Bands under Varying Market Regimes: BTC/USDT (SSRN)](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=5775962)

### Reference Documentation
- [ADX Indicator (StockCharts)](https://chartschool.stockcharts.com/table-of-contents/technical-indicators-and-overlays/technical-indicators/average-directional-index-adx)
- [exchange_calendars (Python library)](https://github.com/gerrymanoim/exchange_calendars)
- [vectorbt documentation](https://vectorbt.dev/)
- [TA-Lib documentation](https://ta-lib.github.io/ta-lib-python/)
- [quantstats documentation](https://github.com/ranaroussi/quantstats)

---

## Supplementary Documents

| Document | Contents |
|----------|---------|
| [research/Research_v1.md](Research_v1.md) | Previous version: multi-asset research |
| [research/technical-indicators-research.md](technical-indicators-research.md) | Detailed 25-indicator analysis with citations (v1 reference) |
| [research/data-considerations-and-architecture.md](data-considerations-and-architecture.md) | Data engineering, libraries, architecture (v1 reference) |
| [notes/Research_Thinking_v1.md](../notes/Research_Thinking_v1.md) | v1 reasoning and methodology |
| [notes/Research_Questions_v1.md](../notes/Research_Questions_v1.md) | 33 questions from v1 (many now answered) |
| notes/Research_Thinking_v2.md | v2 reasoning and methodology (separate document) |
