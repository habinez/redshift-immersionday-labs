---
title: "Selective Filtering"
weight: 35
---


## Selective Filtering
Redshift takes advantage of zone maps which allows the optimizer to skip reading blocks of data when it knows that the filter criteria will not be matched.   In the case of the orders_v3 table, because we have defined a sort key on the o_order_date, queries leveraging that field as a predicate will return much faster.

6. Execute the following two queries noting the execution time of each.  The first query is to ensure the plan is compiled.  The second has a slightly different filter condition to ensure the result cache cannot be used.
```sql
SELECT count(1), sum(o_totalprice)
FROM orders_v3
WHERE o_orderdate between '1992-07-05' and '1992-07-07';
```
```sql
SELECT count(1), sum(o_totalprice)
FROM orders_v3
WHERE o_orderdate between '1992-07-07' and '1992-07-09';
```
7. Execute the following two queries noting the execution time of each.  The first query is to ensure the plan is compiled.  The second has a slightly different filter condition to ensure the result cache cannot be used. You will notice the second query takes significantly longer than the second query in the previous step even though the number of rows which were aggregated is similar.  This is due to the first query's ability to take advantage of the Sort Key defined on the table.
```sql
SELECT count(1), sum(o_totalprice)
FROM orders_v3
where o_orderkey < 600001;
```
```sql
SELECT count(1), sum(o_totalprice)
FROM orders_v3
where o_orderkey < 600002;
```
