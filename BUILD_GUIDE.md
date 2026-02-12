# Build Guide: Architecture & Implementation

This guide explains the technical design of the dbt E-Commerce Analytics orchestration environment.

## 🏗️ Docker Architecture

The project uses a **Multi-Compose** strategy. Instead of one giant `docker-compose.yml`, we have individual files for each orchestrator:
*   `docker-compose-airflow.yaml`
*   `docker-compose-dagster.yaml`
*   `docker-compose-prefect.yaml`

### Shared Services
All orchestrators connect to a single **PostgreSQL 16** container.
*   **Database**: `airflow`
*   **Persistence**: `postgres-db-volume` (Named volume)

---

## 🛠️ Technical Fixes for Windows/Docker Stability

One of the major challenges in this project was handling dbt's transient file management (`dbt_packages/` and `target/`) on Windows host-mounts, which frequently causes "Device busy" or "Permission denied" errors.

### 1. Internal Path Isolation
Instead of mounting `dbt_packages/` to the host, we redirect them to an internal container-only path.
*   **Environment Variable**: `DBT_PACKAGES_PATH` is set to `/opt/.../dbt_packages_internal`.
*   **Effect**: dbt can perform rapid file operations (deleting/recreating folders) without Windows locking the files.

### 2. Fat Image Strategy (Airflow)
For Airflow, we use a custom Dockerfile that copies the project and pre-installs dependencies:
```dockerfile
COPY --chown=airflow:0 . /opt/airflow/dbt
RUN dbt deps
```
This ensures the container starts with a fully ready dbt environment, reducing startup time and initialization errors.

### 3. PostgreSQL Timezone Fix
To support legacy tools and specific client requirements, we symlink timezones in the Postgres container:
```dockerfile
RUN ln -sf /usr/share/zoneinfo/Asia/Kolkata /usr/share/zoneinfo/Asia/Calcutta
```

---

## 📈 Scaling Considerations
*   **Sequential Mode**: Currently optimized for local showcase using sequential context switching.
*   **Production Move**: For production, the "Internal Path" strategy should be replaced by a proper CI/CD process that packages the dbt artifacts into the image itself, rather than mounting any source code from the host.
