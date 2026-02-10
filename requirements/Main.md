# Application requirements

- The application that will analyze historical daily spot price data using [technical analysis](https://en.wikipedia.org/wiki/Technical_analysis) techniques. 
- The assets classes considered will be equities.  No forex, commodities, cryptocurrencies, or anything else.
- Consider long trading positions only. No short selling recommendations. 
- All long positions will use the entire balance of the trading account for buys and sells. Long positions are either completely in the market or out of it.
- The application will analyze one asset at a time.
- The application will report investment strategies that best fit the asset over the length of the analysis. An investment strategy is defined as one or more technical analysis indicators that are used to signal a buy or sell. Examples of technical indicators are moving averages, RSI, MACD. Best fit means indicates buys and sells most profitably over the entire analysis.
- The application will provide a verbose mode that will document what technical analysis indicators it examined and rejected. Include a summary of what was found.
- Make indicators that produce no trades available in verbose output.
- The application will be run once a day to accomodate new daily price data.
- The application should benchmark indicator performance against buy-and-hold and the S&P 500 index.

# Trade profitability reporting
- The application will report for indicators chosen trade-by-trade entry/exit dates with trade profit or loss.
- The application will report indicated buys and sells and the profitability for each trade.
- All trades will be complete. That is, buys will use all available capital and sells will include all shares. 
- The application will provide summary statistics over time such as average length of trade, average profitability, average length of time between trades.
- The application will all the user to indicate the initial trading account size and starting point for trade profitability reporting. Default to account size $10,000 and starting point January 1, 2000.
- The application is stateless and will not require persistence between runs. I understand that results may change when new data is analyzed.
- The application will provide output that includes an actionable signal ("BUY today" / "HOLD" / "SELL today").
- If the maximum acceptable drawdown is reached, sell the position.
- Assume zero return on cash.

# Supplemental Information
- CSV historical data will be provided. Volume data is included. See history-data/NVDA.csv for an example.
- Historical data will be stock-split adjusted and include volume data. The source for historical data has not yet been determined.
- The target trading frequency should be configurable with 30 days as the default.
- Transaction costs are not material at the low target trading volume.
- Ignore earning dates entirely in the analysis.
- Use standard output for analysis. 
- Use standard error for any processing errors.
- Issue warnings over standard error for historical data anomilies such as Missing trading days, Zero-volume days, etc.

# Out of scope
- The application will not support user-supplied custom indicators.

# User options
- The maximum acceptable drawdown should be configurable, but use 30% as the default.
- The maximum number of indicator combinations considered should be configurable with 5 as a default.
- The application configuration should be YAML. Allow the user to specify the configuration location, but default to ~/.config/technical-analysis/config.yaml.

# Product roadmap
- Feel free to make technical implementation recommendations
- In future, I may extend analysis to options.
- In future, I may provide access to an API for pricing data instead of using CSV inputs.
- In future, I may expand output options beyond terminal output.