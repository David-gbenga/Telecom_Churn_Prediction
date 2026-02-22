UK Telecoms Churn (Cease) Prediction & Retention Prioritisation
Project Overview
This project delivers a customer cease-risk prediction workflow for UK Telecoms LTD, designed to help the business prioritise retention efforts by identifying customers most likely to leave in the near term. The output is a ranked list of customers and risk bands that can be used by the retention team to target calls and interventions more effectively.
Data Scientist - Assessment
The solution is implemented in Python and designed to support both:
• Technical review (data science manager / engineers)
• Business review (non-technical stakeholders)

Business Objective
The business goal is to improve customer retention by focusing limited retention resources on customers with the highest likelihood of placing a cease (churn request). Rather than calling all customers, the team needs a data-driven prioritisation system that:
• Identifies customers likely to cease soon
• Ranks them by risk
• Supports targeted retention actions
• Improves retention outcomes per call / effort spent
Business-framed deliverable
A monthly churn-risk scoring pipeline that ranks active customers by likelihood of placing a cease in the next period, with explainable drivers for retention targeting.

Problem Definition
Prediction target
The primary modelling target is a binary label:
• target_cease_next_30d = 1 if a customer places a cease in the 30 days after a given snapshot date
• 0 otherwise
This 30-day horizon is selected because it is operationally useful for a retention team and allows timely intervention.
Why 30 days?
• Aligns with retention call cycles
• Balances urgency and model predictability
• Easy to explain to business stakeholders

Data Sources
The assessment provides four synthetic datasets, all keyed by unique_customer_identifier. These are combined to create a churn modelling dataset.
Data Scientist - Assessment

1. Cease Data
   Contains customer cease events and reasons.
   • unique_customer_identifier
   • cease_placed_date
   • cease_completed_date
   • reason_description
   • reason_description_insight
2. Customer Info (monthly snapshots)
   Customer profile, contract, tenure, and package information.
   • datevalue
   • contract_status
   • contract_dd_cancels
   • dd_cancel_60_day
   • ooc_days
   • Technology
   • speed
   • line_speed
   • sales_channel
   • crm_package_name
   • tenure_days
3. Call Information
   Contact centre interactions (friction / customer intent signals).
   • event_date
   • call_type (e.g., Loyalty, CS&B)
   • talk_time_seconds
   • hold_time_seconds
4. Usage Data (Parquet)
   Daily broadband usage behaviour (key churn signal).
   • calendar_date
   • usage_download_mbs
   • usage_upload_mbs

Analytical Approach
This solution is designed as a production-minded churn workflow, not just a one-off model.
Core design principles

1. Business-first framing (retention prioritisation, not just classification)
2. Leakage-safe feature engineering (features only from information known at prediction time)
3. Time-based validation (simulates real deployment)
4. Explainability (for technical and non-technical stakeholders)
5. Actionable outputs (risk ranking + retention bands)

End-to-End Workflow

1. Data ingestion
   Read CSV/Parquet datasets using pandas (and/or duckdb for larger parquet files).
2. Create customer snapshot table
   Use monthly customer snapshots (customer_info.datevalue) as prediction anchors.
   For each snapshot date:
   • Build features using data available up to that date
   • Create a label based on whether a cease is placed in the next 30 days
3. Feature engineering
   Generate predictive features from:
   • contract and tenure
   • direct debit cancellation behaviour
   • call centre interactions
   • broadband usage trends
   • (optional) historical cease behaviour if prior and leakage-safe
4. Target creation
   Create the binary target target_cease_next_30d.
5. Modelling
   Train a baseline and a stronger tabular model:
   • Baseline: Logistic Regression (interpretable)
   • Main model: LightGBM / XGBoost / CatBoost (recommended for performance on structured data)
6. Evaluation
   Use time-based train/validation/test splits and retention-relevant metrics:
   • ROC AUC
   • PR AUC
   • Recall @ Top K%
   • Precision @ Top K%
7. Scoring & prioritisation
   Score latest active customer base and produce:
   • Customer-level risk score
   • Risk rank
   • Risk band (High / Medium / Low)
8. Business output
   Generate a retention-ready file that can be used by operations:
   • retention_priority_list.csv
9. Monitoring (proposed production step)
   Track:
   • model performance drift
   • conversion/saves from retention calls
   • churn capture rate in top-K risk segments

Leakage Prevention Strategy (Critical)
This is a churn prediction task over time, so data leakage prevention is a core part of the design.
Rule
For any snapshot date T, only use data available on or before T.
Example
• Snapshot date: 2024-06-30
• Features built from data ≤ 2024-06-30
• Target = cease placed between 2024-07-01 and 2024-07-30
Leakage controls
• No future calls or usage used in features
• No use of cease reasons from the target period
• Historical cease features only if strictly before snapshot date
This ensures realistic performance estimates and production-ready logic.
Pasted text

Feature Engineering Plan
A) Customer / Contract Features (core state)
These features describe the customer’s current contract and service context and are expected to be strong churn predictors.
Directly used features
• contract_status
• contract_dd_cancels
• dd_cancel_60_day
• ooc_days
• Technology
• speed
• line_speed
• sales_channel
• crm_package_name
• tenure_days
Derived features
• speed_gap = speed - line_speed
• speed_ratio = line_speed / speed
• is_out_of_contract = (ooc_days > 0)
• days_to_contract_end = abs(ooc_days) when ooc_days < 0
• is_near_contract_end = -30 <= ooc_days <= 0
• dd_cancel_flag = (dd_cancel_60_day > 0)

B) Call Centre Features (friction / retention intent)
Call behaviour often signals dissatisfaction before churn. Features are built using rolling windows (e.g., last 7/30/90 days).
Rolling features
• calls_7d, calls_30d, calls_90d
• loyalty_calls_30d (retention team calls)
• csb_calls_30d
• avg_talk_time_30d
• avg_hold_time_30d
• total_hold_time_30d
• hold_ratio_30d
• repeat_call_count_30d
Behavioural signals captured
• spikes in recent calls
• multiple repeated contacts
• high hold times
• calls to loyalty/retention

C) Usage Features (behavioural decline signals)
Usage behaviour is often a leading indicator of churn intent. Features are built from daily usage data with rolling windows.
Aggregated features
• download_7d, download_30d, download_90d
• upload_30d, upload_90d
• avg_daily_download_30d
• usage_trend_30d_vs_prev30d
• usage_volatility_30d
• zero_usage_days_30d
• weekend_vs_weekday_usage_ratio
Behavioural patterns captured
• sudden reduction in usage
• sustained low/zero activity
• abnormal usage pattern shifts

D) Historical Cease Features (if available, leakage-safe)
Only historical cease behaviour prior to snapshot date is considered.
Potential features:
• prior_cease_count
• days_since_last_cease
• prior_cease_reason_group
Important: Current/future cease information is never used as a feature.

Exploratory Data Analysis (EDA)
The EDA in this project is intentionally targeted and business-focused, not generic.
Key analyses included

1. Churn rate by contract_status
2. Churn rate by ooc_days buckets
3. Churn rate by direct debit cancellation behaviour
4. Churn rate by call volume and loyalty call activity
5. Churn rate by usage decline patterns
6. Churn rate by technology / package / sales channel
   Why this matters
   These views provide immediate, explainable insights to business teams and inform feature engineering and intervention strategies.

Modelling Strategy
Model selection
This project uses a baseline + advanced model approach.
Baseline (interpretability)
• Logistic Regression
• Provides an interpretable benchmark
Main model (performance)
• LightGBM / XGBoost / CatBoost (final choice depends on runtime and categorical handling)
• Chosen for:
o strong tabular performance
o non-linear relationships
o interaction effects
o robustness on mixed feature types

Validation Strategy
Time-based split (not random split)
Because this is a temporal churn problem, random split would overstate performance.
Recommended split
• Train: earlier months
• Validation: later month(s)
• Test: most recent month
This mirrors real deployment and gives a more realistic estimate of how the model will perform on future customers.

Evaluation Metrics (Retention-Oriented)
Accuracy is not sufficient for this use case.
Primary metrics
• ROC AUC — overall ranking quality
• PR AUC — useful when churn is relatively rare
• Recall @ Top K% — how many churners are captured in the top risk segment
• Precision @ Top K% — how efficient retention calls are
Business metric framing
If the retention team can call only the top 10% of customers:
• What percentage of actual churners are captured in that top 10%?
This is the most operationally relevant performance measure.

Explainability
Model explainability is built into the workflow to support trust and actionability.
Methods
• Feature importance
• SHAP values (global and local where feasible)
Example explainability outcomes
Typical high-risk drivers may include:
• Out of contract status
• Direct debit cancellations
• Recent loyalty calls
• Decline in usage
• High repeat call frequency
These insights help both technical and non-technical stakeholders understand why a customer is flagged high risk.

Business Decisioning Output
The model output is converted into an operational retention strategy.
Risk bands (example)
• High Risk (top 5–10%): immediate retention call
• Medium Risk (next 10–20%): targeted offer / SMS / email
• Low Risk: passive monitoring
Final output file
retention_priority_list.csv includes (example):
• unique_customer_identifier
• snapshot_date
• risk_score
• risk_rank
• risk_band
• (optional) top reason codes / SHAP drivers
