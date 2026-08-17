# Thesis Forecasting Project

Reproducible repository for the thesis data pipeline and short-term German electricity load forecasting experiments.

## Repository structure

```text
thesis-forecasting-project/
├── README.md
├── requirements.txt
├── .gitignore
├── notebooks/
│   ├── 01_weather_data_collection.ipynb
│   ├── 02_data_preprocessing.ipynb
│   └── 03_forecasting_experiments.ipynb
├── data/
│   ├── README.md
│   ├── raw/
│   │   ├── load/
│   │   └── weather/
│   └── processed/
├── results/
│   └── README.md
└── figures/
    └── README.md
```

## Workflow

Run the notebooks in numerical order.

### 1. Weather data collection

`notebooks/01_weather_data_collection.ipynb`

- Downloads hourly historical weather from the Open-Meteo Historical Weather API using ERA5-Land.
- Period: 2015-01-01 through 2025-12-31.
- Locations: Hamburg, Berlin, Cologne, Frankfurt and Munich.
- Final covariates: `temperature_mean` and `humidity_mean`, calculated as an equal-weighted mean across the five cities.
- Uses UTC as the canonical weather timestamp.
- Writes the final weather dataset to:
  `data/processed/weather_germany_hourly_2015_2025.csv`.

### 2. Data preprocessing

`notebooks/02_data_preprocessing.ipynb`

The German electricity-load data must first be downloaded from the **SMARD.de Download Center** (Bundesnetzagentur):

[SMARD.de – Download market data](https://www.smard.de/en/downloadcenter/download-market-data/?downloadAttributes=%7B%22superCategoryId%22:2,%22subcategoryId%22:5,%22regionId%22:%22DE%22,%22resolution%22:%22hour%22,%22fileType%22:%22CSV%22,%22from%22:1420066800000,%22to%22:1580511599999%7D)

For this project, the electricity series was downloaded as hourly CSV data for Germany (`DE`) and stored as two source files covering 2015–2024 and 2025. Place both files in `data/raw/load/` before running the preprocessing notebook:

- `Actual_consumption_201501010000_202501010000_Hour.csv`
- `Actual_consumption_202501010000_202601010000_Hour.csv`

These files are **not included in the Git repository**. They should be obtained directly from SMARD so the raw-data provenance is clear and the repository does not redistribute the source dataset.

The notebook:

- parses German local load timestamps and converts them to UTC;
- validates the hourly electricity and weather timelines;
- merges load and weather one-to-one on UTC timestamps;
- derives calendar variables in `Europe/Berlin` local time;
- creates the target variable `grid_load`;
- creates the chronological experiment split.

Split definition:

- **Train:** 2015–2023
- **Validation:** 2024
- **Test:** 2025

Outputs:

- `data/processed/final_combined.csv`
- `data/processed/train.csv`
- `data/processed/validation.csv`
- `data/processed/test.csv`

No global scaling is performed in this notebook. Model-specific scaling is fitted only on the training period in the forecasting notebook.

### 3. Forecasting experiments, results and figures

`notebooks/03_forecasting_experiments.ipynb`

This notebook is the repository version of `thesis-v3-2.ipynb`. Its forecasting methodology and model logic are preserved, while file paths have been made repository-relative.

Models:

- Seasonal Naive
- Ridge Regression
- XGBoost
- LSTM
- Temporal Fusion Transformer (TFT)

Core protocol:

- 24-hour forecast horizon;
- one forecast origin per local calendar day at 00:00 `Europe/Berlin`;
- common forecast origins and target timestamps across models;
- historical load/weather predictors must be strictly earlier than the forecast origin;
- validation is used for model/configuration selection;
- 2025 is reserved for final test evaluation;
- stochastic models use seeds `42`, `123`, and `777`;
- frozen neural lookbacks are LSTM `168 h` and TFT `336 h` unless the optional lookback study is rerun.

Generated experiment artifacts are written under:

`results/thesis_outputs_v3/`

This includes metrics, predictions, configurations, logs, checkpoints and model files. Thesis-ready PNG plots are written directly to:

`figures/`

The model notebook also retains Kaggle compatibility: if the processed CSVs are not present locally, it searches `/kaggle/input` for `train.csv`, `validation.csv`, and `test.csv`.

## Environment

Create an environment and install the dependencies:

```bash
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

For LSTM/TFT training, a CUDA-capable GPU is strongly recommended. The notebook automatically falls back to CPU where supported, although neural training will be substantially slower.

## Reproducibility notes

- Keep UTC as the merge/index time standard.
- Derive calendar variables from `Europe/Berlin` local time.
- Do not fit scalers on validation or test data.
- Do not create predictors from realised target-period load or weather.
- Keep the 2025 test set out of model and hyperparameter selection.
- Large raw/processed datasets and trained model binaries are excluded from Git by default; regenerate them with the notebooks.
