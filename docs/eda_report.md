# EDA Report — Failytics Risk Engine

Exploratory data analysis of the LO2 Microservice Observability Dataset used to build the Failytics failure risk engine.

**Notebook:** [Failytics_EDA_and_Baseline.ipynb](../Failytics_EDA_and_Baseline.ipynb)

---

## Dataset Overview

| Property | Value |
|----------|-------|
| Source | LO2 Microservice Observability Dataset (Zenodo DOI: 10.5281/zenodo.14938118) |
| Service | light-oauth2 — production-grade OAuth2 server |
| Test runs | 100 |
| Scenarios per run | 54 (1 healthy + 53 failure types) |
| Metric rows | 17,200 |
| Log files | 37,800 (~16 GB) |
| Healthy rows | 1,300 (7.6%) |
| Failure rows | 15,900 (92.4%) |
| Raw metric columns | 1,128 per file |

Failure scenarios span: malformed OAuth2 requests (400), invalid credentials (401), missing resources (404), PKCE violations, and registration/update errors across all service types.

---

## Data Cleaning

- **All-NaN columns removed** — Prometheus registers metric names that are never populated for a given node configuration. These carry zero information and are dropped.
- **Zero-variance columns removed** — counters that stay constant across the entire dataset (e.g., node metrics for hardware not present on the host) add no discriminative signal.
- **Duplicate rows removed** — exact duplicates across all columns are dropped.
- **Missing value imputation** — any remaining numeric NaNs are filled with the column median, which is robust to the outliers that often define failure events.
- After cleaning: **17,200 rows, no missing values**. The 1,128-column space reduces substantially after dropping uninformative columns.

---

## Class Imbalance

The dataset has a **12:1 failure-to-healthy ratio**. The 53 failure types are roughly uniformly distributed (~300 rows each), while the healthy class has 1,300 rows.

This rules out accuracy as a meaningful evaluation metric — a trivial classifier that always predicts "failure" scores ~92% without learning anything. ROC-AUC is used as the primary evaluation metric instead.

---

## Metric Signals

### GC Latency
GC duration (p75, p100) is consistently elevated in failure scenarios. Garbage collection pressure appears to be a leading indicator — the service is spending more time in GC relative to normal operation, likely due to object allocation spikes triggered by exception handling paths.

### Goroutines and File Descriptors
Goroutine count and open file descriptor counts show wider variance during failure runs, suggesting resource contention or connection pool saturation under error conditions.

### Memory
Heap allocation and RSS memory distributions overlap substantially between healthy and failure classes at the raw snapshot level. The utilisation ratio (`heap_alloc / heap_sys`) is a more stable signal than absolute byte counts, which grow monotonically with uptime.

### HTTP Counters
The HTTP error rate derived from Prometheus scrape metadata (`promhttp_metric_handler_requests_total`) reflects the metrics endpoint's own request behaviour, not application traffic. Its discriminative value for binary classification is limited in the current form.

---

## Log Signals

### Error Rate
Log ERROR rate is the clearest single separator between healthy and failure classes in the entire dataset. Healthy scenarios produce near-zero ERROR-level log lines; all failure scenarios show meaningfully elevated ERROR rates ranging from ~0.5% to ~3% of total log lines.

### Exceptions
Exception occurrence counts mirror the ERROR rate pattern. Healthy execution paths complete without throwing Java exceptions. Failure scenarios generate structured stack traces (`ApiException`, `SQLIntegrityConstraintViolationException`, etc.) that appear as multi-line blocks in the logs.

### Error Codes
The service emits structured JSON error payloads inline in log messages, e.g.:

```
{"statusCode":404,"code":"ERR12014","message":"CLIENT_NOT_FOUND",...}
```

Error codes are scenario-specific:

| Error Code | Meaning | Concentrates in |
|------------|---------|-----------------|
| `ERR12014` | CLIENT_NOT_FOUND | `access_token_client_id_not_found_404` |
| `ERR12021` | user-related error | `register_user_*` failures |
| `ERR12039` | PKCE verification failure | `code_verifier_*` / `verification_failed_*` |
| `ERR12029` | REFRESH_TOKEN_NOT_FOUND | appears across most scenarios |
| `ERR10010` | generic handler error | appears across most scenarios |

Per-code counts are useful features for multi-class classification (identifying *which* failure occurred) but are somewhat redundant for binary prediction once the overall error rate is included.

### Log Volume
Total log line count is higher for the healthy scenario than for most failure types. Failed requests short-circuit early in the handler chain and produce fewer log lines overall, while successful flows exercise more code paths.

---

## Feature Engineering

Features are constructed from both data sources and fall into four categories:

| Category | Examples |
|----------|---------|
| **Metric unit conversions** | `heap_alloc_mb`, `rss_mb`, `gc_p50_ms`, `gc_p100_ms` |
| **Metric ratio features** | `heap_util_pct` = heap_alloc / heap_sys × 100, `http_error_rate` |
| **Log-derived features** | `log_error_rate`, `log_error_count`, `log_exceptions`, `log_unique_errcodes`, per-code counts |
| **Rolling-window features** | rolling mean, std, rate-of-change over 3, 5, 10-step windows — grouped by `run_end` to prevent leakage across test runs |

Rolling features capture *trajectory* rather than a point-in-time snapshot. A metric drifting upward over several consecutive scrapes is more diagnostic of emerging instability than any single reading.

---

## Baseline Model Results

**Model:** Logistic Regression with `class_weight='balanced'`, trained on 11 metric + 15 log features.

**Evaluation metric — ROC-AUC:** measures the probability that the model ranks a randomly chosen failure observation higher than a randomly chosen healthy one. It is threshold-independent and unaffected by the class imbalance magnitude, making it appropriate here.

| Metric | Value |
|--------|-------|
| CV ROC-AUC (5-fold) | reported in notebook |
| Test ROC-AUC | reported in notebook |
| Test Average Precision | reported in notebook |

**Feature importance highlights:**
- Log ERROR rate and exception count carry the highest positive coefficients among log features, confirming they add discriminative power beyond metrics alone.
- GC latency (p50, p100) and goroutine count are the dominant positive-coefficient metric features, consistent with the EDA observations.
- The model's probability output is directly usable as a continuous failure risk score.

---

## Next Steps

- Incorporate rolling-window log features (error rate trend over time, not just point-in-time count)
- Apply SMOTE oversampling on the training split to improve recall on the healthy minority class
- Replace Logistic Regression with gradient boosting (XGBoost/LightGBM) to capture non-linear interactions between metric and log signals
- Use time-aware cross-validation (`GroupKFold` on `run_end`) to prevent data leakage across test runs
