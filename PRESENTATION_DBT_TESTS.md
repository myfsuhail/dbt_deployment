# Presentation: dbt Testing Strategies
## Ensuring Data Quality in E-Commerce Analytics

---

## 1. Why Test in dbt?
*   **Prevent Regressions**: Ensure code changes don't break existing data logic.
*   **Build Trust**: Stakeholders rely on accurate reporting.
*   **Documentation**: Tests serve as live documentation of business assumptions.

---

## 2. Inbuilt Schema Tests (Generic)
Defined directly in YAML files (`schema.yml`).

*   **`unique`**: Ensures no duplicate values (e.g., `order_id`).
*   **`not_null`**: Ensures column always has data (e.g., `customer_id`).
*   **`accepted_values`**: Limits data to a specific set (e.g., `status` must be 'placed', 'shipped').
*   **`relationships`**: Referential integrity check (e.g., `order.customer_id` must exist in `customers.id`).

---

## 3. Custom Singular Tests
One-off SQL queries that return "failing" rows. Located in the `tests/` directory.

*   **`no_negative_quantities.sql`**: Ensures sales quantites are always > 0.
*   **`no_future_order_dates.sql`**: Prevents data entry errors where orders appear in the future.
*   **`order_amount_consistency.sql`**: Validates that order totals match the sum of items.

---

## 4. Advanced Testing with Packages
Using `dbt-utils` for complex logic.

*   **`equal_rowcount`**: Compare source vs target.
*   **`at_least_one`**: Ensure a column isn't entirely empty.

---

## 5. Testing in Production
*   **Automated execution**: Integrated into Airflow/Dagster/Prefect.
*   **Alerting**: Flows fail if tests don't pass, preventing "bad data" from reaching dashboards.
