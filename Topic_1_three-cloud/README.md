# Regress-It — Three-Cloud Reference Template

> A working, forkable template for a three-cloud architecture:
> **Streamlit (UI) ⇄ FastAPI (Model API) ⇄ Supabase (Data)**. Fork it, wire up
> your own accounts, and reuse the exact same pattern for the other templates in
> this repository.

## Live deployment URLs (fill these in)

| Tier | Platform | URL |
|------|----------|-----|
| **UI** | Streamlit Community Cloud | https://cst435topic1-3gxcnfplqnumgopreogutc.streamlit.app/#model-card |
| **API** | Render.com | https://regress-it-api-w388.onrender.com/healthz |
| **Data** | Supabase | https://iiydxtkgqwpbcrqvrfyl.supabase.co https://supabase.com/dashboard/project/iiydxtkgqwpbcrqvrfyl |

---

## Christian Worldview
I think a data scientist has a responsibility to be honest with a client about the limitations of a regression model. Not every model is going to be perfect, and there can be problems with the data or the way the model makes predictions. Since a non-technical client may not understand those details, it is important to explain them in a way that makes sense instead of just giving them the results and expecting them to understand.

From a Christian worldview, honesty is very important. I believe we should be truthful with people and take responsibility for how we use the information and skills we have. In this situation, that means being honest about what the model can do and where it may not be reliable. Even if the limitations make the model seem less impressive, the client deserves to know the full picture so they can make good decisions.

To me, being a good data scientist is not just about building a model that works. It is also about making sure the model is being used responsibly. Being honest about the limitations of the model helps build trust and shows good stewardship of the data and technology we are working with.

##Decision Justifications
One of the main design decisions in my project was the learning rate. I tested different learning rates to see how they affected the training process. A learning rate of 0.01 consistently produced a model that converged successfully, with MSE values around 4.13 to 4.44, MAE values around 1.60 to 1.71, and R² values close to 0.98. I also tested a much larger learning rate of 1.5, which caused the model to diverge. The results for that run reached non-finite values, which my application records as 1,000,000 so the run can still be saved and displayed. This showed me how using a learning rate that is too large can make the training unstable.

My stopping criterion was 100 epochs. I used the same number of epochs for each run so that I could compare different learning rates under the same training conditions. This makes it easier to see whether the learning rate is helping the model converge instead of giving one model more training time than another.

For the validation split, I used 20% of the dataset as held-out data, with the other 80% used for training. The model's MSE, MAE, and R² are calculated using the held-out data. This gives a better idea of how the model performs on data that was not used during training instead of only measuring how well it fits the training data.

The run-history table makes these comparisons easier because all of the important information is displayed together. I can compare the learning rate, batch size, epochs, MSE, MAE, and R² for each training run. For example, the table makes it easy to see that the runs using a learning rate of 0.01 converged successfully while the run using 1.5 diverged. This gives a clear visual comparison of how the different training settings affected the model.

##Screenshot showing convergence and divergence
<img width="2990" height="810" alt="image" src="https://github.com/user-attachments/assets/c406e893-a8e2-43cb-a898-d04a67713039" />




## What it does

Regress-It is an interactive teaching demo for 1-D linear regression. You pick a
learning rate, batch size, and epoch count; the API trains `y = w·x + b` with
PyTorch mini-batch SGD on synthetic data, reports held-out **MSE / MAE / R²**,
and persists every run. The UI lets you visualise convergence, make predictions,
and browse run history.

## Architecture

```
┌──────────────────────┐   HTTPS/JSON    ┌──────────────────────┐   service-role   ┌──────────────────┐
│  Streamlit Cloud     │ ──────────────► │  FastAPI on Render   │ ───────────────► │  Supabase        │
│  (ui/app.py)         │                 │  (api/main.py)       │   full access    │  Postgres        │
│  thin client, no ML  │                 │  PyTorch training    │                  │  datasets/runs/  │
│                      │ ◄────anon key,  │                      │                  │  predictions     │
│                      │   read-only ────┼─────────────────────┼──────────────────►│  (RLS: anon can  │
└──────────────────────┘   SELECT runs   └──────────────────────┘                  │   only SELECT)   │
                                                                                    └──────────────────┘
```

- **UI never touches the model or writes SQL.** It calls the API over HTTPS and
  performs one read-only `SELECT` on `runs` with the anon public key.
- **API owns the model and all writes**, using the Supabase **service-role** key.
- **Supabase is the single source of truth** for datasets, runs, and predictions.

See [`TUTORIAL.md`](./TUTORIAL.md) for the full step-by-step build and deploy guide.

## Project structure

```
three-cloud/
├── TUTORIAL.md               # Full build + deploy guide (start here)
├── README.md                 # This file
├── MODEL_CARD.md             # Model details, intended use, limitations
├── shared/                   # Code shared by both tiers
│   ├── schemas.py            # Pydantic API contract
│   └── data.py               # Synthetic linear data generator
├── api/                      # FastAPI tier (deploys to Render)
│   ├── main.py               # Endpoints
│   ├── training.py           # PyTorch linear regression
│   ├── db.py                 # Supabase (service-role) data access
│   ├── configs/default.yaml  # Default hyperparameters
│   └── requirements.txt
├── ui/                       # Streamlit tier (deploys to Streamlit Cloud)
│   ├── app.py                # 5-tab thin client
│   ├── requirements.txt      # No torch
│   └── .streamlit/secrets.toml.example
├── db/                       # Database tier (Supabase)
│   ├── migrations/001_init.sql
│   └── seed.py
├── tests/                    # pytest suite
├── render.yaml               # Render blueprint
├── requirements-dev.txt      # Both tiers + pytest (local dev)
└── .env.example
```

## Quickstart (local)

```bash
cd three-cloud

# 1. Install everything (both tiers + test tools)
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt

# 2. Run the tests (6 pass; the live-Supabase test skips without creds)
pytest -q

# 3. Configure secrets
cp .env.example .env                                   # API: SUPABASE_URL + SERVICE key
cp ui/.streamlit/secrets.toml.example ui/.streamlit/secrets.toml

# 4. Run the API
uvicorn api.main:app --reload --port 8000

# 5. In another terminal, run the UI
streamlit run ui/app.py
```

To deploy to the three clouds, follow **Part E** of [`TUTORIAL.md`](./TUTORIAL.md):
apply `db/migrations/001_init.sql` in the Supabase SQL Editor → deploy the API
from `render.yaml` on Render → deploy the UI on Streamlit Community Cloud.

## API endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/datasets` | Create a synthetic dataset |
| `POST` | `/train` | Train a model, persist the run, return metrics |
| `GET`  | `/runs/{run_id}` | Fetch one run |
| `GET`  | `/runs` | List recent runs |
| `POST` | `/predict` | Predict `ŷ` for an `x` using a fitted run |
| `GET`  | `/healthz` | Liveness / DB ping |
| `GET`  | `/version` | Build SHA + framework versions |

## Reusing this pattern

The three-cloud split and the file layout stay identical for every product. Swap
only the model in `api/training.py`, the Pydantic contract in `shared/schemas.py`,
the tables in `db/migrations/`, and the UI tabs — the UI stays a thin client and
Supabase stays the single source of truth. Two worked examples
(`income-insight`, `see-sense`) live alongside this one; see the
final section of [`TUTORIAL.md`](./TUTORIAL.md).

## Checklist

- [ ] Three live URLs listed at the top of this README
- [ ] `datasets`, `runs`, `predictions` tables in Supabase with RLS
- [ ] 6+ API endpoints
- [ ] 5 Streamlit tabs (Concepts, Train, Predict, Run History, Model Card)
- [ ] PyTorch training with held-out MSE/MAE/R²
- [ ] pytest suite passing
- [ ] `MODEL_CARD.md` completed
