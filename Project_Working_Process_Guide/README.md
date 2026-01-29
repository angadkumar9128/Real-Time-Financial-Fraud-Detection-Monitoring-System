
# 📘 Real-Time-Financial-Fraud-Detection-&-Monitoring-System Project 

## 📌 Project Title

**Streaming Fraud Detection Pipeline using Databricks Lakehouse + Isolation Forest + MLflow**

---

## 🎯 Project Goal (Simple Explanation)

Fraud transactions are rare, hidden, and often happen in unusual patterns.

This project builds a complete real-time fraud detection system that:

* Ingests transaction data continuously (streaming)
* Cleans and transforms it into structured layers
* Creates behavioral fraud features using time windows
* Trains an anomaly detection model (Isolation Forest)
* Logs the model with MLflow
* Produces business-ready Gold dashboards

---

## 🏗️ Architecture Overview (End-to-End)

```
Raw Streaming JSON Files
        ↓ (Auto Loader)
Bronze Layer (Raw Delta Stream)
        ↓
Silver Base Layer (Clean + Typed Transactions)
        ↓
Silver Features Layer (5-min Window Fraud Features)
        ↓
ML Training Dataset (Batch Join)
        ↓
Isolation Forest Model + MLflow Tracking
        ↓
Gold Layer (Fraud KPIs + Risk Tables + Predictions)
```

---

# 📂 Dataset Understanding (Transaction Fields Explained)

Each transaction record contains:

* Payment details
* Amount behavior
* Location information
* Email identity
* Fraud label (for evaluation)

Let’s understand the key columns using real examples.

---

## ✅ Example Transaction Row

| Field          | Value       |
| -------------- | ----------- |
| TransactionID  | 2987012     |
| TransactionAmt | 50.00       |
| ProductCD      | W           |
| card4          | visa        |
| card6          | debit       |
| addr1          | 204         |
| addr2          | 87          |
| P_emaildomain  | verizon.net |
| isFraud        | 0           |

This means:

> A legitimate debit card transaction of $50 for product type W, done using Visa, with billing mismatch.

---

---

# 🧾 Field-by-Field Explanation (Beginner Friendly)

---

## 1️⃣ TransactionID

### What it is

Unique identifier for each transaction.

### Why it matters

Used for:

* tracking suspicious transactions
* audit logs
* joining results

### Example

```
TransactionID = 2987012
```

---

## 2️⃣ TransactionAmt

### What it is

Transaction amount (money spent)

### Why it matters

Fraud often involves unusual amounts:

* very high purchases
* abnormal spending patterns

### Feature created

```python
is_high_value_txn = TransactionAmt > 1000
log_transaction_amount = log(TransactionAmt)
```

---

## 3️⃣ TransactionDT + event_timestamp

### What it is

TransactionDT is time offset in seconds.

Converted into:

```python
event_timestamp = timestamp + TransactionDT
```

### Why important

Fraud often happens in bursts.

Used for:

* streaming windows
* velocity features

---

## 4️⃣ Card Fields (card1–card6)

These represent payment identity.

| Column | Meaning         | Example         |
| ------ | --------------- | --------------- |
| card1  | card identifier | 3786            |
| card4  | network type    | visa/mastercard |
| card6  | debit/credit    | debit           |

### Fraud usage

Fraud repeats on specific cards:

> card1 = 3786 suddenly makes 20 transactions in 5 min → suspicious

---

## 5️⃣ Address Fields (addr1, addr2)

### What they represent

Billing vs shipping region codes

### Fraud signal

Mismatch may indicate fraud.

Feature:

```python
is_international_txn = addr1 != addr2
```

Example:

| addr1 | addr2 | Risk       |
| ----- | ----- | ---------- |
| 204   | 204   | Normal     |
| 204   | 87    | Suspicious |

---

## 6️⃣ Distance Fields (dist1, dist2)

### What they represent

Distance between customer and delivery.

Fraudsters often ship goods far away.

Example:

| dist1  | Risk       |
| ------ | ---------- |
| 2 km   | Normal     |
| 500 km | Suspicious |

---

## 7️⃣ Email Domains (P_emaildomain, R_emaildomain)

### Why important

Fraudsters use:

* disposable emails
* mismatched domains

Example:

| Purchaser | Recipient | Risk   |
| --------- | --------- | ------ |
| gmail.com | gmail.com | Normal |
| gmail.com | fake.xyz  | Fraud  |

---

## 8️⃣ ProductCD

Product category code.

Fraud risk differs by product types.

Feature:

```python
product_diversity_5min
```

---

# ⚡ Streaming Pipeline Explained (Most Important Part)

---

## 🔥 Why Streaming?

Fraud detection must happen in near real-time:

* Detect fraud as transactions arrive
* Prevent further loss

Databricks Auto Loader allows streaming without Kafka.

---

# 🥉 Bronze Layer (Raw Streaming Ingestion)

### Purpose

Store raw incoming transactions exactly as received.

### Code

```python
bronze_raw_df = (
    spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .load("/Volumes/.../stream_input/")
)
```

### Output Table

```
fraud_detection_bronzelayer.stream_bronze_data
```

Bronze keeps:

* raw schema
* ingestion metadata
* file lineage

---

# 🥈 Silver Layer (Clean + Feature Engineering)

Silver is split into two parts:

---

## ✅ Silver Base Layer

### Purpose

Clean, typed, row-level transaction facts.

### Tasks

* Type casting
* Null handling
* Simple features

```python
silver_base_df = (
    bronze_df
      .withColumn("TransactionAmt", col("TransactionAmt").cast("double"))
      .filter(col("TransactionAmt").isNotNull())
      .withColumn("is_high_value_txn", when(col("TransactionAmt") > 1000, 1))
)
```

### Output Table

```
silver_transactions_base
```

---

## ✅ Silver Features Layer (5-Min Window Aggregates)

### Purpose

Fraud is behavioral → need time-based context.

### Window Features

| Feature                | Meaning              |
| ---------------------- | -------------------- |
| txn_count_5min         | transaction velocity |
| avg_amount_5min        | average spending     |
| stddev_amount_5min     | spending instability |
| product_diversity_5min | variety of products  |

### Code

```python
silver_features_df = (
    silver_base_stream
      .groupBy(
          col("card1"),
          window(col("event_timestamp"), "5 minutes")
      )
      .agg(
          count("*").alias("txn_count_5min"),
          avg("TransactionAmt").alias("avg_amount_5min")
      )
)
```

### Output Table

```
silver_txn_features_5min
```

---

# 🤖 Machine Learning Layer (Isolation Forest)

---

## Why Isolation Forest?

Fraud is rare → unsupervised anomaly detection is powerful.

Isolation Forest learns:

* normal transaction behavior
* flags rare anomalies

---

## Training Dataset Creation (Batch Join)

```python
training_df = base_df.join(
    features_df,
    (base_df.card1 == features_df.card1) &
    (base_df.event_timestamp >= features_df.window_start) &
    (base_df.event_timestamp < features_df.window_end),
    "left"
)
```

---

## Model Training

```python
model = IsolationForest(contamination=0.02)
model.fit(X)
```

Output:

* anomaly score
* fraud suspicion

---

# 📌 MLflow Tracking

MLflow logs:

* parameters
* metrics
* model artifacts
* signature

```python
mlflow.sklearn.log_model(
    model,
    artifact_path="isolation_forest_model",
    input_example=X.head(5),
    signature=infer_signature(X, model.predict(X))
)
```

---

# 🥇 Gold Layer (Business & Dashboard Tables)

Gold contains:

* fraud KPIs
* risk profiles
* anomaly outputs

Example:

```sql
CREATE TABLE gold_fraud_overview AS
SELECT
  COUNT(*) total_txns,
  SUM(isFraud) fraud_txns,
  AVG(TransactionAmt) avg_amt
FROM silver_transactions_base;
```

Gold tables drive:

* Power BI dashboards
* Fraud monitoring reports

---

# ✅ Final Project Outcome

This system achieves:

✔ Real-time ingestion
✔ Clean Silver transaction pipeline
✔ Streaming behavioral fraud features
✔ ML anomaly detection with Isolation Forest
✔ Model tracking using MLflow
✔ Gold dashboards for fraud analytics

---

# 🎤 Final Summary

> This project implements a Databricks Lakehouse fraud detection pipeline where streaming transactions are ingested into Bronze, cleaned into Silver Base, enriched with 5-minute behavioral window features in Silver Features, trained using Isolation Forest anomaly detection logged via MLflow, and finally aggregated into Gold KPI dashboards for fraud monitoring.

---



Just tell me what you need next.

