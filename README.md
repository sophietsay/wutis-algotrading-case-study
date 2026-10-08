# Intraday Momentum Strategy for SPY: Case Study

This notebook (`STsay_strategy v4.ipynb`) replicates and extends the strategy of Zarattini, Aziz and Barbon (2025), *Beat the Market: An Effective Intraday Momentum Strategy for S&P500 ETF (SPY)*.

- **Part 2** implements a base version of the strategy and one extension from the paper, the VWAP trailing stop, and backtests both on a train/test split with transaction costs.
- **Part 3** tests our own variation, which scales the daily exposure by the probability of a high-volatility regime from a Markov-switching model.

## Files

| File | Description |
|---|---|
| `STsay_strategy v4.ipynb` | The notebook: code, results and discussion |
| `spy_1min.parquet` | 1-minute SPY bars, created by the notebook on the first run (Section 0) |

## How to run

1. Install the dependencies:
   ```
   pip install numpy pandas matplotlib statsmodels pyarrow alpaca-py
   ```
   `alpaca-py` is needed only to download the data.
2. If `spy_1min.parquet` is not in the working directory, set the environment variables `ALPACA_API_KEY` and `ALPACA_SECRET_KEY`. A free Alpaca paper-trading account is sufficient. The keys are never written into the notebook.
3. Run all cells from top to bottom. Once the data file exists, the whole notebook runs in well under a minute.

## Data layout

After loading, every price field is stored as a wide table with one row per trading day and one column per minute (390 columns, 09:30–15:59 New York time):

```
close, open_, high, low, volume, vwap, UB, LB, sigma   ->   days x 390 minutes
```

All later steps are vectorised operations on these tables. Only the backtest loops over days. Decisions are taken at 12 times per day (10:00, 10:30, ..., 15:30), so the signal inputs are sliced down to tables of size days × 12.

## Notebook structure

### Setup

| Cell | Content |
|---|---|
| Imports and parameters | `LOOKBACK = 14`, commission \$0.0035 and slippage \$0.001 per share, `INITIAL_AUM = 100,000`, `SPLIT_DATE = '2023-01-01'` (train before, test from this date) |
| Part 1 | Short summary of the paper: hypothesis, signal, execution, risk control and regimes |

### Part 2. Implementation and backtest

| Section | What it does | Main objects |
|---|---|---|
| 0. Download data | Downloads 1-minute SIP bars from Alpaca year by year (2016 onwards) and saves them to Parquet; skipped if the file exists | `spy_1min.parquet` |
| 1. Load data | Converts timestamps to New York time, keeps 09:30–15:59, computes the daily close and previous close, pivots to days × minutes tables, keeps days that have both the 09:30 and the 15:59 bar, fills rare missing minutes | `df`, `daily_close`, `prev_close_all`, `close`, `open_`, `high`, `low`, `volume` |
| 2. Noise Area | Average absolute move from the open over the previous 14 days, separately for each minute (`rolling(14).mean().shift(1)`); bands use the gap-adjusted references `max/min(Open, previous Close)` | `sigma`, `UB`, `LB`, `valid` |
| 3. VWAP | Cumulative intraday VWAP from the typical price, restarting every day | `vwap` |
| 4. Decision times | Slices the inputs at the bar before each decision time (signal) and takes the open of the bar at the decision time (execution) | `at_signal()`, `price_sig`, `ub_sig`, `lb_sig`, `vwap_sig`, `fill_price`, `close_price`, `open_price` |
| 5. Position rules | `base_positions()` enters on a band breakout and holds until the opposite band is broken (then flips). `vwap_positions()` is long only above `max(UB, VWAP)` and short only below `min(LB, VWAP)` | `pos_base`, `pos_vwap` |
| 6. Backtest | `backtest()` simulates day by day: shares = floor(equity × exposure / open), P&L between consecutive fills, forced exit at the close, costs on every change of position. The optional `exposure` argument is used in Part 3 | `res_base`, `res_vwap` |
| 7. Returns and split | Collects daily returns with SPY buy-and-hold as the benchmark (zero strategy return on days without trading) and defines the train/test masks | `returns`, `trades`, `periods` |
| 8. Metrics | `metrics()` reports annualised return, annualised volatility, Sharpe ratio, maximum drawdown, hit ratio, skewness and number of days, computed per strategy and period | `summary` |
| 9. Plots | Equity curves (log scale) with the split marked; bar charts of Sharpe ratio, return and volatility for train vs test | |

### Part 3. Own strategy: regime-aware intraday momentum

| Section | What it does | Main objects |
|---|---|---|
| 10. Markov-switching model | Two-regime model with switching mean and variance (Hamilton, 1989) on daily SPY returns, fitted on the train period only (`np.random.seed(0)` makes the random starts reproducible) | `ms_res`, `HIGH`, `LOW` |
| 11. Filtered probability | Applies the train parameters to the full sample. Uses filtered (not smoothed) probabilities, shifted by one day so that day *t* uses information up to *t − 1* | `p_high` |
| 12. Regime plot | SPY price vs probability of the high-volatility regime | |
| 13. Scaled strategies | Extension traded with exposure `p_high`, and a naive benchmark that trades only when the 14-day realised volatility is above its train median | `res_ms`, `res_naive`, `rv14` |
| 14. Comparison | Same split, costs and metrics as Part 2, plus average exposure | `returns3`, `summary3` |
| 15. Plots, alpha/beta, yearly returns | Equity curves, metric bar charts, OLS alpha/beta vs SPY per period, yearly returns and average regime probability per year | `yearly3` |

The notebook ends with **Conclusions and further work**.

## Look-ahead safeguards

- The Noise Area on day *t* uses only days *t − 14 … t − 1* (`shift(1)` after the rolling mean).
- The signal at HH:MM uses the close of the bar ending at HH:MM. The trade is executed at the open of the bar starting at HH:MM.
- Position size uses the previous day's equity and the current day's open.
- The regime model and the volatility threshold are estimated on the train period only. The regime probability is filtered and lagged by one day.
- All signals use past data only. The strategy is therefore run on the full sample at once, and the train/test split is applied only at the evaluation stage.
