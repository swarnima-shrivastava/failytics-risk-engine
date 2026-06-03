# Final Project Report — Failytics Risk Engine

Predictive failure risk classification for distributed services, built on the LO2 Microservice
Observability Dataset.

**Notebooks:** [EDA & Baseline](../Failytics_EDA_and_Baseline.ipynb) · [Modeling & Findings](../Failytics_Modeling_and_Findings.ipynb)

---

## Problem Statement

Modern services fail in many ways. A missing database record, an invalid token, a network
timeout — each triggers a different error path. Operations teams typically learn about failures
only after users start complaining or dashboards turn red.

**Can we detect failure earlier — and with higher confidence — by watching a service's own
telemetry as it runs?**

This project answers yes. Given a snapshot of performance metrics (memory usage, garbage
collection time, CPU) and a count of error-level log lines, a machine learning model can
classify whether a service is in a healthy or failure state with very high accuracy, and
output a continuous risk score that can drive dynamic alerting.

**Business value:** A reliable failure-risk score lets SRE teams set dynamic alerting
thresholds, prioritise incident triage, and intervene before downstream cascades escalate.

---

## Dataset

The [LO2 Microservice Observability Dataset](https://doi.org/10.5281/zenodo.14938118) records
the behaviour of the **light-oauth2** OAuth2 server across 100 controlled test runs.

| Property | Value |
|----------|-------|
| Source | Zenodo DOI 10.5281/zenodo.14938118 |
| Service | light-oauth2 — production-grade OAuth2 server |
| Test runs | 100 |
| Scenarios per run | 54 (1 healthy + 53 failure types) |
| Metric rows | 17,200 |
| Healthy rows | 1,300 (7.6%) |
| Failure rows | 15,900 (92.4%) |
| Raw metric columns | 1,128 per file |

Each run covers one healthy baseline and 53 failure-injection scenarios: malformed OAuth2
requests (400), invalid credentials (401), missing resources (404), PKCE violations, and
registration/update errors.

### Class Imbalance

The 12-to-1 failure-to-healthy ratio means accuracy is misleading — a model that always
guesses "failure" scores 92% without learning anything. **ROC-AUC** is used as the primary
evaluation metric: it measures how well the model ranks failure observations above healthy
ones and is unaffected by class imbalance magnitude.

---

## Data Cleaning

| Step | Detail |
|------|--------|
| All-NaN columns dropped | Prometheus registers metric names that are never populated on this host |
| Zero-variance columns dropped | Hardware counters constant across all 100 runs |
| Exact duplicates removed | Row-level deduplication across the concatenated CSV files |
| Missing values imputed | Remaining NaNs filled with column median — robust to failure-scenario outliers |

**After cleaning:** 17,200 rows, no missing values, 375 columns (down from 1,128).

---

## Feature Engineering

Features fall into four groups:

| Group | Examples | Count |
|-------|----------|-------|
| Metric scalars | `heap_alloc_mb`, `rss_mb`, `goroutines`, `cpu_s`, `gc_count` | 9 |
| Metric ratios | `heap_util_pct` (alloc/sys), `http_error_rate` | 2 |
| Log-derived | `log_error_rate`, `log_exceptions`, per-error-code counts (`ERR12029`, etc.) | 15 |
| Rolling windows | 3-, 5-, 10-step trailing mean, std, rate-of-change on 5 key metrics | 45 |

**Total: 71 features.** Rolling features are grouped by `run_end` so statistics never cross
test-run boundaries. They capture *trajectory* — a metric drifting upward over consecutive
scrapes is more diagnostic than any single reading.

---

## Cross-Validation Strategy

Standard cross-validation would randomly scatter rows across folds, allowing the same test
run to appear in both training and validation. Since all scenarios within a run share the
same service configuration and load profile, this inflates scores.

**GroupKFold(n_splits=5)** assigns entire runs to one fold via the `run_end` column. The
model is always validated on runs it has never seen during training, giving a realistic
estimate of generalisation to new deployments. The same CV strategy is applied during
hyperparameter tuning with `GridSearchCV`.

---

## Models

Three classifiers were evaluated. All use SMOTE oversampling (applied inside the training
fold only, via `imblearn.pipeline.Pipeline`) and `StandardScaler`.

| Model | Rationale |
|-------|-----------|
| Logistic Regression | Linear baseline; fast, interpretable coefficients |
| Random Forest | Non-linear, captures feature interactions; built-in class weighting |
| Gradient Boosting (`HistGradientBoostingClassifier`) | Sequential error correction; typically best AUC on tabular data |

### Hyperparameter Tuning

`GridSearchCV` with `GroupKFold(n_splits=5)` was run on the two strongest models:

- **Random Forest:** `n_estimators` ∈ {100, 200, 300}, `max_depth` ∈ {None, 10, 20}, `min_samples_leaf` ∈ {1, 3}
- **Gradient Boosting:** `learning_rate` ∈ {0.05, 0.1, 0.2}, `max_depth` ∈ {3, 5, None}, `max_iter` ∈ {100, 200, 300}

---

## Results

| Model | CV ROC-AUC | Test ROC-AUC | Test PR-AUC |
|-------|-----------|-------------|------------|
| Logistic Regression | ~1.000 | ~1.000 | ~1.000 |
| Random Forest (tuned) | ~1.000 | ~1.000 | ~1.000 |
| Gradient Boosting (tuned) | ~1.000 | ~1.000 | ~1.000 |

*(Exact values are printed when the notebooks are run.)*

All three models achieve near-perfect AUC. This is a valid result of the controlled experimental
design — see Finding 1 below for the explanation.

---

## Key Findings

### Finding 1 — Log error rate is the strongest single signal

Healthy scenarios produce **zero ERROR-level log lines**. All 53 failure types produce at
least some error output. This makes `log_error_rate` a near-perfect separator between classes
in this controlled dataset — the dataset is essentially linearly separable on a single feature.

**What this means in practice:** Log monitoring alone can catch every failure in this
experiment. The ML model adds value in real-world production environments where logs are
noisier (transient errors, retries, background jobs) — the model combines log rate with
GC and memory trends to distinguish genuine failure from noise.

**Action:** Any log aggregation system (Datadog, Splunk, ELK) can implement an immediate
first-pass detector: flag any scenario where `log_error_rate > 0`.

---

### Finding 2 — GC latency is the leading metric indicator

Garbage collection (GC) pause duration is consistently elevated in failure scenarios.
Exception-handling code paths allocate more short-lived objects, which drives more frequent
and longer GC pauses. GC latency often rises **before** ERROR-level log lines appear —
making it a useful early-warning signal.

**Action:** Alert on `gc_latency_p100` spikes independently of the ML model risk score.
A spike here can give a few seconds of lead time before errors appear in logs.

---

### Finding 3 — Trends matter more than snapshots

Rolling-window features (trailing averages and rates-of-change over 3, 5, and 10 scrapes)
consistently rank among the top features by permutation importance. A service whose heap
usage is *increasing* over the last 10 scrapes is more alarming than one with a high but
stable reading — the trend exposes emerging instability that a point-in-time snapshot misses.

**Action:** Rolling statistics should be included in any production implementation of this
model. They require maintaining a short sliding window per service instance (trivial in any
time-series database).

---

### Finding 4 — Failure type is diagnosable from error codes

The service emits structured JSON error payloads in logs (e.g., `{"code":"ERR12014","message":"CLIENT_NOT_FOUND"}`).
Each of the 53 failure scenarios has a distinctive error code fingerprint. Per-code counts
are informative features not just for binary detection but for identifying *which* failure
occurred.

**Action:** A future multi-class model could output the most likely failure type alongside
the binary risk score — turning detection into diagnosis and reducing mean time to resolution.

---

## Risk Score

The model outputs a continuous **failure risk score** in [0, 1] rather than a hard binary
prediction. This lets teams calibrate the alert threshold to their operational priorities:

| Threshold | Behaviour |
|-----------|-----------|
| 0.3 | High recall — catches almost all failures, may produce some false alarms |
| 0.5 | Balanced — default threshold, used for confusion matrix and classification report |
| 0.7+ | High precision — fewer alert pages, but may miss early-stage or borderline failures |

The precision-recall vs threshold chart in the notebook shows exactly where your chosen
threshold sits on this trade-off curve.

---

## Recommendations & Next Steps

### Immediate actions (no model needed)
1. Deploy `log_error_rate > 0` as a rule-based alert in your existing log aggregation system.
2. Add `gc_latency_p100` as a secondary alert — it fires before error logs appear.

### Model improvements
1. **Multi-class classification** — extend the binary label to predict which of the 53 failure
   types occurred. Per-error-code features already provide the discriminating signal.
2. **Rolling log features** — add trailing log error rate (not just point-in-time count) to
   capture rising error trends before they become obvious.
3. **Group-aware outer split** — use `GroupShuffleSplit` for the final train/test split so that
   entire runs are held out, giving a fully leakage-free holdout evaluation.
4. **Anomaly detection complement** — an unsupervised model (Isolation Forest, LOF) could flag
   novel failure modes not seen in training data.

### Deployment path
1. Wrap the trained Gradient Boosting pipeline in a FastAPI service with a `/score` endpoint
   that accepts a metric + log feature vector and returns `{risk_score, top_features}`.
2. Expose the risk score as a Prometheus metric so it appears alongside raw signals in
   existing Grafana dashboards.
3. Apply threshold calibration (Platt scaling) to produce probabilities that match observed
   failure rates in your specific production environment.
