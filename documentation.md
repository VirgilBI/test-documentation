# Additional Information
## Test Users Exclusion
To exclude test users from queries, we should join with the jaffle_shop.billable_entities table and filter where billable_status is TRUE. The billing_id in the customers table corresponds to the billing_id in the billable_entities table.

## Updated jaffle_shop.customers Table
| Column Name | Type | Joins To |
|------------|------|----------|
| id | INT | jaffle_shop.orders.user_id |
| first_name | TEXT | |
| last_name | TEXT | |
| created_at | DATE | |
| billing_id | INT | jaffle_shop.billable_entities.billing_id |

This information should be used to exclude test users (non-billable entities) from relevant queries.