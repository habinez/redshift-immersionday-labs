---
title: "Result Set Caching and Execution Plan Reuse"
weight: 34
---

Redshift enables a result set cache to speed up retrieval of data when it knows that the data in the underlying table has not changed.  It can also re-use compiled query plans when only the predicate of the query has changed.

1. Execute the following query and note the query execution time.  Since this is the first execution of this query Redshift will need to compile the query as well as cache the result set.
```sql
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders_v2 o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

2. Execute the same query a second time and note the query execution time.  In the second execution redshift will leverage the result set cache and return immediately.
```sql
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders_v2 o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

3. Update data in the table and run the query again. When data in an underlying table has changed Redshift will be aware of the change and invalidate the result set cache associated to the query.  Note the execution time is not as fast as Step 2, but faster than Step 1 because while it couldn’t re-use the cache it could re-use the compiled plan.
```sql
UPDATE customer_v3
SET c_mktsegment = c_mktsegment
WHERE c_mktsegment = 'MACHINERY';
```
```sql
VACUUM DELETE ONLY customer_v3;
```
```sql
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders_v2 o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

4. Execute a new query with a predicate and note the query execution time.  Since this is the first execution of this query Redshift will need to compile the query as well as cache the result set.
```sql
SELECT c_mktsegment, count(1)
FROM Customer_v3 c
WHERE c_mktsegment = 'MACHINERY'
GROUP BY c_mktsegment;
```
5. Execute the query with a slightly different predicate and note that the execution time is faster than the prior execution even though a very similar amount of data was scanned and aggregated.  This behavior is due to the re-use of the compile cache because only the predicate has changed.  This type of pattern is typical for BI reporting where the SQL pattern remains consistent with different users retrieving data associated to different predicates.
```sql
SELECT c_mktsegment, count(1)
FROM Customer_v3 c
WHERE c_mktsegment = 'BUILDING'
GROUP BY c_mktsegment;
```


