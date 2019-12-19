---
title: "Distributing and Sorting"
weight : 33
---

 
## Distributing and Sorting
### Distributing Data
One of the key features enabling Redshift’s scale is the possibility to slice the data dynamically across nodes, this can be done evenly or in a round-robin fashion (this is done by default), by a distribution key (or column) or an all distribution which puts all data in all slices.  These options give you the ability to spread out data on a cluster in distribution that can maximize the parallelization potential of the queries.
To help queries run fast, it is recommended to use as a distribution that will be used in regularly joined tables, allowing Redshift to co-locate the data of these different entities, reducing IO and network exchanges.   We will explore different distribution methods and what it does to the data and query performance.
Redshift also uses a specific Sort Column to know in advance what values of a column are in a given block, and to skip reading that entire block if the values it contains don’t fall into the range of a query.
In this sample, queries are based on customer related information (region), making the customer key a good fit for distribution key, and the filters are made on order date ranges, so using it as a sort key helps execution.

1. Create the orders table with default settings, this time changing DISTKEY to customer key and SORTKEY to order date.  
```sql
CREATE TABLE orders_v1 (
  o_orderkey int8 NOT NULL PRIMARY KEY                             ,
  o_custkey int8 NOT NULL                                          ,
  o_orderstatus char(1) NOT NULL                                   ,
  o_totalprice numeric(12,2) NOT NULL                              ,
  o_orderdate date NOT NULL                                        ,
  o_orderpriority char(15) NOT NULL                                ,
  o_clerk char(15) NOT NULL                                        ,
  o_shippriority int4 NOT NULL                                     ,
  o_comment varchar(79) NOT NULL
) DISTKEY (o_custkey) SORTKEY(o_orderdate);
```

2. Import data from the existing table into this table, clean up storage, and build statistics.
```sql
INSERT INTO orders_v1
SELECT * FROM orders;
```

```sql
ANALYZE orders_v1;
```

3. Analyze compression options. You will see that they have changed from the previous entries. Compression depends directly on the data as it is stored on disk, and storage is modified by distribution and sort options.
```sql
ANALYZE COMPRESSION orders_v1;
```

### All Together
This last step will use the new distribution and sort keys, and the compression settings proposed by Redshift.

4. Create the orders table using the recommended compression propositions, keeping DISTKEY to customer key and SORTKEY to order date.  Note: Encoding has not been added to the order date field as per the best practices because the order date is used as a sort key.
```sql
CREATE TABLE orders_v2 (
  o_orderkey int8 NOT NULL PRIMARY KEY ENCODE ZSTD                 ,
  o_custkey int8 NOT NULL ENCODE ZSTD								               ,
  o_orderstatus char(1) NOT NULL ENCODE ZSTD                       ,
  o_totalprice numeric(12,2) NOT NULL ENCODE ZSTD                  ,
  o_orderdate date NOT NULL                                        ,
  o_orderpriority char(15) NOT NULL ENCODE ZSTD                    ,
  o_clerk char(15) NOT NULL ENCODE ZSTD                            ,
  o_shippriority int4 NOT NULL ENCODE ZSTD                         ,
  o_comment varchar(79) NOT NULL ENCODE ZSTD
) DISTKEY (o_custkey) SORTKEY(o_orderdate);
```

5. Import data and build statistics.
```sql
INSERT INTO orders_v2
SELECT * FROM orders_v1;
```
```sql
ANALYZE orders_v2;
```

6. Finally, let do one more version using ALL distribution type to put a copy of all the data in every slice of the cluster creating the largest data foot print but putting this data as close to all other data as possible.
```sql
CREATE TABLE orders_v3 (
  o_orderkey int8 NOT NULL PRIMARY KEY ENCODE ZSTD                     ,
  o_custkey int8 NOT NULL REFERENCES customer_v3(c_custkey) ENCODE ZSTD,
  o_orderstatus char(1) NOT NULL ENCODE ZSTD                           ,
  o_totalprice numeric(12,2) NOT NULL ENCODE ZSTD                      ,
  o_orderdate date NOT NULL SORTKEY                                    ,
  o_orderpriority char(15) NOT NULL ENCODE ZSTD                        ,
  o_clerk char(15) NOT NULL ENCODE ZSTD                                ,
  o_shippriority int4 NOT NULL ENCODE ZSTD                             ,
  o_comment varchar(79) NOT NULL ENCODE ZSTD
) DISTSTYLE ALL;
```

7. Import data and build statistics.
```sql
INSERT INTO orders_v3
SELECT * FROM orders_v2;
```
```sql
ANALYZE orders_v3;
```

### Storage Analysis
As for the customers, this query will analyze the storage used by the four representations of the orders table.

8. Analyze the difference in storage space for these 3 versions of the order table. Compression allows a 50% to 60% storage reduction on the data. The third version is the largest amount of data as it stores all data in this table in all slices of the cluster.   If you wish to learn more detailed information about distributing data in a redshift cluster please see the following:
http://docs.aws.amazon.com/redshift/latest/dg/t_Distributing_data.html.
This query gives you the storage requirements per column for each table, then the total storage for the table (repeated identically on each line).
```sql
SELECT
  CAST(d.attname AS CHAR(50)),
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v1%'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v1,
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v2%'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v2,
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v3%'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v3,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v1%'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v1,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v2%'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v2,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v3%'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v3
FROM (
  SELECT relname, attname, attnum - 1 as colid
  FROM pg_class t
  INNER JOIN pg_attribute a ON a.attrelid = t.oid
  WHERE t.relname LIKE 'orders\_v%') d
INNER JOIN (
  SELECT name, col, MAX(blocknum) AS size_in_mb
  FROM stv_blocklist b
  INNER JOIN stv_tbl_perm p ON b.tbl=p.id
  GROUP BY name, col) b
ON d.relname = b.name AND d.colid = b.col
GROUP BY d.attname
ORDER BY d.attname;
```
### Queries
The query execution speed is also impacted by the distribution settings. This last part will issue the same query on the four versions of the order table, and analyze the time taken to execute these queries.

9. Get, for the year 1995, some information on the orders passed by the customers depending on their market segment, in Asia. This query is for the first table.
```sql
SELECT c_mktsegment, COUNT(o_orderkey) AS orders_count,
AVG(o_totalprice) AS medium_amount,
SUM(o_totalprice) AS orders_revenue
FROM orders_v1 o
INNER JOIN customer_v3 c ON o.o_custkey = c.c_custkey
WHERE o_orderdate BETWEEN '1995-01-01' AND '1995-12-31' AND
c_regionname = 'ASIA'
GROUP BY c_mktsegment;
```

10. Same query for the second table.
```sql
SELECT c_mktsegment, COUNT(o_orderkey) AS orders_count,
AVG(o_totalprice) AS medium_amount,
SUM(o_totalprice) AS orders_revenue
FROM orders_v2 o
INNER JOIN customer_v3 c ON o.o_custkey = c.c_custkey
WHERE o_orderdate BETWEEN '1995-01-01' AND '1995-12-31' AND
c_regionname = 'ASIA'
GROUP BY c_mktsegment;
```

11. For the third table. You will notice that the order of results has changed. This is due to the change in sorting and distribution, since we did not order the resultset (no ORDER clause), the “natural” (storage) order applies.
```sql
SELECT c_mktsegment, COUNT(o_orderkey) AS orders_count,
AVG(o_totalprice) AS medium_amount,
SUM(o_totalprice) AS orders_revenue
FROM orders_v3 o
INNER JOIN customer_v3 c ON o.o_custkey = c.c_custkey
WHERE o_orderdate BETWEEN '1995-01-01' AND '1995-12-31' AND
c_regionname = 'ASIA'
GROUP BY c_mktsegment;
```

12. Analyze the performances of each query. This query gets the 3 last queries ran against the database. The results go up to around 75% query time improvement with the right distribution, sort and compression schemes (orders_v1 vs orders_v2).  The ALL distribution (v3) should really be used only if a dimension table cannot be collocated with the fact table or other important joining tables, you can improve query performance significantly by distributing the entire table to all of the nodes. Using ALL distribution multiplies storage space requirements and increases load times and maintenance operations, so you should weigh all factors before choosing ALL distribution. (The numbers will vary depending on the cluster topology)

```sql
SELECT query, TRIM(querytxt) as SQL, starttime, endtime, DATEDIFF(microsecs, starttime, endtime) AS duration
FROM STL_QUERY
WHERE TRIM(querytxt) like '%orders_v%JOIN%'
ORDER BY starttime DESC
LIMIT 3;
```
