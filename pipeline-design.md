# Task 1 — Pipeline Architecture Diagram

## 1.1 End-to-End Architecture Diagram

The following architecture shows a complete data pipeline for the online retail system. It supports both a one-time historical batch load from the `Online Retail.xlsx` dataset and a continuous stream of new transaction events arriving one row at a time. The pipeline separates ingestion, raw storage, validation, transformation, and serving layers so that data can be monitored, traced, and reprocessed when needed.

```text
┌──────────────────────────────┐         ┌──────────────────────────────┐
│ Historical Batch Source      │         │ Live Transaction Stream      │
│ Online Retail.xlsx dataset   │         │ New transaction events       │
└──────────────┬───────────────┘         └──────────────┬───────────────┘
               │                                        │
               ▼                                        ▼
┌──────────────────────────────┐         ┌──────────────────────────────┐
│ Batch Ingestion              │         │ Stream Ingestion             │
│ Reads Excel file and         │         │ Captures row-by-row events   │
│ converts rows to records     │         │ with ingestion metadata      │
└──────────────┬───────────────┘         └──────────────┬───────────────┘
               └──────────────────────┬─────────────────┘
                                      ▼
                     ┌──────────────────────────────────┐
                     │ Raw / Landing Zone               │
                     │ Append-only source-faithful      │
                     │ storage for all ingested records │
                     └──────────────┬───────────────────┘
                                    ▼
                     ┌──────────────────────────────────┐
                     │ Validation Layer                 │
                     │ Schema, value, and business      │
                     │ rule checks                      │
                     └──────────────┬───────────────────┘
                          valid     │         invalid
                                    │
                    ┌───────────────┘───────────────┐
                    ▼                               ▼
     ┌──────────────────────────────┐   ┌──────────────────────────────┐
     │ Transformation Layer         │   │ Quarantine / Dead Letter     │
     │ Cleaning, standardization,   │   │ Failed records, error codes, │
     │ enrichment, aggregations     │   │ and recovery queue           │
     └──────────────┬───────────────┘   └──────────────┬───────────────┘
                    ▼                                  │
     ┌──────────────────────────────┐                  │
     │ Clean Layer                  │                  │
     │ Validated transaction data   │                  │
     │ with derived columns         │                  │
     └──────────────┬───────────────┘                  │
                    ▼                                  ▼
     ┌──────────────────────────────┐   ┌──────────────────────────────┐
     │ Feature Layer                │   │ Monitoring & Alerting        │
     │ Customer-level features,     │   │ Freshness, volume, schema,   │
     │ BI summaries, ML datasets    │   │ quality, and failure metrics │
     └──────────────┬───────────────┘   └──────────────────────────────┘
                    ├──────────────────────────┐
                    ▼                          ▼
     ┌──────────────────────────────┐   ┌──────────────────────────────┐
     │ BI Dashboard                 │   │ ML Training Pipeline         │
     │ Daily sales, top products,   │   │ Weekly retraining using      │
     │ country breakdowns           │   │ daily refreshed features     │
     └──────────────────────────────┘   └──────────────────────────────┘



1.2 Component Descriptions
Data Sources
Historical Batch Source

The historical batch source is the full Online Retail.xlsx dataset used to bootstrap the pipeline with past transactions. Its input is the original Excel file, and its output is a row-level stream of transaction records passed into the batch ingestion process. This source is loaded once during initial setup or again if a full historical reprocessing is required. The source format is Excel, but it is converted into structured records during ingestion so it can enter the rest of the pipeline consistently.

Live Transaction Stream

The live transaction stream represents new transaction events produced by the retailer’s operational system. Its input is individual order events, typically represented as JSON messages, and its output is a continuous stream of raw transaction records written into the ingestion layer. This component supports near-real-time updates for fresh analytics and up-to-date customer features. Using event-style messages makes it easy to attach metadata such as ingestion time, event ID, and source system.

Ingestion Layer
Batch Ingestion

The batch ingestion component reads the historical Excel file and converts each row into a structured transaction record. Its input is the Online Retail.xlsx file, and its output is an append-only set of raw records written into the landing zone along with ingestion metadata such as source name, load time, and run ID. This component focuses on reliable bulk loading rather than transformation. A batch job or scheduled ingestion workflow is appropriate here because the historical dataset is processed as a bounded file rather than an unbounded stream.

Stream Ingestion

The stream ingestion component captures new transactions one event at a time as they arrive from the live system. Its input is a sequence of transaction events, and its output is source-faithful raw event records stored in the landing zone with metadata such as ingestion timestamp and event identifier. This layer should do as little business logic as possible so that the original incoming event is preserved for debugging and replay. A message queue or streaming transport is a suitable design choice because it supports continuous arrival, buffering, and retry handling.

Raw Storage Layer
Raw / Landing Zone

The raw layer stores all ingested records before validation or transformation. Its input is the output of both batch and stream ingestion, and its output is an immutable record set used by the validation stage for downstream processing or reprocessing. This layer should preserve source content and ingestion metadata in an append-only structure so the pipeline can replay data if rules or transformations change later. A partitioned Parquet layout is a strong choice here because it is efficient for storage and scanning while still preserving source-faithful records with minimal normalization.

Validation and Error Handling
Validation Layer

The validation layer checks whether each raw record satisfies schema, value range, and business rule constraints. Its input is raw ingested records, and its output is either a validated record passed to transformation or a failed record redirected to quarantine. This stage protects downstream layers from malformed, inconsistent, or logically invalid data. Validation can be implemented as rule-based checks that add explicit failure reasons, making debugging and recovery much easier.

Quarantine / Dead Letter Area

The quarantine or dead letter area stores records that fail validation. Its input is invalid records plus failure metadata, and its output is a recoverable dataset for operator review and later reprocessing. Each quarantined record should include the original payload, the failed rule name, the error category, the timestamp of failure, and the pipeline run identifier. A structured Parquet table or error dataset is appropriate because operators may need to query failure patterns over time and replay corrected records back into validation.

Transformation Layer
Transformation Layer

The transformation layer converts validated transactions into analysis-ready datasets. Its input is validated transaction records, and its output is cleaned transaction data with derived columns, standardized values, anomaly flags, and aggregated customer metrics. This stage handles operations such as line total calculation, cancellation flags, date feature extraction, deduplication, and customer-level summarization. The transformations should be idempotent so that re-running the same input produces the same output without duplication or drift.

Processed Storage Layers
Clean Layer

The clean layer stores validated and transformed transaction-level data in a standardized analytical format. Its input is the output of the transformation layer, and its output is a trusted transactional dataset used for reporting, aggregation, and feature generation. This layer should contain standardized data types, cleaned values, derived columns, and consistent business semantics. Parquet is an appropriate format because the clean layer will be scanned frequently for analytical queries such as filtering by date, country, product, or customer.

Feature Layer

The feature layer stores derived datasets built on top of the clean layer, including customer-level features for machine learning and summary tables for BI. Its input is clean transaction data, and its output is a set of compact, analysis-ready tables such as customer revenue metrics, recency features, order frequency metrics, daily sales summaries, and product performance rollups. This layer separates reusable features from lower-level transaction records so downstream consumers do not need to recompute them repeatedly. Parquet is again a strong format choice because feature tables are read-heavy, compressed well, and support efficient column-based access.

Downstream Consumers
BI Dashboard

The BI dashboard consumes curated summary data to support business reporting. Its input is feature-layer summary tables such as daily sales totals, top products, and country-level breakdowns, and its output is a set of visual dashboards and scheduled reports for business users. This consumer follows a nightly batch update pattern because the reporting use case does not require row-by-row real-time serving. Reading from prepared summary tables reduces dashboard latency and avoids placing repeated aggregation load on lower pipeline layers.

ML Training Pipeline

The ML training pipeline consumes customer-level feature snapshots to train a model that predicts whether a customer is high-value. Its input is the feature layer, especially customer history features computed from the observation window, and its output is a training-ready dataset and periodic model retraining runs. In this design, features are refreshed daily while the model is retrained weekly, which balances freshness with computational cost. Separating feature generation from model training also helps prevent leakage and makes the training process reproducible.

Monitoring and Quality Assurance
Monitoring & Alerting

The monitoring and alerting layer tracks the health of the pipeline across ingestion, validation, transformation, and serving. Its input is operational metadata and quality metrics such as ingestion freshness, daily row counts, validation failure rates, null rates by field, schema drift indicators, and late-arriving event rates. Its output is dashboards, logs, and alerts that notify operators when the pipeline behaves unexpectedly. This layer is essential because a pipeline is only useful if failures, stale data, and quality degradation are detected before they affect BI reports or ML training outputs.


# Task 2 — Validation and Error Handling Design

## 2.1 Validation Rules

To maintain high data quality and prevent corrupted records from entering downstream layers, the pipeline includes a structured validation stage. The validation system checks incoming records using three categories of rules: **schema validation**, **value range validation**, and **business rule validation**. Each rule is designed to detect a different type of data issue, ensuring that both structural and logical correctness are enforced before records reach the clean processing layer.

If a record fails any validation rule, it is not allowed to enter the clean dataset and is instead redirected to the quarantine storage area with detailed error metadata.

---

### Schema Validation Rules (Structural Correctness)

Schema validation ensures that incoming records have the correct structure, required fields, and data types. These rules prevent malformed records from entering the pipeline.

| Field | Expected Type | Required | Validation Rule |
|-----|-----|-----|-----|
| InvoiceNo | String | Yes | Must be a non-empty string |
| StockCode | String | Yes | Must be a non-empty string |
| Description | String | No | If present, must be a string |
| Quantity | Integer | Yes | Must be a numeric integer |
| InvoiceDate | Datetime | Yes | Must be a valid timestamp |
| UnitPrice | Float | Yes | Must be a numeric value |
| CustomerID | Numeric | No | Must be numeric if present |
| Country | String | Yes | Must be a valid country name |

These rules ensure that the pipeline only processes structurally valid records. For example, missing required fields, invalid timestamps, or incorrect data types are immediately detected and prevented from entering downstream layers.

---

### Value Range Validation Rules (Sensible Values)

Value range validations ensure that numeric values fall within realistic and logically consistent ranges. These rules protect the system from corrupted data, system bugs, or extreme outliers.

| Field | Validation Rule |
|-----|-----|
| Quantity | Must not be zero |
| Quantity | Absolute value must be less than 10,000 |
| UnitPrice | Must be greater than or equal to 0 |
| UnitPrice | Must not exceed 10,000 |
| InvoiceDate | Cannot be in the future |
| CustomerID | Must be positive if present |

These rules ensure that the data remains statistically reasonable and consistent with expected retail transaction patterns.

---

### Business Rule Validation (Domain Logic)

Business rule validation ensures that relationships between fields follow the retailer’s operational logic. These rules detect inconsistencies that cannot be caught by simple schema or range checks.

| Rule | Description |
|-----|-----|
| Cancellation rule | If `InvoiceNo` begins with **"C"**, the `Quantity` must be negative |
| Non-cancellation rule | If `InvoiceNo` does not begin with **"C"**, the `Quantity` must be positive |
| Revenue consistency | `Quantity × UnitPrice` must produce a valid transaction total |
| Product identifier rule | `StockCode` must not be empty for valid product transactions |
| Customer feature rule | Transactions without `CustomerID` may remain in the transaction dataset but are excluded from customer-level feature generation |
| Duplicate detection rule | Duplicate transactions with identical `InvoiceNo`, `StockCode`, and `InvoiceDate` should be flagged for review |

These rules ensure that transactions follow the retailer’s real-world business processes and prevent logical inconsistencies from affecting analytical results.

Overall, the validation layer enforces more than ten rules across schema, value, and business categories, ensuring that only consistent and trustworthy data proceeds to the transformation stage.

---

## 2.2 Error Handling Flow

Data pipelines must handle validation failures in a controlled and observable manner rather than silently discarding problematic records. The pipeline therefore implements a structured error handling system consisting of **rejection, quarantine storage, monitoring, and recovery procedures**.

---

### Handling Schema Validation Failures

Schema validation failures occur when required fields are missing or when fields contain incorrect data types.

Handling strategy:

- The invalid record is rejected from the main processing pipeline.
- The record is written to the **quarantine dataset** along with metadata describing the validation failure.
- The pipeline logs the failure event and increments a schema validation error counter.
- If the rate of schema failures exceeds a predefined threshold, the monitoring system triggers an alert.

This approach prevents structurally invalid records from corrupting downstream datasets while ensuring that operators are aware of potential upstream system issues.

---

### Handling Value Range Failures

Value range failures occur when numeric values fall outside acceptable boundaries.

Handling strategy:

- The invalid record is written to the **dead letter dataset** together with the validation rule that failed.
- The pipeline records error metrics such as the affected field and observed value.
- Monitoring dashboards track value-range error frequency.
- Alerts are triggered if unusual spikes in invalid values occur.

These procedures allow engineers to quickly detect upstream bugs, incorrect application logic, or corrupted data inputs.

---

### Handling Business Rule Violations

Business rule violations represent logical inconsistencies between fields rather than structural errors.

Handling strategy:

- The failing record is stored in the **quarantine dataset** with a business-rule failure label.
- The pipeline continues processing other valid records normally.
- Detailed logs record the rule violation and associated transaction fields.
- Monitoring alerts operators if the frequency of rule violations increases significantly.

This ensures that logical inconsistencies do not affect analytical datasets or machine learning features.

---

### Quarantine Storage and Recovery

All rejected records are stored in a dedicated **quarantine storage area** together with detailed metadata.

Each quarantined record includes:

- Original raw transaction record
- Validation error category (schema, value range, or business rule)
- Specific validation rule that failed
- Timestamp of the failure
- Pipeline run identifier

This structured storage allows data engineers to inspect problematic records, identify root causes, and repair upstream data issues.

Once the issue is resolved, quarantined records can be **reprocessed by sending them back into the validation stage**. Because the pipeline transformations are idempotent, reprocessing the same records will not produce duplicate outputs.

---

### Monitoring and Alert Integration

Validation metrics are integrated into the pipeline’s monitoring system so that abnormal conditions can be detected early.

Alerts may be triggered when:

- Validation failure rates exceed expected thresholds
- A sudden spike occurs in schema or value errors
- Daily ingestion batches are missing
- Transaction volume drops unexpectedly
- Schema drift is detected in incoming data

By integrating validation metrics into the monitoring layer, the pipeline ensures that data quality issues are detected quickly and resolved before they impact downstream analytics or machine learning workflows.



# Task 3 — Transformation and Storage Design

## 3.1 Data Transformations

After records pass validation, the pipeline performs a set of transformations to convert raw transactional data into structured datasets suitable for analytics and machine learning. These transformations include cleaning operations, derived columns, customer-level aggregations, and feature engineering.

All transformations are designed to be **idempotent**, meaning that running the transformation multiple times with the same input produces the same output. This property allows the pipeline to safely reprocess historical data without introducing duplicates or inconsistencies.

---

### Cleaning Transformations

Cleaning transformations standardize and normalize records that passed validation but still require formatting or deduplication before analysis.

| Transformation | Input | Output | Idempotent |
|---|---|---|---|
| Standardize timestamp format | Raw `InvoiceDate` values | Parsed datetime field | Yes |
| Remove duplicate transactions | Validated transaction records | Unique transaction dataset | Yes |
| Normalize country names | Country field values | Standardized country values | Yes |
| Handle missing descriptions | Description field | Cleaned or null-safe values | Yes |

These transformations ensure that transaction records are consistent and ready for downstream analytical workloads.

---

### Derived Columns

Derived columns enrich transaction-level records with useful calculated attributes that simplify downstream analytics.

| Transformation | Input | Output | Idempotent |
|---|---|---|---|
| Line total calculation | `Quantity`, `UnitPrice` | `line_total = Quantity × UnitPrice` | Yes |
| Cancellation flag | `InvoiceNo` | Boolean `is_cancellation` field | Yes |
| Date feature extraction | `InvoiceDate` | `date`, `hour`, `weekday` fields | Yes |
| Transaction value category | `line_total` | Categorized value segment | Yes |

These derived fields allow analytical queries and reporting systems to avoid recomputing common calculations repeatedly.

---

### Customer-Level Aggregations

Customer-level aggregations summarize transaction histories into behavioral metrics that describe customer activity patterns.

| Transformation | Input | Output | Idempotent |
|---|---|---|---|
| Total customer revenue | Customer transactions | Sum of `line_total` per customer | Yes |
| Order count | Customer transactions | Total number of orders per customer | Yes |
| Product diversity | Customer purchases | Count of unique `StockCode` values | Yes |
| Recency calculation | Customer purchase history | Days since last purchase | Yes |

These metrics provide useful insights for business analytics and serve as the foundation for machine learning features.

---

### Feature Engineering for Machine Learning

The pipeline generates customer-level feature datasets used by the machine learning model that predicts whether a customer will become high-value.

| Transformation | Input | Output | Idempotent |
|---|---|---|---|
| Build feature snapshot | Clean transactions within observation window | One row per customer with ML features | Yes |
| Average order value | Customer purchase history | `avg_order_value` feature | Yes |
| Purchase frequency | Customer transactions in time window | `purchase_count` feature | Yes |
| Product diversity feature | Customer purchase history | `unique_products_purchased` | Yes |
| Recency feature | Last purchase timestamp | `recency_days` feature | Yes |

To avoid **data leakage**, these features are calculated only from transactions that occur **before the observation cutoff date** used for training.

---

## 3.2 Storage Layer Design

The pipeline uses a **layered storage architecture** consisting of raw, clean, and feature layers. Each layer has a different responsibility and supports different workloads.

| Layer | Contents | Format | Update Frequency | Retention |
|---|---|---|---|---|
| Raw | Original ingestion records with metadata | Parquet | Continuous ingestion | Long-term |
| Clean | Validated and standardized transaction data | Parquet | Daily batch updates | Long-term |
| Feature | Customer-level features and analytical aggregates | Parquet | Daily refresh | Medium-term |

---

### Raw Layer

The raw layer stores all ingested records exactly as they were received from upstream systems. This dataset acts as the **immutable source of truth** for the pipeline. Each record includes ingestion metadata such as source identifier, ingestion timestamp, and pipeline run ID.

Although the batch source originates from an Excel file and the streaming source may produce JSON events, the raw layer stores a source-faithful append-only representation in **Parquet** format. This allows efficient storage, partitioning, and scanning while still preserving the original record content.

Maintaining a raw layer ensures that the pipeline can **reprocess historical data** if validation rules, transformations, or business logic change in the future.

---

### Clean Layer

The clean layer stores validated and standardized transaction data after transformation. This dataset contains consistent data types, normalized values, and derived columns such as line totals and cancellation flags.

The clean layer serves as the primary dataset used by analytical workloads. Business reports, aggregation queries, and feature generation jobs all read from this dataset.

The **Parquet columnar format** is well suited for the clean layer because analytical queries frequently scan large datasets but only access a subset of columns.

---

### Feature Layer

The feature layer stores aggregated datasets derived from the clean transaction layer. These datasets include customer-level metrics, machine learning feature tables, and summary tables used by BI dashboards.

Examples of datasets stored in the feature layer include:

- Customer revenue metrics
- Purchase frequency indicators
- Product diversity features
- Daily sales summary tables
- Top-product performance metrics

Feature datasets are refreshed on a **scheduled basis** rather than continuously. Daily feature updates provide fresh data for analytics, while the machine learning model can be retrained periodically using feature snapshots.

---

## 3.3 Incremental Update Strategy

Production pipelines must process new data incrementally rather than recomputing the entire dataset for every pipeline run. The pipeline therefore uses a combination of **high-water mark tracking, sliding reprocessing windows, and scheduled feature refreshes**.

---

### Tracking Processed Data

The pipeline maintains a **high-water mark** representing the most recent transaction timestamp that has been successfully processed. During each pipeline run, only records with timestamps greater than this stored value are processed.

This mechanism ensures that the pipeline processes only new data and prevents duplicate processing of historical records.

Pipeline metadata such as processed timestamps and run identifiers are stored in a checkpoint table so that processing progress can be tracked across pipeline executions.

---

### Handling Late-Arriving Data

In real-world systems, some events may arrive later than expected due to network delays or upstream system latency.

To handle late-arriving records, the pipeline uses a **sliding reprocessing window**. For example, each daily batch run may reprocess the most recent several days of data. This ensures that delayed records are incorporated correctly without requiring full historical recomputation.

Duplicate detection logic ensures that reprocessed records do not create duplicate outputs.

---

### Feature Refresh Strategy

Customer-level features must remain up-to-date for both analytics and machine learning.

The pipeline therefore refreshes feature tables on a **daily schedule** using the latest clean transaction data. These refreshed features support BI dashboards and operational reporting.

The machine learning model is retrained **weekly** using historical feature snapshots generated by the pipeline. Separating feature refresh from model retraining allows the system to maintain fresh data while controlling computational cost and ensuring reproducibility.


## Data Lineage Strategy

To ensure traceability and reproducibility, the pipeline maintains lineage metadata for every dataset it produces. Each pipeline run records the input source, ingestion timestamp, transformation version, row counts, and a unique pipeline run identifier.

This lineage metadata allows engineers to trace records in the clean and feature layers back to their original raw inputs. If a data quality issue, reporting inconsistency, or model performance problem occurs, the lineage information makes it possible to identify which source data and processing logic produced the affected output.

Maintaining lineage also improves debugging, auditing, and reproducibility. By recording how each dataset was created, the pipeline supports reliable reprocessing and makes historical outputs easier to explain and reproduce.












     