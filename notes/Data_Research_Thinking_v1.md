# Research Thinking v1: Data Considerations and Architecture

This document describes the reasoning process and key decisions behind the data considerations, library evaluation, visualization, and architecture research.

---

## Research Approach

### How This Research Was Structured

This research was structured around six topics that represent the foundational engineering questions for building a technical analysis application. The existing research (in the sibling project and in this branch) thoroughly covers indicators, backtesting methodology, profitability metrics, and indicator combination strategies. This research fills the identified gaps:

1. **Spot Price Data Characteristics** -- What exactly is the data we are working with, and how does it differ from other financial data types?
2. **Asset Class Considerations** -- How does the choice of asset class affect indicator selection and data handling?
3. **Data Storage and Processing** -- What formats, cleaning approaches, and volume considerations apply?
4. **Python Libraries** -- Which libraries should power each component, and why?
5. **Visualization** -- How should results be presented?
6. **Architecture** -- How should the application be structured for maintainability and extensibility?

These topics were identified as gaps in the existing Research_Questions_v1.md documents, which explicitly called out "specific asset class behavior," "data source," and "reporting format" as open questions.

### Why These Topics Matter

Every one of these topics directly impacts implementation decisions:

- Getting spot vs futures data wrong means indicator calculations are applied to the wrong type of data.
- Ignoring asset class differences means using volume indicators on forex data (where volume does not exist) or applying standard RSI thresholds to crypto (where they are too tight).
- Choosing the wrong file format or library creates unnecessary friction during development.
- Poor visualization choices mean the user cannot interpret results effectively.
- A weak architecture makes it expensive to add new indicators or change the reporting format.

---

## Key Decisions and Reasoning

### Why CSV as Primary Input

CSV was chosen as the primary input format because:
- It is universally understood and can be created from any spreadsheet, database, or data provider.
- The user can inspect and edit it directly.
- For daily data (even 50 years), file sizes are trivially small (< 1 MB).
- Every Python data library reads CSV natively.
- It imposes no installation requirements on the user.

Parquet is recommended for internal caching only, not for user-facing input, because it requires pyarrow and is not human-readable.

### Why TA-Lib Over pandas-ta as Primary

Despite pandas-ta having a cleaner API, TA-Lib was chosen as the primary indicator library because:
- It is the reference implementation -- other libraries validate against it.
- The C implementation is 2-5x faster, which matters during parameter optimization.
- It has the most comprehensive candlestick pattern library (60+ patterns vs fewer in alternatives).
- It has been used in production trading systems for 20+ years.
- The sustainability concern with pandas-ta (potential archival by July 2026) is a real risk.

The installation friction with TA-Lib (needing to compile the C library) is the main downside. The research recommends documenting clear installation instructions and supporting pandas-ta as a fallback.

### Why vectorbt Over Backtrader or Zipline

vectorbt was chosen as the backtesting framework because:
- The application's core use case is testing 27+ indicators across multiple parameter combinations. This is a parameter sweep problem, and vectorbt is 10-100x faster than event-driven frameworks for this.
- Backtrader has been unmaintained since 2018 -- using it in a new project is not advisable.
- Zipline's installation difficulties and US-equity focus do not align with a multi-asset spot price tool.
- vectorbt's built-in performance metrics (Sharpe, Sortino, Calmar, drawdowns, trade logs) overlap significantly with what the application needs.

The main risk with vectorbt is its learning curve (the vectorized paradigm differs from traditional backtesting). However, the performance gain justifies the investment.

### Why Pipeline Architecture

A pipeline architecture (load -> validate -> calculate -> signal -> backtest -> report) was chosen over alternatives because:
- Each stage has a clear input and output, making testing straightforward.
- Stages can be developed and debugged independently.
- The pipeline can be short-circuited (e.g., stop after indicator calculation for exploration).
- It naturally maps to the application's workflow: data in, analysis, results out.
- It is the simplest architecture that supports all requirements.

An event-driven or microservice architecture would be overengineered for a batch analysis tool.

### Why YAML for Configuration

YAML was chosen over JSON, TOML, or Python config files because:
- It is the most human-readable format for nested configuration.
- It supports comments (unlike JSON).
- It handles lists and nested structures cleanly (the indicator configuration is deeply nested).
- PyYAML is a lightweight, well-maintained dependency.
- TOML would also work but is less natural for deeply nested structures.

### Why Both Static and Interactive Charts

The research recommends generating both static (mplfinance) and interactive (Plotly) charts because they serve different consumption patterns:
- Static charts are essential for PDF reports and automated pipelines.
- Interactive charts are essential for exploratory analysis (zooming into specific trades, hovering for exact values).
- The cost of supporting both is low -- mplfinance and Plotly both integrate with pandas DataFrames.

### Why Asset Class Modes

The research identifies significant differences in indicator effectiveness across asset classes:
- Forex has no volume data, making volume indicators useless.
- Crypto needs adjusted RSI thresholds (80/25 instead of 70/30) and wider stops.
- Commodities favor trend-following more than mean-reversion.
- Equities have the richest data and support all indicator types.

Rather than ignoring these differences, the application should offer asset-class-aware defaults. This does not require different code paths -- just different default configurations.

---

## What Surprised Us

### Daily Data Volumes Are Trivially Small

Even 50 years of daily data for a single asset is under 1 MB in CSV. This means:
- File format performance is irrelevant for input data.
- The entire dataset fits comfortably in memory.
- No need for database infrastructure, streaming, or chunked processing.
- The bottleneck is computation (parameter optimization), not I/O.

This simplifies the architecture considerably. The recommendation for Parquet caching is more about type preservation and convenience than performance.

### Forex Volume Data Does Not Exist

Forex has no centralized exchange, so there is no "real" volume data. What brokers provide is tick volume (number of price changes), which is a weak proxy for actual traded volume. This means:
- An entire category of indicators (OBV, A/D Line, CMF, MFI, VWAP) is effectively unusable for forex.
- Volume confirmation -- which improves Golden Cross accuracy from 54% to 72% -- is not available for forex.
- The application must detect when volume data is absent or unreliable and automatically disable volume-dependent indicators.

This was not addressed in the prior research and is a significant design consideration.

### Crypto's 24/7 Trading Creates Ambiguity

Unlike traditional markets where "daily close" has a precise meaning (e.g., 4:00 PM ET for NYSE), crypto trades continuously. The "daily close" depends on the data source:
- CoinGecko uses midnight UTC.
- Binance uses midnight UTC.
- Some sources use midnight in the user's local timezone.

This means the same "January 15 daily bar" can represent different time periods depending on the source. The application should standardize on midnight UTC and document this convention.

### The Adjusted Close Problem Is Subtle

For equities, using unadjusted prices creates invisible errors:
- A stock that paid 20 quarterly dividends over 5 years has its entire historical price series shifted downward by the cumulative dividend amount when using adjusted close.
- Indicators calculated on adjusted close will give different signals than those calculated on unadjusted close.
- The difference is small on any given day but compounds over time.
- Using unadjusted close and then adding dividends back to returns is a common error that double-counts dividend income.

The application should require adjusted close for equities and provide a validation check.

---

## Risks and Concerns

### TA-Lib Installation Complexity

The single biggest adoption barrier for the recommended stack is TA-Lib installation. The C library must be compiled from source on some systems. Mitigations:
- Provide detailed installation instructions for Linux, macOS, and Windows.
- Support conda installation (conda-forge provides pre-built binaries).
- Use pandas-ta as an automatic fallback if TA-Lib is not installed.
- Consider Docker as a deployment option (pre-built image with TA-Lib).

### vectorbt Learning Curve

vectorbt's vectorized paradigm is unfamiliar to developers used to event-driven backtesting. The main conceptual shift is: instead of iterating through bars and making decisions, you generate boolean entry/exit arrays for the entire series at once, then pass them to the portfolio simulator. Mitigation:
- Wrap vectorbt behind a simple API (the backtest engine in the architecture).
- Provide clear examples for each indicator type.
- The custom code should be in signal generation, not in the backtesting framework itself.

### Report Complexity vs Usability

The recommended report structure (5 sections, multiple charts, trade logs) could be overwhelming. The application should:
- Lead with the summary page (top 5 indicators, key metrics, benchmark comparison).
- Make detailed sections available but not front-and-center.
- Provide a "quick mode" that generates only the summary.

---

## Connections to Prior Research

This research directly addresses several questions raised in the existing Research_Questions_v1.md:

| Question from Research_Questions_v1.md | Addressed In |
|---------------------------------------|-------------|
| Asset class(es) to be analyzed | Section 2: Full asset class analysis with indicator effectiveness comparison |
| Data source | Section 1.6: Data sources table; Section 6.2: Configurable data source architecture |
| Volume data availability | Section 2.2 (forex), Section 2.3 (crypto): Volume availability by asset class |
| Reporting format preferences | Section 5: Full visualization and reporting analysis |
| Programming language preference | Section 6.8: Recommended technology stack |
| Data quality assessment | Section 1.5: Data quality issues and cleaning pipeline |

The remaining unanswered questions (long/short, trade frequency, transaction costs, risk tolerance, single/portfolio, configurable days definition, real-time vs batch, best fit definition, benchmark) require user input and cannot be resolved through research alone. The architecture in Section 6 is designed to accommodate all possible answers to these questions via configuration.
