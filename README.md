An equity trading strategy based on Andreas Clenow's book "Stocks on the Move"

# Weekly Regression Trend Strategy (S&P 500)
**SPY 200DMA regime filter • Inverse-ATR sizing • Position cap • Turnover controls • Wed plan / Thu execute**

This script runs a historical backtest of a **weekly, long-only trend strategy** on a tradable S&P 500 universe.

Trade generator files: generate weekly trading signals

Each week it:
1) **Ranks stocks by trend strength** (a regression slope signal),
2) **Keeps only the best names** (top percentile),
3) **Sizes positions by risk** (inverse ATR volatility),
4) **Optionally blocks new buying** when the market regime is bearish (SPY below 200DMA),
5) **Trades only when changes are meaningful** (drift + minimum trade size + minimum new position).

---

## How the strategy works (plain English)

### 1) Wednesday: “Plan” (no lookahead)
On **Wednesday close**, the system:
- Checks the **SPY regime** (above/below 200DMA)
- Ranks all eligible stocks by `slope_adj`
- Selects the **top `TOP_PERCENTILE`** stocks (default: top 5%)
- Computes **target portfolio weights** using inverse ATR (lower ATR → larger weight)
- Applies a **max position cap** (e.g., 12%)
- Converts target weights into **target shares** using Wednesday close prices (proxy)

It then builds a list of trades to execute the next trading day.

### 2) Thursday: “Execute”
On **Thursday open**, the system executes the planned trades:
- **SELLS first**, then **BUYS**
- If a ticker’s open price is missing, it falls back to last known close
- Enforces a **cash reserve floor** before buying

### 3) Daily: Mark-to-market
Every day it computes portfolio value using **close prices** to create an equity curve.

---

## The big knobs (major variables)

### A) Signal & Lookbacks (upstream inputs)
These are **conceptually part of the system**, but the script *consumes them precomputed*:

- **Regression lookback (signal generation)**  
  `slope_adj` is assumed to come from a regression run over a fixed window (example: 90/126/252 trading days).  
  ✅ Recommended to document as:
  - `REGRESSION_LOOKBACK_DAYS = <N>`

- **ATR lookback (volatility sizing)**  
  The script uses ATR values loaded from files (currently `atr20`).  
  ✅ Recommended to document as:
  - `ATR_LOOKBACK_DAYS = 20`

> If you want these controlled directly inside this script, you’d need to compute regression slopes and ATR from price history here instead of loading them.

---

### B) Selection & Rebalance cadence
- `REBALANCE_DAY = "Wednesday"`  
  Signals are generated weekly on Wednesday.
- `TOP_PERCENTILE = 0.95`  
  Invest in the top 5% of names by slope.

---

### C) Portfolio risk controls
- `MAX_POSITION_WEIGHT = 0.12`  
  No single position should exceed 12% of portfolio value (then weights get redistributed).

- `MIN_CASH_RESERVE = 20000.0`  
  Always keep at least $20k in cash (planning checks + execution checks).

---

### D) SPY regime filter
- `SPY_REGIME_CONFIRM_DAYS = 1`  
  How “sticky” the regime is:
  - `1` = immediate flip when SPY crosses 200DMA
  - `5+` = must stay above/below for N consecutive days to confirm

**Behavior:**  
- If SPY is bearish on Wednesday, the system **does not add new exposure** on Thursday (buys are blocked).  
- Sells/reductions can still happen.

---

### E) Turnover / trade filters (the “don’t churn” rules)
These three settings control how aggressive rebalancing is:

- `DRIFT_THRESHOLD = 0.05`  
  Only trade if the **difference between target weight and current weight** is at least 5%.  
  (Exception: if a position breaches the max cap, it may be forced to trim.)

- `MIN_TRADE_VALUE = 10000`  
  Skip trades smaller than $10,000 (reduces noise + transaction churn).

- `MIN_NEW_POSITION_WEIGHT = 0.005`  
  Don’t open brand-new positions unless they would be at least 0.5% of the portfolio.

---

## Outputs (what files you get)

All output is saved under:

- Trades + equity + rankings:  
  `./13-trading_output_regression_insp500_spyfilter_cap15/`

- Performance summary CSVs:  
  `./14-trading_output_regression_insp500_spyfilter_performance_output_cap15/`

### 1) Executed trades
`13-trades_regression_insp500_spyfilter_cap15.parquet`

### 2) Daily equity curve
`13-equity_curve_regression_insp500_spyfilter_cap15.parquet`

### 3) Weekly rankings (pre-filter)
`13-weekly_rankings_pre_filter_cap15.parquet`

This one is super useful for research/debugging: it includes **all top-ranked names each week** *before* drift/trade-size/min-weight filters remove anything.

### 4) Performance outputs
- `14-regression_insp500_spyfilter-performance_summary_cap15.csv`
- `14-regression_insp500_spyfilter-yearly_comparison_cap15.csv`

Includes CAGR, vol, Sharpe/Sortino, max drawdown, Calmar, plus year-by-year comparison vs SPY.

# File Listings
`0-list_notebooks.ipynb`
Script to list all Jupyter notebook (.ipynb) files in a directory.   Used for automated execution scripts.

`0a-execution_script.ipynb`
Automates the data pull and the back testing

`1-SP500MEMBERSHIPBUILDER.ipynb`
Builds a daily S&P 500 membership matrix from SHARADAR/SP500 events via Nasdaq Data Link.
Loads the API key from NASDAQ_DATA_LINK_API_KEY, downloads constituent add/remove events,
simulates daily membership across all business days in the event range, and saves the full
membership matrix to Parquet at ./1-sp500_membership_daily_matrix/sp500_membership_full.parquet.
Also computes membership metadata (first/last/exit/current status), reports recent additions
and removals, and writes a timestamped diagnostics CSV to system_verification/

`1-SP500MEMBERSHIPBUILDER.`

Consumes the daily S&P 500 membership matrix Parquet, computes first join and last exit dates
per ticker, flags today’s additions/removals and recent changes within RECENT_WINDOW, and saves
the join/exit table to Parquet and CSV alongside a diagnostics CSV of detected events.

`2a-Incremental_price_fetch.ipynb`

Incrementally updates SHARADAR SEP price files for all tickers in the S&P 500 membership matrix.
Loads the Nasdaq Data Link API key, fetches full histories for missing tickers and only new rows
after the latest date for existing files, warns when data is stale beyond MAX_STALE_DAYS, and
persists per-ticker CSVs in ./2-all_prices/sharadar_sep_full.

Use case: run after executing 2b-Bootrap_All_Prices.ipynb to keep SHARADAR SEP data up to date.

`2b-Bootstrap_All_Prices.ipynb`
Run this script when initializing the system to pull all the prices

Downloads full SHARADAR/SEP price history for every ticker that has ever appeared in the S&P 500,
saving per-ticker CSVs in ./2-all_prices/sharadar_sep_full. Loads the membership list from the
daily matrix, fetches data in chunks with API key configured via NASDAQ_DATA_LINK_API_KEY, and
prints diagnostics for missing data, short histories after 1998, stale prices (>30 days), very
late starts, very early terminations, and large gaps (>10 days) between trading days.

`3-adjusted_All_Prices_OHLC.ipynb`
Scans raw SHARADAR/SEP CSVs for all tickers, validates required fields and date continuity,
computes split-adjusted OHLC using closeadj-derived factors, and writes Parquet outputs to
./3-adjusted_All_Prices_OHLC. Logs validation issues (missing columns, nonpositive prices,
invalid dates, large gaps, adj_factor jumps, processing errors) to a timestamped CSV in
./system_verification/3-adjusted_All_Prices_OHLC.

`4-ATR20_adjusted_All_Prices.ipynb`

Computes 20-day ATR from adjusted OHLC parquet files, validating date order and required columns,
logging issues (missing data, non-monotonic dates, insufficient history, negative TR, ATR spikes)
to a timestamped CSV in ./system_verification/4-ATR20_adjusted_All_Prices. Reads per-ticker inputs
from ./3-adjusted_All_Prices_OHLC, writes ATR20-enriched parquet outputs to ./4-ATR20_adjusted_All_Prices.

