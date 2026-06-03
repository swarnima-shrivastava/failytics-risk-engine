# Failytics Risk Engine
**Predictive Failure Risk Intelligence for Multi-Agent and Distributed Systems**

`failytics-risk-engine` is an ML-powered engine that predicts the likelihood of service degradation or failure in multi-agent and distributed environments. By analysing patterns in logs and performance metrics over time, it surfaces emerging instability signals and quantifies short-term failure risk.

---

## Contents

| Artifact | Description |
|----------|-------------|
| [Failytics_EDA_and_Baseline.ipynb](Failytics_EDA_and_Baseline.ipynb) | Data exploration, cleaning, visualisations, and Logistic Regression baseline |
| [Failytics_Modeling_and_Findings.ipynb](Failytics_Modeling_and_Findings.ipynb) | Multi-model comparison, hyperparameter tuning, final evaluation, and findings |
| [docs/final_report.md](docs/final_report.md) | **Final project report** — problem, findings, and recommendations |
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
│       └── logs/           # Java service log files per run/scenario
├── docs/
│   ├── final_report.md
│   ├── eda_report.md
│   ├── capstone-problem-statement.md
│   └── glossary.md
├── Failytics_EDA_and_Baseline.ipynb
├── Failytics_Modeling_and_Findings.ipynb
├── LICENSE
└── README.md
```

---

## The Dataset

The project uses the **LO2 Microservice Observability Dataset**, publicly available on Zenodo:

> DOI: [10.5281/zenodo.14938118](https://doi.org/10.5281/zenodo.14938118)

It records the behaviour of the **light-oauth2** OAuth2 server across 100 controlled test runs,
each covering 54 scenarios (1 healthy + 53 failure types). For each scenario the dataset
provides Prometheus metric scrapes and structured Java service logs.

| Property | Value |
|----------|-------|
| Test runs | 100 |
| Scenarios per run | 54 |
| Rows (after cleaning) | 17,200 |
| Engineered features | 71 |

The data is **not included in this repository** (size). Download it before running the notebooks.

### Download

**Option A — command line**

```bash
pip install zenodo-get
zenodo_get 10.5281/zenodo.14938118 -o data/
```

**Option B — browser**

1. Go to [https://doi.org/10.5281/zenodo.14938118](https://doi.org/10.5281/zenodo.14938118)
2. Download the archive from the Files section
3. Extract so the directory tree matches the structure above

Verify the download:

```bash
ls data/lo2-sample/metrics/ | wc -l   # should print 100
```

---

## Setup

```bash
pip install pandas numpy matplotlib seaborn plotly scikit-learn imbalanced-learn jupyter
jupyter notebook
```

Requires Python 3.9+. Both notebooks must be run from the repository root.

Run **Notebook 1** first — it parses the log files and writes a cache
(`data/lo2-sample/log_features.parquet`) that Notebook 2 depends on.

---

## How the Notebooks Work

**Notebook 1 — [Failytics_EDA_and_Baseline.ipynb](Failytics_EDA_and_Baseline.ipynb)**

Loads all 100 metric CSVs, cleans the data, explores distributions and class imbalance,
analyses log signals, engineers features, and trains a Logistic Regression baseline.

**Notebook 2 — [Failytics_Modeling_and_Findings.ipynb](Failytics_Modeling_and_Findings.ipynb)**

Builds on Notebook 1's feature set to compare three classifiers (Logistic Regression,
Random Forest, Gradient Boosting) using GroupKFold cross-validation, tunes hyperparameters
with GridSearchCV, evaluates the best model on a held-out test set, and presents findings.

---

## Future Work

**Richer modelling**
- **Multi-class classification** — predict which of the 53 failure types occurred, not just whether a failure happened. The per-error-code log features already carry most of the discriminating signal needed for this.
- **Anomaly detection** — complement the supervised classifier with an unsupervised model (Isolation Forest, LOF) to flag novel failure modes not present in the training data.
- **Threshold calibration** — apply Platt scaling or isotonic regression so the risk score is a well-calibrated probability that matches observed failure rates in a target environment.

**Better evaluation**
- **Group-aware outer split** — use `GroupShuffleSplit` for the final train/test split so that entire test runs are held out end-to-end, removing any residual leakage from the current row-level split.
- **Longer time horizons** — the current dataset covers 2-minute controlled bursts. Evaluating on longer production traces would stress-test the rolling-window features and reveal how quickly risk scores degrade for novel workloads.

**Production readiness**
- **Streaming inference** — deploy the model as a sidecar that consumes live Prometheus scrapes and emits a continuous risk score, rather than scoring batches after the fact.
- **Cross-service generalisation** — re-evaluate the model on other microservices in the same stack to measure how much of the learned signal is service-specific versus broadly applicable.
- **Rolling log features** — add trailing log error rate (error trend over time) alongside the current point-in-time log count, for earlier detection of deteriorating conditions.
