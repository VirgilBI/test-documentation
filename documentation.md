# Key Business Definitions and Sample Queries

## Total Spend Per Customer
### Definition: Calculates the total amount each customer has spent, excluding payments made with coupons.
### Sample Query:
```
sql
Copy
SELECT 
    c.id AS customer_id,
    c.first_name,
    c.last_name,
    SUM(p.amount) AS total_spent
FROM 
    jaffle_shop.customers c
JOIN 
    jaffle_shop.orders o ON c.id = o.user_id
JOIN 
    jaffle_shop.payments p ON o.id = p.order_id
WHERE 
    p.payment_method != 'coupon'
GROUP BY 
    c.id, c.first_name, c.last_name
ORDER BY 
    total_spent DESC;
```

## Monthly Active Users
### Definition: The number of users that make at least one purchase per month.
### Sample Query:
```
sql
SELECT
    EXTRACT(YEAR FROM o.order_date) AS year,
    EXTRACT(MONTH FROM o.order_date) AS month,
    COUNT(DISTINCT o.user_id) AS active_users
FROM
    jaffle_shop.orders o
JOIN
    jaffle_shop.payments p ON o.id = p.order_id
WHERE
    p.payment_method != 'coupon'
GROUP BY
    EXTRACT(YEAR FROM o.order_date),
    EXTRACT(MONTH FROM o.order_date)
ORDER BY
    year, month;
```

## Revenue by Line Item and Month
### Definition: The total revenue generated by each line item per month, including orders without specific line items.
### Sample Query:
```
sql
WITH order_revenue AS (
    SELECT
        o.id AS order_id,
        o.order_date,
        COALESCE(oli.line_item_id, 1) AS line_item_id,
        COALESCE(oli.cost, p.amount) AS revenue
    FROM
        jaffle_shop.orders o
    LEFT JOIN
        jaffle_shop.order_line_items oli ON o.id = oli.order_id
    JOIN
        jaffle_shop.payments p ON o.id = p.order_id
    WHERE
        p.payment_method != 'coupon'
),
monthly_revenue AS (
    SELECT
        EXTRACT(YEAR FROM order_date) AS year,
        EXTRACT(MONTH FROM order_date) AS month,
        line_item_id,
        SUM(revenue) AS revenue
    FROM
        order_revenue
    GROUP BY
        EXTRACT(YEAR FROM order_date),
        EXTRACT(MONTH FROM order_date),
        line_item_id
)
SELECT
    CONCAT(
        CAST(year AS VARCHAR),
        '-',
        LPAD(CAST(month AS VARCHAR), 2, '0')
    ) AS month,
    line_item_id,
    revenue
FROM
    monthly_revenue
ORDER BY
    year, month, line_item_id;
```

## GA (General Availability)
### Definition: The date we went live with our product. 1/1/2024.

# Important Notes for Querying
1. Date Functions: Use EXTRACT(YEAR FROM date_column) and EXTRACT(MONTH FROM date_column) for year and month extraction to ensure compatibility across different SQL dialects.
1. String Concatenation: Use CONCAT() function or the || operator (depending on your SQL dialect) for string concatenation.
1. Test Users: Always exclude test users in your queries unless specifically told to include them. Use a LEFT JOIN with the test_users table and filter where tu.user_id IS NULL.
1. Coupon Payments: Unless specified otherwise, exclude payments made with coupons in spend calculations.
1. Date Ranges: When filtering for post-GA data, use order_date >= '2024-01-01' in your WHERE clause.
1. Orders Without Line Items: For orders that exist but have no associated line items, assume they're associated with line_item_id = 1. Use COALESCE(oli.line_item_id, 1) when joining with the order_line_items table. When we first launched, we only had one product (our standard plan) and we didn’t need to track individual line items within an order. Therefore, when you see an order without any associated line items it’s safe to assume that it’s a standard plan order.
1. Revenue Calculation: When calculating revenue, use COALESCE(oli.cost, p.amount) to account for orders with and without line items.
1. Data Types: Be aware of the data types in each column and use appropriate type casting when necessary.