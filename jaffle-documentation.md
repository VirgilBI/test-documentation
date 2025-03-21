# Tables

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



# Canonical Queries

## Orders by Product
SELECT 
  DATE_TRUNC('month', o.order_date) AS month,
  COUNT(COALESCE(oli.line_item_id, 1)) AS items_sold
FROM 
  jaffle_shop.orders o
LEFT JOIN 
  jaffle_shop.order_line_items oli ON o.id = oli.order_id
WHERE 
  o.status = 'completed' -- Assuming we only want completed orders
GROUP BY 
  1
ORDER BY 
  1;
  
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
)

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

## Coupon Payments
Unless specified otherwise, exclude payments made with coupons in spend calculations.

## Orders Without Line Items
For orders that exist but have no associated line items, assume they're associated with line_item_id = 1. Use COALESCE(oli.line_item_id, 1) when joining with the order_line_items table. When we first launched, we only had one product (our standard plan) and we didn't have a line_items table. Therefore, when you see an order without any associated line items it's safe to assume that it's a standard plan order.

## Line Items and Products
Each line item corresponds to a product. The line_item_id in the order_line_items table represents a specific product.

## Revenue Calculation
When calculating revenue, only include revenue from orders where the status is not 'returned' or 'return_pending'.
