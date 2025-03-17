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

## hubspot.deal
| Column Name | Type | Joins To | Notes | 
|------------|------|----------|-------|
| property_dealname | TEXT | | | 
| deal_id | INT | hubspot.deal_company.deal_id | | 
| owner_id | INT | hubspot.owner.owner_id | | 
| property_lead_source | TEXT | | | 
| property_createdate | TIMESTAMP | | | 
| property_hs_closed_won_date | TIMESTAMP | | | 
| property_demo_booked_by_ | TEXT | | The name of the SDR that sourced this deal
| property_hs_closed_amount | FLOAT64 | | Represents the amount of net-new MRR gained when this deal was closed
| property_hs_created_by_user_id | FLOAT64 | | |

## hubspot.deal_stage
| Column Name | Type | Joins To | Notes |
|-------------|------|----------|-------|
| deal_id | INT | hubspot.deal.deal_id | |
| date_entered | TIMESTAMP | | The date and time when the deal entered this stage |
| value | STRING | | Stage ID |

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

## hubspot.users
| Column Name | Type | Joins To |
|------------|------|----------|
| id | INT | |
| email | TEXT | |

# Canonical Queries
  
SELECT * FROM results WHERE date_first_sdr_emailed > first_sdr_deal_date

# Important Notes for Querying

## Coupon Payments
Unless specified otherwise, exclude payments made with coupons in spend calculations.

## Orders Without Line Items
For orders that exist but have no associated line items, assume they're associated with line_item_id = 1. Use COALESCE(oli.line_item_id, 1) when joining with the order_line_items table. When we first launched, we only had one product (our standard plan) and we didn't have a line_items table. Therefore, when you see an order without any associated line items it's safe to assume that it's a standard plan order.

## Line Items and Products
Each line item corresponds to a product. The line_item_id in the order_line_items table represents a specific product.

##Won Deals
A deal is considered 'won' if the property_hs_closed_won_date is not NULL.

# Facts

## SDR Information
```yaml
type: team_members
category: Sales
members:
  - name: Liza McBride
    email: liza.mcbride@jaffleshop.com
    role: SDR
    start_date: 2024-02-05
    stop_date: null
  - name: Sam Sander
    email: sam.sanders@jaffleshop.com
    role: AE
    start_date: 2024-02-15
    stop_date: 2025-01-01
  
