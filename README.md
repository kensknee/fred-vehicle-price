# Forecasting Monthly U.S. New Vehicle Price Changes

A time-series forecasting project comparing ordinary least squares, ridge regression, and gradient boosting using more than 50 years of economic data from FRED.

## Overview

Can macroeconomic indicators improve forecasts of next month's change in U.S. new vehicle prices? Do indicators for recessions and COVID add predictive value?

This project forecasts the monthly percentage change in the new vehicle Consumer Price Index (CPI), rather than the dollar price of an individual vehicle. It combines macroeconomic indicators with recent price momentum and evaluates models against simple forecasting baselines.

The main finding: recent price momentum provides most of the predictive signal. Gradient boosting outperforms a historical-mean baseline, but its performance is close to a baseline that simply predicts last month's change.

## Key Results

The full-feature gradient boosting model achieved:

- Cross-validation MAE of 0.2450 percentage points and RMSE of 0.3125.
- Holdout MAE of 0.1371 percentage points and RMSE of 0.1725 for January 2023–December 2024.
- Lower MAE than the historical-mean baseline in all six validation folds.

The comparison with the last-month baseline is more nuanced: gradient boosting has slightly worse cross-validation MAE but better cross-validation RMSE. Its holdout MAE is slightly lower, though the difference is small.

### Cross-Validation Performance

| Model | MAE | RMSE |
|---|---:|---:|
| Historical-mean baseline | 0.3803 | — |
| Last-month baseline | 0.2389 | 0.3402 |
| Ordinary least squares | 0.2670 | 0.3280 |
| Ridge regression | 0.2712 | 0.3285 |
| Gradient boosting | 0.2450 | 0.3125 |

Metrics are measured in percentage points and averaged across six validation folds. Lower values indicate better performance.

### Holdout Performance

| Model | MAE |
|---|---:|
| Historical-mean baseline | 0.2229 |
| Last-month baseline | 0.1429 |
| Gradient boosting | 0.1371 |

Gradient boosting's holdout RMSE was 0.1725 percentage points.

## What the Analysis Shows

### Price momentum drives most of the forecastability

An OLS model using only the previous month's new vehicle CPI change achieved a cross-validation MAE of 0.2654, compared with 0.2670 for OLS using all predictors.

A lag-only gradient boosting model also performed slightly better than the full-feature GBM in cross-validation:

| Gradient Boosting Features | MAE | RMSE |
|---|---:|---:|
| Previous month's new vehicle change only | 0.2424 | 0.3066 |
| All predictors | 0.2450 | 0.3125 |

These results suggest that the additional macroeconomic variables provide limited incremental forecasting value in this setup.

### Statistical significance does not guarantee better forecasts

In OLS with heteroskedasticity- and autocorrelation-robust standard errors, lagged new vehicle price changes, headline inflation, and used vehicle price changes were statistically significant.

However, adding macroeconomic predictors did not materially improve cross-validation performance over the lag-only models.

### Event indicators do not improve out-of-sample performance

Adding indicators for the 1981–1982 recession, the 2007–2009 recession, and COVID did not improve OLS forecasting performance. Cross-validation MAE ranged from 0.2670 without event indicators to 0.2680 with all three.

## Data

Source: [Federal Reserve Economic Data (FRED)](https://fred.stlouisfed.org/)

The monthly sample covers May 1974–December 2024, with 608 observations before transformations and lagging.

| Variable | FRED Series ID | Role |
|---|---|---|
| New vehicle CPI | `CUSR0000SETA01` | Forecast target and lagged predictor |
| Used vehicle CPI | `CUSR0000SETA02` | Predictor |
| Real GDP per capita | `A939RX0Q048SBEA` | Quarterly predictor |
| Employment-population ratio | `EMRATIO` | Predictor |
| Real disposable personal income per capita | `A229RX0` | Predictor |
| All-items CPI | `CPIAUCSL` | Predictor |

### Feature Preparation

- Convert price and income series to percentage changes.
- Use the monthly percentage-point change in the employment-population ratio.
- Shift quarterly GDP growth by one quarter and forward-fill it to monthly dates.
- Lag the macroeconomic features by one month.
- Include the previous month's new vehicle CPI percentage change.
- Test recession and COVID indicators separately.

The notebook also compares raw levels, log levels, and percentage changes. Percentage changes performed substantially better in the tested specifications.

## Methodology

### Models

- Ordinary least squares: an interpretable linear benchmark.
- Ridge regression: a regularized linear model with standardized predictors.
- Gradient boosting: a nonlinear model using shallow decision trees.

OLS is also used for coefficient interpretation with HAC-robust standard errors.

### Validation

The project uses expanding-window, rolling-origin cross-validation:

1. Train on observations through each forecast origin.
2. Evaluate one-month-ahead predictions over the following 12 months.
3. Repeat for origins in December 2000, 2005, 2010, 2015, 2020, and 2021.
4. Average each fold's MAE and RMSE.

Development data ends in December 2022. The final evaluation period covers January 2023–December 2024.

### Baselines

- Historical mean: predict the average monthly change in the training data.
- Last month: predict that the next monthly change will equal the previous month's change.

These baselines help assess whether model complexity provides meaningful forecasting value.

## Repository Structure

```text
fred-vehicle-price/
├── README.md
├── requirements.txt
├── .gitignore
└── notebooks/
    ├── 02_data.ipynb
    ├── raw_fred_snapshot.csv
    └── results.png
```

- `02_data.ipynb`: complete analysis, explanations, model comparisons, and saved outputs.
- `raw_fred_snapshot.csv`: saved snapshot of the downloaded FRED series.
- `results.png`: saved results visualization.
- `requirements.txt`: Python dependencies.

## How to Run

### 1. Clone the repository

```bash
git clone [https://github.com/kensknee/fred-vehicle-price.git](https://github.com/kensknee/fred-vehicle-price.git)
cd fred-vehicle-price
```

### 2. Create and activate a virtual environment

```bash
python -m venv .venv
```

Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

macOS or Linux:

```bash
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure a FRED API key

Obtain a key from the [FRED API website](https://fred.stlouisfed.org/docs/api/api_key.html).

Create a `.env` file in the repository root:

```env
FRED_API_KEY=your_fred_api_key_here
```

Keep the key private and do not commit the `.env` file.

### 5. Launch the notebook

Launch Jupyter from the notebook directory so the relative paths match the code:

```bash
cd notebooks
jupyter lab 02_data.ipynb
```

Run the cells from top to bottom.

The current notebook downloads data directly from FRED and writes a new CSV snapshot. It does not automatically load the included snapshot as an offline fallback.

## Limitations

- The final evaluation period contains only 24 months, so small performance differences should be interpreted cautiously.
- The holdout was viewed during earlier model iterations and is not a completely untouched final test set.
- Validation folds share much of their training history and should not be treated as independent experiments.
- The percentage-change target has a borderline ADF stationarity result, with a p-value of approximately 0.066.
- Gradient boosting hyperparameters were fixed rather than extensively tuned.
- The analysis uses downloaded historical data rather than a real-time vintage dataset. Lagging observations does not fully account for publication delays or later revisions.
- Forecasts are one month ahead; the project does not evaluate long-horizon vehicle prices.
- Results describe predictive relationships, not causal effects.

## Tools

Python, pandas, NumPy, scikit-learn, statsmodels, matplotlib, fredapi, python-dotenv, and Jupyter.

## Author

Kenneth Zhang
