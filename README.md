# Chap Counterfactual Comparison

This tutorial shows how to use `chap-core`'s CLI to compare model predictions under original and counterfactual climate values. You will:

1. Build a **counterfactual dataset** by modifying a climate column in the original CSV.
2. Run **`chap causal`** to train a model, generate predictions for both datasets, and produce an interactive comparison plot.

> **Caveat:** The comparison reflects the model's response to different feature values, not necessarily a true causal effect. The difference between the two prediction curves is only as causal as the model itself — confounding by other features (included or excluded) can still be present. Treat the output as a model-based sensitivity analysis rather than a causal estimate.

## Workflow

```mermaid
flowchart TD
    A["data/vietnam_monthly.csv<br/>(original)"]
    A --> B["chap causal build-counterfactual<br/>'mean_temperature=x-30' 'rainfall=x*0.01'"]
    B --> C["data/vietnam_monthly_cf.csv<br/>(counterfactual)"]
    A --> D["chap causal --plot"]
    C --> D
    D --> E["Train model on original data<br/>up to split-period"]
    E --> F["Predict from split-period<br/>on both datasets"]
    F --> G["results/causal.html<br/>(comparison plot)"]
```

---

## Prerequisites — Installing chap-core

Follow the [chap contributor setup guide](https://chap.dhis2.org/chap-modeling-platform/contributor/chap-contributor-setup/) to install chap-core:

```bash
git clone https://github.com/dhis2-chap/chap-core.git
cd chap-core
# install uv (see https://docs.astral.sh/uv/getting-started/installation/)
curl -LsSf https://astral.sh/uv/install.sh | sh
uv sync
source .venv/bin/activate
chap --help   # verify the installation
cd ..
```

---

## Example Data

The `data/` directory contains the Vietnam monthly dengue dataset from chap-core:

| File | Description |
|------|-------------|
| `data/vietnam_monthly.csv` | Monthly dengue cases, rainfall, and temperature for Vietnamese provinces (1994 onwards) |
| `data/vietnam_monthly.geojson` | Geographic boundaries for Vietnamese provinces |

**CSV columns:**

| Column | Description |
|--------|-------------|
| `time_period` | Month in `YYYY-MM` format |
| `rainfall` | Monthly rainfall (mm) |
| `mean_temperature` | Mean temperature (°C) |
| `disease_cases` | Reported dengue cases |
| `population` | Province population |
| `location` | Province name |

---

## Step 1 — Build a Counterfactual Dataset

`chap causal build-counterfactual` creates a modified copy of your CSV by applying a mathematical expression to one or more columns. Use `x` to refer to the original cell value.

The following command simulates a **30°C reduction in mean temperature** and a **99% reduction in rainfall** across the entire time series:

```bash
chap causal build-counterfactual \
    data/vietnam_monthly.csv \
    "mean_temperature=x-30" \
    "rainfall=x*0.01" \
    --output-csv data/vietnam_monthly_cf.csv
```

This produces `data/vietnam_monthly_cf.csv` with every `mean_temperature` value reduced by 30°C and every `rainfall` value scaled to 1% of its original.

**Apply the transformation only from a specific month onward:**

```bash
chap causal build-counterfactual \
    data/vietnam_monthly.csv \
    "mean_temperature=x-30" \
    "rainfall=x*0.01" \
    --start-time-period 2009-01 \
    --output-csv data/vietnam_monthly_cf.csv
```

**Expression syntax:**

| Construct | Examples |
|-----------|---------|
| Original value | `"x"` |
| Arithmetic operators | `"x*0.5"`, `"x+10"`, `"x-5"`, `"x/2"`, `"x**2"` |
| Functions | `"abs(x)"`, `"round(x)"` |
| Combinations | `"abs(x*0.1-5)"`, `"round(x+0.5)"` |
| Multiple columns | `"rainfall=x*0.5" "mean_temperature=x-2"` |

Missing values (`NaN`) are preserved unchanged regardless of the expression.

---

## Step 2 — Generate a Comparison Plot

`chap causal` trains a model on the original dataset, then predicts from a chosen split period to the end of both the original and counterfactual datasets. With `--plot` it writes an interactive HTML comparison. `--cf-start-period 2009-08` tells chap to apply the counterfactual values from this time period. This is relevant for models that take into account previous time period climate feature values to make predictions.

```bash
mkdir -p results
chap causal \
    --model-name https://github.com/chap-models/chap_pymc \
    --dataset-csv data/vietnam_monthly.csv \
    --counterfactual-csv data/vietnam_monthly_cf.csv \
    --counterfactual-columns mean_temperature rainfall \
    --split-period 2010-07 \
    --cf-start-period 2009-08 \
    --output-file results/causal.nc \
    --plot
```

**What each argument does:**

| Argument | Description |
|----------|-------------|
| `--model-name` | Model to use — a GitHub URL or local path |
| `--dataset-csv` | The original (observed) dataset |
| `--counterfactual-csv` | The counterfactual dataset produced in Step 1 |
| `--counterfactual-columns` | Columns that differ between the two datasets (space-separated if more than one) |
| `--split-period` | Period that separates the training window (before) from the prediction window (from this period to the end of the dataset) |
| `--cf-start-period` | Period from which the counterfactual values are applied to the counterfactual prediction scenario |
| `--output-file` | Path for the NetCDF results file |
| `--plot` | Also write an HTML comparison plot alongside the NetCDF files |

The model is trained **once** on the original data up to (but not including) `2010-07`, then predictions are generated for both datasets from `2010-07` onwards (the last 6 months of the dataset).

---

## Output

After the command completes you will find:

```
results/
├── causal.nc       # predictions for the original dataset
├── causal_cf.nc    # predictions for the counterfactual dataset
└── causal.html     # interactive side-by-side comparison plot
```

Open `results/causal.html` in a browser to explore how the predicted disease burden differs between the original climate conditions and the counterfactual scenario.

![Causal analysis comparison plot showing original vs counterfactual predicted dengue cases side by side for Vietnamese provinces](data/results_screenshot.png)

The plot shows side-by-side panels for each location: **Original** (predictions under observed climate) and **Counterfactual** (predictions under the modified climate). Both panels use the same colour scheme — compare the prediction bands across the two panels to see the effect of the counterfactual scenario. Scroll down to see additional provinces.
