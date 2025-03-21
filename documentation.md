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
| property_company_score_clay_ | NUMERIC | |
| property_crm_clay_ | TEXT | |

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
  SELECT 'Hugh McMackin' AS sdr_name, DATE '2024-02-05' AS start_date, NULL AS stop_date, 'hugh.mcmackin@usehatchapp.com' AS email UNION ALL
  SELECT 'Alex Marshall', DATE '2024-02-15', DATE '2025-01-01', 'alex.marshall@usehatchapp.com' UNION ALL
  SELECT 'Rob Jones', DATE '2024-02-26', DATE '2024-06-17', 'rob.jones@usehatchapp.com' UNION ALL
  SELECT 'Ben Humphreys', DATE '2024-07-08', DATE '2024-06-24', 'ben.humphreys@usehatchapp.com' UNION ALL
  SELECT 'Collin Buckley', DATE '2024-07-22', NULL, 'colin.buckley@usehatchapp.com' UNION ALL
  SELECT 'Mary Kate Cusack', DATE '2024-10-21', DATE '2025-02-17', 'marykate@usehatchapp.com' UNION ALL
  SELECT 'Bryan Gotti', DATE '2024-02-24', NULL, 'bryan.gotti@usehatchapp.com' UNION ALL
  SELECT 'Aidan Demian', DATE '2025-03-03', NULL, 'aidan.demian@usehatchapp.com' UNION ALL
  SELECT 'Michael Dowers', DATE '2025-03-03', NULL, 'michael.dowers@usehatchapp.com' UNION ALL
  SELECT 'Kyle Camposano', DATE '2025-03-03', NULL, 'kyle.camposano@usehatchapp.com'
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

## Company Funnel
WITH 
-- Get SDR emails from the sdrs table
sdr_emails AS (
  SELECT email
  FROM dbt_analytics.sdrs
),

-- Find the first email sent to each company by an SDR
first_email_by_company AS (
  SELECT 
    a.id AS account_id,
    a.crm_id,
    MIN(e.sent_at) AS date_first_sdr_emailed,
    FIRST_VALUE(u.email) OVER (
      PARTITION BY a.id 
      ORDER BY e.sent_at 
      ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS first_sdr_emailed_by
  FROM 
    salesloft.account a
  JOIN 
    salesloft.people p ON a.id = p.account_id
  JOIN 
    salesloft.email e ON p.id = e.recipient_id
  JOIN
    salesloft.users u ON e.user_id = u.id
  -- Only include SDR users
  JOIN 
    sdr_emails s ON u.email = s.email
  GROUP BY
    a.id, a.crm_id, u.email, e.sent_at
),

-- Get unique first email data per company
first_email_unique AS (
  SELECT 
    account_id,
    crm_id,
    MIN(date_first_sdr_emailed) AS date_first_sdr_emailed,
    -- Use a separate subquery to get the first emailer for each account
    ARRAY_AGG(first_sdr_emailed_by ORDER BY date_first_sdr_emailed ASC LIMIT 1)[OFFSET(0)] AS first_sdr_emailed_by
  FROM 
    first_email_by_company
  GROUP BY
    account_id, crm_id
),

-- Find deals created by SDRs with ranking to identify the first deal for each company
ranked_sdr_created_deals AS (
  SELECT 
    d.deal_id,
    d.property_dealname AS deal_name,
    dc.company_id,
    d.property_createdate,
    d.property_hs_closed_won_date,
    u.email AS creator_email,
    c.property_name AS company_name,
    ROW_NUMBER() OVER (PARTITION BY dc.company_id ORDER BY d.property_createdate ASC) AS deal_rank
  FROM 
    hubspot.deal d
  JOIN 
    hubspot.users u ON CAST(d.property_hs_created_by_user_id AS INT64) = u.id
  JOIN
    sdr_emails s ON u.email = s.email
  JOIN
    hubspot.deal_company dc ON d.deal_id = dc.deal_id
  JOIN
    hubspot.company c ON dc.company_id = c.id
),

-- Get only the first deals created by SDRs for each company
sdr_created_deals AS (
  SELECT 
    deal_id,
    deal_name,
    company_id,
    property_createdate,
    property_hs_closed_won_date,
    creator_email,
    company_name
  FROM 
    ranked_sdr_created_deals
  WHERE 
    deal_rank = 1
),

-- Find first deal date created by SDRs for each company
company_first_sdr_deal AS (
  SELECT 
    company_id,
    company_name,
    MIN(property_createdate) AS first_sdr_deal_date,
    ARRAY_AGG(deal_id ORDER BY property_createdate ASC LIMIT 1)[OFFSET(0)] AS first_sdr_deal_id,
    ARRAY_AGG(deal_name ORDER BY property_createdate ASC LIMIT 1)[OFFSET(0)] AS first_sdr_deal_name,
    ARRAY_AGG(creator_email ORDER BY property_createdate ASC LIMIT 1)[OFFSET(0)] AS first_sdr_deal_creator_email
  FROM 
    sdr_created_deals
  GROUP BY 
    company_id, company_name
),

-- Find first won opportunity created by SDRs for each company
company_first_sdr_won_oppt AS (
  SELECT 
    company_id,
    company_name,
    MIN(property_hs_closed_won_date) AS date_first_sdr_opportunity_won
  FROM 
    sdr_created_deals
  WHERE
    property_hs_closed_won_date IS NOT NULL
  GROUP BY 
    company_id, company_name
),

-- Find first SDR-created deal that became an SQO for each company
company_first_sdr_sqo AS (
  SELECT 
    scd.company_id,
    MIN(ds.date_entered) AS date_first_sdr_SQO
  FROM 
    sdr_created_deals scd
  JOIN 
    hubspot.deal_stage ds ON scd.deal_id = ds.deal_id
  WHERE 
    ds.value = '145109412' -- SQO stage value
  GROUP BY 
    scd.company_id
), 

results AS (
SELECT 
  cfd.company_id,
  cfd.company_name,
  fe.date_first_sdr_emailed,
  fe.first_sdr_emailed_by,
  cfd.first_sdr_deal_id,
  cfd.first_sdr_deal_name,
  cfd.first_sdr_deal_creator_email,
  cfd.first_sdr_deal_date,
  sqo.date_first_sdr_SQO, 
  wo.date_first_sdr_opportunity_won,
  c.property_company_score_clay_
FROM 
  company_first_sdr_deal cfd
LEFT JOIN 
  company_first_sdr_sqo sqo ON cfd.company_id = sqo.company_id
LEFT JOIN 
  company_first_sdr_won_oppt wo ON cfd.company_id = wo.company_id
LEFT JOIN 
  first_email_unique fe ON CAST(cfd.company_id AS STRING) = fe.crm_id
LEFT JOIN
  hubspot.company c ON cfd.company_id = c.id
WHERE 
  fe.date_first_sdr_emailed >= TIMESTAMP '2024-01-01'
ORDER BY 
  cfd.first_sdr_deal_date
)
  
SELECT * FROM results WHERE date_first_sdr_emailed > first_sdr_deal_date

# Important Notes for Querying

## Current AEs
When referring to "current AEs", this means owners with email addresses matthewv@usehatchapp.com, nicholas.wood@usehatchapp.com, and alex.marshall@usehatchapp.com.

## Won Deals
A deal is considered 'won' if the property_hs_closed_won_date is not NULL.

## Identifying Usage or Overage Revenue
When querying the customer_recurring_revenue table, you can identify "usage" or "overage" revenue by using the following filter:
   ```sql
   WHERE lower(product_name) LIKE '%bot%'
      OR lower(product_name) LIKE '%sms%'
      OR lower(product_name) LIKE '%ai conversations%'
      OR lower(product_name) LIKE '%mms%'
      OR lower(product_name) LIKE '%usage%'
   ```

   Only exclude this revenue if explicitly told to do so
   
## Monthly Queries
When querying the customer_recurring_revenue table on a monthly basis, use the filter 'WHERE dt = month_end_date' to summing across daily snapshots.

## SQOs
A deal is considered a "Sales Qualified Opportunity" (SQO) if it has entered the deal_stage with value '145109412'.

# Facts

## SDR Information
```yaml
type: team_members
category: Sales
members:
  - name: Hugh McMackin
    email: hugh.mcmackin@usehatchapp.com
    role: SDR
    start_date: 2024-02-05
    stop_date: null
  - name: Alex Marshall
    email: alex.marshall@usehatchapp.com
    role: SDR
    start_date: 2024-02-15
    stop_date: 2025-01-01
  - name: Rob Jones
    email: rob.jones@usehatchapp.com
    role: SDR
    start_date: 2024-02-26
    stop_date: 2024-06-17
  - name: Ben Humphreys
    email: ben.humphreys@usehatchapp.com
    role: SDR
    start_date: 2024-07-08
    stop_date: 2024-06-24
  - name: Collin Buckley
    email: colin.buckley@usehatchapp.com
    role: SDR
    start_date: 2024-07-22
    stop_date: null
  - name: Mary Kate Cusack
    email: marykate@usehatchapp.com
    role: SDR
    start_date: 2024-10-21
    stop_date: 2025-02-17
  - name: Bryan Gotti
    email: bryan.gotti@usehatchapp.com
    role: SDR
    start_date: 2025-02-24
    stop_date: null
  - name: Aidan Demian
    email: aidan.demian@usehatchapp.com
    role: SDR
    start_date: 2025-03-03
    stop_date: null
  - name: Kyle Camposano
    email: kyle.camposano@usehatchapp.com
    role: SDR
    start_date: 2025-03-03
    stop_date: null
  - name: Michael Dowers
    email: michael.dowers@usehatchapp.com
    role: SDR
    start_date: 2025-03-03
    stop_date: 2025-03-15
  - name: Jacob Kaplan
    email: jacob.kaplan@usehatchapp.com
    role: SDR
    start_date: 2025-03-01
    stop_date: null
