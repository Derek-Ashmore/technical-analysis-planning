# Research Thinking v2

This document describes the reasoning process and thinking behind the revised technical analysis research, given the confirmed requirements.

---

## What Changed from v1

The user provided answers to the critical and important questions from `Research_Questions_v1.md`. These answers resolved the major architectural uncertainties:

| Question | Answer | Impact |
|----------|--------|--------|
| Asset class | Equities only | Remove forex/commodity/crypto considerations; leverage equity-specific research |
| Data source | CSV (Nasdaq format) | Must handle $-prefixed prices, MM/DD/YYYY dates, reverse-chronological order |
| Volume data | Available in CSV | Keep all volume indicators (OBV, MFI, A/D Line, CMF); volume confirmation is a major advantage |
| Long/Short | Long-only | All bearish signals become exit signals; no short entries ever |
| Trade frequency | Configurable, default 30 days | Swing/position trading parameters; not intraday |
| Transaction costs | Not material | Still model at 0.05-0.15% for overfitting detection |
| Max drawdown | Configurable, default 30% | Hard constraint; auto-disqualify strategies exceeding this |
| Single/portfolio | Single asset | No portfolio correlation analysis needed |
| Benchmark | Buy-and-hold + S&P 500 | Need SPY data from yfinance; report Alpha, Beta, Information Ratio |
| Run cadence | Once daily | Two-phase architecture: daily signal + periodic optimization |
| Verbose mode | Required | Show rejected indicators with reasons |
| Max combinations | Configurable, default 5 | Search space is tractable; exhaustive enumeration feasible |

---

## Key Decisions and Reasoning

### Why Williams %R Was Added

v1 did not emphasize Williams %R despite it being a standard momentum oscillator. New backtest evidence from QuantifiedStrategies.com showed it is the single best-performing swing trading indicator on S&P 500 equities:
- 81% win rate with 2-day lookback
- 11.9% CAGR vs 7.3% for RSI
- Lowest max drawdown of all single indicators tested
- Profit factor above 2.0 across nearly all parameter sets

This made it a clear addition to the primary indicator set. The 2-period lookback is unusual (most indicators use 14+) but the evidence was compelling.

### Why backtesting.py Replaces vectorbt

This was a significant change from v1. The reasoning:

1. **vectorbt's free version is in maintenance-only mode**: The developer (polakowo) has shifted focus to vectorbt PRO (proprietary, paid, invite-only). The free version (0.21.0) still works but receives no new features.
2. **backtesting.py has built-in walk-forward optimization**: This is critical for the application's methodology. vectorbt requires custom implementation of walk-forward analysis.
3. **backtesting.py is actively maintained** (v0.6.5) with a simpler API for this use case.
4. **backtesting.py generates interactive Bokeh charts**: Useful for future HTML report output.

The trade-off is that vectorbt is faster (NumPy + Numba vs pure pandas). But for daily-frequency data on a single equity (~2500 bars), performance is not a bottleneck. Even a full recalculation takes milliseconds.

### Why Default to Cash in Ranging Markets

v1 recommended switching between trend-following and mean-reversion indicators based on ADX. For long-only, we revised this to default to cash in ranging markets (ADX < 20). The reasoning:

1. **The 30% max drawdown constraint** is the binding factor. Capital preservation is more important than capturing small mean-reversion gains.
2. **Markets trend only ~30% of the time**. The other 70% includes ranging, choppy, and transitional conditions where signals are unreliable.
3. **Long-only has a structural advantage**: Sitting in cash during uncertain periods and re-entering during trends still captures most of the upside due to equities' long bias.
4. **Mean-reversion for long-only is riskier**: In a true ranging market, buying dips can turn into catching falling knives if the range breaks to the downside.

We made mean-reversion mode configurable rather than eliminating it entirely. Some users may want the higher activity level.

### Why Dual-Price Approach

The most nuanced decision in v2 was how to handle prices for indicator calculation vs return calculation:

- **For indicators (split-adjusted only)**: Indicators like SMA, RSI, and MACD measure price momentum and trends. Dividend adjustments distort the price series and can create false signals (e.g., a $1 dividend on a $50 stock looks like a 2% price drop, triggering false SMA crossover or RSI sell signals).
- **For returns (fully adjusted)**: When computing PnL and comparing against benchmarks, dividends are real returns that must be counted.

This creates a design tension: the application must maintain two price series and know which to use when. The Nasdaq CSV provides raw (unadjusted) prices, so the application must:
1. Apply split factors from yfinance to get split-adjusted prices (for indicators)
2. Apply full adjustment from yfinance to get adjusted close (for returns)
3. Detect when the CSV data has already been adjusted (or not) and handle both cases

### Why Revised Composite Scoring Weights

The v1 weights (Sharpe 30%, MaxDD 25%, PF 20%, Expectancy 15%, WR 10%) were general-purpose. With the confirmed 30% max drawdown constraint:

- **MaxDD increased to 30%**: It's now the hard constraint. Any strategy exceeding this is automatically disqualified. The remaining scoring should further differentiate among strategies that pass the filter.
- **Calmar Ratio added at 10%**: Directly measures return-per-unit-of-drawdown, perfectly aligned with the drawdown constraint. This wasn't in v1 because the constraint wasn't known.
- **Win Rate increased to 15%**: For long-only mean-reversion strategies (which the application supports), win rate matters more than for trend-following. Higher win rate also provides better psychological comfort for the user.
- **Expectancy reduced to 5%**: Expectancy is partially redundant with win rate + profit factor.

### Why Typer + Rich + Pydantic Settings

The architecture research evaluated several options for each component:

- **Typer over Click/argparse**: Type-hint-based API with auto-documentation. Less boilerplate than Click. The --verbose flag and subcommands map naturally. Created by the FastAPI author, so well-designed and maintained.
- **Rich over tabulate/plain text**: Color-coded tables make profit/loss instantly scannable. Progress bars during optimization runs. Well-maintained (v14.2.0, Jan 2026).
- **Pydantic Settings over Hydra/Dynaconf/plain YAML**: Type-safe validation catches configuration errors early. Supports YAML files + env vars + CLI overrides with clear precedence. Integrates naturally with the type-hint stack (Typer + Pydantic).
- **uv over Poetry/pip**: 10-100x faster dependency resolution. Native pyproject.toml support. Lock files for reproducibility. Single binary that replaces multiple tools.

### Why Exhaustive Search Is Feasible

The max 5 indicator combinations default sounds like it could create a large search space, but the hierarchical filtering architecture constrains it:
- Layer 1 (Trend filter): ~3-5 candidates
- Layer 2 (Confirmation): ~5-8 candidates
- Layer 3 (Entry signal): ~5-8 candidates

This gives ~108 valid layer combinations (3 × 6 × 6), not C(N,5) = thousands. Exhaustive enumeration is absolutely feasible. However, we recommend applying the Deflated Sharpe Ratio correction since even 108 tests can surface false positives.

---

## What Surprised Us in v2

### Parabolic SAR Performance Is Terrible

v1 included Parabolic SAR as a viable indicator. Deeper backtest evidence in v2 revealed only 19% win rate on DJIA 30 stocks over 12 years (source: PatternsWizard). It generates excessive whipsaws in the ~70% of the time markets are ranging. It's now deprioritized and only usable with an ADX > 25 filter (which negates much of its value since you'd already know you're in a trend).

### Chandelier Exit Is Clearly Superior

The Chandelier Exit (ATR-based trailing stop) emerged as clearly superior to fixed percentage stops, Parabolic SAR, and even MA-based exits. It adapts to volatility, prevents premature stop-outs, and is widely used by equity position traders. This was present but not sufficiently emphasized in v1.

### NVDA's Extraordinary Performance Is a Trap

NVDA had extraordinary returns during the AI boom (2015-2024). Backtesting on this data alone would make almost any strategy look good due to survivorship bias and the unprecedented single-stock performance. The application needs to:
- Warn users that results are specific to the tested equity
- Report excess return over buy-and-hold (not just absolute return) to strip out the stock's inherent performance
- Compare against S&P 500 to show whether the strategy adds value beyond market beta

### Transaction Costs Still Matter (Even When "Not Material")

The user stated transaction costs are not material. However, the research found that strategies overstate returns by 3.7-8.2% annually without cost modeling. Including a small cost (0.05-0.15% per trade) serves as an overfitting filter: a strategy that only works with zero friction is likely overfitted. We recommend modeling costs at a low default even though they're described as immaterial.

---

## Research Approach for v2

### Three Parallel Research Workstreams

We decomposed the revision into three parallel agent teams:

1. **Indicators Research** -- How the indicator recommendations, signal rules, and combinations should change for equities, long-only, with volume. This was the largest workstream.
2. **Methodology Research** -- How the backtesting methodology, benchmarking, data handling, and statistical validation should change for CSV input, daily cadence, and S&P 500 benchmarking.
3. **Architecture Research** -- How the application stack, CLI design, configuration, and extensibility should work given the confirmed requirements and future roadmap.

### Compilation Approach

The team lead compiled all findings into a single cohesive document (Research_v2.md) rather than delegating compilation to a fourth agent. This produced a more coherent result with consistent structure and cross-references between sections.

---

## Risks and Concerns

### Library Dependency Risk

The application depends on several external libraries:
- **TA-Lib**: Requires C library installed system-wide. This is a friction point for users.
- **yfinance**: Scraping-based; could break if Yahoo Finance changes their API. Intended for personal use.
- **backtesting.py**: Smaller project than vectorbt; less community support.

Mitigation: Use abstractions (DataProvider, Strategy) so libraries can be swapped without rewriting the application.

### Single-Equity Overfitting

Testing indicator combinations on a single equity over 10 years is inherently limited. The statistical significance requirements (30+ trades, WFE > 50%, DSR correction) mitigate this, but results should always be presented with appropriate caveats.

### Complexity vs Usability (Carried from v1)

The revised research adds even more complexity (dual-price approach, CPCV, DSR, earnings handling). The application must present clear, actionable results despite this internal complexity. The verbose mode is the release valve: simple results by default, full detail on request.
