# Technical Analysis Research Report v3

## Executive Summary

This is a revised research report incorporating the updated requirements from `requirements/Main.md` and answering the vendor questions from `instructions/VendorQuestions.md`. The key requirement changes driving this revision are:

- **Historical data will be stock-split adjusted** -- the previous dual-price approach is simplified since the data source will provide split-adjusted prices
- **Data source not yet determined** -- no longer locked to raw Nasdaq CSV; may be an API or other source in the future
- **Earnings dates explicitly ignored** -- confirmed: ignore entirely in the analysis
- **Standard output conventions** -- analysis to stdout, errors/warnings to stderr
- **No custom indicators** -- explicitly out of scope
- **YAML configuration confirmed** -- configurable location, default `~/.config/technical-analysis/config.yaml`
- **Verbose mode detail** -- must include indicators that produce no trades

**Key changes from v2**: Data handling simplified (split-adjusted input), dual-price approach revised, vendor questions answered (Python vs Go/Rust, S&P 500 data sources, stock split data sources, stock dividend data sources), and several open questions from v2 are now resolved.

---

## Table of Contents

1. [Vendor Question Answers](#1-vendor-question-answers)
2. [Requirements Delta from v2](#2-requirements-delta-from-v2)
3. [Revised Data Handling](#3-revised-data-handling)
4. [S&P 500 Benchmark Data Sources](#4-sp-500-benchmark-data-sources)
5. [Stock Split Data Sources](#5-stock-split-data-sources)
6. [Stock Dividend Data Sources](#6-stock-dividend-data-sources)
7. [Updated Stack Recommendation](#7-updated-stack-recommendation)
8. [Revised Design Recommendations](#8-revised-design-recommendations)
9. [Status of v2 Open Questions](#9-status-of-v2-open-questions)

---

## 1. Vendor Question Answers

### VQ1: Python vs Go vs Rust -- Advantages and Disadvantages

**Question**: You seem to have settled on a Python implementation, which presents runtime requirements. What would be the advantages and disadvantages of using a platform, such as Go or Rust, to minimize runtime requirements?

#### Comparison Table

| Criterion | Python | Go | Rust |
|-----------|--------|----|----|
| **Technical indicator libraries** | TA-Lib (150+ indicators, C core, v0.6.8, active) | go-talib (CGo wrapper, ~100 indicators, sporadic updates) | ta-rs (~30 indicators, v0.5, community-maintained) |
| **Backtesting frameworks** | backtesting.py (walk-forward built-in, active) | None with walk-forward support | None with walk-forward support |
| **Time series / data handling** | pandas (industry standard) | gonum (numerical), no pandas equivalent | polars-rs (growing), ndarray (lower-level) |
| **CLI framework** | Typer (type-hint based, auto-docs) | cobra (mature, widely used) | clap (mature, derive macros) |
| **Terminal tables/colors** | Rich (v14.2, tables, progress, color) | lipgloss/bubbletea (good TUI ecosystem) | comfy-table, indicatif (adequate) |
| **YAML config** | Pydantic Settings (type-safe, validated) | viper (mature, multi-format) | serde + config-rs (good) |
| **Market data API clients** | yfinance (Yahoo Finance, mature) | No equivalent (would need HTTP + manual parsing) | No equivalent (would need HTTP + manual parsing) |
| **Quant finance ecosystem** | Mature (NumPy, SciPy, statsmodels, scikit-learn) | Minimal | Minimal |
| **Runtime requirements** | Python 3.10+ interpreter + TA-Lib C library | Single binary (~10-20 MB) | Single binary (~5-15 MB) |
| **Startup time** | ~200-500ms (interpreter + imports) | ~5-10ms | ~1-5ms |
| **Memory usage** | ~50-100 MB (interpreter + pandas overhead) | ~10-30 MB | ~5-20 MB |
| **Development speed** | Fast (rich ecosystem, less boilerplate) | Moderate (more boilerplate, good tooling) | Slower (steep learning curve, ownership model) |

#### Advantages of Go/Rust

1. **Single binary deployment**: No Python interpreter, no virtual environment, no TA-Lib C library installation. Just copy one file.
2. **Faster startup**: 5-10ms (Go) or 1-5ms (Rust) vs 200-500ms (Python). Relevant if running as a cron job thousands of times.
3. **Lower memory**: 10-30 MB (Go) or 5-20 MB (Rust) vs 50-100 MB (Python with pandas).
4. **No dependency management friction**: No pip/uv, no virtual environments, no C library compilation.
5. **Cross-compilation**: Easy to build for Linux, macOS, Windows from a single machine.

#### Disadvantages of Go/Rust

1. **No walk-forward backtesting framework exists** in either language. The application would need to implement walk-forward analysis from scratch -- a significant development effort (estimated 2-4 weeks additional work).
2. **Far fewer technical indicators available**: Go has ~100 via CGo wrapper (still requires C TA-Lib), Rust has ~30 in ta-rs. Missing indicators would need to be implemented manually.
3. **No yfinance equivalent**: Would need to build HTTP clients to fetch Yahoo Finance data, parse JSON responses, handle rate limiting, and deal with API changes manually.
4. **No pandas equivalent**: Time series operations (resampling, rolling windows, alignment, merge) that are one-liners in pandas become significant code in Go/Rust.
5. **Smaller quant finance community**: Far fewer Stack Overflow answers, blog posts, and examples for financial analysis.
6. **Higher development time**: Estimated 2-3x longer development timeline due to ecosystem gaps.
7. **Go still needs CGo for TA-Lib**: The Go TA-Lib wrapper (go-talib) uses CGo, which still requires the C library. This negates the "single binary" advantage for Go specifically.

#### Recommendation: Stay with Python

**Python is the correct choice for this application.** The reasons:

1. **The runtime requirements are minimal**: This application runs once daily as a batch job. The 200-500ms Python startup time is inconsequential for a daily cron job. Memory usage of ~100 MB is trivial on any modern system.
2. **The ecosystem gap is fundamental, not a maturity issue**: Go and Rust lack backtesting frameworks with walk-forward support, comprehensive indicator libraries, and financial data API clients. These are not gaps that will close soon -- the quantitative finance community is overwhelmingly Python-based.
3. **TA-Lib's C core provides the performance where it matters**: The actual indicator calculations (the compute-intensive part) run at C speed through TA-Lib. Python is just the orchestration layer.
4. **The deployment friction is manageable**: Using `uv` for package management and providing a `Dockerfile` or installation script for the TA-Lib C library addresses the runtime requirements concern.
5. **Development time matters**: Building walk-forward backtesting, 150+ indicators, and Yahoo Finance integration from scratch in Go/Rust would add months to the timeline with no meaningful performance benefit for a daily batch job on 2500 bars.

**If runtime deployment is a strong concern**, the best mitigation within Python is:
- Use `uv` for fast, reproducible dependency management
- Provide a `Dockerfile` for zero-friction deployment
- Consider `PyInstaller` or `Nuitka` to produce a single executable (trades build complexity for deployment simplicity)
- Use `conda-forge` for TA-Lib installation (avoids manual C library compilation)

---

### VQ2: Recommendations for Obtaining Historical S&P 500 Index Data

**Question**: Please present recommendations for obtaining historical S&P 500 index data.

#### Ranked Recommendations

**Primary: yfinance (SPY ETF)**

| Attribute | Detail |
|-----------|--------|
| **Source** | Yahoo Finance via `yfinance` Python library |
| **Ticker** | SPY (SPDR S&P 500 ETF Trust) |
| **Data available** | OHLCV + Adjusted Close, daily, from January 1993 (SPY inception) |
| **Cost** | Free |
| **Rate limits** | Unofficial; ~2,000 requests/hour practical limit |
| **Reliability** | Good. Had disruptions in 2024 when Yahoo changed APIs; yfinance adapted quickly. v0.2.50+ is stable. |
| **Integration** | `pip install yfinance` then `yf.download("SPY", start="2015-01-01")` |
| **Adjusted close** | Accounts for dividends and splits automatically |
| **Usage terms** | Personal/research use. Not for commercial redistribution. |

**Why SPY over ^GSPC (S&P 500 Index)**:
- SPY is investable (the index is not), making it a fairer benchmark for a trading strategy
- SPY's adjusted close includes dividend reinvestment, which ^GSPC does not
- SPY has minimal tracking error vs the actual S&P 500 index (~0.01% annually)
- SPY is the most liquid ETF in the world, so data quality is excellent

**Secondary: Alpha Vantage**

| Attribute | Detail |
|-----------|--------|
| **Source** | Alpha Vantage REST API |
| **Endpoint** | `TIME_SERIES_DAILY_ADJUSTED` for SPY |
| **Cost** | Free tier: 25 requests/day. Premium starts at $49.99/month. |
| **Data available** | 20+ years of daily OHLCV + adjusted close |
| **Integration** | `pip install alpha_vantage` or direct REST calls |
| **API key** | Required (free registration) |
| **Reliability** | Stable API, well-documented. Free tier is heavily rate-limited. |

**Tertiary: Tiingo**

| Attribute | Detail |
|-----------|--------|
| **Source** | Tiingo REST API |
| **Endpoint** | `/iex` or `/tiingo/daily` for SPY |
| **Cost** | Free tier: 500 unique symbols/month, 50 requests/hour |
| **Data available** | 20+ years, adjusted prices included |
| **Integration** | REST API, also supported by `pandas-datareader` |
| **Reliability** | Stable, well-regarded in quant community |

**Offline fallback: Local CSV cache**

- Download SPY data via yfinance on first run, cache to `~/.cache/technical-analysis/SPY.csv`
- Re-download when cache is older than 1 day (since the app runs daily)
- If no internet access, fall back to cached data with a warning on stderr
- This addresses reliability concerns with any single API

**Not recommended**:

| Source | Reason |
|--------|--------|
| FRED (Federal Reserve) | Only has S&P 500 close price (no OHLCV). Useful as cross-reference but not primary. |
| Quandl/Nasdaq Data Link | Most free datasets deprecated or require paid subscription |
| Polygon.io free tier | 5 API calls/min, 2-year historical limit on free tier. Too restrictive. |
| Stooq.com CSV | Good for manual download, but no API. Not suitable for automated daily benchmarking. |
| investing.com | Scraping required; ToS prohibit automated access |

#### Implementation Approach

```
Priority chain: yfinance → Alpha Vantage → local cache → error with helpful message
```

The application should:
1. Attempt yfinance first (no API key needed, best data quality)
2. Fall back to Alpha Vantage if yfinance fails (requires API key in config)
3. Fall back to local cache if both APIs fail
4. Error with a clear message if no benchmark data is available, suggesting manual CSV download

---

### VQ3: Options for Obtaining Stock Split Information

**Question**: Please present options for obtaining stock split information, which affect the analysis.

#### Context Change in v3

The user now states that **historical data will be stock-split adjusted**. This changes the role of split data from "required for price adjustment" to "needed for validation and cross-referencing." The application still needs split information to:

1. **Validate that CSV data is actually split-adjusted** (detect anomalies)
2. **Reconcile different data sources** (if data source changes in the future)
3. **Inform the user** about significant corporate actions in the analysis period

#### Ranked Recommendations

**Primary: yfinance `stock.splits`**

| Attribute | Detail |
|-----------|--------|
| **Access** | `yf.Ticker("NVDA").splits` returns a pandas Series |
| **Coverage** | All US exchange-listed equities. Historical depth varies: 20+ years for major equities. |
| **Split types** | Forward splits (2:1, 3:1, 4:1, 10:1, etc.), reverse splits (1:2, 1:10, etc.) |
| **Data format** | Date index, float value (e.g., 10.0 for a 10:1 split) |
| **Accuracy** | Reliable for major equities. Cross-reference with SEC filings for critical decisions. |
| **Cost** | Free |
| **Rate limits** | Same as yfinance general limits (~2,000 req/hour) |

**NVDA split history via yfinance**:
```
2024-06-10    10.0  (10:1 forward split)
2021-07-20     4.0  (4:1 forward split)
2007-09-11     1.5  (3:2 forward split)
2006-04-07     2.0  (2:1 forward split)
2001-09-17     2.0  (2:1 forward split)
2000-06-27     2.0  (2:1 forward split)
```

**Secondary: Alpha Vantage Stock Splits**

| Attribute | Detail |
|-----------|--------|
| **Endpoint** | `SPLITS` function |
| **Coverage** | US equities, good depth |
| **Cost** | Free tier: 25 requests/day |
| **Integration** | REST API with JSON response |

**Other sources assessed**:

| Source | Availability | Notes |
|--------|-------------|-------|
| **SEC EDGAR** | Yes (8-K filings) | Comprehensive but requires parsing complex XBRL/JSON. Best for verification, not primary data. |
| **Polygon.io** | Free tier: very limited | Full history requires paid subscription ($29/month). Excellent data quality. |
| **Financial Modeling Prep** | Yes, free tier | 250 requests/day. Good coverage but less battle-tested. |
| **Tiingo** | Limited | Split data available for supported tickers. Less comprehensive than yfinance. |
| **IEX Cloud** | Deprecated free tier | Now requires paid plan. Previously good option. |

#### Split Detection Heuristics

To validate whether CSV data is already split-adjusted, the application should:

1. **Compare with known split dates**: Download split history from yfinance, check if the CSV shows sudden price changes at those dates
2. **Ratio analysis**: If a known 10:1 split occurred on date X, check if `price[X-1] / price[X]` is approximately 10 (unadjusted) or approximately 1 (adjusted)
3. **Volume cross-check**: After a forward split, raw volume should increase by roughly the split ratio. If volume jumps by 10x at a known 10:1 split date, the volume data is unadjusted.
4. **Price range sanity check**: Pre-split NVDA at $1,200+ (before 10:1 split in June 2024) indicates unadjusted data. Split-adjusted pre-split prices would be ~$120.

#### Recommendation

Use **yfinance as the primary source** for split data. It's free, comprehensive, well-integrated with the existing Python stack, and provides enough depth for validation purposes. Alpha Vantage as a secondary cross-reference. No need for SEC EDGAR parsing unless audit-grade accuracy is required.

---

### VQ4: Options for Obtaining Stock Dividend Information

**Question**: Please present options for obtaining stock dividend information, which affect the analysis.

#### Role of Dividend Data in the Application

Dividends affect the analysis in two ways:

1. **Total return calculation**: When computing strategy returns for benchmarking against buy-and-hold and S&P 500, dividends received while holding a position must be counted as return.
2. **Adjusted price computation**: The "adjusted close" used for return calculation incorporates dividends. If the data source provides only raw close prices, dividend data is needed to compute adjusted close.

Since the user states data will be split-adjusted, the question is whether it will also be dividend-adjusted. This depends on the eventual data source (TBD).

#### Ranked Recommendations

**Primary: yfinance `stock.dividends` + `stock.history()`**

| Attribute | Detail |
|-----------|--------|
| **Dividends** | `yf.Ticker("NVDA").dividends` returns a pandas Series of dividend amounts per share |
| **Adjusted Close** | `yf.Ticker("NVDA").history()["Close"]` returns fully adjusted close (split + dividend adjusted) |
| **Coverage** | All US exchange-listed equities with dividend history |
| **Dividend types** | Regular dividends, special/extra dividends |
| **Date type** | Ex-dividend date (the actionable date for trading) |
| **Historical depth** | Full available history (20+ years for major equities) |
| **Accuracy** | Good for regular dividends. Special dividends occasionally missing or delayed. |
| **Cost** | Free |

**NVDA dividend characteristics** (for context):
- NVDA pays a small quarterly dividend (~$0.04/share post-split)
- Dividend yield is minimal (~0.03%), so the impact on total return is negligible for NVDA
- However, other equities (e.g., AT&T, Coca-Cola) have yields of 3-6%, making dividend data critical

**Secondary: Alpha Vantage Dividends**

| Attribute | Detail |
|-----------|--------|
| **Endpoint** | `DIVIDENDS` function |
| **Coverage** | US equities, includes ex-date and payment date |
| **Cost** | Free tier: 25 requests/day |
| **Data** | Declaration date, ex-date, record date, payment date, amount |

**Other sources assessed**:

| Source | Availability | Notes |
|--------|-------------|-------|
| **Polygon.io** | Free tier: very limited | Full dividend history requires paid plan. Excellent data quality. |
| **Financial Modeling Prep** | Yes, free tier | 250 requests/day. Good for basic dividend history. |
| **SEC EDGAR** | Yes (10-K, 10-Q filings) | Comprehensive but requires parsing. Best for audit. |
| **Tiingo** | Limited | Dividend data available but less comprehensive. |
| **FRED** | No | No individual stock dividend data. Has aggregate market dividend yield. |

#### Dividend Adjustment Methodology

When the data source provides split-adjusted but NOT dividend-adjusted prices, the application needs to compute adjusted close:

**Adjustment factor calculation**:
```
For each ex-dividend date (working backwards from most recent):
  factor = (Close_before_ex - Dividend) / Close_before_ex
  Apply factor to all prices before the ex-date
```

**Total return calculation** (for a position held from entry to exit):
```
Total Return = (Exit Price - Entry Price + Dividends Received) / Entry Price
```

Where `Dividends Received` is the sum of all dividends with ex-dates between entry and exit dates.

**yfinance provides both options**:
- `history(auto_adjust=True)` returns fully adjusted prices (default behavior)
- `history(auto_adjust=False)` returns raw prices with a separate `Adj Close` column
- `dividends` property returns raw dividend amounts per share on ex-dates

#### Recommendation

Use **yfinance as the primary source** for dividend data. The library provides both raw dividends (for manual total return calculation) and fully adjusted prices (for benchmarking). Alpha Vantage as secondary fallback. The application should compute total return using raw dividends received during holding periods rather than relying solely on adjusted close, as this gives more accurate trade-by-trade PnL reporting.

---

## 2. Requirements Delta from v2

### Changes in Main.md Since v2

| Requirement | v2 Understanding | v3 (Current) | Impact |
|-------------|-----------------|---------------|--------|
| **Price data adjustment** | Raw Nasdaq CSV, unadjusted. App must apply split factors. | "Historical data will be stock-split adjusted" | Simplifies data handling. Dual-price approach changes. |
| **Data source** | Assumed Nasdaq CSV exclusively | "The source for historical data has not yet been determined" | Must design flexible DataProvider abstraction. |
| **Earnings handling** | Open question (recommended Option A: ignore) | "Ignore earning dates entirely in the analysis" | Confirmed. No earnings-date filter needed. |
| **Output conventions** | Assumed Rich terminal output | "Use standard output for analysis. Use standard error for any processing errors. Issue warnings over standard error for historical data anomalies" | stdout/stderr separation is required. Rich must output to stdout for analysis, stderr for warnings. |
| **Custom indicators** | Not addressed | "The application will not support user-supplied custom indicators" | Confirmed out of scope. Simplifies architecture. |
| **YAML config** | Recommended YAML at `~/.config/...` | "The application configuration should be YAML. Allow the user to specify the configuration location, but default to `~/.config/technical-analysis/config.yaml`" | Confirmed as recommended. |
| **Verbose mode** | Show rejected indicators with reasons | "Make indicators that produce no trades available in verbose output" | Additional detail: must explicitly list zero-trade indicators. |
| **Initial account size** | Not specified | "$10,000 default, starting point January 1, 2000" | New: configurable account size and start date. |
| **Actionable signal** | Not explicitly required | "BUY today" / "HOLD" / "SELL today" | Must output a current-day action recommendation. |
| **Drawdown trigger** | Max drawdown as scoring metric | "If the maximum acceptable drawdown is reached, sell the position" | Drawdown is now an active risk management trigger, not just a scoring metric. |
| **Cash return** | Risk-free rate recommended | "Assume zero return on cash" | Changed from v2 recommendation. Cash earns 0%. |

### Significant Impacts

1. **Data handling is simpler**: Since data will be split-adjusted, the application does not need to apply split factors to the CSV data. Split data from yfinance is needed only for validation, not correction.

2. **Dividend adjustment remains necessary**: Split-adjusted is not the same as dividend-adjusted. The application still needs dividend data for accurate total return calculations. The v2 dual-price approach simplifies to:
   - Split-adjusted prices from CSV → use for indicator calculation AND signal generation
   - Dividend data from yfinance → add to return calculations for benchmarking accuracy

3. **Cash return is zero**: v2 recommended using the risk-free rate. The user explicitly states zero return on cash. This simplifies the implementation (no FRED T-bill integration needed) but slightly changes the metrics comparison.

4. **Drawdown is an active trigger**: In v2, drawdown was a scoring/filtering metric. Now it's also a live trading rule: if position drawdown reaches the configured max (default 30%), the app must signal SELL. This is both a backtesting rule and a daily signal rule.

5. **Account size and start date**: The application needs configurable initial account size (default $10,000) and start date (default 2000-01-01) for trade profitability reporting.

---

## 3. Revised Data Handling

### v3 Data Flow (Simplified from v2)

```
CSV Input (split-adjusted) ──→ Parse & Validate ──→ Indicator Calculation
                                     │
                                     ├──→ yfinance: dividends ──→ Total Return Calculation
                                     ├──→ yfinance: SPY data ──→ Benchmark Comparison
                                     └──→ yfinance: splits ──→ Validation (optional)
```

### Price Approach (Revised from v2 Dual-Price)

| Purpose | Price Source | Notes |
|---------|-------------|-------|
| Indicator calculation | CSV split-adjusted close | Direct from input data |
| Signal generation | CSV split-adjusted close | Direct from input data |
| Trade PnL (basic) | CSV split-adjusted close | Entry/exit price difference |
| Trade PnL (total return) | CSV close + yfinance dividends | Includes dividends received during holding period |
| Benchmark comparison | yfinance adjusted close (SPY) | Apples-to-apples with adjusted SPY data |

### CSV Parsing (Updated)

The Nasdaq-format CSV handling from v2 remains valid:
1. Read CSV, parse dates with `%m/%d/%Y` format
2. Strip `$` prefix from price columns
3. Rename columns to standard: `date, open, high, low, close, volume`
4. Sort chronologically (ascending by date)
5. Set date as index

**New in v3**: Since data will be split-adjusted, skip the split-adjustment step. Add a validation step to confirm data appears split-adjusted (compare with yfinance split history if internet is available).

### Data Validation (Updated)

All validation warnings go to **stderr** (per updated requirements):

1. Ensure High >= Low for each bar (note: High >= Open and Close >= Low are not always true for gapped opens/closes)
2. Check for missing trading days (gaps beyond weekends/holidays)
3. Flag zero-volume days
4. Detect potential unadjusted splits (sudden price ratios matching known split factors)
5. Warn if data has fewer than 200 bars
6. **New**: Validate that data appears split-adjusted by cross-referencing known split dates

### Flexible Data Source Design

Since the data source is TBD, the DataProvider abstraction from v2 is even more important:

```
DataProvider (ABC)
├── CSVProvider         ← current implementation
├── APIProvider         ← future (API data source)
└── YFinanceProvider    ← for benchmark data (SPY) and supplementary data (splits, dividends)
```

Each provider should normalize data to the same internal format regardless of source.

---

## 4. S&P 500 Benchmark Data Sources

(See VQ2 answer above for full details)

### Summary Recommendation

| Priority | Source | Notes |
|----------|--------|-------|
| **Primary** | yfinance (SPY) | Free, comprehensive, Python-native, adjusted close included |
| **Secondary** | Alpha Vantage (SPY) | Requires API key, 25 req/day free tier |
| **Tertiary** | Local CSV cache | Cache downloaded data for offline resilience |
| **Cross-reference** | FRED (^SP500) | Close-only, useful for sanity checking |

### Key Decision: SPY ETF for Benchmarking

SPY is preferred over the ^GSPC index because:
- It is investable (the index itself is not)
- Its adjusted close includes dividend reinvestment
- Minimal tracking error (~0.01% annually)
- Most liquid ETF in the world (excellent data quality)
- Available from January 1993 (33+ years of history)

For periods before SPY inception (pre-January 1993), fall back to ^GSPC with the caveat that it does not include dividends.

---

## 5. Stock Split Data Sources

(See VQ3 answer above for full details)

### Summary Recommendation

| Priority | Source | Usage |
|----------|--------|-------|
| **Primary** | yfinance `stock.splits` | Validation, cross-referencing, user information |
| **Secondary** | Alpha Vantage | Backup cross-reference |
| **Tertiary** | SEC EDGAR | Audit-grade verification (if ever needed) |

### Key Change from v2

In v2, split data was **required** to adjust raw CSV prices. In v3, split data is **optional** -- used for validation and user information only, since the input data will be pre-adjusted.

---

## 6. Stock Dividend Data Sources

(See VQ4 answer above for full details)

### Summary Recommendation

| Priority | Source | Usage |
|----------|--------|-------|
| **Primary** | yfinance `stock.dividends` | Total return calculation, trade-by-trade PnL |
| **Secondary** | Alpha Vantage | Backup source for dividend data |
| **Tertiary** | yfinance adjusted close | Cross-reference for total return validation |

### Key Insight

Even though the user's CSV data will be split-adjusted, **dividend adjustment is a separate concern**. The application needs dividend data to:
1. Report accurate total return for each trade (including dividends received during holding period)
2. Compare fairly against SPY benchmark (which uses dividend-adjusted returns)
3. Report accurate buy-and-hold return (must include dividends for fair comparison)

For low-yield stocks like NVDA (~0.03% yield), dividends barely matter. For high-yield stocks (e.g., utilities, REITs at 3-6% yield), omitting dividends would significantly understate buy-and-hold returns and make the active strategy look artificially better.

---

## 7. Updated Stack Recommendation

### Changes from v2

| Component | v2 Recommendation | v3 Recommendation | Change Reason |
|-----------|-------------------|-------------------|---------------|
| **Language** | Python (assumed) | Python (confirmed with analysis) | Vendor question analysis confirms Python superiority for this use case |
| **Cash return** | Risk-free rate from FRED | Zero (per requirement) | User explicitly stated "Assume zero return on cash" |
| **FRED integration** | Needed (T-bill data) | **Removed** | No longer needed since cash return is zero |
| **Data adjustment** | Full split-adjustment pipeline | Validation-only pipeline | Input data is pre-split-adjusted |

### v3 Stack (Final)

| Component | Library | Version/Status | Role |
|-----------|---------|---------------|------|
| Language | **Python 3.10+** | -- | Minimum version for modern type hints |
| Package Manager | **uv** | Active | Fast dependency resolution, lock files |
| CLI Framework | **Typer** | Active | Subcommands, --verbose flag, type-hint based |
| Indicators | **TA-Lib** | v0.6.8 (Oct 2025), Active | 150+ indicators, C speed |
| Backtesting | **backtesting.py** | v0.6.5, Active | Walk-forward built-in, Bokeh charts |
| Benchmark Data | **yfinance** | Active | SPY data, split/dividend history |
| Terminal Output | **Rich** | v14.2.0 (Jan 2026), Active | Color tables, progress bars |
| Configuration | **Pydantic Settings** | Active | Type-safe YAML config + CLI override |
| Testing | **pytest + hypothesis** | Active | Unit, integration, property-based |

**Removed from stack**: FRED integration (no longer needed for cash returns).

### Configuration Parameters (Updated from v2)

```yaml
analysis:
  trade_frequency_days: 30
  max_drawdown_percent: 30
  max_indicator_combinations: 5
  min_data_points: 200
  initial_account_size: 10000
  start_date: "2000-01-01"

backtesting:
  method: walk_forward
  train_window_years: 3
  test_window_months: 6
  reoptimize_frequency: monthly
  mean_reversion_mode: false  # ranging market behavior

benchmarks:
  sp500_ticker: "SPY"
  cash_return: 0  # zero per requirement
  alpha_vantage_api_key: ""  # optional fallback

output:
  verbose: false
  csv_export: true
  csv_export_path: ./output/

data:
  expect_split_adjusted: true
  validate_splits: true  # cross-reference with yfinance
```

---

## 8. Revised Design Recommendations

Updates and additions to the v2 design recommendations:

1. **Use split-adjusted CSV data directly** for indicator calculation and signal generation. No split-adjustment pipeline needed (simplified from v2).

2. **Obtain dividend data from yfinance** for total return calculations. Include dividends received during holding periods in trade-by-trade PnL.

3. **Cash earns zero return** (changed from v2 risk-free rate recommendation per user requirement).

4. **Implement active drawdown monitoring**: If a position's drawdown from peak reaches the configured max (default 30%), generate a SELL signal immediately. This is both a backtesting rule and a live signal rule.

5. **Output to stdout for analysis, stderr for warnings/errors**: Use Rich for formatting on stdout. Print data validation warnings and processing errors to stderr.

6. **Include zero-trade indicators in verbose output**: When an indicator combination produces no trades in the analysis period, report it in verbose mode with the reason (e.g., "no entry signal triggered" or "ADX always below threshold").

7. **Configurable account size and start date**: Default $10,000 and January 1, 2000. These affect trade profitability reporting and PnL calculations.

8. **Generate actionable daily signal**: The output must include one of: "BUY today", "HOLD", or "SELL today" based on current indicator state.

9. **Implement SPY benchmark data with fallback chain**: yfinance → Alpha Vantage → local cache → graceful error.

10. **Validate split adjustment**: Optionally cross-reference CSV data with yfinance split history to confirm data is actually split-adjusted. Warn on stderr if discrepancies are detected.

11. **All v2 recommendations not explicitly changed above remain in effect**: Walk-forward analysis, composite scoring, anti-overfitting measures, ADX regime detection, Chandelier Exit, hierarchical filtering, etc.

---

## 9. Status of v2 Open Questions

### Resolved by Updated Requirements

| # | Question | Resolution |
|---|----------|------------|
| 5 | CSV data: split-adjusted? | **Yes** -- "Historical data will be stock-split adjusted" |
| 5 | Will other CSVs follow same format? | **TBD** -- "The source for historical data has not yet been determined" |
| 7 | Earnings date handling | **Ignore entirely** -- "Ignore earning dates entirely in the analysis" |
| 8 | Terminal vs web output | **Terminal** -- "Use standard output for analysis" + Rich for formatting |
| 9 | Config file location | **Confirmed** -- `~/.config/technical-analysis/config.yaml`, user-specifiable |
| 10 | Data validation behavior | **Warnings to stderr** -- "Issue warnings over standard error for historical data anomalies" |

### Still Open (Carried to v3 Questions)

| # | Question | Status |
|---|----------|--------|
| 1 | Python version / uv setup | Still open -- recommend 3.10+ with uv |
| 2 | TA-Lib C library installation | Still open -- needed regardless of language choice |
| 3 | backtesting.py vs vectorbt | Still open -- research strongly recommends backtesting.py |
| 6 | Ranging market behavior | Still open -- recommend default to cash |
| 11 | Internet requirement for split/dividend data | Still open -- yfinance needed for dividends and benchmarking |

### Newly Answered by Vendor Questions

| VQ# | Question | Answer |
|-----|----------|--------|
| VQ1 | Python vs Go/Rust | Python confirmed as correct choice |
| VQ2 | S&P 500 data source | yfinance (SPY) primary, Alpha Vantage secondary, local cache fallback |
| VQ3 | Stock split data | yfinance (validation only, since CSV is pre-adjusted) |
| VQ4 | Stock dividend data | yfinance (for total return calculation) |

---

## Unchanged from v2

The following sections from Research v2 remain fully valid and are not repeated here. Refer to `research/Research_v2.md` for:

- Section 1: Indicator Recommendations (Williams %R, RSI, MACD, MAs, Bollinger Bands, ADX, OBV, ATR)
- Section 2: Signal Rules for Long-Only Trading
- Section 3: Exit Strategies (Chandelier Exit primary)
- Section 4: Indicator Combinations (top 5)
- Section 5: ADX Regime Detection
- Section 6: Composite Scoring Weights
- Section 7: Backtesting Methodology (walk-forward analysis)
- Section 10: Pattern Recognition
- Section 13: Anti-Overfitting Measures
- Section 14: Equity-Specific Considerations
