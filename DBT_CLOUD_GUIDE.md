# dbt Cloud Deployment Guide

This guide walks you through deploying this project using dbt Cloud, the managed platform for dbt.

## Prerequisites

- A [dbt Cloud](https://cloud.getdbt.com/) account (Free tier works).
- A GitHub account.
- This project pushed to a GitHub repository.

## Step 1: Connect to GitHub

1.  Log in to dbt Cloud.
2.  Go to **Account Settings** -> **Integrations**.
3.  Click **Connect GitHub Account** and authorize dbt Cloud.

## Step 2: Create a New Project

1.  From the top menu, click **Deploy** -> **Environments** or simply **Create new project**.
2.  **Name your project**: e.g., `Ecommerce Analytics`.
3.  **Choose Connections**:
    - For this demo, dbt Cloud cannot connect to your local DuckDB file.
    - **Option A (Mock)**: Choose "Snowflake" or "BigQuery" if you have a trial account.
    - **Option B (Demo Mode)**: Use the "dbt Quickstart" flow if available, or explain that in a real scenario, you would connect to a cloud warehouse (Snowflake, BigQuery, Redshift, Databricks).
    - *Note for Presentation*: You can show the "Import from Repository" screen to demonstrate how easy it is.

## Step 3: Configure the Repository

1.  Select **Git Clone**.
2.  Choose the repository you pushed this code to.
3.  dbt Cloud will detect `dbt_project.yml`.

## Step 4: Define an Environment

1.  Go to **Deploy** -> **Environments**.
2.  Click **Create Environment**.
3.  Name: `Production`.
4.  Type: `Deployment`.
5.  dbt Version: Select `Version 1.5` or later (matching your local version).

## Step 5: Create a Job

1.  In your `Production` environment, click **Create Job**.
2.  **Name**: `Nightly Build`.
3.  **Execution Settings**:
    - Commands:
        ```bash
        dbt build
        ```
4.  **Triggers**:
    - Turn on **Run on schedule**.
    - Set timing (e.g., Every day at 12:00 AM UTC).
    - Turn on **Run on Pull Requests** (CI/CD) if desired for a CI job.
5.  Click **Save**.

## Step 6: Run the Job

1.  Click **Run Now**.
2.  dbt Cloud will clone your repo, install dependencies (`dbt deps`), and run your models.
3.  Click into the run to show the detailed logs and the beautiful run history UI.


---

## 🔗 Airflow API Integration (Triggering Jobs)

You can orchestrate your dbt Cloud jobs using Airflow by using the **dbt Cloud Provider**.

### 1. Configure Airflow Connection

1.  Open the Airflow Web UI.
2.  Go to **Admin** -> **Connections**.
3.  Click **+** to add a new connection.
4.  **Connection ID**: `dbt_cloud_default`
5.  **Connection Type**: `dbt Cloud`
6.  **Account ID**: Your dbt Cloud Account ID (found in the URL: `https://cloud.getdbt.com/next/deploy/accounts/<ACCOUNT_ID>/...`)
7.  **API Token**: Generate a Service Account Token in dbt Cloud (**Account Settings** -> **Service Tokens**).

### 2. Sample DAG Implementation

I have provided a sample DAG in `dags/dbt_cloud_dag.py`. It uses the `DbtCloudRunJobOperator` which:
- Triggers the job using the dbt Cloud API.
- Polls for completion.
- Fails the Airflow task if the dbt Cloud job fails.

```python
from airflow.providers.dbt.cloud.operators.dbt import DbtCloudRunJobOperator

run_job = DbtCloudRunJobOperator(
    task_id="run_dbt_cloud_job",
    job_id=12345,  # Found in your Job URL
    wait_for_termination=True
)
```

### 3. Benefits of this approach
- **Hybrid Orchestration**: Use Airflow to trigger source ingestion and dbt Cloud to handle the heavy lifting of transformations.
- **Enhanced Visibility**: See your dbt Cloud runs directly within the Airflow DAG graph and logs.
- **Dependency Management**: Ensure complex non-dbt tasks (like data validation or external API calls) finish before your Cloud job starts.
