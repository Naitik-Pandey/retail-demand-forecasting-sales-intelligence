# Retail Demand Forecasting & Sales Intelligence System

An end-to-end retail forecasting and business intelligence project built using Python, machine learning, PostgreSQL, SQL, and Power BI.

The project forecasts future weekly sales at the **Store × Department × Week** level and converts model outputs into an analytical system for evaluating demand patterns, store performance, department performance, forecast coverage, and model quality.

## Project Overview

Retail demand forecasting requires more than training a machine-learning model. A useful forecasting system must also address:

* time-series leakage
* seasonality
* missing data
* store and department heterogeneity
* holiday demand
* model validation across future periods
* operational handling of unusual predictions
* analytical storage
* business-facing visualization

This project implements the complete workflow from raw Walmart sales data to an interactive Power BI dashboard.

## Business Problem

The objective is to forecast future weekly sales for individual store-department combinations and provide decision-makers with a business intelligence layer for understanding:

* expected future sales
* changes relative to comparable prior-year weeks
* high-demand stores
* high-demand departments
* holiday and seasonal demand patterns
* model forecast coverage
* fallback prediction usage
* model validation performance

The project focuses on **demand forecasting and sales intelligence**.

It does not claim true inventory optimization because the source dataset does not contain inventory levels, stock-outs, replenishment quantities, lead times, or holding costs.

## Dataset

The project uses the Walmart Store Sales Forecasting dataset.

Four source tables are used:

| Dataset  |    Rows | Grain                     |
| -------- | ------: | ------------------------- |
| Train    | 421,570 | Store × Department × Week |
| Test     | 115,064 | Store × Department × Week |
| Features |   8,190 | Store × Week              |
| Stores   |      45 | Store                     |

The historical training period runs from February 2010 through October 2012.

The forecast/test period runs from November 2012 through July 2013.

### Primary Target

`Weekly_Sales`

### Important Features

* Store
* Department
* Store Type
* Store Size
* Temperature
* Fuel Price
* CPI
* Unemployment
* MarkDown1–5
* Holiday indicator
* historical annual sales lag

## Data Architecture

```text
Raw Walmart CSV Data
        |
        v
Python / Pandas
        |
        v
Data Quality + EDA
        |
        v
Feature Engineering
        |
        v
Time-Based Validation
        |
        v
Seasonal Baseline
        |
        v
Machine Learning Models
        |
        v
Rolling Out-of-Time Validation
        |
        v
Final Random Forest
        |
        v
Future Forecast Generation
        |
        v
Operational Forecast Layer
        |
        v
PostgreSQL Star Schema
        |
        v
SQL Analytical Layer
        |
        v
Power BI Dashboard
```

## Data Quality and Preparation

The project explicitly checks:

* exact duplicates
* logical-key duplicates
* missing values
* inconsistent holiday flags
* negative historical sales
* zero sales
* date ranges
* store metadata completeness
* feature availability across historical and future periods

No exact duplicate rows or logical Store-Department-Date duplicates were found in the historical sales data.

Approximately 0.30% of historical sales observations were negative.

These values were retained because negative retail sales can represent returns, refunds, corrections, or accounting adjustments. They were not automatically treated as data errors.

## Missing Data Strategy

The MarkDown variables contain substantial missingness.

Missing markdown values were not assumed to mean known zero.

For modeling:

1. a missing-value indicator was generated
2. the numerical markdown value was filled with zero
3. the model therefore received both the filled value and information indicating that the original value was missing

CPI and unemployment also contain missing observations in the future portion of the dataset.

## Exploratory Data Analysis

Key historical observations included:

* Holiday weeks showed higher average Store-Department sales than non-holiday weeks.
* Type A stores generated substantially higher average sales than Types B and C.
* Store size showed a strong positive association with average store-level weekly sales.
* Late-November and December periods produced major seasonal demand peaks.
* Thanksgiving weeks were among the highest-sales weeks in the historical dataset.
* Demand varied substantially across departments.

These relationships are interpreted as associations rather than causal effects.

## Leakage-Safe Feature Engineering

A major design requirement was preventing future information from leaking into model training.

### Exact 52-Week Lag

A historical annual lag was constructed using:

```text
Current Date - 364 Days
```

and matching on:

```text
Store + Department + Date
```

This was preferred over a simple `shift(52)` because 52 observations are not guaranteed to correspond to exactly 52 calendar weeks when observations are missing.

The resulting feature:

```text
Lag_364d
```

became the most important predictor in the final model.

## Validation Strategy

Random train-test splitting was avoided because this is a forecasting problem.

Using random splitting would allow the model to learn from observations that occur chronologically after observations in the validation set.

Instead, the project uses:

1. chronological development validation
2. exact 52-week seasonal baseline
3. rolling out-of-time validation

This more closely simulates real forecasting deployment.

## Seasonal Baseline

The primary baseline predicts sales using the exact Store-Department sales observation from 364 days earlier.

When an exact prior-year observation is unavailable, a historical Store-Department median is used, followed by a global historical median if necessary.

The baseline provides a realistic benchmark that the machine-learning models must outperform.

## Evaluation Metric

The primary evaluation metric is **Weighted Mean Absolute Error (WMAE)**.

Holiday observations receive a weight of 5 and regular weeks receive a weight of 1.

This reflects the greater business importance of forecasting important holiday periods accurately.

## Models Evaluated

Several modeling approaches were tested:

* Seasonal baseline
* Ridge Regression
* Decision Tree
* Random Forest
* XGBoost

Ridge Regression performed poorly because a single global linear relationship struggled with the highly heterogeneous and nonlinear Store-Department demand patterns.

Decision Trees improved on Ridge but did not consistently outperform the seasonal benchmark.

Random Forest produced the strongest and most stable performance.

XGBoost was evaluated as a challenger but did not outperform the final Random Forest configuration.

## Final Model

The selected model is a tuned Random Forest using:

* Store
* Department
* Store Type
* Store Size
* Temperature
* Fuel Price
* CPI
* Unemployment
* MarkDown variables
* MarkDown missingness indicators
* exact 364-day sales lag

Holiday status is used in observation weighting during training and evaluation.

The selected Random Forest configuration uses approximately:

```text
n_estimators = 100
max_depth = 18
min_samples_leaf = 10
max_features = 0.5
bootstrap = True
random_state = 42
```

## Rolling Validation Results

The final model was evaluated across three chronological out-of-time validation windows.

| Validation Fold | Seasonal Baseline WMAE | Random Forest WMAE | Improvement |
| --------------- | ---------------------: | -----------------: | ----------: |
| Fold 1          |                1858.36 |            1674.75 |       9.88% |
| Fold 2          |                1833.64 |            1689.90 |       7.84% |
| Fold 3          |                1736.12 |            1560.97 |      10.09% |

### Overall Result

**Mean WMAE improvement: 9.27%**

Mean Seasonal Baseline WMAE:

```text
1809.37
```

Mean Random Forest WMAE:

```text
1641.87
```

This rolling validation result is the primary model-performance result used in the project.

## Feature Importance and Ablation Analysis

The exact annual sales lag was by far the strongest feature.

Important predictors also included:

* Department
* Store Size
* Store metadata
* macroeconomic variables
* MarkDown variables

Removing `Lag_364d` caused model performance to deteriorate dramatically, confirming that annual sales seasonality is fundamental to this forecasting problem.

The final model can therefore be interpreted as learning nonlinear corrections around a strong prior-year demand signal.

## Future Forecast Generation

The final model was trained using all historical observations with a valid exact annual lag.

Future predictions were generated for:

**115,064 Store-Department-Week observations**

Prediction routing:

| Prediction Method          |    Rows | Coverage |
| -------------------------- | ------: | -------: |
| Random Forest              | 113,023 |   98.23% |
| Historical Median Fallback |   2,041 |    1.77% |

No future target labels are available, so these predictions are not described as test-set accuracy.

## Operational Forecast Layer

The raw model outputs are preserved separately from business-operational forecasts.

A small number of fallback forecasts were negative:

```text
78 rows
```

Because negative demand is not operationally meaningful for planning, a second field was created:

```text
Operational_Forecast = max(Raw_Forecast, 0)
```

The raw prediction remains available for auditing.

This separation avoids silently modifying model output.

## PostgreSQL Analytical Layer

Forecast and historical data were loaded into PostgreSQL using a star-schema-inspired design.

### Dimension Tables

* `dim_store`
* `dim_department`
* `dim_date`

### Fact Tables

* `fact_sales`
* `fact_forecast`
* `fact_features`

Primary and foreign-key constraints preserve data integrity.

Indexes were created for commonly queried date and department fields.

## SQL Analytics

SQL analysis includes:

* store performance ranking
* department demand ranking
* store-type comparison
* weekly and monthly historical patterns
* prior-year comparable analysis
* future store rankings
* future department rankings
* forecast versus exact 52-week comparable sales
* analytical views for Power BI

Example analytical views include:

```text
vw_weekly_forecast_comparison
vw_store_forecast_comparison
```

## Forecast Business Outlook

Across the complete 39-week future horizon:

```text
Forecast Sales                 ≈ 1.927B
Prior-Year Comparable Sales    ≈ 1.896B
Forecast Difference            ≈ +30.48M
Forecast Difference %          ≈ +1.61%
```

37 of the 39 forecast weeks were above their corresponding exact 52-week historical comparison.

These figures represent **forecast versus prior-year comparable sales**, not realized future growth.

## Power BI Dashboard

The final Power BI report contains four analytical pages.

### 1. Executive Overview

Provides:

* Forecast Sales
* Prior-Year Comparable Sales
* Forecast Change
* Forecast Change %
* Stores Covered
* weekly forecast comparison
* Top 10 stores
* Top 10 departments
* interactive slicers

### 2. Store & Department Performance

Provides:

* store rankings
* department rankings
* store-type analysis
* store size versus forecast demand
* forecast change by store
* department demand concentration

### 3. Forecast & Model Quality

Provides:

* rolling validation results
* Seasonal Baseline WMAE
* Random Forest WMAE
* 9.27% mean WMAE improvement
* Random Forest forecast coverage
* fallback coverage
* operational forecast adjustment monitoring

### 4. Store Detail

Provides drill-through analysis for individual stores, including:

* future sales
* comparable historical sales
* weekly demand
* department contribution
* forecast change

## Dashboard Screenshots

### Executive Overview

![Executive Overview](images/executive_overview.png)

### Store & Department Performance

![Store and Department Performance](images/store_department_performance.png)

### Forecast & Model Quality

![Forecast Model Quality](images/forecast_model_quality.png)

### Store Drill-Through

![Store Drillthrough](images/store_drillthrough.png)

## Repository Structure

```text
retail-demand-forecasting-sales-intelligence/
|
├── README.md
├── requirements.txt
├── .gitignore
|
├── notebooks/
│   └── retail_demand_forecasting.ipynb
|
├── sql/
│   ├── schema.sql
│   ├── analytical_queries.sql
│   └── views.sql
|
├── powerbi/
│   └── Retail_Demand_Forecasting_Sales_Intelligence.pbix
|
├── images/
│   ├── executive_overview.png
│   ├── store_department_performance.png
│   ├── forecast_model_quality.png
│   └── store_drillthrough.png
|
└── outputs/
    └── sample_forecasts.csv
```

## Technology Stack

### Data Analysis

* Python
* Pandas
* NumPy

### Machine Learning

* Scikit-learn
* XGBoost

### Database and Analytics

* PostgreSQL
* SQL

### Business Intelligence

* Power BI
* DAX
* Power Query

### Modeling Concepts

* time-series validation
* lag features
* leakage prevention
* regression
* ensemble learning
* feature importance
* ablation analysis
* weighted error metrics
* star-schema modeling

## Key Technical Lessons

### Grain determines architecture

Historical sales use a Store-Department-Week grain, while external features use a Store-Week grain.

Understanding these grains was necessary for correct joins, aggregation, SQL schema design, and Power BI relationships.

### Missing does not mean zero

Markdown missingness was retained explicitly through missing-value indicators.

### Outlier does not automatically mean error

Large holiday sales observations and negative historical sales were investigated rather than blindly removed.

### Time-series validation matters

Random splitting would have produced an unrealistic forecasting evaluation.

### Baselines matter

A sophisticated machine-learning model is only valuable if it improves upon a credible seasonal forecasting baseline.

### Test predictions are not test accuracy

Future test labels are unavailable, so forecast quality claims are based on rolling historical validation rather than unlabeled future predictions.

## Limitations

The project has several important limitations:

1. Future ground-truth sales are unavailable.
2. The dataset does not contain inventory levels.
3. The project therefore does not claim true inventory optimization.
4. External events not represented in the historical features cannot be anticipated reliably.
5. The model relies heavily on prior-year demand and is therefore less suitable for entirely new Store-Department combinations.
6. Feature importance is predictive rather than causal.
7. Operational forecasts floor negative predictions to zero, while raw predictions are retained separately.

## Future Improvements

Potential extensions include:

* probabilistic forecasting and prediction intervals
* hierarchical forecasting reconciliation
* LightGBM or CatBoost benchmarking
* department-specific models
* SHAP-based explainability
* automated model monitoring
* scheduled PostgreSQL data refresh
* Power BI Service deployment
* inventory optimization using a dataset containing actual inventory and replenishment variables

## Project Result

The project demonstrates a complete data-science and analytics workflow:

**421K+ historical observations → leakage-safe forecasting → rolling validation → 115K+ future predictions → PostgreSQL analytical layer → interactive Power BI dashboard.**

The selected Random Forest reduced **mean WMAE by 9.27% compared with an exact 52-week seasonal baseline across three rolling out-of-time validation periods**, while directly generating **98.23% of future predictions**.
