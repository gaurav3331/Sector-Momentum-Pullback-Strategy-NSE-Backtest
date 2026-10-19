# Sector Momentum Pullback Strategy: NSE Backtest

A Python / Jupyter backtest of a **trend-following pullback strategy** on Indian (NSE) stocks.
It buys strong stocks in strong sectors when a short pullback ends, and rides the move with a wide ATR trailing stop.

> **Status:** research project. Results are historical backtests, not a promise of future returns. Read the [Limitations](#limitations) before drawing conclusions.

---

## Strategy rules

| Step | Rule |
|------|------|
| 1. Sector filter | The stock's **sector index monthly RSI > 55**. Stocks with no sector index use **Nifty 50 monthly RSI > 55**. |
| 2. Trend filter | Stock **monthly RSI > 60** and **weekly RSI > 58**. |
| 3. Signal | **Daily RSI crosses back above 50** (the pullback is over). |
| 4. Entry | Buy at the **next day's open**. |
| 5. Stop-loss | Initial stop = entry − 4 × ATR(14). It then **trails** at highest close − 4 × ATR and only moves up. |
| 6. Exit | Stop hit intraday (exit at the stop price) or gap below the stop (exit at the open). There is no profit target. |
| 7. Position sizing | About 6.67% of equity per trade, whole shares, **max 15 open positions**. If there are more signals than free slots, the strongest **60-day momentum** is taken first. |

RSI is Wilder's RSI (period 14). Weekly and monthly RSI are calculated on weekly/monthly closes and mapped back to daily rows using the last completed period, so there is no look-ahead.

---

## What the notebook does

1. **Loads and cleans data**: stock and index CSVs, date index, missing values, duplicates.
2. **Calculates indicators**: daily, weekly and monthly RSI for every stock and every sector index.
3. **Classifies stocks into sectors**: 19 stock groups mapped to 9 sector indices; the rest are "Other" and use Nifty 50.
4. **Visualises the filters**: timelines of when each sector was above RSI 60 and when each stock had weekly and monthly RSI above 60.
5. **Runs the backtest** (`run_backtest`): trade log, win/loss counts, return buckets, results by sector and by year.
6. **Compares with Nifty 50**: CAGR, volatility, Sharpe, Sortino, Calmar, drawdown, beta, alpha, VaR/CVaR, plus drawdown and rolling Sharpe charts.
7. **Exports a trade log**: every trade with entry/exit dates, duration, return, R-multiple and a timeline chart.

---

## Results (historical, 2003-07 to 2023-11, ~20 years, 218 stocks)

| Metric | Strategy | Nifty 50 (price index) |
|---|---|---|
| CAGR | 21.8% | 14.7% |
| Max drawdown | −47.8% | −59.9% |
| Calendar years beating Nifty | 12 of 21 | |

| Trade statistics | |
|---|---|
| Signals / trades taken | 6,227 / 1,069 (~53 trades per year) |
| Win rate | 47.5% |
| Average R | 0.30 |
| Profit factor | 2.43 |
| Average holding time | ~74 days |
| Average open positions | ~11 |

Risk metrics from the notebook's comparison cell:

| | Strategy | Nifty 50 |
|---|---|---|
| Annual volatility | 17.5% | 22.0% |
| Sharpe ratio | 0.90 | 0.48 |
| Sortino ratio | 1.43 | 0.68 |
| Max drawdown (this cell) | −39.9% | −59.9% |
| Beta / annual alpha | 0.36 / ~11.9% | |

The strategy wins less than half of its trades. Its profits come from a few large winners that the wide trailing stop lets run, while losses stay small. The drawdown figures in the two cells above differ because they are calculated from different equity series; re-running the notebook reproduces both.

---

## Limitations

- **Trading costs default to zero.** `cost` is set to `0.0`. Real brokerage, STT, stamp duty and slippage reduce returns. Set `cost` (for example `0.002` per side) before trusting the numbers.
- **Survivorship bias.** The stock universe is a fixed list. Stocks that were delisted or dropped out of the indices over the years are likely missing, which flatters results.
- **In-sample parameters.** Thresholds (55 / 60 / 58 / 50) and the 4 × ATR stop were chosen by looking at the same history. No walk-forward or out-of-sample test has been done yet.
- **Idealised fills.** The backtest assumes fills at the next open and at the stop price using daily bars. Real fills, especially on gaps and in illiquid stocks, will differ.
- **Large, long drawdowns.** The strategy had drawdowns of about 40-48% and a drawdown lasting roughly 2.7 years. Most trades lose; the edge relies on holding a few big winners.
- **Uneven results.** The strategy beat Nifty in 12 of 21 calendar years and lost money in several years (for example 2008, 2011, 2016, 2018, 2019 and 2022).

---

## Data

Data is **not included** in this repository. Add your own daily price CSVs in this layout:

```
stocks/
    RELIANCE.NS.csv
    TCS.NS.csv
    ...
indices/
    NIFTY50.csv                  (required)
    Automobiles.csv
    Banks.csv
    FMCG.csv
    IT.csv
    Metals_and_Mining.csv
    Oil_and_Gas.csv
    Power.csv
    Pharma_and_Healthcare.csv
    Realty.csv
```

Each CSV needs the columns `Date, Open, High, Low, Close` (daily data, dates as `YYYY-MM-DD`). Stock file names must match the tickers used in the sector lists inside the notebook.

---

## Getting started

```bash
git clone <your-repo-url>
cd <your-repo-name>
pip install numpy pandas matplotlib jupyter
jupyter notebook trend_pullback_strategy.ipynb
```

Place the `stocks/` and `indices/` folders next to the notebook, then run all cells from top to bottom.

---

## Settings

All strategy settings are in the `FINAL` dictionary:

| Setting | Default | Meaning |
|---|---|---|
| `sector_th` | 55 | Minimum sector monthly RSI |
| `monthly_th` / `weekly_th` | 60 / 58 | Minimum stock monthly / weekly RSI |
| `pullback_rsi` | 50 | Daily RSI level the signal crosses back above |
| `atr_mult` | 4.0 | Trailing stop distance in ATRs |
| `max_pos` | 15 | Maximum open positions |
| `size_pct` | 0.0667 | Fraction of equity per trade (when `sizing="pct"`) |
| `sizing` | `"pct"` | `"pct"` = fixed % of equity, `"risk"` = risk a fixed % per trade |
| `risk_pct` | 0.01 | Equity risked per trade (when `sizing="risk"`) |
| `include_other` | `True` | Include stocks that have no sector index (they use Nifty 50) |
| `cost` | 0.0 | Cost per side, e.g. `0.002` = 0.2% on buy and on sell |
| `max_hold` | `None` | Optional maximum holding time in calendar days |
| `start` / `end` | `None` | Optional date range to test |

Example: run the backtest with costs and a different stop.

```python
stats, trades, equity = run_backtest({"cost": 0.002, "atr_mult": 5})
```

---

## Disclaimer

This project is for education and research only. It is not investment advice. Trading involves risk of loss, and past performance does not guarantee future results.
