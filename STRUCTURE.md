# Project Structure & File Overview

This project is organized into several key areas: dbt code, orchestrator configurations, and deployment scripts.

## 📁 Root Directory
*   `README.md`: Main entry point and project overview.
*   `dbt_project.yml`: Core dbt configuration (models, paths, vars).
*   `profiles.yml`: Connection configuration for PostgreSQL.
*   `packages.yml`: External dbt package dependencies.
*   `requirements.txt`: Python dependencies for the dbt environment.

## 📁 dbt Project Details
*   `models/`: Core logic of the analytics pipeline.
    *   `staging/`: Initial data cleaning and casting.
    *   `intermediate/`: Complex joins and business logic.
    *   `marts/`: Final reporting tables (Dimensions and Facts).
*   `seeds/`: Raw CSV data sources (`customers.csv`, `orders.csv`, `products.csv`).
*   `tests/`: Custom SQL data quality tests.

## 📁 Orchestrator Integration
*   `dags/`: Airflow DAG definitions.
    *   `dbt_dag.py`: Uses Astronomer Cosmos to dynamically parse dbt models.
*   `dagster/`: Dagster-specific assets and pipelines.
    *   `dbt_pipeline.py`: Defines Dagster assets based on dbt models.
*   `prefect/`: Prefect flow definitions.
    *   `dbt_flow.py`: Executes dbt commands as Prefect tasks.

## 📁 Infrastructure & DevOps
*   `postgres/`: Custom PostgreSQL setup with timezone fixes.
*   `docker-compose-*.yaml`: Orchestrator-specific container configurations.
*   `.dockerignore`: Excludes transient files from container builds to ensure stability.

## 📁 Build Artifacts (Container Internal)
*   `/opt/.../dbt_packages_internal/`: Isolated path for dbt packages within containers.
*   `/opt/.../dbt_target_internal/`: Isolated path for compiled SQL and logs.
