# Research Questions v1: Consolidated Information Requests

Questions arising from all four research workstreams that would help refine the application design and implementation.

---

## Priority Summary

| Priority | Questions | Impact |
|----------|-----------|--------|
| **Before implementation** | Q1.1, Q2.1, Q2.2, Q3.1, Q5.1, Q7.1, Q7.2 | Determines scope and architecture |
| **During design** | Q1.2, Q1.3, Q3.2, Q4.1, Q5.2, Q6.1, Q6.2 | Affects specific design decisions |
| **During development** | Q4.2, Q4.3, Q4.4, Q5.3, Q6.3, Q7.3, Q7.4 | Implementation details |

---

## 1. Asset Class and Data Context

**Q1.1: What specific type of spot price data will the application analyze?**
The effectiveness of technical analysis varies significantly by asset class. Academic evidence is strongest for commodities and FX, weaker for developed market equities. The target asset class determines:
- Which indicators to prioritize (trend-following for commodities, mean-reversion for equities)
- Whether volume indicators can be used (not for forex)
- Default parameter adjustments (wider stops for crypto)
- Data source recommendations

**Q1.2: For commodity data, does "spot price" refer to physical spot or front-month futures proxy?**
Many commodity "spot price" datasets (including FRED) report front-month futures prices as a proxy. True physical spot prices may only be available for precious metals. This affects whether we need to handle contract roll artifacts.

**Q1.3: What is the expected time span of the historical data?**
Minimum 3-5 years recommended for reliable backtesting. 10+ years enables walk-forward analysis with sufficient cycles. This informs the recommended testing methodology and which indicators can be used (200-day SMA needs 200 bars of warmup).

**Q1.4: Will the application need to support multiple data providers simultaneously?**
(e.g., gold from FRED, stocks from Yahoo Finance) This would require a unified data loader interface with provider-specific adapters.

---

## 2. Trading Assumptions

**Q2.1: Should the application assume long-only trading, or long and short?**
Many indicators generate both buy and sell signals. Should sell signals mean "exit long only" or also "enter short"? Short selling introduces borrowing costs and different dynamics. This significantly affects backtest results and complexity.

**Q2.2: What is the assumed starting capital and position sizing model?**
Options:
- Fixed fractional (e.g., always invest 100% of capital)
- Fixed dollar amount per trade
- Risk-based sizing (e.g., risk 1-2% per trade based on ATR stop distance) -- **recommended**

**Q2.3: Are there constraints on trade frequency?**
Should the application avoid entering new positions too soon after exiting? Is there a maximum number of open positions (presumably 1 for single-instrument analysis)?

---

## 3. Indicator Selection and Scope

**Q3.1: Should the application test indicator combinations in v1?**
Testing individual indicators is manageable (~25 indicators x ~3 parameter sets = ~75 backtests). Testing combinations exponentially increases the search space (e.g., 25 choose 3 = 2,300 three-indicator combinations). This also increases overfitting risk. Recommendation: include 5-6 proven combinations (listed in research) but not exhaustive combinatorial search.

**Q3.2: Is there a preferred initial set of indicators, or should all 25 be included?**
The Tier 1 set (EMA, RSI, MACD, ATR, ADX, Bollinger Bands) covers the essential dimensions. Starting with Tier 1+2 (10 indicators) would be a practical v1 scope.

---

## 4. Output and Reporting

**Q4.1: What level of detail is expected in trade-by-trade reporting?**
Options range from:
- Summary statistics only (as described in requirements)
- Trade-by-trade list with entry/exit dates, prices, and P&L
- Full daily equity curve data
- Visualization (charts, heatmaps) or data-only output
All of the above are implementable; the question is which to include in v1.

**Q4.2: How should indicators that produce no trades (or fewer than 30) be handled?**
Options:
- Exclude from ranking entirely
- Include with "insufficient data" caveat
- Report but flag as statistically unreliable
Recommendation: Report with a flag; do not rank alongside indicators with sufficient trades.

**Q4.3: Should the application produce a definitive recommendation or just ranked data?**
The requirements say "tell me what indicators best fit the data." Should the output be "The best indicator is X" or a ranked list with metrics? Recommendation: ranked list with the top indicator highlighted, plus a caveat about statistical confidence and market regime dependence.

**Q4.4: Is PDF output required, or is HTML sufficient?**
HTML reports support interactive Plotly charts and are easier to generate. PDF adds a dependency (WeasyPrint) and limits interactivity. If HTML is sufficient, the implementation is simpler and the output is richer.

---

## 5. Pattern Recognition

**Q5.1: What constitutes a "pattern" in the context of the configurable N-day window?**
The requirements mention "identifying patterns over a configurable number of days." This could mean:
- (a) Which indicator signals occurred in the last N days
- (b) What price action patterns (higher highs/lows) formed in the last N days
- (c) What market regime (trending/ranging) is detected based on the last N days
- (d) All of the above
Recommendation: (d) all of the above, with regime detection as the primary pattern output.

**Q5.2: Should the configurable lookback window (default 30 days) apply to:**
- (a) Pattern detection window only
- (b) The entire analysis period (only analyze the last 30 days)
- (c) All indicator lookback periods simultaneously
- (d) A separate, independent setting from indicator parameters
Recommendation: (d) -- the 30-day window is for pattern detection, independent of indicator-specific parameters (RSI=14, MACD=12/26/9, etc.).

**Q5.3: Should pattern recognition feed back into indicator evaluation?**
For example, should the application test whether indicators are more profitable during specific pattern conditions (e.g., "RSI works better during detected downtrends")? This is the regime-dependent evaluation and is recommended, but adds scope.

---

## 6. Technical Implementation

**Q6.1: Is Docker an acceptable deployment mechanism?**
Docker would solve the TA-Lib installation problem (pre-built image with C dependencies). If Docker is acceptable, installation documentation becomes trivial.

**Q6.2: Should the application be CLI-only, or include a web interface?**
The proposed architecture is CLI-first (run a command, get a report). A web interface (Dash, Streamlit, FastAPI) would add significant scope but improve usability. This is a major scope decision.

**Q6.3: What Python version is the minimum target?**
Recommended: Python 3.10+ (for match statements, type union syntax). If constrained to older versions, alternative syntax is needed.

---

## 7. Scope and Future Considerations

**Q7.1: Will the application analyze a single instrument at a time, or compare across multiple instruments?**
Single-instrument analysis has different requirements than multi-instrument portfolio analysis. The architecture supports both, but multi-instrument adds correlation analysis, portfolio-level metrics, and data alignment complexity.

**Q7.2: Is real-time or forward-testing capability planned for a future version?**
If so, the architecture should accommodate this from the start (event-driven compatibility). The current recommendation (vectorbt + pipeline) is optimized for batch analysis.

**Q7.3: Will the application support custom indicator definitions?**
The proposed Indicator base class + registry pattern supports this naturally, but it affects documentation and testing scope.

**Q7.4: Should the application eventually support portfolio-level analysis?**
Testing indicators across multiple instruments with correlation analysis and portfolio-level risk metrics is a natural extension but requires significantly different architecture considerations.

---

## Most Critical Questions (Top 5)

1. **Q1.1 -- Target asset class(es)?** Determines indicator selection, default parameters, volume handling, and data source.
2. **Q2.1 -- Long-only or long-short?** Fundamentally changes backtest results and system design.
3. **Q3.1 -- Test combinations in v1?** Major scope decision affecting timeline.
4. **Q5.1 -- What is a "pattern"?** Clarifies the 30-day window feature.
5. **Q7.1 -- Single or multiple instruments?** Determines data handling and architecture scope.
