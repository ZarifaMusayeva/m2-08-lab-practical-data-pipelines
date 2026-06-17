# Pipeline Architecture & Design Document

---

## Task 1: Pipeline Architecture Diagram

```
┌─────────────────────────┐          ┌──────────────────────┐
│  Batch Source:          │          │  Live Stream:        │
│  Online Retail.xlsx     │          │  Row-by-Row          │
└────────────┬────────────┘          └──────────┬───────────┘
             │                                  │
             └──────────────┬───────────────────┘
                            ▼
                   ┌─────────────────┐
                   │  Ingestion Layer │
                   └────────┬────────┘
                            ▼
                   ┌─────────────────┐       ┌──────────────────────────┐
                   │   Raw Parquet   │. . . .│  Monitoring and Alerting │
                   └────────┬────────┘       └──────────────────────────┘
                            ▼                           ▲
                   ◇─────────────────◇                 │
                   ◇  Validation     ◇ · · · · · · · · ┘
                   ◇    Stage        ◇
                   ◇─────────────────◇
                    /               \
                   /                 \
               [Fail]              [Pass]
                  ▼                   ▼
   ┌──────────────────────┐    ┌─────────────────┐
   │  Quarantine /        │    │  Transformation │
   │  Dead Letter Queue   │    └────────┬────────┘
   └──────────────────────┘            ▼
                                ┌─────────────────┐
                                │   Clean Data    │
                                └───────┬─────────┘
                                       / \
                                      /   \
                                     ▼     ▼
                        ┌──────────────┐  ┌───────────────┐
                        │   Feature    │  │  BI Dashboard │
                        │  Engineering │  └───────────────┘
                        └──────┬───────┘
                               ▼
                        ┌─────────────────┐
                        │  Feature Store  │
                        └──────┬──────────┘
                               ▼
                        ┌─────────────────┐
                        │    ML Model     │
                        └─────────────────┘
```

### Component Descriptions

- **Batch Source:** The historical data comes from the `Online Retail.xlsx` file. It is loaded once and sent to the ingestion layer.

- **Live Stream:** New transactions arrive one row at a time. Each new order is sent to the ingestion layer in real time.

- **Ingestion Layer:** Receives data from both sources. It reads the data and sends it to raw storage.

- **Raw Parquet:** All incoming records are stored here exactly as received. Nothing is changed or deleted at this stage.

- **Validation Stage:** Every record is checked against schema, value range and business rules. Records that pass go to transformation, records that fail go to quarantine.

- **Quarantine / Dead Letter Queue:** Failed records are stored here for review. The team is notified and decides whether to fix and resubmit them.

- **Transformation:** Valid records are cleaned and new columns are added, such as `LineTotal` and `is_cancellation`.

- **Clean Data:** Stores all validated and transformed records. The BI dashboard reads directly from this layer.

- **Feature Engineering:** Customer-level features such as total revenue and order count are calculated here for the ML model.

- **Feature Store:** Stores the final features. Updated daily and kept for 90 days.

- **BI Dashboard:** Reads from the clean layer to show daily sales and country-level reports.

- **ML Model:** Reads from the feature store and is retrained weekly to predict high-value customers.

- **Monitoring and Alerting:** Checks the pipeline after each stage. Sends alerts if something goes wrong.

---

## Task 2: Validation and Error Handling Design

### 2.1 – Define Validation Rules

#### A – Schema Validations (Structural Correctness)

| # | Field | Rule |
|---|-------|------|
| 1 | `InvoiceNo` | String format, must not be empty |
| 2 | `StockCode` | String format, must not be empty |
| 3 | `Description` | String format, may be empty |
| 4 | `Quantity` | Integer format, cannot be null or zero |
| 5 | `InvoiceDate` | Datetime format, not null |
| 6 | `UnitPrice` | Float format, must not be empty |
| 7 | `CustomerID` | Integer format, must not be empty |
| 8 | `Country` | String format, must not be empty |

#### B – Value Range Validations (Sensible Values)

| # | Field | Rule |
|---|-------|------|
| 1 | `Quantity` | For non-cancellation records: must be `> 0`. For cancellation records: must be `< 0` |
| 2 | `Quantity` | Must be within the range `[-10,000, 10,000]`. Values outside this range are treated as data entry errors |
| 3 | `UnitPrice` | Must be `>= 0.0`. Negative prices are invalid |

#### C – Business Rule Validations (Domain Logic)

| # | Rule |
|---|------|
| 1 | If `InvoiceNo` starts with `'C'`, then `Quantity` must be negative |
| 2 | If `Quantity` is negative and `InvoiceNo` does **not** start with `'C'`, this is an inconsistency — the record is quarantined |
| 3 | Fractional or negative `CustomerID`s are invalid |

---

### 2.2 – Design the Error Handling Flow

#### For Category A (Schema Violations)

- Record is rejected entirely.
- Written to the quarantine table as Parquet, with appended fields: `rule` and `reason`.
- If the error rate in a batch goes above **1%**, the data engineering team is notified to investigate.
- After notification, the problematic records must be corrected and resubmitted through the pipeline. They go through the same validation checks again — if they pass, they are moved to the clean layer; if not, they stay in quarantine for further review.

#### For Category B (Value Range Violations)

- Records are rejected and quarantined for review.
- Written to the quarantine table.
- A summary report is generated listing the count and percentage of range failures per rule, and sent to the data team.
- If the problem is a known bug, the team fixes it and resubmits the affected records.

#### For Category C (Business Rule Violations)

- Records are rejected and quarantined for review.
- Written to the quarantine table.
- Any business rule failure is reported to the data engineering team **immediately**.
- The team reviews the quarantined records and decides whether the data is valid or not. Valid records are resubmitted; invalid ones are discarded.

---

## Task 3: Transformation and Storage Design

### 3.1 – Define Transformations

#### Transformation 1 – Whitespace Stripping

| | Detail |
|---|---|
| **Input** | String fields from the clean layer |
| **Output** | Same fields with leading and trailing whitespace removed from all text fields |
| **Idempotent** | Yes — applying `strip()` twice produces the same result |

#### Transformation 2 – Line Total Calculation

| | Detail |
|---|---|
| **Input** | `Quantity` and `UnitPrice` |
| **Output** | New column: `LineTotal = Quantity * UnitPrice` |
| **Idempotent** | Yes — same result |

#### Transformation 3 – Cancellation Flag

| | Detail |
|---|---|
| **Input** | `InvoiceNo` field |
| **Output** | New boolean column: `is_cancellation = True` if `InvoiceNo` starts with `'C'`, else `False` |
| **Idempotent** | Yes |

#### Transformation 4 – Date Parts Extraction

| | Detail |
|---|---|
| **Input** | `InvoiceDate` (datetime) |
| **Output** | New columns: `InvoiceYear`, `InvoiceMonth`, `InvoiceDayOfWeek` |
| **Idempotent** | Yes — taken from a single datetime field |

#### Transformation 5 – Customer-Level Aggregation

| | Detail |
|---|---|
| **Input** | All clean records grouped by `CustomerID` |
| **Output** | One row per `CustomerID` with: |

Output columns:

- `TotalRevenue` — sum of `LineTotal` for non-cancellation records
- `TotalOrders` — count of distinct `InvoiceNo` values
- `AvgOrderValue` — `TotalRevenue / TotalOrders`
- `UniqueProducts` — count of distinct `StockCode` values purchased

| **Idempotent** | Yes — given the same input records and the same run date, output is identical |

#### Transformation 6 – RFM Metrics

| | Detail |
|---|---|
| **Input** | Clean records and the outputs from Transformation 5 |
| **Output** | RFM metrics for each customer |
| **Idempotent** | Yes |

Output columns:

- **Recency** — Number of days since the customer's last purchase
- **Frequency** — Total number of unique orders placed
- **Monetary** — Total amount spent by the customer

---

### 3.2 – Design the Storage Layers

| Layer | Contents | Format | Update Frequency | Retention |
|-------|----------|--------|------------------|-----------|
| **Raw** | All ingested records, exactly as received | Parquet | Stream / Batch | Never deleted |
| **Clean** | Validated and transformed records | Parquet | Stream / Batch | Indefinite |
| **Feature** | Customer and product features summarized in one row per entity, updated daily | Parquet | Daily | 90 days |

> **Why Parquet?**
> All three layers use Parquet format. Parquet is columnar, which makes reading and filtering data much faster compared to CSV or JSON. It also takes less storage space and works with tools like `pandas`.
>
> - The **raw layer** is never modified, so Parquet is a good fit since it is designed for write-once storage.
> - The **clean layer** is partitioned by date, which allows queries to skip unnecessary data and run faster.
> - The **feature layer** stores 90 days of snapshots so the ML team can train on historical data and track changes over time.

---

### 3.3 – Design Incremental Updates

- The pipeline keeps a record of which data has already been processed.
  - For **batch data**: the last processed file is saved.
  - For **stream data**: the last read message position is saved.
- This way, the pipeline does not process the same data twice.
- **Late-arriving records:** Sometimes records arrive after their partition was already processed. In this case, they are added to the correct partition with a `late` flag.
- **Real-time feature updates:** When a new transaction arrives through the stream, that customer's features are also updated immediately so the ML model always has fresh data.