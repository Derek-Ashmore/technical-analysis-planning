# Application requirements

- The application that will analyze historical daily spot price data using [technical analysis](https://en.wikipedia.org/wiki/Technical_analysis) techniques. 
- The assets classes considered will be equities.  No forex, commodities, cryptocurrencies, or anything else.
- Consider long trading positions only. No short selling recommendations.
- The application will analyze one asset at a time.
- The application will tell me what technical analysis indicators, such as moving averages, RSI, MACD, and others that best fit the data. Best fit means indicates buys and sells most profitably.
- The application will provide a verbose mode that will document what technical analysis indicators it examined and rejected. Include a summary of what was found.
- The application will report for indicators chosen trade-by-trade entry/exit dates with trade profit or loss.
- The application will report indicated buys and sells and the profitability for each trade. 
- The application will provide summary statistics over time such as average length of trade, average profitability, average length of time between trades.
- The application will be run once a day to accomodate new daily price data.
- The application should benchmark indicator performance against buy-and-hold and the S&P 500 index.

# Supplemental Information
- CSV historical data will be provided. Volume data is included. See history-data/NVDA.csv for an example.
- The target trading frequency should be configurable with 30 days as the default.
- Transaction costs are not material at the low target trading volume.
- The maximum acceptable drawdown should be configurable, but use 30% as the default.
- The maximum number of indicator combinations considered should be configurable with 5 as a default.

# Product roadmap
- Feel free to make technical implementation recommendations
- In future, I may extend analysis to options.
- In future, I may provide access to an API for pricing data instead of using CSV inputs.