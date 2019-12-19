---
title: "Join Strategies"
Weight : 36
---

Because or the distributed architecture of Redshift, in order to process data which is joined together, data may have to be broadcast from one node to another.  It’s important to analyze the explain plan on a query to identify which join strategies is being used and how to improve it.

8. Execute an EXPLAIN on the following query.  If you recall, both of these tables are distributed on the custkey.  This results in a join strategy of “Hash Join DS_DIST_NONE” and a relatively low overall “cost”.
```sql
EXPLAIN
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders_v2 o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

9. Execute an EXPLAIN on the following query.  If you recall, this version of the orders is distributed on the orderkey.  This results in a join strategy of “Hash Join DS_BCAST_INNER” and a relatively high overall “cost”.
```sql
EXPLAIN
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

10. Create a new version of the orders tables which is both distributed and sorted on the values as the customer table.  Execute an EXPLAIN and notice this results in a join strategy of “Merge Join DS_DIST_NONE” with the lowest cost of the three.
```sql
CREATE TABLE orders_v4
DISTKEY(o_custkey) SORTKEY (o_custkey) as
SELECT * FROM orders_v2;
```
```sql
EXPLAIN
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM Customer_v3 c
JOIN Orders_v4 o on c.c_custkey = o.o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

11. Execute an EXPLAIN plan on the following query which is missing the join condition.  This results in a join strategy of “XN Nested Loop DS_BCAST_INNER” and throws a warning about the cartesian product.  
```sql
EXPLAIN
SELECT * FROM region, nation;
```