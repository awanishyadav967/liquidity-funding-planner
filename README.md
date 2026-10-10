# Liquidity Funding Planner

Liquidity Funding Planner is a Python app for estimating future cash balances across currencies and deciding where a limited central funding pool may be needed. It answers two practical questions: **which currency accounts could fall below their minimum cash buffer, and how much funding should each receive?**

The project has a Streamlit dashboard for exploring assumptions and a command-line demonstration. Its default cashflow data is generated in code for EUR, USD, JPY, BRL, INR, and AUD. The sample data and exchange rates are illustrative, not live financial data.

## How it works

1. **Read cashflows.** Use the generated sample or upload a CSV/XLSX file of daily net cashflows. Positive values bring cash in; negative values use cash.
2. **Forecast each currency.** The app fits a daily SARIMAX model with weekly seasonality. If fitting fails or does not converge, it repeats the latest seven-day pattern as a fallback. Forecast charts include a mean and a 95% interval.
3. **Project balances.** For each future day, it adds forecast net cashflow to the previous balance, starting from an opening balance you can edit.
4. **Find buffer breaches.** A shortfall is the amount by which a projected balance falls below the minimum cash buffer. The planner takes the largest shortfall for each currency over the selected horizon.
5. **Allocate central funding.** It converts those shortfalls to EUR using editable example FX rates. An optimizer recommends amounts within the available EUR funding pool, each currency's shortfall, transfer costs, and priority weights.
6. **Review results.** The dashboard shows forecast plots, projected balances, funding tables, remaining shortfalls, and an Excel download. An optional backtest compares forecast errors with a seasonal-naive baseline.

This produces a **scenario-based funding recommendation**. It does not schedule transfers by day or connect to bank accounts.

## Run the dashboard

Install Python, then use a terminal from this repository:

```bash
git clone https://github.com/awanishyadav967/liquidity-funding-planner.git
cd liquidity-funding-planner
python -m venv .venv
```

Activate the environment:

```powershell
# Windows PowerShell
.venv\Scripts\Activate.ps1
```

```bash
# macOS or Linux
source .venv/bin/activate
```

Install the dependencies and launch Streamlit:

```bash
python -m pip install -r requirements.txt
python -m streamlit run app.py
```

Open the local address printed in the terminal, usually `http://localhost:8501`. The first run can take time because a forecasting model is fitted for every selected currency.

## Use the dashboard

The sidebar lets you switch between **simulated sample data** and a file upload, select currencies, and choose a forecast horizon from 7 to 90 days. It also lets you set opening balances, minimum cash buffers, FX rates to EUR, total central funding, transfer costs, and priority weights.

A higher priority weight favors that currency when funding is scarce. A rate shown as `1 USD in EUR` means the EUR value of one USD. After changing inputs, review the projected balance and funding tables to see which accounts breach their buffer and which gaps remain unfunded.

Choose **Run backtest** to evaluate one currency using three chronological folds and a seven-day forecast. The table shows mean absolute error (MAE) and mean absolute scaled error (MASE) for the forecast pipeline and seasonal-naive comparison.

## Upload format

Put dates in the first column and one currency per remaining column. Values are daily **net** cashflows in each currency's own units:

```csv
Date,EUR,USD,JPY
2025-01-01,1200,-450,85000
2025-01-02,-300,900,-42000
2025-01-03,750,125,63000
```

The example shows the structure; an actual upload needs **at least 60 observations**. Dates must be unique and in ascending order, and currency columns must be numeric. Missing values are forward-filled before validation, but a missing first value or an entirely empty column cannot be filled. Daily observations are intended because the forecast uses daily future dates.

## Results and downloads

The **projected balance** is the opening balance plus cumulative forecast net cashflows. The **maximum shortfall** is the largest gap below the chosen buffer over the forecast horizon. Funding is recommended in EUR and also displayed in the local currency, alongside remaining unfunded amounts and estimated transfer costs.

The dashboard downloads `liquidity_forecast_output.xlsx` with historical cashflows, forecasts, projected balances, liquidity shortfalls, and funding recommendations. Forecast intervals are shown on screen; the dashboard export does not currently include interval sheets.

For a terminal-based example using the generated data and default assumptions, run:

```bash
python main.py
```

This prints summary results, writes `output.xlsx`, and opens a forecast plot. The command-line workbook also includes lower and upper forecast interval sheets.

## Code layout

| File or folder | Purpose |
| --- | --- |
| `app.py` | Streamlit dashboard and user controls |
| `main.py` | Command-line example |
| `liquidity_forecasting/data.py` | Synthetic cashflows and preprocessing |
| `liquidity_forecasting/model.py` | SARIMAX forecasts and fallback |
| `liquidity_forecasting/balances.py` | Balance projections and buffer gaps |
| `liquidity_forecasting/fx.py` | Currency conversion to EUR |
| `liquidity_forecasting/allocation.py` | Funding optimization |
| `liquidity_forecasting/evaluation.py` | Backtesting and error measures |
| `liquidity_forecasting/export.py` | Excel reports |
| `tests/` | Automated checks |

To run the checks used by the included CI workflow:

```bash
python -m pip install -r requirements-dev.txt
ruff check .
ruff format --check .
python -m pytest
```

## Limitations

The sample cashflows and FX rates are invented. Forecasts use each currency's historical cashflows without known invoices, holidays, or payment calendars. Prediction intervals cannot capture every operational shock. Recommendations use the maximum shortfall per currency rather than a dated transfer schedule. Check real cash positions, approved FX rates, and treasury constraints before using any result for an actual funding decision.
