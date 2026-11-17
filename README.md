# Case Study 1: Robinhood Traders and the Case of GameStop

MATH 60231, Financial Econometrics, HEC Montréal, Fall 2026 (individual work)

This project studies Robinhood (RH) retail traders from 2019 to early 2021, in three steps:

1. Build the portfolio of a representative RH investor from the RH holdings data and compare it with the market (CRSP value-weighted index).
2. Measure its risk with Value-at-Risk and expected shortfall.
3. Test whether a rolling AR(p) forecast could have predicted the GameStop (GME) short squeeze of January 2021.

The analysis is in the notebook; the discussion is in the report.

## Folder layout

```text
Case study 1/
├── README.md
├── requirements.txt          Python packages (pinned versions)
├── case_study_1.ipynb        the analysis: every table and figure in the report
├── Data/
│   ├── RH_data (1).csv       RH holdings: date, users_holding, tic
│   ├── stock_data (1).csv    daily CRSP data: Ticker, DlyCalDt, DlyPrc, DlyRet, DlyVol, vwretd
│   └── gme_yahoo.csv         GME adjusted close, 2019-01-01 to 2021-02-28
├── Output/                   tables and figures saved by the notebook (PNG)
├── docs/
│   └── The Case of Robinhood - fall2026.pdf   the assignment
└── report/
    └── Case study - Write-up.docx             the report
```

## Getting the data

The RH holdings file, the CRSP file and the assignment PDF are not in this repository: the CRSP data is licensed (WRDS) and the assignment belongs to the course. To rerun the notebook, place them here:

- `Data/RH_data (1).csv`: Robinhood holdings (Robintrack), as provided with the course.
- `Data/stock_data (1).csv`: CRSP daily stock file with the columns listed above (requires WRDS access).
- `docs/The Case of Robinhood - fall2026.pdf`: the assignment (course material, not needed to run the code).

`Data/gme_yahoo.csv` is included.

## How to run

Requires Python 3.11 or later.

```bash
python -m venv .venv
.venv\Scripts\activate          # Windows (macOS / Linux: source .venv/bin/activate)
pip install -r requirements.txt
```

Open `case_study_1.ipynb`, select the `.venv` kernel and run **Restart & Run All**.

- Run it from this folder, because the data paths are relative (`Data/`).
- The notebook makes no network calls. GME data was downloaded once from Yahoo Finance and saved to `Data/gme_yahoo.csv`.
- Every table and figure is written to `Output/` as a PNG, named after its number (for example `Table_2.1_...png`).

## Where each question is answered

| Brief | Notebook section | Report section | Tables / figures |
| --- | --- | --- | --- |
| T1 (a)–(f): RH portfolio vs the market | T1 | Technical Questions, T1 | Table 1.1, Figures 1.1–1.2 |
| T2 (a)–(b): VaR and ES, 5% and 1% | T2 | Technical Questions, T2 | Table 2.1 |
| T3 (a): rolling AR(p) forecast | T3 data, T3 (a) | Technical Questions, T3 | Figures 3.1–3.3, Table 3.1 |
| T3 (b): forecast accuracy | T3 (b) | Technical Questions, T3 | Table 3.2, Figure 3.4 |
| T3 (c): accuracy during the event | T3 (c) | Technical Questions, T3 | Table 3.3, Figure 3.5 |
| D1: performance | uses T1 | Discussion, D1 | Table 1.1, Figure 1.2 |
| D2: risk–reward | D2 | Discussion, D2 | Tables 1.1, 2.1, 4.1 |
| D3: the GameStop event | uses T3 | Discussion, D3 | Tables 3.2–3.3, Figures 3.2, 3.5 |
| D4: VAR framework (hypothetical, no code) | – | Discussion, D4 | – |

All tables and figures are in the report's appendix.

## Data

- **RH holdings** (`RH_data`): number of RH accounts holding each stock, each calendar day, 2019-01-01 to 2020-08-13 (575 days, 1,445 tickers).
- **CRSP** (`stock_data`): daily returns of the same stocks, 2019-01-02 to 2020-08-13 (408 trading days, 1,388 tickers), plus `vwretd`, the CRSP value-weighted market return.
- **GME** (`gme_yahoo.csv`): the CRSP file stops on 13 Aug 2020, before the event, so GME prices come from Yahoo Finance (adjusted close, so returns include dividends and splits). On the 407 days both sources cover, the returns are identical (correlation 1.0).

## Assumptions

### Cleaning (notebook section 1)

- Dates like `01jan2019` are parsed with the format `%d%b%Y`. Non-numeric CRSP codes become missing values.
- 19 exact duplicate rows in `stock_data` are dropped.
- TAP and STZ have two share classes under one ticker. Each day the class with the higher volume (the main, liquid class) is kept.
- 251 rows with no return (mostly a stock's first day, plus long gaps for WORX, LK and MBRX) are dropped, not set to 0: a missing return is unknown, not 0%.
- Holdings on weekends and holidays carry no return, so the sample is the 408 trading days.

### Portfolio construction (T1)

- **Weights** are popularity weights: `users_holding` counts accounts, not shares or dollars.
- **No look-ahead:** the holdings of day *t* already react to day *t*'s returns, so the return of day *t* uses the holdings of the previous day:

  $$
  r_{RH,t} = \sum_i w_{i,t-1} \, r_{i,t}
  $$

  $$
  w_{i,t-1} = \frac{\text{users\_holding}_{i,t-1}}{\displaystyle\sum_{j \,\in\, \text{has a return at } t} \text{users\_holding}_{j,t-1}}
  $$

- **Missing tickers:** 57 RH tickers (about 2% of holdings: foreign ADRs, tickers with a dot such as BRK.B, leveraged ETNs) have no CRSP data. They are dropped and the weights re-normalized each day, so the portfolio is fully invested. Dropping the leveraged ETNs probably understates RH risk slightly.
- **Gaps in the RH data** (24–29 Jan 2019, 7–15 Jan 2020, 3 Mar 2019): on the 11 trading days affected, the last recorded holdings are carried forward (up to 9 days old).
- The first trading day (2019-01-02) uses the 2019-01-01 holdings, so no day is lost.
- The portfolio rebalances daily, with no trading costs or taxes.
- **Benchmark:** CRSP VW (`vwretd`), the value-weighted return of all NYSE, AMEX and NASDAQ stocks with dividends, used in place of the S&P 500. Correlation with RH: 0.881.

### Risk measures (T2)

- VaR and ES are reported as **positive losses** (VaR = 0.03 means "lose at least 3% in one day").
- **Empirical (historical)**, the main method, makes no assumption about the distribution:

  $$
  \text{VaR}_\alpha = -\,q_\alpha(r)
  $$

  $$
  \text{ES}_\alpha = -\,\mathbb{E}\left[\, r \mid r \le q_\alpha(r) \,\right]
  $$

- **Normal (parametric)**, for comparison only:

  $$
  \text{VaR}_\alpha = -\left(\mu + \sigma \, z_\alpha\right)
  $$

  $$
  \text{ES}_\alpha = -\left(\mu - \sigma \, \frac{\varphi(z_\alpha)}{\alpha}\right)
  $$

- At 1%, the empirical figures rest on about 4 days, mostly the March 2020 crash.

### GME forecast (T3)

- **Lag order chosen on pre-event data only** (to 30 Nov 2020). AIC and BIC both select 0 lags; the PACF shows nothing beyond chance (lag 9 is treated as noise).
- **AR(1)** is used, because AR(0) would equal the historical-mean benchmark.
- **Rolling window of 250 trading days** (about one year), one-step-ahead: the forecast for day *t* uses days *t*−250 to *t*−1 only.
- **Benchmark:** the historical mean over the same rolling window.

  $$
  R^2_{OS} = 1 - \frac{\text{MSE}_{AR}}{\text{MSE}_{mean}}
  $$

- **Diebold–Mariano test** on squared errors, with the Harvey–Leybourne–Newbold small-sample correction. The two models are nested, but with a fixed rolling window the test compares forecasting methods and stays valid (Giacomini–White).
- **Event window:** 1 Dec 2020 to 26 Feb 2021 (60 forecasts), against 31 Dec 2019 to 30 Nov 2020 outside (232 forecasts).

### Risk–reward (D2)

- **Sharpe ratio** = mean / standard deviation, with no risk-free rate, annualized with √252.
- **Tail-adjusted ratio** = mean daily return / empirical 5% ES.

## Key results

| | RH portfolio | CRSP VW |
| --- | --- | --- |
| Annualized mean | 35.1% | 22.6% |
| Annualized volatility | 35.3% | 26.9% |
| Cumulative return | +59.3% | +36.0% |
| Skewness / excess kurtosis | −0.71 / 7.7 | −0.96 / 14.0 |
| Worst day | −11.9% | −11.8% |
| VaR / ES 5% (empirical) | 3.0% / 5.4% | 2.4% / 4.4% |
| VaR / ES 1% (empirical) | 8.2% / 10.4% | 5.8% / 8.2% |
| Sharpe ratio (annualized) | 0.99 | 0.84 |
| Mean / ES 5% | 0.026 | 0.020 |

- RH earns more than the market for proportionally more risk; its tails look like the market's, scaled up.
- The normal model understates the 1% risk: RH VaR 5.0% (normal) vs 8.2% (empirical).
- **GME:** the AR(1) does slightly worse than the historical mean (RMSE 0.168 vs 0.158, R²_OS = −0.13), but the difference is not significant (DM p = 0.44). During the event its RMSE is 4.6 times larger than outside it, and it reacts one day late: on 28 Jan 2021 it forecasts +73% when GME falls 44%.

## References

The full list is in the report's bibliography. The main sources:

- Welch, I. (2022). The wisdom of the Robinhood crowd. *The Journal of Finance, 77*(3), 1489–1527.
- Barber, B. M., Huang, X., Odean, T., & Schwarz, C. (2022). Attention-induced trading and returns: Evidence from Robinhood users. *The Journal of Finance, 77*(6), 3141–3190.
