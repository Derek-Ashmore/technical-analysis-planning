# Research Thinking v3: Methodology, Reasoning, and Decision-Making

This document describes the reasoning process, key decisions, confidence assessments, and trade-offs behind the v3 research revision. It is the companion "thinking" document to `research/Research_v3.md`.

---

## 1. Revision Context

### What Triggered the Revision

Research v2 was conducted with most scope questions answered but left 5 "must answer before implementation" questions (NQ1.1, NQ1.2, NQ3.1, NQ3.2, NQ7.2) and several "should answer during design" questions open. The updated requirements in `requirements/Main.md` resolve all 5 critical questions and introduce new constraints that invalidate portions of the v2 architecture.

Additionally, 4 vendor questions (`instructions/VendorQuestions.md`) require new research into platform choice, data sources, and corporate actions handling.

### What Changed Between v2 and v3 Requirements

| Dimension | v2 Assumption | v3 Confirmed Scope | Impact |
|-----------|---------------|-------------------|--------|
| Execution model | Dual-mode (full + daily), stateful | **Stateless, single mode** | Eliminates state directory, daily mode, re-optimization scheduling, cache |
| Output | HTML reports (Plotly, Jinja2, mplfinance) | **stdout text, stderr for errors/warnings** | Removes 4 dependencies, simplifies reporting to text formatting |
| Earnings handling | 3x ATR gap detection, signal suppression | **Ignore entirely** | Removes earnings gap detection, suppression logic, gap trade reporting |
| Starting capital | Not specified | **$10,000 default** | Now explicit in config defaults |
| Start date | Not specified | **January 1, 2000 default** | Now explicit in config defaults |
| Transaction costs | 0.02% token cost | **0% default ("not material")** | User explicitly stated not material; configurable |
| Drawdown behavior | Staged circuit breaker (15/20/25%) | **Hard stop: "If max DD reached, sell"** | Binary behavior replaces staged response |
| Position sizing | Recommended 100% but not confirmed | **"All long positions will use the entire balance"** | Confirmed 100% in/out |
| Cash return | Recommended 0% but not confirmed | **"Assume zero return on cash"** | Confirmed |
| Data source | Nasdaq CSV specifically | **CSV, source undetermined** | Must support flexible formats, not just Nasdaq |
| Config location | Generic | **`~/.config/technical-analysis/config.yaml`** | XDG-compliant path specified |
| Custom indicators | Deferred | **"Out of scope"** | Confirmed: no custom indicators |
| Verbose mode | Implied | **"Verbose mode that documents examined and rejected indicators"** | Explicit requirement with specific output expectations |
| No-trade indicators | Not addressed | **"Make indicators that produce no trades available in verbose output"** | New explicit requirement |

The net effect is a significant simplification. v2 was designing for a stateful system with HTML output. v3 is designing for a stateless CLI tool with text output. This is not a reduction in capability -- the analytical core is identical. The simplification is entirely in the infrastructure layer.

### Vendor Questions as New Research Dimension

v3 introduces a new research dimension not present in v1 or v2: explicit technology and data source evaluation requested by the user. The 4 vendor questions are:

1. **VQ1:** Python vs Go/Rust for runtime requirements
2. **VQ2:** Historical S&P 500 index data sources
3. **VQ3:** Stock split information sources
4. **VQ4:** Stock dividend information sources

These required web research into current (2026) API availability, pricing, and capabilities -- not just theoretical analysis.

---

## 2. Research Approach

### Workstream Structure

The v3 research was conducted through three parallel workstreams:

**Workstream 1: Requirements Diff Analysis**

Systematic comparison of v2 assumptions against updated `requirements/Main.md` to identify every invalidated assumption, every confirmed assumption, and every new requirement not present in v2. This workstream produced the "What Changed" tables in Section 3 of the research document.

**Workstream 2: Python vs Go/Rust Evaluation**

Comprehensive comparison across six dimensions: runtime requirements, library ecosystem, performance at scale, development time, deployment options, and future evolution path. This workstream addressed VQ1.

**Workstream 3: Financial Data Sources**

Web research into current (2026) availability and capabilities of financial data APIs for S&P 500 benchmark data, stock split information, and stock dividend information. This workstream addressed VQ2, VQ3, and VQ4. Key discovery: IEX Cloud shut down August 31, 2024, requiring removal from all prior data source recommendations.

### Research Method for Vendor Questions

The vendor questions required different methodology from v1/v2 research (which was primarily analytical reasoning about technical indicators and backtesting frameworks). VQ1 required comparative technology assessment. VQ2-VQ4 required current-state API and data source evaluation with attention to recent changes (e.g., yfinance v0.2.51 behavior change in December 2024, IEX Cloud shutdown in August 2024).

Web searches were conducted for each data source to verify current availability, pricing, API capabilities, and known limitations as of February 2026.

---

## 3. Key Decisions and Reasoning

### 3.1 Stateless Execution: The Most Impactful Change

#### Why Stateless Simplifies Everything

The updated requirements state: "The application is stateless and will not require persistence between runs. I understand that results may change when new data is analyzed."

This single statement eliminates approximately 30-40% of the v2 design surface:

- **State directory** (position.json, trades.csv, last_run.json, cache/) -- eliminated
- **Daily mode vs full mode** -- eliminated (only one mode: full analysis every run)
- **Re-optimization scheduling** (quarterly, 63-day interval) -- eliminated (re-optimize every run)
- **Computation caching** (pyarrow) -- eliminated (no cache between runs)
- **Cross-run position tracking** -- eliminated (position derived from signal history each run)
- **Strategy locking** -- eliminated (no locked-in strategy between runs)

The stateless model transforms the application into a pure function: `f(config, historical_data) -> report + signal`. Every invocation is self-contained. This is architecturally cleaner and eliminates an entire category of bugs related to stale state, corrupted state files, and state/data version mismatches.

#### The Signal Determination Challenge

The one complication of stateless execution is deriving the current signal (BUY today / HOLD / SELL today) without knowing the current position. In v2's stateful model, the application tracked whether it was currently "in" or "out" of a position. In v3, this must be derived from the data itself.

The solution: replay the top strategy's signals across the entire historical period, find the most recent signal transition (last BUY date vs last SELL date), and determine the current state from that. If the last BUY is more recent than the last SELL, the strategy considers itself "in position." If the last SELL is more recent, it considers itself "out."

This is computationally trivial (the signal history is already computed during backtesting) but conceptually important. It means the "current signal" is deterministic given the data and strategy parameters -- two runs on the same data will always produce the same signal.

#### Indicator Instability is Accepted, Not Fought

v2 identified a tension (Tension 6 in Thinking v2) about the daily signal mode assuming stationarity. With stateless execution, the user explicitly accepts that "results may change when new data is analyzed." This resolves the tension: if the top strategy changes between runs, that is expected behavior, not a bug.

The v2 recommendation for consensus reporting (showing agreement among top 5 strategies) remains valid and is incorporated into the v3 stdout report structure.

### 3.2 stdout Output: Unix Philosophy

#### Why Text Output is Simpler and More Composable

The updated requirements specify: "Use standard output for analysis. Use standard error for any processing errors. Issue warnings over standard error for historical data anomalies."

This is classic Unix philosophy: stdout for results, stderr for diagnostics. It enables:

- **Piping:** `ta-analyze NVDA.csv | grep "Signal:"` to extract just the signal
- **Redirection:** `ta-analyze NVDA.csv > report.txt 2> warnings.log` to separate output streams
- **Scripting:** Parse stdout programmatically for automated trading workflows
- **Simplicity:** No HTML template engine, no chart rendering, no browser dependency

The trade-off is losing visual charts (Plotly interactive, mplfinance static). However, the requirements note: "In future, I may expand output options beyond terminal output." This leaves the door open for chart generation as a future addition without requiring it now.

#### What Gets Removed

Removing HTML output eliminates 4 dependencies:
- **Plotly 5.18+** (large dependency tree, ~25MB installed)
- **Jinja2 3.1+** (template engine)
- **mplfinance 0.12.10+** (matplotlib-based candlestick charts)
- **pyarrow 14+** (was used for caching, not directly output-related, but also eliminated by stateless)

This reduces total major dependencies from 12 to 8, which meaningfully reduces installation complexity and potential dependency conflicts.

#### Report Structure Design

The stdout report structure was designed to be:
1. **Scannable** -- Key information (top strategy, current signal) appears early
2. **Parseable** -- Consistent formatting allows grep/awk extraction
3. **Complete** -- All trade-by-trade data included (not truncated)
4. **Progressive** -- Summary first, details later, verbose at the end

The `--- Section Name ---` delimiter format was chosen because it is both human-readable and easily machine-parseable.

### 3.3 Earnings Dates: Complete Removal

#### Why Ignoring is Simpler Than Detecting

v2 designed a 3x ATR heuristic for earnings gap detection: if a day's price gap exceeds 3x the 14-day ATR, it was flagged as a likely earnings event, and signals were suppressed for 2 days before and 1 day after.

The updated requirements state: "Ignore earning dates entirely in the analysis." This removes:
- The gap detection heuristic (which had false positives for non-earnings gaps)
- Signal suppression logic (which added conditional complexity to every signal generator)
- Gap trade reporting (a separate reporting section)
- Any dependency on an earnings calendar

The analytical impact is minor for two reasons:
1. Most technical indicators already absorb earnings gaps into their calculations (RSI resets, MACD adjusts, Bollinger Bands widen). The indicators were not designed to avoid earnings -- they were designed to respond to all price movements.
2. The walk-forward framework's out-of-sample testing naturally evaluates whether a strategy survives earnings volatility. If a strategy is destroyed by earnings gaps, it will show up as poor OOS performance.

### 3.4 Python vs Go/Rust: Ecosystem is Decisive

#### The Scale Argument

The application processes ~2,500 rows of daily data for a single instrument. At this scale:
- Python with TA-Lib computes all indicators in < 50ms
- Go would be < 25ms
- Rust would be < 15ms

The difference (35ms) is imperceptible for a once-daily batch job. Performance becomes relevant only at 500+ instruments processed concurrently or real-time requirements -- neither applies.

Memory: Python ~50-80MB, Go ~10-15MB, Rust ~5-10MB. All trivial on modern hardware.

Startup: Python 200ms-2s (import overhead), Go/Rust 1-5ms. Irrelevant for batch processing.

#### The Library Gap

This is where the decision becomes clear. The Python ecosystem provides 60-70% of the application's functionality through existing, validated libraries:

- **TA-Lib:** 150+ indicators, validated against Bloomberg/Reuters for decades. Go has go-talib with ~40-50 indicators. Rust has ta-rs and yata with ~20-50 indicators each.
- **vectorbt:** Vectorized backtesting with built-in walk-forward analysis. No equivalent exists in Go or Rust.
- **quantstats:** Professional performance analytics (200+ metrics). No equivalent.
- **pandas:** Data manipulation and time series handling. Go's gonum/dataframe is primitive. Rust's polars is good but verbose.
- **yfinance:** One-line S&P 500 data download. Go/Rust would require manual HTTP and JSON parsing.

Building equivalent functionality from scratch in Go or Rust: estimated 6,000-15,000 lines vs 2,000-4,000 in Python. Development time: 8-16 weeks (Go) or 8-14 weeks (Rust) vs 2-4 weeks (Python).

#### The Correctness Risk

Financial calculations are subtly tricky. TA-Lib has been computing indicators for decades with professional validation. A custom Go/Rust implementation of MACD, RSI, or Bollinger Bands introduces correctness risk that is difficult to detect. Off-by-one errors in lookback periods, different handling of NaN values, or slightly different exponential smoothing formulas can produce materially different signals.

The risk is not that Go/Rust implementations would be wrong -- it is that verifying correctness across 25+ indicators requires substantial testing effort that is unnecessary when using a validated library.

#### Runtime Concerns Have Python-Native Solutions

The vendor question specifically asked about "minimizing runtime requirements." Python's runtime can be mitigated:

1. **Docker:** Package everything in a container. User needs only Docker, not Python.
2. **pandas-ta:** Pure Python alternative to TA-Lib, eliminating the C library dependency. 130+ indicators, imperceptible performance difference at 2,500 rows.
3. **conda-pack:** Create a relocatable environment with all dependencies. Distribute as tarball.
4. **PyInstaller/Nuitka:** Create standalone executable (100-300MB, platform-specific).

#### Confidence Level

**High.** The ecosystem advantage is not a marginal difference -- it is a 3-5x development time multiplier. The only scenario where Go/Rust would be preferred is if the application evolves to process hundreds of instruments in real-time, which is explicitly not in scope.

### 3.5 yfinance as Unified External Dependency

#### Consolidating Three Data Needs

The application needs external data for three purposes:
1. S&P 500 benchmark data (VQ2)
2. Stock split information for validation (VQ3)
3. Dividend information for flagging (VQ4)

Rather than introducing three separate data providers, yfinance serves all three:
- `yf.download("^GSPC", ...)` for S&P 500 data
- `yf.Ticker("NVDA").splits` for split history
- `yf.Ticker("NVDA").dividends` for dividend history

This keeps the external dependency surface minimal: one library, one upstream data source (Yahoo Finance).

#### The auto_adjust Trap

A critical finding: yfinance v0.2.51 (December 2024) changed `auto_adjust` to default `True`. This means `yf.download()` now returns dividend+split adjusted prices by default. The `Adj Close` column no longer exists in default output.

If the user's CSV is split-adjusted only (Nasdaq default), and yfinance ^GSPC data is dividend+split adjusted, the benchmark comparison has a systematic ~1.3% annual bias (the S&P 500 dividend yield). The solution is to always use `auto_adjust=False` when downloading benchmark data via yfinance.

This is the kind of subtle bug that could produce misleading results for months without being noticed. Documenting it prominently in the research is critical.

#### Graceful Degradation

If yfinance is unavailable (network issue, API change, Yahoo ToS enforcement):
1. S&P 500 benchmark: Skip with warning, report only buy-and-hold comparison
2. Split validation: Skip with warning, rely only on heuristic detection
3. Dividend flagging: Skip with warning, note that dividend impact is not assessed

The application's core functionality (indicator analysis, backtesting, signal generation) does not depend on yfinance. External data enhances the output but is not required.

### 3.6 Drawdown Hard Stop as Active SELL Signal

#### v2 vs v3 Interpretation

v2 designed a staged circuit breaker:
- Level 1 (15% DD): Reduce position size (not applicable with 100% in/out)
- Level 2 (20% DD): No new entries
- Level 3 (25% DD): Force close all positions

v3 replaces this with a simpler binary rule: "If the maximum acceptable drawdown is reached, sell the position." This is clearer and more actionable:

- **In backtesting:** Force exit when drawdown from equity peak reaches the configured maximum (default 30%). Record the forced exit in the trade log.
- **In current signal determination:** If the top strategy is currently "in position" and the drawdown from peak since entry exceeds the maximum, the signal is SELL today regardless of what the indicator says.

The staged circuit breaker was an over-engineering of a simple requirement. The user said "sell" -- not "reduce," not "stage," not "gradually exit."

### 3.7 Transaction Cost: 0% Default

#### Departing from v2's Token Cost

v2 introduced a 0.02% token transaction cost to prevent degenerate solutions (excessive trading for minimal edge). v3 defaults to 0% because the user explicitly stated costs are "not material."

The trade frequency scoring penalty (5% composite weight penalizing deviation from the 30-day target) already handles the degenerate-solution problem. A strategy that trades every day will score poorly on trade frequency alignment without needing a cost penalty.

However, transaction cost remains configurable. If the user later wants to model costs, the infrastructure is there.

### 3.8 Flexible CSV Parser

#### Data Source Undetermined

v2 designed specifically for Nasdaq CSV format (Date, Close/Last, Volume, Open, High, Low with dollar signs and MM/DD/YYYY dates). v3 must accommodate an undetermined data source.

The approach: auto-detect common CSV formats by looking at column names, date formats, and value formatting. Support a column mapping configuration for non-standard formats. This is a small increase in parser complexity but a significant increase in flexibility.

Known formats to auto-detect:
- **Nasdaq:** Date, Close/Last ($prefix), Volume, Open, High, Low -- descending dates, MM/DD/YYYY
- **Yahoo Finance:** Date, Open, High, Low, Close, Adj Close, Volume -- ascending dates, YYYY-MM-DD
- **Alpha Vantage:** timestamp, open, high, low, close, volume -- ascending, YYYY-MM-DD
- **Generic:** Any columns mappable via YAML config

---

## 4. What Changed from v2 and Why

### 4.1 Architectural Changes

| Change | v2 | v3 | Why |
|--------|----|----|-----|
| Execution model | Dual-mode, stateful | **Single-mode, stateless** | "Stateless, no persistence between runs" |
| Output | HTML (Plotly, Jinja2, mplfinance) | **stdout text** | "Use standard output for analysis" |
| Earnings handling | 3x ATR gap detection + suppression | **Removed** | "Ignore earning dates entirely" |
| Dependencies | 12 major | **8 major** | Removed Plotly, Jinja2, mplfinance, pyarrow |
| Drawdown behavior | Staged circuit breaker | **Binary hard stop** | "If max DD reached, sell the position" |
| Transaction cost | 0.02% token | **0% default** | "Not material" |
| CSV parser | Nasdaq-specific | **Flexible auto-detect** | "Data source has not been determined" |
| Config path | Generic | **`~/.config/technical-analysis/config.yaml`** | Explicit requirement |

### 4.2 Analytical Changes

No changes to the analytical core:
- Indicator tier rankings: unchanged
- 5 curated combinations: unchanged
- WFA parameters (504/126 rolling): unchanged
- Composite scoring weights: unchanged
- Rejection criteria: unchanged

This was a deliberate finding. The updated requirements affected infrastructure (how the application runs) but not analysis (what the application computes). The research from v2 on indicators, combinations, and backtesting remains fully valid.

### 4.3 New Additions (Not Present in v2)

| Addition | Source |
|----------|--------|
| Python vs Go/Rust analysis | VQ1 |
| S&P 500 data source recommendations | VQ2 |
| Stock split information sources | VQ3 |
| Stock dividend information sources | VQ4 |
| yfinance auto_adjust v0.2.51 behavior change | VQ2/VQ4 research |
| IEX Cloud shutdown (August 2024) | VQ2/VQ3/VQ4 research |
| Stateless signal determination algorithm | NQ1.1 resolution |
| stdout report structure | Output requirement change |
| Flexible CSV parser design | Data source requirement change |
| Split validation heuristic | VQ3 recommendation |
| Dividend impact flagging | VQ4 recommendation |

---

## 5. Vendor Question Methodology

### VQ1: Python vs Go/Rust

The analysis was structured across six comparison dimensions, weighted by relevance to this specific application:

1. **Library ecosystem** (decisive weight): Compared available libraries for technical analysis, backtesting, and analytics across all three languages.
2. **Performance at scale** (low weight): Benchmarked expected execution times at the application's actual scale (2,500 rows, single instrument).
3. **Runtime requirements** (medium weight): Compared deployment size, system dependencies, and distribution complexity.
4. **Development time** (high weight): Estimated lines of code and development weeks based on library coverage gaps.
5. **Correctness risk** (medium weight): Assessed the risk of introducing financial calculation bugs through custom implementations.
6. **Future evolution** (low weight): Considered a hybrid approach for potential future scaling requirements.

The conclusion (Python) was not close. The library ecosystem advantage is so large that it would take a very different application profile (real-time, multi-instrument, low-latency) to shift the recommendation.

### VQ2: S&P 500 Data Sources

Evaluated 5 options, ranked by simplicity and reliability:
1. User-provided CSV (preferred -- zero additional dependencies)
2. yfinance automatic download (recommended fallback)
3. FRED (close-only, insufficient for full analysis)
4. Alpha Vantage (viable but rate-limited)
5. Polygon.io (good quality but requires paid subscription for practical use)

Key finding: ^GSPC (the index itself) is preferred over SPY (the ETF) because SPY has a 0.0945% annual expense ratio that creates cumulative drag.

### VQ3: Stock Split Information

The primary recommendation is yfinance `ticker.splits` because it is already in the technology stack. A built-in validation heuristic was also designed to detect suspected unadjusted splits in data by looking for day-over-day close ratios near common split factors.

### VQ4: Stock Dividend Information

Research revealed the most nuanced findings:
- Dividend impact varies dramatically by yield (0% for NVDA to 12%+ for REITs)
- The yfinance auto_adjust change in v0.2.51 is a critical consistency concern
- The "double-counting trap" (adding dividends to dividend-adjusted prices) is a common implementation error
- Benchmark consistency requires matching the adjustment basis between asset and benchmark data

---

## 6. Confidence Levels

| Recommendation | Confidence | Notes |
|---------------|:-:|---|
| Python over Go/Rust | High | Ecosystem advantage is 3-5x; scale does not justify compiled language |
| yfinance as unified external data source | High | Already in stack; covers all three data needs; widely maintained |
| `auto_adjust=False` for benchmark consistency | High | Documented behavior change; mathematical bias is quantifiable (~1.3%/year) |
| IEX Cloud removal | High | Confirmed shutdown August 2024; no longer operational |
| Stateless architecture | High | User explicitly stated "stateless, no persistence" |
| stdout text output | High | User explicitly stated "standard output for analysis" |
| Earnings dates ignored | High | User explicitly stated "ignore earning dates entirely" |
| Drawdown hard stop (not staged) | High | User stated "if max DD reached, sell the position" |
| Flexible CSV parser | High | User stated "data source has not been determined" |
| Signal determination via replay | Medium-High | Logically sound and deterministic; edge cases exist (see Questions v3) |
| 0% default transaction cost | Medium-High | User stated "not material"; trade frequency scoring handles degenerate cases |
| ^GSPC over SPY for benchmark | Medium | Theoretically cleaner; practical difference is small (0.0945%/year) |
| Split validation heuristic | Medium | Useful but has false positive risk during market crashes |
| Dividend flagging approach | Medium | Accept-and-flag is pragmatic; full dividend adjustment would be more complete |
| User CSV as primary S&P 500 source | Medium | Simplest but adds user burden; yfinance fallback mitigates |

### Confidence Changes from v2

All v2 analytical recommendations (indicator tiers, combinations, WFA, scoring) remain at their v2 confidence levels. The new v3 recommendations are generally High confidence because they flow directly from explicit user requirements rather than inference.

---

## 7. Connections Between Research Streams

### Statelessness Connects to Everything

The stateless requirement is the most cross-cutting decision in v3. It affects:
- **Architecture:** No state directory, no cache, no re-optimization schedule
- **Signal determination:** Must derive current position from signal replay
- **Output:** No state-dependent "what changed since last run" reporting
- **Technology stack:** Remove pyarrow (caching)
- **Configuration:** No "daily mode" vs "full mode" toggle

### stdout Output Connects to Technology Stack

Removing HTML output eliminates 3 dependencies (Plotly, Jinja2, mplfinance) and removes the chart rendering pipeline. This also affects the reporting module design: instead of HTML template + chart components, it is a single text formatter.

### Vendor Questions Connect to Data Architecture

VQ2 (S&P 500 data) requires a benchmark loader module with graceful degradation. VQ3/VQ4 (splits/dividends) require a corporate actions module for validation and flagging. Both use yfinance, consolidating the external dependency.

### VQ1 (Python vs Go/Rust) Validates the Entire Technology Stack

If Go or Rust had been recommended, the entire technology stack from v2 would need replacement. Python's confirmation preserves all v2 library recommendations (TA-Lib, vectorbt, quantstats, pandas, exchange_calendars) without change.

---

## 8. Open Tensions and Trade-offs

### Tension 1: Stateless Re-optimization vs. Stability

Every run re-optimizes from scratch. The top strategy may change between runs as new data arrives. The user accepted this ("I understand that results may change"), but in practice, frequent strategy switches produce contradictory signals.

The mitigation is consensus reporting across top 5 strategies and the observation that WFA with 504-day IS windows produces relatively stable rankings -- a single new data point rarely changes the top strategy.

### Tension 2: Flexible CSV Parser vs. Complexity

Supporting multiple CSV formats adds parser complexity compared to v2's Nasdaq-only parser. Auto-detection heuristics may misidentify formats in edge cases.

The mitigation is supporting a YAML column mapping configuration that overrides auto-detection, giving the user explicit control when auto-detection fails.

### Tension 3: yfinance Dependency for External Data

yfinance depends on Yahoo Finance's unofficial API, which could change or become restricted. The application's core functionality does not depend on yfinance (analysis works on user CSV alone), but benchmark comparison and split/dividend validation would degrade.

The mitigation is graceful degradation: if yfinance fails, skip the dependent features with warnings rather than failing the entire analysis.

### Tension 4: 0% Transaction Cost vs. Degenerate Solutions

Removing the token cost (v2's 0.02%) could theoretically allow strategies with hundreds of trades and tiny per-trade gains to rank highly. The trade frequency scoring penalty should prevent this, but it is a softer constraint than a per-trade cost.

If this proves problematic during implementation, the configurable transaction cost can be set to a small positive value.

### Tension 5: Split Heuristic False Positives

The split validation heuristic (detecting day-over-day close ratios near common split factors) will produce false positives during genuine market crashes. A 50% single-day decline would trigger the 2:1 split detector. Cross-referencing with S&P 500 data (did the market also crash?) mitigates this, but requires the benchmark data to be available.

### Tension 6: Dividend-Adjusted vs. Split-Adjusted Data Inconsistency

If the user provides split-adjusted but not dividend-adjusted data (Nasdaq default), and the application uses yfinance with `auto_adjust=False` for the benchmark, both are on the same basis (split-adjusted only). But if the user provides fully adjusted data from a different source, the benchmark comparison may be inconsistent.

The application should detect whether an `Adj Close` column is present and adjust its benchmark data handling accordingly.

---

## 9. Relationship to Prior Thinking Documents

### What This Document Supersedes

This document supersedes `notes/Research_Thinking_v2.md` for all topics that changed:
- Execution model reasoning (stateless replaces dual-mode)
- Output reasoning (stdout replaces HTML)
- Earnings handling reasoning (ignore replaces detect)
- Drawdown behavior reasoning (hard stop replaces staged)
- Transaction cost reasoning (0% replaces 0.02%)
- Data format reasoning (flexible replaces Nasdaq-only)

### What v2 Thinking Document Still Covers

The following v2 reasoning remains the authoritative reference:
- Indicator re-ranking methodology and justifications (Section 3.1)
- WFA parameter derivation (504/126 IS/OOS, rolling windows) (Section 3.2)
- Drawdown constraint as hard filter vs soft penalty (Section 3.3)
- Trade frequency as soft constraint (Section 3.4)
- Combination strategy reasoning (5 curated, not algorithmic) (Section 3.5)
- Composite scoring weight derivation (Section 3.6)
- Williams %R long-only advantage (Section 5)
- Volume indicator upgrade reasoning (Section 3.1)
- 30% drawdown rejecting buy-and-hold NVDA finding (Section 5)

### What v1 Thinking Document Still Covers

The v1 thinking document remains the authoritative reference for:
- TA-Lib over pandas-ta selection
- vectorbt over Backtrader selection
- CSV as primary input rationale
- YAML for configuration rationale
- Pipeline architecture design principles
- Regime detection via ADX rationale
