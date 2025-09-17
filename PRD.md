# Product Requirements Document (PRD): Customer Transactions Data Model

## 1. Product Overview

This PRD outlines the requirements for developing a data model to analyze customer transactions. The system will ingest customer and transaction data from S3, process it using dbt transformations in Snowflake, and support incremental updates, data quality tests, and automated scheduling. The primary goal is to create a robust, scalable data warehouse structure for business intelligence and analytics.

**Tech Stacks:** Python, GitHub, dbt, Snowflake.

## 2. Objectives

- Build a basic data model for customer transaction analysis.
- Implement staging, fact, and dimension tables with proper relationships.
- Ensure data quality through tests and cleansing macros.
- Enable incremental data loading for efficiency.
- Automate daily data ingestion and processing via Snowpipe and dbt jobs.
- Support Type 1 Slowly Changing Dimensions (SCD) for customer name updates.

## 3. Functional Requirements

### 3.1 Data Ingestion

- Ingest `customers.csv` and `transactions.json` files from S3 into Snowflake using COPY INTO commands.
- Handle CSV and JSON formats appropriately.
- Use Snowpipe for automatic loading triggered by S3 event notifications.

### 3.2 Data Transformation

- Create a staging table (`stg_transactions`) for initial data processing.
- Build a fact table (`fact_transactions`) with incremental materialization.
- Develop dimension tables (`dim_customers`) using surrogate keys.
- Implement Type 1 SCD for customer name changes.
- Apply data cleansing: Remove special characters (@, #, $) from customer names using a custom macro.
- Use a macro to remove default schema names where necessary.

### 3.3 Data Quality and Testing

- Implement dbt tests to validate data quality across all tables.
- Ensure relationships between fact and dimension tables are validated.
- Verify that all columns pass defined tests.

### 3.4 Scheduling and Automation

- Configure Snowpipe for auto-loading data upon S3 uploads.
- Schedule daily dbt jobs (using dbt Cloud or cron + dbt CLI) to run `dbt run` and `dbt test`.

## 4. Technical Requirements

### 4.1 Technologies

- **Python:** For scripting and automation tasks (e.g., data preprocessing if needed).
- **GitHub:** Version control for dbt project code.
- **dbt:** Data transformation tool for building models, macros, and tests.
- **Snowflake:** Cloud data warehouse for storage and processing.

### 4.2 Environment Setup

- Snowflake account with necessary permissions for data loading and querying.
- S3 bucket configured for data uploads with event notifications for Snowpipe.
- dbt project initialized and connected to Snowflake.
- GitHub repository for code collaboration and CI/CD.

### 4.3 Dependencies

- dbt packages for incremental models and SCD implementations.
- Python libraries (e.g., boto3 for S3 interactions if required).

## 5. Data Model Specifications

### 5.1 Source Data

- **customers.csv:** Fields - customer_id (number), customer_name (varchar), email (varchar), filename (varchar).
- **transactions.json:** Nested structure with customer_id, transaction_details array (transaction_id, amount, transaction_date, filename).

### 5.2 Snowflake Tables

- **raw.source.customers:** As per DDL (customer_id, customer_name, email, filename).
- **raw.source.stg_transactions:** VARIANT type for raw JSON data.
- **raw.source.transactions:** Flattened transactions (transaction_id, customer_id, amount, transaction_date, filename).

### 5.3 dbt Models

- **stg_transactions:** Staging model for transaction data preparation.
- **fact_transactions:** Fact table with incremental materialization, surrogate keys.
- **dim_customers:** Dimension table with Type 1 SCD, cleaned customer names.

### 5.4 Macros

- **clean_customer_names:** Removes special characters from customer names.
- **remove_default_schema:** Strips default schema prefixes.

### 5.5 Keys and Relationships

- Use surrogate keys for dimension tables.
- Foreign key relationships between fact_transactions and dim_customers.

## 6. Data Ingestion and Scheduling

### 6.1 Ingestion Process

- Upload source files to S3.
- Snowpipe auto-loads data into raw Snowflake tables via COPY INTO.
- dbt transformations process data into modeled tables.

### 6.2 Scheduling

- **Snowpipe:** Event-driven loading from S3.
- **dbt Jobs:** Daily execution of dbt run and dbt test (e.g., via dbt Cloud or cron).

## 7. Acceptance Criteria

- `dim_customers` table contains cleaned customer names (special characters removed).
- `fact_transactions` table updates incrementally with only new transactions.
- All dbt tests pass for data quality.
- Relationships between fact and dimension tables are validated.
- Daily jobs run successfully without manual intervention.
- Data from S3 is ingested accurately into Snowflake.

## 8. Assumptions and Constraints

- Source data formats remain consistent as specified.
- Snowflake and S3 access is pre-configured.
- dbt project follows standard folder structure.
- No real-time processing required; daily batch is sufficient.
- Budget and resource constraints align with cloud services used.

## 9. Risks and Mitigations

- Data quality issues: Mitigated by dbt tests and cleansing macros.
- Ingestion failures: Monitored via Snowflake logs and alerts.
- Scheduling conflicts: Use dbt Cloud for reliable job orchestration.

## 10. Timeline and Milestones

- Phase 1: Environment setup and data ingestion (1-2 days).
- Phase 2: dbt model development (2-3 days).
- Phase 3: Testing and scheduling (1-2 days).
- Phase 4: Validation and deployment (1 day).

This PRD serves as the blueprint for implementation. Any changes should be reviewed and approved via GitHub pull requests.
