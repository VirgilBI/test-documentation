# Tables

[Previous table information remains unchanged]

# Canonical Queries

[Previous canonical queries remain unchanged]

## Revenue by Growth Bucket and Month (Detailed)
```sql
WITH date_spine AS (
  SELECT DATE_TRUNC(dt, MONTH) AS month
  FROM dbt_analytics.customer_recurring_revenue
  WHERE dt >= '2024-01-01'
  GROUP BY 1
),
customers AS (
  SELECT DISTINCT customer_id, customer_name
  FROM dbt_analytics.customer_recurring_revenue
),
customer_months AS (
  SELECT d.month, c.customer_id, c.customer_name
  FROM date_spine d
  CROSS JOIN customers c
),
monthly_mrr AS (
  SELECT
    DATE_TRUNC(dt, MONTH) AS month,
    customer_id,
    SUM(net_mrr_calc) AS mrr
  FROM dbt_analytics.customer_recurring_revenue
  WHERE dt = month_end_date
  GROUP BY 1, 2
),
mrr_with_previous AS (
  SELECT
    cm.month,
    cm.customer_id,
    cm.customer_name,
    IFNULL(m.mrr, 0) AS current_month_mrr,
    LAG(IFNULL(m.mrr, 0), 1) OVER (PARTITION BY cm.customer_id ORDER BY cm.month) AS previous_month_mrr
  FROM customer_months cm
  LEFT JOIN monthly_mrr m ON cm.month = m.month AND cm.customer_id = m.customer_id
),
growth_category_by_customer_and_month AS (
  SELECT
    month,
    customer_id,
    customer_name AS customer,
    current_month_mrr,
    previous_month_mrr,
    current_month_mrr - previous_month_mrr AS delta,
    LEAST(current_month_mrr, previous_month_mrr) AS recurring_revenue,
    CASE WHEN previous_month_mrr = 0 THEN current_month_mrr ELSE 0 END AS net_new,
    CASE WHEN current_month_mrr = 0 AND previous_month_mrr > 0 THEN previous_month_mrr ELSE 0 END AS churn,
    CASE WHEN current_month_mrr < previous_month_mrr AND current_month_mrr != 0 THEN previous_month_mrr - current_month_mrr ELSE 0 END AS contraction,
    CASE WHEN current_month_mrr > previous_month_mrr AND previous_month_mrr != 0 THEN current_month_mrr - previous_month_mrr ELSE 0 END AS expansion
  FROM mrr_with_previous
),
growth_by_month AS (
  SELECT 
    month,
    SUM(current_month_mrr) AS current_month_mrr,
    SUM(previous_month_mrr) AS previous_month_mrr,
    SUM(recurring_revenue) AS recurring_revenue,
    SUM(net_new) AS net_new,
    SUM(churn) AS churn,
    SUM(contraction) AS contraction,
    SUM(expansion) AS expansion
  FROM growth_category_by_customer_and_month
  GROUP BY month
)

SELECT * FROM growth_by_month
```

This query provides a detailed breakdown of revenue by growth bucket and month. It uses the `dbt_analytics.customer_recurring_revenue` table to calculate monthly recurring revenue (MRR) for each customer and then categorizes the revenue changes into different growth buckets: recurring revenue, net new, churn, contraction, and expansion.

Key points:
1. It creates a date spine starting from 2024-01-01.
2. It considers all customers, even if they don't have revenue in a particular month.
3. It calculates the current month's MRR and the previous month's MRR for each customer.
4. It categorizes the revenue changes into different buckets:
   - Recurring revenue: The lesser of current and previous month's MRR
   - Net new: When a customer had no revenue in the previous month but has revenue in the current month
   - Churn: When a customer had revenue in the previous month but has no revenue in the current month
   - Contraction: When the current month's revenue is less than the previous month's (but not zero)
   - Expansion: When the current month's revenue is more than the previous month's

This query provides a more granular view of revenue changes compared to the previous growth bucket query.

# Important Notes for Querying
[Previous notes remain unchanged]

7. Detailed Growth Analysis: When performing a detailed analysis of revenue changes, consider the following categories:
   - Recurring Revenue: The stable part of the revenue that continues from the previous month
   - Net New: Revenue from new customers who didn't exist in the previous month
   - Churn: Lost revenue from customers who existed in the previous month but not in the current month
   - Contraction: Decrease in revenue from existing customers
   - Expansion: Increase in revenue from existing customers