# dbt Deployment Guide: Orchestration Strategies Comparison

This guide provides a comprehensive comparison of deployment strategies for dbt projects, showcasing four different approaches implemented in this demonstration project.

## Table of Contents
1. [Why Orchestration Matters](#why-orchestration-matters)
2. [Deployment Architecture Overview](#deployment-architecture-overview)
3. [Strategy 1: Apache Airflow](#strategy-1-apache-airflow)
4. [Strategy 2: Dagster](#strategy-2-dagster)
5. [Strategy 3: Prefect](#strategy-3-prefect)
6. [Strategy 4: CI/CD with GitHub Actions](#strategy-4-cicd-with-github-actions)
7. [Comparison Matrix](#comparison-matrix)
8. [Choosing the Right Strategy](#choosing-the-right-strategy)
9. [Best Practices](#best-practices)

---

## Why Orchestration Matters

### The Evolution from Manual Execution

**Stage 1: Manual Scripts**
```bash
# Run this every morning at 9 AM (don't forget!)
dbt seed
dbt run
dbt test
```

**Problems:**
- Human error (forgetting to run, wrong order)
- No visibility into execution status
- No retry logic on failure
- Can't handle dependencies between models
- Doesn't scale

**Stage 2: Cron Jobs**
```bash
0 9 * * * cd /path/to/dbt && dbt run
```

**Problems:**
- No visibility (check logs manually)
- No dependency management
- Hard to debug failures
- No alerting

**Stage 3: Modern Orchestration**
```python
# Automated, observable, and intelligent
airflow_dag = DAG(
    schedule_interval="@daily",
    retries=2,
    on_failure_callback=alert_team
)
```

**Benefits:**
✓ Automated scheduling
✓ Web UI for monitoring
✓ Dependency management
✓ Retry logic
✓ Alerting and notifications
✓ Resource management
✓ Logging and auditing

### Key Orchestration Capabilities

| Capability | Description | Business Value |
|------------|-------------|----------------|
| **Scheduling** | Run at specific times/intervals | Ensures fresh data for morning reports |
| **Dependencies** | Model A waits for Model B | Prevents incomplete data cascades |
| **Retries** | Auto-retry failed tasks | Handles transient network/DB issues |
| **Alerting** | Notify on failure | Enables rapid response |
| **Monitoring** | Real-time execution tracking | Visibility into pipeline health |
| **Resource Management** | Parallel execution, thread control | Faster builds, cost optimization |

---

## Deployment Architecture Overview

This project demonstrates a **shared infrastructure** approach where three orchestrators can run the same dbt project:

```
┌─────────────────────────────────────────────────────────────┐
│                     dbt Project (Shared)                    │
│  models/ | seeds/ | tests/ | dbt_project.yml               │
└────────────┬───────────┬───────────┬────────────────────────┘
             │           │           │
   ┌─────────┴───┐  ┌────┴────┐  ┌──┴──────┐
   │  Airflow    │  │ Dagster │  │ Prefect │
   │  Container  │  │Container│  │Container│
   └─────────────┘  └─────────┘  └─────────┘
             │           │           │
             └───────────┴───────────┘
                         │
                ┌────────▼────────┐
                │   PostgreSQL    │
                │  (Shared Warehouse)
                └─────────────────┘
```

**Key Design Decisions:**

1. **One Orchestrator at a Time** - Prevents port conflicts and resource contention
2. **Shared Database** - Data persists across orchestrator switches
3. **Internal Container Paths** - Avoids Windows file-locking issues
4. **Docker Compose** - Consistent environment across tools

---

## Strategy 1: Apache Airflow

### Overview

**Apache Airflow** is the most widely-used open-source orchestration platform, originally developed at Airbnb.

**Philosophy:** "Everything is a DAG (Directed Acyclic Graph)"

### Architecture

```
Airflow Scheduler
        │
        ├─ dbt_deps (Install packages)
        ├─ dbt_seed (Load CSV data)
        ├─ dbt_run (Build models)
        └─ dbt_test (Run tests)
```

### Implementation Details

**File:** `dags/dbt_dag.py`

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

with DAG(
    dag_id='dbt_ecommerce',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@daily',
    catchup=False,
    default_args={'retries': 2}
) as dag:

    dbt_deps = BashOperator(
        task_id='dbt_deps',
        bash_command='cd /opt/airflow/dbt && dbt deps --profiles-dir .'
    )

    dbt_seed = BashOperator(
        task_id='dbt_seed',
        bash_command='cd /opt/airflow/dbt && dbt seed --profiles-dir .'
    )

    dbt_run = BashOperator(
        task_id='dbt_run',
        bash_command='cd /opt/airflow/dbt && dbt run --profiles-dir .'
    )

    dbt_test = BashOperator(
        task_id='dbt_test',
        bash_command='cd /opt/airflow/dbt && dbt test --profiles-dir .'
    )

    # Define dependencies
    dbt_deps >> dbt_seed >> dbt_run >> dbt_test
```

### Hybrid Approach: Airflow + dbt Cloud

**File:** `dags/dbt_cloud_dag.py`

This demonstrates a **hybrid deployment pattern** where:
- **Airflow** handles scheduling and orchestration
- **dbt Cloud** handles the transformation execution

```python
from airflow import DAG
from airflow.providers.dbt.cloud.operators.dbt import DbtCloudRunJobOperator

with DAG(
    dag_id='dbt_cloud_job',
    schedule_interval='@daily'
) as dag:

    trigger_dbt_cloud = DbtCloudRunJobOperator(
        task_id='run_dbt_cloud_job',
        job_id=12345,  # Your dbt Cloud job ID
        check_interval=30,
        timeout=3600,
        wait_for_termination=True,
        dbt_cloud_conn_id='dbt_cloud_default'
    )
```

**When to Use This:**
- Need Airflow for non-dbt tasks (data extraction, external APIs)
- Want dbt Cloud features (IDE, documentation hosting)
- Prefer managed dbt infrastructure

### Startup & Access

**Start Airflow:**
```bash
docker-compose -f docker-compose-airflow.yaml up -d
```

**Access Web UI:**
- URL: http://localhost:8080
- Username: `admin`
- Password: `admin`

**Stop Airflow:**
```bash
docker-compose -f docker-compose-airflow.yaml down
```

### Airflow Features Demonstrated

#### 1. Task-Level Granularity
Each dbt command is a separate task, giving you:
- Individual task logs
- Per-task retry logic
- Task-level monitoring

#### 2. DAG Visualization
Airflow provides a graph view showing:
- Task dependencies
- Task status (success, failed, running)
- Execution timeline

#### 3. Retry Configuration
```python
default_args = {
    'retries': 2,
    'retry_delay': timedelta(minutes=5)
}
```

#### 4. Scheduling Options
```python
schedule_interval='@daily'      # Every day at midnight
schedule_interval='0 9 * * *'   # Every day at 9 AM
schedule_interval='@hourly'     # Every hour
schedule_interval=None          # Manual trigger only
```

### Pros & Cons

**Pros:**
✓ Industry standard with huge community
✓ Extensive integrations (400+ operators)
✓ Rich UI with execution history
✓ Task-level control and monitoring
✓ Mature ecosystem

**Cons:**
✗ Steeper learning curve
✗ Heavier infrastructure requirements
✗ Requires separate scheduler and webserver
✗ Can be over-engineered for simple workflows

**Best For:**
- Complex data pipelines with multiple tools
- Organizations already using Airflow
- Scenarios requiring extensive integrations

---

## Strategy 2: Dagster

### Overview

**Dagster** is a modern orchestration platform designed around the concept of "Software-Defined Assets".

**Philosophy:** "Focus on what you're building (assets), not how you're building it (tasks)"

### Architecture

```
Dagster Assets Graph
        │
        ├─ stg_customers (asset)
        ├─ stg_orders (asset)
        ├─ int_order_items (asset)
        ├─ dim_customers (asset)
        └─ fct_daily_sales (asset)
```

### Implementation Details

**File:** `dagster/dbt_pipeline.py`

```python
from dagster import AssetExecutionContext, Definitions
from dagster_dbt import DbtCliResource, dbt_assets, DbtProject

# Point to your dbt project
dbt_project = DbtProject(
    project_dir="/dbt",
    packaged_project_dir="/dbt",
)

# Automatically create assets from dbt models
@dbt_assets(manifest=dbt_project.manifest_path)
def ecommerce_dbt_assets(context: AssetExecutionContext, dbt: DbtCliResource):
    # Run all models
    yield from dbt.cli(["build"], context=context).stream()

# Define the Dagster deployment
defs = Definitions(
    assets=[ecommerce_dbt_assets],
    resources={
        "dbt": DbtCliResource(project_dir="/dbt"),
    },
)
```

### Key Concepts

#### 1. Software-Defined Assets
Every dbt model becomes a **Dagster Asset**:

```
dbt Model: dim_customers.sql
    ↓
Dagster Asset: dim_customers
    - Knows dependencies (upstream models)
    - Knows downstream consumers
    - Tracks materialization history
```

#### 2. Automatic Lineage
Dagster parses your `manifest.json` to understand:
- Which models depend on which others
- Column-level lineage
- Test relationships

#### 3. Selective Materialization
```python
# Materialize only specific assets
dbt.cli(["build", "--select", "stg_customers+"], context=context)
```

### Startup & Access

**Start Dagster:**
```bash
docker-compose -f docker-compose-dagster.yaml up -d
```

**Access Web UI:**
- URL: http://localhost:3000

**Stop Dagster:**
```bash
docker-compose -f docker-compose-dagster.yaml down
```

### Dagster Features Demonstrated

#### 1. Asset Catalog
View all data assets in one place:
- dbt models
- Seeds
- Tests (as asset checks)

#### 2. Lineage Graph
Visual representation of:
- Upstream dependencies
- Downstream consumers
- Cross-model relationships

#### 3. Asset Materialization History
Track when each asset was last materialized:
- Timestamp
- Success/failure status
- Metadata (rows, runtime, etc.)

#### 4. Sensors & Schedules
```python
from dagster import ScheduleDefinition

daily_schedule = ScheduleDefinition(
    job=ecommerce_job,
    cron_schedule="0 9 * * *"
)
```

### Pros & Cons

**Pros:**
✓ Modern, intuitive UI
✓ Asset-centric paradigm (matches dbt's model approach)
✓ Automatic lineage from dbt manifest
✓ Great for data quality monitoring
✓ Column-level lineage

**Cons:**
✗ Newer tool (smaller community)
✗ Less third-party integrations than Airflow
✗ Requires understanding of "asset" paradigm
✗ Some advanced features require Dagster Cloud

**Best For:**
- Data teams focused on data assets vs tasks
- Organizations prioritizing data lineage
- Modern data stacks (dbt + cloud warehouse)

---

## Strategy 3: Prefect

### Overview

**Prefect** is a Python-native orchestration platform emphasizing developer experience and flexibility.

**Philosophy:** "Write Python, not YAML or DSLs"

### Architecture

```
Prefect Flow
        │
        ├─ @task dbt_seed()
        ├─ @task dbt_run()
        ├─ @task dbt_test()
        └─ @task dbt_docs_generate()
```

### Implementation Details

**File:** `prefect/dbt_flow.py`

```python
from prefect import flow, task
from prefect_dbt.cli.commands import DbtCoreOperation

@task(name="dbt_seed")
def dbt_seed():
    return DbtCoreOperation(
        commands=["dbt seed --profiles-dir /dbt"],
        project_dir="/dbt",
        profiles_dir="/dbt"
    ).run()

@task(name="dbt_run")
def dbt_run():
    return DbtCoreOperation(
        commands=["dbt run --profiles-dir /dbt"],
        project_dir="/dbt",
        profiles_dir="/dbt"
    ).run()

@task(name="dbt_test")
def dbt_test():
    return DbtCoreOperation(
        commands=["dbt test --profiles-dir /dbt"],
        project_dir="/dbt",
        profiles_dir="/dbt"
    ).run()

@task(name="dbt_docs_generate")
def dbt_docs_generate():
    return DbtCoreOperation(
        commands=["dbt docs generate --profiles-dir /dbt"],
        project_dir="/dbt",
        profiles_dir="/dbt"
    ).run()

@flow(name="dbt_ecommerce_analytics")
def dbt_ecommerce_flow():
    """E-commerce analytics dbt pipeline"""
    seed_result = dbt_seed()
    run_result = dbt_run()
    test_result = dbt_test()
    docs_result = dbt_docs_generate()
    
    return {
        "seed": seed_result,
        "run": run_result,
        "test": test_result,
        "docs": docs_result
    }

if __name__ == "__main__":
    dbt_ecommerce_flow()
```

### Startup & Access

**Start Prefect:**
```bash
docker-compose -f docker-compose-prefect.yaml up -d
```

**Access Web UI:**
- URL: http://localhost:4200

**Stop Prefect:**
```bash
docker-compose -f docker-compose-prefect.yaml down
```

### Prefect Features Demonstrated

#### 1. Pure Python
Everything is written in Python:
- No YAML configs
- No DSLs to learn
- Full IDE support (autocomplete, type hints)

#### 2. Decorators for Tasks & Flows
```python
@task(retries=3, retry_delay_seconds=60)
def my_task():
    # Task logic here
    pass

@flow(name="My Flow")
def my_flow():
    result = my_task()
    return result
```

#### 3. Dynamic Workflows
```python
@flow
def dynamic_dbt_flow(models: list):
    for model in models:
        run_dbt_model(model)
```

#### 4. Event-Driven Triggers
```python
from prefect.events import Event

# Trigger flow when S3 file arrives
@flow
def process_new_data(event: Event):
    dbt_run()
```

### Pros & Cons

**Pros:**
✓ Python-native (familiar for data engineers)
✓ Excellent developer experience
✓ Flexible and dynamic workflows
✓ Modern UI with great observability
✓ Easy to integrate with Python code

**Cons:**
✗ Smaller ecosystem than Airflow
✗ Requires Python knowledge
✗ Some features require Prefect Cloud
✗ Less mature than Airflow

**Best For:**
- Python-heavy data teams
- Dynamic, event-driven workflows
- Teams prioritizing developer experience

---

## Strategy 4: CI/CD with GitHub Actions

### Overview

**GitHub Actions** provides automated testing and deployment triggered by git events.

**Philosophy:** "Continuous Integration / Continuous Deployment"

### Architecture

```
Git Event (push/PR)
        │
        ├─ Test Job (on PR)
        │   ├─ dbt deps
        │   ├─ dbt seed (dev)
        │   ├─ dbt run (dev)
        │   └─ dbt test (dev)
        │
        └─ Deploy Job (on main push)
            ├─ dbt seed (prod)
            ├─ dbt run (prod)
            ├─ dbt test (prod)
            └─ dbt docs generate (prod)
```

### Implementation Details

**File:** `.github/workflows/dbt-deploy.yml`

```yaml
name: dbt CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dbt dependencies
        run: |
          pip install -r requirements.txt
          dbt deps --profiles-dir .

      - name: Run dbt debug
        run: dbt debug --profiles-dir .

      - name: Seed data
        run: dbt seed --profiles-dir . --target dev

      - name: Run models
        run: dbt run --profiles-dir . --target dev

      - name: Run tests
        run: dbt test --profiles-dir . --target dev

  deploy:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          dbt deps --profiles-dir .

      - name: Deploy to production
        run: |
          dbt seed --profiles-dir . --target prod
          dbt run --profiles-dir . --target prod
          dbt test --profiles-dir . --target prod
          dbt docs generate --profiles-dir . --target prod
```

### GitHub Actions Features

#### 1. Trigger Events
```yaml
on:
  push:              # On every push
  pull_request:      # On PR creation/update
  schedule:          # Cron schedule
    - cron: '0 9 * * *'
  workflow_dispatch: # Manual trigger
```

#### 2. Environment-Based Deployment
```yaml
- name: Run dbt (dev)
  run: dbt run --target dev
  
- name: Run dbt (prod)
  run: dbt run --target prod
  if: github.ref == 'refs/heads/main'
```

#### 3. Secrets Management
```yaml
env:
  DBT_POSTGRES_HOST: ${{ secrets.POSTGRES_HOST }}
  DBT_POSTGRES_USER: ${{ secrets.POSTGRES_USER }}
  DBT_POSTGRES_PASSWORD: ${{ secrets.POSTGRES_PASSWORD }}
```

#### 4. PR Status Checks
- Tests must pass before merge
- Visual indicators on PR
- Detailed logs for failures

### Workflow Examples

#### Pull Request Workflow
```
1. Developer creates PR
2. GitHub Actions triggers "test" job
3. dbt runs against dev environment
4. If tests fail → PR shows red X
5. If tests pass → PR shows green checkmark
6. Code reviewer can see test results
7. PR is merged
```

#### Production Deployment Workflow
```
1. PR merged to main
2. GitHub Actions triggers "deploy" job
3. dbt runs against prod environment
4. Models are built in production schema
5. Tests validate production data
6. Documentation is generated
7. (Optional) Slack notification sent
```

### Pros & Cons

**Pros:**
✓ No infrastructure to manage
✓ Tight integration with git workflow
✓ Automated testing on every PR
✓ Environment-based deployment
✓ Free for public repos (generous free tier for private)

**Cons:**
✗ Limited to git-triggered events
✗ No web UI for ad-hoc runs
✗ Less suitable for scheduled batch jobs
✗ Requires understanding of YAML

**Best For:**
- Git-centric workflows
- Automated testing on pull requests
- Small to medium teams
- Projects already on GitHub

---

## Comparison Matrix

| Feature | Airflow | Dagster | Prefect | GitHub Actions |
|---------|---------|---------|---------|----------------|
| **Paradigm** | Task-based DAGs | Asset-based | Python flows | CI/CD pipelines |
| **Learning Curve** | Steep | Moderate | Gentle | Moderate |
| **UI Quality** | Good | Excellent | Excellent | GitHub UI |
| **Scheduling** | Cron, sensors | Cron, sensors | Cron, events | Git events, cron |
| **dbt Integration** | Cosmos, operators | Native | Native | CLI |
| **Lineage Visibility** | Manual | Automatic | Manual | None |
| **Infrastructure** | Scheduler + webserver + DB | All-in-one | Server + worker | Serverless |
| **Local Development** | Docker | Docker | Docker | N/A |
| **Production Maturity** | Very mature | Maturing | Maturing | Mature |
| **Community Size** | Very large | Growing | Growing | Very large |
| **Best For** | Complex pipelines | Data assets | Python teams | Git workflows |

### Use Case Recommendations

| Scenario | Recommended Tool | Reason |
|----------|------------------|--------|
| **Complex multi-tool pipeline** | Airflow | 400+ integrations |
| **Data lineage focus** | Dagster | Automatic asset lineage |
| **Python-heavy workflows** | Prefect | Python-native |
| **PR-based testing** | GitHub Actions | Git integration |
| **Startup, simple pipeline** | Prefect or Dagster | Easier setup |
| **Enterprise, existing Airflow** | Airflow | Leverage existing investment |
| **dbt-only pipeline** | Any (all work well) | Choose based on team preference |

---

## Choosing the Right Strategy

### Decision Framework

#### 1. Team Skillset
- **Python-heavy team?** → Prefect
- **DevOps/infrastructure team?** → Airflow
- **Data-focused team?** → Dagster
- **Small team, tight on resources?** → GitHub Actions

#### 2. Pipeline Complexity
- **Simple dbt-only?** → Prefect or Dagster
- **Complex multi-tool ETL?** → Airflow
- **Git-centric, automated testing?** → GitHub Actions

#### 3. Existing Infrastructure
- **Already using Airflow?** → Stick with Airflow
- **Already on GitHub?** → Start with GitHub Actions
- **Greenfield project?** → Dagster or Prefect

#### 4. Budget & Resources
- **Limited infrastructure budget?** → GitHub Actions (serverless)
- **Need enterprise support?** → Airflow (Astronomer), Dagster (Dagster Labs), Prefect (Prefect Cloud)
- **Self-hosted preferred?** → Airflow or Dagster

#### 5. Feature Requirements
- **Need data lineage?** → Dagster
- **Need extensive integrations?** → Airflow
- **Need Python flexibility?** → Prefect
- **Need automated PR testing?** → GitHub Actions

---

## Best Practices

### 1. Environment Separation

**Always use separate dev/prod targets:**

```yaml
# profiles.yml
ecommerce_demo:
  target: dev
  outputs:
    dev:
      type: postgres
      host: localhost
      schema: analytics_dev
    prod:
      type: postgres
      host: prod-warehouse.company.com
      schema: analytics_prod
```

**Why?**
- Test changes safely in dev
- Prevent accidental production overwrites
- Enable CI/CD testing

### 2. Orchestrator-Agnostic dbt Code

**Keep dbt code independent of orchestrator:**

✓ **Good:**
```sql
-- models/staging/stg_customers.sql
SELECT * FROM {{ ref('raw_customers') }}
```

✗ **Bad:**
```sql
-- Don't hardcode orchestrator-specific logic
SELECT * FROM {{ var('airflow_execution_date') }}
```

**Why?**
- Easier to switch orchestrators
- Simpler local development
- More portable code

### 3. Incremental Models for Large Tables

**For production at scale:**

```sql
-- models/marts/fct_orders.sql
{{
    config(
        materialized='incremental',
        unique_key='order_id',
        on_schema_change='append_new_columns'
    )
}}

SELECT *
FROM {{ ref('stg_orders') }}
{% if is_incremental() %}
WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

**Why?**
- Faster builds (only process new data)
- Lower warehouse costs
- Essential for large datasets

### 4. Monitoring & Alerting

**Airflow:**
```python
on_failure_callback = lambda context: send_slack_alert(context)
```

**Dagster:**
```python
@asset(op_tags={"slack_on_failure": "data-alerts"})
def my_asset():
    pass
```

**Prefect:**
```python
@task(on_failure=[send_slack_alert])
def my_task():
    pass
```

**GitHub Actions:**
```yaml
- name: Notify on failure
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
```

### 5. Documentation as Code

**Generate and host dbt docs:**

```bash
dbt docs generate
dbt docs serve  # Local

# Or host on S3/GCS for team access
dbt docs generate
aws s3 sync target/ s3://dbt-docs-bucket/
```

**Why?**
- Living documentation
- Column-level lineage
- Searchable model catalog

### 6. Selective Execution

**Don't rebuild everything every time:**

```bash
# Only changed models
dbt run --select state:modified+

# Only specific models
dbt run --select stg_orders+

# Only specific tags
dbt run --select tag:daily
```

**Why?**
- Faster builds
- Lower costs
- More frequent updates

### 7. Testing in CI/CD

**Always test before deploying:**

```yaml
# Pull Request: Test only
on: pull_request
  run: dbt test --target dev

# Main Branch: Test + Deploy
on: push (main)
  run: |
    dbt test --target prod
    if [ $? -eq 0 ]; then
      dbt run --target prod
    fi
```

### 8. Resource Management

**Configure threads based on environment:**

```yaml
# profiles.yml
dev:
  threads: 4   # Local development

prod:
  threads: 16  # Production warehouse
```

**Why?**
- Optimize warehouse utilization
- Prevent overwhelming local database
- Control costs

---

## Summary

This dbt project demonstrates **four distinct deployment strategies**:

1. **Airflow** - Task-based orchestration with extensive integrations
2. **Dagster** - Asset-centric with automatic lineage
3. **Prefect** - Python-native with developer-friendly experience
4. **GitHub Actions** - Git-integrated CI/CD automation

**Key Takeaways:**

| Strategy | Best For | Key Strength |
|----------|----------|--------------|
| Airflow | Complex pipelines | Ecosystem & integrations |
| Dagster | Data lineage focus | Asset-centric design |
| Prefect | Python teams | Developer experience |
| GitHub Actions | Automated testing | Git integration |

**There is no "best" orchestrator—only the best fit for your team, use case, and infrastructure.**

---

## Next Steps

- **[Try each orchestrator](SETUP.md)** with the provided Docker Compose files
- **[Review the dbt models](DATA_MODEL.md)** to understand what's being orchestrated
- **[Explore testing strategies](TESTING_GUIDE.md)** integrated with orchestration
- **Choose one orchestrator** and deploy it to your production environment

---

**Questions or suggestions?** Open an issue on GitHub or refer to the orchestration tool documentation:
- [Airflow Docs](https://airflow.apache.org/docs/)
- [Dagster Docs](https://docs.dagster.io/)
- [Prefect Docs](https://docs.prefect.io/)
- [GitHub Actions Docs](https://docs.github.com/en/actions)
