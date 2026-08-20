 Mobile Money Fraud Detector

AI/ML Capstone Project — Intermediate Level**
Detecting anomalous mobile-money transactions for Nigerian agent-banking networks.

---

1. Problem Context

Nigeria's mobile-money and agent-banking ecosystem (OPay, PalmPay, Moniepoint, MTN MoMo,
bank USSD agents, etc.) processes millions of transactions daily — cash-in, cash-out,
wallet transfers, airtime/data purchases, and bill payments. Agents and platforms are
routinely targeted by fraud patterns such as:

Agent float draining — a compromised customer or agent PIN is used to withdraw
  almost the entire balance in a single cash-out.
SIM-swap / account takeover — after gaining control of a victim's line, fraudsters
  fire off a rapid burst of transfers within minutes, often at odd hours.
Smurfing / structuring — splitting a large transfer into several smaller ones,
  each just under common reporting/limit thresholds, to avoid detection.
Rogue/compromised agents — a small number of agents processing an abnormally high
  volume of large cash-outs relative to their normal profile.

Manually reviewing every transaction is impossible at scale — this project builds a
machine-learning system that automatically scores each transaction for fraud risk.

 2. What Was Built (MVP)

An end-to-end fraud-detection pipeline that takes a mobile-money transaction as input
and returns:

| Output | Description |
|---|---|
| Fraud Flag | Binary 0/1 decision from a supervised classifier |
| Fraud Probability | Calibrated probability (0–1) of the transaction being fraudulent |
| Anomaly Score | 0–100 unsupervised anomaly score (works even without labels) |
| Risk Level | LOW / MEDIUM / HIGH, for easy triage by a fraud/risk team |
| Evaluation Metrics | Precision, Recall, F1, ROC-AUC, PR-AUC, confusion matrix |

Two complementary models were trained and compared:

1. Isolation Forest (unsupervised) — flags anomalies without needing fraud labels;
   useful for catching novel fraud patterns not yet seen in historical data.
2. Random Forest Classifier (supervised) — learns known fraud typologies from
   labelled historical data; much higher precision/recall once labels exist.

 3. Repository Structure


mobile_money_fraud/
├── README.md                          <- this file
├── requirements.txt
├── data/
│   └── mobile_money_transactions.csv  <- synthetic dataset (61,078 transactions)
├── notebooks/
│   └── Mobile_Money_Fraud_Detector.ipynb   <- full walkthrough notebook (EDA → model → eval)
├── src/
│   ├── generate_data.py               <- synthetic data generator (fraud typologies)
│   ├── train_and_evaluate.py          <- feature engineering + training + evaluation
│   ├── inference.py                   <- MVP: score a single new transaction
│   └── build_notebook.py              <- (dev utility) builds the .ipynb deliverable
├── models/
│   ├── random_forest_fraud_model.joblib
│   ├── isolation_forest_anomaly_model.joblib
│   ├── feature_scaler.joblib
│   └── feature_columns.json
└── outputs/
    ├── evaluation_metrics.json
    ├── sample_scored_transactions.csv
    ├── fraud_rate_by_type.png
    ├── roc_curve.png
    ├── precision_recall_curve.png
    ├── confusion_matrix.png
    ├── feature_importance.png
    └── anomaly_score_distribution.png
```

 4. Dataset

Real mobile-money transaction logs are proprietary and cannot be shared publicly, so this
project uses a **synthetic dataset generator** (`src/generate_data.py`) that mimics the
structure of well-known mobile-money fraud research datasets (in the spirit of the
academic PaySim simulator), adapted to the Nigerian agent-banking context, with four
realistic fraud typologies injected at a **realistic ~1.8% fraud prevalence**:

| Transactions | Count |
|---|---|
| Total | 61,078 |
| Fraudulent | 1,078 (1.77%) |
| Legitimate | 60,000 |

> To use real data instead: replace `data/mobile_money_transactions.csv` with an
> anonymized export containing the same columns (`transaction_id`, `timestamp`,
> `customer_id`, `agent_id`, `transaction_type`, `channel`, `amount`, `old_balance`,
> `new_balance`, `is_fraud`) and re-run `train_and_evaluate.py`.

 5. Feature Engineering

Raw fields alone (amount, balances, type) aren't enough — fraud shows up in **behavior**:

- Balance-consistency: `balance_error`, `amount_to_balance_ratio`, `drains_account`,
  `zero_after` — catch float-draining and account-takeover patterns.
- Velocity: `seconds_since_last_txn`, `is_rapid_repeat`, `txns_last_24h` (rolling
  24h count per customer) — catch SIM-swap bursts.
- Time: `hour`, `is_night`, `is_weekend` — fraud skews toward unusual hours.
- Agent behavior: `agent_avg_amount`, `agent_txn_count`, `amount_vs_agent_avg` —
  catch rogue/compromised agents.
- Threshold proximity: `near_threshold` — catches structuring/smurfing just under
  the ₦100,000 reporting limit.

 6. Results

| Metric (held-out test set, 15,270 txns) | Random Forest (supervised) | Isolation Forest (unsupervised) |
|---|---|---|
| ROC-AUC | **0.9998** | 0.8629 |
| PR-AUC (Average Precision) | **0.9927** | 0.3203 |
| Precision (fraud class) | 0.911 | 0.413 |
| Recall (fraud class) | 0.989 | 0.441 |
| F1 (fraud class) | 0.948 | 0.427 |

Confusion matrix (Random Forest, threshold = 0.5):

|  | Predicted Legit | Predicted Fraud |
|---|---|---|
| Actual Legit | 14,974 | 26 |
| Actual Fraud | 3 | 267 |

Out of 270 fraudulent transactions in the test set, the model correctly caught 267
(98.9% recall) while only misflagging 26 out of 15,000 legitimate transactions
(0.17% false-positive rate).

Top predictive features: `new_balance`, `amount_to_balance_ratio`, `balance_error`,
`amount`, `drains_account`, `seconds_since_last_txn` — confirming the model learned the
exact behavioral signals fraud investigators would look for.

See `outputs/evaluation_metrics.json` for full numeric detail and the `.png` charts for
visuals (ROC curve, precision-recall curve, confusion matrix heatmap, feature importance,
anomaly-score distribution, fraud-rate-by-type bar chart).

Recommended production setup:** use the Random Forest as the primary fraud
flag/score, and run the Isolation Forest in parallel as an anomaly monitor — any
transaction it scores as highly anomalous but the supervised model scores low gets
routed to human review, since it may represent an emerging fraud pattern not yet in
the training data.

 7. How to Run

 Option A — Google Colab (recommended, matches suggested tools)
1. Upload the `mobile_money_fraud/` folder to your Google Drive (or clone the repo into
   Colab).
2. Open `notebooks/Mobile_Money_Fraud_Detector.ipynb` in Colab.
3. Run all cells top to bottom (adjust the data path if needed — Colab's working
   directory differs from a local clone).

 Option B — Local Python environment
```bash
git clone <your-repo-url>
cd mobile_money_fraud
pip install -r requirements.txt

 1. Generate the synthetic dataset
python3 src/generate_data.py

 2. Engineer features, train both models, evaluate, save outputs
python3 src/train_and_evaluate.py

 3. Score a new transaction (MVP inference demo)
python3 src/inference.py
```

Scoring a single transaction (the MVP contract)
```python
from src.inference import load_artifacts, score_transaction

rf_model, iso_forest, scaler, feature_cols = load_artifacts()

txn = {
    "transaction_id": "TXN123456",
    "amount": 148500,
    "old_balance": 150000,
    "new_balance": 1500,
    "transaction_type": "CASH_OUT",
    "channel": "AGENT_POS",
    "hour": 2,                      # 2am
    "seconds_since_last_txn": 90,   # rapid repeat
}

result = score_transaction(txn, rf_model, iso_forest, scaler, feature_cols)
print(result)
{'transaction_id': 'TXN123456', 'fraud_flag': 1, 'fraud_probability': 0.5887,
  'anomaly_score': 68.35, 'risk_level': 'MEDIUM'}
```

 8. Tools Used

Python 3, **Pandas, NumPy — data generation & wrangling
Scikit-learn— `IsolationForest`, `RandomForestClassifier`, `StandardScaler`,
  metrics
- Matplotlib / Seaborn— visualizations
- Joblib— model persistence
- Designed to run in **Google Colab** (or any local Jupyter environment)

 9. Limitations & Future Work

- Trained on synthetic data; before production use, retrain on real, anonymized,
  labelled transaction history from an actual mobile-money platform.
- Velocity/agent features are computed in batch here; production would need a
  real-time feature store to compute `seconds_since_last_txn`, `txns_last_24h`,
  and agent statistics on the fly.
- No graph-based features yet (e.g. shared devices/SIMs/recipient accounts across a
  fraud ring) — a strong candidate for v2.
- No rules layer for hard regulatory thresholds (e.g. CBN reporting limits) — currently
  the model *learns* proximity-to-threshold as one signal among many, but a v2 could
  combine ML scoring with explicit compliance rules.
- Add a REST API wrapper (FastAPI/Flask) around `inference.py` for real-time
  integration with agent apps and USSD gateways.

 10. Demo Video

A 2–3 minute walkthrough demo script/outline is provided in
`demo_video_script.md`, covering: the problem, the dataset, model training, fraud prediction, and evaluation

---
Capstone project — Mobile Money Fraud Detector 
