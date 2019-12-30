---
title: "Compressing and De-Normalizing"
weight: 32
---

### Standard layout
Redshift operates on high amounts of data. In order to optimize Redshift workloads, one of the key principles is to lower the amount of data stored. Diminishing this volume is achieved by using a set of compression algorithms. Instead of working on entire rows of data, containing values of different types and function, Redshift operates in a columnar fashion, this gives the opportunity to implement algorithms that can operate on single columns of data, thus greatly enhancing their efficiency. In this example we will load data into a table and test what compression scheme can be used.

Note: You can apply compression encodings to columns in tables automatically when using the COPY command into an empty table.  However, in this lab we will analyze and apply compression manually after data has been loaded to demonstrate the performance gains of using correct compression.

1. Create the customer table using the default settings, only specifying DISTKEY to the customer key.
```sql
CREATE TABLE customer_v1 (
  c_custkey int8 NOT NULL PRIMARY KEY                      ,
  c_name varchar(25) NOT NULL                              ,
  c_address varchar(40) NOT NULL                           ,
  c_nationkey int4 NOT NULL REFERENCES nation(n_nationkey) ,
  c_phone char(15) NOT NULL                                ,
  c_acctbal numeric(12,2) NOT NULL                         ,
  c_mktsegment char(10) NOT NULL                           ,
  c_comment varchar(117) NOT NULL
) DISTKEY (c_custkey);
```

2. Import data from the customer table and analyze statistics.
```sql
INSERT INTO customer_v1
SELECT * FROM customer;
```
```sql
ANALYZE customer_v1;
```

3. Analyze the storage optimization options for this table. You can choose the compression scheme you want or let Redshift determine one for you. This query analyzes the data in the table and presents recommendations on how to improve storage and performances for this table. The result of this statement is important and gives you insights on how to optimize storage based on the current real data stored in the table.
```sql
ANALYZE COMPRESSION customer_v1;
```

### Compression Optimization
Based on the results from the previous “ANALYZE COMPRESSION” command, we can insert the same data in a new table and analyze the difference in storage.

4. Create the new customer table using the hints provided by the last compression analysis.  
```sql
CREATE TABLE customer_v2 (
  c_custkey int8 NOT NULL ENCODE DELTA PRIMARY KEY   ,
  c_name varchar(25) NOT NULL ENCODE ZSTD            ,
  c_address varchar(40) NOT NULL ENCODE ZSTD         ,
  c_nationkey int4 NOT NULL ENCODE ZSTD              ,
  c_phone char(15) NOT NULL ENCODE ZSTD              ,
  c_acctbal numeric(12,2) NOT NULL ENCODE ZSTD       ,
  c_mktsegment char(10) NOT NULL ENCODE ZSTD         ,
  c_comment varchar(117) NOT NULL ENCODE ZSTD
) DISTKEY (c_custkey);
```

5. Import data from the previous table into this table and analyze statistics.
```sql
INSERT INTO customer_v2
SELECT * FROM customer_v1;
```
```sql
ANALYZE customer_v2;
```

6. Analyze the storage space for these tables, before and after compression. The table stores by column the amount of storage used in MiB. You should see about a 50% savings on the storage of the second table compared to first. This query gives you the storage requirements per column for each table, then the total storage for the table (repeated identically on each line).
```sql
SELECT
  CAST(d.attname AS CHAR(50)),
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v1%'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v1,
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v2%'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v2,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v1%'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v1,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) LIKE '%v2%'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v2
FROM (
  SELECT relname, attname, attnum - 1 as colid
  FROM pg_class t
  INNER JOIN pg_attribute a ON a.attrelid = t.oid
  WHERE t.relname LIKE 'customer\_v%') d
INNER JOIN (
  SELECT name, col, MAX(blocknum) AS size_in_mb
  FROM stv_blocklist b
  INNER JOIN stv_tbl_perm p ON b.tbl=p.id
  GROUP BY name, col) b
ON d.relname = b.name AND d.colid = b.col
GROUP BY d.attname
ORDER BY d.attname;
```

### Data De-Normalizing
Compression allows the storage of “reference” data inside the fact table, removing some of the needs for “star” or “snowflake” database designs for storage optimization. In this section of the lab we will de-normalize the nation and region information into the customer table, integrating these columns and analyzing the differences.

7. Create the new customer table, de-normalizing nation and region names to be included directly in the customer table.
```sql
CREATE TABLE customer_v3 (
  c_custkey int8 NOT NULL ENCODE DELTA PRIMARY KEY                ,
  c_name varchar(25) NOT NULL ENCODE ZSTD                         ,
  c_address varchar(40) NOT NULL ENCODE ZSTD                      ,
  c_nationname char(25) NOT NULL ENCODE ZSTD                      ,
  c_regionname char(25) NOT NULL ENCODE ZSTD                      ,
  c_phone char(15) NOT NULL ENCODE ZSTD                           ,
  c_acctbal numeric(12,2) NOT NULL ENCODE ZSTD                    ,
  c_mktsegment char(10) NOT NULL ENCODE ZSTD                      ,
  c_comment varchar(117) NOT NULL ENCODE ZSTD
) DISTKEY (c_custkey);
```

8. Import data from the previous table into this table. Note the joins to flatten the schema and build statistics.
```sql
INSERT INTO customer_v3(c_custkey, c_name, c_address, c_nationname, c_regionname, c_phone, c_acctbal, c_mktsegment, c_comment)
SELECT c_custkey, c_name, c_address, n_name, r_name, c_phone, c_acctbal, c_mktsegment, c_comment
FROM customer_v2
INNER JOIN nation ON c_nationkey = n_nationkey
INNER JOIN region ON n_regionkey = r_regionkey;
```
```sql
ANALYZE customer_v3;
```

9. Analyze the difference in storage space for these three versions of the customer table. Adding the columns just added the space required for this compressed column, and with the compression algorithm used, the difference is less than 2% the size of the table. This query gives you the storage requirements per column for each table, then the total storage for the table (repeated identically on each line).
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
  WHERE t.relname LIKE 'customer\_v%') d
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
While we won’t cover the details of Redshift queries, this section gives an example of a single query processed on all three tables. Data warehousing systems being designed for WORM (Write Once Read Many) type of workloads, the optimization of the table must be made knowing what the queries that will run on it will be. Redshift proposes system tables to analyze query performances, we will start using them.

10. Get customers from the “Asia” region from the first table.
```sql
SELECT COUNT(c_custkey)
FROM customer_v1 c
INNER JOIN nation n ON c.c_nationkey = n.n_nationkey
INNER JOIN region r ON n.n_regionkey = r.r_regionkey
WHERE r.r_name = 'ASIA';
```

11. From the second table.
```sql
SELECT COUNT(c_custkey)
FROM customer_v2 c
INNER JOIN nation n ON c.c_nationkey = n.n_nationkey
INNER JOIN region r ON n.n_regionkey = r.r_regionkey
WHERE r.r_name = 'ASIA';
```

12. And from the third table.
```sql
SELECT COUNT(c_custkey)
FROM customer_v3 c
WHERE c.c_regionname = 'ASIA';
```

13. Analyze the performances of each query. This query gets the 3 last queries ran against the database, and you should get your three last queries. The first executions may be comparably equal on the three queries, but if you repeat the execution multiple times (which is the case in a data warehousing environment), you will see that the query targeting the compressed table (second table) performs up to about 40% better than the one targeting the uncompressed table (first table), and that the query targeting the denormalized table (third table) has speed gains up to 50% compared to the one on the second table (or approximatively a 75% better than the one on the uncompressed table). (The numbers may vary depending on the cluster topology)
```sql
SELECT query, TRIM(querytxt) as SQL, starttime, endtime, DATEDIFF(microsecs, starttime, endtime) AS duration
FROM STL_QUERY
WHERE TRIM(querytxt) like '%customer%'
ORDER BY starttime DESC
LIMIT 3;
```