# Failytics – Capstone Problem Statement  
UC Berkeley Professional Certificate in AI & ML  

## 1. Overview of the Question

Can short-term service failure risk in multi-agent or distributed systems be predicted using historical telemetry signals such as logs and performance metrics?

This project aims to develop a supervised machine learning model that estimates the probability of service degradation or failure within a defined prediction horizon.


## 2. Data Required

The project will use the publicly available LO2 Microservice Observability Dataset (Zenodo DOI: 10.5281/zenodo.14938118), which contains:

- Service-level logs  
- Performance metrics (e.g., latency, error rates, resource utilization)  
- Timestamped execution traces  
- Labeled anomalous or erroneous events  

Telemetry signals will be aggregated into fixed time windows to construct model features. Binary failure labels will be defined based on whether error-tagged events occur within a specified prediction horizon.

## 3. Proposed Techniques

1. **Time-Windowed Feature Engineering**  
   Since failure risk is inherently temporal, raw logs and metrics will be transformed into fixed-length observation windows per service (e.g., last *N* minutes). Within each window, we will compute summary statistics and trend features (mean/variance, percentiles, deltas/slopes) and log-derived features (template/severity frequencies) to capture early instability patterns while keeping the representation schema-agnostic.

2. **Supervised Failure Risk Modeling with Class Imbalance Handling**  
   The prediction task is framed as supervised binary classification: using an observation window to estimate the probability of an error event within a future prediction horizon (next *M* minutes). A Logistic Regression baseline will provide a simple benchmark, followed by Gradient Boosting (XGBoost/LightGBM) to model non-linear interactions typical in complex systems. Because failure events are relatively rare, SMOTE (or an equivalent resampling strategy) will be applied *only to the training split* to mitigate class imbalance and improve recall without introducing evaluation leakage.

3. **Operational Evaluation + Explainability for Actionable Risk Scores**  
   Model performance will be evaluated using Precision, Recall, F1, and PR-AUC (appropriate for imbalanced failure prediction), along with Top-*K* ranking accuracy to reflect operational triage workflows. To make risk scores actionable, feature importance or SHAP-based explanations will be used to surface the primary telemetry drivers behind high-risk predictions.

