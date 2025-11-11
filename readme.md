# Take-Home Challenge 

## Overview

This challenge implements a production-ready ETL pipeline that consolidates three disparate data sources into a unified Customer 360 view. The pipeline handles real-world data quality issues including inconsistent formatting, duplicates, invalid records, and missing values to produce a clean, analytical dataset.

## Quick Start

### Prerequisites

```bash
pip install pandas pyarrow
```

**Required Python version:** 3.7+

### Input Files Required

Ensure these files are in the same directory as the script:
- `crm_leads.csv` - CRM lead data (comma-separated)
- `transactions.txt` - Transaction records (pipe-delimited)
- `web_activity.json` - Web activity logs (JSON Lines format)

### Running the Pipeline
**Run in Jupyter Notebook**
```bash
jupyter notebook ETL_pipeline_Lalith.ipynb
```

The pipeline will execute all ETL stages automatically and generate output files in the working directory.

## Output Files

1. **customer_360.parquet** - Clean, consolidated customer dataset ready for analytics
2. **rejected_transactions.log** - Log of invalid transactions with rejection reasons

## Key Design Decisions

### 1. Deduplication Strategy

**Decision:** Keep the most recent record per email address

**Rationale:**
- Email serves as the natural business key for leads
- Most recent record reflects the current state of the lead
- Sorts by `creation_date` descending before dropping duplicates
- Preserves data freshness while eliminating redundancy

**Implementation:**
```python
df = df.sort_values('creation_date', ascending=False)
df = df.drop_duplicates(subset='email', keep='first')
```

### 2. Data Quality & Validation

**Standardization Applied:**
- **Names:** Title case formatting (`str.title()`)
- **Emails:** Lowercase and trimmed of whitespace
- **Dates:** Converted to `datetime` objects using pandas automatic parsing
- **Amounts:** Coerced to numeric, with invalid values flagged

**Validation Rules:**
- Transactions must have positive amounts (>0)
- Web activity records must have non-null `user_uuid`
- All rejected records are logged with specific reasons

### 3. Error Handling Approach

**Three-tier logging strategy:**

1. **INFO level:** Pipeline progress and record counts
2. **Validation layer:** Business rule violations captured to rejection log
3. **Exception handling:** Try-catch in main() for pipeline-level failures

**Rejection Log Format:**
```
transaction_id|status|amount|reason
TX-4003|Refunded|-10.0|Non-positive amount (-10.0)
```

### 4. Data Integration Challenge

**Challenge:** No explicit identity mapping provided between datasets

**Solution:** To maintain referential and analytical integrity, I implemented a simple positional mapping between 
user_uuid and email so that lead information could be represented in the final dataset.
However, since no explicit identity map was provided to link crm_leads.csv with either transactions.txt or
web_activity.json, this mapping serves only as a placeholder to simulate a complete join for demonstration purposes.

In production:
- Woudl use verified email ↔ user_uuid mapping table (e.g., sourced from authentication or signup data) to ensure data accuracy and integrity.


### 5. Aggregation Strategy

**Transactions:** 
- Grouped by `user_uuid` with status breakdown (completed/pending)
- Maintains both count and sum for each status
- Preserves comma-separated list of `transaction_id` for audit trail

**Web Activity:**
- Total page views summed per user
- Most recent activity timestamp captured

**Join Logic:** 
- Outer joins used throughout to preserve all records
- Missing numeric values filled with 0
- NULL timestamps preserved (NaT) to distinguish no-activity users

### 6. Technology Choices

**Pandas:**
- Sufficient for medium-scale data processing
- Excellent data quality tooling (type coercion, deduplication)
- Simple transition to Dask or PySpark for scale

**Parquet with PyArrow:**
- Columnar format optimized for analytics
- 10x+ compression vs CSV
- Schema enforcement prevents downstream type errors
- Native support in modern data warehouses

## Data Quality Issues Addressed

| Issue | Detection | Resolution |
|-------|-----------|------------|
| Duplicate leads | Email matching | Keep most recent by `creation_date` |
| Inconsistent casing | Pattern analysis | Title case names, lowercase emails |
| Type mismatches | `pd.to_numeric()`, `pd.to_datetime()` | Automatic coercion with error handling |
| Invalid amounts | Business rule validation | Filter negative values, log rejections |
| Missing identifiers | `.notna()` checks | Remove records with null `user_uuid` |
| Null values | Post-join nulls | Fill with 0 for metrics, preserve NaT for timestamps |

## Cloud Deployment Architecture

### Recommended AWS Deployment as I am most experienced with AWS

**Pipeline Orchestration:**
```
AWS Step Functions → Lambda (for small files)
           OR
AWS Glue Jobs (for larger datasets)
```

**Infrastructure:**
1. **Ingestion Layer**
   - S3 buckets for raw data (separate buckets per source)
   - EventBridge triggers on file arrival

2. **Processing Layer**
   - AWS Glue ETL job running this Python script
   - Glue Catalog for schema management
   - CloudWatch for logging and monitoring

3. **Storage Layer**
   - S3 for `customer_360.parquet` (partitioned by date)
   - S3 for `rejected_transactions.log` with versioning

4. **Analytics Layer**
   - Athena for ad-hoc queries
   - Redshift Spectrum for complex analytics
   - QuickSight for dashboards

**IAM Roles:**
- Glue execution role with S3 read/write
- Least privilege access to input/output buckets

**Monitoring:**
```python
# CloudWatch metrics to track:
- Records processed per source
- Rejected transactions
- null user uuid in web acitivity
- Pipeline execution time
- Data freshness (time since last run)
```

**Alternative: Azure** or Data bricks if needed
- Azure Data Factory for orchestration
- Azure Databricks for Spark-based processing
- ADLS Gen2 for storage
- Synapse Analytics for querying

**Alternative: GCP**
- Cloud Composer (Airflow) for orchestration
- Dataflow or Dataproc for processing
- Cloud Storage for data lake
- BigQuery for analytics

---

**Future Work:**

- Since we also have rejected transaction logs, we can use them to find user refund rate, cancel rate churn or other things that might help with the business growth long term

**Author:** Lalith :)
**Date:** November 2025  
**Version:** 1.0
