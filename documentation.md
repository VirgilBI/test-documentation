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

## hubspot.deal_stage
| Column Name | Type | Joins To | Notes |
|-------------|------|----------|-------|
| deal_id | INT | hubspot.deal.deal_id | |
| date_entered | TIMESTAMP | | The date and time when the deal entered this stage |
| value | INT | | Stage ID |

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

# Canonical Queries

## Net returns by SDR and Month
[Query remains unchanged]

## Net New MRR by SDR and Month
[Query remains unchanged]

# Important Notes for Querying
1. Current AEs: When referring to "current AEs", this means owners with email addresses matthewv@usehatchapp.com, nicholas.wood@usehatchapp.com, and alex.marshall@usehatchapp.com.
2. Won Deals: A deal is considered 'won' if the property_hs_closed_won_date is not NULL.
3. Sales Qualified Opportunity (SQO): A deal is considered an SQO if it has entered the deal_stage with value '145109412'.
4. Identifying Usage or Overage Revenue: When querying the customer_recurring_revenue table, you can identify "usage" or "overage" revenue by using the following filter:
   ```sql
   WHERE lower(product_name) LIKE '%bot%'
      OR lower(product_name) LIKE '%sms%'
      OR lower(product_name) LIKE '%ai conversations%'
      OR lower(product_name) LIKE '%mms%'
      OR lower(product_name) LIKE '%usage%'
   ```

   Only exclude this revenue if explicitly told to do so
5. Monthly Queries: When querying the customer_recurring_revenue table on a monthly basis, use the filter 'WHERE dt = month_end_date' to summing across daily snapshots.