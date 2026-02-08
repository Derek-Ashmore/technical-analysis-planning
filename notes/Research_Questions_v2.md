# Research Questions v2: Resolution Status and New Questions

Updated question tracking following the v2 research revision. This document maps the resolution of all 33 v1 questions based on clarified requirements, identifies new questions arising from those clarifications, and prioritizes the remaining open items for the implementation phase.

---

## 1. Priority Summary Table

### Questions by Status

| Status | Count | Description |
|--------|:-----:|-------------|
| Fully answered | 12 | Requirements are clear; no further input needed |
| Partially answered | 5 | Direction is known but specifics need confirmation |
| Not answered (deferrable) | 10 | Can proceed with recommended defaults |
| Not applicable | 6 | Eliminated by scope decisions (no commodities, no forex, etc.) |

### Questions by Priority

| Priority | Count | Questions |
|----------|:-----:|-----------|
| Must answer before implementation | 5 | NQ1.1, NQ1.2, NQ3.1, NQ3.2, NQ7.2 |
| Should answer during design | 6 | NQ1.3, NQ2.1, NQ4.1, NQ4.2, NQ5.1, NQ6.4 |
| Can defer to development | 12 | NQ2.2, NQ6.1-6.3, NQ6.5, NQ7.1, NQ9.1, Q4.4, Q6.1, Q6.3, Q7.3, Q1.3, Q5.3 |

### Top 5 Most Critical Questions

1. **NQ1.1 + NQ1.2: Daily run mode semantics** -- Determines whether the application is a historical report generator or an operational signal tool. Both capabilities appear required.
2. **NQ3.1: Definition of "most profitable"** -- This is the core objective function for indicator ranking. Total return with a drawdown constraint is the recommended approach.
3. **NQ3.2: Drawdown as hard filter vs soft penalty** -- Fundamentally changes the ranking algorithm. Hard filter (reject strategies exceeding 30%) is recommended.
4. **NQ7.2: Position sizing model** -- 100% in/out simplifies everything for single-instrument analysis and is the recommended default.
5. **NQ2.1: S&P 500 benchmark data source** -- Needed for the required dual benchmark comparison (buy-and-hold vs S&P 500).

---

## 2. v1 Question Resolution Map

### Category 1: Asset Class and Data Context

**Q1.1: What specific type of spot price data will the application analyze?**
- Status: **ANSWERED**
- Answer: Equities only. No forex, commodities, or crypto.
- Impact: Eliminates multi-asset mode logic, forex volume workarounds, crypto threshold adjustments, and commodity contract roll handling. Volume indicators are fully usable. All indicator defaults apply without asset-class branching.
- Architectural change: Remove asset_class configuration field. Remove per-asset-class parameter adjustment logic. Simplify data validation to equity OHLCV only.

**Q1.2: For commodity data, does "spot price" refer to physical spot or front-month futures proxy?**
- Status: **NOT APPLICABLE**
- Reason: No commodity data. Equities only.

**Q1.3: What is the expected time span of the historical data?**
- Status: **PARTIALLY ANSWERED**
- Known: The sample dataset is approximately 10 years (NVDA, 2,515 rows). This is sufficient for walk-forward analysis with multiple in-sample/out-of-sample cycles.
- Remaining question: Is 10 years the expected norm for all stocks the user will analyze? The application should validate data length at load time and warn if fewer than 3 years are provided (minimum for meaningful WFA).
- Recommended default: Accept any length >= 1 year for basic analysis; require >= 3 years for WFA; recommend 5+ years for full regime analysis.

**Q1.4: Will the application need to support multiple data providers simultaneously?**
- Status: **ANSWERED**
- Answer: CSV input now, API in a future version. Single data source per run.
- Impact: Build a clean DataLoader interface with a CSV implementation. Design for a second implementation (API) later, but do not build it in v1.

### Category 2: Trading Assumptions

**Q2.1: Should the application assume long-only trading, or long and short?**
- Status: **ANSWERED**
- Answer: Long-only. A sell signal means "exit long position to cash." Never short.
- Impact: Simplifies backtesting significantly. No borrowing cost modeling. No short-specific risk metrics. Signal semantics are binary: in (long) or out (cash).

**Q2.2: What is the assumed starting capital and position sizing model?**
- Status: **NOT ANSWERED** (deferrable with strong default)
- Recommended default: 100% capital in/out for single-instrument analysis. This is the simplest model and avoids the complexity of fractional sizing, risk-based sizing, and portfolio allocation -- none of which are relevant for single-instrument, long-only analysis.
- Note: Configurable if needed later, but 100% in/out should be the v1 implementation.

**Q2.3: Are there constraints on trade frequency?**
- Status: **ANSWERED**
- Answer: 30-day target holding period, configurable. This is the target average trade duration, not a hard constraint on when trades can occur.
- Impact: Trade frequency becomes a scoring dimension. Strategies whose average holding period deviates significantly from the target receive a penalty in the composite ranking.

### Category 3: Indicator Selection and Scope

**Q3.1: Should the application test indicator combinations in v1?**
- Status: **ANSWERED**
- Answer: Yes. Maximum 5 combinations reported, configurable. The application should test more combinations internally but surface only the top performers.
- Impact: Combination testing is in scope. Limit the combinatorial explosion by testing only proven pairings (MACD+RSI+ADX, RSI+Bollinger, etc.) rather than exhaustive search.

**Q3.2: Is there a preferred initial set of indicators, or should all 25 be included?**
- Status: **NOT EXPLICITLY ANSWERED** (deferrable with strong default)
- Recommended: Start with Tier 1 + Tier 2 (12 indicators). Report all indicator results in verbose mode. Highlight the best performers in the summary.
- Note: Since the application is equities-only, all indicators including volume-based ones are valid.

### Category 4: Output and Reporting

**Q4.1: What level of detail is expected in trade-by-trade reporting?**
- Status: **ANSWERED**
- Answer: Entry and exit dates with profit/loss per trade. Both individual trade detail and summary statistics.
- Impact: The reporting module needs a trade log table (entry date, exit date, entry price, exit price, P&L absolute, P&L percent, holding period) plus summary aggregates.

**Q4.2: How should indicators that produce no trades (or fewer than 30) be handled?**
- Status: **NOT ANSWERED** (deferrable with strong default)
- Recommended: Require a minimum of 20 trades per WFA window and 30+ trades total for a strategy to be ranked. Strategies with fewer trades should be reported with an "insufficient data" flag in verbose mode but excluded from the primary ranking.

**Q4.3: Should the application produce a definitive recommendation or just ranked data?**
- Status: **PARTIALLY ANSWERED**
- Known: "Tell me what indicators best fit the data" implies a definitive answer, not just raw data.
- Recommended: Highlight the single best strategy prominently, present the top 5 as supporting context, and provide the full ranking in verbose mode. Include confidence indicators (agreement among top strategies, OOS consistency, parameter stability).

**Q4.4: Is PDF output required, or is HTML sufficient?**
- Status: **NOT ANSWERED** (deferrable)
- Recommended: HTML as default (supports Plotly interactive charts, easier to generate, richer output). PDF as an optional export (adds WeasyPrint dependency). Implement HTML first; add PDF if explicitly requested.

### Category 5: Pattern Recognition

**Q5.1: What constitutes a "pattern" in the context of the configurable N-day window?**
- Status: **SUBSTANTIALLY CLARIFIED**
- Answer: The 30-day window is the TARGET TRADE FREQUENCY (average holding period), not a pattern detection window. This is a much simpler concept than v1 assumed. There is no separate "pattern recognition" pipeline stage.
- Impact: Eliminates the multi-scale pattern detection feature (15/30/60 day windows) from v1 scope. The 30-day value feeds into the trade frequency scoring dimension instead.

**Q5.2: Should the configurable lookback window (default 30 days) apply to pattern detection, analysis period, or indicator parameters?**
- Status: **CLARIFIED**
- Answer: The 30-day value is the trade frequency target. It is independent of indicator parameters (RSI=14, MACD=12/26/9, etc.) and independent of the analysis window (full historical data).
- Impact: This is a scoring parameter, not a calculation parameter.

**Q5.3: Should pattern recognition feed back into indicator evaluation?**
- Status: **NOT DIRECTLY ANSWERED** (deferrable)
- Recommended: Implement ADX as a filter/combination option within the standard indicator framework, not as a separate pipeline stage. Regime-dependent evaluation is valuable but can be implemented as "ADX + X" combination strategies rather than a distinct architectural concern.

### Category 6: Technical Implementation

**Q6.1: Is Docker an acceptable deployment mechanism?**
- Status: **NOT ANSWERED** (deferrable)
- Note: Docker would simplify TA-Lib installation. Can be added at any time without architectural impact.

**Q6.2: Should the application be CLI-only, or include a web interface?**
- Status: **EFFECTIVELY ANSWERED**
- Answer: CLI/batch processing. "Run once a day" describes a batch job, not an interactive session.
- Impact: No web framework needed. CLI entry point with YAML configuration. Output is a report file (HTML), not a live dashboard.

**Q6.3: What Python version is the minimum target?**
- Status: **NOT ANSWERED** (deferrable)
- Recommended: Python 3.11+ (for improved error messages, tomllib, performance improvements). If the user's environment requires an older version, 3.10+ is the minimum for match statements and modern type syntax.

### Category 7: Scope and Future Considerations

**Q7.1: Will the application analyze a single instrument at a time, or compare across multiple instruments?**
- Status: **ANSWERED**
- Answer: Single instrument at a time. No cross-instrument comparison, no correlation analysis, no portfolio-level metrics.
- Impact: Simplifies data loading, backtesting, and reporting. One CSV in, one report out.

**Q7.2: Is real-time or forward-testing capability planned for a future version?**
- Status: **PARTIALLY ANSWERED**
- Known: "Run once a day" is essentially forward testing with a daily batch cadence. The application produces a daily report, not a live stream.
- Remaining question: Whether state persists between daily runs (see NQ1 below).

**Q7.3: Will the application support custom indicator definitions?**
- Status: **NOT ANSWERED** (deferrable)
- Recommended: Not in v1. Built-in indicators only. The Indicator base class + registry pattern supports future custom indicators without architectural changes.

**Q7.4: Should the application eventually support portfolio-level analysis?**
- Status: **ANSWERED**
- Answer: No. Single instrument analysis only.
- Impact: No portfolio allocation, no correlation matrices, no cross-instrument risk metrics.

---

## 3. New Questions from v2

These questions arise from the clarified requirements and represent gaps that were not visible during v1 research.

### NQ1: Daily Run Semantics

The requirement to "run once a day" introduces questions about application state and output purpose that were not addressed in v1.

**NQ1.1: Does each daily run re-analyze everything or apply a locked-in strategy?**
- Priority: **MUST ANSWER BEFORE IMPLEMENTATION**
- Context: Two fundamentally different modes are possible. Mode A: full re-analysis every day (recalculate all indicators, re-rank, potentially change the recommended strategy). Mode B: lock in the best strategy from an initial analysis, then each daily run applies that locked strategy to produce today's signal.
- Impact: Mode A is stateless (no persistence between runs). Mode B requires state management (saved strategy selection, parameter lock, last signal tracking).
- Recommended default: Mode A (full re-analysis daily) with a stability report showing whether the top recommendation has changed since the previous run. This avoids state management complexity while still providing the operational continuity of Mode B.

**NQ1.2: Does the daily output include an actionable signal ("BUY today" / "HOLD" / "SELL today")?**
- Priority: **MUST ANSWER BEFORE IMPLEMENTATION**
- Context: There is a fundamental difference between a historical analysis tool ("here is what worked best over the last 10 years") and an operational signal tool ("based on the best strategy, here is what you should do today").
- Impact: A historical report requires only backtesting. An operational signal requires applying the winning strategy to today's data and producing a current-state signal (in position / out of position / signal to enter / signal to exit).
- Recommended default: Both. The report includes historical analysis AND a current signal section showing what the top strategy (and top 5) are signaling as of the most recent bar.

**NQ1.3: How is indicator instability handled when the top recommendation changes day-to-day?**
- Priority: **SHOULD ANSWER DURING DESIGN**
- Context: If the application re-analyzes daily (NQ1.1 Mode A), the "best" strategy might flip between runs as new data arrives. This could produce contradictory signals.
- Impact: Determines whether the application needs a stability metric or hysteresis mechanism.
- Recommended default: Show signals from the top 5 strategies. If they agree (consensus), confidence is high. If they disagree, report the split and flag low confidence. No forced stability -- let the data speak.

### NQ2: S&P 500 Benchmark Data

The requirement to benchmark against both buy-and-hold AND the S&P 500 introduces a data dependency that was not present in v1.

**NQ2.1: Where does S&P 500 benchmark data come from?**
- Priority: **SHOULD ANSWER DURING DESIGN**
- Options:
  - (a) User provides a second CSV file containing S&P 500 data
  - (b) Application fetches S&P 500 data automatically via yfinance or similar API
  - (c) Application ships with built-in S&P 500 historical data (stale over time)
- Impact: Option (a) keeps the application fully offline but adds user burden. Option (b) introduces a network dependency but is seamless. Option (c) is unreliable for daily runs.
- Recommended: Option (a) as primary (user provides SPY or ^GSPC CSV), option (b) as automatic fallback if yfinance is installed.

**NQ2.2: Total return or price return for the S&P 500 comparison?**
- Priority: **CAN DEFER**
- Context: The sample NVDA data appears to be price-only (no adjusted close column for dividends). If the S&P 500 comparison uses total return (with dividends reinvested) while the stock analysis uses price return only, the comparison is not apples-to-apples.
- Recommended: Use price return for both. Document the limitation. If adjusted close is available, use it for both.

### NQ3: Definition of "Most Profitable"

The core objective -- finding "the most profitable" indicators -- requires a precise definition for the ranking algorithm.

**NQ3.1: How is "most profitable" defined for the purpose of ranking strategies?**
- Priority: **MUST ANSWER BEFORE IMPLEMENTATION**
- Options:
  - (a) Total return over the full period
  - (b) Annualized return (CAGR)
  - (c) Risk-adjusted return (Sharpe ratio)
  - (d) Composite score (weighted blend of return, risk, consistency)
- Impact: This is the objective function. It determines what the application optimizes for and what it reports as "best."
- Recommended default: Total return as the primary ranking metric, with a 30% maximum drawdown hard filter. Strategies that exceed 30% drawdown are excluded from the ranking regardless of return. Risk-adjusted metrics (Sharpe, Sortino) are reported but do not override total return in the primary ranking.

**NQ3.2: Is the 30% maximum drawdown a hard filter or a soft penalty?**
- Priority: **MUST ANSWER BEFORE IMPLEMENTATION**
- Context: Hard filter means any strategy exceeding 30% max drawdown is rejected outright, even if it has the highest total return. Soft penalty means drawdown is factored into a composite score but does not automatically disqualify.
- Impact: Hard filter is simpler to implement and produces cleaner results. Soft penalty is more nuanced but may surface high-return strategies that narrowly exceed the threshold.
- Recommended: Hard filter. Reject strategies exceeding 30% max drawdown. This is a risk management constraint, not a scoring dimension. Report rejected strategies separately in verbose mode so the user can see what was filtered out.

### NQ4: Trade Frequency Precision

The 30-day trade frequency target needs precise definition for implementation.

**NQ4.1: Is 30 days the average holding period or the average time between signals?**
- Priority: **SHOULD ANSWER DURING DESIGN**
- Context: Average holding period = average duration of each trade (entry to exit). Time between signals = average gap between one trade's exit and the next trade's entry. For a strategy that is "always in" (exits one trade and immediately enters the next), these are the same. For a strategy with cash periods between trades, they differ significantly.
- Recommended: Average holding period (time in each trade). This is more intuitive and directly relates to how long capital is committed per position.

**NQ4.2: How strictly should the 30-day target be enforced?**
- Priority: **SHOULD ANSWER DURING DESIGN**
- Context: Should the application reject strategies whose average holding period is far from 30 days (e.g., a strategy with 3-day average trades or 200-day average trades)? Or should deviation from the target be a scoring penalty?
- Recommended: Soft constraint. Apply a scoring penalty that increases with distance from the target. Do not reject strategies outright based on trade frequency alone. A strategy with a 15-day or 60-day average holding period should still be ranked, but with a penalty relative to a strategy with a 30-day average.

### NQ5: Combination Limit Interpretation

**NQ5.1: Does "maximum 5 combinations" mean 5 tested or 5 reported?**
- Priority: **SHOULD ANSWER DURING DESIGN**
- Context: Testing only 5 combinations limits the search space severely. Reporting only 5 allows broader internal testing while keeping the output digestible.
- Recommended: Test more combinations internally (the proven pairings from research, plus systematic 2-indicator combinations from different categories). Report the top 5 in the primary output. Full results available in verbose mode.

### NQ6: Data Format Details

**NQ6.1: Are dollar signs present in price columns?**
- Priority: **CAN DEFER**
- Resolution: Auto-detect and strip. The data parser should handle `$123.45` and `123.45` transparently.

**NQ6.2: Are variant column names expected (e.g., "Close/Last" vs "Close")?**
- Priority: **CAN DEFER**
- Resolution: Build a column name mapping with common aliases. The Nasdaq format uses "Close/Last"; Yahoo Finance uses "Close"; other sources use "Adj Close." Map all known variants.

**NQ6.3: Is the data sorted ascending or descending by date?**
- Priority: **CAN DEFER**
- Resolution: Auto-detect sort order and normalize to ascending (oldest first). The Nasdaq format is descending (newest first); most other formats are ascending.

**NQ6.4: How should stock split and dividend adjustments be handled?**
- Priority: **SHOULD ANSWER DURING DESIGN**
- Known: The sample NVDA data appears to be split-adjusted but NOT dividend-adjusted (no "Adj Close" column).
- Impact: If the data is not split-adjusted, indicator calculations will produce false signals at split dates. If not dividend-adjusted, total return calculations will understate actual returns.
- Recommended: Accept data as-is. Implement a split detection heuristic (flag days where Close/Previous Close ratio is near a common split factor like 0.5, 0.25, 0.333). Warn the user if potential unadjusted splits are detected. Do not attempt to correct splits automatically.

**NQ6.5: Should the application support multiple CSV formats or only Nasdaq format?**
- Priority: **CAN DEFER**
- Recommended: Build a Nasdaq format parser as the primary implementation. Add a configurable column mapping in YAML for other formats. This covers the immediate need while supporting future data sources without code changes.

### NQ7: Position Management

**NQ7.1: What return is assumed on cash (when not in a position)?**
- Priority: **CAN DEFER**
- Recommended: Zero return on cash. This is the simplest assumption and is conservative (actual cash earns interest). Can be made configurable later.

**NQ7.2: Is position sizing 100% in/out?**
- Priority: **MUST ANSWER BEFORE IMPLEMENTATION**
- Context: For single-instrument, long-only analysis, the simplest model is 100% of capital in when a buy signal fires, 100% to cash when a sell signal fires. Alternative models (fractional Kelly, risk-parity, fixed-dollar) add complexity without clear benefit for single-instrument analysis.
- Recommended: Yes, 100% in/out as the default. Make it configurable for future flexibility, but implement and test with 100% in/out.

### NQ8: Forward-Looking Output

**NQ8.1: Is this a historical analyzer or an actionable signal tool?**
- Priority: **MUST ANSWER BEFORE IMPLEMENTATION** (overlaps NQ1.2)
- Context: This is the same question as NQ1.2 framed differently. A historical analyzer says "over the last 10 years, RSI was the best indicator." An actionable signal tool says "RSI is signaling BUY as of today's close."
- Recommended: Both. The primary output is historical analysis with ranking. The secondary output is the current signal state of the top-ranked strategies applied to the most recent data.

### NQ9: Options Roadmap

**NQ9.1: Does the future options extension affect v1 architecture?**
- Priority: **CAN DEFER**
- Context: The user mentioned options as a future direction. If options support requires fundamental architectural changes, those should be anticipated in v1.
- Recommended: No impact on v1 architecture. The current pipeline (Load -> Validate -> Calculate -> Signal -> Backtest -> Report) is agnostic to instrument type. For future readiness, define an abstract Instrument model and a PositionLeg model so that options (which have strikes, expirations, and Greeks) can be added without refactoring the core pipeline.

---

## 4. Prioritized Question List

### Block 1: Must Answer Before Implementation

These five questions determine fundamental architectural decisions. Proceeding without answers risks significant rework.

| ID | Question | Recommended Default | Why It Blocks |
|----|----------|-------------------|---------------|
| NQ1.1 | Re-analyze daily or lock strategy? | Re-analyze (stateless) | Determines state management architecture |
| NQ1.2 | Include actionable daily signal? | Yes | Determines output format and signal pipeline |
| NQ3.1 | Definition of "most profitable"? | Total return with drawdown filter | Core objective function for ranking |
| NQ3.2 | 30% drawdown: hard filter or soft? | Hard filter | Affects ranking algorithm design |
| NQ7.2 | Position sizing 100% in/out? | Yes | Affects backtest engine and metrics |

**If no answer is received:** Proceed with all recommended defaults. They form a coherent, simple system that can be adjusted later.

### Block 2: Should Answer During Design

These six questions affect specific design choices but have reasonable defaults that do not risk major rework.

| ID | Question | Recommended Default | Design Impact |
|----|----------|-------------------|---------------|
| NQ1.3 | Handling recommendation instability? | Show consensus across top 5 | Report format design |
| NQ2.1 | S&P 500 data source? | User CSV primary, yfinance fallback | Data loader interface design |
| NQ4.1 | 30 days = holding period or signal gap? | Average holding period | Scoring function definition |
| NQ4.2 | Trade frequency enforcement strictness? | Soft penalty | Scoring function weights |
| NQ5.1 | "Max 5 combinations" = tested or reported? | 5 reported, more tested | Combination engine scope |
| NQ6.4 | Split/dividend adjustment handling? | Accept as-is, warn on detection | Data validator design |

### Block 3: Can Defer to Development

These twelve questions can be resolved during implementation with reasonable defaults and no architectural risk.

| ID | Question | Recommended Default |
|----|----------|-------------------|
| Q1.3 | Data length expectations beyond sample? | Validate >= 1 year, warn < 3 years |
| Q4.4 | PDF or HTML output? | HTML default, PDF optional |
| Q5.3 | Regime feedback into evaluation? | ADX as combination option |
| Q6.1 | Docker acceptable? | Optional, not required |
| Q6.3 | Python version minimum? | 3.11+ |
| Q7.3 | Custom indicator definitions? | Not in v1 |
| NQ2.2 | Total return vs price return for S&P? | Price return for both |
| NQ6.1 | Dollar signs in data? | Auto-strip |
| NQ6.2 | Variant column names? | Auto-map common aliases |
| NQ6.3 | Data sort order? | Auto-detect, normalize ascending |
| NQ6.5 | Multiple CSV formats? | Nasdaq primary + configurable mapping |
| NQ7.1 | Cash return assumption? | Zero |
| NQ9.1 | Options roadmap impact on v1? | No impact; add abstract models |

---

## 5. Implications for Implementation

### 5.1 Architectural Simplifications from Answered Questions

The following v1 design elements can be removed or significantly simplified based on answered questions:

| v1 Design Element | v2 Change | Reason |
|-------------------|-----------|--------|
| Asset class configuration and per-class parameter branching | **Remove** | Equities only; no multi-asset logic needed |
| Volume indicator disable logic for forex | **Remove** | Equities always have volume data |
| Crypto threshold adjustments (RSI 80/25) | **Remove** | No crypto support |
| Commodity contract roll detection | **Remove** | No commodity support |
| Long-short backtesting logic | **Remove** | Long-only; sell = exit to cash |
| Multi-instrument data alignment | **Remove** | Single instrument per run |
| Portfolio-level risk metrics | **Remove** | Single instrument analysis |
| Pattern detection pipeline stage (15/30/60 day windows) | **Replace** with trade frequency scoring | 30-day value is holding period target, not pattern window |
| Web interface consideration | **Remove** | CLI/batch only |
| Interactive dashboard mode | **Remove** | Report generation, not live monitoring |

### 5.2 Architectural Additions from New Requirements

| New Requirement | Architectural Impact |
|----------------|---------------------|
| S&P 500 dual benchmark | Add benchmark data loader (second CSV or API). Benchmark comparison in metrics and reporting. |
| Daily actionable signal (NQ1.2) | Add a "current signal" module that applies the top strategy to the most recent bar and reports the current position state. |
| Recommendation stability tracking (NQ1.3) | Add a consensus report showing agreement/disagreement among top 5 strategies on the current signal. |
| 30% max drawdown hard filter (NQ3.2) | Add a pre-ranking filter stage that excludes strategies exceeding the drawdown threshold before composite scoring. |
| Trade frequency scoring (Q2.3 clarified) | Add a trade frequency penalty function to the composite scoring module, penalizing strategies whose average holding period deviates from the 30-day target. |
| 100% position sizing (NQ7.2) | Simplify backtest engine: binary in/out, no fractional sizing, no risk-based position calculation. |

### 5.3 Revised Pipeline

Based on answered questions, the v2 pipeline is:

```
Load CSV
  -> Validate (equity OHLCV, date range, split detection)
  -> Load Benchmark (S&P 500 CSV or API)
  -> Calculate Indicators (Tier 1+2, 12 indicators)
  -> Generate Signals (individual + combinations)
  -> Backtest (long-only, 100% in/out, next-bar-open execution)
  -> Walk-Forward Analysis (top performers from initial screen)
  -> Filter (reject strategies with >30% max drawdown)
  -> Score & Rank (total return primary, trade frequency penalty)
  -> Current Signal (apply top strategies to most recent bar)
  -> Report (HTML: summary + trade log + charts + current signal)
```

Key differences from v1 pipeline:
- Benchmark loading is a new stage
- No separate pattern detection stage
- Explicit drawdown filter before ranking
- Current signal generation is a new output stage
- Single output format (HTML) rather than multi-format consideration

### 5.4 Configuration Simplification

v1 YAML configuration included fields for asset class, multi-instrument, long-short toggle, pattern windows, and web interface. The v2 configuration is significantly simpler:

```yaml
# Core v2 configuration (simplified from v1)
data:
  file: "path/to/stock.csv"          # Required
  benchmark_file: "path/to/spy.csv"  # Optional (fallback: yfinance)
  date_column: "Date"                # Auto-detected if omitted
  column_mapping: {}                 # Auto-detected if omitted

analysis:
  indicators: "tier1+2"              # Default: all Tier 1 and Tier 2
  max_combinations_reported: 5       # Top N combinations in output
  target_holding_period_days: 30     # Trade frequency target

backtest:
  transaction_cost_pct: 0.1          # Round-trip percentage
  slippage_pct: 0.05                 # Per-trade slippage
  max_drawdown_pct: 30.0             # Hard filter threshold

ranking:
  primary_metric: "total_return"     # Objective function
  trade_frequency_penalty: true      # Penalize deviation from target

output:
  format: "html"                     # HTML with Plotly charts
  verbose: false                     # Full ranking + trade details
  include_current_signal: true       # Today's actionable signal
```

Fields removed from v1 configuration: `asset_class`, `trading_direction` (always long-only), `pattern_windows`, `multi_scale_analysis`, `web_interface`, `position_sizing_model` (always 100% in/out for v1).

---

## 6. Cross-Reference: v1 Questions to v2 Status

Quick lookup table mapping every v1 question to its current resolution.

| v1 ID | Short Description | v2 Status | Resolution Reference |
|-------|------------------|-----------|---------------------|
| Q1.1 | Target asset class | ANSWERED | Equities only |
| Q1.2 | Commodity spot vs futures | NOT APPLICABLE | No commodities |
| Q1.3 | Historical data time span | PARTIALLY ANSWERED | ~10 years sample; validate at runtime |
| Q1.4 | Multiple data providers | ANSWERED | CSV now, API future, single source per run |
| Q2.1 | Long-only or long-short | ANSWERED | Long-only |
| Q2.2 | Starting capital and sizing | NOT ANSWERED (defer) | Recommend 100% in/out |
| Q2.3 | Trade frequency constraints | ANSWERED | 30-day target, configurable |
| Q3.1 | Test combinations in v1 | ANSWERED | Yes, max 5 reported |
| Q3.2 | Preferred indicator set | NOT EXPLICIT (defer) | Recommend Tier 1+2 |
| Q4.1 | Trade detail level | ANSWERED | Entry/exit dates + P&L per trade |
| Q4.2 | Few-trade indicator handling | NOT ANSWERED (defer) | Minimum 20/window, 30+ total |
| Q4.3 | Recommendation vs ranked data | PARTIALLY ANSWERED | Best + top 5 + full in verbose |
| Q4.4 | PDF or HTML | NOT ANSWERED (defer) | HTML default |
| Q5.1 | What is a "pattern" | CLARIFIED | 30-day = trade frequency target |
| Q5.2 | Lookback window meaning | CLARIFIED | Independent of indicator params |
| Q5.3 | Pattern feedback to evaluation | NOT ANSWERED (defer) | ADX as combination option |
| Q6.1 | Docker acceptable | NOT ANSWERED (defer) | Optional |
| Q6.2 | CLI or web interface | EFFECTIVELY ANSWERED | CLI/batch |
| Q6.3 | Python version minimum | NOT ANSWERED (defer) | Recommend 3.11+ |
| Q7.1 | Single or multiple instruments | ANSWERED | Single instrument |
| Q7.2 | Real-time / forward-testing | PARTIALLY ANSWERED | Daily batch = forward testing |
| Q7.3 | Custom indicator definitions | NOT ANSWERED (defer) | Not in v1 |
| Q7.4 | Portfolio-level analysis | ANSWERED | No |

---

## Sources

- [notes/Research_Questions_v1.md](Research_Questions_v1.md) -- Original 33 questions
- [research/Research_v1.md](../research/Research_v1.md) -- v1 consolidated research
- [research/technical-indicators-research.md](../research/technical-indicators-research.md) -- Detailed indicator analysis
- [research/data-considerations-and-architecture.md](../research/data-considerations-and-architecture.md) -- Data engineering and architecture research
