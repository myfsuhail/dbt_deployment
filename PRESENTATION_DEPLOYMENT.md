# Presentation: dbt Deployment Strategies
## Scaling Orchestration in Modern Data Stacks

---

## 1. Orchestration Evolution
*   **Manual Scripts**: Error-prone, no visibility.
*   **Modern Orchestrators**: Automated, observable, and multi-tool aware.

---

## 2. The Showcase Architecture
This project demonstrates three different deployment patterns:

*   **Airflow (Cosmos)**: Task-per-model. High granular control. Best for complex dependency graphs.
*   **Airflow (API)**: Hybrid deployment pattern. Airflow handles orchestration while dbt Cloud handles the transformation lifecycle via API trigger.
*   **Dagster**: Software-Defined Assets. Focuses on "what" instead of "how". Excellent UI for data lineage.
*   **Prefect**: Python-first. Seamless integration with custom Python logic and non-SQL tasks.

---

## 3. Deployment Patterns
*   **CI/CD (GitHub Actions)**: Automated testing on every Pull Request.
*   **Containerization (Docker)**: Uniform environment from local dev to production.
*   **Postgres Warehouse**: Robust, shared storage for cross-tool consistency.

---

## 4. Hardening for Local Development
*   **Internal Path Isolation**: Solves Windows/Docker mount conflicts.
*   **Sequential Switching**: Predictable resource management.
*   **Named Volumes**: Persistent database state across container reboots.

---

## 5. Conclusion
A robust deployment isn't just about the tool; it's about:
*   **Reliability**: Using isolated paths.
*   **Observability**: Leveraging orchestrator UIs.
*   **Consistency**: A shared schema across the entire team.
