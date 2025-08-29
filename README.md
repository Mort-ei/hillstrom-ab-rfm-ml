# A/B Testing + RFM + Uplift Modeling (Hillstrom)

End-to-end experimentation project on the **Kevin Hillstrom / MineThatData** email test.  
Covers **ITT** analysis, **RFM** segmentation, and a **T-learner uplift model** to rank customers by *incremental* conversion, then packages outputs for a **Power BI** dashboard.

---

## 🎯 Goals

- Answer **“Did the campaign work?”** (ITT/CUPED, binary & continuous outcomes)
- Answer **“For whom does it work?”** (RFM/channel/newbie heterogeneity)
- Answer **“Who should we target?”** (T-learner uplift, Qini/AUUC, decile lift)
- Ship **BI-ready tables** and a simple **targeting scenario** file

---

## 🧰 Tech

- **Python**: pandas, numpy, scikit-learn (HGB, CalibratedClassifierCV), matplotlib
- **Jupyter** notebooks
- **Power BI**: star-like model, slicers, DAX KPIs (top-% targeting)
- (Optional) statsmodels / catboost

---

## 📦 Repository Structure

```
hillstrom-ab-rfm-ml/
├─ notebooks/
│  ├─ 01_ingest_clean.ipynb          # load + clean → hillstrom_clean.parquet
│  ├─ 02_rfm_features.ipynb          # derive R/F/M → rfm_features.parquet
│  ├─ 03_experiment_itt_cuped.ipynb  # ITT for visit/conv/spend + CUPED
│  ├─ 04_heterogeneity_rfm.ipynb     # R×F×M, channel, newbie lifts + plots
│  ├─ 06_uplift_tlearner.ipynb       # T-learner uplift (Mens/Womens vs Control)
│  └─ 07_packaging_bi.ipynb          # curated tables + scenario CSV for BI
├─ data/
│  ├─ processed/                     # intermediate parquet/csv (large, gitignored)
│  └─ curated/                       # BI-ready outputs (facts/dims/scenarios)
├─ output/
│  ├─ plots/                         # Qini curves, calibration, heatmaps
│  └─ bi/                            # optional CSV mirrors for Power BI
├─ requirements.txt                  # (optional) pinned deps
└─ README.md
```

> Note: We intentionally skipped a separate “propensity only” notebook; uplift modeling is the portfolio-worthy deliverable for targeting.

---

## 🗂️ Dataset

**MineThatData E-Mail Analytics Challenge**  
Randomized A/B/C test: **Mens E-Mail**, **Womens E-Mail**, **No E-Mail (control)**. Outcomes over 2 weeks:

- Binary: `visit`, `conversion`
- Continuous: `spend`

Pre-period features (no leakage): `recency`, `history`, `mens`, `womens`, `zip_code`, `newbie`, `channel` (+ derived `R`,`F`,`M`).

---

## 🔁 Reproduce Locally

**Python 3.10–3.12** recommended.

```bash
# create / activate venv (Windows PowerShell)
py -3.11 -m venv .venv
.\.venv\Scripts\Activate

python -m pip install -U pip setuptools wheel
pip install -U pandas numpy scikit-learn scipy matplotlib pyarrow fastparquet
# optional extras:
# pip install statsmodels catboost scikit-uplift

# optional: register kernel for Jupyter
python -m ipykernel install --user --name hillstrom-venv --display-name "hillstrom (venv)"
```

**Run order:**

1. `01_ingest_clean.ipynb` → `data/processed/hillstrom_clean.parquet`  
2. `02_rfm_features.ipynb` → `data/processed/rfm_features.parquet`  
3. `03_experiment_itt_cuped.ipynb` (optional for BI)  
4. `04_heterogeneity_rfm.ipynb` (optional for BI)  
5. `06_uplift_tlearner.ipynb` →  
   - `data/processed/uplift_scores.parquet`  
   - `data/processed/uplift_metrics.json`  
   - `data/processed/qini_curve_{mens,womens}.csv`  
   - `data/processed/lift_by_decile_{mens,womens}.csv`  
6. `07_packaging_bi.ipynb` → BI-ready tables in `data/curated/` and CSV mirrors in `output/bi/`

---

## 🧪 Methods (what’s implemented)

- **ITT (Intention-to-Treat):**  
  - Binary outcomes: pooled Wald CIs, Wilson intervals per arm  
  - Spend: Welch / robust OLS (HC1) CIs  
- **CUPED:** pre-period covariate adjustment to tighten CIs (optional)
- **Heterogeneity:** R×F×M heatmaps; channel/newbie bars with CIs
- **Uplift (T-learner):**  
  - Two calibrated models per arm (`mens` vs `control`, `womens` vs `control`)  
  - **Qini curves**, **AUUC** integral, **decile lift** QC  
  - Per-user **uplift scores** and **deciles** saved for targeting

---

## 📄 Key Outputs (for analysis & BI)

**Curated (for BI)** — in `data/curated/`:
- `dim_customer.parquet` — pre-period features (+RFM if present)
- `fact_outcomes.parquet` — assigned arm & outcomes
- `fact_uplift.parquet` — `uplift_mens`, `uplift_womens`, deciles, convenience flags
- `scenario_uplift_targeting.csv` — top-X% scenarios per arm (N target, mean uplift, expected incremental conversions, gross/net value)
- *(optional)* `summary_itt.csv` — normalized table with ITT/CUPED effects and CIs

**Plots** — in `output/plots/`:
- `qini_mens.png`, `qini_womens.png`
- Any heterogeneity or calibration plots you choose to save

---

## 📊 Power BI Dashboard (how to build it)

### Data import & model
- **Get Data** → Folder: `output/bi` (or directly `data/curated`)  
  Load:
  - `dim_customer.csv` (or parquet)
  - `fact_outcomes.csv`
  - `fact_uplift.csv`
  - `scenario_uplift_targeting.csv`
  - *(optional)* `summary_itt.csv`, `qini_curve_mens.csv`, `qini_curve_womens.csv`
- **Model view**: set relationships (single direction, dim → fact)
  - `dim_customer[customer_id]` → `fact_outcomes[customer_id]` (1:* )
  - `dim_customer[customer_id]` → `fact_uplift[customer_id]` (1:* )

### Pages & visuals

**Page A — Overview**
- Table (from `summary_itt.csv`): show `comparison`, `metric`, `effect`, `ci_low`, `ci_high`, `p_value`
- Cards: baseline rate (`p_c` for conversion), lift for selected arm/metric
- Slicers: `metric`, `comparison` (e.g., “mens vs control”)

**Page B — Segments**
- Matrix heatmap (if you exported `het_rfm.csv`): rows = `R`, cols = `M`, value = `lift` (conversion), diverging color centered at 0
- Bar charts by `channel` and `newbie`, colored by `comparison`

**Page C — Targeting (Uplift)**
- Slicers: **Arm** (“mens”/“womens”), **Top %** (What-If), **Send Cost**, **Revenue per Conversion**, **Gross Margin**
- KPI cards: **Targeted Count**, **Mean Uplift (Targeted)**, **Incremental Conversions**, **Net Value**
- Table of top deciles (optionally show predicted uplift decile distribution)

### DAX pack (paste into a Measures table or into `fact_uplift`)

Create a small table **Arm** with column `arm` values: `mens`, `womens`.  
Create What-If parameters for **Top %**, **Send Cost**, **Revenue per Conversion**, **Gross Margin**.

```DAX
Selected Arm :=
SELECTEDVALUE ( Arm[arm], "mens" )

Uplift Threshold :=
VAR pct  = DIVIDE ( SELECTEDVALUE('Top %'[Top % Value], 30), 100 )
VAR thrM = PERCENTILEX.INC ( ALLSELECTED(fact_uplift), fact_uplift[uplift_mens],   1 - pct )
VAR thrW = PERCENTILEX.INC ( ALLSELECTED(fact_uplift), fact_uplift[uplift_womens], 1 - pct )
RETURN IF ( [Selected Arm] = "mens", thrM, thrW )

Targeted Count :=
VAR thr = [Uplift Threshold]
RETURN
IF (
    [Selected Arm] = "mens",
    COUNTROWS ( FILTER ( ALLSELECTED(fact_uplift), fact_uplift[uplift_mens]   >= thr ) ),
    COUNTROWS ( FILTER ( ALLSELECTED(fact_uplift), fact_uplift[uplift_womens] >= thr ) )
)

Mean Uplift (Targeted) :=
VAR thr = [Uplift Threshold]
VAR colM = AVERAGEX ( FILTER ( ALLSELECTED(fact_uplift), fact_uplift[uplift_mens]   >= thr ), fact_uplift[uplift_mens] )
VAR colW = AVERAGEX ( FILTER ( ALLSELECTED(fact_uplift), fact_uplift[uplift_womens] >= thr ), fact_uplift[uplift_womens] )
RETURN IF ( [Selected Arm] = "mens", colM, colW )

Incremental Conversions :=
[Targeted Count] * [Mean Uplift (Targeted)]

Gross Revenue :=
[Incremental Conversions] * SELECTEDVALUE('Revenue per Conversion'[Revenue per Conversion Value], 1)

Gross Profit :=
[Gross Revenue] * SELECTEDVALUE('Gross Margin'[Gross Margin Value], 0.5)

Send Cost (Total) :=
[Targeted Count] * SELECTEDVALUE('Send Cost'[Send Cost Value], 0)

Net Value :=
[Gross Profit] - [Send Cost (Total)]
```

> **Tip:** format `Mean Uplift (Targeted)` as Percentage. Add tooltips showing AUUC/decile tables if you like.

---

## 📈 Interpreting Results (what to highlight)

- **AUUC (Qini)**: area under Qini; higher = better targeting separation.  
- **Decile lift**: the top predicted uplift deciles should show higher **observed** incremental conversion vs lower deciles.  
- **Scenario**: show net value at a practical targeting threshold (e.g., **Top 20%** by uplift).

You can add your numbers here after running:
- `AUUC (Mens vs Control): …`  
- `AUUC (Womens vs Control): …`  
- `Top-20% targeting: +N incremental conversions, Net $V`

---

## 🧪 Testing & QA

- No leakage: only **pre-period** features in models.  
- Calibrated models (sigmoid) for stable probabilities.  
- Stratified splits by **treatment × outcome** to keep base rates balanced.  
- Decile QC to ensure uplift makes sense experimentally.

---

## 🔐 .gitignore (suggested)

# Python / venv
.venv/
__pycache__/
*.py[cod]

# Jupyter
.ipynb_checkpoints/

# OS / IDE
.DS_Store
Thumbs.db
.vscode/

# Large data & outputs
data/processed/*
data/curated/*.parquet
output/plots/**
output/bi/**

---

## 🚀 Push to GitHub

```bash
# from project root
git init
git add .
git commit -m "Initial commit: A/B + RFM + T-learner uplift (Hillstrom)"
git branch -M main

# create empty repo on GitHub named hillstrom-ab-rfm-ml, then:
git remote add origin https://github.com/<your-username>/hillstrom-ab-rfm-ml.git
git push -u origin main
```
(If prompted for a password over HTTPS, use a **personal access token**.)

---

## 🙏 Attribution

Dataset: **Kevin Hillstrom / MineThatData** E-Mail Analytics Challenge (educational use).  
This repo is for learning/demonstration purposes only.
