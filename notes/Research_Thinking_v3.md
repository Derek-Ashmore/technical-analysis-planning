# Research Thinking v3

This document describes the reasoning process and thinking behind the v3 research revision, including the vendor question analysis and requirements delta assessment.

---

## What Changed from v2

The user updated `requirements/Main.md` with answers to several open questions from v2, and added `instructions/VendorQuestions.md` with four specific questions requiring research.

### Requirements Changes

| Change | v2 → v3 | Reasoning Impact |
|--------|---------|-----------------|
| Data is split-adjusted | Raw → pre-adjusted | Major simplification. No split-adjustment pipeline needed. |
| Data source TBD | Nasdaq CSV assumed → undetermined | Reinforces need for DataProvider abstraction. |
| Cash return zero | Risk-free rate recommended → zero | Removes FRED dependency. Simplifies implementation. |
| Earnings: ignore | Recommended → confirmed | No earnings filter needed. |
| Output to stdout/stderr | Assumed terminal → explicit conventions | Rich must target stdout; warnings to stderr. |
| No custom indicators | Not addressed → explicitly excluded | Reduces scope. |
| Active drawdown trigger | Scoring metric → live trading rule | Drawdown monitor is now a real-time feature. |
| Account size + start date | Not specified → $10,000, Jan 1 2000 | New configuration parameters. |
| Actionable signal required | Implied → explicit (BUY/HOLD/SELL today) | Must generate daily recommendation. |

---

## Vendor Question Analysis Approach

### Three Parallel Research Agents

We decomposed the vendor questions into three parallel research workstreams:

1. **Language/Platform Research** (language-researcher) -- Python vs Go vs Rust comparison for this specific application, focusing on concrete library availability rather than abstract language characteristics.

2. **S&P 500 Data Research** (sp500-researcher) -- Comprehensive survey of free data sources for S&P 500 benchmarking, including API-based and downloadable options.

3. **Splits & Dividends Research** (splits-dividends-researcher) -- Survey of stock split and dividend data sources, detection techniques, and adjustment methodology.

### Compilation Approach

As with v2, the team lead compiled all findings into a single cohesive document rather than delegating to a fourth agent.

---

## Key Decisions and Reasoning

### Why Python Is the Correct Choice (VQ1)

The vendor question about Go/Rust was reasonable -- a compiled binary eliminates Python runtime friction. However, the analysis revealed a fundamental ecosystem gap, not just a maturity difference:

1. **No walk-forward backtesting framework** exists in Go or Rust. This alone would add 2-4 weeks of development time, as walk-forward analysis is architecturally complex (sliding windows, parameter optimization within windows, out-of-sample validation, efficiency calculation).

2. **Technical indicator coverage is thin**: Go's best option (go-talib) still requires the C library via CGo, negating the "no runtime" advantage. Rust's ta-rs covers only ~30 indicators vs TA-Lib's 150+.

3. **No yfinance equivalent**: Both Go and Rust would require building custom HTTP clients for Yahoo Finance, handling API changes, parsing responses, and managing rate limiting -- all of which yfinance already handles.

4. **The performance argument is moot**: This application processes 2,500 bars once per day. Even pure Python (without TA-Lib's C core) could handle this in seconds. The actual compute-intensive work (indicator calculation) runs at C speed through TA-Lib regardless of the orchestration language.

5. **Development time**: Building the missing infrastructure in Go/Rust would roughly double the development timeline for no functional benefit.

The correct mitigation for Python deployment friction is operational, not architectural: provide a Dockerfile, use uv for dependency management, and optionally use conda-forge for TA-Lib installation.

### Why SPY Over ^GSPC for Benchmarking (VQ2)

This might seem like a minor decision, but it has subtle implications:

- **^GSPC is a price-only index**: It does not include dividend reinvestment. S&P 500's historical ~10% annual return includes ~2% from dividends. Using ^GSPC as a benchmark would systematically understate market returns by ~2%/year, making active strategies look better than they actually are relative to the market.

- **SPY is investable**: A benchmark should represent something the investor could actually buy. You can't buy ^GSPC directly. SPY (or VOO, IVV) is the investable equivalent.

- **SPY's adjusted close from yfinance includes dividends**: This means `yf.download("SPY")["Adj Close"]` gives the correct total return series with zero additional work.

- **Tracking error is negligible**: SPY's expense ratio is 0.0945% and tracking error vs S&P 500 is ~0.01% annually. For our purposes, this is indistinguishable from the index.

We chose SPY over VOO (Vanguard) because SPY has history from 1993 vs VOO's 2010, giving 17 more years of benchmark data.

### Why Split Data Role Changed (VQ3)

In v2, the application had to:
1. Download split history from yfinance
2. Apply split adjustment factors to raw CSV prices
3. Adjust volume inversely
4. Handle compounding adjustments for multiple splits

In v3, since the CSV data will be split-adjusted, the split data role reduces to:
1. Validate that the CSV data is actually split-adjusted (optional, defensive)
2. Inform the user about significant corporate actions
3. Cross-reference if data source changes in the future

This is a significant simplification. The split-adjustment pipeline was one of the more error-prone parts of the v2 design (compounding adjustments, volume inversion, detection of already-adjusted data). Removing it as a requirement reduces development risk.

However, we still recommend downloading split data for validation. If a user provides data from a new source that turns out to be unadjusted, the application should detect this and warn rather than produce silently wrong results.

### Why Dividend Data Is Still Critical (VQ4)

"Split-adjusted" and "dividend-adjusted" are different things:

- **Split-adjusted**: Prices corrected for stock splits so the historical series is continuous. A $1,200 pre-10:1-split price becomes $120 in the adjusted series.
- **Dividend-adjusted**: Prices reduced by the ex-dividend amount on ex-dates, so the total return is reflected in price changes alone.

The user states data will be split-adjusted. They did NOT say dividend-adjusted. This means:
- Indicator calculation: Fine, since indicators should use split-adjusted (not dividend-adjusted) prices anyway
- Return calculation: Needs dividend data to compute accurate total returns

For NVDA specifically (dividend yield ~0.03%), this barely matters. But for other equities (REITs, utilities, telecoms with 3-6% yields), omitting dividends from return calculations would:
- Understate buy-and-hold returns by 3-6% annually
- Make the active strategy look better than it actually is
- Produce misleading Sharpe ratios and alpha calculations

### Why Cash Return Is Zero (Requirement Change)

v2 recommended using the risk-free rate (3-month T-bill from FRED). The user explicitly stated "Assume zero return on cash."

This is actually more conservative for the active strategy. When the strategy is out of the market (ranging markets, after sells), it earns nothing. With a risk-free rate, the strategy would get ~4-5% annualized on cash, which flatters its risk-adjusted returns.

The user's choice means:
- Strategy returns during flat periods = 0% (more conservative)
- No FRED integration needed (simpler implementation)
- The "time in market" metric becomes even more important to report (since being out of market has a real opportunity cost of missing potential gains)

### Active Drawdown Trigger (New Requirement)

The Main.md states: "If the maximum acceptable drawdown is reached, sell the position."

This is different from v2's approach where drawdown was a scoring/filtering metric only. Now it's also a **live trading rule**:

1. **During backtesting**: When a position's drawdown from its peak reaches the configured max (default 30%), force an exit at the next available close price. This affects backtest results.
2. **During daily signal generation**: If the current open position's drawdown from peak exceeds 30%, output "SELL today" regardless of what indicators say.
3. **Precedence**: The drawdown trigger overrides indicator signals. Even if all indicators say HOLD, a 30% drawdown forces a SELL.

This is a capital preservation mechanism. It's the right default for a long-only equity strategy where a single catastrophic loss can destroy years of gains.

---

## What Surprised Us in v3

### Go's TA-Lib Wrapper Still Needs C

The go-talib library uses CGo (Go's C interop), which means it still requires the TA-Lib C library installed system-wide. This completely negates Go's "single binary deployment" advantage for this application. If you need TA-Lib (and you do, for 150+ indicators), you need the C library regardless of whether the orchestration is Python, Go, or Rust.

### The Data Source Being TBD Changes the Architecture Priority

In v2, we knew the data source (Nasdaq CSV) and could design a specific pipeline. With the data source now TBD, the DataProvider abstraction moves from "nice architectural pattern" to "critical requirement." The application must be able to accept data from:
- Nasdaq-format CSV (current example)
- Other CSV formats (different column names, date formats, price formats)
- APIs (future roadmap)

This means the CSV parsing logic should be configurable (column mapping, date format, price format) rather than hardcoded for Nasdaq format.

### Zero Cash Return Makes the Strategy Comparison Tougher

With zero cash return, an active strategy that sits in cash 70% of the time (common in ADX-based regime detection) needs to earn significantly more during its 30% in-market time to beat buy-and-hold. For example, if buy-and-hold returns 10% annually and the strategy is in the market 30% of the time, it needs to earn ~33% annualized during its active periods just to match buy-and-hold. This is a high bar and makes the composite scoring and strategy selection more challenging.

---

## Risks and Concerns

### Internet Dependency

The application now relies on internet access for:
1. SPY benchmark data (yfinance)
2. Dividend data for total return calculations (yfinance)
3. Split data for validation (yfinance, optional)

This is unavoidable for benchmarking but creates a point of failure. The fallback chain (yfinance → Alpha Vantage → local cache) mitigates this, but the first run always requires internet.

### yfinance Stability

yfinance is an unofficial Yahoo Finance wrapper that has historically broken when Yahoo changes their API. The library is well-maintained and typically adapts quickly (within days), but there's inherent risk in depending on unofficial API access. Mitigations:
- Local cache for resilience
- Alpha Vantage as secondary source
- Abstract data access behind DataProvider for easy swap

### Data Source Ambiguity

The data source being "not yet determined" is both a flexibility and a risk. If the eventual data source has a different format, date convention, or adjustment status, the application will need adaptation. The DataProvider abstraction helps, but there may be edge cases we can't anticipate without knowing the specific source.

### Dividend Data Completeness

yfinance dividend data is generally reliable for regular dividends on major US equities. However:
- Special dividends may be missing or delayed
- Stock dividends (paid in shares, not cash) may not be fully captured
- REITs and other entities with complex distribution structures may have incomplete data
- Very small dividends (like NVDA's $0.04/quarter) are sometimes rounded or missing

For this application's initial target (equities like NVDA), this is unlikely to matter. But for broader use, dividend data should be cross-validated with a second source.
