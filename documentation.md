# Tables

[Previous table information remains unchanged]

# Canonical Queries

[Previous canonical queries remain unchanged]

## Revenue by Growth Bucket and Month
```sql
WITH monthly_revenue AS (
  SELECT
    mth,
    customer_id,
    customer_name,
    SUM(net_mrr_calc) AS mrr
  FROM
    dbt_analytics.customer_recurring_revenue
  WHERE
    dt = month_end_date
  GROUP BY
    1, 2, 3
),

growth_buckets AS (
  SELECT
    current_month.mth,
    current_month.customer_id,
    current_month.customer_name,
    current_month.mrr AS current_mrr,
    prev_month.mrr AS previous_mrr,
    CASE
      WHEN prev_month.mrr IS NULL THEN 'New'
      WHEN current_month.mrr > prev_month.mrr THEN 'Expansion'
      WHEN current_month.mrr < prev_month.mrr THEN 'Contraction'
      WHEN current_month.mrr = prev_month.mrr THEN 'Flat'
      ELSE 'Churned'
    END AS growth_bucket
  FROM
    monthly_revenue current_month
  LEFT JOIN
    monthly_revenue prev_month
    ON current_month.customer_id = prev_month.customer_id
    AND current_month.mth = DATE_ADD(prev_month.mth, INTERVAL 1 MONTH)
)

SELECT
  mth,
  growth_bucket,
  SUM(current_mrr) AS total_mrr
FROM
  growth_buckets
GROUP BY
  1, 2
ORDER BY
  1, 2
```

This query calculates the revenue by growth bucket (New, Expansion, Contraction, Flat, Churned) and month. It uses the `dbt_analytics.customer_recurring_revenue` table to determine monthly revenue for each customer, then compares it with the previous month to categorize the growth.

# Important Notes for Querying
[Previous notes remain unchanged]

6. Growth Buckets: When analyzing revenue changes, use the following categories:
   - New: Customer didn't exist in the previous month
   - Expansion: Current month's MRR is higher than the previous month
   - Contraction: Current month's MRR is lower than the previous month
   - Flat: Current month's MRR is the same as the previous month
   - Churned: Customer existed in the previous month but not in the current month