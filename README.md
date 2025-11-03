# Modeling Weather-Dependent Electricity Demand in Victoria Using Generalized Additive Mixed Models (GAMMs)

Welcome to the **Weather-Dependent Load Forecasting** repository. This project models and forecasts **daily electricity demand in Victoria, Australia (2013–2025)** by explicitly accounting for **nonlinear weather effects** and evolving rooftop PV impacts. The framework integrates **AEMO demand**, **BoM weather**, and **CER rooftop PV** data; unifies multi-station weather with **PCA**; defines **data‑driven local seasons** via **GMM**; and fits an interpretable **GAMM** to capture nonlinearities, seasonal regimes, and autocorrelation. The best model achieves **R² ≈ 0.94** and **MAPE ≈ 3.28%** out of sample.

---

## 👤 Authors & Affiliations

- **Rong Xu** — Monash University  
- **Rachael E. Quill (Mentor)** — State Electricity Commission Victoria

Repository: https://github.com/xurong980623/Weather-dependent-load-forecasting

---

## License

This project is licensed under the [MIT License](./LICENSE).

---

## 🗂 Repository Structure

| Folder / File                                | Purpose                                                                                                                |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| **Input Data/**                              | Raw input datasets (e.g., AEMO demand, BoM weather, CER PV data). |
| **report.qmd / report.html**                 | Quarto analysis report.                                                                       |
| **original code and analysis/**              | Exploratory or supplementary scripts.                                                                                  |
| **Presentation/**                            | Quarto presentation materials.                                                                           |                                                                          |
| **Weather-dependent-load-forecasting.Rproj** | RStudio project file for reproducibility.                                                                              |
| **report_cache/**, **report_files/**         | Quarto build outputs (auto-generated).                                 |
| **references.bib**                           | BibTeX citation database.                                                                                              |
| **README.md**                                | Main documentation — already includes Abstract, Methodology, Reproducibility, and Dependencies.                        |                                           |
| **styles/**                                  | Custom CSS for Quarto slides styling.                                                                               |


---

## 🧭 Project Overview

Accurate load forecasting is vital for **planning, operations, and market efficiency**. Victoria’s demand has become **more weather‑sensitive** due to climate variability and rapid **rooftop PV** adoption (post‑2019 Solar Homes Program). This project builds a **transparent, adaptive** framework that:
1) unifies multi‑station weather via **PCA** (virtual weather station),  
2) segments regimes with **GMM local seasons**, and  
3) models nonlinear effects using **GAMM** with regime‑specific variance.  

**Key outcome:** Interpretable forecasts with industry‑grade accuracy (**MAPE ≈ 3.28%, R² ≈ 0.94**).

---

## 📦 Data Sources (2013‑07‑01 → 2025‑08‑05)

- **AEMO**: VIC1 Scheduled Demand (half‑hourly → aggregated to **daily MWh**).  
- **BoM**: Daily **Tmax, Tmin, Solar exposure (MJ/m²)** at **Melbourne (OP), Morwell, Ballarat**.  
- **CER**: Postcode‑level **rooftop PV capacity & counts** (aggregated to VIC).

**Modeling window:** **2019‑07‑01 → 2025‑06‑30** (pre‑2019 used for context/EDA).

---

## 🧪 Methodology (end‑to‑end)

1. **Integration & QC**: Align daily demand with BoM weather; interpolate rare missing weather days (<1%) using `zoo::na.approx()`; validate ranges & alignment.  
2. **PCA weather fusion**: Build regional indices `tmax_pca`, `tmin_pca`, `solar_pca`. **PC1+PC2 explain ~87%** of variance → stable, multicollinearity‑free inputs.  
3. **Temperature indicators**: Construct interpretable metrics:  
   - `Tmean_pca = (Tmax_pca + Tmin_pca)/2` (comfort)  
   - `Trange_pca = Tmax_pca − Tmin_pca` (diurnal spread)  
   - `HDD = max(0, 16.5 − Tmean_pca)` (AEMO VIC)  
   - `CDD = max(0, Tmean_pca − 18.0)` (AEMO VIC)  
4. **Local seasons via GMM** (`mclust`): 6 weather regimes by BIC: **Cold & Clear, Cold & Cloudy, Hot & Very Sunny, Warm & Bright, Mild & Sunny, Mild & Cloudy**.  
5. **GAM / GAMM modeling** (`mgcv`): Nonlinear smooths & tensor interactions by regime; calendar terms (DOY, DOW, holidays); lags (`D[t‑1]`, `D[t‑7]`); heteroskedasticity via **`varIdent(~1 | local_season)`**.  
6. **Validation**: Train **2019‑07‑01 → 2024‑06‑30**; test **2024‑07‑01 → 2025‑06‑30**. Diagnostics (`gam.check`, Ljung–Box, `gratia::appraise`).  
7. **Forecasts**: Recursive h‑step; short‑ to medium‑term horizons (3d, 7d, 14d, 30d, 365d) and **July 2025** out‑of‑training evaluation.

---

## 🧰 Model Grid & Best Spec

**Season groupings × Temperature constructions (9 variants):**

| ID | Season grouping | Temperature representation |
|---|---|---|
| 01 | Month | Tmin, Tmax |
| 02 | Month | Tmean, Trange |
| 03 | Month | HDD, CDD |
| 04 | Traditional Season | Tmin, Tmax |
| 05 | Traditional Season | Tmean, Trange |
| 06 | Traditional Season | HDD, CDD |
| 07 | **Local Season** | Tmin, Tmax |
| 08 | **Local Season** | **Tmean, Trange** |
| 09 | **Local Season** | HDD, CDD |

**Best trade‑off (accuracy + diagnostics):** **Local Season — Tmean/Trange (ID 08)**.  
- Typical formula sketch:  
  ```r
  demand ~ t + t_sin + t_cos + local_season +
           s(Tmean_pca, by = local_season) +
           s(Trange_pca, by = local_season) +
           s(solar_pca, by = local_season) +
           ti(Tmean_pca, Trange_pca) +
           ti(Tmean_pca, solar_pca) +
           ti(Trange_pca, solar_pca) +
           s(doy, bs = "cc") + dow + is_holiday +
           demand_lag1 + demand_lag7
  # In GAMM: weights = varIdent(~1 | local_season)
  ```

---

## 📈 Results (hold‑out 2024‑07‑01 → 2025‑06‑30)

**Overall:** **R² ≈ 0.94**, **MAPE ≈ 3.28%** (best GAMM with varIdent).  
Representative **test** comparison (excerpt):
| Model | R² | RMSE (MWh) | MAPE | LB14 | LB28 | Verdict |
|---|---:|---:|---:|:--:|:--:|---|
| 07 Local — Tmin/Tmax | 0.932 | 4883.9 | 3.288 | ✅ | ✅ | **Pass** |
| **08 Local — Tmean/Trange** | **0.933** | **4865.4** | **3.294** | ✅ | ✅ | **Pass** |
| 09 Local — HDD/CDD | 0.933 | 4842.4 | 3.341 | ❌ | ✅ | Fail |

**H‑step forecasts (test):**

| Horizon | R² | RMSE | MAPE |
|---|---:|---:|---:|
| 3‑day | 0.776 | 2605.0 | 1.68 |
| 7‑day | 0.932 | 4430.8 | 2.59 |
| 14‑day | 0.932 | 3572.6 | 1.96 |
| 30‑day | 0.868 | 4609.8 | 2.51 |
| 365‑day | 0.939 | 4843.0 | 3.28 |

**July 2025** (post‑training): RMSE ≈ **5.5 GWh**, MAPE ≈ **3.14%**.

---

## 🔁 Reproducibility

### Dependencies

Install the following R packages (deduplicated from the project’s library block). For reproducibility, consider `renv::init()` after installing.

```r
# Core packages
install.packages(c(
  "tidyverse","lubridate","tsibble","fabletools","feasts","forecast","broom",
  "mgcv","nlme","gratia","mclust","yardstick",
  "here","readxl","zoo","imputeTS","missRanger","stringr","forcats",
  "ggplot2","ggrepel","patchwork","ggpubr","gridExtra","kableExtra","gt","scales","RColorBrewer","latex2exp"
))

# Optional (EDA/visualization/extras)
install.packages(c(
  "plotly","gganimate","gifski","GGally","car","energy","fy","ragg","knitr","strucchange"
))
```



### Requirements
- **R ≥ 4.5.1** (tested)  
- **Quarto ≥ 1.4**  
- R packages:
  ```r
  install.packages(c(
    "tidyverse","lubridate","tsibble","fpp3","mgcv","gratia",
    "mclust","broom","patchwork","kableExtra","zoo"
  ))
  ```

### Steps
```bash
# 1) Clone
git clone https://github.com/xurong980623/Weather-dependent-load-forecasting.git
cd Weather-dependent-load-forecasting

# 2) (Optional) Set an R project / renv if used

# 3) Prepare data
#   - Place/verify AEMO, BoM, CER files under data/
#   - Run scripts in scripts/ for cleaning, PCA, clustering, and features

# 4) Fit & evaluate models
#   - Run modeling script(s) under scripts/ (GAM/GAMM, diagnostics, forecasts)

# 5) Render the report
quarto render report.qmd --to html
# or
quarto render report.qmd --to pdf
```

---

## 🧠 Interpretation Highlights

- **Temperature** drives a **U‑shaped** demand curve: heating at low T, cooling at high T; minima near mild days.  
- **Solar exposure** has a **negative** effect (PV offset), strongest on clear/sunny regimes.  
- **Local seasons (GMM)** outperform calendar seasons for realism and diagnostics.  
- **Heteroskedasticity** differs by regime; **GAMM + varIdent** stabilizes residuals.

---

## ⚠️ Limitations

- Uses **observed** weather; operational deployment requires **forecast** weather (NWP/ensembles).  
- **Daily** granularity; does not model **intraday** load shape.  
- Clusters built from weather only (not joint with demand).  
- Excludes **humidity / wind chill**; periodic **retraining** advised.  
- **Extrapolation risk** under unprecedented weather or PV uptake.

---

## 🚀 Future Work

- Integrate **NWP** (ensembles) for **probabilistic** day‑ahead forecasts.  
- Compare to **ML** baselines (XGBoost, LSTM).  
- Add **humidity/wind**; incorporate **EV charging** and **battery** signals.  
- Scale to **sub‑regions** with regionalized GMM.

---

## 🙏 Acknowledgments

This work was completed as part of a **Monash University** capstone in collaboration with the **State Electricity Commission (SEC) Victoria**.  
We thank mentors and reviewers for guidance on methodology, diagnostics, and system context.

