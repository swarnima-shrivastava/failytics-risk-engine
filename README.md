# Failytics Risk Engine
**Predictive Failure Risk Intelligence for Multi-Agent and Distributed Systems**

`failytics-risk-engine` is an ML-powered engine that predicts the likelihood of service degradation or failure in multi-agent and distributed environments. By analysing patterns in logs and performance metrics over time, it surfaces emerging instability signals and quantifies short-term failure risk, enabling teams to intervene before incidents escalate.

---

## Contents

| Artifact | Description |
|----------|-------------|
| [Failytics_EDA_and_Baseline.ipynb](Failytics_EDA_and_Baseline.ipynb) | EDA and baseline model notebook |
| [docs/eda_report.md](docs/eda_report.md) | EDA findings and baseline model results |
| [docs/capstone-problem-statement.md](docs/capstone-problem-statement.md) | Project proposal |
| [docs/glossary.md](docs/glossary.md) | Term definitions |

---

## Project Structure

```
failytics-risk-engine/
├── data/
│   └── lo2-sample/
│       ├── metrics/        # 100 Prometheus scrape CSVs (one per test run)
│       └── logs/           # Java service logs per run/scenario
├── docs/
│   ├── eda_report.md
│   ├── capstone-problem-statement.md
│   ├── glossary.md
│   └── README.md
├── Failytics_EDA_and_Baseline.ipynb
├── LICENSE
└── README.md
```

---

## Getting the Data

The dataset is **not included in this repository** (excluded via `.gitignore` due to size). It is publicly available on Zenodo:

> **LO2 Microservice Observability Dataset**
> DOI: [10.5281/zenodo.14938118](https://doi.org/10.5281/zenodo.14938118)

### Download steps

**Option A — browser**

1. Go to [https://doi.org/10.5281/zenodo.14938118](https://doi.org/10.5281/zenodo.14938118)
2. Download the dataset archive from the Files section
3. Extract it and place the contents so the directory tree matches:

```
failytics-risk-engine/
└── data/
    └── lo2-sample/
        ├── metrics/       ← 100 CSV files named light-oauth2-data-{timestamp}.csv
        └── logs/          ← per-run, per-scenario service log files
```

**Option B — command line**

```bash
# Install the Zenodo download helper (optional but convenient)
pip install zenodo-get

# Download into a local 'data/' directory
zenodo_get 10.5281/zenodo.14938118 -o data/
```

Or with `wget` directly (replace `<file-url>` with the URL shown on the Zenodo page):

```bash
wget -P data/ <file-url>
```

After extraction, confirm the expected structure exists before running the notebook:

```bash
ls data/lo2-sample/metrics/ | wc -l   # should print 100
```

---

## Setup

```bash
pip install pandas numpy matplotlib seaborn plotly scikit-learn jupyter
jupyter notebook Failytics_EDA_and_Baseline.ipynb
```

Requires Python 3.9+. The notebook must be run from the repository root.

---

## How the Notebook Reads and Prepares the Data

The notebook handles data in two stages, both clearly marked:

**Stage 1 — Loading raw CSVs ([Section 2, cells 2–3](Failytics_EDA_and_Baseline.ipynb))**

All 100 per-run metric CSVs are loaded from `data/lo2-sample/metrics/` using `glob`, concatenated into a single dataframe (`df_raw`), and sanity-checked on the key identifier columns (`timestamp`, `run_end`, `test_name`).

**Stage 2 — Feature engineering for model training ([Section 5, cells 5–7](Failytics_EDA_and_Baseline.ipynb))**

Raw Prometheus counters are transformed into a model-ready feature matrix (`feat`). This includes:
- Unit-normalised scalars (bytes → MB, seconds → ms)
- Ratio features (`heap_util_pct`, `http_error_rate`)
- Rolling-window statistics (mean, std, rate-of-change over 3/5/10 steps, grouped by `run_end` to avoid leakage across test runs)

The final `X` (feature matrix) and `y` (binary label) arrays that are passed directly to `train_test_split` and the sklearn `Pipeline` are assembled in **[Section 6, cell 1](Failytics_EDA_and_Baseline.ipynb)**.

