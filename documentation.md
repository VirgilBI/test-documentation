# Tables

## hubspot.deal
| Column Name | Type | Joins To | Notes | 
|------------|------|----------|-------|
| property_dealname | TEXT | | | 
| deal_id | INT | hubspot.deal_company.deal_id | | 
| owner_id | INT | hubspot.owner.owner_id | | 
| property_lead_source | TEXT | | | 
| property_createdate | TIMESTAMP | | | 
| property_hs_closed_won_date | TIMESTAMP | | | 
| property_demo_booked_by_ | TEXT | | | The name of the SDR that sourced this deal
| property_hs_closed_amount | FLOAT64 | | Represents the amount of net-new MRR gained when this deal was closed

## hubspot.owner
| Column Name | Type | Joins To |
|------------|------|----------|
| owner_id | INT | hubspot.deal.owner_id |
| _fivetran_synced | TIMESTAMP | |
| active_user_id | INT | |
| created_at | TIMESTAMP | |
| email | TEXT | |
| first_name | TEXT | |
| is_active | BOOLEAN | |
| last_name | TEXT | |
| updated_at | TIMESTAMP | |

## dbt_analytics.customer_recurring_revenue
| Column Name | Type | Joins To |
|-------------|------|-------------|
| dt | DATE | |
| mth | DATE | |
| month_end_date | DATE | |
| customer_id | TEXT | hubspot.company.property_stripe_customer_id |
| customer_name | TEXT | |
| customer_domain | TEXT | |
| product_name | TEXT | |
| net_arr_calc | NUMERIC | |
| net_mrr_calc | NUMERIC | |

## hubspot.deal_company
| Column Name | Type | Joins To |
|-------------|------|----------|
| company_id | INT | |
| deal_id | INT | hubspot.deal.deal_id |
| _fivetran_synced | TIMESTAMP | |
| type_id | INT | |
| category | TEXT | |

## hubspot.company
| Column Name | Type | Joins To |
|-------------|------|----------|
| id | INT | |
| property_name | TEXT | |
| property_stripe_customer_id | TEXT | dbt_analytics.customer_recurring_revenue.customer_id |

## jaffle_shop.customers
| Column Name | Type | Joins To |
|------------|------|----------|
| id | INT | jaffle_shop.orders.user_id |
| first_name | TEXT | |
| last_name | TEXT | |
| created_at | DATE | |
| billing_id | INT | |

## jaffle_shop.orders
| Column Name | Type | Joins To |
|------------|------|----------|
| id | INT | jaffle_shop.payments.order_id, jaffle_shop.order_line_items.order_id |
| user_id | INT | jaffle_shop.customers.id |
| order_date | DATE | |
| status | TEXT | |

## jaffle_shop.payments
| Column Name | Type | Joins To |
|------------|------|----------|
| id | INT | |
| order_id | INT | jaffle_shop.orders.id |
| payment_method | TEXT | |
| amount | DECIMAL | |

## jaffle_shop.order_line_items
| Column Name | Type | Joins To |
|------------|------|----------|
| line_item_id | INT | |
| order_id | INT | jaffle_shop.orders.id |
| price | DECIMAL | |

## jaffle_shop.billable_entities
| Column Name | Type | Joins To |
|------------|------|----------|
| billing_id | INT | |
| billable_status | BOOL | |

#Canonical Queries
# Canonical Queries

## Net returns by SDR and Month
WITH ranked_deals AS (
  SELECT 
    dc.company_id,
    d.deal_id,
    d.property_createdate,
    ROW_NUMBER() OVER (PARTITION BY dc.company_id ORDER BY d.property_createdate ASC) AS deal_rank
  FROM 
    hubspot.deal_company dc
  JOIN 
    hubspot.deal d ON dc.deal_id = d.deal_id
),

first_deal_by_company as (
SELECT 
  company_id,
  deal_id AS first_deal_id
FROM 
  ranked_deals
WHERE 
  deal_rank = 1
ORDER BY 
  company_id), 
  
first_sdr_by_company as (
  select company_id, first_deal_id, property_demo_booked_by_, property_createdate as deal_created_date
  from first_deal_by_company 
  join hubspot.deal  on first_deal_id = deal.deal_id
  
  ),
  
  first_deal_by_customer as (
select fdbc.*, company.property_stripe_customer_id
from first_deal_by_company fdbc
join hubspot.company on fdbc.company_id = company.id
),

first_sdr_by_customer as (
select fsbc.*, company.property_stripe_customer_id
from first_sdr_by_company fsbc
join hubspot.company on fsbc.company_id = company.id

), 


mrr_by_customer_and_month as (
SELECT 
  crr.customer_id,
  crr.mth AS month,
  SUM(crr.net_mrr_calc) AS mrr
FROM 
  dbt_analytics.customer_recurring_revenue crr
WHERE 
  crr.dt = crr.month_end_date
GROUP BY 1, 2 
),


mrr_by_customer_and_month_with_sdr as (
select month, property_demo_booked_by_, sum(mrr) as total_mrr
from mrr_by_customer_and_month
left join first_sdr_by_customer on mrr_by_customer_and_month.customer_id = first_sdr_by_customer.property_stripe_customer_id
group by 1, 2
), 

 sdr_dates AS (
  SELECT 'Hugh McMackin' AS sdr_name, DATE '2024-02-05' AS start_date, NULL AS stop_date UNION ALL
  SELECT 'Alex Marshall', DATE '2024-02-15', '2025-01-01' UNION ALL
  SELECT 'Rob Jones', DATE '2024-02-26', DATE '2024-06-17' UNION ALL
  SELECT 'Ben Humphreys', DATE '2024-07-08', DATE '2025-02-24' UNION ALL
  SELECT 'Collin Buckley', DATE '2024-07-22', NULL UNION ALL
  SELECT 'Mary Kate Cusack', DATE '2024-10-21', DATE '2025-02-17' UNION ALL
  SELECT 'Bryan Gotti', DATE '2025-02-24', NULL UNION ALL
  SELECT 'Aidan Demian', DATE '2025-03-03', NULL UNION ALL
  SELECT 'Michael Dowers', DATE '2025-03-03', NULL UNION ALL
  SELECT 'Kyle Camposano', DATE '2025-03-03', NULL
),

months AS (
  SELECT DISTINCT DATE_TRUNC(DATE_ADD(DATE '2024-02-01', INTERVAL n MONTH), MONTH) AS month
  FROM UNNEST(GENERATE_ARRAY(0, TIMESTAMP_DIFF(CURRENT_DATE(), DATE '2024-02-01', MONTH))) AS n
),

expanded_sdrs AS (
  SELECT 
    m.month, 
    s.sdr_name, 
    6000 AS cost
  FROM sdr_dates s
  JOIN months m 
    ON m.month BETWEEN DATE_TRUNC(s.start_date, MONTH) 
                   AND COALESCE(DATE_TRUNC(s.stop_date, MONTH), CURRENT_DATE()) -- Handle NULL stop dates up to current month
),

sdr_costs as (
SELECT * 
FROM expanded_sdrs
ORDER BY month, sdr_name), 

results as (



select coalesce(property_demo_booked_by_, sdr_name) as sdr_name, coalesce(mrr_by_customer_and_month_with_sdr.month, sdr_costs.month) as month, coalesce(total_mrr, 0)  - coalesce(cost, 0) as net_return 
from mrr_by_customer_and_month_with_sdr
full join sdr_costs on (property_demo_booked_by_ = sdr_name and  sdr_costs.month = mrr_by_customer_and_month_with_sdr.month)
where sdr_name IS NOT NULL
and property_demo_booked_by_ != ''
order by coalesce(mrr_by_customer_and_month_with_sdr.month, sdr_costs.month) desc)

select * from results
where month < date_trunc(CURRENT_DATE, month)

## Net New MRR by SDR and Month
SELECT
  FORMAT_TIMESTAMP('%Y-%m', property_hs_closed_won_date) AS month,
  property_demo_booked_by_ AS SDR,
  SUM(property_hs_closed_amount) AS total_closed_amount
FROM
  hubspot.deal
WHERE
  property_hs_closed_won_date IS NOT NULL
  and property_demo_booked_by_ IS NOT NULL
  and property_demo_booked_by_ != ''
GROUP BY
  1, 2
ORDER BY
  1, 2

# Important Notes for Querying
1. Coupon Payments: Unless specified otherwise, exclude payments made with coupons in spend calculations.
2. Orders Without Line Items: For orders that exist but have no associated line items, assume they're associated with line_item_id = 1. Use COALESCE(oli.line_item_id, 1) when joining with the order_line_items table. When we first launched, we only had one product (our standard plan) and we didn't have a line_items table. Therefore, when you see an order without any associated line items it's safe to assume that it's a standard plan order.
3. Line Items and Products: Each line item corresponds to a product. The line_item_id in the order_line_items table represents a specific product.
4. Won Deals: A deal is considered 'won' if the property_hs_closed_won_date is not NULL.
5. Identifying Usage or Overage Revenue: When querying the customer_recurring_revenue table, you can identify "usage" or "overage" revenue by using the following filter:
   ```sql
   WHERE lower(product_name) LIKE '%bot%'
      OR lower(product_name) LIKE '%sms%'
      OR lower(product_name) LIKE '%ai conversations%'
      OR lower(product_name) LIKE '%mms%'
      OR lower(product_name) LIKE '%usage%'
   ```

   Only exclude this revenue if explicitly told to do so
6. Monthly Queries: When querying the customer_recurring_revenue table on a monthly basis, use the filter 'WHERE dt = month_end_date' to summing across daily snapshots.

