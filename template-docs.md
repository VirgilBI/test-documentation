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
| property_demo_booked_by | TEXT | | The name of the SDR that sourced this deal
| property_hs_closed_amount | FLOAT64 | | Represents the amount of net-new MRR gained when this deal was closed
| property_hs_created_by_user_id | FLOAT64 | | |

## hubspot.deal_stage
| Column Name | Type | Joins To | Notes |
|-------------|------|----------|-------|
| deal_id | INT | hubspot.deal.deal_id | |
| date_entered | TIMESTAMP | | The date and time when the deal entered this stage |
| value | STRING | | Stage ID |



# Canonical Queries

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

## Won Deals
A deal is considered 'won' if the property_hs_closed_won_date is not NULL.
   
## Monthly Queries
When querying the customer_recurring_revenue table on a monthly basis, use the filter 'WHERE dt = month_end_date' to summing across daily snapshots.

## SQOs
A deal is considered a "Sales Qualified Opportunity" (SQO) if it has entered the deal_stage with value '145109412'.
