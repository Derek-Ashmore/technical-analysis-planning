# Data Research Questions v1: Additional Information Requests

Questions arising from the data considerations and architecture research that would help refine implementation decisions.

---

## Questions for the User

### 1. Data Source Specifics

**Q1.1: Will the application primarily consume CSV files, or should it also support real-time API data fetching?**
The architecture supports both, but knowing the primary use case determines which deserves more polish. CSV-first means focusing on column mapping flexibility and format detection. API-first means building a robust connector layer with caching and rate limiting.

**Q1.2: For commodity data, does "spot price" refer to physical spot or front-month futures proxy?**
Many commodity "spot price" datasets (including FRED) actually report front-month futures prices as a proxy for spot. True physical spot prices may only be available for precious metals (London Fix, for example). This distinction affects whether we need to worry about contract roll artifacts in the data.

**Q1.3: Will the application need to support multiple data providers simultaneously (e.g., gold from FRED, stocks from Yahoo Finance)?**
This would require a unified data loader interface with provider-specific adapters. The proposed architecture supports this, but the scope of adapter development depends on how many providers are needed.

### 2. Asset Class Scope

**Q2.1: Should the application support all four asset classes (commodities, forex, crypto, equities) from the start, or should we focus on one initially?**
Each asset class has unique data handling requirements. Commodities and equities are the simplest starting points. Crypto adds 24/7 trading complexity. Forex adds the volume data absence problem. Building for all four from day one is achievable but increases the initial scope.

**Q2.2: For equities, should the application handle individual stocks only, or also ETFs and indices?**
ETFs and indices are simpler (no earnings, fewer corporate actions for indices) but add scope. ETF data typically includes volume and adjusted close, making them the easiest equity-type asset to start with (e.g., SPY for S&P 500 exposure).

### 3. Volume Data Handling

**Q3.1: How should the application behave when volume data is absent?**
Three options were identified:
- **(a)** Automatically disable volume-dependent indicators and warn the user.
- **(b)** Require the user to explicitly opt out of volume indicators.
- **(c)** Accept tick volume as a proxy (with reduced confidence flagging).

Option (a) is recommended for the best user experience. Confirmation of this design decision would be helpful.

**Q3.2: For forex, should tick volume from brokers be accepted as a proxy, or should volume indicators be categorically disabled?**
Academic research is mixed on whether tick volume is a meaningful proxy. Some studies show correlation between tick volume and actual volume; others find it too noisy. The safest approach is to disable volume indicators for forex entirely, but some users may want the option.

### 4. Reporting and Output

**Q4.1: Is PDF output a hard requirement, or is HTML sufficient?**
HTML reports are significantly easier to generate and support interactivity (Plotly charts). PDF adds a dependency (WeasyPrint or pdfkit) and limits chart interactivity. If HTML is sufficient, the implementation is simpler and the output is richer.

**Q4.2: Should the application produce a single comprehensive report, or separate output files?**
The proposed architecture generates both: a comprehensive HTML report and separate CSV/data files. Confirmation that this is the desired approach would be helpful.

**Q4.3: Is console/terminal output important for quick runs?**
A "quick mode" that prints summary results to the terminal (top 5 indicators, key metrics) without generating full reports would be useful for iterative analysis. Should this be included in the initial scope?

### 5. Configuration and Defaults

**Q5.1: Should the application run with sensible defaults and no configuration file, or always require explicit configuration?**
The recommended approach is: run with defaults if no config file is provided, but allow overriding via YAML config. This makes the tool immediately usable while supporting advanced customization.

**Q5.2: For the configurable lookback window (default 30 days), should this apply to:**
- **(a)** Pattern detection window only
- **(b)** The entire analysis period (only analyze the last 30 days)
- **(c)** All indicator lookback periods simultaneously
- **(d)** A separate, independent setting from indicator parameters

Option (d) is recommended -- the 30-day window should be for pattern detection, independent of indicator-specific parameters (which have their own standard values like RSI=14, MACD=12/26/9). This was also raised as an open question in the prior research.

### 6. Technical Implementation

**Q6.1: Is Docker an acceptable deployment mechanism?**
Docker would solve the TA-Lib installation problem entirely (pre-built image with all C dependencies). If Docker is acceptable, the installation documentation becomes trivial. If not, detailed platform-specific build instructions are needed.

**Q6.2: Should the application be CLI-only, or should it include a web interface?**
The proposed architecture is CLI-first (run a command, get a report). A web interface (Dash, Streamlit, or FastAPI) would add significant scope but improve usability. This is a scope decision that affects timeline.

**Q6.3: What Python version is the minimum target?**
The recommended stack requires Python 3.10+ for match statements and type union syntax. If the user's environment is constrained to an older version, alternative syntax is needed.

---

## Summary of Priority

| Priority | Questions | Reason |
|----------|-----------|--------|
| **Before implementation** | Q1.1 (CSV vs API), Q2.1 (asset class scope), Q5.1 (defaults), Q6.2 (CLI vs web) | Determines initial scope and architecture |
| **During design** | Q1.2 (spot definition), Q3.1 (volume behavior), Q4.1 (PDF requirement), Q5.2 (lookback semantics) | Affects specific design decisions |
| **During development** | Q1.3 (multi-provider), Q2.2 (ETFs/indices), Q3.2 (tick volume), Q4.2-4.3 (output format details), Q6.1 (Docker), Q6.3 (Python version) | Implementation details that can be decided later |
