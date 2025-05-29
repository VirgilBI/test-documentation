# Tables

## example_table
| Column Name | Type | Joins To | Notes |
|-------------|------|----------|-------|
| One Column | NUMERIC | | The Quick Brown Fox |
| Two Column | NUMERIC | | Jumped Over The Lazy Dog |
| Three Column | NUMERIC | | The Quick Brown Fox |

# Canonical Queries
SELECT * FROM example_table; 

# Important Notes for Querying

## SQL Dialect Notes
In SQL, any column that appears in the SELECT clause of a GROUP BY query must be: in the GROUP BY clause, or used inside an aggregate function, or allowed via a function that ignores grouping semantics, like ANY_VALUE() in some SQL dialects.

# Facts

## Data Tables
The example_table is filled with example data. It should never be used under any circumstances. 
