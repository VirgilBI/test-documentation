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

## salesloft.email
| Column Name | Type | Joins To |
|------------|------|----------|
| user_id | INT | |
| recipient_id | INT | |
| sent_at | TIMESTAMP | |

## salesloft.people
| Column Name | Type | Joins To |
|------------|------|----------|
| id | INT | |
| account_id | INT | |
| owner_id | INT | |
| email_address | TEXT | |

## salesloft.users
| Column Name | Type | Joins To |
|------------|------|----------|
| id | INT | |
| email | TEXT | |

## salesloft.account
| Column Name | Type | Joins To |
|------------|------|----------|
| id | INT | |
| crm_id | INT | hubspot.company.id |
| name | TEXT | |

# Important Notes for Querying
1. Coupon Payments: Unless specified otherwise, exclude payments made with coupons in spend calculations.
2. Orders Without Line Items: For orders that exist but have no associated line items, assume they're associated with line_item_id = 1. Use COALESCE(oli.line_item_id, 1) when joining with the order_line_items table. When we first launched, we only had one product (our standard plan) and we didn't have a line_items table. Therefore, when you see an order without any associated line items it's safe to assume that it's a standard plan order.
3. Line Items and Products: Each line item corresponds to a product. The line_item_id in the order_line_items table represents a specific product.