# Setup Guide: Local Environment

This guide walks you through setting up the dbt E-Commerce Analytics demonstration project on your local machine, including both Docker-based orchestration and native dbt development.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Quick Start (Docker Orchestrators)](#quick-start-docker-orchestrators)
3. [Local dbt Development](#local-dbt-development)
4. [Troubleshooting](#troubleshooting)
5. [Next Steps](#next-steps)

---

## Prerequisites

### Required Software

| Tool | Version | Purpose | Installation |
|------|---------|---------|--------------|
| **Docker Desktop** | Latest | Run PostgreSQL and orchestrators | [Download](https://www.docker.com/products/docker-desktop) |
| **Git** | 2.x+ | Clone repository | [Download](https://git-scm.com/downloads) |
| **Python** | 3.10-3.11 | Run dbt locally | [Download](https://www.python.org/downloads/) |

### Optional Software

| Tool | Purpose |
|------|---------|
| **VS Code** | Recommended IDE with dbt extensions |
| **DBeaver / pgAdmin** | Database GUI for exploring tables |
| **Postman** | For testing dbt Cloud API (hybrid deployment) |

### System Requirements

- **RAM:** 8GB minimum, 16GB recommended
- **Disk Space:** 10GB free space
- **OS:** Windows 10/11, macOS, or Linux
- **Docker:** Ensure Docker Desktop is running before proceeding

---

## Quick Start (Docker Orchestrators)

### Step 1: Clone the Repository

```bash
git clone <repository-url>
cd dbt_deployment
```

### Step 2: Choose and Start an Orchestrator

**Important:** Run only **one orchestrator at a time** to avoid port conflicts and resource issues.

#### Option A: Apache Airflow

**Start Airflow:**
```bash
docker-compose -f docker-compose-airflow.yaml up -d
```

**Wait for initialization** (2-3 minutes):
```bash
docker-compose -f docker-compose-airflow.yaml logs -f airflow-init
```

**Access Airflow UI:**
- URL: http://localhost:8080
- Username: `admin`
- Password: `admin`

**Run the DAG:**
1. Navigate to the DAGs page
2. Enable the `dbt_ecommerce` DAG
3. Click the "Play" button to trigger manually

**Stop Airflow:**
```bash
docker-compose -f docker-compose-airflow.yaml down
```

---

#### Option B: Dagster

**Start Dagster:**
```bash
docker-compose -f docker-compose-dagster.yaml up -d
```

**Wait for startup** (1-2 minutes):
```bash
docker-compose -f docker-compose-dagster.yaml logs -f dagster_webserver
```

**Access Dagster UI:**
- URL: http://localhost:3000

**Materialize Assets:**
1. Navigate to the "Assets" tab
2. Select all dbt assets
3. Click "Materialize selected"

**Stop Dagster:**
```bash
docker-compose -f docker-compose-dagster.yaml down
```

---

#### Option C: Prefect

**Start Prefect:**
```bash
docker-compose -f docker-compose-prefect.yaml up -d
```

**Wait for startup** (1-2 minutes):
```bash
docker-compose -f docker-compose-prefect.yaml logs -f prefect-server
```

**Access Prefect UI:**
- URL: http://localhost:4200

**Run the Flow:**
1. The flow runs automatically on container startup
2. View flow runs in the UI under "Flow Runs"
3. To run manually, exec into the container:
```bash
docker exec -it prefect-worker python /prefect/dbt_flow.py
```

**Stop Prefect:**
```bash
docker-compose -f docker-compose-prefect.yaml down
```

---

### Step 3: Verify Data

**Connect to PostgreSQL:**
```bash
# Find the postgres container name
docker ps | grep postgres

# Connect to the database
docker exec -it <postgres-container-name> psql -U airflow -d airflow
```

**Query the data:**
```sql
-- List all schemas
\dn

-- View staging models
SELECT * FROM staging.stg_customers LIMIT 5;

-- View marts
SELECT * FROM marts.dim_customers;
SELECT * FROM marts.fct_daily_sales;

-- Summary report
SELECT * FROM marts.rpt_sales_summary;

-- Exit
\q
```

---

## Local dbt Development

For local development and testing **without Docker orchestrators**:

### Step 1: Install Python Dependencies

**Create virtual environment (recommended):**
```bash
python -m venv venv

# Windows
venv\Scripts\activate

# macOS/Linux
source venv/bin/activate
```

**Install dbt and dependencies:**
```bash
pip install -r requirements.txt
```

**Verify installation:**
```bash
dbt --version
```

Expected output:
```
Core:
  - installed: 1.8.0
  - latest:    1.8.0 - Up to date!

Plugins:
  - postgres: 1.8.0 - Up to date!
```

---

### Step 2: Install dbt Packages

```bash
dbt deps --profiles-dir .
```

This installs `dbt_utils` and other dependencies defined in `packages.yml`.

---

### Step 3: Start PostgreSQL (Standalone)

If not using an orchestrator, start just the PostgreSQL container:

```bash
docker run -d \
  --name dbt-postgres \
  -e POSTGRES_USER=airflow \
  -e POSTGRES_PASSWORD=airflow \
  -e POSTGRES_DB=airflow \
  -p 5432:5432 \
  postgres:16
```

---

### Step 4: Run dbt Commands

**Check connection:**
```bash
dbt debug --profiles-dir .
```

**Load seed data:**
```bash
dbt seed --profiles-dir .
```

**Build all models:**
```bash
dbt run --profiles-dir .
```

**Run tests:**
```bash
dbt test --profiles-dir .
```

**Build and test (combined):**
```bash
dbt build --profiles-dir .
```

**Generate documentation:**
```bash
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

Documentation will be available at http://localhost:8080

---

### Step 5: Selective Execution

**Run specific model:**
```bash
dbt run --select stg_customers --profiles-dir .
```

**Run model and downstream dependencies:**
```bash
dbt run --select stg_customers+ --profiles-dir .
```

**Run model and upstream dependencies:**
```bash
dbt run --select +dim_customers --profiles-dir .
```

**Run specific layer:**
```bash
dbt run --select staging --profiles-dir .
dbt run --select marts --profiles-dir .
```

**Run specific tests:**
```bash
dbt test --select stg_customers --profiles-dir .
dbt test --select test_type:singular --profiles-dir .
```

---

## Troubleshooting

### Common Issues

#### Issue: Port Already in Use

**Error:**
```
Error starting userland proxy: listen tcp4 0.0.0.0:8080: bind: address already in use
```

**Solution:**
1. Check if another orchestrator is running:
```bash
docker ps
```

2. Stop conflicting containers:
```bash
docker-compose -f docker-compose-[tool].yaml down
```

3. Or change the port in the docker-compose file

---

#### Issue: Docker Containers Won't Start

**Error:**
```
Container exited with code 1
```

**Solution:**
1. Check logs:
```bash
docker-compose -f docker-compose-[tool].yaml logs
```

2. Ensure Docker Desktop has enough resources:
   - Settings → Resources → Memory: At least 4GB
   - Settings → Resources → CPUs: At least 2

3. Restart Docker Desktop

---

#### Issue: dbt Connection Failed

**Error:**
```
Database Error: could not connect to server
```

**Solution:**
1. Verify PostgreSQL is running:
```bash
docker ps | grep postgres
```

2. Check `profiles.yml` connection settings:
```yaml
host: localhost  # or 'postgres' if running in container
port: 5432
user: airflow
password: airflow
dbname: airflow
```

3. Test connection:
```bash
dbt debug --profiles-dir .
```

---

#### Issue: Windows File Locking

**Error:**
```
Device busy or permission denied
```

**Solution:**
This is why we use internal container paths. If you still encounter this:

1. Stop all containers:
```bash
docker-compose -f docker-compose-airflow.yaml down
docker-compose -f docker-compose-dagster.yaml down
docker-compose -f docker-compose-prefect.yaml down
```

2. Wait 10 seconds

3. Restart the desired orchestrator

---

#### Issue: Test Failures

**Error:**
```
FAIL 3 no_negative_quantities
```

**Solution:**
This indicates data quality issues. To debug:

1. Check stored failures (if enabled):
```sql
SELECT * FROM dbt_test__audit.no_negative_quantities;
```

2. Run the test query manually:
```bash
dbt show --select tests/no_negative_quantities.sql --profiles-dir .
```

3. Fix source data or adjust test logic

---

## Next Steps

### Explore the Documentation

- **[Testing Guide](TESTING_GUIDE.md)** - Learn about the testing strategies implemented
- **[Deployment Guide](DEPLOYMENT_GUIDE.md)** - Compare orchestration patterns
- **[Data Model](DATA_MODEL.md)** - Understand the models and business logic

### Extend the Project

1. **Add new models** - Create additional transformations
2. **Add custom tests** - Implement business-specific validations
3. **Try different orchestrators** - Compare Airflow, Dagster, and Prefect
4. **Configure CI/CD** - Set up GitHub Actions for your repository

### Learn More

- [dbt Documentation](https://docs.getdbt.com/)
- [dbt Best Practices](https://docs.getdbt.com/guides/best-practices)
- [dbt Discourse Community](https://discourse.getdbt.com/)

---

## Quick Reference Commands

### Docker Orchestrators

```bash
# Start
docker-compose -f docker-compose-[airflow|dagster|prefect].yaml up -d

# View logs
docker-compose -f docker-compose-[tool].yaml logs -f

# Stop
docker-compose -f docker-compose-[tool].yaml down

# Restart
docker-compose -f docker-compose-[tool].yaml restart
```

### Local dbt

```bash
# Core commands
dbt deps --profiles-dir .
dbt seed --profiles-dir .
dbt run --profiles-dir .
dbt test --profiles-dir .
dbt build --profiles-dir .

# Documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .

# Selective execution
dbt run --select stg_customers --profiles-dir .
dbt run --select marts --profiles-dir .
dbt test --select test_type:singular --profiles-dir .
```

### Database Access

```bash
# Connect to PostgreSQL
docker exec -it <container> psql -U airflow -d airflow

# Useful queries
\dn                    # List schemas
\dt staging.*          # List staging tables
\dt marts.*            # List marts tables
SELECT * FROM marts.rpt_sales_summary;
```

---

**Ready to explore?** Start with one of the orchestrators and then dive into the [Testing Guide](TESTING_GUIDE.md) or [Data Model documentation](DATA_MODEL.md)!
