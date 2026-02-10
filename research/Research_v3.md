# Technical Analysis Application Research v3

## Scope: US Equities, Long-Only, Stateless CLI Tool

---

## 1. Executive Summary

### What Changed from v2

Research v2 addressed US equities with long-only trading, designing a dual-mode (full + daily) stateful application with HTML report output. Based on updated requirements, v3 introduces three fundamental changes: **stateless execution**, **stdout output**, and **earnings date ignorance**. These changes simplify the architecture substantially while preserving the analytical core.

**Key changes:**

- **Stateless execution:** Application has no persistence between runs. Every invocation re-analyzes all historical data from scratch. Removes the entire daily mode architecture, state directory (position.json, trades.csv, last_run.json, cache/), re-optimization scheduling, and cross-run position tracking. Each run is a pure function: `f(config, historical_data) -> report + signal`.
- **Standard output:** Analysis results go to stdout as formatted text. Processing errors go to stderr. Data quality warnings (missing trading days, zero-volume days) go to stderr. Removes HTML report generation, Plotly, Jinja2, mplfinance from the stack. Future roadmap includes expanding output options beyond terminal.
- **Earnings dates ignored:** "Ignore earning dates entirely in the analysis." Removes the v2 earnings gap detection (3x ATR heuristic), signal suppression around earnings, and gap trade reporting.
- **Data source undetermined:** Historical data source has not been determined. Data will be stock-split adjusted with volume. Application should support flexible CSV formats, not just Nasdaq.
- **Configuration location specified:** Default `~/.config/technical-analysis/config.yaml` with user override.
- **Starting parameters explicit:** $10,000 default capital, January 1, 2000 default start date.
- **Transaction costs:** Explicitly "not material." Default to 0% (configurable).
- **v2 questions resolved:** Stateless answers NQ1.1, actionable signal answers NQ1.2, drawdown hard stop answers NQ3.2, 100% position sizing answers NQ7.2, zero cash return answers NQ7.1.

**Key findings (unchanged from v2):**

- Indicator tier rankings remain the same (Tier 1: EMA, RSI, MACD, ATR, ADX, Bollinger Bands, Williams %R)
- Walk-forward analysis parameters unchanged (504-day IS / 126-day OOS)
- 5 curated combinations unchanged
- Composite scoring weights unchanged
- TA-Lib + vectorbt remains the recommended core stack

**Key findings (new in v3):**

- Python remains the clear platform choice despite Go/Rust runtime advantages (see Section 2)
- yfinance is the recommended single external dependency for S&P 500 data, split validation, and dividend information
- IEX Cloud has shut down (August 2024) and must be removed from consideration
- yfinance auto_adjust behavior changed in v0.2.51 (December 2024), affecting benchmark data consistency
- The stateless model simplifies architecture by ~30-40% while the stdout output model removes 4 dependencies

---

## 2. Vendor Question Answers

### VQ1: Python vs Go/Rust -- Runtime Requirements Trade-offs

#### Runtime Requirements Comparison

| Metric | Python | Go | Rust |
|--------|--------|-----|------|
| Runtime required | Yes (50-100MB) | No | No |
| Deployment size | 300-500MB | 15-30MB | 5-20MB |
| Docker image | 800MB-1.2GB | 15-30MB | 5-20MB |
| System-level deps | TA-Lib C library | None | None |
| Cross-platform dist | Possible but painful | Trivial | Possible with effort |

Python requires the interpreter (50-100MB) plus compiled extensions. TA-Lib specifically requires a separate C library installation (`apt-get install`, `brew install`, or compile from source) before the Python wrapper can be pip-installed. Total virtual environment footprint is 300-500MB.

Go and Rust both produce single static binaries (15-30MB for Go, 5-20MB for Rust) with zero runtime dependencies. Cross-compilation is built into Go; Rust requires target-specific toolchains.

#### Library Ecosystem Comparison

| Capability | Python | Go | Rust |
|-----------|--------|-----|------|
| TA indicators (150+) | TA-Lib (C speed) | ~40-50 (go-talib) | ~20-50 (ta-rs/yata) |
| Vectorized backtesting | vectorbt | None | None |
| Performance tearsheets | quantstats | None | None |
| DataFrame operations | pandas (excellent) | gonum/dataframe (primitive) | polars-rust (good, verbose) |
| Benchmark data download | yfinance (1 line) | Manual HTTP | Manual HTTP |
| Candlestick charts | mplfinance | None adequate | None adequate |
| Interactive charts | Plotly | go-echarts (basic) | None adequate |
| Trading calendars | exchange_calendars | None | None |

The Python ecosystem advantage is decisive. vectorbt (backtesting), quantstats (analytics), and pandas (data manipulation) have no equivalents in Go or Rust. Building equivalent functionality from scratch would require an estimated 6,000-15,000 lines of code vs. 2,000-4,000 in Python.

#### Performance at This Scale

For a single instrument with ~2,500 rows of daily data:

- **Python with TA-Lib:** All indicators complete in < 50ms. Full WFA pipeline in 2-5 seconds.
- **Go:** All indicators in < 25ms. Negligible difference at this scale.
- **Rust:** All indicators in < 15ms. Negligible difference at this scale.

Memory usage: Python ~50-80MB, Go ~10-15MB, Rust ~5-10MB. All trivial on modern hardware.

Startup time: Python 200ms-2s (import overhead), Go/Rust 1-5ms. Irrelevant for a once-daily batch job.

**Performance differences become meaningful only at 500+ instruments processed concurrently or with real-time requirements. Neither applies to this application.**

#### Development Time Estimate

| Language | Estimated Lines | Estimated Time | Risk |
|----------|:-:|:-:|---|
| Python | 2,000-4,000 | 2-4 weeks | Low (libraries handle 60-70% of functionality) |
| Go | 8,000-15,000 | 8-16 weeks | High (backtesting, analytics, WFA from scratch) |
| Rust | 6,000-12,000 | 8-14 weeks | High (same gaps as Go, polars helps with data) |

#### Recommendation

**Python is the clear choice for this application.** The ecosystem advantage outweighs runtime benefits at every decision point:

1. **Scale does not justify Go/Rust:** 2,500 rows, single instrument, once-daily execution.
2. **Library coverage is decisive:** 60-70% of functionality comes from existing Python libraries. Reimplementing in Go/Rust introduces both development time and correctness risk.
3. **Financial calculation precision:** TA-Lib has been validated against Bloomberg/Reuters for decades. Custom reimplementations carry subtle bug risk.
4. **Runtime concerns have Python-native solutions:**
   - **Docker:** Package as container, user runs `docker run ta-app`. Runtime requirement: Docker only.
   - **pandas-ta instead of TA-Lib:** Pure Python alternative with 130+ indicators, eliminates the C library dependency entirely. Performance difference is imperceptible at 2,500 rows.
   - **conda-pack:** Create relocatable environment with all deps including TA-Lib. Distribute as tarball.
   - **PyInstaller/Nuitka:** Create standalone executable (100-300MB, platform-specific).

**Hybrid approach for future evolution:** If the application later needs a web API or multi-instrument real-time processing, a Go/Rust service layer can invoke the Python analysis core via subprocess or gRPC. The analytical logic stays in Python where the ecosystem advantage is overwhelming.

### VQ2: Historical S&P 500 Index Data

The application requires S&P 500 data for dual benchmark comparison (strategy vs. buy-and-hold of asset AND S&P 500).

#### Recommended Options

**Option 1: User-Provided CSV (Primary)**

The user provides an S&P 500 CSV from the same source as their stock data, in the same format. This requires zero additional parser logic.

- **Preferred ticker:** ^GSPC (S&P 500 index, no management fee drag) over SPY (ETF, 0.0945% annual expense ratio creating ~0.9% cumulative drag over 10 years).
- **If user has SPY data:** Acceptable, but note that SPY underperforms the pure index by its expense ratio. VOO (0.03% expense ratio) is a closer proxy if SPY is not available.
- **Nasdaq.com provides** historical data for both ^GSPC and SPY in the same format as individual stock data.

**Option 2: yfinance Automatic Download (Fallback)**

```python
import yfinance as yf
data = yf.download("^GSPC", start="2000-01-01", auto_adjust=False)
```

- **Coverage:** Full daily OHLCV back to 1927 for ^GSPC.
- **auto_adjust=False is critical:** Ensures Close column is split-adjusted only (matching Nasdaq CSV behavior). With auto_adjust=True (the default since v0.2.51), Close is dividend+split adjusted, creating inconsistency with the user's stock data which is split-adjusted only.
- **Reliability:** yfinance is actively maintained (2025-2026), widely used, and ^GSPC data has been stable.
- **Legal note:** Yahoo Finance Terms of Service restrict commercial redistribution. For personal use analysis, yfinance is acceptable. This application is a personal analysis tool.
- **Cache:** Download once per run, no persistence needed (stateless application).

**Option 3: FRED (Federal Reserve Economic Data)**

- **Series:** SP500 (daily close from 1971-02-05 to present)
- **Limitation:** Close price only -- no Open, High, Low, or Volume data
- **Use case:** Sufficient for return comparison only (not for technical indicator calculation on the benchmark)
- **API:** Free, reliable, no rate limits for daily data

**Option 4: Alpha Vantage**

- **Free tier:** 25 API calls/day, 5/minute. Sufficient for a single benchmark download.
- **Data:** Full OHLCV plus adjusted close, split/dividend adjusted
- **Assessment:** Viable if yfinance is unavailable, but the 25-call daily limit is restrictive

**Option 5: Polygon.io**

- **Free tier:** 5 API calls/minute, comprehensive market data
- **Assessment:** Good quality but free tier is very limited; primarily useful with paid subscription

#### Graceful Degradation

If neither user CSV nor yfinance download succeeds:
1. Emit warning to stderr: "S&P 500 benchmark data unavailable. Reporting only asset buy-and-hold benchmark."
2. Skip all S&P-relative metrics (excess return vs S&P, alpha, beta, information ratio)
3. Report only asset buy-and-hold comparison

#### Consistency Concern

If the user's stock CSV is split-adjusted but NOT dividend-adjusted (Nasdaq CSV default), while yfinance ^GSPC data IS dividend-adjusted, the benchmark comparison has a systematic bias of approximately 1.3% annually (the S&P 500 dividend yield). The application should use `auto_adjust=False` when downloading benchmark data via yfinance to match the user's data treatment, or document the discrepancy.

### VQ3: Stock Split Information

#### Context

The requirements state: "Historical data will be stock-split adjusted." Split information is needed for **validation** (confirming data is properly adjusted), not correction.

#### Recommended Options

**Option 1: yfinance `ticker.splits` (Primary)**

```python
import yfinance as yf
splits = yf.Ticker("NVDA").splits  # Date-indexed Series with split ratios
```

- Returns split history with dates and ratios (e.g., 2024-06-10: 10.0 for NVDA's 10:1 split)
- Already in the technology stack, zero additional dependencies
- Comprehensive for US equities, goes back decades

**Option 2: Financial Modeling Prep (Secondary)**

- Dedicated stock split history endpoint with clean JSON API
- Free tier: 250 requests/day
- Provides dates, numerator, denominator (e.g., 10:1)
- Requires API key registration

**Option 3: Alpha Vantage (Tertiary)**

- `TIME_SERIES_DAILY_ADJUSTED` includes split coefficient per row
- Free tier: 25 calls/day
- Split data extracted from daily data rather than a dedicated endpoint

**Option 4: SEC EDGAR (Reference Only)**

- Official filings (8-K, Item 5.03) for authoritative records
- Not practical for programmatic extraction
- Useful for manual verification of disputed splits

#### Built-in Validation Heuristic

The application should detect suspected unadjusted splits in data:

- **Price-based:** Day-over-day close ratio near common split factors (2:1 ≈ 0.50, 3:1 ≈ 0.33, 4:1 ≈ 0.25, 5:1 ≈ 0.20, 10:1 ≈ 0.10) within 2% tolerance
- **Volume confirmation:** Volume should change by the inverse ratio (2:1 split → volume roughly doubles)
- **Market context:** Check if S&P 500 also dropped similarly (rules out genuine crash)
- **Reverse splits:** Also detect 1:2 ≈ 2.0, 1:5 ≈ 5.0, 1:10 ≈ 10.0

Common split ratios for US equities: 2:1, 3:1, 3:2, 4:1, 5:1, 7:1 (AAPL 2014), 10:1 (NVDA 2024), 20:1 (AMZN 2022).

**Output behavior:** When a suspected unadjusted split is detected, emit warning to stderr with date, observed price ratio, and likely split ratio. Cross-reference with yfinance split data if available. The application should NOT attempt automatic correction -- warn and let the user provide corrected data.

### VQ4: Stock Dividend Information

#### Context

Nasdaq CSV data is split-adjusted but NOT dividend-adjusted. For low-dividend stocks (NVDA ~0.03% yield), this is negligible. For high-dividend stocks, it materially understates returns and can create false sell signals on ex-dividend dates.

#### Impact by Dividend Yield

| Category | Examples | Approx Yield | 4-Year Cumulative Impact | Indicator Impact |
|----------|---------|:---:|:---:|---|
| Zero/negligible | NVDA, AMZN, GOOG | 0-0.1% | < 0.5% | None |
| Low | MSFT, AAPL | 0.3-0.8% | 1-3% | Negligible |
| Moderate | JNJ, PG | 2-3% | 8-12% | Minor false signals on ex-dates |
| High | XOM, VZ, T | 3-5% | 12-22% | Material; multiple false sell signals/year |
| Very high (REITs) | O, AGNC, NLY | 5-12%+ | 22-50%+ | Severe; analysis accuracy compromised |

#### Recommended Options

**Option 1: yfinance `ticker.dividends` (Primary)**

```python
import yfinance as yf
dividends = yf.Ticker("AAPL").dividends  # Date-indexed Series with per-share amounts
```

- Returns dividend history keyed by ex-dividend date (correct for price impact analysis)
- Also provides `ticker.actions` combining splits and dividends
- Already in the technology stack

**Important yfinance change (December 2024):** In v0.2.51+, `auto_adjust=True` is the default. All OHLC prices are dividend+split adjusted. The `Adj Close` column no longer exists in default output. This means yfinance "Close" is NOT equivalent to Nasdaq CSV "Close/Last" (which is split-adjusted only).

**Option 2: Financial Modeling Prep (Secondary)**

- Dedicated Dividends Company API with record dates, payment dates, declaration dates, ex-dividend dates, and amounts
- Provides `close` (split-adjusted only) and `adjClose` (split+dividend adjusted) separately -- the clearest separation
- Free tier: 250 requests/day

**Option 3: Alpha Vantage (Tertiary)**

- `TIME_SERIES_DAILY_ADJUSTED` includes dividend amount per day
- Separate `DIVIDENDS` function for history
- Free tier: 25 calls/day

**Option 4: Tiingo (Specialty)**

- Highest data quality for dividend data specifically
- Dedicated corporate actions feeds with full detail
- Free tier: 50 symbols/hour
- Provides both adjusted and unadjusted prices

#### Recommended Handling Strategy

**Accept data as-is with detection and flagging:**

1. Use the CSV prices (split-adjusted, not dividend-adjusted) for all indicator calculations and backtesting
2. Download dividend history from yfinance for the stock being analyzed
3. On each ex-dividend date, flag the expected price drop in output
4. Include in report header: "Data is split-adjusted, not dividend-adjusted. Return calculations do not include dividends."
5. If trailing 12-month dividend yield exceeds 1%, emit warning to stderr with the yield and approximate annual return understatement
6. **Future enhancement:** Support an `adj_close` column in CSV. If present, use it for return calculations.

#### The Double-Counting Trap

If using dividend-adjusted close prices, do NOT also add dividend payments to return calculations. The adjustment already accounts for dividends by reducing all historical prices. Adding dividends on top double-counts them. Conversely, if using raw prices, dividends MUST be added for accurate total return.

#### Benchmark Consistency

Ensure the benchmark (S&P 500) and the asset use the same adjustment basis. If the asset CSV is not dividend-adjusted, use `auto_adjust=False` when downloading benchmark data via yfinance to match. Otherwise the S&P 500 benchmark has a systematic ~1.3% annual advantage from included dividends.

#### Defunct Data Source: IEX Cloud

IEX Cloud shut down on August 31, 2024. All API products have been retired. It should be removed from consideration in any data source evaluation.

---

## 3. Revised Architecture

### 3.1 What Changed from v2

The v3 architecture eliminates approximately 30-40% of the v2 design surface through three simplifications:

| v2 Design Element | v3 Change | Reason |
|-------------------|-----------|--------|
| Daily/full dual modes | **Remove** | Stateless -- single mode only |
| State directory (position.json, trades.csv, last_run.json, cache/) | **Remove** | Stateless -- no persistence |
| Re-optimization schedule (quarterly, 63-day interval) | **Remove** | Stateless -- re-optimize every run |
| HTML report generation (Jinja2, Plotly, mplfinance) | **Remove** | stdout text output |
| Computation caching (pyarrow) | **Remove** | Stateless -- no cache |
| Earnings gap detection (3x ATR heuristic) | **Remove** | "Ignore earning dates entirely" |
| Earnings signal suppression | **Remove** | "Ignore earning dates entirely" |
| Gap trade reporting | **Remove** | "Ignore earning dates entirely" |
| Nasdaq-only CSV parser | **Replace** with flexible CSV parser | Data source TBD |
| Generic config/ directory | **Replace** with XDG path | `~/.config/technical-analysis/config.yaml` |

### 3.2 Revised Pipeline

```
Configuration (YAML from ~/.config/technical-analysis/config.yaml or CLI override)
    |
    v
Data Loading (CSV, format auto-detected)
    |
    v
Data Validation (warnings to stderr: missing days, zero-volume, suspected splits)
    |
    v
Benchmark Loading (S&P 500 CSV or yfinance fallback; skip if unavailable)
    |
    v
Indicator Calculation (Tier 1-3, all configured indicators)
    |
    v
Signal Generation (long-only, entry/exit per indicator and combination)
    |
    v
Walk-Forward Analysis (504-day IS / 126-day OOS, rolling windows)
    |                   |
    |              Drawdown Hard Stop
    |              (sell if >= configured max DD)
    v
Performance Metrics + Dual Benchmark Comparison
    |
    v
Composite Scoring & Ranking
    |
    v
Current Signal Determination (BUY today / HOLD / SELL today)
    |
    v
Report to stdout (summary + trade log + current signal)
    |   Verbose: full indicator evaluation with rejection reasons
    |   Warnings/errors to stderr
    v
Exit (no state saved, no files written)
```

### 3.3 Signal Determination Logic (New for Stateless Model)

In v2, the daily mode maintained position state across runs. In v3, the signal is derived entirely within a single execution:

1. Run complete WFA and ranking to select the top strategy
2. Using the top strategy's parameters, generate signals across the entire historical period
3. Identify the most recent signal transition:
   - Find the last BUY signal date and the last SELL signal date
   - If last BUY > last SELL: strategy is currently "in position"
   - If last SELL > last BUY: strategy is currently "out of position"
   - If no signals: strategy has never triggered
4. Check drawdown: If currently in position, compute drawdown from peak since entry. If >= configured max, signal is SELL regardless
5. Determine today's signal:
   - If today generates a new BUY signal and currently out: **BUY today**
   - If today generates a new SELL signal and currently in: **SELL today**
   - If drawdown limit reached and currently in: **SELL today**
   - Otherwise: **HOLD** (with context: "in position since [date]" or "out since [date]")

### 3.4 Stdout Report Structure

```
=== Technical Analysis Report: [TICKER] ===
Analysis period: [START] to [END] ([N] trading days)
Starting capital: $[AMOUNT]

--- Top Strategy ---
#1: [e.g., MACD(12,26,9) + RSI(14)] (Composite score: 0.847)
    Walk-forward windows: [N] (504d IS / 126d OOS)
    OOS total return: [X]%     Max drawdown: [X]%
    Sharpe: [X]                Win rate: [X]%

#2: [next strategy] (Composite score: 0.791)
    ...
[Up to configured max combinations]

--- Current Signal (Top Strategy) ---
Signal: [BUY today | HOLD | SELL today]
Position state: [In position since YYYY-MM-DD | Out since YYYY-MM-DD]
Current drawdown from peak: [X]% (limit: [X]%)

--- Trade Log (Top Strategy) ---
#   Entry Date   Exit Date    Entry $    Exit $    P&L $     P&L %    Days
1   2000-03-15   2000-05-22   25.30      31.40     +6.10     +24.1%   48
2   ...
...

--- Summary Statistics ---
Total trades: [N]               Win rate: [X]%
Average trade P&L: [X]%         Average trade duration: [X] days
Average time between trades: [X] days
Max consecutive losses: [N]     Final portfolio value: $[X]

--- Benchmark Comparison ---
                        Strategy    Buy & Hold    S&P 500
Total return            X%          X%            X%
Annual return (CAGR)    X%          X%            X%
Max drawdown            X%          X%            X%
Sharpe ratio            X           X             X
Sortino ratio           X           X             X

--- [Verbose: Indicator Evaluation] ---
(Only shown with --verbose flag)

Examined [N] indicator/combination evaluations:

  ACCEPTED (ranked by composite score):
  1. MACD+RSI: OOS return X%, MaxDD X%, Sharpe X, Score: 0.847
  2. Williams %R+EMA: OOS return X%, MaxDD X%, Sharpe X, Score: 0.791
  ...

  REJECTED:
  5. Donchian Breakout: MaxDD 38.7% (exceeds 30% limit)
  6. Stochastic: CAGR 7.3% vs buy-and-hold 11.8%
  ...

  NO TRADES GENERATED:
  8. Ichimoku Cloud: No entry signals in backtest period
  ...
```

### 3.5 Revised Technology Stack

| Component | v2 | v3 | Change |
|-----------|----|----|--------|
| Language | Python 3.11+ | Python 3.11+ | Unchanged |
| Data manipulation | pandas 2.x | pandas 2.x | Unchanged |
| Indicator computation | TA-Lib 0.4.28+ | TA-Lib 0.4.28+ | Unchanged |
| Backtesting | vectorbt 0.26+ | vectorbt 0.26+ | Unchanged |
| Performance analytics | quantstats | quantstats | Unchanged |
| Static charts | mplfinance 0.12.10+ | **REMOVED** | No chart output |
| Interactive charts | Plotly 5.18+ | **REMOVED** | No chart output |
| Report generation | Jinja2 3.1+ | **REMOVED** | stdout text |
| Trading calendar | exchange_calendars | exchange_calendars | Unchanged |
| S&P 500 / validation data | yfinance | yfinance | Unchanged (expanded role) |
| Configuration | PyYAML 6.0+ | PyYAML 6.0+ | Unchanged |
| CLI | click 8.x | click 8.x | Unchanged |
| Caching | pyarrow 14+ | **REMOVED** | Stateless |
| Testing | pytest 7.4+ | pytest 7.4+ | Unchanged |

**Net: 12 major dependencies reduced to 8.** Removing Plotly (large dependency tree), Jinja2, mplfinance, and pyarrow significantly reduces install footprint.

yfinance's role expands from "S&P 500 benchmark download" to also providing split validation data and dividend information for flagging.

### 3.6 Revised Module Structure

```
src/
  __init__.py
  config/
    __init__.py
    loader.py              # YAML config loading, XDG path resolution
    defaults.py            # Default parameter values ($10k, 2000-01-01, etc.)
  cli/
    __init__.py
    main.py                # CLI entry point (click), single command
  data/
    __init__.py
    loader.py              # DataLoader ABC + flexible CSV loader (auto-detect format)
    validator.py           # OHLCV validation, split detection heuristic
    calendar.py            # NYSE calendar integration
    benchmark.py           # S&P 500 loading (user CSV or yfinance fallback)
    corporate_actions.py   # Split/dividend info from yfinance for validation
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
    generator.py           # Signal generation (long-only, entry/exit)
    regime.py              # ADX-based regime detection
    composite.py           # Multi-indicator composite scoring
    current.py             # Current signal determination (BUY/HOLD/SELL)
  backtest/
    __init__.py
    engine.py              # vectorbt wrapper (long-only, 100% in/out)
    walk_forward.py        # WFA orchestration (504/126 rolling)
    metrics.py             # Performance metrics (full set)
    scoring.py             # Composite scoring and ranking
    drawdown.py            # Drawdown hard stop (sell at max DD)
    benchmark.py           # Dual benchmark comparison
  reporting/
    __init__.py
    text_formatter.py      # Formatted text output to stdout
    diagnostics.py         # Warnings and errors to stderr
    verbose.py             # Verbose indicator evaluation output
  pipeline.py              # Main orchestration

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
  test_drawdown.py
  test_benchmark.py
  test_verbose.py
  test_pipeline.py
```

### 3.7 Revised YAML Configuration

```yaml
# ~/.config/technical-analysis/config.yaml

data:
  file: "path/to/stock.csv"          # Required
  # Column mapping (auto-detected if omitted):
  # columns:
  #   date: Date
  #   close: "Close/Last"
  #   volume: Volume
  #   open: Open
  #   high: High
  #   low: Low

benchmark:
  sp500:
    source: file                     # file or auto (yfinance download)
    path: "path/to/sp500.csv"        # Required if source: file
    # source: auto                   # Alternative: download ^GSPC via yfinance

analysis:
  start_date: "2000-01-01"          # Default
  initial_capital: 10000             # Default
  max_combinations_reported: 5       # Default
  target_holding_period_days: 30     # Default

backtest:
  transaction_cost_pct: 0.0          # Default (not material)
  max_drawdown_pct: 30.0             # Default (hard stop)
  walk_forward:
    in_sample_days: 504
    out_of_sample_days: 126
    step_days: 126
    warmup_days: 200
    method: rolling

output:
  verbose: false                     # Full indicator evaluation
```

---

## 4. Indicator Tier Rankings (Unchanged from v2)

The indicator tier rankings established in v2 remain valid. No new requirements affect indicator selection.

### 4.1 Tier 1 -- Must Implement (7 indicators)

| # | Indicator | Primary Use |
|---|-----------|-------------|
| 1 | **RSI** | Mean-reversion + confirmation |
| 2 | **Williams %R** | Mean-reversion + timing |
| 3 | **EMA** | Trend signal |
| 4 | **MACD** | Trend + momentum |
| 5 | **ATR** | Risk management (not signal) |
| 6 | **ADX** | Trend strength filter |
| 7 | **Bollinger Bands** | Mean reversion + breakout |

### 4.2 Tier 2 -- High Priority (5 indicators)

| # | Indicator | Primary Use |
|---|-----------|-------------|
| 8 | **OBV** | Volume confirmation (primary) |
| 9 | **CMF** | Volume confirmation (secondary) |
| 10 | **MA Crossover** | Trend change detection |
| 11 | **Supertrend** | Trend + trailing stop |
| 12 | **Elder's Triple Screen** | Multi-timeframe system |

### 4.3 Tier 3 -- Valuable Additions (9 indicators)

Keltner Channels, CCI, Stochastic, Parabolic SAR, Donchian Channels, A/D Line, VPT, SMA, Ichimoku Cloud.

### 4.4 Tier 4 -- Supplementary (4 indicators)

ROC, Standard Deviation, MA Ribbon, Momentum (raw).

Full tier ranking details, justifications, and change rationale from v1 are documented in `research/Research_v2.md` Sections 2-3.

---

## 5. Recommended Combinations (Unchanged from v2)

The 5 curated combinations remain unchanged:

1. **MACD + RSI Confirmation** -- Trend-following with momentum (60-73% expected win rate)
2. **ADX-Filtered EMA Crossover** -- Trend with regime filter (55-65% WR)
3. **RSI + Bollinger Mean-Reversion** -- Mean-reversion (70-85% WR)
4. **Supertrend + MACD** -- Trend with momentum (55-67% WR)
5. **Williams %R + EMA Trend** -- Mean-reversion within trend (65-80% WR)

Full entry/exit logic, filters, and expected characteristics are documented in `research/Research_v2.md` Section 7.

---

## 6. Backtesting Framework (Minor Changes from v2)

### 6.1 WFA Parameters (Unchanged)

| Parameter | Value |
|-----------|-------|
| In-sample window | 504 trading days (~2 years) |
| Out-of-sample window | 126 trading days (~6 months) |
| Step size | 126 trading days |
| Warmup period | 200 trading days |
| Method | Rolling windows |
| Minimum data | 830 trading days (~3.3 years) |

### 6.2 Execution Model (Minor Changes)

| Parameter | v2 | v3 | Change |
|-----------|----|----|--------|
| Transaction cost | 0.02% per trade | 0% default (configurable) | Requirement: "not material" |
| Starting capital | Not specified | $10,000 | Now explicit |
| Start date | Not specified | January 1, 2000 | Now explicit |
| Drawdown behavior | Staged circuit breaker (15/20/25%) | Hard stop at configured max | "If max DD reached, sell" |

All other execution model parameters (next-bar-open execution, 100% position sizing, long-only direction) remain unchanged.

### 6.3 Drawdown Hard Stop

The v2 staged circuit breaker (Level 1: 15% reduce size, Level 2: 20% no new entries, Level 3: 25% force close) is replaced by a simpler hard stop:

- **Trigger:** Current drawdown from equity peak >= configured maximum (default 30%)
- **Action:** Immediate sell signal, regardless of indicator state
- **In backtest:** Force exit at the trigger point. Record the forced exit in trade log.
- **In current signal:** If the top strategy's current position has drawdown >= max, signal is SELL today.
- **The staged circuit breaker may still be offered** as a configurable option, but the default behavior is binary: below threshold = normal, at/above = sell.

### 6.4 Composite Scoring (Unchanged from v2)

| Metric | Weight |
|--------|:---:|
| Sharpe Ratio | 0.15 |
| Sortino Ratio | 0.15 |
| Calmar Ratio | 0.15 |
| Profit Factor | 0.15 |
| Expectancy | 0.05 |
| Max Drawdown (penalty) | 0.10 |
| Number of Trades | 0.05 |
| OOS Consistency | 0.10 |
| Trade Frequency Alignment | 0.05 |
| Benchmark Outperformance | 0.05 |

### 6.5 Rejection Criteria (Unchanged from v2)

| Criterion | Threshold |
|-----------|-----------|
| Max drawdown exceeds limit | > configured max (default 30%) |
| Below buy-and-hold return | Strategy CAGR < buy-and-hold CAGR |
| Insufficient trades | < 30 total trades |
| Negative expectancy | Expectancy <= 0 |
| OOS degradation | OOS Sharpe < 50% of IS Sharpe |

---

## 7. Complete Change Summary: v2 to v3

### 7.1 What Was Removed

| v2 Element | Reason |
|------------|--------|
| Daily run mode (full + daily) | Stateless execution |
| State persistence (position.json, trades.csv, last_run.json) | Stateless execution |
| Computation cache (state/cache/) | Stateless execution |
| Re-optimization schedule (63-day interval) | Stateless -- re-optimize every run |
| HTML report generation | stdout output |
| Plotly interactive charts | stdout output |
| Jinja2 templating | stdout output |
| mplfinance static charts | stdout output |
| pyarrow caching | Stateless |
| Earnings gap detection (3x ATR heuristic) | "Ignore earning dates entirely" |
| Earnings signal suppression | "Ignore earning dates entirely" |
| Gap trade reporting | "Ignore earning dates entirely" |
| Staged circuit breaker (15/20/25%) | Replaced by hard stop |
| Nasdaq-only CSV parser | Flexible CSV (data source TBD) |

### 7.2 What Was Added

| v3 Element | Reason |
|------------|--------|
| Stateless signal determination algorithm | Core new feature for stateless model |
| Text-based stdout report formatter | Replaces HTML report |
| stderr diagnostics (warnings, errors) | Clean Unix-style I/O separation |
| Flexible CSV format auto-detection | Data source not yet determined |
| XDG config path (`~/.config/technical-analysis/`) | Requirement: specified config location |
| Explicit defaults ($10k, 2000-01-01) | Requirements now specify |
| Zero-trade indicator verbose output | "Make indicators that produce no trades available in verbose output" |
| Drawdown hard stop as active SELL trigger | "If max DD reached, sell the position" |
| Split validation via yfinance | VQ3 recommendation |
| Dividend impact flagging via yfinance | VQ4 recommendation |
| IEX Cloud removal | Shut down August 2024 |

### 7.3 What Was Unchanged

- Indicator tier rankings (all 25 indicators, same tiers)
- 5 curated combinations (same entry/exit logic)
- WFA parameters (504/126 rolling)
- Composite scoring weights (10 metrics, same weights)
- Rejection criteria (same 5 criteria)
- Core technology stack (Python, pandas, TA-Lib, vectorbt, quantstats, exchange_calendars, yfinance, PyYAML, click)
- Long-only signal design
- Volume confirmation framework
- Trade frequency as soft constraint
- Cash period tracking
- NYSE calendar integration

---

## Sources

### Vendor Question Research
- [yfinance GitHub Repository](https://github.com/ranaroussi/yfinance)
- [FRED S&P 500 Series](https://fred.stlouisfed.org/series/SP500)
- [Alpha Vantage Complete 2026 Guide](https://alphalog.ai/blog/alphavantage-api-complete-guide)
- [Alpha Vantage API Documentation](https://www.alphavantage.co/documentation/)
- [Financial Modeling Prep Dividends API](https://site.financialmodelingprep.com/developer/docs/stable/dividends-company)
- [Polygon.io Split/Dividend Adjustment Info](https://polygon.io/knowledge-base/article/is-polygons-stock-data-adjusted-for-splits-or-dividends)
- [IEX Cloud Shutdown Analysis](https://www.alphavantage.co/iexcloud_shutdown_analysis_and_migration/)
- [Tiingo End of Day Stock Price Data](https://www.tiingo.com/products/end-of-day-stock-price-data)
- [SPY ETF Trust Fund Facts](https://www.ssga.com/us/en/intermediary/etfs/spdr-sp-500-etf-trust-spy)
- [Yahoo Terms of Service](https://legal.yahoo.com/us/en/yahoo/terms/otos/index.html)
- [Why Adj Close Disappeared in yfinance](https://medium.com/@josue.monte/why-adj-close-disappeared-in-yfinance-and-how-to-adapt-6baebf1939f6)
- [Go-talib Library](https://github.com/markcheno/go-talib)
- [ta-rs Rust TA Library](https://crates.io/crates/ta)
- [yata Rust TA Library](https://crates.io/crates/yata)

### Prior Research (v1/v2 -- Still Valid)
- All sources from `research/Research_v2.md` Sources section remain valid
- `research/technical-indicators-research.md` -- 25-indicator detailed analysis
- `research/data-considerations-and-architecture.md` -- Data engineering reference

---

## Supplementary Documents

| Document | Contents |
|----------|---------|
| [research/Research_v2.md](Research_v2.md) | Previous version with full indicator analysis |
| [research/Research_v1.md](Research_v1.md) | Original multi-asset research |
| [notes/Research_Thinking_v3.md](../notes/Research_Thinking_v3.md) | v3 reasoning and methodology |
| [notes/Research_Questions_v3.md](../notes/Research_Questions_v3.md) | v3 questions and resolution status |
| [notes/Research_Thinking_v2.md](../notes/Research_Thinking_v2.md) | v2 reasoning (still valid for unchanged areas) |
| [notes/Research_Questions_v2.md](../notes/Research_Questions_v2.md) | v2 questions (resolution status updated in v3) |
