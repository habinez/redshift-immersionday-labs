---
title: "Table Maintainance - Analyze"
weight: 26
---

## Table Maintenance - Analyze
You should at regular intervals, update the statistical metadata that the query planner uses to build and choose optimal plans.  You can analyze a table explicitly by running the ANALYZE command.  When you load data with the COPY command, you can perform an analysis on incrementally loaded data automatically by setting the STATUPDATE option to ON.  When loading into an empty table, the COPY command by default performs the ANALYZE operation.

Run the ANALYZE command against the CUSTOMER table.

```sql
analyze customer;
```
To find out when ANALYZE commands were run, you can query system tables and view such as STL_QUERY and STV_STATEMENTTEXT and include a restriction on padb_fetch_sample.  For example, to find out when the CUSTOMER table was last analyzed, run this query:
```sql
select query, rtrim(querytxt), starttime
from stl_query
where
querytxt like 'padb_fetch_sample%' and
querytxt like '%customer%'
order by query desc;
```
Note: Time timestamp of the ANALYZE will correlate to when the COPY command was executed and there will be no entry for the second analyze statement.  Redshift knows that it does not need to run the ANALYZE operation as no data has changed in the table.
