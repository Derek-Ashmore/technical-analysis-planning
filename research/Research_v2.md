# Technical Analysis Research Report v2

## Executive Summary

This is a revised research report incorporating the confirmed application requirements. The key constraints now driving the design are:

- **Equities only** (stocks, ETFs) -- no forex, commodities, or crypto
- **Long-only positions** -- buy and sell-to-close only, no short selling
- **CSV data with volume** -- Nasdaq-format CSV (Date, Close/Last, Volume, Open, High, Low)
- **Single asset at a time** -- no portfolio-level analysis
- **Daily batch processing** -- run once per day to incorporate new data
- **Benchmark against buy-and-hold and S&P 500**
- **Configurable parameters** -- trade frequency (default 30 days), max drawdown (default 30%), max indicator combinations (default 5)
- **Future extensibility** -- options analysis, API data source

**Key changes from v1**: All short-selling signals are reinterpreted as exit signals. Williams %R is added as a top indicator based on backtest evidence. The backtesting stack recommendation changes from vectorbt to backtesting.py (walk-forward support, active maintenance). ADX regime detection defaults to cash in ranging markets rather than mean-reversion shorts. Composite scoring weights are revised to increase max drawdown and add Calmar Ratio. A dual-price approach is recommended (split-adjusted for indicators, fully adjusted for returns).

---

## Table of Contents

1. [Indicator Recommendations for Equities (Long-Only)](#1-indicator-recommendations-for-equities-long-only)
2. [Signal Rules for Long-Only Trading](#2-signal-rules-for-long-only-trading)
3. [Exit Strategies](#3-exit-strategies)
4. [Indicator Combinations](#4-indicator-combinations)
5. [ADX Regime Detection for Long-Only](#5-adx-regime-detection-for-long-only)
6. [Composite Scoring (Revised)](#6-composite-scoring-revised)
7. [Backtesting Methodology](#7-backtesting-methodology)
8. [Benchmarking](#8-benchmarking)
9. [Data Handling](#9-data-handling)
10. [Pattern Recognition](#10-pattern-recognition)
11. [Application Architecture](#11-application-architecture)
12. [Library and Tooling Recommendations](#12-library-and-tooling-recommendations)
13. [Anti-Overfitting Measures](#13-anti-overfitting-measures)
14. [Equity-Specific Considerations](#14-equity-specific-considerations)
15. [Design Recommendations (Revised)](#15-design-recommendations-revised)

---

## 1. Indicator Recommendations for Equities (Long-Only)

### Primary Indicators (Recommended for Implementation)

#### Williams %R -- NEW in v2

- **Why added**: Backtests show it is the single best swing trading indicator for equities, with the lowest max drawdown and most stable results across parameter sets.
- **Performance on S&P 500 (SPY)**: 2-day lookback achieved 81% win rate and 11.9% CAGR vs 7.3% for RSI (source: QuantifiedStrategies.com).
- **Profit factor**: Above 2.0 for nearly all parameter sets tested.
- **Parameters for daily data**: 14-period for trend assessment, 2-period for entry timing.
- **Long-only signal**: Buy when Williams %R(2) < -90; exit when close > yesterday's high OR Williams %R > -30.

#### RSI (Relative Strength Index)

- **Retained from v1**. Academic studies show 81% directional prediction accuracy on equities (source: UniversePG 2024 study).
- **Parameters**: 14 periods (standard). Use 30/70 thresholds.
- **Long-only adjustment**: RSI > 70 is NOT a short signal. Instead: wait for RSI to cross BACK BELOW 70 to confirm momentum shift before exiting. Stocks can remain overbought for extended periods in strong uptrends.
- **RSI 40-50 zone**: In strong uptrends, pullbacks to this zone are buy-the-dip opportunities.

#### MACD (Moving Average Convergence Divergence)

- **Retained from v1**. 78% average prediction accuracy across multiple equity markets (source: UniversePG 2024).
- **Parameters**: 12/26/9 (standard). Works well on daily equity charts.
- **Long-only adjustment**: Bearish crossover = exit existing long (not "enter short").
- **Histogram turning negative**: Early warning to tighten stops.

#### Moving Averages (SMA/EMA)

- **Retained from v1**. The most reliable indicators for long-term equity investors.
- **Parameters**: 20 EMA (short-term), 50 SMA (medium-term), 200 SMA (long-term).
- **Crossover strategies**: 20/50 for swing trading, 50/200 (Golden Cross/Death Cross) for position trading.
- **Evidence**: EMA crossover strategies beat buy-and-hold in 61% of tested stocks (source: SAGE Journals 2025).

#### Bollinger Bands

- **Retained from v1**. Standard mean-reversion tool for ranging markets.
- **Parameters**: 20-period SMA, 2 standard deviations.
- **Long-only adjustment**: Upper band touch is NOT a short signal. Instead: tighten trailing stop or take partial profits.
- **Squeeze**: Only act on upside breakouts for long-only.

#### ADX/DMI (Average Directional Index)

- **Retained from v1**. Critical regime detector.
- **Parameters**: 14 periods. Thresholds: < 20 (ranging), > 25 (trending), > 40 (strong trend).
- **Long-only role**: Determines whether to use trend-following signals, mean-reversion entries, or go to cash.

#### OBV (On-Balance Volume)

- **Elevated in v2** (was present but not emphasized in v1). Now a mandatory volume confirmation indicator since volume data is available.
- **Divergences**: Price makes new high but OBV does not = weakening rally (exit warning).
- **OBV breaking to new highs before price**: Leading buy signal (institutional accumulation).
- **Volume confirmation**: Golden Cross with 40%+ volume increase showed 72% accuracy vs 54% without (source: v1 research).

#### ATR (Average True Range)

- **Retained from v1**. Essential for stop-loss placement and position sizing.
- **Parameters**: 14-period (standard), 22-period for Chandelier Exit.
- **Usage**: Stop placement at 2-3x ATR from entry. Chandelier Exit uses ATR(22) * 3.0.

### Secondary Indicators (Available but Lower Priority)

| Indicator | Parameters | Role | Notes |
|-----------|-----------|------|-------|
| Stochastic (14/3/3) | Slowed for position trading | Entry timing | Default 5/3/3 too fast for 30-day holding |
| CCI (20) | Standard | Overbought/oversold | Useful as secondary confirmation |
| MFI (14) | Standard | Volume-weighted RSI | Complements RSI with volume data |
| CMF (20) | Standard | Money flow direction | Good for trend confirmation |
| A/D Line | Cumulative | Accumulation/distribution | Divergences signal reversals |

### Indicators Excluded or Deprioritized

| Indicator | Reason |
|-----------|--------|
| **VWAP** | Primarily intraday; resets daily. Not useful for 30-day swing/position trading. |
| **Parabolic SAR** | Only 19% win rate on DJIA 30 stocks over 12 years (source: PatternsWizard). Excessive whipsaws in ranging markets. Only usable with ADX > 25 filter. |
| **Ichimoku Cloud** | Overly complex for the value it adds. The individual components (MA crossovers, cloud support/resistance) are better served by simpler indicators. |
| **Donchian Channels** | Redundant with Bollinger Bands and Keltner Channels for this use case. |

---

## 2. Signal Rules for Long-Only Trading

### Fundamental Principle

All signals are reinterpreted for long-only:
- **Bullish signals** → BUY (enter long position)
- **Bearish signals** → SELL-TO-CLOSE (exit existing long position) or GO TO CASH
- **No signal is ever interpreted as "enter short"**

### RSI Long-Only Rules

| Condition | Action |
|-----------|--------|
| RSI crosses above 30 from below | BUY -- oversold reversal |
| RSI in 40-50 zone in uptrend | BUY -- dip in uptrend |
| RSI crosses above 50 | Trend confirmation (hold/add) |
| RSI crosses below 70 from above | Consider EXIT -- momentum fading |
| Bullish divergence (price lower low, RSI higher low) | BUY signal |

**Important**: Do NOT sell immediately when RSI reaches 70. Wait for RSI to cross BACK BELOW 70. Equities can remain overbought for extended periods during strong uptrends.

### MACD Long-Only Rules

| Condition | Action |
|-----------|--------|
| MACD crosses above Signal Line | BUY |
| MACD crosses above zero line | Strong trend confirmation (hold/add) |
| MACD histogram growing | Strengthening momentum (hold) |
| MACD crosses below Signal Line | EXIT existing long |
| MACD histogram shrinking | Tighten trailing stop |

### Bollinger Bands Long-Only Rules

| Condition | Action |
|-----------|--------|
| Price at/below lower band + RSI < 30 | BUY (mean reversion entry) |
| Price returns to middle band from below | Take partial profits or hold |
| Price at/above upper band | Tighten trailing stop (NOT short) |
| Bollinger Squeeze (bands narrowing) | Prepare for breakout; only act on UPSIDE |

### Combined RSI + MACD Entry (Long-Only)

- **Entry**: RSI crosses above 50 AND MACD is bullish (MACD > Signal), OR MACD bullish crossover while RSI is at/above 50.
- **Exit**: RSI crosses above 65 (custom threshold) AND MACD bearish crossover.
- **Performance**: 55-73% win rate (65-73% in trending markets). Source: QuantifiedStrategies.com.

---

## 3. Exit Strategies

### Recommended Primary Exit: Chandelier Exit (ATR-Based Trailing Stop)

- **Formula**: `Highest High(22) - ATR(22) * 3.0`
- **How it works**: Dynamically adapts to volatility. As price rises, the stop rises. As volatility increases, the stop widens.
- **For volatile stocks (e.g., NVDA)**: Increase multiplier to 4-5 to avoid premature stop-outs.
- **Developed by**: Charles Le Beau. Widely used by equity position traders.
- **Source**: StockCharts, QuantifiedStrategies.com.

### Alternative Exits (Use in Combination)

| Exit Method | Rule | Best For |
|-------------|------|----------|
| ATR trailing stop | ATR(14) * 2-3 from highest high | Swing trades |
| RSI overbought reversal | RSI crosses back below 70 | Mean-reversion exits |
| MACD bearish crossover | MACD < Signal Line | Trend-following exits |
| MA close | Price closes below 20 EMA (swing) or 50 SMA (position) | Trend change confirmation |
| Max drawdown stop | Position loss exceeds X% of portfolio | Capital preservation |

### Exit Methods to Avoid

- **Parabolic SAR**: Only 19% win rate in backtests. Excessive whipsaws.
- **Fixed percentage stops**: Don't adapt to volatility. A 5% stop on a volatile stock triggers too often.
- **Time-based exits alone**: Exiting after N days regardless of price action leaves money on the table in strong trends.

---

## 4. Indicator Combinations

### Top 5 Recommended Combinations for Long-Only Equity Swing/Position Trading

#### Combination 1: ADX Trend-Following

- **Regime**: ADX > 25 (trending market)
- **Layer 1 (Trend)**: Price above 50 and 200 SMA
- **Layer 2 (Confirmation)**: ADX > 25
- **Layer 3 (Entry)**: MACD bullish crossover + RSI above 50
- **Layer 4 (Exit)**: Chandelier Exit (ATR-based trailing stop)
- **Expected**: Higher returns, win rate 55-65%, larger average win

#### Combination 2: Mean-Reversion Entry

- **Regime**: ADX < 20 (ranging market) -- only if mean-reversion mode enabled
- **Layer 1 (Regime)**: ADX < 20
- **Layer 2 (Entry)**: RSI < 30 AND price at/below lower Bollinger Band
- **Layer 3 (Confirmation)**: OBV not declining (volume not confirming breakdown)
- **Layer 4 (Exit)**: Price returns to 20-period SMA (Bollinger middle band)
- **Expected**: Higher win rate 65-75%, smaller average gain

#### Combination 3: Williams %R Mean Reversion

- **Entry**: Williams %R(2) < -90
- **Exit**: Today's close > yesterday's high OR Williams %R > -30
- **Filter**: Only when 200 SMA is rising (long-term uptrend)
- **Expected**: 81% win rate, 11.9% CAGR on S&P 500 (source: QuantifiedStrategies)

#### Combination 4: RSI + MACD Dual Confirmation

- **Entry**: RSI crosses above 50 + MACD bullish crossover simultaneously
- **Exit**: RSI > 65 + MACD bearish crossover
- **Performance**: 55-73% win rate depending on regime

#### Combination 5: Volume-Confirmed Breakout

- **Layer 1 (Trend)**: Price above 50 SMA
- **Layer 2 (Setup)**: Price consolidating near resistance
- **Layer 3 (Entry)**: Breakout above resistance + volume > 1.5x 20-day average
- **Layer 4 (Confirmation)**: OBV making new highs
- **Layer 5 (Exit)**: Chandelier Exit or price closes below 20 EMA

### Combination Design Principles

1. Always include a **trend filter** (50 or 200 SMA) as foundation
2. Add a **momentum oscillator** (RSI or Williams %R) for entry timing
3. Use **volume confirmation** (OBV) for signal quality
4. Apply a **volatility-based exit** (Chandelier/ATR trailing stop) instead of fixed stops
5. Each combination should use indicators from **different categories** (trend + momentum + volume + volatility) to avoid redundancy

---

## 5. ADX Regime Detection for Long-Only

### Regime Classification

| ADX Level | Regime | Recommended Action (Long-Only) |
|-----------|--------|-------------------------------|
| ADX > 40 | Strong trend | Full position, wider trailing stops (higher ATR multiplier) |
| ADX 25-40 | Trending | Use trend-following signals, standard stops |
| ADX 20-25 | Transitional | Reduce position size by 50%, tighter stops |
| ADX < 20 | Ranging | **Default: Go to cash** (configurable) |

### Why Default to Cash in Ranging Markets

1. **30% max drawdown constraint** favors capital preservation
2. Markets only trend ~30% of the time; the other 70% is mostly noise
3. Mean-reversion in ranging markets works but requires more active management and more trades
4. For a 30-day default trading frequency, being out of market during ranging periods is acceptable
5. A long-only strategy that sits in cash during uncertain periods still benefits from equities' structural long bias when it re-enters

### Configurable Mean-Reversion Mode

- Users who want more active trading can enable mean-reversion mode for ranging markets (ADX < 20)
- When enabled: use RSI + Bollinger Bands to buy dips toward support and sell at resistance
- Mean reversion has higher win rates (65-75%) but smaller average gains per trade
- This should be a configuration option, not the default

---

## 6. Composite Scoring (Revised)

### Changes from v1

The scoring weights are revised to reflect long-only trading with a 30% max drawdown constraint:

| Metric | v1 Weight | v2 Weight | Change Rationale |
|--------|-----------|-----------|-----------------|
| Max Drawdown / CDaR | 25% | **30%** | Hard constraint at 30%; most critical for capital preservation |
| Sharpe Ratio | 30% | **25%** | Still important but slightly reduced since DD is the binding constraint |
| Profit Factor | 20% | **15%** | Reduced slightly to accommodate Calmar |
| Win Rate | 10% | **15%** | More important for long-only mean-reversion strategies |
| Calmar Ratio | -- | **10%** | NEW -- directly measures return per unit of drawdown |
| Expectancy | 15% | **5%** | Reduced; covered by other metrics |

### Composite Score Formula

```
Score = 0.30 * Norm_MaxDD + 0.25 * Norm_Sharpe + 0.15 * Norm_ProfitFactor
      + 0.15 * Norm_WinRate + 0.10 * Norm_Calmar + 0.05 * Norm_Expectancy
```

### Pre-Scoring Filter

Before computing composite scores, **automatically disqualify** any strategy with:
- Historical max drawdown > 30% (configurable threshold)
- Fewer than 30 closed trades (insufficient statistical significance)
- Walk-Forward Efficiency < 50%
- Deflated Sharpe Ratio not statistically significant

---

## 7. Backtesting Methodology

### Walk-Forward Analysis Structure

**Window sizing for ~10 years (2500 bars) of daily data:**
- **In-sample (training) window**: 3 years (~750 bars)
- **Out-of-sample (validation) window**: 6 months (~125 bars)
- **Rolling forward**: Advance by 6 months each step
- **Total windows**: ~14 walk-forward windows over 10 years

**Walk-Forward Efficiency (WFE)**: Ratio of annualized OOS returns to IS returns. WFE > 50-60% indicates the strategy is not overfitted.

### Two-Phase Architecture

1. **Daily run (lightweight)**: Load data → append new bar → recalculate indicators with current parameters → generate signal → output recommendation
2. **Periodic optimization (heavyweight)**: Full walk-forward analysis → test all combinations → select best parameters → save for daily use

**Re-optimization frequency**: Monthly (default, configurable). Do NOT re-optimize daily -- it's unnecessary and increases overfitting risk.

### Return Calculation During Flat Periods

- **Default**: Cash earns the risk-free rate (3-month T-bill rate from FRED)
- **Configurable**: User can set cash return to 0% if preferred
- **Rationale**: A long-only strategy that sits in cash during bear markets has a real advantage in risk-adjusted terms
- **Report "time in market"** alongside returns to make this visible

### Statistical Significance Requirements

| Requirement | Threshold | Notes |
|-------------|-----------|-------|
| Minimum closed trades | 30 | Absolute minimum (Central Limit Theorem) |
| Recommended trades | 80+ | For robust conclusions |
| Walk-forward windows | 10-15 | Good balance of granularity and statistical power |
| t-test on trade returns | p < 0.05 | Mean return significantly different from zero |
| Monte Carlo iterations | 1,000+ | Shuffle trade returns to assess chance probability |

---

## 8. Benchmarking

### Buy-and-Hold Benchmark

Compare strategy performance against simply buying the equity on day one and holding through the entire period.

**Metrics to compare (side-by-side):**
- Total cumulative return
- Annualized return (CAGR)
- Sharpe Ratio
- Sortino Ratio (better for long-only -- only penalizes downside volatility)
- Calmar Ratio
- Maximum drawdown
- Time in market (strategy will be < 100%)
- Win rate and profit factor (strategy only)

### S&P 500 Benchmark

**Data source**: SPY ETF adjusted close data from yfinance.
- SPY is preferred over ^GSPC because it is investable and includes dividends.
- Use adjusted close from yfinance to account for SPY's own dividends.
- Fall back to ^GSPC for periods before SPY inception (Jan 1993).

**Additional metrics vs S&P 500:**
- **Alpha**: Strategy excess return over benchmark (Jensen's alpha)
- **Beta**: Strategy correlation/sensitivity to market moves
- **Information Ratio**: (Strategy Return - Benchmark Return) / Tracking Error
- **Active Return**: Simple difference in returns

**Implementation**: Download SPY data via yfinance on the same date range as the user's CSV. Align dates (handle market holidays). Calculate all metrics on matched date ranges.

### Benchmark Presentation

Include benchmark rows in the same ranking table as indicator combinations for direct comparison:

```
+------+-------------------------+--------+--------+--------+--------+
| Rank | Strategy                | CAGR   | Sharpe | MaxDD  | Score  |
+------+-------------------------+--------+--------+--------+--------+
|  1   | EMA20+RSI14+BB20        | 18.2%  | 1.85   | -22%   | 0.903  |
|  2   | MACD+ADX+WilliamsR      | 15.7%  | 1.62   | -18%   | 0.860  |
+------+-------------------------+--------+--------+--------+--------+
| B&H  | Buy and Hold (NVDA)     | 12.1%  | 0.95   | -45%   | 0.640  |
| SPY  | S&P 500 (SPY)           | 10.3%  | 0.88   | -34%   | 0.620  |
+------+-------------------------+--------+--------+--------+--------+
```

---

## 9. Data Handling

### CSV Format and Parsing

The Nasdaq-format CSV has these characteristics:
- **Column order**: Date, Close/Last, Volume, Open, High, Low
- **Date format**: MM/DD/YYYY
- **Price format**: Prefixed with `$` (e.g., `$185.41`)
- **Volume**: Plain integer
- **Sort order**: Reverse chronological (newest first) -- must sort ascending
- **Column name**: `Close/Last` contains a slash -- needs explicit rename

**Parsing pipeline**:
1. Read CSV with pandas, parse dates with `%m/%d/%Y` format
2. Strip `$` prefix from price columns
3. Rename columns to standard lowercase: `date, open, high, low, close, volume`
4. Sort chronologically (ascending by date)
5. Set date as index

### Dual-Price Approach

| Purpose | Price to Use | Rationale |
|---------|-------------|-----------|
| Indicator calculation (SMA, RSI, MACD, etc.) | Split-adjusted raw close | Indicators should reflect actual trading activity; dividend adjustments create false signals |
| Return/PnL calculation | Fully adjusted close (split + dividend) | Accurate total return comparison with benchmarks |
| Benchmarking | Adjusted close (both strategy and benchmark) | Fair comparison requires consistent adjustment |

### Stock Split Handling

- **Critical for NVDA**: 10:1 split (June 2024), 4:1 split (July 2021), plus older splits
- CSV data from Nasdaq is NOT split-adjusted
- **Must apply split adjustment factors retroactively** to all prices before each split date
- Split factors compound: pre-2021 NVDA prices need both the 4:1 and 10:1 adjustments
- Volume must be adjusted inversely (multiply by split ratio)
- **Source for split data**: yfinance `stock.splits` history

### Dividend Handling

- For signal generation: do NOT adjust for dividends
- For return calculation: include dividend payments as additional return on ex-date
- NVDA pays small quarterly dividends (~$0.01/share quarterly), impact is minimal
- Source: yfinance `stock.dividends`

### Data Validation

1. Ensure High >= Open, Close >= Low for each bar
2. Check for missing trading days (gaps beyond weekends/holidays)
3. Flag zero-volume days for investigation
4. Detect potential unadjusted splits (sudden 2x, 4x, 10x price jumps)
5. Warn if data has fewer than 200 bars (insufficient for 200-day SMA)

---

## 10. Pattern Recognition

### Bullish Patterns (Long Entry Signals)

**Priority 1 -- Highest reliability:**
- Moving average crossovers (Golden Cross: 50-day crosses above 200-day SMA)
- Two-candle reversal patterns: Bullish Engulfing, Bullish Harami (72.85% success rate per SSRN study)

**Priority 2 -- Good reliability:**
- Three-candle patterns: Morning Star, Three White Soldiers
- Chart patterns: Cup and Handle (equity-specific), Double Bottom, Ascending Triangle

**Priority 3 -- Secondary confirmation:**
- Single-candle patterns: Hammer, Inverted Hammer (use as confirmation only, not standalone)

### Bearish Patterns (Reinterpreted as EXIT Signals)

| Pattern | Long-Only Action |
|---------|-----------------|
| Bearish Engulfing | EXIT existing long position |
| Evening Star | EXIT existing long position |
| Three Black Crows | EXIT + go to cash |
| Head and Shoulders | Tighten stops immediately; EXIT on neckline break |
| Death Cross (50 below 200 SMA) | EXIT all positions; go to cash |
| Shooting Star / Hanging Man | Tighten trailing stop |

### Patterns Deprioritized

- Purely bearish continuation patterns (descending triangles, bear flags) -- no actionable long-only signal
- Single-candle patterns as standalone signals -- too noisy on daily charts

### Pattern Confirmation Requirements

- Require volume confirmation for all breakout patterns (volume > 1.5x 20-day average)
- Patterns at support levels are more reliable than patterns in free-fall
- Candlestick patterns work on daily charts for equities (not intraday, per research evidence)

---

## 11. Application Architecture

### Recommended Stack

| Component | Library | Version/Status | Role |
|-----------|---------|---------------|------|
| CLI Framework | **Typer** | Active | Subcommands, --verbose flag, type-hint based |
| Indicators | **TA-Lib** | v0.6.8 (Oct 2025), Active | 150+ indicators, C speed |
| Backtesting | **backtesting.py** | v0.6.5, Active | Walk-forward built-in, Bokeh charts |
| Benchmark Data | **yfinance** | Active | S&P 500 (SPY) data, split/dividend history |
| Terminal Output | **Rich** | v14.2.0 (Jan 2026), Active | Color tables, progress bars |
| Configuration | **Pydantic Settings** | Active | Type-safe config, YAML + CLI override |
| Packaging | **uv** | Active | Fast, modern, lock files |
| Testing | **pytest + hypothesis** | Active | Unit, integration, property-based |

### Key Stack Change: backtesting.py Replaces vectorbt

**Why**: vectorbt's free version is in maintenance-only mode (development shifted to paid vectorbt PRO). backtesting.py is actively maintained, has built-in walk-forward optimization support, and a simpler API for this use case.

### Design Patterns

1. **DataProvider (Repository pattern)**: Abstract base class with `get_price_data()` and `get_benchmark_data()`. Implement `CSVProvider` now, `APIProvider` later.
2. **Strategy pattern**: `BaseStrategy` ABC with `generate_signals()` and `evaluate()`. Implement `EquityStrategy` now, `OptionsStrategy` later.
3. **Signal type**: Asset-agnostic `Signal` dataclass that can represent different instrument types in the future.

### CLI Structure

```
app analyze --csv path/to/data.csv [--config config.yaml] [--verbose]
app report [--format csv|terminal] [--output path]
app config [--show | --set key=value]
```

### Configuration Hierarchy (lowest to highest priority)

1. Built-in defaults (in code)
2. YAML config file (`~/.config/technical-analysis/config.yaml`)
3. Environment variables
4. CLI arguments (highest priority)

### Configuration Parameters

```yaml
analysis:
  trade_frequency_days: 30
  max_drawdown_percent: 30
  max_indicator_combinations: 5
  min_data_points: 200

backtesting:
  method: walk_forward
  train_window_years: 3
  test_window_months: 6
  reoptimize_frequency: monthly

reporting:
  verbose: false
  export_csv: true
  export_path: ./output/

benchmarks:
  sp500_ticker: "SPY"
  cash_return: risk_free  # or "zero"
```

---

## 12. Library and Tooling Recommendations

### Library Status Assessment (2025-2026)

| Library | Status | Version | Notes |
|---------|--------|---------|-------|
| **TA-Lib** | Active, healthy | 0.6.8 (Oct 2025) | Python 3.9-3.14 support. C core. 40+ contributors. |
| **backtesting.py** | Active | 0.6.5 | Walk-forward optimization built-in. Bokeh charts. |
| **yfinance** | Active | Current | For SPY benchmark data and split/dividend history only |
| **Rich** | Active | 14.2.0 (Jan 2026) | Terminal tables, color coding, progress bars |
| **vectorbt (free)** | Maintenance only | 0.21.0 | Dev focus shifted to paid PRO version. Still functional. |
| **pandas-ta** | At risk | -- | Will be archived by July 2026 unless funding. Avoid. |
| **backtrader** | Abandoned | -- | Unmaintained since 2018. Avoid. |

### TA-Lib Installation Note

TA-Lib requires a C library (`ta-lib`) installed system-wide before the Python wrapper. On Ubuntu: `sudo apt install ta-lib` or build from source. The Python package (`TA-Lib`) is installed via uv/pip after the C library.

### Reporting Output

- **Primary**: Rich terminal tables with color-coded profit/loss
- **Secondary**: CSV export for further analysis in Excel/Google Sheets
- **Future**: HTML reports via Bokeh (backtesting.py already generates interactive charts)

---

## 13. Anti-Overfitting Measures

### Core Practices

1. **Deflated Sharpe Ratio (DSR)**: Apply the Bailey/Lopez de Prado multiple testing correction. Adjusts observed Sharpe ratio downward based on number of trials, skewness, and kurtosis. Most important single anti-overfitting measure.

2. **Walk-Forward Efficiency (WFE)**: OOS annualized return / IS annualized return. Values > 50-60% indicate robustness. Report for every strategy.

3. **Combinatorial Purged Cross-Validation (CPCV)**: Constructs multiple train/test splits with purging (removes overlapping samples) and embargo periods. Produces a distribution of OOS estimates. Lower Probability of Backtest Overfitting (PBO) than standard walk-forward.

4. **Parameter count limit**: Fewer parameters = less overfitting risk. Limit each strategy to 3-5 parameters. The hierarchical filtering approach is good because each layer has few parameters.

5. **Limited parameter values per indicator**: Test only a few values per parameter (e.g., RSI period: test 7, 14, 21 -- not 5-50 in steps of 1).

6. **Transaction cost modeling**: Even though user says costs aren't material, modeling 0.05-0.15% per trade catches overfitted strategies that only work with zero friction. Strategies overstate returns by 3.7-8.2% annually without costs.

7. **Pre-registration**: Define "good" thresholds BEFORE testing (Sharpe > 1.0, MaxDD < 30%, WFE > 50%). Don't move goalposts after seeing results.

### Multiple Testing Adjustment

With 100+ combinations tested, apply Bonferroni correction or DSR to avoid selecting a combination that looks good by chance. Testing 3000 combinations on 2500 bars almost guarantees some will appear profitable by luck alone.

### 2024 Survey Evidence

Traders using formal controls (CPCV, SPA, or pre-registration) had 23% higher consistency between backtest and live results.

---

## 14. Equity-Specific Considerations

### Structural Long Bias

US equities have a structural long bias (~10% annualized returns historically). This means:
- Long-only strategies have a tailwind that doesn't exist in forex or commodities
- Buy-the-dip strategies have historically outperformed in equities
- Even a mediocre long-only strategy benefits from the upward drift

### Dividend-Adjusted Prices

All return calculations should use dividend-adjusted closing prices. Raw prices create false performance metrics around ex-dividend dates.

### Earnings Season Effects

- Earnings announcements cause extreme price gaps that can trigger false indicator signals
- NVDA regularly gaps 5-15% on earnings
- **v1 recommendation**: Ignore in signal generation but handle gaps properly in backtesting
- **Gap handling**: The backtester must fill at actual open price after gaps, not at the theoretical trigger price
- **Future enhancement**: Optional earnings-date filter to exclude signals near earnings

### Post-Earnings Announcement Drift (PEAD)

A well-documented anomaly where stocks continue drifting in the direction of the earnings surprise for weeks after the announcement. Technical indicators will naturally respond to this drift -- no special handling needed for v1.

### Volume Characteristics

- Volume confirmation is critical for breakouts in equities -- much more reliable than forex
- Volume spikes on breakouts increase reliability significantly
- Candlestick patterns hold better in highly liquid US equities due to stronger participation

### Single-Equity Limitations

- Results from backtesting one stock (e.g., NVDA) may not generalize to other equities
- NVDA specifically had extraordinary performance 2015-2024 due to the AI boom -- be cautious about extrapolating
- The application should warn users that results are specific to the tested equity

---

## 15. Design Recommendations (Revised)

Based on all v2 research findings, the application should:

1. **Implement Walk-Forward Analysis** with 3-year in-sample, 6-month out-of-sample windows. Re-optimize monthly (configurable). Report Walk-Forward Efficiency for every strategy.

2. **Use a two-phase architecture**: Lightweight daily signal generation + periodic heavyweight optimization.

3. **Compute all core metrics** for every indicator test: CAGR, Sharpe, Sortino, Calmar, MaxDD, Win Rate, Profit Factor, Expectancy, Avg Trade Duration, Time in Market.

4. **Benchmark against buy-and-hold AND S&P 500 (SPY)** using the same metrics, displayed side-by-side.

5. **Use revised composite scoring**: MaxDD 30%, Sharpe 25%, PF 15%, Win Rate 15%, Calmar 10%, Expectancy 5%. Auto-disqualify strategies exceeding 30% max drawdown.

6. **Apply anti-overfitting measures**: Deflated Sharpe Ratio, WFE threshold, CPCV, parameter count limits, multiple testing correction.

7. **Use dual-price approach**: Split-adjusted raw prices for indicator calculation, fully adjusted prices for return computation.

8. **Handle stock splits** by cross-referencing with yfinance split history and adjusting CSV data accordingly.

9. **Default to cash in ranging markets** (ADX < 20) with configurable mean-reversion option.

10. **Use Chandelier Exit** (ATR-based trailing stop) as the primary exit mechanism.

11. **Include Williams %R** as a primary indicator alongside RSI, MACD, MAs, Bollinger Bands, ADX, and OBV.

12. **Reinterpret all bearish signals** as exit/cash signals, never short entries.

13. **Report returns during flat periods** at the risk-free rate (configurable, default from FRED T-bill data).

14. **Use backtesting.py** instead of vectorbt for walk-forward support and active maintenance.

15. **Structure for extensibility** using Strategy pattern and DataProvider abstraction for future options and API data source support.

16. **Verbose mode**: Show all tested combinations with rejection reasons, grouped by reason, with near-misses highlighted.

---

## Academic Research Summary (Updated)

### Recent Studies (2023-2026)

| Study | Finding | Source |
|-------|---------|--------|
| MACD/RSI Prediction Accuracy (2024) | RSI: 81% accuracy, MACD: 78% across 26 stocks in 7 markets | UniversePG |
| RSI + MA Combined (2025) | Combined outperforms either individually for equity strategies | SAGE Journals |
| EMA Crossover (2024) | Beat buy-and-hold in 61% of tested stocks | SSRN (Mahajan) |
| ML Feature Importance (2024) | RSI + EMA + Bollinger hybrid outperformed simpler setups for tail risk management | arxiv:2412.15448 |
| Candlestick Patterns (2024) | Harami: 72.85% success rate; patterns work on daily charts, not intraday | SSRN |
| Constrained MaxDD Optimization (2024) | MILP approach 200x faster than Markowitz; more robust results | arxiv:2401.02601 |
| Walk-Forward Best Practices (2024) | Formal controls (CPCV, pre-registration) yield 23% higher backtest-to-live consistency | arXiv:2512.12924v1 |
