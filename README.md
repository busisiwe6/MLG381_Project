# MLG381 — Diabetes & Heart Disease Decision Support

## Overview

This project was developed for **BC Analytics**, a health-tech framing aimed at improving patient outcomes through data-driven healthcare. The original intent was to **spot diabetes-related risk earlier** and give healthcare providers **interpretable signals** they can combine with clinical judgment.

The codebase **extends that scope**: an optional **heart disease** pipeline (Statlog/Cleveland-style features) runs **in the same Dash app** as the diabetes workflow, and **notebook-derived figures** can be embedded for exploratory analysis—in addition to predictions and driver-style explanations.

This tool supports decisions and **does not replace medical advice**.

## Objectives

The project aims to:

- Predict **diabetes stage** (`diabetes_stage`) from lifestyle and clinical inputs.
- When trained, estimate **heart disease risk / class** from structured cardiac attributes.
- Surface **lifestyle and clinical drivers** behind the model output (including optional **SHAP**-style explanations when enabled).
- Explore **patient segmentation** in notebooks (e.g. **K-Means**, k = 3).
- Deliver an **interactive Dash dashboard** with one tab per trained model plus a **Notebook figures** gallery.

## Methodology (CRISP-DM)

1. **Business understanding** — Providers need transparent, repeatable risk context to support—not replace—clinical decisions.
2. **Data understanding** — `Diabetes_and_LifeStyle_Dataset` (and related names) supply lifestyle metabolism fields; Statlog-style heart CSVs supply cardiac risk attributes when used.
3. **Data preparation** — Cleaning, encoding, and scaling via `SRC/prepare_diabetes_data.py` and `SRC/prepare_heart_disease_data.py`.
4. **Modelling** — Supervised learners are trained and serialized to `ARTIFACTS/`; **Random Forest** is the primary model served to the web app for each pipeline. **Decision Tree** and **XGBoost** are trained for diabetes as comparators (`SRC/train.py`).

## Risk classification & analysis (summary)

| Area | What we use |
|------|----------------|
| Diabetes — primary | **Random Forest** (plus Decision Tree & XGBoost for comparison in `SRC/train.py`) |
| Heart — primary | **Random Forest** (`SRC/train_hd.py`) |
| Segmentation | **K-Means** (e.g. k = 3) in notebooks under `NOTEBOOKS/` |
| Drivers | Feature-importance paths in-app; optional **SHAP** when `ENABLE_SHAP` is set and UI bundles include background data |
| Web app | **Dash** in `SRC/app.py` — predictions, guidance text, embedded **notebook figures** (`SRC/extract_notebook_figures.py`) |

## Deployment (cloud)

The live service runs as **Gunicorn** over **WSGI** (`wsgi.py`, see `Procfile`): bind **`0.0.0.0:$PORT`**. Typical hosts include **Railway**, **Render** (`render.yaml`), or any platform that injects **`PORT`** at runtime.

---

## What’s in the repo

| Area | Contents |
|------|-----------|
| `SRC/app.py` | Dash UI: tabs per loaded model + “Notebook figures” gallery |
| `SRC/dataset_runtime.py` | Loads artifact bundles from `ARTIFACTS/` (`db` / `hd` panels) |
| `SRC/prepare_diabetes_data.py` | Diabetes CSV → train/test splits + `DataModel_db.pkl` / `UIModel_db.pkl` |
| `SRC/prepare_heart_disease_data.py` | Heart (Statlog) CSV → splits + `DataModel_hd.pkl` / `UIModel_hd.pkl` |
| `SRC/train.py` | Trains RF / DT / XGBoost on diabetes splits; saves `Diabetes_*.pkl` |
| `SRC/train_hd.py` | Trains heart Random Forest → `Heart_rfModel.pkl` |
| `SRC/extract_notebook_figures.py` | Exports PNGs from `NOTEBOOKS/*.ipynb` → `SRC/assets/notebook_figures/` |
| `NOTEBOOKS/` | Jupyter analyses (e.g. `Diabetes_Lifestyle.ipynb`, `Heart_Disease.ipynb`, `K-Means.ipynb`) |
| `DATA/` | Place CSV datasets here (often gitignored — use `DATASET_URL` on deploy) |
| `ARTIFACTS/` | Trained models + UI bundles (often gitignored) |
| `wsgi.py` | WSGI entry: `application` for Gunicorn |
| `Procfile` | Heroku/Railway-style web process |
| `render.yaml` | Optional Render.com service definition |

The app starts only if **at least one** full bundle exists under `ARTIFACTS/` (RF + DataModel + UIModel for that pipeline).

---

## Local setup

```bash
python -m venv .venv
.venv\Scripts\activate          # Windows
# source .venv/bin/activate     # macOS / Linux

pip install -r requirements.txt
```

### Diabetes pipeline

```bash
python SRC/prepare_diabetes_data.py   # or SRC/prepare_data.py (compat alias)
python SRC/train.py
```

Produces (among others) `ARTIFACTS/Diabetes_rfModel.pkl`, `DataModel_db.pkl`, `UIModel_db.pkl`.

### Heart pipeline 

```bash
python SRC/prepare_heart_disease_data.py
python SRC/train_hd.py
```

Produces `ARTIFACTS/Heart_rfModel.pkl`, `DataModel_hd.pkl`, `UIModel_hd.pkl`.

### Run the app

From the **project root** (so `DATA/` and `ARTIFACTS/` resolve correctly):

```bash
python SRC/app.py
# or
python -m SRC.app
```

Default dev URL: `http://127.0.0.1:8051` — see `SRC/app.py` if the port differs.

### Notebook figures in the UI

After you run notebook cells so figures appear in outputs, regenerate assets:

```bash
python SRC/extract_notebook_figures.py
```

This updates `SRC/assets/notebook_figures/manifest.json` and PNGs consumed by the “Notebook figures” tab.

---

## Environment variables

| Variable | Purpose |
|----------|---------|
| `DATASET_URL` | HTTPS URL to download the diabetes CSV during **prepare** if the file is not in `DATA/` |
| `ENABLE_SHAP` | Set to `1` / `true` to enable SHAP-based driver analysis in the app (heavier CPU/RAM) |

---

## Deployment notes

- **Railway / similar:** use the `Procfile` pattern — Gunicorn binds to `0.0.0.0:$PORT` with one worker when memory is tight.
- **Render:** `render.yaml` documents an example **build** (install + prepare + train) and **start** command; adjust for one or two pipelines.
- Ensure the **build** produces `ARTIFACTS/*.pkl` or commit artifacts appropriate to your policy. If `DATA/*.csv` is not in Git, set `DATASET_URL` (and any heart data source you rely on) in the host’s environment.
