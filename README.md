# Modeling Weather-Dependent Electricity Demand in Victoria Using Generalized Additive Mixed Models (GAMMs)

This repository contains the Quarto report, R code, and documentation for **modeling and forecasting daily electricity demand in Victoria, Australia (2013–2025)**.  
The project integrates **AEMO electricity demand**, **BoM weather**, and **CER rooftop PV** data.  
Using **Principal Component Analysis (PCA)**, **Gaussian Mixture Models (GMM)**, and **Generalized Additive Mixed Models (GAMMs)**, it builds a transparent, interpretable, and high-accuracy weather-dependent forecasting framework.  
The final GAMM achieves **R² ≈ 0.94** and **MAPE ≈ 3.28%** on out-of-sample data.

---

## 👤 Authors & Affiliations

- **Rong Xu** — Master of Business Analytics, Monash University  
- **Rachael E. Quill (Mentor)** — State Electricity Commission Victoria  

**Repository:** [github.com/xurong980623/Weather-dependent-load-forecasting](https://github.com/xurong980623/Weather-dependent-load-forecasting)

---

## 🧭 Project Overview

Electricity demand in Victoria has become increasingly **weather-sensitive** due to climate variability, electrification, and widespread **rooftop PV adoption**.  
This project develops a **data-driven forecasting framework** that:

1. **Unifies weather data** from multiple BoM stations using **PCA** (virtual weather station).  
2. **Defines data-driven seasons** with **Gaussian Mixture Models (GMM)** to identify meteorological regimes.  
3. **Models nonlinear weather-demand relationships** using a **GAMM**, capturing regime-specific variance and autocorrelation.  

**Goal:** Provide interpretable, reliable, and scalable forecasts for operational energy planning and policy analysis.

---

## 🗂 Repository Structure

| Folder / File | Description |
| :------------- | :----------- |
| **Input Data/** | Raw AEMO, BoM, and CER datasets  |
| **Presentation/** | Quarto presentation slides |
| **picture/** | Figures and visual outputs used in the report |
| **report.qmd / report.html** | Main Quarto analysis file and rendered outputs|
| **Modeling Weather-Dependent Electricity Demand in Victoria Using Generalized Additive Mixed Models (GAMMs).pdf** | Final polished report for submission and offline viewing. |
| **original code and analysis.qmd** | Early version used for development and testing |
| **Weather-dependent-load-forecasting.Rproj** | RStudio project file |
| **references.bib** | Bibliography for citations |
| **README.md** | This documentation file |
| **LICENSE** | Project license (MIT License) |
| **.gitignore** | Excludes temporary and cache files |

---

## 📦 Data Sources (2013-07-01 → 2025-08-05)

- **AEMO**: Half-hourly VIC1 Scheduled Demand (aggregated to daily MWh)  
- **BoM**: Daily **Tmax**, **Tmin**, and **Solar Exposure (MJ/m²)** from  
  - Melbourne (Olympic Park)  
  - Morwell (Latrobe Valley)  
  - Ballarat (Aerodrome)  
- **CER**: Postcode-level rooftop PV capacity and installations (aggregated to Victoria)  

**Modeling period:** 2019-07-01 → 2025-06-30 (earlier data used for EDA and model calibration).

---

## 🧪 Methodology (Step-by-Step)

1️⃣ **Data Integration & Cleaning**  
- Merge AEMO demand and BoM weather data by date.  
- Interpolate rare missing values (<1%) with `zoo::na.approx()`.  
- Validate data ranges and consistency.

2️⃣ **PCA Weather Fusion**  
- Combine Tmax, Tmin, and Solar exposure into unified regional indices (`tmax_pca`, `tmin_pca`, `solar_pca`).  
- PC1 + PC2 explain ~87% of total variance.

3️⃣ **Derived Temperature Indicators**  
| Variable | Formula | Interpretation |
|:--|:--|:--|
| `Tmean` | (Tmax + Tmin) / 2 | Mean comfort temperature |
| `Trange` | Tmax − Tmin | Diurnal temperature spread |
| `HDD` | max(0, 16.5 − Tmean) | Heating Degree Days |
| `CDD` | max(0, Tmean − 18) | Cooling Degree Days |

4️⃣ **Local Seasons (GMM Clustering)**  
- Cluster days by PCA weather features using `mclust` to form six meteorological regimes: *Cold & Cloudy*, *Cold & Clear*, *Hot & Very Sunny*, *Warm & Bright*, *Mild & Sunny*, *Mild & Cloudy*.

5️⃣ **Modeling with GAMM**  
```r
demand ~ t + t_sin + t_cos + local_season +
         s(Tmean_pca, by = local_season) +
         s(Trange_pca, by = local_season) +
         s(solar_pca, by = local_season) +
         ti(Tmean_pca, solar_pca) +
         s(doy, bs = "cc") + dow + is_holiday +
         demand_lag1 + demand_lag7
# GAMM includes variance structure: varIdent(~1 | local_season)
```

6️⃣ **Validation & Forecasting**  
- Train (2019–2024), Test (2024–2025).  
- Metrics: R², RMSE, MAPE, Ljung–Box.  
- Recursive multi-horizon forecasts (3–365 days).

---

## 📈 Model Performance Summary

| Horizon | R² | RMSE (MWh) | MAPE (%) |
|:--------:|:--:|:-----------:|:---------:|
| 3-day | 0.776 | 2605.0 | 1.68 |
| 7-day | 0.932 | 4430.8 | 2.59 |
| 14-day | 0.932 | 3572.6 | 1.96 |
| 30-day | 0.868 | 4609.8 | 2.51 |
| 365-day | 0.939 | 4843.0 | 3.28 |

**Highlights:**  
- Short-term forecasts (≤14 days) achieve high accuracy.  
- July 2025 out-of-sample: RMSE ≈ 5.5 GWh, MAPE ≈ 3.1%.  
- Residuals pass Ljung–Box → no autocorrelation.

---

## ⚙️ Reproducibility Guide

### Requirements
- **R ≥ 4.3**
- **Quarto ≥ 1.4**

### Install Dependencies
```r
install.packages(c(
  "tidyverse","lubridate","tsibble","fabletools","feasts","forecast",
  "mgcv","nlme","gratia","mclust","yardstick","kableExtra","gt",
  "patchwork","ggrepel","zoo","imputeTS","here","stringr"
))
```

### Run Analysis
```bash
# Clone repository
git clone https://github.com/xurong980623/Weather-dependent-load-forecasting.git
cd Weather-dependent-load-forecasting

# Place AEMO, BoM, CER data in /Input Data/
# Render report
quarto render report.qmd --to html
# or
quarto render report.qmd --to pdf
```

---

## 🔍 Key Insights

- **Temperature:** U-shaped demand relationship — heating at low T, cooling at high T.  
- **Solar Exposure:** Negative effect (rooftop PV offset).  
- **Local Seasons:** GMM regimes outperform calendar seasons.  
- **Variance Structure:** `varIdent()` stabilizes residual variance.

---

## ⚠️ Limitations & Future Work

**Limitations:**  
- Observed (not forecast) weather data.  
- Daily aggregation — no intraday detail.  
- No humidity/wind variables.

**Future Work:**  
- Integrate **NWP forecasts** for probabilistic demand prediction.  
- Compare with **ML models** (XGBoost, LSTM).  
- Add **EV charging** and **battery** effects.  
- Extend to **regional sub-networks**.

---

## 🧾 License

This project is licensed under the [MIT License](./LICENSE).  

---

## 🙏 Acknowledgments

This project was conducted as part of a Monash University capstone, in collaboration with the
State Electricity Commission Victoria (SEC VIC).
Special thanks to Rachael E. Quill and the SEC Energy Market team for their mentorship, data insights, and methodological guidance.