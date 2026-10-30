# AAPL Intraday Time-Slot Return Analyzer

A Python notebook workflow that splits each trading session into fixed-length time slots, measures the return of every slot, and shows which times of day have historically been positive or negative. It also includes a walk-forward slot-based backtest and per-slot Sharpe ratios.

The current configuration analyzes **AAPL hourly bars over about 2 years**, but the ticker and slot size can be changed in one place.

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Data Source and Limitations](#data-source-and-limitations)
4. [Project Structure](#project-structure)
5. [Requirements](#requirements)
6. [Installation](#installation)
7. [Configuration](#configuration)
8. [How It Works](#how-it-works)
9. [Notebook Walkthrough](#notebook-walkthrough)
10. [Outputs](#outputs)
11. [Interpreting the Results](#interpreting-the-results)
12. [Changing the Analysis](#changing-the-analysis)
13. [Known Limitations and Caveats](#known-limitations-and-caveats)
14. [Troubleshooting](#troubleshooting)
15. [Roadmap](#roadmap)
16. [License and Disclaimer](#license-and-disclaimer)

---

## Overview

Many intraday traders believe certain hours of the session behave differently: the open is volatile, midday is quiet, and the close has its own pattern. This project tests that idea directly. It:

1. Downloads regular-hours intraday price data from Yahoo Finance.
2. Aggregates it into slots of a chosen length (for example 1 hour).
3. Computes the bar-to-bar return of each slot in each session.
4. Averages those returns by time of day across all sessions.
5. Visualises the results as heatmaps and bar charts.
6. Runs a walk-forward backtest in which each slot is traded long or short based only on its past behaviour.
7. Computes a Sharpe ratio for each slot.

---

## Features

- **Configurable slot size:** any multiple of the base data interval (5 minutes or 60 minutes, depending on the data source).
- **Regular trading hours only:** data is filtered to 09:30–16:00 New York time.
- **Session-level heatmap:** every session's slot return shown as a grid, from red (negative) to green (positive).
- **Per-slot statistics:** mean return, average up-move, average down-move, and up-rate for each slot.
- **Walk-forward backtest:** a slot's position on any day depends only on prior sessions, avoiding lookahead bias.
- **Performance metrics:** total return, annualised Sharpe, maximum drawdown, and hit rate, compared against buy and hold.
- **Per-slot Sharpe ratio:** annualised and ranked, with an optional risk-free rate.
- **Top and bottom slots:** a quick summary of the strongest and weakest times of day.

---

## Data Source and Limitations

Data comes from [yfinance](https://github.com/ranaroussi/yfinance), an unofficial wrapper around Yahoo Finance. Intraday history is limited by interval:

| Base interval | Maximum history | Approx. sessions | Use case |
|---|---|---|---|
| 1 minute | ~7 days (in practice) | ~5 | Not recommended for analysis |
| 5 minutes | ~60 days | ~40 | Fine-grained intraday timing, short sample |
| 60 minutes | ~730 days | ~500 | Hourly analysis, longer sample (default) |
| Daily | 20+ years | 5,000+ | Daily research only; cannot show intraday timing |

**Implication:** yfinance cannot provide 20 years of intraday data. For longer intraday histories, use a paid vendor such as Polygon.io, Databento, Alpaca, or IEX Cloud, and replace the `fetch_intraday` function.

---

## Project Structure

```
aapl-intraday-analyzer/
├── README.md                     # This file
├── aapl_slot_backtester.ipynb    # Main notebook (10 cells + Sharpe cell)
├── requirements.txt              # Python dependencies
└── outputs/                      # Optional: saved figures and CSVs
```

If you keep the code as cells rather than a notebook, the same ten sections map onto separate `.py` files or a single script in the order described in [Notebook Walkthrough](#notebook-walkthrough).

---

## Requirements

- Python 3.9 or newer
- Jupyter Notebook, JupyterLab, or VS Code with the Jupyter extension
- Internet access (to download data from Yahoo Finance)

Python packages:

```
numpy
pandas
yfinance
matplotlib
```

---

## Installation

```bash
# 1. Clone or create the project folder
mkdir aapl-intraday-analyzer && cd aapl-intraday-analyzer

# 2. (Recommended) Create a virtual environment
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install numpy pandas yfinance matplotlib jupyter

# 4. Launch Jupyter
jupyter notebook aapl_slot_backtester.ipynb
```

Run the cells in order from top to bottom. Use **Kernel → Restart & Run All** after changing any configuration value.

---

## Configuration

All user-adjustable settings are in **Cell 2**:

| Variable | Default | Description |
|---|---|---|
| `TICKER` | `'AAPL'` | Any Yahoo Finance ticker (stocks, ETFs, indices like `'^GSPC'`) |
| `BASE_INTERVAL` | `'60m'` | yfinance interval to download: `'5m'` or `'60m'` |
| `INTERVAL_SECONDS` | `3600` | Slot length in seconds; must be a multiple of the base interval |
| `LOOKBACK_DAYS` | `729` | Calendar days of history to request (max ~729 for `'60m'`, ~58 for `'5m'`) |
| `MARKET_OPEN_SEC` | `9*3600 + 30*60` | Session start (09:30 New York) |
| `MARKET_CLOSE_SEC` | `16*3600` | Session end (16:00 New York) |
| `TOP_N` | `3` | Number of best and worst slots to display |
| `WARMUP` (Cell 8) | `5` | Sessions used before the backtest starts trading |
| `RISK_FREE_ANNUAL` (Sharpe cell) | `0.0` | Annual risk-free rate for excess-return Sharpe |

---

## How It Works

### 1. Download and filter
The raw 60-minute (or 5-minute) bars are downloaded and converted to New York time. Only bars whose start time falls between `MARKET_OPEN_SEC` and `MARKET_CLOSE_SEC` are kept, which removes pre-market and after-hours trading.

### 2. Build slots
Each bar is assigned to a slot by flooring its time since the open to a multiple of `INTERVAL_SECONDS`. Bars within the same slot in the same session are combined into a single OHLCV record (first open, highest high, lowest low, last close, summed volume).

### 3. Compute returns
Within each session, the return of a slot is:

```
ret = Close(slot) / Close(previous slot) − 1
```

The first slot of each session uses its own open as the reference, so the overnight gap is excluded from intraday timing analysis.

### 4. Aggregate by time of day
For each slot position (for example 10:30), the mean return, average up-move, average down-move, and up-rate are computed across all sessions.

### 5. Walk-forward backtest
On each session, each slot is traded according to the sign of its **average return over all earlier sessions**:

- Average > 0 → long
- Average < 0 → short
- Not enough history → flat

Because the rule uses only past data, the backtest avoids lookahead bias. Returns are compounded across slots within each session.

### 6. Sharpe ratio per slot
For each slot:

```
Sharpe = (mean return − risk-free per bar) / std(return) × √252
```

---

## Notebook Walkthrough

| Cell | Purpose |
|---|---|
| 1 | Imports |
| 2 | Configuration (ticker, interval, lookback, session times) |
| 3 | Download and filter intraday data |
| 4 | Build slots and compute within-session returns |
| 5 | Per-slot statistics (mean, up/down averages, up-rate) |
| 6 | Session × slot return heatmap |
| 7 | Average return per slot bar chart |
| 8 | Walk-forward slot backtest |
| 9 | Performance metrics and equity curve |
| 10 | Top and bottom slots |
| Extra | Sharpe ratio per slot and bar chart (run after Cell 4) |

---

## Outputs

- **Session × slot heatmap:** rows are trading dates, columns are slot start times, colour shows the return in basis points. Use it to spot clusters of strong or weak slots and changes in behaviour over time.
- **Average return bar chart:** one bar per slot, green for positive average, red for negative.
- **Performance table:** total return, annualised Sharpe, maximum drawdown, and session hit rate for the slot strategy and buy and hold.
- **Equity curve:** growth of 1 for both strategies.
- **Top and bottom slots table:** average return, up-rate, and number of observations.
- **Sharpe table and chart:** each slot's annualised Sharpe, sorted from best to worst.

---

## Interpreting the Results

- **Statistical noise:** with about 500 sessions and 7 hourly slots, each slot has roughly 500 observations. A difference of a few basis points in average return is usually not statistically meaningful. Consider a t-test or bootstrap before acting on any pattern.
- **Sharpe ratios are approximate:** annualising with √252 is standard for daily data. Applied to hourly observations it is a rough comparison tool, not an exact annual figure.
- **Ranking, not absolute values:** the most useful output is the relative ordering of slots, not their absolute levels.
- **Regime dependence:** results from 2024–2026 reflect that period's market conditions. Check whether the pattern holds across sub-periods before trusting it.
- **Backtest realism:** the strategy trades every slot, so it can flip positions up to seven times a day. Transaction costs, spreads, and slippage would reduce returns substantially.

---

## Changing the Analysis

**Different ticker:**
```python
TICKER = 'MSFT'
```

**Five-minute slots over ~60 days:**
```python
BASE_INTERVAL = '5m'
INTERVAL_SECONDS = 300
LOOKBACK_DAYS = 58
```

**Two-hour slots:**
```python
INTERVAL_SECONDS = 7200         # must be a multiple of the 3600-second base
```

**Excess-return Sharpe with a 4% risk-free rate:**
```python
RISK_FREE_ANNUAL = 0.04
```

**Stronger backtest rule:** replace `np.sign(hist_mean)` with a threshold, for example `np.where(hist_mean > 0.5e-4, 1, np.where(hist_mean < -0.5e-4, -1, 0))`, to trade only when the historical average exceeds a minimum size.

---

## Known Limitations and Caveats

1. **Short intraday history:** yfinance caps intraday data at about two years for hourly bars and 60 days for 5-minute bars.
2. **Survivorship and selection bias:** the analysis covers only the chosen ticker. Patterns found in one stock may not generalise.
3. **Multiple comparisons:** testing seven hourly slots and reporting the best one overstates its significance. Adjust for this when drawing conclusions.
4. **No costs:** commissions, bid-ask spreads, and slippage are not modelled.
5. **No position sizing or risk management:** the backtest assumes full exposure on every slot.
6. **Yahoo data quality:** occasional missing bars, adjusted prices, or revisions can affect results. Verify important findings against a second source.
7. **Market regime:** results depend on the period sampled, and intraday patterns can disappear as they become widely traded.

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| `DataFrame is empty` | Interval and lookback exceed yfinance's limit | Lower `LOOKBACK_DAYS` (60 for `'5m'`, 729 for `'60m'`) |
| `AssertionError` on `INTERVAL_SECONDS` | Slot length is not a multiple of the base interval | Use a multiple of 300 (5m) or 3600 (60m) |
| `MultiIndex` errors in column access | Newer yfinance version returns grouped columns | The code already flattens columns; update yfinance if it persists |
| Sharpe cell fails | Cell 4 hasn't been run | Run Cells 1–4 first |
| Heatmap is unreadable | Too many sessions for the figure size | Adjust `figsize` or the `step` value in Cell 6 |
| Rate limit or connection error | Yahoo throttling | Wait a few minutes and rerun |

---

## Roadmap

- Add transaction costs and slippage to the backtest
- Add bootstrap confidence intervals for slot averages
- Support multiple tickers in one run and rank them
- Save heatmaps and tables to `outputs/`
- Add a vendor data adapter (Polygon or Alpaca) for longer intraday history
- Add a day-of-week breakdown alongside time of day

---

## License and Disclaimer

This project is for educational and research purposes. It is not financial advice, and nothing here is a recommendation to buy or sell any security. Past performance, especially from a short sample, does not predict future results. Verify data and conclusions independently before making any trading decisions.
