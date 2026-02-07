# Research Questions v1

Additional requests for information and clarifications needed from the user to proceed with application design and development.

---

## Critical Questions (Needed Before Development)

### 1. What asset class(es) will be analyzed?

- Equities (stocks, ETFs)?
- Commodities (gold, oil, agricultural)?
- Forex (currency pairs)?
- Cryptocurrencies?
- Multiple asset classes?

**Why this matters**: Different asset classes have different characteristics:
- Equities have dividends, splits, corporate actions
- Commodities have seasonality, contango/backwardation
- Forex has no volume data from exchanges (tick volume only)
- Crypto trades 24/7 with no defined "daily" close time
- The indicator selection and data handling varies significantly by asset class

### 2. What is the data source?

- Will data be provided as CSV files?
- Should the application fetch data from an API (e.g., Yahoo Finance, Alpha Vantage)?
- Is there an existing data pipeline?
- What date range of historical data is available?

**Why this matters**: Determines data ingestion architecture, whether volume data is available, and how much historical data we have for backtesting (more data = more reliable statistical results).

### 3. Is volume data available?

- Some spot price data sources only provide OHLC without volume
- Volume is critical for several indicators (OBV, VWAP, A/D Line, CMF, MFI)
- If no volume data, we need to exclude volume-based indicators

**Why this matters**: If volume data is unavailable, we lose an entire category of indicators and the powerful volume confirmation technique (which improves accuracy from 54% to 72% for MA crossovers).

### 4. Long-only or Long/Short?

- Should the application only identify buy signals (long-only)?
- Or should it also identify short/sell-short opportunities?
- Or just long entry and exit (buy and sell to close)?

**Why this matters**: Long-only strategies are simpler and suitable for assets like stocks in retirement accounts. Long/short doubles the number of trading opportunities but requires margin capability and adds complexity.

---

## Important Questions (Needed for Design)

### 5. What is the target trade frequency?

- Position trading (weeks to months, few trades per year)?
- Swing trading (days to weeks, several trades per month)?
- Active daily trading (multiple signals per week)?

**Why this matters**: Determines which indicator parameters and lookback windows to prioritize. Position trading favors 50/200 SMA, daily RSI 14. Swing trading favors 10/20 EMA, shorter RSI periods.

### 6. What are realistic transaction costs?

- What brokerage/platform will be used?
- Are there per-trade commissions?
- What is the typical bid-ask spread for the assets being traded?
- Should slippage be modeled?

**Why this matters**: Transaction costs can erase backtested edge entirely. A 2023 study found results are "very sensitive to the introduction of moderate transaction costs." We need realistic cost assumptions to produce trustworthy profitability results.

### 7. Risk tolerance and drawdown limits?

- Is there a maximum acceptable drawdown (e.g., 10%, 20%, 30%)?
- Should the ranking system heavily penalize drawdown-prone indicators?
- Is there a maximum position size constraint?

**Why this matters**: Affects the weighting in the composite scoring system. A conservative user would weight MaxDD heavily (40%+); an aggressive user might weight Sharpe and total return more heavily.

### 8. Single asset or portfolio?

- Will the application analyze one asset at a time?
- Or should it handle a portfolio of multiple assets simultaneously?
- If portfolio, should it consider correlation between assets?

**Why this matters**: Portfolio analysis adds significant complexity (correlation matrices, portfolio-level metrics, diversification effects) and should be scoped separately if needed.

---

## Clarification Questions (Nice to Have)

### 9. Reporting format preferences?

- Terminal/console output?
- HTML report with charts?
- CSV export of trades?
- Interactive dashboard?
- PDF report?

**Why this matters**: Affects technology choices and development scope. A simple CSV trade log is very different from an interactive dashboard.

### 10. How should "configurable number of days" work?

The requirement mentions "identifying patterns over a configurable number of days, default 30." Clarification needed:
- Does this mean the lookback window for indicators (e.g., 30-day RSI)?
- Or the window for chart/candlestick pattern detection?
- Or the entire analysis period (only analyze the last 30 days of data)?
- Or all of the above?

### 11. Real-time or batch processing?

- Should the application run once on historical data and produce a report?
- Or should it update continuously as new daily data arrives?
- Or should it be able to do both?

**Why this matters**: Real-time processing requires a different architecture (event-driven, streaming) vs batch (run-once analysis).

### 12. Programming language preference?

- Python is the de facto standard for this type of analysis (best library ecosystem)
- Is Python acceptable, or is another language preferred?
- Any framework preferences (Flask, FastAPI, CLI-only)?

### 13. What does "best fit" mean to you specifically?

The research proposes a weighted composite score (Sharpe 30%, MaxDD 25%, Profit Factor 20%, Expectancy 15%, Win Rate 10%). Questions:
- Does this weighting align with your priorities?
- Is maximizing total return more important than minimizing risk?
- Should the application present multiple "best" indicators (top 5?) or just the single best?
- Should it recommend indicator *combinations* as well as individual indicators?

### 14. Benchmark comparison?

- Should indicator performance be compared against buy-and-hold?
- Any other benchmark strategies to compare against?
- This is considered best practice to demonstrate that active trading adds value over passive holding.

---

## Data-Specific Questions (Needed When Data Is Available)

### 15. Data quality assessment needed

Once we have access to the actual data:
- How many years of history are available? (Need enough for statistical significance)
- Are there gaps or missing periods?
- Is the data already adjusted for splits/dividends?
- What is the typical daily price range (affects ATR-based calculations)?
- Is volume data reliable or sparse?

### 16. Asset-specific characteristics

- Does the asset exhibit seasonality?
- Are there known structural breaks in the data (regime changes, policy shifts)?
- What is the typical volatility profile?
- Are there any known correlations with other assets we should account for?

---

## Summary of Priority

| Priority | Questions | Reason |
|----------|-----------|--------|
| **Must answer before coding** | #1 (asset class), #2 (data source), #3 (volume), #4 (long/short) | Fundamental architecture decisions |
| **Should answer before coding** | #5 (frequency), #6 (costs), #7 (risk), #8 (single/portfolio) | Design decisions |
| **Can answer during development** | #9-#14 | Implementation details |
| **Answer when data available** | #15-#16 | Data-dependent tuning |
