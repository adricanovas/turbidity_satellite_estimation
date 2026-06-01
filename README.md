# Surface Turbidity Estimation in the Mar Menor Lagoon

Estimating surface water turbidity in the **Mar Menor** coastal lagoon (Region of
Murcia, Spain) from **Planet SuperDove** multispectral imagery, calibrated against
**CTD in-situ** measurements. The repository contains the full machine-learning
pipeline, from data matching to a spatial turbidity map.

## Data

- **Satellite** — Planet SuperDove scenes (8 spectral bands, 443–865 nm) over the lagoon.
- **In-situ** — UPCT CTD network: surface turbidity (sampled at 0.5 m) from 11 stations.
- **Matched dataset** — 341 CTD ↔ satellite samples across 31 acquisition dates (2023–2024).

## Method

- **Targets** — four variants of the turbidity target are compared: `base`, `log1p`,
  Yeo-Johnson (`yeojohnson`) and `trimmed` (1–99 % quantiles).
- **Features** — circular image patches are extracted at five scales
  (16 / 32 / 64 / 128 / 256 px ≈ 48–768 m diameter); per-band statistics and spectral
  indices (NDTI, NDWI, NDCI, …) give ~32 features per sample.
- **Selection** — dimensionality-reduction benchmark (PCA, Kernel PCA, UMAP,
  autoencoder, RFECV) with `GroupKFold` to avoid station/date leakage.
- **Models** — Gradient Boosting (HistGB) and CatBoost tuned with Optuna over the
  4 datasets × 5 patch scales (40 runs), plus advanced methods (XGBoost, LightGBM,
  stacking).
- **Best model** — `log1p` · 64 px (192 m) · **CatBoost** (test MAE = 0.253 in log1p space).

## Pipeline (run the notebooks in order)

| Notebook | Purpose |
|---|---|
| `00_date_clustering` | Acquisition-date planning (turbidity regimes, coverage gaps) |
| `01_data_exploration` | CTD ↔ satellite matching and EDA |
| `02_data_preprocessing` | Build the four target-variable datasets |
| `03_satellite_imagery_processing` | Multi-scale circular patch extraction, band statistics |
| `04_feature_engineering` | Band statistics + spectral indices |
| `05_dimensionality_reduction` | Reduction / feature-selection benchmark |
| `05_model_training` | GBR + CatBoost training and tuning (Optuna) |
| `05b_advanced_methods` | XGBoost, LightGBM, stacking |
| `06_model_evaluation` | Metrics, heatmaps, overfitting diagnosis, residuals |
| `07_predictions_and_mapping` | Grid inference and turbidity maps (GeoTIFF) |

## Project layout

```
code/         the 10 pipeline notebooks
data/
  raw/UPCT/   raw CTD measurements
  datasets/   matched and preprocessed datasets
geospatial/   lagoon polygon, CTD points, QGIS project
results/      metrics (CSV), figures, trained models, GeoTIFF rasters
imgs/         miscellaneous images
```

## Setup & run (uv)

This project uses [uv](https://docs.astral.sh/uv/) for environment and dependency
management. From the project root:

```bash
uv sync                # create .venv and install everything (writes uv.lock)
uv run jupyter lab     # open the notebooks in the project environment
```

Then run the notebooks in the numerical order shown above.

Notes:
- Python 3.10–3.12 is supported (prebuilt wheels for PyTorch, CatBoost, rasterio, etc.).
- PyTorch installs the CPU build by default; uncomment the CUDA index block in
  `pyproject.toml` and re-run `uv sync` to use GPU wheels.
- No system GDAL is required — `rasterio`, `geopandas` and `shapely` ship binary wheels.

## Outputs

Running the pipeline regenerates everything in `results/`: per-configuration metrics
(`model_metrics.csv`), per-sample predictions, evaluation figures, the serialized best
model, and the predicted turbidity rasters/maps for the lagoon.
