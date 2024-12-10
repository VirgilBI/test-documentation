# Tables

[... existing table definitions ...]

# Key Business Definitions and Sample Queries

[... existing definitions and queries ...]

# Important Notes for Querying
1. Date Functions: Use EXTRACT(YEAR FROM date_column) and EXTRACT(MONTH FROM date_column) for year and month extraction to ensure compatibility across different SQL dialects.
2. String Concatenation: Use CONCAT() function or the || operator (depending on your SQL dialect) for string concatenation.
3. Coupon Payments: Unless specified otherwise, exclude payments made with coupons in spend calculations.
4. Date Ranges: When filtering for post-GA data, use order_date >= '2024-01-01' in your WHERE clause.
5. Orders Without Line Items: For orders that exist but have no associated line items, assume they're associated with line_item_id = 1. Use COALESCE(oli.line_item_id, 1) when joining with the order_line_items table. When we first launched, we only had one product (our standard plan) and we didn't need to track individual line items within an order. Therefore, when you see an order without any associated line items it's safe to assume that it's a standard plan order.
6. Revenue Calculation: When calculating revenue, use COALESCE(oli.price, p.amount) to account for orders with and without line items.
7. Data Types: Be aware of the data types in each column and use appropriate type casting when necessary.
8. Test Users: Exclude test users from all analyses unless explicitly told to include them. A test user is any user that has a billable_status of FALSE in the jaffle_shop.billable_entities table.
9. October Data Exclusion: Due to the creation of many test users in October, exclude all October data from revenue calculations unless explicitly specified otherwise.