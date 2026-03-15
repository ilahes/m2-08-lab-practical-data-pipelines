![logo_ironhack_blue 7](https://user-images.githubusercontent.com/23629340/40541063-a07a0a8a-601a-11e8-91b5-2f13e4e6b441.png)

# Lab | Designing a Data Pipeline Architecture

## Overview

In this lab you will design a complete data pipeline architecture — on paper, without writing implementation code. The scenario is realistic: you are the data engineer for an online retailer, and your team needs a pipeline that ingests transactional data, validates and transforms it, stores it for analytics and ML, and monitors its own health.

You already know the data well from a previous lab (the Online Retail Dataset). Now you must design the system that processes it in production: handling both historical batch loads and new transactions arriving one at a time.

**This is a design lab.** Your deliverable is an architecture document, not a working codebase. Tomorrow's assessment will ask you to implement the pipeline you design here, so take the design seriously — the better your architecture, the smoother the implementation will go.

## Scenario

You are building a data pipeline for the same UK-based online retailer whose data you cleaned in the previous lab. The pipeline must:

1. **Ingest** transactional data from two sources:
   - A historical batch file (the full `Online Retail.xlsx` dataset, loaded once)
   - A live stream of new transactions arriving **one row at a time** (simulating real-time order events)

2. **Validate** every record against schema rules and business constraints before it enters the clean layer.

3. **Transform** validated data into analysis-ready form: compute derived columns, aggregate customer-level features, and flag anomalies.

4. **Store** the results in a layered structure (raw → clean → feature) so that any stage can be reprocessed if needed.

5. **Monitor** pipeline health: data freshness, volume, schema consistency, and quality metrics.

The pipeline will eventually serve two consumers:
- A **BI dashboard** that shows daily sales summaries, top products, and country-level breakdowns (batch, updated nightly)
- An **ML model** that predicts whether a customer is high-value based on their transaction history (retrained weekly, features updated daily)

## Learning Goals

By completing this lab, you will be able to:

- Decompose a data pipeline into well-defined stages with clear responsibilities
- Design a validation layer that catches schema violations and business rule failures
- Plan a layered storage architecture (raw → clean → feature) and justify format choices
- Identify failure modes and design resilience mechanisms (retries, dead letter queues, checkpoints)
- Design monitoring that detects data quality degradation, freshness issues, and schema drift
- Reason about offline vs. online data paths and where they intersect
- Create a data lineage strategy that traces records from source to consumer

## Prerequisites

- Familiarity with the Online Retail Dataset (from the previous lab)
- Understanding of ETL workflows, data formats (CSV, Parquet), and storage engines (from earlier lessons)
- Understanding of batch vs. stream processing concepts

## Requirements

- Fork this repository to your own GitHub account.
- Clone your fork to your machine.
- Create a file named `pipeline-design.md` (or `pipeline-design.pdf` if you prefer diagramming tools).
- All your design work goes in this single file.

---

## Task 1: Pipeline Architecture Diagram

### 1.1 — Draw the end-to-end architecture

Create an architecture diagram (text-based is fine — use ASCII art, Mermaid syntax, or describe it in structured text) showing every component of your pipeline. Your diagram must include:

- **Data sources**: the historical batch file and the live transaction stream
- **Ingestion layer**: how each source enters the pipeline
- **Landing zone (raw storage)**: where raw data is stored before any processing
- **Validation stage**: where and how records are checked
- **Quarantine / dead letter area**: where failed records go
- **Transformation stage**: where validated data is cleaned and enriched
- **Storage layers**: raw, clean, and feature layers with format choices
- **Consumer interfaces**: how the BI dashboard and the ML model access the data
- **Monitoring**: where quality checks and alerts happen

Example format (you should expand this significantly):

```
┌─────────────────┐    ┌─────────────────┐
│  Batch File     │    │  Live Stream    │
│ (Online Retail  │    │ (row-by-row     │
│  .xlsx)         │    │  transactions)  │
└────────┬────────┘    └────────┬────────┘
         │                      │
         └──────────┬───────────┘
                    ▼
         ┌──────────────────┐
         │   Ingestion      │
         │   Layer          │
         └────────┬─────────┘
                  ▼
              ... etc ...
```

### 1.2 — Describe each component

For every box in your diagram, write a short paragraph (3-5 sentences) explaining:
- What the component does
- What it receives as input and produces as output
- What technology or format you would use (e.g., "raw storage uses Parquet files organized by date")

### Deliverable for Task 1

An architecture diagram and component descriptions covering the full pipeline from sources to consumers.

---

## Task 2: Validation and Error Handling Design

### 2.1 — Define validation rules

Based on your experience with the Online Retail Dataset, define a complete set of validation rules. Organize them into three categories:

**Schema validations** (structural correctness):
- List every field, its expected type, and whether it is required
- Example: "`InvoiceNo` must be a non-empty string"

**Value range validations** (sensible values):
- Define acceptable ranges for numeric fields
- Example: "`UnitPrice` must be > 0 for non-cancellation records"

**Business rule validations** (domain logic):
- Define cross-field consistency checks
- Example: "If `InvoiceNo` starts with 'C', then `Quantity` must be negative"

### 2.2 — Design the error handling flow

For each validation category, specify what happens when a record fails:
- Is the record rejected entirely, partially accepted, or flagged for review?
- Where does the failed record go? (dead letter queue, quarantine table, error log)
- How would an operator be alerted?
- How would you recover quarantined records after fixing the source issue?

### Deliverable for Task 2

A complete validation rule set (at least 8-10 rules) and an error handling flow for each failure category.

---

## Task 3: Transformation and Storage Design

### 3.1 — Define transformations

List every transformation you would apply to validated data. For each transformation, specify:
- **Input**: what the transformation receives
- **Output**: what it produces
- **Idempotency**: is this transformation safe to re-run? If not, how would you make it idempotent?

Your transformations should include at minimum:
- Cleaning operations (handling remaining edge cases after validation)
- Derived columns (line totals, cancellation flags, date components)
- Customer-level aggregations (total revenue, order count, product diversity, recency)
- Feature engineering for the ML model (using only data from the observation window)

### 3.2 — Design the storage layers

Define three storage layers:

| Layer | Contents | Format | Update frequency | Retention |
|---|---|---|---|---|
| Raw | | | | |
| Clean | | | | |
| Feature | | | | |

For each layer, justify your format choice. Consider:
- Who reads from this layer and how?
- How large will it grow over time?
- Does it need to support time-travel (reading historical versions)?

### 3.3 — Design incremental updates

Explain how your pipeline handles new data arriving over time:
- How do you track what has already been processed? (high-water mark, change data capture, etc.)
- How do you handle late-arriving records?
- When and how do customer-level features get refreshed?

### Deliverable for Task 3

Transformation list, storage layer table with justifications, and an incremental update strategy.

---

## Submission

### What to submit

Submit the following file:

- `pipeline-design.md` (or `pipeline-design.pdf`)

This file should contain all three tasks: architecture diagram, validation design, and transformation and storage design.

### Definition of done (checklist)

Before you submit, make sure:

- [ ] The architecture diagram shows all pipeline stages from source to consumer
- [ ] Every component in the diagram has a written description
- [ ] At least 8-10 validation rules are defined across schema, value, and business categories
- [ ] Error handling flow covers rejection, quarantine, alerting, and recovery
- [ ] Transformations are listed with inputs, outputs, and idempotency considerations
- [ ] Storage layers are defined with format choices and justifications
- [ ] The design accounts for both batch (historical) and incremental (row-by-row) data

### How to submit (Git workflow)

When you are done, save your design document, then run:

```bash
git add .
git commit -m "Solved m2-08 lab"
git push -u origin HEAD
```

- Create a pull request from your fork.
- Paste the link to your pull request in the Student Portal.

## Evaluation Criteria

Your work will be evaluated on **completeness**, **coherence**, and **depth of reasoning**. **Completeness** means all three tasks are addressed with sufficient detail. **Coherence** means the components fit together — your validation rules match the data you described, your storage formats match your query patterns, and your design accounts for both batch and incremental data flows. **Depth** means you explain *why* you made each design choice, not just *what* you chose.
