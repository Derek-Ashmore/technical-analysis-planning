# Technical Analysis Research: Data Considerations, Libraries, Visualization, and Architecture

## Executive Summary

This report provides comprehensive research on the data engineering, library ecosystem, visualization, and architecture dimensions of a technical analysis application for daily spot price data. It complements the existing indicator-focused research (covering 27+ indicators, backtesting methodology, profitability metrics, and indicator combination strategies) by addressing the foundational questions that must be answered before implementation: what is spot price data, how does it behave across asset classes, how should it be stored and processed, which Python libraries should power the system, how should results be visualized, and what application architecture best supports the requirements.

**Key findings:**

- Spot price data differs fundamentally from futures and options data, and the distinction matters for indicator applicability and data cleaning.
- Indicator effectiveness varies significantly by asset class. Trend-following indicators outperform in commodities and forex; mean-reversion indicators work better in equities; crypto requires adaptive approaches due to extreme volatility.
- For daily data, CSV is the simplest viable format; Parquet is preferred for performance at scale. A pandas DataFrame with DatetimeIndex is the universal in-memory representation.
- TA-Lib remains the best indicator computation library despite installation friction. pandas-ta is a strong alternative with sustainability concerns. vectorbt is the best backtesting framework for this use case.
- mplfinance for static candlestick charts and Plotly for interactive exploration cover all visualization needs. PDF/HTML report generation via Jinja2 templates provides professional output.
- A pipeline architecture (load -> validate -> calculate -> signal -> backtest -> report) with a modular indicator engine and YAML-based configuration provides the best balance of simplicity, extensibility, and maintainability.

---

## Table of Contents

1. [Spot Price Data Characteristics](#1-spot-price-data-characteristics)
2. [Asset Class Considerations for Technical Analysis](#2-asset-class-considerations-for-technical-analysis)
3. [Data Storage and Processing](#3-data-storage-and-processing)
4. [Python Libraries for Technical Analysis](#4-python-libraries-for-technical-analysis)
5. [Visualization and Reporting](#5-visualization-and-reporting)
6. [Application Architecture Considerations](#6-application-architecture-considerations)

---

## 1. Spot Price Data Characteristics

### 1.1 What Constitutes "Spot Price" Data

**Spot price** (also called cash price or current price) is the price at which an asset can be bought or sold for immediate delivery. This is in contrast to:

| Price Type | Definition | Settlement | Key Differences from Spot |
|-----------|-----------|-----------|--------------------------|
| **Spot** | Current market price for immediate delivery | T+0 to T+2 depending on asset class | Baseline -- what we are analyzing |
| **Futures** | Agreed price for delivery at a future date | Contract expiration date | Includes cost of carry (storage, insurance, financing). Contango/backwardation creates basis risk. Rolls between contracts create artificial price jumps. |
| **Options** | Right (not obligation) to buy/sell at strike price | Exercise/expiration date | Price is a derivative of spot, includes time value and implied volatility premium. Not directly comparable to spot. |
| **Forward** | Like futures but OTC (over-the-counter), customized | Agreed future date | Similar to futures but with counterparty risk and no standardization. |

**Why this matters for the application:**

- The requirement specifies "historical daily spot price data." This means we are working with the simplest, most direct representation of an asset's value.
- Spot prices do not have contract roll issues (a major data cleaning problem with futures).
- Spot prices do not have time decay (a fundamental characteristic of options data).
- All standard technical analysis indicators were originally designed for and validated against spot/cash market data, so they apply directly without adjustment.
- However, for commodities, "spot price" can be ambiguous -- it sometimes refers to the front-month futures contract price, which is the most liquid proxy for spot. The user should clarify whether their commodity data represents true physical spot prices or front-month futures proxies.

### 1.2 OHLCV Data Format

The standard representation for daily price data is OHLCV:

| Field | Description | Usage in Technical Analysis |
|-------|------------|---------------------------|
| **Open** | First traded price of the day | Gap analysis, candlestick patterns, Ichimoku calculations |
| **High** | Highest traded price during the day | Bollinger Bands, ATR, Stochastic, Donchian Channels, candlestick patterns, True Range calculation |
| **Low** | Lowest traded price during the day | Same as High -- forms the range component |
| **Close** | Last traded price of the day (or settlement price) | Most critical field -- used by virtually all indicators (SMA, EMA, RSI, MACD, etc.) |
| **Volume** | Number of units traded during the day | OBV, VWAP, A/D Line, CMF, MFI, volume confirmation signals |

**Additional fields that may be present:**

| Field | Description | When to Use |
|-------|------------|------------|
| **Adjusted Close** | Close adjusted for dividends and splits | Required for equities when calculating returns over periods that include corporate actions |
| **Dividend** | Per-share dividend paid on the date | Only needed if computing adjusted close manually; do NOT add dividends if already using adjusted close (double-counting) |
| **Split Ratio** | Stock split or reverse split ratio | Only needed if data is not pre-adjusted |

**Data validation rules (must be enforced on load):**

1. `High >= max(Open, Close)` -- the high must be at or above both open and close
2. `Low <= min(Open, Close)` -- the low must be at or below both open and close
3. `High >= Low` -- always
4. `Volume >= 0` -- volume cannot be negative
5. `Close > 0` -- prices must be positive (for most assets)
6. Dates must be monotonically increasing with no duplicates

### 1.3 Data Frequency Considerations for Daily Data

The application uses daily data. Key considerations:

**Trading days vs calendar days:**

- Most traditional markets trade approximately 252 days per year (US equities).
- Forex trades approximately 260 days per year (Mon-Fri, most holidays observed).
- Cryptocurrency trades 365 days per year (no holidays, no weekends off).
- Commodity exchanges have their own holiday schedules (e.g., CME Group closes for about 9 US holidays per year).

**Implications for indicator calculations:**

- A "200-day moving average" means 200 trading days, not 200 calendar days.
- For equities, 200 trading days is roughly 10 calendar months.
- For crypto, 200 trading days is roughly 200 calendar days (about 6.5 months).
- This means the same indicator "period" covers different calendar durations depending on asset class. The application should be clear about whether parameters refer to trading days or calendar days. Convention: trading days.

**Daily data resolution trade-offs:**

| Aspect | Daily Data Advantage | Daily Data Limitation |
|--------|---------------------|----------------------|
| Signal quality | Fewer false signals than intraday | Misses intraday price extremes |
| Data volume | Manageable -- ~252 rows/year | Cannot capture intraday patterns |
| Indicator suitability | All standard indicators designed for daily | Some patterns (VWAP) are less meaningful |
| Analysis horizon | Best for swing and position trading | Not suitable for day trading strategies |
| Data availability | Widely available, often free | -- |
| Backtesting speed | Fast -- small datasets | -- |

### 1.4 Handling Missing Data, Gaps, and Holidays

**Types of gaps in daily data:**

1. **Weekends** (Sat-Sun for traditional markets): Expected, not missing data. The application should not interpolate or fill weekends. Simply skip them -- there is no trading day to represent.

2. **Market holidays**: Expected gaps. The application should recognize these as legitimate non-trading days. A configurable holiday calendar per market would be ideal but is not strictly necessary for daily data analysis -- the data simply will not have rows for those dates.

3. **Unexpected gaps** (data errors, exchange outages): These are genuine missing data and must be handled.

4. **Trading halts**: Rare for daily data (usually resolved within the same day) but can occur. The daily bar may show reduced volume or unusual OHLC relationships.

**Gap handling strategies:**

| Strategy | When to Use | Implementation |
|----------|-----------|----------------|
| **Forward fill (ffill)** | Isolated missing days (1-2 days) | `df.ffill()` -- carries last known close forward. Simple, preserves trend. Volume should be set to 0 or NaN for filled days. |
| **Linear interpolation** | Small gaps (1-3 days) with smooth trends | `df.interpolate(method='linear')` -- creates artificial prices between known points. Reasonable for Close, problematic for OHLC. |
| **Drop rows** | If gap is large (> 5 days) or data is questionable | Simply exclude the gap period. Indicator calculations reset after the gap. |
| **Flag and warn** | All cases | Always flag gaps in data quality report. Let the user decide. |

**Recommended approach:**

1. On data load, identify all gaps by comparing consecutive dates against the expected trading calendar.
2. For gaps of 1-2 trading days: forward-fill OHLC, set Volume to 0, flag the dates in a data quality log.
3. For gaps of 3+ trading days: do not fill. Split the analysis into segments before and after the gap. Indicator calculations that span the gap should be marked as unreliable.
4. Never fill weekends or known holidays -- these are not missing data.

### 1.5 Data Quality Issues and Cleaning

**Common data quality problems:**

| Issue | Detection Method | Resolution |
|-------|-----------------|-----------|
| **Stale/repeated prices** | Check for consecutive identical OHLC rows | Flag for review; may indicate illiquid market or data feed issue. Do not automatically remove -- some thinly traded assets legitimately have unchanged prices. |
| **Price spikes/outliers** | Z-score > 3-4 standard deviations from rolling mean | Verify against news/events. If confirmed real (flash crash, earnings), keep. If data error, replace with interpolated value or NaN. |
| **Negative or zero prices** | Simple range check | Remove or flag. Exception: some commodities briefly traded negative (WTI crude in April 2020) but this was a futures phenomenon, not spot. |
| **OHLC relationship violations** | `High < Low` or `Close > High` | Almost always a data error. Drop or flag the row. |
| **Volume anomalies** | Zero volume on known trading days; sudden 100x volume spike | Zero volume may indicate partial day or data issue. Extreme spikes should be verified against news. |
| **Duplicate dates** | Check for duplicate index entries | Keep the row with higher volume (more likely to be correct) or the later-arriving data. |
| **Timezone issues** | Close price does not match known exchange close | Ensure all data uses consistent timezone. Convert to exchange local time if mixing sources. |
| **Unadjusted splits** | Sudden 50% or 100% price drop with matching volume increase | Apply split adjustment retroactively or use pre-adjusted data source. |

**Data cleaning pipeline (recommended order):**

1. Parse dates and set as DatetimeIndex
2. Sort by date ascending
3. Remove duplicate dates
4. Validate OHLC relationships (High >= Low, etc.)
5. Check for and handle missing dates (gaps)
6. Detect outliers using rolling z-score (window=20, threshold=4)
7. Verify split/dividend adjustments if applicable
8. Log all data quality issues with dates and descriptions
9. Report summary statistics (date range, row count, gap count, issue count)

### 1.6 Common Data Sources for Historical Spot Prices

| Source | Asset Classes | Cost | Data Quality | Volume Data | Notes |
|--------|-------------|------|-------------|------------|-------|
| **Yahoo Finance (yfinance)** | Equities, ETFs, Indices, Forex, Crypto | Free | Good for equities; forex/crypto less reliable | Yes (equities) | Unofficial API, may break. Best for equities. Personal use only. |
| **Alpha Vantage** | Equities, Forex, Crypto, Commodities | Free tier (25 req/day), paid plans | Good | Yes | Official API with keys. Free tier very limited. |
| **FRED (Federal Reserve)** | Commodities (gold, oil, gas), economic indicators | Free | Excellent (government source) | No | Best for commodity spot prices. No OHLC -- only daily close. |
| **Quandl/Nasdaq Data Link** | Commodities, Futures, Equities | Free and paid datasets | Varies by dataset | Varies | Wide range. Some datasets discontinued. |
| **CoinGecko / CoinMarketCap** | Cryptocurrencies | Free tier available | Good | Yes | Best free sources for crypto OHLCV. |
| **OANDA** | Forex | Free with account | Excellent | Tick volume (not real volume) | Professional forex data. Tick volume is a proxy only. |
| **IEX Cloud** | Equities (US) | Paid (free tier exists) | Excellent | Yes | High-quality US equity data. |
| **Polygon.io** | Equities, Options, Forex, Crypto | Paid (free tier) | Excellent | Yes | Professional-grade. Comprehensive coverage. |
| **CSV files (user-provided)** | Any | Free | User-dependent | User-dependent | Most flexible input method. Application should support this as primary input. |

**Recommendation for the application:**

- Primary input: CSV file (maximum flexibility, no API dependency)
- Secondary input: configurable API connector (yfinance as default for quick prototyping)
- The application should not be tightly coupled to any single data source

---

## 2. Asset Class Considerations for Technical Analysis

### 2.1 Commodities

**Sub-categories and characteristics:**

| Commodity Type | Examples | Key Characteristics |
|---------------|---------|-------------------|
| **Precious metals** | Gold (XAU), Silver (XAG), Platinum | Safe haven behavior, inverse correlation to USD, low-moderate volatility, strong trend-following characteristics |
| **Energy** | Crude Oil (WTI, Brent), Natural Gas | High volatility, geopolitical sensitivity, seasonality (heating/cooling seasons), supply-driven shocks |
| **Agricultural** | Corn, Wheat, Soybeans, Coffee, Sugar | Strong seasonality (planting/harvest cycles), weather sensitivity, mean-reverting over long periods |
| **Industrial metals** | Copper, Aluminum, Iron Ore | Cyclical with economic activity, China demand sensitivity |

**Indicators that work best for commodities:**

- **Trend-following indicators dominate**: Commodities exhibit strong, persistent trends more often than equities. Moving average crossovers (50/200 SMA) and MACD perform well, particularly for precious metals and energy.
- **ADX is particularly valuable**: Commodity markets alternate between strong trends and consolidation more cleanly than equities, making ADX-based regime detection highly effective.
- **Donchian Channels / Turtle Trading**: Originally designed for commodities. The Turtle Trading system (Donchian breakouts) produced exceptional returns on commodity futures.
- **Bollinger Bands**: Effective for commodities due to volatility clustering. Bollinger Squeezes reliably precede major commodity moves.
- **Seasonal indicators**: Commodities benefit from seasonal analysis overlays that are not relevant for most other asset classes.
- **Volume indicators**: Less useful for commodity spot prices because true volume data is often unavailable (FRED provides close-only data). If using futures proxies, volume and open interest are available and highly informative.

**Commodities-specific data issues:**
- "Spot" price for commodities often means front-month futures price
- Contango/backwardation can cause spot and futures prices to diverge
- Storage costs create a natural cost-of-carry premium in futures vs spot
- Some commodities trade in specific units (troy ounces for gold, barrels for oil) that affect position sizing

**Typical daily volatility (annualized):**
- Gold: 12-18%
- Silver: 20-30%
- Crude Oil: 25-40%
- Natural Gas: 35-60%
- Agricultural: 15-30%

### 2.2 Forex Pairs

**Characteristics:**

- Trades nearly 24 hours (Sunday 5pm ET to Friday 5pm ET), creating continuous data with no gaps except weekends
- Extremely liquid (USD pairs: $6+ trillion daily volume)
- Prices are ratios (EUR/USD = euros per dollar), so price is always relative
- No true "volume" data from centralized exchange -- only tick volume from individual brokers
- Tight spreads on major pairs (EUR/USD: 0.1-1 pip), wider on exotics
- Macro-driven (interest rates, central bank policy, economic data)

**Indicators that work best for forex:**

- **Moving average crossovers**: The most consistently profitable approach in forex according to academic studies. EMA crossovers (12/26, 50/200) are standard.
- **RSI**: Works well for identifying reversals in range-bound pairs (EUR/CHF historically) but fails in trending pairs during central bank intervention periods.
- **MACD**: Strong performer on major pairs. MACD divergence is particularly useful for identifying trend exhaustion.
- **Ichimoku Cloud**: Originally designed for Japanese markets but widely used in forex. The cloud provides excellent support/resistance levels for major pairs.
- **ATR**: Critical for forex position sizing and stop-loss placement. Forex ATR is typically expressed in pips.
- **Stochastic Oscillator**: Effective for ranging pairs and for timing entries in the direction of the larger trend.
- **Parabolic SAR**: Works well in trending forex markets (major trends can persist for months in forex).

**Volume-dependent indicators are problematic for forex:**
- OBV, A/D Line, CMF, MFI, and VWAP all require volume data
- Forex tick volume (number of price changes, not units traded) is a weak proxy
- The application should warn or disable volume-based indicators when analyzing forex data
- If tick volume is provided, it can be used as an approximation but results should be flagged as lower confidence

**Typical daily volatility (annualized):**
- Major pairs (EUR/USD, GBP/USD): 6-12%
- Cross pairs (EUR/GBP, AUD/NZD): 5-10%
- Exotic pairs (USD/TRY, USD/ZAR): 12-25%

### 2.3 Cryptocurrencies

**Characteristics:**

- Trades 24/7/365 -- no weekends, no holidays
- Extremely high volatility (3-10x equities)
- Relatively short history (Bitcoin since 2009, most altcoins since 2017+)
- Highly correlated within the asset class (Bitcoin leads, altcoins follow with amplification)
- Prone to structural breaks (regulatory actions, exchange failures, halvings)
- Market structure is fragmented across exchanges (different prices on different exchanges)
- "Daily close" is ambiguous -- typically midnight UTC but varies by data source

**Indicators that work best for crypto:**

- **Moving averages**: Work well but require shorter periods than traditional markets due to faster cycles. 20/50 EMA crossovers outperform 50/200 SMA in crypto. The 200-day SMA remains an important long-term support/resistance level for Bitcoin.
- **RSI**: Very effective but overbought/oversold thresholds need adjustment. In crypto bull markets, RSI can remain above 70 for weeks. Suggested thresholds: overbought at 80 (not 70), oversold at 25 (not 30).
- **Bollinger Bands**: Excellent for crypto due to volatility clustering. The squeeze-to-breakout pattern is highly reliable in crypto.
- **MACD**: Standard parameters (12/26/9) work but weekly MACD on daily charts provides better trend identification.
- **Volume indicators**: Volume data is available and useful for crypto (unlike forex). OBV divergence is a strong signal. However, wash trading on some exchanges inflates volume artificially.
- **ATR**: Essential for crypto due to extreme volatility. Stop-losses should be wider (2-3x ATR minimum) compared to equities (1.5-2x ATR).
- **Fibonacci retracements**: Crypto markets show strong Fibonacci level adherence, likely due to the prevalence of algorithmic trading.

**Crypto-specific data issues:**
- Need to define "close time" consistently (midnight UTC is standard)
- Exchange fragmentation means different sources report different prices
- Use aggregated data (CoinGecko, CoinMarketCap) rather than single-exchange data
- Wash trading inflates volume on some exchanges -- use "reported volume" cautiously
- Pre-2017 data is sparse for most assets other than Bitcoin and Ethereum
- Network events (hard forks, halvings) can create structural breaks

**Typical daily volatility (annualized):**
- Bitcoin: 50-80%
- Ethereum: 70-100%
- Large-cap altcoins: 80-150%
- Small-cap altcoins: 100-300%

### 2.4 Equities and Indices

**Characteristics:**

- Most structured and regulated data (exchanges report official OHLCV)
- Trading hours are fixed and well-defined (NYSE: 9:30 AM - 4:00 PM ET)
- Corporate actions (dividends, splits, mergers) require data adjustments
- Earnings reports create known volatility events
- True volume data is available and highly informative
- Long history available (S&P 500 data back to 1928)

**Indicators that work best for equities:**

- **RSI**: Equities exhibit mean-reversion tendencies more than commodities or forex. RSI oversold/overbought signals are highly effective, particularly for large-cap stocks and indices.
- **MACD**: Reliable for equities, especially on indices. MACD+RSI combination shows 73% win rate on equities.
- **Bollinger Bands**: Excellent for individual stocks, especially during earnings-driven volatility expansion. Mean-reversion to the middle band is a strong signal for range-bound large-caps.
- **Moving averages**: 50/200 SMA Golden Cross and Death Cross are the most widely watched equity signals. Self-fulfilling prophecy effect is strongest in equities because the most participants watch these levels.
- **Volume indicators (all of them)**: Equities have the best volume data. OBV, A/D Line, CMF, and MFI are all highly effective. Volume confirmation is most valuable for equities.
- **OBV divergence**: Institutional accumulation/distribution is most detectable in equities via OBV because institutional order flow is large relative to normal volume.
- **Ichimoku Cloud**: Works well for indices and large-cap stocks. Less effective for small-caps due to gapping behavior.

**Equities-specific data issues:**
- **Adjusted vs unadjusted prices**: This is the single most important data decision for equities. All indicator calculations should use adjusted close to account for dividends and splits. Failure to adjust produces incorrect signals around corporate action dates.
- **Survivorship bias**: Historical datasets that only include currently listed stocks overstate backtesting performance. Delisted companies (often failures) are excluded, making past results look better than they were.
- **Earnings gaps**: Stocks can gap 5-20% on earnings reports. These gaps distort indicator readings. Consider flagging earnings dates and treating post-earnings gaps as regime changes.

**Typical daily volatility (annualized):**
- Large-cap indices (S&P 500): 12-20%
- Large-cap individual stocks: 20-35%
- Small-cap stocks: 30-50%
- Sector ETFs: 15-30%

### 2.5 Indicator Effectiveness Comparison by Asset Class

| Indicator | Commodities | Forex | Crypto | Equities |
|-----------|:-----------:|:-----:|:------:|:--------:|
| SMA/EMA Crossovers | Strong | Strong | Moderate | Strong |
| RSI | Moderate | Moderate | Strong (adjusted thresholds) | Strong |
| MACD | Strong | Strong | Strong | Strong |
| Stochastic | Moderate | Strong | Moderate | Moderate |
| Bollinger Bands | Strong | Moderate | Strong | Strong |
| ATR | Essential | Essential | Essential | Important |
| ADX/DMI | Very Strong | Strong | Moderate | Strong |
| OBV | Limited (no volume) | Not applicable (no volume) | Moderate (wash trading risk) | Very Strong |
| MFI | Limited | Not applicable | Moderate | Strong |
| VWAP | Not applicable | Not applicable | Moderate | Strong |
| Ichimoku | Moderate | Strong | Moderate | Strong |
| Donchian Channels | Very Strong | Moderate | Moderate | Moderate |
| Parabolic SAR | Strong | Strong | Moderate (too volatile) | Moderate |

**Key takeaway**: The application should allow the user to specify asset class and use this to inform default indicator selection and parameter tuning. A "commodity mode" would emphasize trend-following and disable volume indicators. A "forex mode" would disable volume indicators and adjust for tick volume. A "crypto mode" would adjust RSI thresholds and widen stop-loss multipliers. An "equity mode" would enable all indicators and require adjusted close data.

### 2.6 Volatility Characteristics Summary

| Asset Class | Annualized Volatility Range | Volatility Behavior | Implications for TA |
|-------------|:-------------------------:|---------------------|-------------------|
| Equities (indices) | 12-20% | Volatility clustering, skew (crashes faster than rallies), mean-reverting | Standard indicator parameters work well. VIX can supplement analysis. |
| Equities (individual) | 20-50% | Earnings-driven jumps, sector rotation | Wider stop-losses around earnings. Sector-relative analysis valuable. |
| Forex (majors) | 6-12% | Lowest volatility asset class, regime-dependent (calm vs intervention) | Tighter stops, more sensitive indicators. Leverage is common (not modeled in TA). |
| Commodities | 15-60% | Supply shock driven, seasonal, geopolitical | ATR-based stops essential. Regime detection critical. |
| Crypto | 50-300% | Highest volatility, extreme tail events, 24/7 | Wide stops, adjusted thresholds, smaller positions. Standard parameters are too tight. |

---

## 3. Data Storage and Processing

### 3.1 File Formats

| Format | Pros | Cons | Best For |
|--------|------|------|----------|
| **CSV** | Universal, human-readable, easy to create/edit, supported everywhere | Slow for large files, no type enforcement, no compression, larger file sizes | Primary input format -- users can create/export CSV from any spreadsheet or data source. Daily data size is small enough that CSV performance is not a concern. |
| **JSON** | Self-describing (includes field names), supports nested data, widely used in APIs | Verbose (larger files), slower to parse for tabular data, not ideal for time series | API responses, configuration files, metadata. Not recommended for primary price data storage. |
| **Parquet** | Columnar storage (fast queries), compressed (5-10x smaller than CSV), preserves data types, fast read with pandas | Binary format (not human-readable), requires pyarrow or fastparquet library, harder to create manually | Internal storage after initial load. Ideal for caching processed data, storing computed indicator values. Best if historical data exceeds 10,000+ rows. |
| **HDF5** | Fast read/write, supports multiple datasets in one file, compression | Portability issues, can become corrupted, less popular than Parquet | Legacy choice. Parquet has largely replaced HDF5 for new Python data projects. |
| **SQLite** | SQL query support, single-file database, no server needed | Overhead for simple time series, slower than Parquet for analytics | Useful if the application needs trade logging, metadata storage, or relational queries. Not needed for input data. |
| **Feather** | Extremely fast read/write, good for pandas interop | No compression, less portable than Parquet | Temporary intermediate files, caching between pipeline stages. |

**Recommendation:**
- Input: CSV (user-facing, maximum compatibility)
- Internal cache: Parquet (performance for computed indicators and backtest results)
- Trade logs and results: CSV for export, Parquet for internal storage
- Configuration: YAML or JSON

**Data size estimates for daily data:**

| Dataset | Rows | CSV Size | Parquet Size |
|---------|------|----------|-------------|
| 1 year, 1 asset | ~252 | ~15 KB | ~5 KB |
| 5 years, 1 asset | ~1,260 | ~75 KB | ~20 KB |
| 20 years, 1 asset | ~5,040 | ~300 KB | ~80 KB |
| 50 years, 1 asset | ~12,600 | ~750 KB | ~200 KB |
| 20 years, 100 assets | ~504,000 | ~30 MB | ~8 MB |

Daily data is extremely compact. Even 50 years of data for a single asset is under 1 MB in CSV. Performance is not a concern for data I/O at this scale. The choice of CSV for input and Parquet for caching is driven by usability, not necessity.

### 3.2 Data Normalization Considerations

**Price normalization:**
- **Do not normalize prices before indicator calculation.** Moving averages, RSI, MACD, and virtually all standard indicators are designed to work on raw (or adjusted) prices. Normalizing to [0,1] or z-scoring prices would break these calculations.
- **Exception: cross-asset comparison.** If comparing indicator effectiveness across assets with different price scales (gold at $2000 vs a stock at $50), normalize the performance metrics (Sharpe ratio, returns), not the prices.
- **Percentage returns are naturally normalized.** For comparing performance, convert to percentage returns: `return = (close_t - close_{t-1}) / close_{t-1}`.

**When normalization IS needed:**
- Composite indicator scoring: When combining RSI (0-100), MACD (unbounded), and Bollinger %B (0-1) into a single composite score, normalize each to a common scale (z-score or min-max scaling).
- Cross-asset backtesting comparison: Normalize profitability metrics, not prices.
- Machine learning inputs: If using ML for regime detection, normalize features.

### 3.3 Handling Different Time Zones

**The problem:**
- NYSE closes at 4:00 PM ET (UTC-5 or UTC-4 during DST)
- LSE closes at 4:30 PM GMT (UTC+0)
- Tokyo closes at 3:00 PM JST (UTC+9)
- Forex "closes" at 5:00 PM ET (New York close, rolled to next day)
- Crypto has no close -- typically midnight UTC is used

**Recommended approach:**
1. Store all dates as date-only (no time component) in the data. A daily bar is identified by its date, not a timestamp.
2. Use the exchange's local business date as the date value. A bar dated "2024-01-15" means the trading session of January 15 in the exchange's local time.
3. If mixing data from multiple exchanges (e.g., comparing gold to equities), align on calendar date. The "January 15" bar for gold and the "January 15" bar for S&P 500 may represent slightly different calendar periods (different closing times) but this is standard practice and the discrepancy is negligible for daily data analysis.
4. For crypto, use UTC midnight as the daily close time.
5. Do not attempt to time-align intraday -- daily data inherently abstracts this away.

**Implementation:**
```python
# Use pandas DatetimeIndex with date only (no time)
df.index = pd.to_datetime(df['date']).dt.normalize()
# Or simply:
df.index = pd.to_datetime(df['date'])
```

### 3.4 Corporate Actions (Equities)

**Stock splits:**
- A 2:1 split doubles the number of shares and halves the price. If not adjusted, this appears as a 50% price drop in the data and creates a false sell signal on every indicator.
- All historical prices before the split must be divided by the split ratio.
- All historical volumes before the split must be multiplied by the split ratio.
- Most data sources (Yahoo Finance, IEX) provide pre-adjusted data. Always verify.

**Dividends:**
- On the ex-dividend date, the stock price drops by approximately the dividend amount.
- If using unadjusted prices, dividends create artificial downward gaps that distort indicators.
- Use "adjusted close" which accounts for dividends by retroactively reducing all prior prices.
- Critical: if using adjusted close, do NOT also add dividends to return calculations. This is double-counting and is a common error.

**Mergers and acquisitions:**
- When a company is acquired, its price series ends. The application should handle end-of-series gracefully.
- Spin-offs can create new price series. The parent company's historical prices may be adjusted.

**Recommendation for the application:**
- Require adjusted close data for equities. Document this requirement clearly.
- Provide a validation check that detects likely unadjusted data (sudden 50% drops with no corresponding market event).
- If the user provides unadjusted data, warn them and offer to continue with the caveat that results near corporate action dates will be unreliable.

### 3.5 Data Volume and History Requirements

**Minimum data requirements by indicator:**

| Indicator | Minimum Bars Needed | Recommended Minimum | Explanation |
|-----------|:------------------:|:-------------------:|------------|
| SMA(200) | 200 | 400+ | Need 200 bars for the first valid value, plus buffer for the indicator to stabilize |
| EMA(200) | 200 | 300+ | EMA converges faster than SMA but still needs warmup |
| RSI(14) | 15 | 100+ | First 14 bars for initial calculation, more for stability |
| MACD(12,26,9) | 35 | 100+ | 26 for slow EMA + 9 for signal line |
| Bollinger(20) | 20 | 100+ | 20 for SMA and standard deviation |
| ATR(14) | 15 | 50+ | 14-bar lookback plus one bar for True Range |
| Stochastic(14,3,3) | 17 | 50+ | 14 lookback + 3 for %K + 3 for %D |
| Ichimoku(9,26,52) | 78 | 200+ | 52-bar lookback + 26-bar forward projection |
| ADX(14) | 28 | 100+ | Double the period for DMI smoothing |
| OBV | 1 | 50+ | Cumulative -- starts from day 1 |

**Minimum data for reliable backtesting:**

| Purpose | Minimum Years | Recommended Years | Reasoning |
|---------|:------------:|:-----------------:|-----------|
| Indicator calculation | 1 year | 2 years | Allow all indicators (including 200-day SMA) to initialize with buffer |
| Basic backtesting | 2 years | 5 years | Need enough trades for statistical significance (minimum 30 trades) |
| Walk-forward analysis | 3 years | 5-10 years | Need multiple IS/OOS windows to evaluate robustness |
| Regime analysis | 5 years | 10+ years | Need to capture multiple bull/bear/sideways market regimes |
| Publication-grade research | 10 years | 20+ years | Cover multiple market cycles for robust conclusions |

**Recommendation:** The application should require a minimum of 2 years (approximately 500 trading days) of data and recommend 5+ years. It should warn the user if the data is insufficient for the selected indicators or backtesting methodology.

### 3.6 Indicator Warmup and Data Padding

When computing indicators, the first N bars (where N is the lookback period) produce NaN or unreliable values. This "warmup period" must be handled:

- **Option 1 (Recommended):** Load extra historical data beyond the analysis window for warmup. If the user wants to analyze 2023, load from mid-2022 to allow the 200-day SMA to initialize. Then trim the warmup period from results.
- **Option 2:** Accept NaN values at the start of the analysis. All indicators, signals, and backtesting start only after all indicators have valid values.
- **Option 3:** Use shorter indicator periods if data is limited. For example, use SMA(50) instead of SMA(200) if only 1 year of data is available.

The application should calculate the required warmup period based on the selected indicators and either request additional data or adjust the effective analysis start date.

---

## 4. Python Libraries for Technical Analysis

### 4.1 TA-Lib (Technical Analysis Library)

**Overview:** TA-Lib is the gold standard for technical indicator computation. Originally written in C, with Python bindings via the `ta-lib` Python package (wrapping `TA-Lib` C library).

**Capabilities:**
- 150+ indicator functions covering all standard categories
- 60+ candlestick pattern recognition functions (largest such collection)
- Overlap studies (MA types), momentum indicators, volume indicators, volatility indicators, price transforms, cycle indicators, pattern recognition
- Operates on NumPy arrays for high performance

**Pros:**
- Speed: C implementation is 2-5x faster than pure Python alternatives
- Reliability: Battle-tested for 20+ years, used in production trading systems worldwide
- Completeness: Most comprehensive indicator and candlestick pattern library available
- Accuracy: Reference implementation -- other libraries validate against TA-Lib
- NumPy integration: Works directly with NumPy arrays and pandas Series

**Cons:**
- Installation: Requires compiling the C library first (`ta-lib` C library must be installed before the Python package). This is the single biggest friction point. On Linux: `apt-get install ta-lib` or compile from source. On macOS: `brew install ta-lib`. On Windows: download pre-compiled binaries.
- API design: Function-based API (not object-oriented). Each indicator is a separate function call.
- Documentation: Official docs are sparse. Community resources fill the gap.
- No DataFrame integration: Returns NumPy arrays, not pandas DataFrames (easy to wrap but requires boilerplate).

**Installation:**
```bash
# Linux (Ubuntu/Debian)
sudo apt-get install -y ta-lib  # or build from source
pip install ta-lib

# macOS
brew install ta-lib
pip install ta-lib

# Windows -- download binary from pre-compiled wheel sites
pip install TA_Lib-0.4.xx-cpXX-cpXX-win_amd64.whl

# Alternative: conda (easiest cross-platform)
conda install -c conda-forge ta-lib
```

**Usage example:**
```python
import talib
import numpy as np

close = np.array(df['close'], dtype=float)
high = np.array(df['high'], dtype=float)
low = np.array(df['low'], dtype=float)
volume = np.array(df['volume'], dtype=float)

# Indicators
sma_50 = talib.SMA(close, timeperiod=50)
rsi_14 = talib.RSI(close, timeperiod=14)
macd, signal, hist = talib.MACD(close, fastperiod=12, slowperiod=26, signalperiod=9)
upper, middle, lower = talib.BBANDS(close, timeperiod=20, nbdevup=2, nbdevdn=2)
atr_14 = talib.ATR(high, low, close, timeperiod=14)

# Candlestick patterns (returns 0, 100, or -100)
hammer = talib.CDLHAMMER(open_arr, high, low, close)
engulfing = talib.CDLENGULFING(open_arr, high, low, close)
```

**Recommendation:** Use as the primary indicator computation engine. Wrap in a helper class that accepts pandas DataFrames and returns DataFrames.

### 4.2 pandas-ta

**Overview:** Pure Python technical analysis library built as a pandas extension. Adds a `.ta` accessor to DataFrames.

**Capabilities:**
- 150+ indicators (comparable to TA-Lib in coverage)
- Pandas-native: works directly as DataFrame extension
- Includes strategy system for running multiple indicators at once
- Custom indicator support via templates

**Pros:**
- Installation: Pure Python -- `pip install pandas-ta`. No C compilation required.
- API design: Clean, Pythonic, pandas-native. `df.ta.rsi(length=14)` is more readable than `talib.RSI(close, timeperiod=14)`.
- Strategy system: Can define groups of indicators and compute all at once.
- Active development (as of early 2025).
- Good documentation and examples.

**Cons:**
- Performance: 2-5x slower than TA-Lib for large datasets (pure Python vs C). For daily data with reasonable history (< 50,000 rows), this difference is imperceptible (milliseconds vs tens of milliseconds).
- Sustainability concern: The maintainer has indicated the project may be archived by July 2026 if community support does not materialize. This is a real risk for long-term maintenance.
- Fewer candlestick patterns than TA-Lib.
- Less battle-tested in production environments.
- Some indicators have subtle numerical differences from TA-Lib (edge cases in smoothing).

**Usage example:**
```python
import pandas_ta as ta

# Single indicator
df['rsi'] = df.ta.rsi(length=14)
df['sma_50'] = df.ta.sma(length=50)

# Multiple indicators via strategy
strategy = ta.Strategy(
    name="Multi-Indicator",
    ta=[
        {"kind": "sma", "length": 50},
        {"kind": "sma", "length": 200},
        {"kind": "rsi", "length": 14},
        {"kind": "macd", "fast": 12, "slow": 26, "signal": 9},
        {"kind": "bbands", "length": 20, "std": 2},
        {"kind": "atr", "length": 14},
    ]
)
df.ta.strategy(strategy)
```

**Recommendation:** Use as a fallback if TA-Lib installation is problematic, or for rapid prototyping. Do not depend on it as the sole indicator library given sustainability concerns.

### 4.3 ta (Technical Analysis Library for Python)

**Overview:** Another pure Python library by Dario Lopez Padial. Simpler and more focused than pandas-ta.

**Capabilities:**
- ~40 indicators (fewer than TA-Lib or pandas-ta)
- Organized into 6 categories: volume, volatility, trend, momentum, others, all
- Wrapper classes that accept pandas DataFrames directly

**Pros:**
- Simple installation: `pip install ta`
- Clean API with wrapper functions: `ta.add_all_ta_features(df, open, high, low, close, volume)`
- Good for quick exploration -- single function call adds all indicators to a DataFrame
- Actively maintained

**Cons:**
- Limited indicator coverage (~40 vs 150+ for TA-Lib/pandas-ta)
- No candlestick pattern recognition
- Slower than TA-Lib
- Less configurable -- fewer parameter options per indicator
- Name collision: `ta` is a very common import name, can conflict with other packages

**Usage example:**
```python
import ta

# Add all indicators at once
df = ta.add_all_ta_features(
    df, open="open", high="high", low="low", close="close", volume="volume"
)

# Or individual indicators
df['rsi'] = ta.momentum.RSIIndicator(close=df['close'], window=14).rsi()
df['macd'] = ta.trend.MACD(close=df['close']).macd()
```

**Recommendation:** Useful for quick prototyping and exploration but insufficient for a production application. The limited indicator set and lack of candlestick patterns make it unsuitable as the primary library.

### 4.4 Backtesting Frameworks

#### Backtrader

**Overview:** Event-driven backtesting framework with broker simulation, strategy development, and live trading support.

**Capabilities:**
- Strategy development via Python classes
- Built-in broker simulation (commissions, slippage, margin)
- Data feed support (CSV, pandas, live feeds)
- Cerebro engine for orchestrating backtests
- Plotting via matplotlib integration
- Analyzer system for performance metrics (Sharpe, drawdown, trade log)

**Pros:**
- Well-designed OOP API for strategy development
- Extensive documentation and community resources
- Supports multiple data feeds and timeframes
- Built-in broker simulation handles realistic execution

**Cons:**
- **Unmaintained since 2018.** This is the critical issue. No bug fixes, no Python version compatibility updates, no new features. Dependencies may break.
- Slow for parameter optimization (event-driven architecture)
- Complex for simple indicator testing (overengineered for our use case)
- Memory hungry for large backtests

**Recommendation:** Do not use. The abandoned maintenance status makes it unsuitable for a new project. Use vectorbt instead.

#### Zipline (zipline-reloaded)

**Overview:** Event-driven backtesting framework originally developed by Quantopian. Continued as `zipline-reloaded` by the community.

**Capabilities:**
- Full event-driven simulation
- Pipeline API for factor analysis
- Handles corporate actions, dividends, splits natively
- Benchmarking against indices
- Integration with Alphalens for factor analysis and Pyfolio for performance analysis

**Pros:**
- Most realistic simulation of actual portfolio management
- Handles equity-specific concerns (splits, dividends) natively
- Pipeline API is powerful for factor-based strategies
- Quantopian heritage means many strategy examples exist

**Cons:**
- Installation is notoriously difficult (heavy dependencies: bcolz, bottleneck, empyrical)
- Primarily designed for US equities -- adding other asset classes requires custom data bundles
- Event-driven architecture is slow for parameter sweeps
- Small active community (Quantopian shut down in 2020)
- Overkill for daily indicator testing

**Recommendation:** Not recommended for this project. The installation complexity and US-equity focus do not align with a multi-asset-class spot price analysis tool. Consider only if the application evolves into a full portfolio management system.

#### vectorbt (VectorBT)

**Overview:** High-performance vectorized backtesting framework built on NumPy and Numba. Designed for speed-first analysis.

**Capabilities:**
- Vectorized backtesting (operates on entire arrays, not bar-by-bar)
- Parameter optimization across thousands of combinations in seconds
- Built-in indicator library (can also use TA-Lib outputs)
- Portfolio simulation with position sizing
- Comprehensive performance metrics (Sharpe, Sortino, Calmar, drawdowns, trade log)
- Plotting via Plotly integration

**Pros:**
- **Speed**: 10-100x faster than event-driven frameworks for parameter optimization. Can test 10,000+ parameter combinations in seconds on daily data.
- Flexible signal generation: Works with any NumPy boolean array as entry/exit signals
- Excellent performance analytics: Built-in computation of all major metrics
- Active development and growing community
- Good documentation
- Plotly-based interactive charts
- Supports custom indicators via NumPy/Numba

**Cons:**
- Steep learning curve: The vectorized paradigm is different from traditional strategy development
- PRO version is paid (but free version is sufficient for this project)
- Less realistic execution modeling than event-driven frameworks (no look-ahead protection is enforced -- the user must ensure signals do not use future data)
- API can be verbose for complex strategies

**Usage example:**
```python
import vectorbt as vbt
import pandas as pd

# Load data
price = pd.Series(df['close'], index=df.index)

# Simple RSI-based strategy
rsi = vbt.RSI.run(price, window=14)
entries = rsi.rsi_crossed_below(30)  # buy when RSI < 30
exits = rsi.rsi_crossed_above(70)    # sell when RSI > 70

# Backtest
portfolio = vbt.Portfolio.from_signals(
    price,
    entries=entries,
    exits=exits,
    init_cash=10000,
    fees=0.001  # 0.1% per trade
)

# Results
print(portfolio.stats())
print(portfolio.trades.records_readable)

# Parameter optimization
rsi_multi = vbt.RSI.run(price, window=range(5, 30))
# ... test all 25 RSI periods simultaneously
```

**Recommendation:** Use as the primary backtesting framework. Its speed advantage is critical for testing 27+ indicators across multiple parameter combinations. The vectorized approach is well-suited for the "test all indicators and rank them" use case.

### 4.5 Custom Implementations vs Library Use

**When to use libraries:**
- Standard indicators (SMA, EMA, RSI, MACD, Bollinger Bands, etc.): Always use libraries. These are well-tested and performance-optimized. Reimplementing them introduces bug risk for no benefit.
- Candlestick patterns: Use TA-Lib (60+ patterns). Reimplementing these is error-prone and time-consuming.
- Standard backtesting metrics: Use vectorbt or pyfolio. These metrics are subtle (e.g., annualized Sharpe from daily data requires sqrt(252) scaling, Sortino requires downside deviation only).

**When custom implementation is appropriate:**
- Composite indicator scoring: The weighted scoring system for ranking indicators is application-specific and must be custom.
- Signal generation logic: The hierarchical filtering system (trend filter -> confirmation -> entry) is custom logic that combines library-computed indicators.
- Regime detection: ADX-based regime detection is simple custom logic using library-computed ADX values.
- Report generation: Application-specific output formatting.
- Data loading and validation: Application-specific pipeline.
- Walk-forward analysis orchestration: While individual backtests use vectorbt, the walk-forward window management is custom.

### 4.6 Performance Considerations

For daily data with typical volumes (< 50,000 rows per asset), performance is not a bottleneck in any meaningful sense. However, performance matters when:

1. **Parameter optimization**: Testing 10,000+ parameter combinations. vectorbt handles this in seconds via vectorization. Event-driven frameworks take minutes to hours.
2. **Multi-indicator evaluation**: Testing 27+ indicators, each with multiple parameter sets. With vectorbt, this is feasible in under a minute for 10 years of daily data.
3. **Walk-forward analysis**: Repeated backtests across sliding windows. vectorbt's speed makes WFA practical.

**Performance comparison (approximate, 5 years daily data, single indicator):**

| Operation | TA-Lib | pandas-ta | ta | Custom NumPy |
|-----------|:------:|:---------:|:--:|:----------:|
| Single indicator calculation | < 1 ms | 2-5 ms | 5-10 ms | 1-3 ms |
| All 27 indicators | < 10 ms | 50-100 ms | N/A (limited set) | 20-50 ms |

| Operation | vectorbt | backtrader | zipline |
|-----------|:--------:|:----------:|:------:|
| Single backtest | 5-20 ms | 100-500 ms | 200-1000 ms |
| 1000 parameter combinations | 1-5 sec | 100-500 sec | 200-1000 sec |

For this application's use case, TA-Lib + vectorbt provides performance that is orders of magnitude beyond what is needed, ensuring excellent user experience even with extensive parameter optimization.

### 4.7 Additional Supporting Libraries

| Library | Purpose | Notes |
|---------|---------|-------|
| **pyfolio** | Portfolio performance and risk analysis | Tearsheet-style analysis. Pairs well with zipline but works standalone. |
| **empyrical** | Return and risk statistics | Compute Sharpe, Sortino, max drawdown, alpha, beta from returns series. Lightweight. |
| **scipy** | Statistical functions, signal processing | `scipy.signal.argrelextrema()` for local extrema detection (chart patterns). Savitzky-Golay filter for noise reduction. |
| **hmmlearn** | Hidden Markov Models | Advanced market regime detection. `GaussianHMM` trained on daily returns identifies hidden market states. |
| **yfinance** | Yahoo Finance data download | Quick data sourcing for prototyping. `yf.download('AAPL', start='2019-01-01')` returns OHLCV DataFrame. |
| **fredapi** | FRED data download | Access Federal Reserve economic data including commodity prices. Requires free API key. |

---

## 5. Visualization and Reporting

### 5.1 Chart Types for Technical Analysis

| Chart Type | Description | When to Use |
|-----------|-----------|-----------|
| **Candlestick** | Shows OHLC as a body (open-close) with wicks (high-low). Green/white = close > open, Red/black = close < open. | Primary chart type for TA. Essential for candlestick pattern identification. Most information per bar. |
| **OHLC Bar** | Horizontal ticks for open (left) and close (right), vertical line for high-low range. | Alternative to candlestick. Some traders prefer for longer time series (less visual clutter). |
| **Line** | Connects closing prices only. | Simplest. Good for showing trend direction and overlay indicators. Used for long-term views. |
| **Area** | Line chart with filled area below. | Good for equity curves and cumulative returns. Visual emphasis on magnitude. |
| **Heikin-Ashi** | Modified candlesticks using averaged OHLC. Smooths trends. | Trend identification. Reduces noise. Note: not real prices, so not suitable for entry/exit display. |

**Recommendation:** Default to candlestick charts for price display. Use line charts for indicator overlays and equity curves.

### 5.2 Overlay Indicators vs Separate Panels

**Overlay indicators** (plotted on the same axes as price):
- Moving averages (SMA, EMA, WMA, DEMA, TEMA)
- Bollinger Bands
- Keltner Channels
- Donchian Channels
- Ichimoku Cloud
- Parabolic SAR
- VWAP

**Separate panel indicators** (plotted in their own sub-chart below price):
- RSI (0-100 scale)
- MACD (histogram + lines)
- Stochastic Oscillator (0-100)
- Williams %R (-100 to 0)
- CCI (unbounded)
- ADX/DMI (0-100)
- OBV (unbounded cumulative)
- CMF (-1 to +1)
- MFI (0-100)
- Volume (bar chart)
- ATR (separate scale)

**Layout recommendation:**
```
+------------------------------------------+
|  Price Chart (candlestick)               |
|  + overlay indicators (MAs, BBands...)   |
|  + buy/sell signal markers               |
+------------------------------------------+
|  Volume (bar chart)                      |
+------------------------------------------+
|  Indicator Panel 1 (e.g., RSI)           |
|  + overbought/oversold lines             |
+------------------------------------------+
|  Indicator Panel 2 (e.g., MACD)          |
|  + signal line + histogram               |
+------------------------------------------+
```

Standard practice is to display 1 price chart with overlays, 1 volume panel, and 1-3 indicator panels. More than 3 indicator panels becomes visually overwhelming.

### 5.3 Visualization Libraries

#### matplotlib + mplfinance

**mplfinance** is a matplotlib-based library specifically designed for financial charts.

**Capabilities:**
- Candlestick, OHLC, line, and renko charts
- Built-in support for volume panel
- Overlay and panel indicator support via `addplot`
- Customizable styles (classic, yahoo, nightclouds, etc.)
- Tight integration with pandas DataFrames

**Pros:**
- Purpose-built for financial charts
- High-quality static charts suitable for reports and PDFs
- Simple API: `mpf.plot(df, type='candle', volume=True, mav=(20, 50))`
- Matplotlib ecosystem for customization
- Good for automated report generation (save to PNG/SVG)

**Cons:**
- Static charts only (no interactivity)
- Limited zoom/pan (requires matplotlib widget backends)
- Customization beyond standard patterns can be complex
- No tooltips or hover information

**Usage example:**
```python
import mplfinance as mpf

# Basic candlestick with volume and moving averages
mpf.plot(df, type='candle', volume=True, mav=(20, 50, 200),
         title='Price with Moving Averages',
         style='yahoo',
         savefig='chart.png')

# With additional indicator panels
ap = [
    mpf.make_addplot(df['rsi'], panel=2, ylabel='RSI'),
    mpf.make_addplot(df['macd'], panel=3, ylabel='MACD'),
    mpf.make_addplot(df['signal'], panel=3),
]
mpf.plot(df, type='candle', volume=True, addplot=ap,
         panel_ratios=(4, 1, 2, 2))
```

**Recommendation:** Use for static chart generation in reports (PDF/HTML).

#### Plotly

**Capabilities:**
- Interactive candlestick, OHLC, line, and area charts
- Zoom, pan, hover tooltips, range selector, date range slider
- Subplots for multiple indicator panels
- Web-based rendering (HTML output)
- Dash framework for full dashboard applications

**Pros:**
- Rich interactivity: hover to see exact values, zoom into specific periods, select date ranges
- Excellent HTML output for web-based reports
- Professional appearance out of the box
- Supports all needed chart types
- Can be embedded in HTML reports or served via Dash

**Cons:**
- Heavier dependency than matplotlib
- Not suitable for PDF output (requires screenshot/conversion)
- Large file sizes for HTML with many data points
- Slower to render with very large datasets (> 10,000 points -- not a concern for daily data)

**Usage example:**
```python
import plotly.graph_objects as go
from plotly.subplots import make_subplots

fig = make_subplots(rows=4, cols=1, shared_xaxes=True,
                    row_heights=[0.5, 0.15, 0.15, 0.2],
                    vertical_spacing=0.02)

# Candlestick
fig.add_trace(go.Candlestick(
    x=df.index, open=df['open'], high=df['high'],
    low=df['low'], close=df['close'], name='Price'
), row=1, col=1)

# Volume
fig.add_trace(go.Bar(x=df.index, y=df['volume'], name='Volume'), row=2, col=1)

# RSI
fig.add_trace(go.Scatter(x=df.index, y=df['rsi'], name='RSI'), row=3, col=1)
fig.add_hline(y=70, line_dash="dash", line_color="red", row=3, col=1)
fig.add_hline(y=30, line_dash="dash", line_color="green", row=3, col=1)

# MACD
fig.add_trace(go.Scatter(x=df.index, y=df['macd'], name='MACD'), row=4, col=1)
fig.add_trace(go.Bar(x=df.index, y=df['macd_hist'], name='Histogram'), row=4, col=1)

fig.update_layout(height=800, xaxis_rangeslider_visible=False)
fig.write_html('interactive_chart.html')
```

**Recommendation:** Use for interactive exploration and HTML report output.

#### Comparison

| Feature | mplfinance | Plotly | Bokeh |
|---------|:----------:|:------:|:-----:|
| Static charts | Excellent | Good | Good |
| Interactive | No | Excellent | Good |
| PDF output | Native | Requires conversion | Requires conversion |
| HTML output | Via inline PNG | Native | Native |
| Candlestick support | Excellent (purpose-built) | Good | Basic |
| Learning curve | Low | Low-Medium | Medium |
| Performance (large data) | Excellent | Good | Good |
| Dependency size | Small (matplotlib) | Medium | Medium |

### 5.4 Interactive vs Static Charts

**Static charts (mplfinance) -- use for:**
- PDF report generation
- Automated report pipelines (no user interaction needed)
- Print-friendly output
- Embedding in documents
- Lightweight, fast generation

**Interactive charts (Plotly) -- use for:**
- Exploratory analysis
- HTML reports delivered via email or web
- Detailed examination of specific time periods
- Presentations and demos
- When users need to zoom into specific trades

**Recommendation:** Generate both. The application should produce a PDF/HTML report with static charts for the summary sections, and provide an interactive HTML chart file for detailed exploration. This dual approach covers both consumption patterns.

### 5.5 Report Generation Approaches

**Option 1: Jinja2 HTML Templates (Recommended)**

Generate HTML reports using Jinja2 templating with embedded charts (static PNGs or Plotly JSON).

```python
from jinja2 import Template

template = Template("""
<html>
<head><title>Technical Analysis Report</title></head>
<body>
    <h1>Analysis Report: {{ asset_name }}</h1>
    <h2>Summary Statistics</h2>
    <table>
        {% for metric in metrics %}
        <tr><td>{{ metric.name }}</td><td>{{ metric.value }}</td></tr>
        {% endfor %}
    </table>
    <h2>Best Indicators</h2>
    {{ ranking_table }}
    <h2>Price Chart with Signals</h2>
    {{ price_chart }}
    <h2>Trade Log</h2>
    {{ trade_table }}
</body>
</html>
""")
```

**Pros:** Full control over layout, professional appearance, can embed Plotly charts, easy to add CSS styling.

**Option 2: WeasyPrint or pdfkit for PDF**

Convert HTML reports to PDF using WeasyPrint (pure Python) or pdfkit (wkhtmltopdf wrapper).

```python
import weasyprint
html_content = generate_html_report(results)
weasyprint.HTML(string=html_content).write_pdf('report.pdf')
```

**Option 3: Console/Terminal Output**

For quick analysis, print summary statistics and top indicators to the terminal. Good for scripted/automated runs.

**Option 4: CSV Export**

Export trade logs and summary statistics as CSV for further analysis in Excel or other tools.

**Recommended report structure:**

```
Report Output:
  1. Summary page
     - Asset name, date range, data quality summary
     - Top 5 indicators by composite score
     - Buy-and-hold benchmark comparison
     - Key statistics (total return, Sharpe, max drawdown)

  2. Indicator details (one section per top indicator)
     - Parameters used
     - Performance metrics (full table)
     - Price chart with signals overlaid
     - Equity curve
     - Trade statistics

  3. Trade log
     - Full table of all trades (entry/exit date, price, P/L, duration)
     - Sortable by date, P/L, duration

  4. Comparison dashboard
     - Side-by-side metrics for all tested indicators
     - Ranking chart
     - Regime analysis (which indicator worked when)

  5. Data quality appendix
     - Gaps detected
     - Outliers flagged
     - Warnings and notes
```

### 5.6 Chart Annotations for Signals

Buy and sell signals should be visually marked on price charts:

- **Buy signals**: Green upward triangle markers below the price bar
- **Sell signals**: Red downward triangle markers above the price bar
- **Stop-loss levels**: Dashed red horizontal lines
- **Take-profit levels**: Dashed green horizontal lines
- **Regime changes**: Vertical shaded regions (green = trending, blue = ranging)
- **Pattern identifications**: Annotated shapes (rectangles for channels, lines for trendlines)

Both mplfinance and Plotly support these annotations. mplfinance uses `addplot` with marker styles; Plotly uses `add_trace` with scatter markers and `add_vrect`/`add_hline` for regions and levels.

---

## 6. Application Architecture Considerations

### 6.1 High-Level Architecture

The application follows a pipeline architecture:

```
Input Configuration
        |
        v
   Data Loading ---------> Data Validation ---------> Data Quality Report
        |
        v
   Indicator Calculation (all configured indicators)
        |
        v
   Signal Generation (buy/sell signals per indicator)
        |
        v
   Backtesting (per indicator, with walk-forward analysis)
        |
        v
   Performance Metrics Calculation
        |
        v
   Indicator Ranking (composite scoring)
        |
        v
   Report Generation (HTML/PDF/CSV)
```

### 6.2 Input: Configurable Data Source

The application should support multiple data input methods:

**File-based input (primary):**
```yaml
data:
  source: file
  path: /path/to/data.csv
  format: csv  # or parquet, json
  columns:
    date: Date
    open: Open
    high: High
    low: Low
    close: Close
    volume: Volume  # optional
  date_format: "%Y-%m-%d"
```

**API-based input (secondary):**
```yaml
data:
  source: api
  provider: yfinance  # or alpha_vantage, polygon
  symbol: AAPL
  start_date: "2019-01-01"
  end_date: "2024-12-31"
  api_key: ${API_KEY}  # from environment variable
```

**Data loader interface:**
```python
class DataLoader(ABC):
    @abstractmethod
    def load(self, config: dict) -> pd.DataFrame:
        """Load OHLCV data into a pandas DataFrame with DatetimeIndex."""
        pass

class CSVLoader(DataLoader):
    def load(self, config: dict) -> pd.DataFrame:
        # Read CSV, parse dates, validate columns, set index
        ...

class YFinanceLoader(DataLoader):
    def load(self, config: dict) -> pd.DataFrame:
        # Fetch from Yahoo Finance API
        ...
```

### 6.3 Processing Pipeline

Each stage of the pipeline should be modular and independently testable:

**Stage 1: Data Loading and Validation**
- Load raw data via configured data source
- Validate OHLCV relationships
- Detect and handle missing data
- Identify outliers
- Produce data quality report

**Stage 2: Indicator Calculation**
- Calculate all configured indicators using TA-Lib
- Handle warmup periods (NaN for initial bars)
- Store computed indicators as additional DataFrame columns

**Stage 3: Signal Generation**
- For each indicator, generate buy/sell signals based on configured rules
- Each signal generator produces a DataFrame with columns: `signal` (+1 buy, -1 sell, 0 hold), `signal_type` (description)
- Handle conflicting signals according to configured priority

**Stage 4: Backtesting**
- For each indicator's signals, run backtest using vectorbt
- Apply transaction costs
- Compute trade log (entry/exit/P&L)
- Calculate walk-forward analysis windows
- Compute all performance metrics

**Stage 5: Ranking and Analysis**
- Compute composite score for each indicator
- Rank indicators
- Compare to buy-and-hold benchmark
- Identify regime-dependent performance

**Stage 6: Report Generation**
- Generate HTML report with charts
- Export trade log as CSV
- Produce PDF summary (optional)

### 6.4 Output Structure

The application should produce the following outputs:

```
output/
  {asset}_{date}/
    report.html              # Full interactive report
    report.pdf               # PDF summary (optional)
    charts/
      price_overview.png     # Full period price chart
      best_indicator_1.png   # Top indicator with signals
      best_indicator_2.png   # Second best indicator
      equity_curves.png      # All indicator equity curves overlaid
      drawdown_comparison.png
      interactive_chart.html # Plotly interactive chart
    data/
      trade_log.csv          # All trades for all indicators
      indicator_rankings.csv # Ranking table with all metrics
      indicator_values.csv   # Computed indicator values
      summary_statistics.csv # Summary stats for each indicator
    config/
      analysis_config.yaml   # Configuration used for this run (reproducibility)
```

### 6.5 Configuration

A YAML-based configuration file provides maximum flexibility:

```yaml
# analysis_config.yaml

# Asset configuration
asset:
  name: "Gold Spot"
  class: commodity  # commodity, forex, crypto, equity

# Data source
data:
  source: file
  path: data/gold_daily.csv
  date_column: Date
  ohlcv_columns:
    open: Open
    high: High
    low: Low
    close: Close
    volume: Volume  # null if not available

# Analysis parameters
analysis:
  start_date: null  # null = use all available data
  end_date: null
  lookback_window: 30  # configurable pattern window (default 30 days)

# Indicators to test
indicators:
  trend:
    - name: sma_crossover
      params: {fast: [10, 20, 50], slow: [50, 100, 200]}
    - name: ema_crossover
      params: {fast: [10, 12, 20], slow: [26, 50, 200]}
    - name: macd
      params: {fast: 12, slow: 26, signal: 9}
    - name: ichimoku
      params: {tenkan: 9, kijun: 26, senkou: 52}
  momentum:
    - name: rsi
      params: {period: [7, 14, 21], overbought: 70, oversold: 30}
    - name: stochastic
      params: {k_period: 14, k_smooth: 3, d_smooth: 3}
    - name: williams_r
      params: {period: 14}
    - name: cci
      params: {period: 20}
    - name: mfi
      params: {period: 14}
      requires_volume: true
  volatility:
    - name: bollinger_bands
      params: {period: 20, std_dev: 2}
    - name: keltner_channels
      params: {period: 20, atr_mult: 2}
    - name: donchian_channels
      params: {period: 20}
  volume:
    - name: obv
      requires_volume: true
    - name: cmf
      params: {period: 20}
      requires_volume: true

# Backtesting parameters
backtest:
  initial_capital: 10000
  position_size: 1.0  # fraction of capital per trade
  transaction_cost: 0.001  # 0.1% per trade
  slippage: 0.0005  # 0.05% slippage
  direction: long_only  # long_only, long_short
  walk_forward:
    enabled: true
    in_sample_ratio: 0.7
    method: rolling  # rolling, anchored

# Ranking configuration
ranking:
  weights:
    sharpe_ratio: 0.30
    max_drawdown: 0.25
    profit_factor: 0.20
    expectancy: 0.15
    win_rate: 0.10
  min_trades: 30
  benchmark: buy_and_hold

# Regime detection
regime:
  enabled: true
  method: adx  # adx, bollinger_width, hmm
  adx_threshold_trending: 25
  adx_threshold_ranging: 20

# Output configuration
output:
  directory: output
  formats: [html, csv]  # html, pdf, csv
  charts: true
  interactive_charts: true
```

### 6.6 Extensibility: Adding New Indicators

The modular indicator engine should make adding new indicators straightforward:

**Indicator interface:**
```python
class Indicator(ABC):
    """Base class for all indicators."""

    @property
    @abstractmethod
    def name(self) -> str:
        """Human-readable name."""
        pass

    @property
    @abstractmethod
    def category(self) -> str:
        """Category: trend, momentum, volatility, volume, pattern."""
        pass

    @property
    @abstractmethod
    def requires_volume(self) -> bool:
        """Whether this indicator needs volume data."""
        pass

    @abstractmethod
    def calculate(self, df: pd.DataFrame, params: dict) -> pd.DataFrame:
        """Calculate indicator values. Returns DataFrame with indicator columns."""
        pass

    @abstractmethod
    def generate_signals(self, df: pd.DataFrame, params: dict) -> pd.Series:
        """Generate buy(+1)/sell(-1)/hold(0) signals."""
        pass

    @property
    @abstractmethod
    def default_params(self) -> dict:
        """Default parameters for this indicator."""
        pass

    @property
    @abstractmethod
    def min_bars_required(self) -> int:
        """Minimum number of bars needed for valid calculation."""
        pass
```

**Indicator registry:**
```python
class IndicatorRegistry:
    """Registry for all available indicators."""

    _indicators: dict[str, type[Indicator]] = {}

    @classmethod
    def register(cls, indicator_class: type[Indicator]):
        cls._indicators[indicator_class.name] = indicator_class

    @classmethod
    def get(cls, name: str) -> type[Indicator]:
        return cls._indicators[name]

    @classmethod
    def list_all(cls) -> list[str]:
        return list(cls._indicators.keys())

    @classmethod
    def list_by_category(cls, category: str) -> list[str]:
        return [name for name, cls_ in cls._indicators.items()
                if cls_.category == category]
```

Adding a new indicator requires:
1. Create a new class implementing the `Indicator` interface
2. Register it with the `IndicatorRegistry`
3. Add default configuration to the YAML template
4. No changes to the pipeline, backtesting, or reporting code needed

### 6.7 Error Handling and Logging

The application should implement comprehensive error handling:

- **Data loading errors**: File not found, malformed CSV, missing columns -- clear error messages with guidance
- **Insufficient data**: Warn when data is too short for selected indicators. Suggest alternatives.
- **Missing volume**: Automatically disable volume-dependent indicators with a warning
- **Indicator calculation errors**: Catch NaN propagation, division by zero in oscillators at extremes
- **Backtest anomalies**: Warn if fewer than 30 trades are generated (statistically insufficient)

Logging should be structured (Python `logging` module) with levels:
- **INFO**: Pipeline stage progression, indicator completion, key results
- **WARNING**: Data quality issues, insufficient trades, disabled indicators
- **ERROR**: Failed calculations, configuration errors
- **DEBUG**: Detailed computation steps, intermediate values

### 6.8 Recommended Technology Stack Summary

| Component | Recommended | Alternative | Rationale |
|-----------|------------|------------|-----------|
| Language | Python 3.10+ | -- | Best library ecosystem for financial analysis |
| Data manipulation | pandas 2.x | polars | Universal standard, all libraries expect pandas |
| Indicator computation | TA-Lib | pandas-ta | Speed, reliability, most comprehensive |
| Backtesting | vectorbt | custom NumPy | Speed for parameter optimization, built-in metrics |
| Static charts | mplfinance | matplotlib | Purpose-built for financial charts |
| Interactive charts | Plotly | Bokeh | Best interactivity, easy HTML export |
| Report generation | Jinja2 + HTML | -- | Flexible, professional output |
| PDF generation | WeasyPrint | pdfkit | Pure Python, no external dependencies |
| Configuration | YAML (PyYAML) | TOML | Human-readable, supports nested config |
| Data I/O | pandas (CSV) + pyarrow (Parquet) | -- | Standard tools for both formats |
| Testing | pytest | unittest | Better fixtures, parametrize for indicator testing |
| Logging | Python logging | loguru | Standard library, no extra dependency |

### 6.9 Proposed Project File Structure

```
technical-analysis/
  src/
    __init__.py
    config.py              # Configuration loading and validation
    data/
      __init__.py
      loader.py            # DataLoader interface and implementations
      validator.py         # Data validation and cleaning
      quality.py           # Data quality report generation
    indicators/
      __init__.py
      base.py              # Indicator base class and registry
      trend.py             # SMA, EMA, MACD, Ichimoku, Parabolic SAR
      momentum.py          # RSI, Stochastic, Williams %R, CCI, MFI
      volatility.py        # Bollinger Bands, ATR, Keltner, Donchian
      volume.py            # OBV, A/D Line, CMF, VWAP
      patterns.py          # Candlestick and chart patterns
    signals/
      __init__.py
      generator.py         # Signal generation from indicator values
      regime.py            # Market regime detection (ADX-based)
      composite.py         # Multi-indicator composite signals
    backtest/
      __init__.py
      engine.py            # Backtesting engine (vectorbt wrapper)
      walk_forward.py      # Walk-forward analysis
      metrics.py           # Performance metric calculations
      scoring.py           # Composite scoring and ranking
    reporting/
      __init__.py
      charts.py            # Chart generation (mplfinance + Plotly)
      html_report.py       # HTML report generation
      pdf_report.py        # PDF report generation
      export.py            # CSV/data export
    pipeline.py            # Main pipeline orchestration
  tests/
    test_data_loader.py
    test_validator.py
    test_indicators.py
    test_signals.py
    test_backtest.py
    test_metrics.py
    test_pipeline.py
  config/
    default_config.yaml    # Default configuration
    example_configs/
      commodity_gold.yaml
      forex_eurusd.yaml
      crypto_btc.yaml
      equity_spy.yaml
  data/
    sample/                # Sample data files for testing
  output/                  # Generated reports (gitignored)
  requirements.txt
  setup.py
```

---

## Appendix A: Key Python Package Versions

| Package | Recommended Version | Notes |
|---------|:------------------:|-------|
| Python | 3.10 - 3.12 | 3.10+ for match statements, 3.12 for performance |
| pandas | 2.1+ | Major performance improvements in 2.x |
| numpy | 1.26+ | Compatible with pandas 2.x |
| ta-lib (Python) | 0.4.28+ | Requires TA-Lib C library |
| pandas-ta | 0.3.14b1+ | Watch for archival status |
| vectorbt | 0.26+ | Free version sufficient |
| mplfinance | 0.12.10+ | Stable, well-maintained |
| plotly | 5.18+ | Significant improvements in recent versions |
| jinja2 | 3.1+ | Template engine for reports |
| weasyprint | 60+ | PDF generation |
| pyyaml | 6.0+ | Configuration file parsing |
| pyarrow | 14+ | Parquet file support |
| pytest | 7.4+ | Testing framework |

## Appendix B: Data Source API Rate Limits

| Source | Free Tier Limit | Paid Tier | Notes |
|--------|:-------------:|:---------:|-------|
| Yahoo Finance (yfinance) | Unofficial -- no official limit but throttled | N/A | Can be blocked. Use responsibly. |
| Alpha Vantage | 25 requests/day | 75-1200 req/min depending on plan | Best free option for multi-asset |
| FRED | 120 requests/minute | Same | Government data, very reliable |
| CoinGecko | 10-30 requests/minute | Higher | Best for crypto |
| Polygon.io | 5 API calls/minute (free) | Unlimited (paid) | Professional grade |
| IEX Cloud | 50,000 messages/month (free) | Higher tiers | US equities focused |

For a batch analysis application processing daily data, even the most restrictive free tiers are sufficient. A single API call typically retrieves an entire price history. Rate limits only matter if fetching many symbols.

## Appendix C: Minimum Data Requirements Quick Reference

To run the full suite of 27+ indicators with walk-forward analysis:

- **Absolute minimum**: 500 trading days (~2 years) -- allows 200-day SMA initialization plus 300 days for basic backtesting
- **Recommended minimum**: 1,260 trading days (~5 years) -- allows proper walk-forward analysis with multiple windows
- **Ideal**: 2,520+ trading days (~10 years) -- captures multiple market cycles and regime changes

The application should calculate the effective analysis start date after accounting for the warmup period of the longest indicator selected, and warn the user if remaining data is insufficient for statistically significant backtesting (< 30 trades expected).
