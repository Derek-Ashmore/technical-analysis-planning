# Technical Analysis Research Report v1

## Executive Summary

This report presents comprehensive research on technical analysis (TA) indicators and methodologies for analyzing historical daily spot price data. The goal is to identify which indicators best fit the data -- meaning those that indicate buys and sells most profitably -- and to establish a rigorous framework for evaluating, comparing, and reporting on indicator performance.

The research covers five categories of indicators (27+ individual indicators), backtesting methodology, profitability metrics, indicator combination strategies, open-source tooling, and pattern recognition approaches.

**Key finding**: No single indicator is sufficient. Academic research and practitioner experience consistently show that multi-indicator strategies using hierarchical filtering (trend filter + momentum entry + volatility-based risk management) outperform single-indicator approaches by approximately 23%. The application should use ADX as a market-regime detector to dynamically select between trend-following and mean-reversion indicator sets.

---

## Table of Contents

1. [Technical Analysis Indicators](#1-technical-analysis-indicators)
   - [1.1 Trend-Following Indicators](#11-trend-following-indicators)
   - [1.2 Momentum Indicators](#12-momentum-indicators)
   - [1.3 Volatility Indicators](#13-volatility-indicators)
   - [1.4 Volume Indicators](#14-volume-indicators)
   - [1.5 Pattern-Based / Complex Indicator Systems](#15-pattern-based--complex-indicator-systems)
2. [Backtesting Methodology](#2-backtesting-methodology)
3. [Profitability Metrics](#3-profitability-metrics)
4. [Trade-Level Reporting](#4-trade-level-reporting)
5. [Summary Statistics](#5-summary-statistics)
6. [Indicator Ranking and "Best Fit" Determination](#6-indicator-ranking-and-best-fit-determination)
7. [Indicator Combination Strategies](#7-indicator-combination-strategies)
8. [Proven Indicator Combinations](#8-proven-indicator-combinations)
9. [Adaptive Indicator Selection and Market Regime Detection](#9-adaptive-indicator-selection-and-market-regime-detection)
10. [Pattern Identification](#10-pattern-identification)
11. [Open-Source Libraries and Tools](#11-open-source-libraries-and-tools)
12. [Data Format and Preprocessing](#12-data-format-and-preprocessing)
13. [Common Pitfalls](#13-common-pitfalls)
14. [Optimization Approaches](#14-optimization-approaches)
15. [Academic Research Summary](#15-academic-research-summary)
16. [Design Recommendations](#16-design-recommendations)

---

## 1. Technical Analysis Indicators

### 1.1 Trend-Following Indicators

#### Simple Moving Average (SMA)

- **What it measures**: Arithmetic mean of closing prices over N periods. Smooths price data to identify trend direction.
- **Parameters for daily data**: 10, 20, 50, 100, 200 periods. The 50 and 200-day SMAs are the most widely watched.
- **Buy/Sell signals**:
  - Buy when price crosses above SMA from below
  - Sell when price crosses below SMA from above
  - **Golden Cross**: 50-day SMA crosses above 200-day SMA (strong buy)
  - **Death Cross**: 50-day SMA crosses below 200-day SMA (strong sell)
- **Strengths**: Simple to compute, widely followed (self-fulfilling), excellent for identifying long-term trends and major support/resistance zones
- **Weaknesses**: Highest lag of all MA types, gives equal weight to old and new data, generates late signals in fast-moving markets
- **Market regime**: Works well in trending markets; generates many false signals (whipsaws) in ranging/choppy markets
- **Academic evidence**: Ned Davis Research shows stocks experiencing a Golden Cross outperformed the market by avg 1.5% over 3 months. Death Cross stocks underperformed by avg 2.24% over 6 months. False signals occur ~35% of the time.

#### Exponential Moving Average (EMA)

- **What it measures**: Weighted moving average giving exponentially more weight to recent prices, making it more responsive than SMA.
- **Parameters for daily data**: 10, 20, 50, 200 periods. 12 and 26-period EMAs used in MACD calculation.
- **Buy/Sell signals**:
  - Buy when price crosses above EMA; sell when price crosses below
  - EMA crossover: Buy when shorter EMA (e.g., 12) crosses above longer EMA (e.g., 26)
  - Strong buy when price breaks above EMA with high volume
- **Strengths**: More responsive to recent price changes than SMA, better for momentum-based strategies, catches trends earlier
- **Weaknesses**: More prone to false signals due to sensitivity, can be noisy in choppy markets
- **Market regime**: Better than SMA in trending markets due to faster response; still problematic in ranging markets

#### Weighted Moving Average (WMA)

- **What it measures**: Assigns linearly decreasing weights to older prices. Most recent price gets highest weight.
- **Parameters for daily data**: 5-20 periods typical; 10-period WMA common
- **Strengths**: More responsive than both SMA and EMA for very short-term trading
- **Weaknesses**: Too focused on recent prices, can generate many false signals

#### Double Exponential Moving Average (DEMA)

- **What it measures**: Advanced EMA applying exponential smoothing twice to reduce lag.
- **Formula**: `DEMA = 2 * EMA(price, N) - EMA(EMA(price, N), N)`
- **Parameters for daily data**: 10, 20, 50, 100, 200 periods
- **Strengths**: Significantly less lag than EMA, good for swing trading
- **Weaknesses**: More noise/false signals, higher sensitivity to whipsaws

#### Triple Exponential Moving Average (TEMA)

- **What it measures**: Applies triple exponential smoothing for minimal lag.
- **Formula**: `TEMA = (3 * EMA1) - (3 * EMA2) + EMA3`
- **Parameters for daily data**: 10, 20, 50 periods
- **Strengths**: Lowest lag of all standard MAs, catches trend changes earliest
- **Weaknesses**: Most prone to false signals of all MA types

#### Moving Average Crossover Strategies Summary

| Strategy | Type | Parameters | Best For |
|----------|------|-----------|----------|
| Price vs MA | Single crossover | Price vs 20/50/200 SMA or EMA | Simplest, most signals |
| Short/Long MA | Double crossover | 10/50, 20/50, 50/200 | Reduced noise |
| Short/Medium/Long | Triple crossover | 10/50/200 | Best confirmation |
| Golden Cross | Long-term | 50-day vs 200-day SMA | Position trading |

- Golden Cross averaged gains of 7.43% over 40 days with highest probability of success (Quantifiable Edges research)

---

### 1.2 Momentum Indicators

#### RSI (Relative Strength Index)

- **What it measures**: Speed and magnitude of recent price changes to evaluate overbought/oversold conditions. Ranges 0-100.
- **Formula**: `RSI = 100 - (100 / (1 + RS))`, where `RS = avg gain / avg loss over N periods`
- **Parameters for daily data**: 14 periods (standard), 7 (more sensitive), 21 (smoother)
- **Overbought/Oversold**: Above 70 = overbought, Below 30 = oversold
- **Buy/Sell signals**:
  - Buy when RSI crosses above 30 from below (oversold reversal)
  - Sell when RSI crosses below 70 from above (overbought reversal)
  - Bullish divergence: price makes lower low but RSI makes higher low
  - Bearish divergence: price makes higher high but RSI makes lower high
  - Centerline crossover: RSI crossing above/below 50 for trend confirmation
- **Strengths**: Works very well in ranging markets, clear signal levels, widely understood
- **Weaknesses**: Can remain overbought/oversold for extended periods in strong trends
- **Market regime**: Best in ranging/mean-reverting markets; in strong trends, overbought/oversold readings confirm trend strength rather than signaling reversal
- **Academic evidence**: Combined MACD+RSI strategies show 73% win rate

#### MACD (Moving Average Convergence Divergence)

- **What it measures**: Trend-following momentum indicator showing relationship between two EMAs.
- **Components**: MACD Line = 12-EMA minus 26-EMA; Signal Line = 9-EMA of MACD Line; Histogram = MACD minus Signal
- **Parameters for daily data**: 12, 26, 9 (standard)
- **Buy/Sell signals**:
  - Buy when MACD crosses above Signal Line
  - Sell when MACD crosses below Signal Line
  - Buy when MACD crosses above zero line (bullish momentum confirmed)
  - Histogram growing = strengthening momentum; shrinking = weakening
  - Divergence signals with price
- **Strengths**: Combines trend and momentum information, widely used, versatile
- **Weaknesses**: Lagging indicator, not bounded (no absolute overbought/oversold levels)
- **Market regime**: Excels in trending markets with 70-80% success rate and 2-3% per trade; many false crossovers in ranging markets

#### Stochastic Oscillator

- **What it measures**: Compares closing price to price range over N periods. Ranges 0-100.
- **Components**: %K (fast line) and %D (slow line, SMA of %K)
- **Parameters for daily data**: 14, 3, 3 (14-period lookback, 3-period %K smoothing, 3-period %D)
- **Overbought/Oversold**: Above 80 = overbought, Below 20 = oversold
- **Buy/Sell signals**:
  - Buy when %K crosses above %D in oversold territory (below 20)
  - Sell when %K crosses below %D in overbought territory (above 80)
- **Strengths**: Clear bounded range, early signal generation
- **Weaknesses**: Very sensitive, can stay overbought/oversold in strong trends

#### Williams %R

- **What it measures**: Momentum oscillator showing overbought/oversold by comparing close to high-low range. Ranges 0 to -100.
- **Parameters for daily data**: 14 periods (standard)
- **Overbought/Oversold**: 0 to -20 = overbought, -80 to -100 = oversold
- **Buy/Sell signals**:
  - Buy when %R crosses above -80 from below
  - Sell when %R crosses below -20 from above
  - Failure swing patterns for high-probability trades

#### CCI (Commodity Channel Index)

- **What it measures**: Deviation of price from its statistical average. Unbounded oscillator.
- **Formula**: `CCI = (Typical Price - SMA of TP) / (0.015 * Mean Deviation)`
- **Parameters for daily data**: 20 periods (standard)
- **Levels**: Above +100 = overbought; Below -100 = oversold
- **Signal selectivity**: Captures ~70-80% of values between +/-100 so signals only fire 20-30% of time

#### ROC (Rate of Change)

- **What it measures**: Pure momentum -- percentage change in price over N periods.
- **Formula**: `ROC = [(Current Price - Price N periods ago) / Price N periods ago] * 100`
- **Parameters for daily data**: 10-14 (standard), 20-30 (smoother)
- **Buy/Sell signals**: Buy when ROC crosses above zero; sell when crosses below zero
- **Note**: No fixed overbought/oversold levels; thresholds are asset-specific

#### MFI (Money Flow Index)

- **What it measures**: Volume-weighted RSI combining price AND volume to measure buying/selling pressure. Ranges 0-100.
- **Parameters for daily data**: 14 periods (standard)
- **Overbought/Oversold**: Above 80 = overbought; Below 20 = oversold
- **Buy/Sell signals**:
  - Buy when MFI drops below 20 then rebounds above 20
  - Sell when MFI rises above 80 then drops below 80
  - Failure swing patterns are high-probability signals
- **Strengths**: Incorporates volume (unlike RSI), failure swings are high-probability

---

### 1.3 Volatility Indicators

#### Bollinger Bands

- **What it measures**: Volatility envelope around a moving average using standard deviations.
- **Components**: Middle = 20-period SMA; Upper = SMA + 2 SD; Lower = SMA - 2 SD
- **Parameters for daily data**: 20, 2 (standard)
- **Buy/Sell signals**:
  - Buy when price touches/crosses below lower band (potential oversold bounce)
  - Sell when price touches/crosses above upper band (potential overbought)
  - **Bollinger Squeeze**: Bands narrow significantly = low volatility, anticipating breakout
  - Walk the band: In strong trends, price "walks" along a band (continuation, not reversal)
  - %B indicator: Measures where price is relative to bands (0 = lower, 1 = upper)
- **Market regime**: In ranging markets, great for mean-reversion. In trending markets, price walks the bands.

#### ATR (Average True Range)

- **What it measures**: Average of true ranges over N periods. Measures volatility magnitude, not direction.
- **True Range**: `max(H-L, |H-Prev Close|, |L-Prev Close|)`
- **Parameters for daily data**: 14 periods (standard)
- **Usage** (not a directional signal generator):
  - Stop-loss placement: 1.5x to 3x ATR from entry
  - Position sizing: Smaller positions when ATR high, larger when low
  - Volatility breakout: Price moves > 1-2x ATR from previous close
  - Low ATR periods indicate consolidation; breakouts tend to be powerful

#### Keltner Channels

- **What it measures**: Volatility-based channels using ATR around an EMA.
- **Components**: Middle = 20-period EMA; Upper = EMA + 2 * ATR; Lower = EMA - 2 * ATR
- **Buy/Sell signals**:
  - Buy when price closes above upper channel (bullish breakout)
  - Sell when price closes below lower channel (bearish breakout)
  - **Keltner/Bollinger Squeeze**: When Bollinger Bands contract inside Keltner Channels, extremely low volatility precedes significant moves
- **Strengths**: Smoother than Bollinger Bands (uses ATR instead of SD), excellent for breakout identification

#### Donchian Channels

- **What it measures**: Highest high and lowest low over N periods.
- **Components**: Upper = Highest high of N periods; Lower = Lowest low of N periods; Middle = (Upper + Lower) / 2
- **Parameters for daily data**: 20 periods (standard)
- **Buy/Sell signals**:
  - Buy when price breaks above upper band (new N-period high)
  - Sell when price breaks below lower band (new N-period low)
  - **Turtle Trading system**: Buy on 20-day breakout, exit on 10-day breakout in opposite direction
- **Historical note**: The Turtle Trading system (using Donchian Channels) produced average annual returns of 80% in the 1980s.

---

### 1.4 Volume Indicators

#### OBV (On-Balance Volume)

- **What it measures**: Cumulative volume indicator. Adds volume on up days, subtracts on down days.
- **Formula**: If close > prev close: `OBV = prev OBV + volume`; If close < prev close: `OBV = prev OBV - volume`
- **Buy/Sell signals**:
  - Bullish divergence: Price makes lower lows but OBV makes higher lows (accumulation)
  - Bearish divergence: Price makes higher highs but OBV makes lower highs (distribution)
  - OBV breaking to new highs before price = leading buy signal
- **Strengths**: Leading indicator (OBV often moves before price), shows institutional accumulation/distribution

#### VWAP (Volume-Weighted Average Price)

- **What it measures**: Average price weighted by volume.
- **Formula**: `VWAP = Cumulative(Price * Volume) / Cumulative(Volume)`
- **Buy/Sell signals**: Price above VWAP = bullish; below = bearish; acts as dynamic support/resistance
- **Note for daily data**: Anchored VWAP (from significant events) is more useful than daily-reset VWAP for position-level trading

#### Accumulation/Distribution Line (A/D Line)

- **What it measures**: Volume-weighted measure of money flow considering where close falls within the high-low range.
- **Formula**: `Money Flow Multiplier = [(Close - Low) - (High - Close)] / (High - Low)`
- **Strengths**: Better than OBV because it considers where close falls in range (not just up/down day)
- **Signals**: Divergences between A/D Line and price indicate potential reversals

#### Chaikin Money Flow (CMF)

- **What it measures**: Bounded version of A/D Line concept. Ranges from -1 to +1.
- **Parameters for daily data**: 20 or 21 periods (standard)
- **Buy/Sell signals**:
  - CMF above zero = buying pressure; below zero = selling pressure
  - Buy when CMF crosses above zero; sell when crosses below
  - Strong signals: CMF above +0.25 or below -0.25

---

### 1.5 Pattern-Based / Complex Indicator Systems

#### Ichimoku Cloud (Ichimoku Kinko Hyo)

- **What it measures**: Complete trading system providing trend direction, momentum, support/resistance, and signals.
- **Components**:
  1. Tenkan-sen (Conversion Line): (9-period high + 9-period low) / 2
  2. Kijun-sen (Base Line): (26-period high + 26-period low) / 2
  3. Senkou Span A (Leading Span A): (Tenkan + Kijun) / 2, plotted 26 periods ahead
  4. Senkou Span B (Leading Span B): (52-period high + 52-period low) / 2, plotted 26 periods ahead
  5. Chikou Span (Lagging Span): Current close plotted 26 periods behind
- **Parameters for daily data**: 9, 26, 52 (traditional)
- **Buy/Sell signals**:
  - **TK Cross**: Buy when Tenkan crosses above Kijun (especially above cloud)
  - **Cloud breakout**: Buy when price moves above cloud
  - **Cloud color**: Green (Span A > Span B) = bullish; Red = bearish
  - **Strong buy**: All 5 conditions align bullishly
  - Cloud thickness indicates support/resistance strength
- **Strengths**: All-in-one system, multiple confirmation signals, projects future support/resistance
- **Weaknesses**: Complex, many components can conflict, lagging in fast markets

#### Parabolic SAR (Stop and Reverse)

- **What it measures**: Trailing stop system following price like a parabola.
- **Parameters for daily data**: AF start = 0.02, AF increment = 0.02, AF maximum = 0.20
- **Buy/Sell signals**:
  - Buy when SAR dots flip from above to below price
  - Sell when SAR dots flip from below to above price
  - SAR value serves as automatic trailing stop-loss
- **Strengths**: Always in the market, provides automatic stop-loss levels
- **Weaknesses**: Many false signals in sideways markets. Pair with ADX > 25 filter.

#### ADX/DMI (Average Directional Index / Directional Movement Index)

- **What it measures**: Trend STRENGTH (not direction). Combined with +DI/-DI for direction.
- **Parameters for daily data**: 14 periods (standard)
- **Interpretation**:
  - ADX < 20: No trend / ranging market (use mean-reversion strategies)
  - ADX 20-25: Trend may be developing
  - ADX > 25: Strong trend (use trend-following strategies)
  - ADX > 40: Very strong trend
- **Buy/Sell signals**:
  - Buy when +DI crosses above -DI, confirmed by ADX > 25
  - Sell when -DI crosses above +DI, confirmed by ADX > 25
- **Critical role**: ADX is the tool for distinguishing trending from ranging markets. Use it as a filter for all other indicators.

---

### Indicator Classification by Market Regime

| Regime | Best Indicators |
|--------|----------------|
| **Trending** | MA crossovers, MACD, Parabolic SAR, Ichimoku Cloud, Keltner Channels, Donchian Channels, ADX/DMI |
| **Ranging** | RSI, Stochastic, Williams %R, Bollinger Bands (mean reversion), CCI |
| **Both** | MFI, OBV, A/D Line, ATR, VWAP, CMF |

---

## 2. Backtesting Methodology

### 2.1 Walk-Forward Analysis (Recommended Primary Method)

Walk-Forward Analysis (WFA) divides historical data into sequential in-sample (IS) and out-of-sample (OOS) segments. The strategy is optimized on IS data, then tested on the next OOS segment. The window slides forward and repeats.

**Two variants:**
- **Anchored WFA**: IS window always starts from the beginning and grows. Better for longer-term strategies.
- **Unanchored (Rolling) WFA**: Fixed-size IS window slides forward. Faster adaptation. More suitable for shorter-term strategies.

**Recommended IS/OOS ratio**: 70-80% in-sample, 20-30% out-of-sample.

### 2.2 Time Series Cross-Validation

Standard K-Fold CV is dangerous for financial time series because it breaks chronological ordering.

**Appropriate methods:**
1. **Time Series Split (Expanding Window CV)**: Each fold uses all prior data as training and the next period as test.
2. **Rolling Window CV**: Fixed-size training window slides forward.
3. **Combinatorial Purged Cross-Validation (CPCV)** (Lopez de Prado): State-of-the-art method with purging (removes overlapping labels) and embargoing (removes auto-correlated observations near test boundaries).

### 2.3 Avoiding Bias

- **Look-Ahead Bias**: Never use information unavailable at decision time. Use only data up to and including the current bar.
- **Survivorship Bias**: Use datasets that include delisted/failed assets. Asset universe at each point should reflect what was actually available.

### 2.4 Statistical Significance

- **Monte Carlo Simulation**: Reshuffle actual trades across 1,000+ random scenarios to test robustness.
- **Minimum sample size**: 30 trades absolute minimum; 200-300 recommended for reliable inference.
- **Deflated Sharpe Ratio**: Adjusts for multiple testing bias.

---

## 3. Profitability Metrics

### Core Metrics (with formulas)

| Metric | Formula | Interpretation |
|--------|---------|---------------|
| **Total Return** | `(Final - Initial) / Initial * 100%` | Overall performance |
| **Annualized Return (CAGR)** | `((Final/Initial) ^ (1/Years)) - 1` | Yearly equivalent |
| **Sharpe Ratio** | `(Rp - Rf) / sigma_p` | >1.0 acceptable, >2.0 very good, >3.0 excellent |
| **Sortino Ratio** | `(Rp - MAR) / sigma_d` | Like Sharpe but only penalizes downside volatility |
| **Calmar Ratio** | `Annualized Return / |Max Drawdown|` | Return per unit of worst drawdown |
| **Max Drawdown** | `(Trough - Peak) / Peak * 100%` | Worst peak-to-trough decline |
| **Win Rate** | `Winning Trades / Total Trades * 100%` | Alone insufficient; 30% WR can be profitable |
| **Profit Factor** | `Gross Profit / Gross Loss` | >1.0 profitable, >1.5 good, >2.0 very strong |
| **Expectancy** | `(Win% * Avg Win) - (Loss% * Avg Loss)` | Average expected profit per trade |

### Annualized Sharpe from Daily Data
```
Annualized Sharpe = (Mean Daily Return - Daily Risk-Free Rate) / Std Dev of Daily Returns * sqrt(252)
```

### Sortino Downside Deviation
```
sigma_d = sqrt(sum(min(Ri - MAR, 0)^2) / N)
```
Only periods where return < MAR contribute.

---

## 4. Trade-Level Reporting

Each trade record should capture:

| Field | Description |
|-------|-------------|
| Entry Date | When the entry signal was generated |
| Entry Price | Actual or simulated execution price |
| Entry Signal | Which indicator/condition triggered (e.g., "RSI crossed below 30") |
| Exit Date | When the exit signal was generated |
| Exit Price | Actual or simulated execution price |
| Exit Signal | What triggered exit (take-profit, stop-loss, indicator) |
| Direction | Long or Short |
| Position Size | Number of units/shares |
| P/L ($) | Dollar profit/loss after costs |
| P/L (%) | Percentage profit/loss after costs |
| Duration | Trading days held |
| MAE | Maximum Adverse Excursion (worst unrealized loss) |
| MFE | Maximum Favorable Excursion (best unrealized profit) |

**Per-trade P/L:**
- Long: `(Exit Price - Entry Price) * Size - Costs`
- Short: `(Entry Price - Exit Price) * Size - Costs`

---

## 5. Summary Statistics

The application should report:

| Statistic | Formula |
|-----------|---------|
| **Avg Trade Length** | `sum(Duration_i) / N` (in trading and calendar days) |
| **Avg Profitability** | `Total Net Profit / N` (dollar and percentage) |
| **Avg Time Between Trades** | `sum(Entry_i+1 - Exit_i) / (N-1)` |
| **Win/Loss Ratio** | `Avg Winning Trade / Avg Losing Trade` |
| **Max Consecutive Wins** | Longest streak of winners |
| **Max Consecutive Losses** | Longest streak of losers |
| **Max Favorable Excursion (MFE)** | Best unrealized profit before close |
| **Max Adverse Excursion (MAE)** | Worst unrealized loss before close |

MAE/MFE analysis enables optimal stop-loss and take-profit placement.

---

## 6. Indicator Ranking and "Best Fit" Determination

### Priority Hierarchy for Ranking

1. **Risk-adjusted return (Sharpe or Sortino)**: Primary criterion -- balances return against risk
2. **Maximum drawdown**: Critical constraint; excessive drawdowns should eliminate candidates
3. **Profit factor**: Quick measure of edge quality
4. **Expectancy**: Average expected profit per trade; ensures statistical viability
5. **Win rate + avg win/loss ratio**: Together determine long-term viability
6. **Number of trades**: Sufficient sample size (min 30, ideally 200+)
7. **Consistency**: Low variance across different time periods

### Composite Scoring

```
Score = w1*Norm_Sharpe + w2*Norm_ProfitFactor + w3*Norm_MaxDD + w4*Norm_Expectancy + w5*Norm_WinRate
```

Suggested weights: Sharpe 30%, MaxDD 25%, Profit Factor 20%, Expectancy 15%, Win Rate 10%.

Normalize metrics via min-max scaling or z-score. Use Pareto optimality for multi-objective comparison.

### Determining Best Fit

- Test each indicator via walk-forward analysis on the specific asset
- Compare performance across market regimes (trending, ranging, volatile, calm)
- Prefer indicators performing consistently across regimes over those excelling in only one
- Use regime detection (ADX, ATR) to match indicators to conditions
- Choose indicators providing uncorrelated signals for combination strategies

---

## 7. Indicator Combination Strategies

### Multi-Indicator Architecture

The most effective systems combine three dimensions:
- **Trend Indicators** (direction): EMA, SMA, ADX
- **Momentum Indicators** (speed/strength): RSI, MACD, Stochastic
- **Volatility Indicators** (range/risk): Bollinger Bands, ATR, Keltner Channels

### Signal Scoring Systems

- **Simple scoring**: +1 bullish, -1 bearish per indicator. Sum for composite.
- **Normalized composite**: Map all indicator outputs to [-1.0, +1.0] range.
- **Weighted scoring**: Weight by historical accuracy of each indicator.
- **Consensus/voting**: Trade only when >50% of indicators agree.

### Hierarchical Filtering (Recommended)

Three-layer structure:
1. **Layer 1 - Trend Recognition**: EMA fast/slow determines primary trend direction. Only trade in this direction.
2. **Layer 2 - Trend Confirmation**: SMA or price-above-MA filter ensures alignment with trend.
3. **Layer 3 - Entry Signal**: MACD/RSI confirms actual entry with momentum.

Signals only generated when ALL filter conditions met simultaneously. Prioritizes signal quality over frequency. Studies show ~23% improvement vs single-indicator approaches.

### Signal Confirmation Techniques

- **Cross-indicator confirmation**: Never trade unless trend + momentum align. MACD+RSI together reduces false signals by 65% vs MACD alone.
- **Volume confirmation**: Golden Cross with 40%+ volume increase showed 72% accuracy vs 54% for low-volume crossovers.
- **Divergence analysis**: When both RSI and MACD show divergence around the same swing, confidence increases substantially.

---

## 8. Proven Indicator Combinations

### MACD + RSI
- **Long**: RSI < 30 (oversold) + MACD bullish crossover + price above support
- **Short**: RSI > 70 (overbought) + MACD bearish crossover
- **Exit Long**: RSI approaches 70 AND MACD flattens/crosses below signal
- **Performance**: 65-73% win rate in trending; 45-55% in sideways. Daily/weekly most consistent.

### Moving Average Crossover + Volume
- **Golden Cross**: 50-MA crosses above 200-MA
- **Confirmation**: Both MAs trending upward, price above both, volume +40%, RSI > 50
- **Exit**: Opposite crossover or ATR-based trailing stop
- **Limitation**: Lagging; signals come AFTER trend begins

### Bollinger Bands + RSI
- **Buy**: Price at/below lower band AND RSI < 30
- **Sell**: Price at/above upper band AND RSI > 70
- **Squeeze**: Prepare for breakout but wait for RSI directional confirmation

### ADX + DMI
- **Prerequisite**: ADX > 25 (strong trend confirmed)
- **Buy**: +DI crosses above -DI while ADX > 25
- **Sell**: -DI crosses above +DI while ADX > 25
- **Stop**: One tick below signal day low (long) / above signal day high (short)

### Elder Triple Screen Trading System
1. **Screen 1 (Tide)**: Weekly MACD/MA identifies primary trend. Only trade in this direction.
2. **Screen 2 (Wave)**: Daily oscillators find pullbacks within trend.
3. **Screen 3 (Ripple)**: Precise entry technique (support/resistance, trailing stops).

---

## 9. Adaptive Indicator Selection and Market Regime Detection

### Market Regime Detection

**ADX-based (Simple, Recommended)**:
- ADX > 25: Trending -- use trend-following indicators
- ADX < 20: Ranging -- use mean-reversion indicators
- Bollinger Band Width: Narrow = low vol/ranging; Wide = high vol/trending

**Hidden Markov Models (Advanced)**:
- Use `hmmlearn` Python library with `GaussianHMM`
- Trained on daily returns to identify hidden states (typically 2-4)
- States map to: Strong Bull, Weak Bull, Sideways, Bear
- Outperformed other approaches in comparative studies

**Machine Learning**:
- K-Means clustering on volatility, trend strength, momentum features
- Random Forest classifier trained on market breadth features

### Dynamic Indicator Switching

- Trending regime detected: Activate MACD, MA crossovers, ADX strategies
- Ranging regime detected: Activate RSI, Stochastic, Bollinger mean-reversion

---

## 10. Pattern Identification

### Chart Patterns (Algorithmically Detectable)

**Reversal**: Head & Shoulders, Double/Triple Top, Double/Triple Bottom
**Continuation**: Ascending/Descending Triangles, Flags, Pennants, Wedges

**Detection approach**: Use rolling window to find local extrema via `scipy.signal.argrelextrema()`, then check mathematical relationships. Noise reduction via Savitzky-Golay or Kalman filters.

### Candlestick Patterns

TA-Lib supports 60+ patterns including:
- **Single**: Doji, Hammer, Inverted Hammer, Hanging Man, Shooting Star, Marubozu
- **Double**: Engulfing, Harami, Piercing Line, Dark Cloud Cover
- **Triple**: Morning Star, Evening Star, Three White Soldiers, Three Black Crows

### Window Size Effects (Default 30 Days)

- **10-20 days**: More sensitive, higher false positives, better for swing trading
- **30 days (default)**: Good balance for daily data, captures most common patterns
- **50-90 days**: Captures major patterns, fewer false positives, misses shorter setups
- Window should be configurable per asset -- different assets have different cycle lengths

---

## 11. Open-Source Libraries and Tools

| Library | Type | Strengths | Weaknesses | Status |
|---------|------|-----------|------------|--------|
| **TA-Lib** | Indicator computation | 150+ indicators, C speed, battle-tested | C dependency complicates install | Active (2025) |
| **pandas-ta** | Indicator computation | Pythonic, pandas-native, 150+ indicators | Slower than TA-Lib | At risk (archive by July 2026) |
| **vectorbt** | Backtesting | Fastest Python backtester, NumPy/Numba | Steep learning curve, PRO is paid | Active |
| **backtrader** | Backtesting | Simple strategy dev, broker integration | Unmaintained since 2018 | Abandoned |
| **zipline-reloaded** | Backtesting | Event-driven, Quantopian pedigree | Tricky install, small community | Maintained |
| **yfinance** | Data sourcing | Free, Python-native, OHLCV | Personal use only, scraping-based | Active |
| **mplfinance** | Visualization | Candlestick/OHLC charts | Plotting only | Active |
| **PatternPy** | Pattern recognition | H&S, Tops/Bottoms, S/R detection | Limited pattern set | Active |
| **hmmlearn** | Regime detection | HMM for market state identification | Requires ML knowledge | Active |

### Recommended Stack

1. **TA-Lib** for indicator computation (speed + reliability)
2. **vectorbt** for backtesting (speed + flexibility)
3. **yfinance** or CSV for data sourcing
4. **PatternPy** + custom scipy-based detection for chart patterns
5. **hmmlearn** for advanced regime detection (optional)

---

## 12. Data Format and Preprocessing

### OHLCV Format

- **O**pen, **H**igh, **L**ow, **C**lose, **V**olume
- Standard format for all TA libraries
- Pandas DataFrame with DatetimeIndex is the universal Python representation

### Data Cleaning

1. Missing data: Forward-fill or interpolation for isolated gaps; investigate unexpected gaps
2. Zero volume: Flag and investigate (may indicate data error or illiquidity)
3. Outliers: Verify against actual market events
4. Validation: Ensure High >= Open/Close >= Low for each bar
5. Adjusted Close: Use for all indicator calculations when dividends/splits are a factor
6. **Critical**: Do NOT add dividends if using Adjusted Close (double-counting)

---

## 13. Common Pitfalls

### Overfitting / Curve Fitting
- Limit strategies to 3-5 core parameters
- If IS-to-OOS performance degrades significantly, overfitting is likely
- Mitigation: Walk-forward analysis, OOS testing, Monte Carlo simulation

### Transaction Costs and Slippage
- Model commissions, spreads, and exchange fees
- Slippage models: Fixed percentage (0.1%), dynamic (volatility-based), square-root market impact
- Realistic slippage can trim simulated returns by 0.5-3% annually

### Data Quality
- Missing data, outlier spikes, improper split/dividend adjustments
- Use point-in-time data where possible

---

## 14. Optimization Approaches

| Method | Description | Best For |
|--------|-------------|----------|
| **Grid Search** | Exhaustive parameter sweep | Small parameter spaces (< 5 params) |
| **Genetic Algorithm** | Evolutionary optimization | Large, complex spaces |
| **Bayesian Optimization** | Probabilistic surrogate model | Expensive evaluations |
| **Walk-Forward Optimization** | Combine WFA + parameter optimization | Production-grade adaptive systems |

**Walk-Forward Efficiency**: Ratio of OOS performance to IS performance. Values near 1.0 indicate robust parameters.

### Configurable Lookback Windows

- Default 30 days for daily data
- Optimal sizes vary by indicator (RSI: 14, MACD: 12/26/9, Bollinger: 20, ATR: 14)
- **Rule of thumb**: Window = half the cycle length you're trying to capture
- Make window size an optimizable parameter
- Consider adaptive windows that adjust based on volatility

---

## 15. Academic Research Summary

### Support for Technical Analysis
- From 1988-2004, 26/38 studies found positive results. Of 95 modern studies, 56 found positive results.
- TA more profitable in futures and forex than stocks
- SMA indicators performed best in emerging markets; RSI better for developed markets
- ML-based TA approaches capture effects beyond what traditional EMH predicts

### Limitations
- Results "very sensitive to the introduction of moderate transaction costs" (2023 study of 6,000+ rules)
- Recently best-performing rules "perform significantly worse than buy-and-hold in the future"
- Traditional indicators alone have limited predictive power

### Bottom Line
- Strongest empirical support for combining indicators with quantitative/ML models
- Multi-indicator strategies consistently outperform single-indicator approaches
- No single indicator is sufficient alone
- Behavioral finance confirms exploitable patterns exist contrary to pure EMH

---

## 16. Design Recommendations

Based on all research findings, the application should:

1. **Implement Walk-Forward Analysis** as the primary backtesting method (anchored and rolling options)
2. **Compute all core metrics** for every indicator test: CAGR, Sharpe, Sortino, Calmar, MaxDD, Win Rate, Profit Factor, Expectancy, Avg Trade Duration, Avg Time Between Trades
3. **Trade log format**: Entry/exit date, price, signal, direction, P/L ($/%),duration, MAE, MFE
4. **Weighted composite scoring** for indicator ranking with configurable weights
5. **Statistical validation**: Min 30 trades (ideally 200+), Monte Carlo (1,000+ iterations), p-value reporting
6. **Transaction cost modeling**: Configurable costs with default 0.1% round-trip
7. **Configurable lookback windows**: Default 30 days, include as optimizable parameter
8. **Hierarchical filtering architecture**: Trend filter (Layer 1) -> Confirmation (Layer 2) -> Entry signal (Layer 3)
9. **ADX-based regime detection**: Dynamically select indicator set based on market conditions
10. **Guard against overfitting**: Limit to 3-5 parameters, OOS validation, walk-forward efficiency ratio
11. **Modular indicator engine**: Common interface normalizing outputs to standard scale, easy to add/remove indicators
12. **Pattern recognition**: TA-Lib for candlestick patterns, scipy-based chart pattern detection with configurable windows
