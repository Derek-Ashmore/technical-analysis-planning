# Research Questions v2

Additional requests for information and clarifications remaining after the user answered the critical and important questions from v1.

---

## Status of v1 Questions

| # | Question | Status | Answer |
|---|----------|--------|--------|
| 1 | Asset class | **Answered** | Equities only |
| 2 | Data source | **Answered** | CSV (Nasdaq format) |
| 3 | Volume data | **Answered** | Available in CSV |
| 4 | Long/Short | **Answered** | Long-only |
| 5 | Trade frequency | **Answered** | Configurable, default 30 days |
| 6 | Transaction costs | **Answered** | Not material |
| 7 | Risk tolerance | **Answered** | Max drawdown configurable, default 30% |
| 8 | Single/portfolio | **Answered** | Single asset at a time |
| 9 | Reporting format | **Partially answered** | Verbose mode + trade-by-trade. Terminal + CSV assumed. |
| 10 | Configurable days | **Answered** | Target trading frequency (not lookback window) |
| 11 | Real-time/batch | **Answered** | Run once daily |
| 12 | Language preference | **Not answered** | Python assumed (best ecosystem for this domain) |
| 13 | "Best fit" meaning | **Answered** | "Indicates buys and sells most profitably" |
| 14 | Benchmark | **Answered** | Buy-and-hold + S&P 500 |
| 15 | Data quality | **Deferred** | Assess when data pipeline built |
| 16 | Asset characteristics | **Deferred** | Assess per-equity |

---

## New Questions for v2

### Design Decisions (Needed Before Coding)

#### 1. Python version and environment

- The research recommends **uv** for package management with **pyproject.toml**. Is this acceptable?
- What Python version should be the minimum? (3.10+ recommended for modern type hints)
- Is there an existing Python environment or should we create a fresh one?

#### 2. TA-Lib C library installation

- TA-Lib requires a C library installed at the system level before the Python wrapper can be installed. This is a one-time setup step but can be friction.
- Is TA-Lib acceptable given this requirement, or would a pure-Python alternative (at the cost of performance) be preferred?
- Should we provide installation scripts for the C library?

#### 3. Backtesting library: backtesting.py vs vectorbt

- The research recommends **backtesting.py** over vectorbt because:
  - vectorbt free version is in maintenance-only mode (dev shifted to paid PRO)
  - backtesting.py has built-in walk-forward optimization
  - Simpler API for this use case
- However, vectorbt is faster and more flexible for advanced use cases.
- Is backtesting.py acceptable, or is there a preference for vectorbt?

#### 4. S&P 500 benchmark data source

- The research recommends downloading **SPY** adjusted close data from **yfinance** for S&P 500 benchmarking.
- yfinance requires internet access and uses Yahoo Finance's API (personal/research use).
- Is this acceptable? Or should S&P 500 data also be provided as CSV?

#### 5. Example CSV data characteristics

- The provided NVDA.csv appears to be from Nasdaq's historical data export.
- Is the data already split-adjusted, or is it raw (unadjusted)?
- How many years of history does the full file contain?
- Will other equity CSVs follow the same format (Nasdaq export)?
- We noticed the CSV is sorted newest-first (reverse chronological). Should the application expect this consistently?

---

### Clarification Questions (Can Answer During Development)

#### 6. Ranging market behavior

- When the market is detected as ranging (ADX < 20), the research recommends **defaulting to cash** (no position).
- An alternative is to use mean-reversion signals (buy dips, sell at resistance) in ranging markets.
- **Preference**: Default to cash, or default to mean-reversion? (Research recommends cash with configurable mean-reversion option.)

#### 7. Earnings date handling

- The research notes that earnings announcements cause large price gaps that can trigger false signals.
- **Options**:
  - A) Ignore earnings dates entirely (let indicators respond naturally to price moves)
  - B) Flag signals near earnings dates with a warning
  - C) Optionally suppress signals within N days of known earnings dates
- **Preference**: Option A is simplest and recommended for v1. Confirm this is acceptable.

#### 8. Output preferences

- The research recommends **Rich** (terminal library) for primary output with **CSV export** as secondary.
- Is terminal-based output acceptable, or is a web-based dashboard preferred?
- Should the application generate any charts/visualizations? (backtesting.py can produce interactive Bokeh charts)

#### 9. Configuration file location

- The research recommends a YAML config file at `~/.config/technical-analysis/config.yaml` with CLI argument overrides.
- Is this location acceptable, or should the config live in the project directory?

---

### Data-Specific Questions (Assess When Building Pipeline)

#### 10. CSV data validation rules

When we build the data pipeline, we need to validate the CSV data. What should happen when validation detects issues?
- Missing trading days (gaps beyond weekends/holidays): Skip and warn? Interpolate?
- Zero-volume days: Flag and continue? Exclude from analysis?
- Price anomalies (High < Low, negative prices): Reject the file? Skip the bar?
- Insufficient data (< 200 bars for 200-day SMA): Warn and reduce indicator set? Error out?

#### 11. Historical dividend and split data

- The application needs split data to adjust raw CSV prices. The research recommends using yfinance `stock.splits` for this.
- For return calculation, dividend data is also needed (yfinance `stock.dividends`).
- Is it acceptable to require internet access for this supplementary data? Or should split/dividend data also be providable via CSV?

---

## Summary of Priority

| Priority | Questions | Reason |
|----------|-----------|--------|
| **Answer before coding** | #1 (Python/uv), #2 (TA-Lib), #3 (backtesting lib), #4 (SPY data), #5 (CSV format) | Technology and data pipeline decisions |
| **Can answer during development** | #6 (ranging behavior), #7 (earnings), #8 (output), #9 (config location) | Implementation details with sensible defaults |
| **Assess when building pipeline** | #10 (validation), #11 (split/dividend data) | Data-dependent |
