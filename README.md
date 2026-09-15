# UPI Fraud Ring & Merchant Analytics
**TransOrg AgentIQ Datathon 2026 — Track 1: FinTech & BFSI**

An analytics pipeline that turns four messy, synthetic UPI payments datasets into a
queryable star-schema database of transactions, KYC, merchants, and chargebacks —
with documented fraud-risk and churn metrics for merchant and user risk scoring.

## Problem statement

A National Payments Authority needs to analyze micro-transaction data to spot circular
money-laundering rings, synthetic identity fraud, and compromised merchant accounts,
starting from raw logs with missing UTR numbers, mismatched PAN/Aadhaar formats,
OCR errors, and currency symbols embedded in numeric columns.

## What's in this repo

| File | Rows | What it is |
|---|---|---|
| `track1_upi_transactions_clean.csv` | 20,000 | Cleaned UPI transaction ledger |
| `track1_kyc_records_cleaned.csv` | 35,878 | Cleaned customer KYC records |
| `track1_merchants_master_cleaned.csv` | 6,000 | Cleaned merchant master |
| `json_cleaned.csv` | 2,800 | Cleaned chargeback/dispute records (originally JSON) |
| `track1_analytics_layer.ipynb` | — | Builds the star-schema DB and computes all metrics |
| `track1_analytics.db` | — | Output SQLite database (6 indexed tables) |

See `DATA_DICTIONARY.md` for the full column-by-column reference.

## Pipeline

```
Raw data (messy)  →  Cleaning  →  4 cleaned CSVs (this repo)
                                        ↓
                          track1_analytics_layer.ipynb
                                        ↓
              track1_analytics.db (star schema, 6 tables)
```

### Cleaning

Each cleaned file carries columns that flag what the cleaning step found and fixed —
for example `ticket_size_was_negative`, `missing_mcc_or_category`, and
`id_collision_flag` in the merchant file; `utr_missing` and `mcc_missing` in
transactions; `impossible_reporting_delay` and `disputed_amount_missing` in
chargebacks. These flags are the audit trail of what was wrong in the raw data and how
it was handled (rather than silently dropped).

> **Note:** the original cleaning notebook/script that produced these files isn't in
> this repo yet. This is a known gap — see [Known limitations](#known-limitations).

### Analytics layer (`track1_analytics_layer.ipynb`)

Run this notebook top to bottom (it expects the four cleaned CSVs above in the same
folder) to rebuild `track1_analytics.db`. It:

1. **Resolves entity IDs.** `user_id` and `merchant_id` aren't reliable primary keys as-is
   — the same logical entity can appear under multiple ID formats. The notebook
   documents this and resolves it into clean `dim_users` / `dim_merchants` tables,
   flagging ambiguous cases rather than silently merging them.
2. **Builds a star schema**:

   | Table | Grain |
   |---|---|
   | `dim_users` | 1 row per resolved `user_id` |
   | `dim_merchants` | 1 row per resolved `merchant_id` |
   | `fact_transactions` | 1 row per `txn_id` |
   | `fact_chargebacks` | 1 row per `complaint_id` |
   | `merchant_metrics` | 1 row per `merchant_id` — derived rates, fraud risk, churn |
   | `user_metrics` | 1 row per `user_id` — derived rates, fraud risk |

3. **Computes metrics** (full formulas in the notebook, section 4):
   - **Merchant Fraud Risk Score (0–100):** a weighted blend of normalized
     chargeback rate (30%), dispute-severity-weighted rate (20%), failure rate (20%),
     reversal rate (15%), and share of counterpart users flagged high-risk in KYC
     (15%) — with a hard floor of 90 for merchants already marked suspended/blocked.
   - **User Fraud Risk Score (0–100):** chargeback rate (30%), failure rate (15%),
     reversal rate (15%), KYC risk segment (25%), and KYC conflict count (15%) — floor
     of 80 for rejected/failed KYC.
   - **Merchant Churn:** a merchant is churned if it was active in the 30–60 day
     baseline window before the latest transaction but had zero transactions in the
     most recent 30 days.

4. **Persists everything to `track1_analytics.db`** as six indexed SQLite tables,
   ready to query directly (`pd.read_sql`) or plug into a dashboard.

## How to run

```bash
pip install pandas numpy
jupyter notebook track1_analytics_layer.ipynb
# Run all cells — regenerates track1_analytics.db from the 4 CSVs in this folder
```

Then query the DB directly, e.g.:

```python
import sqlite3, pandas as pd
conn = sqlite3.connect("track1_analytics.db")
pd.read_sql("""
    SELECT merchant_id, fraud_risk_score FROM merchant_metrics
    ORDER BY fraud_risk_score DESC LIMIT 10
""", conn)
```

## Status

- [x] Data cleaning (4 raw files → 4 cleaned files)
- [x] Analytics layer — star schema + fraud risk + churn metrics, in SQLite
- [ ] Executive dashboard
- [ ] Bonus: natural-language-to-chart AI agent

## Known limitations

- The cleaning step that produced the four `*_cleaned.csv` files was done inline and
  the notebook/script for it isn't checked into this repo — only its output. Raw vs.
  cleaned row counts aren't documented here yet.
- No dashboard or AI agent yet — this repo currently covers the Data Rescue and
  Analytics Layer stages of the challenge only.
