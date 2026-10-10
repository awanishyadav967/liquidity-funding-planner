# Liquidity Funding Planner

A Python and Streamlit application for exploring a treasury question: **will each currency account stay above its minimum cash buffer, and how should limited central funding cover any projected gaps?**

The app forecasts daily net cashflow for each selected currency, adds those forecasts to opening balances, identifies buffer breaches, converts the largest shortfall in each currency to EUR, and recommends funding amounts within a central budget. It also shows historical and forecast charts, optional backtest results, and an Excel download.

The default data and FX rates are illustrative. This is an analytical demonstration, not a live treasury system or a source of executable FX rates.

## What the dashboard does

1. **Load cashflows.** Use the built-in synthetic dataset or upload a CSV/XLSX file with a date column and one numeric column per currency.
2. **Forecast daily cashflows.** The default horizon is 30 days; the sidebar allows 7 to 90 days. Each currency gets its own SARIMAX model with weekly seasonality. If fitting fails or does not converge, the app uses a seven-day seasonal-naive forecast.
3. **Estimate uncertainty.** Forecast charts show a mean and a 95% interval. These are model-based estimates, so unexpected payments or market shocks can still fall outside them.
4. **Find liquidity gaps.** The app adds forecast cashflows to an editable opening balance, then compares each projected balance with its editable minimum buffer. A shortfall is the amount needed to restore that buffer.
5. **Recommend funding.** The largest projected shortfall for each currency is converted to EUR using editable example FX rates. A linear optimization allocates no more than the central funding available or the shortfall required for any currency. Transfer cost and priority weight influence which gaps receive funding first.
6. **Review and export.** Charts and tables show forecasts, balances, shortfalls, recommended amounts, remaining gaps, and estimated transfer costs. The **Run backtest** button compares the forecasting pipeline with a seasonal-naive baseline using MAE and MASE.

The built-in sample generates repeatable daily cashflows for **EUR, USD, JPY, BRL, INR, and AUD** using a fixed random seed. It includes recurring receipts, payroll, month-end payments, routine variation, and occasional shocks. No real financial data is included.

## Run locally

Clone this repository and install the application dependencies in a virtual environment:

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

Then install and start the dashboard:

```bash
python -m pip install -r requirements.txt
python -m streamlit run app.py
```

Open the local URL printed by Streamlit, normally `http://localhost:8501`. The first forecast may take some time because it fits a model for each selected currency.

For a command-line demonstration using the synthetic dataset and default assumptions, run `python main.py`. It prints projected minima, shortfalls, funding recommendations, and estimated transfer cost; writes `output.xlsx`; and opens a forecast plot.

## Upload your own cashflows

The first column must contain dates. Each other column represents daily **net** cashflow in that currency's local units. Positive amounts add cash; negative amounts use cash.

```csv
Date,EUR,USD,JPY
2025-01-01,1200,-450,85000
2025-01-02,-300,900,-42000
2025-01-03,750,125,63000
```

The example shows the layout; an upload needs **at least 60 rows**. Dates must be unique and sorted oldest to newest, and currency columns must be numeric. Missing values are forward-filled before validation, so a missing first value or an entirely empty currency column still causes an error. Daily observations are the intended input because the forecasting code creates daily future dates.

In the sidebar, select the currencies to include and adjust their opening balances, minimum buffers, and conversion rates. A rate labeled `1 USD in EUR`, for example, is the EUR value of one USD. Set the EUR amount available for central funding. Under **Optimization Settings**, edit transfer costs and priority weights. A higher priority weight generally favors that currency when funds are scarce.

## How to read the results

- **Forecast chart:** projected daily net cashflow in local currency, with a 95% interval.
- **Projected cash balance:** opening balance plus cumulative forecast net cashflows.
- **Maximum shortfall:** the largest amount by which a projected balance falls below its minimum buffer during the selected horizon.
- **Recommended funding:** the allocation in EUR and its local-currency equivalent.
- **Remaining shortfall:** projected need left after recommended funding.
- **Transfer cost:** estimated from recommended EUR funding and the chosen transfer cost rate.

The optimizer uses **one maximum shortfall per currency over the whole horizon**. It does not schedule transfers on specific days. Its objective combines priority and transfer cost, subject to nonnegative allocations, each currency's maximum shortfall, and the total central funding limit.

The optional backtest uses three chronological folds with a seven-day horizon. **MAE** measures average absolute error in the currency's units. **MASE** scales error against an in-sample seasonal-naive benchmark; the dashboard also compares the pipeline directly with a seasonal-naive forecast over the same folds.

## Excel output

The dashboard's **Download Excel Output** button creates `liquidity_forecast_output.xlsx` with these sheets: `Historical_Cashflows`, `Forecasts`, `Projected_Balances`, `Liquidity_Shortfalls`, and `Funding_Recommendations`.

The command-line workflow writes `output.xlsx` and also includes `Forecast_Lower_95` and `Forecast_Upper_95`. The dashboard currently displays forecast intervals in charts but does not pass them into its Excel export.

## Project layout

| Path | Purpose |
| --- | --- |
| `app.py` | Streamlit dashboard, inputs, charts, backtest button, and download |
| `main.py` | Command-line demonstration with synthetic data |
| `liquidity_forecasting/data.py` | Synthetic cashflows and missing-value preprocessing |
| `liquidity_forecasting/model.py` | SARIMAX forecasts, intervals, and fallback |
| `liquidity_forecasting/balances.py` | Projected balances and buffer shortfalls |
| `liquidity_forecasting/fx.py` | Conversion of local amounts to EUR |
| `liquidity_forecasting/allocation.py` | Funding allocation and optimization |
| `liquidity_forecasting/evaluation.py` | Backtesting and forecast accuracy metrics |
| `liquidity_forecasting/export.py` | Excel workbook generation |
| `tests/` | Automated checks for the calculation modules |

To run the project's checks, install the development requirements and use the same commands as CI:

```bash
python -m pip install -r requirements-dev.txt
ruff check .
ruff format --check .
python -m pytest
```

## Limits and attribution

The sample cashflows and FX rates are invented. Forecasts use each currency's past cashflows without incorporating known invoices, payment calendars, holidays, or macroeconomic information. Prediction intervals reflect model uncertainty, and funding recommendations are based on assumptions entered by the user. Review actual cash positions, governed FX rates, and operational constraints before using any analysis in a real treasury process.

The source code and MIT license attribute the original project to [Andorta](https://github.com/Andorta). This README describes the behavior of the code included here.
