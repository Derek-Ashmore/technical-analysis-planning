# Research Questions v3: Resolution Status and New Questions

Updated question tracking following the v3 research revision. This document maps the resolution of all v2 open questions based on the updated `requirements/Main.md`, identifies new questions arising from v3 research, and prioritizes remaining open items for implementation.

---

## 1. Priority Summary Table

### Questions by Status

| Status | Count | Description |
|--------|:-----:|-------------|
| Fully answered | 28 | Requirements are clear; no further input needed |
| Partially answered | 2 | Direction is known but specifics need confirmation |
| Not answered (deferrable) | 5 | Can proceed with recommended defaults |
| Not applicable | 6 | Eliminated by scope decisions |
| New v3 questions | 10 | Arising from v3 research and vendor questions |

### Questions by Priority

| Priority | Count | Questions |
|----------|:-----:|-----------|
| Should answer before implementation | 3 | NQ3.1 (v3), NQ3.3 (v3), NQ3.5 (v3) |
| Should answer during design | 4 | NQ3.2 (v3), NQ3.4 (v3), NQ3.6 (v3), NQ3.7 (v3) |
| Can defer to development | 8 | NQ3.8-NQ3.10 (v3), Q6.1, Q6.3, Q7.3, NQ6.5, NQ9.1 |

### v2 "Must Answer Before Implementation" Resolution

All 5 v2 critical questions are now resolved:

| v2 ID | Question | v3 Resolution |
|-------|----------|--------------|
| NQ1.1 | Re-analyze daily or lock strategy? | **RESOLVED:** Stateless. Every run re-analyzes everything. "The application is stateless and will not require persistence between runs." |
| NQ1.2 | Include actionable daily signal? | **RESOLVED:** Yes. "The application will provide output that includes an actionable signal ('BUY today' / 'HOLD' / 'SELL today')." |
| NQ3.1 | Definition of "most profitable"? | **RESOLVED:** "Best fit means indicates buys and sells most profitably over the entire analysis." Combined with composite scoring for risk adjustment. |
| NQ3.2 | 30% drawdown: hard filter or soft? | **RESOLVED:** Hard stop. "If the maximum acceptable drawdown is reached, sell the position." Active SELL trigger, not just backtest filter. |
| NQ7.2 | Position sizing 100% in/out? | **RESOLVED:** "All long positions will use the entire balance of the trading account for buys and sells. Long positions are either completely in the market or out of it." |

---

## 2. v2 Question Resolution Map (Updated for v3)

### v2 Questions Now Fully Resolved

**NQ1.1: Does each daily run re-analyze everything or apply a locked-in strategy?**
- v2 Status: Must answer before implementation
- v3 Status: **RESOLVED**
- Resolution: "The application is stateless and will not require persistence between runs. I understand that results may change when new data is analyzed."
- Impact: Single execution mode. No state directory. No strategy locking. Every run is complete re-analysis.

**NQ1.2: Does the daily output include an actionable signal?**
- v2 Status: Must answer before implementation
- v3 Status: **RESOLVED**
- Resolution: "The application will provide output that includes an actionable signal ('BUY today' / 'HOLD' / 'SELL today')."
- Impact: Signal determination module required. Must derive current position state from signal replay within a single stateless execution.

**NQ1.3: How is indicator instability handled when the top recommendation changes?**
- v2 Status: Should answer during design
- v3 Status: **RESOLVED** (by stateless acceptance)
- Resolution: "I understand that results may change when new data is analyzed." User explicitly accepts recommendation instability. Consensus reporting across top strategies recommended but not required.

**NQ2.1: Where does S&P 500 benchmark data come from?**
- v2 Status: Should answer during design
- v3 Status: **RESOLVED**
- Resolution: "The application should benchmark indicator performance against buy-and-hold and the S&P 500 index." Source not specified, but v3 research recommends user CSV primary with yfinance fallback. Graceful degradation if unavailable.

**NQ3.1: How is "most profitable" defined?**
- v2 Status: Must answer before implementation
- v3 Status: **RESOLVED**
- Resolution: "Best fit means indicates buys and sells most profitably over the entire analysis." Interpreted as composite score (risk-adjusted total return with drawdown constraint) per v2 recommendation. Requirements also specify trade-by-trade P&L reporting and summary statistics.

**NQ3.2: Is the 30% maximum drawdown a hard filter or a soft penalty?**
- v2 Status: Must answer before implementation
- v3 Status: **RESOLVED**
- Resolution: "If the maximum acceptable drawdown is reached, sell the position." This goes beyond a hard filter -- it is an active SELL trigger during both backtesting and current signal determination.

**NQ4.1: Is 30 days the average holding period or the average time between signals?**
- v2 Status: Should answer during design
- v3 Status: **RESOLVED**
- Resolution: "The target trading frequency should be configurable with 30 days as the default." The term "trading frequency" combined with "target" confirms this is a soft preference for average time between trades, matching v2's recommended interpretation.

**NQ4.2: How strictly should the 30-day target be enforced?**
- v2 Status: Should answer during design
- v3 Status: **RESOLVED**
- Resolution: The word "target" (not "limit" or "maximum") confirms v2's soft constraint recommendation. Scoring penalty, not hard filter.

**NQ5.1: Does "maximum 5 combinations" mean 5 tested or 5 reported?**
- v2 Status: Should answer during design
- v3 Status: **RESOLVED**
- Resolution: "The maximum number of indicator combinations considered should be configurable with 5 as a default." The word "considered" could mean tested or reported. v3 interpretation: this is the number reported/considered in the final output. Internal testing covers more combinations from the curated set.

**NQ6.4: How should stock split and dividend adjustments be handled?**
- v2 Status: Should answer during design
- v3 Status: **RESOLVED**
- Resolution: "Historical data will be stock-split adjusted and include volume data." Data is pre-adjusted. VQ3/VQ4 research provides validation and flagging recommendations.

**NQ7.1: What return is assumed on cash?**
- v2 Status: Can defer
- v3 Status: **RESOLVED**
- Resolution: "Assume zero return on cash."

**NQ7.2: Is position sizing 100% in/out?**
- v2 Status: Must answer before implementation
- v3 Status: **RESOLVED**
- Resolution: "All long positions will use the entire balance of the trading account for buys and sells."

### v2 Questions Still Open (Deferrable)

**Q1.3: Expected time span of historical data?**
- Status: **PARTIALLY ANSWERED** (deferrable)
- Known: Sample is ~10 years. Starting default is January 1, 2000 (up to ~26 years).
- Recommended: Accept any length >= 1 year for basic analysis; require >= 3 years for WFA; recommend 5+ years.

**Q6.1: Is Docker an acceptable deployment mechanism?**
- Status: **NOT ANSWERED** (deferrable)
- Recommended: Optional. Not required for v1.

**Q6.3: What Python version is the minimum target?**
- Status: **NOT ANSWERED** (deferrable)
- Recommended: Python 3.11+.

**Q7.3: Will the application support custom indicator definitions?**
- Status: **ANSWERED**
- Resolution: "The application will not support user-supplied custom indicators." This is now explicitly out of scope.

**NQ9.1: Does the future options extension affect architecture?**
- Status: **PARTIALLY ANSWERED** (deferrable)
- Known: "In future, I may extend analysis to options." No immediate architectural impact. Abstract Instrument model recommended for future readiness.

### v1 Questions -- Complete Resolution Summary

All 33 v1 questions are now fully resolved or not applicable:

| v1 ID | Status | Resolution |
|-------|--------|-----------|
| Q1.1 | ANSWERED | Equities only |
| Q1.2 | NOT APPLICABLE | No commodities |
| Q1.3 | PARTIALLY ANSWERED | ~10 years typical; validate at runtime |
| Q1.4 | ANSWERED | CSV now, API future |
| Q2.1 | ANSWERED | Long-only |
| Q2.2 | ANSWERED | $10,000 starting capital, 100% in/out |
| Q2.3 | ANSWERED | 30-day target, configurable |
| Q3.1 | ANSWERED | Yes, max 5 combinations reported |
| Q3.2 | ANSWERED | Tier 1+2 default; all in verbose |
| Q4.1 | ANSWERED | Entry/exit dates + P&L per trade |
| Q4.2 | ANSWERED | Min 20/window, 30+ total; show in verbose |
| Q4.3 | ANSWERED | Best strategy + top 5 + full in verbose |
| Q4.4 | ANSWERED | stdout text (replaces HTML/PDF) |
| Q5.1 | ANSWERED | 30-day = trade frequency target |
| Q5.2 | ANSWERED | Independent of indicator params |
| Q5.3 | ANSWERED | ADX as combination filter |
| Q6.1 | NOT ANSWERED (defer) | Docker optional |
| Q6.2 | ANSWERED | CLI/batch, stdout output |
| Q6.3 | NOT ANSWERED (defer) | Recommend 3.11+ |
| Q7.1 | ANSWERED | Single instrument |
| Q7.2 | ANSWERED | Stateless = implicit forward testing |
| Q7.3 | ANSWERED | Out of scope (explicit) |
| Q7.4 | ANSWERED | No portfolio analysis |

---

## 3. New Questions from v3

These questions arise from the v3 requirements update and vendor question research. They are numbered NQ3.x (v3 new questions).

### NQ3.1: Signal Semantics Without Position Knowledge

- Priority: **SHOULD ANSWER BEFORE IMPLEMENTATION**
- Context: In stateless mode, the application derives "current position" by replaying strategy signals. But the user may have actually exited a position manually (e.g., panic sell during a drawdown) that the replay considers "in position." The application has no way to know the user's actual position.
- Question: Should the signal output explicitly state that it assumes the user followed all prior signals from the strategy? Or should it report the raw indicator signal independent of assumed position state?
- Recommended: Report both the raw indicator signal ("RSI(14) is at 28, below buy threshold 30") AND the derived position state ("Strategy assumes: in position since 2025-11-03"). This lets the user reconcile their actual position with the strategy's assumed state.
- Impact: Affects the "Current Signal" section of stdout report format.

### NQ3.2: Output Column Width and Terminal Compatibility

- Priority: **SHOULD ANSWER DURING DESIGN**
- Context: The trade log table and benchmark comparison table require formatted columns. Terminal width varies (80-column minimum? 120? unlimited?).
- Question: What minimum terminal width should the output assume?
- Recommended: Design for 80-column width as the default. Use `--wide` or a config option for wider terminals. Truncate long stock names or values if needed to fit 80 columns.
- Impact: Affects text formatter module design.

### NQ3.3: How to Handle Insufficient Data for WFA

- Priority: **SHOULD ANSWER BEFORE IMPLEMENTATION**
- Context: WFA requires minimum 830 trading days (~3.3 years) of data. If the user provides less data, the application cannot perform walk-forward analysis.
- Question: Should the application (a) refuse to run, (b) run without WFA and warn, or (c) reduce WFA window sizes to fit the available data?
- Recommended: Option (b) -- run a simplified analysis without WFA and issue a prominent warning to stderr. Use in-sample-only backtesting as a fallback, clearly labeled as "not walk-forward validated."
- Impact: Affects pipeline flow control and the minimum data validation logic.

### NQ3.4: Verbose Output Scope and Level

- Priority: **SHOULD ANSWER DURING DESIGN**
- Context: Requirements say "verbose mode that will document what technical analysis indicators it examined and rejected. Include a summary of what was found." Also: "Make indicators that produce no trades available in verbose output."
- Question: How detailed should verbose output be? Options:
  - (a) List of all indicators tested with pass/fail status and key metrics
  - (b) Full parameter optimization history per indicator
  - (c) Walk-forward window-by-window breakdown per indicator
- Recommended: Option (a) for the default verbose mode. Options (b) and (c) are too detailed for stdout and would overwhelm the output. A future `--debug` flag could enable deeper output.
- Impact: Affects verbose output formatter and the amount of data passed from the backtest engine to the reporting module.

### NQ3.5: Multiple Strategies Producing Conflicting Signals

- Priority: **SHOULD ANSWER BEFORE IMPLEMENTATION**
- Context: The top strategy signals "BUY today" but the #2 and #3 strategies signal "SELL today." Should the application report only the top strategy's signal, or highlight the disagreement?
- Question: Should the current signal section show signals from all top N strategies, or only the #1 strategy?
- Recommended: Show the #1 strategy's signal prominently. Below it, show a consensus summary: "3 of 5 strategies agree: BUY" or "Strategies are split: 2 BUY, 2 HOLD, 1 SELL." This gives the user confidence context without overriding the top strategy's signal.
- Impact: Affects the current signal determination module and report structure.

### NQ3.6: S&P 500 Data Unavailable -- Reporting Impact

- Priority: **SHOULD ANSWER DURING DESIGN**
- Context: If no S&P 500 benchmark data is available (no user CSV, yfinance fails), the benchmark comparison table has empty columns.
- Question: Should the benchmark comparison section be omitted entirely, or show "N/A" for S&P columns?
- Recommended: Show the table with "N/A" for S&P columns and a note: "S&P 500 data unavailable; provide via config or ensure internet connectivity for yfinance download." This is more informative than omitting the section.
- Impact: Affects text formatter and benchmark comparison module.

### NQ3.7: Starting Date vs. Data Availability

- Priority: **SHOULD ANSWER DURING DESIGN**
- Context: Default start date is January 1, 2000. If the stock was not publicly traded until later (e.g., IPO in 2004), or if the CSV only contains data from 2015, the start date is before available data.
- Question: Should the application (a) error out, (b) silently start from the first available date, or (c) warn and start from the first available date?
- Recommended: Option (c). Warn to stderr: "Configured start date 2000-01-01 precedes first data point 2015-03-20. Analysis begins at 2015-03-20."
- Impact: Affects data loader and date range validation.

### NQ3.8: pandas-ta as TA-Lib Alternative

- Priority: **CAN DEFER TO DEVELOPMENT**
- Context: VQ1 research identified pandas-ta as a pure Python alternative to TA-Lib, eliminating the C library dependency. At 2,500 rows, performance difference is imperceptible.
- Question: Should the application support pandas-ta as a fallback when TA-Lib is not installed?
- Recommended: Design the indicator module with an abstraction layer that could use either backend. Implement TA-Lib first (more indicators, better validated). If TA-Lib installation proves problematic for users, add pandas-ta support as a runtime fallback.
- Impact: Affects indicator module architecture (abstraction layer vs. direct TA-Lib calls).

### NQ3.9: Report Header Metadata

- Priority: **CAN DEFER TO DEVELOPMENT**
- Context: The stdout report should include metadata: application version, analysis timestamp, data file path, configuration used. This aids in reproducing results.
- Question: How much metadata should be included in the report header?
- Recommended: Include: application version, analysis timestamp (UTC), data file path, date range analyzed, number of trading days, starting capital, configuration file path, and key config values (max drawdown, trade frequency target, max combinations). Keep it concise (5-8 lines).
- Impact: Affects report header format.

### NQ3.10: Exit Code Semantics

- Priority: **CAN DEFER TO DEVELOPMENT**
- Context: As a CLI tool, the exit code communicates success/failure to calling scripts.
- Question: What exit codes should be used?
- Recommended:
  - 0: Success (analysis complete, signal generated)
  - 1: Error (data loading failed, invalid configuration, etc.)
  - 2: Warning-level completion (analysis ran but with significant warnings, e.g., no strategies passed the drawdown filter)
- Impact: Affects CLI entry point and error handling design.

---

## 4. Prioritized Question List

### Block 1: Should Answer Before Implementation

These 3 questions affect core design decisions. Proceeding with defaults is possible but carries some rework risk.

| ID | Question | Recommended Default | Why It Matters |
|----|----------|-------------------|----------------|
| NQ3.1 | Signal semantics without position knowledge? | Report both raw signal and derived state | Determines signal output format |
| NQ3.3 | Insufficient data for WFA? | Run without WFA, warn prominently | Determines pipeline flow control |
| NQ3.5 | Conflicting signals across top strategies? | Show #1 signal + consensus summary | Determines signal section format |

**If no answer is received:** Proceed with all recommended defaults. They are pragmatic and can be adjusted without major rework.

### Block 2: Should Answer During Design

These 4 questions affect specific module designs but have reasonable defaults.

| ID | Question | Recommended Default | Design Impact |
|----|----------|-------------------|---------------|
| NQ3.2 | Terminal width assumption? | 80 columns default | Text formatter layout |
| NQ3.4 | Verbose output detail level? | Indicator pass/fail list with key metrics | Verbose output module scope |
| NQ3.6 | S&P 500 unavailable display? | Show table with N/A columns | Benchmark comparison formatter |
| NQ3.7 | Start date before data availability? | Warn and use first available date | Data loader validation |

### Block 3: Can Defer to Development

These 5 questions can be resolved during implementation with no architectural risk.

| ID | Question | Recommended Default |
|----|----------|-------------------|
| NQ3.8 | pandas-ta as TA-Lib fallback? | TA-Lib only; abstraction layer for future |
| NQ3.9 | Report header metadata? | Version, timestamp, file, date range, key config |
| NQ3.10 | Exit code semantics? | 0=success, 1=error, 2=warning |
| Q6.1 | Docker deployment? | Optional |
| Q6.3 | Python version minimum? | 3.11+ |

---

## 5. Question Count Trajectory

| Version | Total Questions | Fully Resolved | Open | Must-Answer |
|---------|:-:|:-:|:-:|:-:|
| v1 | 33 | 0 | 33 | 5 (inferred) |
| v2 | 49 (33 v1 + 16 new) | 12 | 21 (+6 N/A) | 5 |
| v3 | 59 (49 v2 + 10 new) | 42 | 8 (+6 N/A) | 0 (3 "should answer") |

The question count trajectory shows convergence. v3 has zero "must answer" blockers -- all remaining questions have reasonable defaults that can proceed without user input. The 3 "should answer" questions affect output formatting, not core architecture.

---

## 6. Cross-Reference: All Questions by Status

### Fully Resolved (42)

| ID | Short Description | Resolution Source |
|----|------------------|------------------|
| Q1.1 | Target asset class | v2: Equities only |
| Q2.1 | Long-only or long-short | v2: Long-only |
| Q2.2 | Starting capital | v3: $10,000 default, 100% in/out |
| Q2.3 | Trade frequency | v2: 30-day target, configurable |
| Q3.1 | Test combinations | v2: Yes, max 5 reported |
| Q3.2 | Indicator set | v3: Tier 1+2 default; verbose shows all |
| Q4.1 | Trade detail level | v2: Entry/exit dates + P&L |
| Q4.2 | Few-trade indicators | v3: Show in verbose (explicit requirement) |
| Q4.3 | Recommendation format | v3: Best + top N + verbose |
| Q4.4 | Output format | v3: stdout text |
| Q5.1 | Pattern definition | v2: 30-day = trade frequency |
| Q5.2 | Lookback window | v2: Independent of indicator params |
| Q5.3 | Regime feedback | v2: ADX as combination filter |
| Q6.2 | CLI or web | v3: CLI, stdout/stderr |
| Q7.1 | Single/multi instrument | v2: Single |
| Q7.2 | Forward testing | v3: Stateless = daily re-analysis |
| Q7.3 | Custom indicators | v3: Explicitly out of scope |
| Q7.4 | Portfolio analysis | v2: No |
| Q1.4 | Multiple data providers | v2: CSV now, API future |
| NQ1.1 | Daily re-analysis or lock | v3: Stateless, re-analyze every run |
| NQ1.2 | Actionable signal | v3: Yes, BUY/HOLD/SELL |
| NQ1.3 | Indicator instability | v3: User accepts changes |
| NQ2.1 | S&P 500 data source | v3: User CSV or yfinance fallback |
| NQ2.2 | Total vs price return | v3: Match data adjustment basis |
| NQ3.1 (v2) | Most profitable definition | v3: Composite scoring with drawdown filter |
| NQ3.2 (v2) | Drawdown hard/soft | v3: Hard stop, active SELL |
| NQ4.1 | 30 days = holding period | v3: Trading frequency target |
| NQ4.2 | Frequency strictness | v3: Soft (target, not limit) |
| NQ5.1 | 5 combinations meaning | v3: Configurable max reported |
| NQ6.1 | Dollar signs | v2: Auto-strip |
| NQ6.2 | Variant column names | v3: Auto-detect, configurable mapping |
| NQ6.3 | Data sort order | v2: Auto-detect, normalize ascending |
| NQ6.4 | Split/dividend handling | v3: Split-adjusted accepted; VQ3/VQ4 for validation |
| NQ6.5 | Multiple CSV formats | v3: Flexible auto-detect (data source TBD) |
| NQ7.1 | Cash return | v3: Zero (explicit) |
| NQ7.2 (v2) | Position sizing | v3: 100% in/out (explicit) |
| NQ8.1 | Historical vs signal tool | v3: Both (explicit) |
| VQ1 | Python vs Go/Rust | v3: Python (ecosystem decisive) |
| VQ2 | S&P 500 data sources | v3: User CSV + yfinance fallback |
| VQ3 | Stock split info | v3: yfinance + heuristic validation |
| VQ4 | Stock dividend info | v3: yfinance + flagging |

### Not Applicable (6)

| ID | Reason |
|----|--------|
| Q1.2 | No commodities |
| Q1.2 related forex | No forex |
| Q1.2 related crypto | No crypto |
| v2 Daily mode questions | Stateless (no daily mode) |
| v2 State persistence questions | Stateless (no state) |
| v2 HTML report questions | stdout output (no HTML) |

### Still Open -- Deferrable (8)

| ID | Question | Default |
|----|----------|---------|
| Q1.3 | Data length expectations | Validate >= 1 year, warn < 3 years |
| Q6.1 | Docker deployment | Optional |
| Q6.3 | Python version | 3.11+ |
| NQ3.8 | pandas-ta fallback | TA-Lib primary only |
| NQ3.9 | Report header metadata | Standard set |
| NQ3.10 | Exit code semantics | 0/1/2 |
| NQ9.1 | Options roadmap impact | Minimal; abstract models |

### Open -- Should Answer (7)

| ID | Priority | Question |
|----|----------|----------|
| NQ3.1 | Before impl | Signal semantics without position knowledge |
| NQ3.3 | Before impl | Insufficient data for WFA |
| NQ3.5 | Before impl | Conflicting signals across strategies |
| NQ3.2 | During design | Terminal width assumption |
| NQ3.4 | During design | Verbose output detail level |
| NQ3.6 | During design | S&P 500 unavailable display |
| NQ3.7 | During design | Start date before data availability |

---

## Sources

- [requirements/Main.md](../requirements/Main.md) -- Updated requirements driving v3
- [instructions/VendorQuestions.md](../instructions/VendorQuestions.md) -- Vendor questions
- [research/Research_v3.md](../research/Research_v3.md) -- v3 research document
- [notes/Research_Questions_v2.md](Research_Questions_v2.md) -- v2 questions (superseded by this document)
- [notes/Research_Questions_v1.md](Research_Questions_v1.md) -- v1 original 33 questions
