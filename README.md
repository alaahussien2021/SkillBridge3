# SkillBridge — Salary Prediction Module

This package covers ONE slice of the full SkillBridge platform: the
**salary prediction model + API + demo UI**. It's built directly from
`freelance_jobs2.xlsx` (all 10 sheets), not just the `freelance_jobs` sheet.

## Folder structure

```
skillbridge/
├── data/
│   └── freelance_jobs2.xlsx      <- put your workbook here
├── src/
│   ├── features.py               <- shared feature engineering (train + serve)
│   └── train_salary_model.py     <- trains + evaluates + saves the model
├── api/
│   └── main.py                   <- FastAPI backend (/predict-salary, /lookups, ...)
├── model/                        <- created after training (model + metrics)
├── predictor.html                <- standalone frontend, calls the API
└── requirements.txt
```

## 1. Install dependencies

```bash
pip install -r requirements.txt
```

## 2. Train the model

```bash
python src/train_salary_model.py
```

This will:
- Load and join all 10 sheets (jobs, categories, countries, job_skills, platforms).
- Build a unified target `Estimated_Hourly_Rate`:
  - **Hourly jobs** → use `Hourly_Rate` directly.
  - **Fixed Price jobs** → derive it as `avg(Budget_Min, Budget_Max) / estimated_hours`,
    where `estimated_hours = duration_days * 6` (an explicit, documented assumption —
    freelancers are assumed to spend ~6 productive hours/day on a given contract).
- **Exclude** `Applicants_Count`, `Proposals_Count`, `Hire_Rate`, `Job_Status` from the
  features — these are only known *after* a job has been posted and has already
  collected interaction data, so using them would leak future information into a
  price shown *before* a job is posted.
- Train a **Linear Regression baseline** and a **CatBoost** model (CatBoost is the
  main pick because it handles the many categorical columns — Experience_Level,
  Country, Category, Region, Platform_Type — natively, without one-hot blow-up).
- Run 5-fold cross-validation on CatBoost and print feature importances.
- Save `model/catboost_salary_model.cbm` and `model/model_meta.joblib`.

**Results on this dataset (110,550 jobs):**

| Model | MAE ($/hr) | RMSE | R² |
|---|---|---|---|
| Linear Regression (baseline) | 3.45 | 6.13 | 0.590 |
| **CatBoost** | **2.64** | **5.40** | **0.682** |

Top drivers: `Experience_Level` (62%), `Cost_Index` (17%), `Category_Name` (12%).

## 3. Run the API

```bash
uvicorn api.main:app --reload --port 8000
```

Swagger docs: http://localhost:8000/docs

Endpoints:
- `GET /health`
- `GET /model-info` — metrics, feature importance, excluded-leakage list
- `GET /lookups` — categories/countries/skills/durations for populating a UI
- `POST /predict-salary` — the real prediction endpoint

## 4. Open the frontend

Just open `predictor.html` in a browser (double-click it, or serve it with any
static server). It auto-detects whether the API is running at
`http://localhost:8000`:
- **API online** → real CatBoost predictions + real feature-importance bars.
- **API offline** → falls back to a simple demo formula so the UI still works,
  and shows an "OFFLINE — USING DEMO FORMULA" badge so it's never misleading.

If you deploy the API elsewhere, change `API_BASE` at the top of the
`<script>` block in `predictor.html`.

## Known limitations (carried over from the data itself)

- The dataset only has the **employer/job side** — no real candidate/graduate
  skill profiles yet, so the "Skill Gap" concept in the UI is a placeholder
  (suggests skills from high-value categories not yet in the user's list),
  not a true CV-vs-market gap analysis. Wire that up once graduate data exists.
- `Estimated_Hourly_Rate` for Fixed Price jobs is a **derived approximation**,
  not something Upwork/Fiverr-style platforms would report — it depends on the
  6-hours/day assumption in `src/features.py`. Change `HOURS_PER_DAY_ASSUMPTION`
  there if you have better evidence.
- Salary associations from this model are correlational, not causal — a skill
  being linked to a higher predicted rate doesn't mean picking it up **causes**
  a raise.
