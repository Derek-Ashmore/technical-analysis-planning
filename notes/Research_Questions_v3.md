# Research Questions v3

Additional requests for information and clarifications remaining after the v3 research revision.

---

## Status of All Prior Questions

### v1 Questions

| # | Question | Status | Resolution |
|---|----------|--------|------------|
| 1 | Asset class | **Answered** | Equities only |
| 2 | Data source | **Partially answered** | CSV, but source TBD |
| 3 | Volume data | **Answered** | Available in CSV |
| 4 | Long/Short | **Answered** | Long-only |
| 5 | Trade frequency | **Answered** | Configurable, default 30 days |
| 6 | Transaction costs | **Answered** | Not material |
| 7 | Risk tolerance | **Answered** | Max drawdown configurable, default 30% |
| 8 | Single/portfolio | **Answered** | Single asset |
| 9 | Reporting format | **Answered** | stdout for analysis, stderr for errors/warnings |
| 10 | Configurable days | **Answered** | Target trading frequency |
| 11 | Real-time/batch | **Answered** | Once daily |
| 12 | Language preference | **Answered** | Python (confirmed by vendor question analysis) |
| 13 | "Best fit" meaning | **Answered** | Most profitable buys and sells |
| 14 | Benchmark | **Answered** | Buy-and-hold + S&P 500 |
| 15 | Data quality | **Partially answered** | Warnings to stderr. Behavior details still open. |
| 16 | Asset characteristics | **Deferred** | Per-equity assessment |

### v2 Questions

| # | Question | Status | Resolution |
|---|----------|--------|------------|
| 1 | Python version / uv | **Open** | See v3 Q1 below |
| 2 | TA-Lib C library | **Open** | See v3 Q2 below |
| 3 | backtesting.py vs vectorbt | **Open** | See v3 Q3 below |
| 4 | S&P 500 data source | **Answered** | yfinance (SPY) primary. See Research_v3.md VQ2. |
| 5 | CSV data characteristics | **Mostly answered** | Split-adjusted, source TBD |
| 6 | Ranging market behavior | **Open** | See v3 Q4 below |
| 7 | Earnings handling | **Answered** | Ignore entirely |
| 8 | Output preferences | **Answered** | stdout for analysis, Rich terminal formatting |
| 9 | Config location | **Answered** | `~/.config/technical-analysis/config.yaml`, user-specifiable |
| 10 | Validation rules | **Partially answered** | Warnings to stderr. Specific behaviors still open. |
| 11 | Internet for split/dividend data | **Open** | See v3 Q5 below |

### Vendor Questions

| # | Question | Status | Resolution |
|---|----------|--------|------------|
| VQ1 | Python vs Go/Rust | **Answered** | Python confirmed. See Research_v3.md VQ1. |
| VQ2 | S&P 500 data sources | **Answered** | yfinance (SPY) primary. See Research_v3.md VQ2. |
| VQ3 | Stock split data | **Answered** | yfinance for validation. See Research_v3.md VQ3. |
| VQ4 | Stock dividend data | **Answered** | yfinance for total return. See Research_v3.md VQ4. |

---

## Remaining Questions for v3

### Design Decisions (Needed Before Coding)

#### 1. Python version and environment setup

- Research recommends **Python 3.10+** (minimum for modern type hints like `X | Y` union syntax) with **uv** for package management.
- Is Python 3.10 as a minimum version acceptable? (3.12 is current stable, 3.10 is oldest still-supported in Feb 2026.)
- Should the project use `pyproject.toml` with uv, or is there a different preference?
- Is a Dockerfile desired for deployment?

#### 2. TA-Lib C library dependency

- TA-Lib requires a C library (`ta-lib`) installed system-wide. This is unavoidable given the need for 150+ indicators at C speed.
- **Installation options**:
  - `conda install -c conda-forge ta-lib` (easiest, no manual compilation)
  - `sudo apt install libta-lib-dev` + `pip install TA-Lib` (Ubuntu/Debian)
  - Build from source (most portable, most friction)
  - Dockerfile with pre-installed TA-Lib (zero-friction for Docker users)
- **Question**: Which installation approach is preferred? Is conda available, or should we provide a build script?
- **Alternative**: Use `pandas-ta` (pure Python, no C dependency) at the cost of ~10x slower indicator calculation. However, pandas-ta is at risk of archival by July 2026. Not recommended.

#### 3. Backtesting library confirmation

- Research recommends **backtesting.py** over vectorbt because:
  - Walk-forward optimization is built in
  - Actively maintained (v0.6.5)
  - Simpler API for this use case
  - vectorbt free version is maintenance-only (dev shifted to paid PRO)
- **Question**: Is backtesting.py acceptable? Any objections to this choice?

---

### Clarification Questions (Can Answer During Development)

#### 4. Ranging market behavior (carried from v2)

- When ADX < 20 (ranging market), the research recommends **defaulting to cash** (no position).
- Alternative: Use mean-reversion signals (RSI + Bollinger Bands) to buy dips.
- **Research recommendation**: Default to cash, with mean-reversion as a configurable option.
- **Question**: Is this approach acceptable?

#### 5. Internet access requirement

- The application requires internet access for:
  - SPY benchmark data (yfinance)
  - Dividend data for total return calculations (yfinance)
  - Split validation (yfinance, optional)
- First run always requires internet. Subsequent runs can use cached data.
- **Question**: Is internet access acceptable? Or should benchmark/dividend data also be providable as CSV?

#### 6. Data validation behavior specifics

- The requirements state: "Issue warnings over standard error for historical data anomalies."
- **Open specifics**:
  - Missing trading days: Skip and warn? Or interpolate?
  - Zero-volume days: Include in analysis with warning? Or exclude?
  - Price anomalies (High < Low): Reject the entire file? Skip just that bar?
  - Insufficient data (< 200 bars): Warn and reduce indicator set? Or error out?
- **Research recommendation**: Skip/warn for anomalies, error for critical issues (< 200 bars), never interpolate.

#### 7. CSV format flexibility

- The example data (NVDA.csv) uses Nasdaq format: `$`-prefixed prices, `MM/DD/YYYY` dates, `Close/Last` column name, reverse chronological order.
- Since the data source is TBD, should the application handle other CSV formats (different column names, date formats, no `$` prefix)?
- **Research recommendation**: Support a configurable column mapping in the YAML config, with Nasdaq format as the default.

---

## Summary of Priority

| Priority | Questions | Reason |
|----------|-----------|--------|
| **Answer before coding** | #1 (Python/uv), #2 (TA-Lib installation), #3 (backtesting lib) | Environment and dependency setup |
| **Can answer during development** | #4 (ranging behavior), #5 (internet access), #6 (validation), #7 (CSV flexibility) | Implementation details with sensible defaults |

---

## Comparison: Questions Remaining Over Time

| Version | Total Questions | Answered | Remaining |
|---------|----------------|----------|-----------|
| v1 | 16 | 0 | 16 |
| v2 | 11 new + 16 prior = 27 total | 14 | 13 |
| v3 | 7 new + 13 prior = 20 total | 13 | 7 |

The question set is converging. The 7 remaining questions are implementation-level decisions that can be answered incrementally. The 3 "answer before coding" questions are primarily about development environment setup, not architectural decisions.
