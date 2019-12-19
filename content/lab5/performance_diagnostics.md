---
title: "Performance Diagnostics"
weight: 53
---

There are a few utilities that provide visibility into Redshift Spectrum:

* [EXPLAIN](http://docs.aws.amazon.com/redshift/latest/dg/r_EXPLAIN.html) - Provides the query execution plan, which includes info around what processing is pushed down to Spectrum. Steps in the plan that include the prefix S3 are executed on Spectrum; for instance, the plan for the query above has a step “S3 Seq Scan clickstream.uservisits_csv10” indicating that Spectrum performs a scan on S3 as part of the query execution.

* [SVL_S3QUERY_SUMMARY](http://docs.aws.amazon.com/redshift/latest/dg/r_SVL_S3QUERY_SUMMARY.html) - Provides statistics for Redshift Spectrum queries are stored in this table. While the execution plan presents cost estimates, this table stores actual statistics of past query runs.  

* [SVL_S3PARTITION](https://docs.aws.amazon.com/redshift/latest/dg/r_SVL_S3PARTITION.html) - Provides details about Amazon Redshift Spectrum partition pruning at the segment and node slice level.

1. Run the following query on the SVL_S3QUERY_SUMMARY table:
```sql
select elapsed, s3_scanned_rows, s3_scanned_bytes, 
  s3query_returned_rows, s3query_returned_bytes, files, avg_request_parallelism 
from svl_s3query_summary 
where query = pg_last_query_id() 
order by query,segment;
```

The query should return results similar to the following:

|elapsed|s3_scanned_rows|s3_scanned_bytes|s3query_returned_rows|s3query_returned_bytes|files|avg_request_parallelism|
|---|---|---|---|---|---|---|
|209773697|3758774345|6.61358E+11|66270117|1060321872|5040|9.77|

The diagnostics reveal why our query took so long. For instance, *s3_scanned_row* reveals that the query scans nearly 3.8B records, which is the entire data set. 

Using below query you can see Spectrum scans all partitions.
```sql
SELECT query, segment,
       MIN(starttime) AS starttime,
       MAX(endtime) AS endtime,
       datediff(ms,MIN(starttime),MAX(endtime)) AS dur_ms,
       MAX(total_partitions) AS total_partitions,
       MAX(qualified_partitions) AS qualified_partitions,
       MAX(assignment) as assignment_type
FROM svl_s3partition
WHERE query=pg_last_query_id()
GROUP BY query, segment;
```

2. Run the same Redshift Spectrum query again, but with EXPLAIN
```sql
EXPLAIN 
SELECT c.c_name, c.c_mktsegment, t.prettyMonthYear, SUM(uv.adRevenue)
FROM clickstream.uservisits_csv10 as uv
RIGHT OUTER JOIN customer as c ON c.c_custkey = uv.custKey
INNER JOIN (
  SELECT DISTINCT d_yearmonthnum, (d_month||','||d_year) as prettyMonthYear 
  FROM dwdate WHERE d_yearmonthnum >= 199810) as t ON uv.yearMonthKey = t.d_yearmonthnum
WHERE c.c_custkey <= 3
GROUP BY c.c_name, c.c_mktsegment, t.prettyMonthYear, uv.yearMonthKey 
ORDER BY c.c_name, c.c_mktsegment, uv.yearMonthKey  ASC
```

The output will look similar to the example below. Don’t worry about understanding the details of the query plan at this time.  

```
QUERY PLAN
XN Merge  (cost=1141056800867.31..1141056801917.31 rows=420000 width=78)
Merge Key: c.c_name, c.c_mktsegment, uv.yearmonthkey
->  XN Network  (cost=1141056800867.31..1141056801917.31 rows=420000 width=78)
Send to leader
->  XN Sort  (cost=1141056800867.31..1141056801917.31 rows=420000 width=78)
Sort Key: c.c_name, c.c_mktsegment, uv.yearmonthkey
->  XN HashAggregate  (cost=141056760589.25..141056761639.25 rows=420000 width=78)
->  XN Hash Join DS_DIST_ALL_NONE  (cost=37.34..141056596142.86 rows=13155711 width=78)
Hash Cond: ("outer".yearmonthkey = "inner".d_yearmonthnum)
->  XN Hash Join DS_DIST_ALL_NONE  (cost=4.50..141051766084.98 rows=375877435 width=46)
Hash Cond: ("outer".custkey = "inner".c_custkey)
->  XN Partition Loop  (cost=0.00..94063327993.62 rows=3758774345000 width=16)
->  XN Seq Scan PartitionInfo of clickstream.uservisits_csv10 uv  (cost=0.00..10.00 rows=1000 width=0)
->  XN S3 Query Scan uv  (cost=0.00..93969358.62 rows=3758774345 width=16)
->  S3 Seq Scan clickstream.uservisits_csv10 uv location:"s3://redshift-spectrum-datastore-csv10" format:TEXT  (cost=0.00..56381615.17 rows=3758774345 width=16)
Filter: ((custkey <= 3) AND (yearmonthkey >= 199810))
->  XN Hash  (cost=3.75..3.75 rows=300 width=38)
->  XN Seq Scan on customer c  (cost=0.00..3.75 rows=300 width=38)
Filter: (c_custkey <= 3)
->  XN Hash  (cost=32.82..32.82 rows=7 width=36)
->  XN Subquery Scan t  (cost=0.00..32.82 rows=7 width=36)
->  XN Unique  (cost=0.00..32.75 rows=7 width=18)
->  XN Seq Scan on dwdate  (cost=0.00..32.43 rows=64 width=18)
Filter: (d_yearmonthnum >= 199810)
```

The takeaway is that the query plan reveals how Redshift Spectrum is leverage in the query. The linke below indicates that Redshift Spectrum is leveraged as part of the query execution to perform a scan. It also reveals that partitions were not used. We will explore this in more detail in the next part of the lab.
```
->  S3 Seq Scan clickstream.uservisits_csv10 uv location:"s3://redshift-spectrum-datastore-csv10" format:TEXT  (cost=0.00..56381615.17 rows=3758774345 width=16)
```

