# Implementation Guide: Customer Transactions Data Model

This guide provides step-by-step instructions for implementing the Customer Transactions Data Model as outlined in the PRD. It covers environment setup, data ingestion, dbt transformations, testing, and automation.

## Prerequisites

- Snowflake account with ACCOUNTADMIN or equivalent permissions.
- AWS S3 bucket with permissions to configure event notifications.
- GitHub account for version control.
- Python 3.8+ installed locally.
- dbt CLI installed (`pip install dbt-snowflake`).
- Access to dbt Cloud (optional, for scheduling).

## Phase 1: Environment Setup (1-2 days)

### 1.1 Initialize dbt Project

1. Create a new directory for the project:

   ```
   mkdir dbt_customer_transactions
   cd dbt_customer_transactions
   ```

2. Initialize a new dbt project:

   ```
   dbt init customer_transactions
   cd customer_transactions
   ```

3. Configure dbt profiles for Snowflake. Edit `~/.dbt/profiles.yml`:

   ```yaml
   customer_transactions:
     target: dev
     outputs:
       dev:
         type: snowflake
         account: your_account.snowflakecomputing.com
         user: your_user
         password: your_password
         role: your_role
         database: your_database
         warehouse: your_warehouse
         schema: raw
         threads: 4
   ```

4. Test the connection:
   ```
   dbt debug
   ```

### 1.2 Set Up Snowflake Database and Schema

1. Create the necessary databases and schemas in Snowflake:

   ```sql
   CREATE DATABASE IF NOT EXISTS your_database;
   USE DATABASE your_database;
   CREATE SCHEMA IF NOT EXISTS raw;
   CREATE SCHEMA IF NOT EXISTS analytics;
   ```

2. Create raw tables for data ingestion:

   ```sql
   -- For customers.csv
   CREATE OR REPLACE TABLE raw.customers (
     customer_id NUMBER,
     customer_name VARCHAR,
     email VARCHAR,
     filename VARCHAR
   );

   -- For transactions.json (raw)
   CREATE OR REPLACE TABLE raw.stg_transactions (
     raw_data VARIANT
   );

   -- Flattened transactions
   CREATE OR REPLACE TABLE raw.transactions (
     transaction_id VARCHAR,
     customer_id NUMBER,
     amount NUMBER,
     transaction_date DATE,
     filename VARCHAR
   );
   ```

### 1.3 Configure S3 Bucket and Snowpipe

1. Create an S3 bucket (if not already done) and set up event notifications for Snowpipe.

2. Create Snowpipe for customers:

   ```sql
   CREATE OR REPLACE PIPE raw.customer_pipe
   AS COPY INTO raw.customers
   FROM @your_s3_stage
   FILE_FORMAT = (TYPE = CSV, SKIP_HEADER = 1, FIELD_DELIMITER = ',');
   ```

3. Create Snowpipe for transactions:

   ```sql
   CREATE OR REPLACE PIPE raw.transaction_pipe
   AS COPY INTO raw.stg_transactions
   FROM @your_s3_stage
   FILE_FORMAT = (TYPE = JSON);
   ```

4. Refresh the pipes after setup.

### 1.4 Set Up GitHub Repository

1. Initialize Git in the project directory:

   ```
   git init
   git add .
   git commit -m "Initial dbt project setup"
   ```

2. Create a new repository on GitHub and push:
   ```
   git remote add origin https://github.com/your_username/customer_transactions.git
   git push -u origin main
   ```

## Phase 2: dbt Model Development (2-3 days)

### 2.1 Create Staging Models

1. Create `models/staging/stg_customers.sql`:

   ```sql
   {{ config(materialized='view') }}

   SELECT
     customer_id,
     customer_name,
     email,
     filename
   FROM {{ source('raw', 'customers') }}
   ```

2. Create `models/staging/stg_transactions.sql`:

   ```sql
   {{ config(materialized='incremental', unique_key='transaction_id') }}

   SELECT
     raw_data:transaction_id::STRING AS transaction_id,
     raw_data:customer_id::NUMBER AS customer_id,
     raw_data:amount::NUMBER AS amount,
     TO_DATE(raw_data:transaction_date::STRING, 'YYYY-MM-DD') AS transaction_date,
     raw_data:filename::STRING AS filename
   FROM {{ source('raw', 'stg_transactions') }}
   {% if is_incremental() %}
     WHERE raw_data:transaction_date > (SELECT MAX(transaction_date) FROM {{ this }})
   {% endif %}
   ```

### 2.2 Create Dimension Models

1. Create `models/dimensions/dim_customers.sql`:

   ```sql
   {{ config(materialized='incremental', unique_key='customer_id') }}

   WITH cleaned_customers AS (
     SELECT
       customer_id,
       {{ clean_customer_names('customer_name') }} AS customer_name,
       email,
       filename
     FROM {{ ref('stg_customers') }}
   )

   SELECT
     customer_id,
     customer_name,
     email,
     filename
   FROM cleaned_customers
   {% if is_incremental() %}
     -- Type 1 SCD: Overwrite on changes
   {% endif %}
   ```

### 2.3 Create Fact Models

1. Create `models/facts/fact_transactions.sql`:

   ```sql
   {{ config(materialized='incremental', unique_key='transaction_id') }}

   SELECT
     t.transaction_id,
     c.customer_id AS customer_key,  -- Surrogate key if needed
     t.amount,
     t.transaction_date,
     t.filename
   FROM {{ ref('stg_transactions') }} t
   LEFT JOIN {{ ref('dim_customers') }} c ON t.customer_id = c.customer_id
   {% if is_incremental() %}
     WHERE t.transaction_date > (SELECT MAX(transaction_date) FROM {{ this }})
   {% endif %}
   ```

### 2.4 Implement Macros

1. Create `macros/clean_customer_names.sql`:

   ```sql
   {% macro clean_customer_names(column_name) %}
     REGEXP_REPLACE({{ column_name }}, '[#@$]', '')
   {% endmacro %}
   ```

2. Create `macros/remove_default_schema.sql`:
   ```sql
   {% macro remove_default_schema(table_name) %}
     REPLACE({{ table_name }}, 'default.', '')
   {% endmacro %}
   ```

## Phase 3: Testing and Scheduling (1-2 days)

### 3.1 Implement dbt Tests

1. Create tests for staging models. Add to `models/staging/stg_customers.yml`:

   ```yaml
   version: 2
   models:
     - name: stg_customers
       columns:
         - name: customer_id
           tests:
             - not_null
             - unique
         - name: customer_name
           tests:
             - not_null
         - name: email
           tests:
             - not_null
   ```

2. Similarly, add tests for other models.

3. Run tests:
   ```
   dbt test
   ```

### 3.2 Set Up Scheduling

1. For dbt Cloud: Create a new project, connect to the GitHub repo, and schedule daily runs for `dbt run` and `dbt test`.

2. For local/cron: Create a script `run_dbt.sh`:

   ```bash
   #!/bin/bash
   cd /path/to/project
   dbt run
   dbt test
   ```

3. Schedule with cron:
   ```
   crontab -e
   # Add: 0 2 * * * /path/to/run_dbt.sh  # Daily at 2 AM
   ```

## Phase 4: Validation and Deployment (1 day)

### 4.1 Validate Implementation

1. Upload sample data to S3 and verify Snowpipe ingestion.

2. Run dbt models and check table contents in Snowflake.

3. Ensure all tests pass.

4. Verify incremental loading and SCD behavior.

### 4.2 Deploy to Production

1. Update dbt profiles for production environment.

2. Merge changes to main branch and deploy via dbt Cloud or CI/CD.

3. Monitor daily jobs and set up alerts for failures.

## Troubleshooting

- **Connection Issues:** Verify Snowflake credentials and network access.
- **Ingestion Failures:** Check S3 permissions and Snowpipe configurations.
- **Model Errors:** Use `dbt compile` to debug SQL.
- **Test Failures:** Review data quality and adjust tests as needed.

## Additional Resources

- [dbt Documentation](https://docs.getdbt.com/)
- [Snowflake Documentation](https://docs.snowflake.com/)
- [Snowpipe Guide](https://docs.snowflake.com/en/user-guide/data-load-snowpipe.html)

This guide ensures a complete implementation aligned with the PRD requirements.
