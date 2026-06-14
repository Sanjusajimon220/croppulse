# CropPulse

**Operational crop monitoring from Sentinel-1, Sentinel-2 and ERA5 — using radar to fill optical cloud gaps.**

CropPulse is a reproducible Earth-observation pipeline that mirrors the kind of operational agricultural-monitoring services used in production (cloud-gap-filled vegetation curves, crop-type mapping, drought monitoring, and a yield indicator). It is built end-to-end on open data and open tools for an arable region of Lower Saxony, Germany.

Region: Hildesheim Börde (Lower Saxony) · Crop year: 2021 · 7 crops · 210 fields

---

## What it does

| Component | Method | Mirrors the service |
|---|---|---|
| **Cloud-gap-filled NDVI/LAI** | Train a model mapping Sentinel-1 radar → Sentinel-2 NDVI, then predict NDVI on every (cloud-free) radar date to fill optical gaps | Daily LAI |
| **Per-field phenology** | Start / peak / end of season from the gap-filled curve | Cultivation Patterns |
| **Crop classification** | Random Forest on the gap-filled NDVI time series, 7 crops | Cultivation Patterns |
| **Drought monitor** | Multi-year growing-season anomalies: ERA5 rainfall deficit + NDVI anomaly | Drought Monitor |
| **Yield proxy** | Season-integrated NDVI (iNDVI) as a relative biomass indicator | Yield Prediction |

The centerpiece is the **Sentinel-1 → Sentinel-2 fusion**: radar sees through clouds, so it can reconstruct the optical greenness curve on dates when Sentinel-2 is obscured — the core problem operational LAI services exist to solve.

## Results

- **Radar → NDVI gap-fill:** 5-fold cross-validated **R² = 0.65** across 7 crops (gradient boosting with temporal lag/rolling features; RF baseline 0.57). Per-crop R² ranges from **0.81 (maize)** and 0.78 (sugar beet) down to **0.38 (winter barley)** — radar predicts greenness best for tall, structurally dynamic crops and worst for cereals and grassland, consistent with the SAR literature.
- **Crop classification:** **0.98** 5-fold CV accuracy over 7 crops (210 fields). The few errors are confined to phenologically overlapping crops (maize / sugar beet / potatoes among the summer row crops; barley / rape among the winter crops) — i.e. the model errs only where a human agronomist would. Winter wheat and winter barley are separated cleanly because barley is harvested earlier.
- **Phenology:** recovers seven distinct, agronomically correct crop calendars — including the correct ordering of winter barley being harvested before winter wheat.
- **Drought monitor:** independently re-detects the two well-documented severe German droughts (**2018** and **2022**) as years that are both rainfall-deficient and NDVI-depressed, while correctly leaving wetter years (2021, 2023) unflagged.
- **Yield proxy:** season-integrated NDVI ranks crops sensibly (grassland highest — green most of the year; potatoes lowest — shortest season). Quantitative validation against official district yield statistics is in progress.

## Method (short)

1. Load EuroCrops field polygons + harmonized crop labels for the AOI; sample 30 fields per crop.
2. Build cloud-masked Sentinel-2 NDVI/LAI per field (Cloud Score+ masking) and speckle-filtered Sentinel-1 VV/VH per field.
3. Pair each radar acquisition with the nearest clear Sentinel-2 NDVI; train a regressor `[VV, VH, VV–VH, lag/rolling features, day-of-year, crop] → NDVI`; predict on all radar dates → continuous gap-filled curve.
4. Derive phenology, classify crops, compute multi-year drought anomalies (ERA5 + NDVI), and integrate NDVI for the yield proxy.

## Repository structure

```
croppulse/
├── README.md
├── requirements.txt
├── croppulse.ipynb          # full pipeline, runs in Google Colab cell-by-cell
└── figures/                 # exported result figures
```

## How to run

The pipeline runs in **Google Earth Engine** (Python API) via Google Colab:

1. Download the EuroCrops Lower Saxony shapefile (`DE_LS_2021.zip`, Zenodo record `10118572`) and upload it as an Earth Engine asset.
2. Open `croppulse.ipynb` in Colab, set your EE project and the asset path in the config cell.
3. Run cells top to bottom. The notebook prints the field count, gap-fill R², phenology table, classification accuracy + confusion matrix, drought table, and the yield proxy.

## Data & licenses

- **EuroCrops** — field boundaries + harmonized crop labels. CC-BY-SA 4.0. Schneider et al., *Sci Data* 10, 612 (2023). Funded by DLR / BMWK; TUM.
- **Sentinel-1 GRD**, **Sentinel-2 SR**, **ERA5-Land** — accessed via the Google Earth Engine catalog (Copernicus / ECMWF).
- Open Python stack: Earth Engine API, geemap, scikit-learn, pandas, NumPy, Matplotlib, SciPy.

## Honest limitations

- **LAI** is currently an empirical NDVI-based proxy, not a rigorous biophysical retrieval (SNAP Biophysical Processor / a trained S2→LAI model is the next step).
- Results are for a **single AOI and a single year (2021)** — robustness across regions/years is not yet established.
- Classification uses a **random** train/test split; spatial autocorrelation between nearby fields means a spatially-blocked cross-validation would be more conservative and is the rigorous next step.
- The yield indicator is **relative** (NDVI-days) and not yet calibrated/validated against official yield statistics.

## Author

Sanju Sajimon — MSc Remote Sensing & Geoinformatics (KIT).
