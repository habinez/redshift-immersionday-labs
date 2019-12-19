---
title: "Predicate Pushdown"
weight: 56
---

In the last section, we learned that Spectrum excels at performing large aggregations. In this section, we’ll experiment the results of pushing more work down to Redshift Spectrum.

1. Run the following query. After running this query a few times, you should observe execution times in the range of 4 seconds.
```sql
SELECT c.c_name, c.c_mktsegment, t.prettyMonthYear, uv.totalRevenue
FROM (
  SELECT customer, visitYearMonth, SUM(adRevenue) as totalRevenue 
  FROM clickstream.uservisits_parquet1
  WHERE customer <= 3 and visitYearMonth >= 199810
  GROUP BY  customer, visitYearMonth) as uv
RIGHT OUTER JOIN customer as c ON c.c_custkey = uv.customer
INNER JOIN (
  SELECT DISTINCT d_yearmonthnum, (d_month||','||d_year) as prettyMonthYear 
  FROM dwdate WHERE d_yearmonthnum >= 199810) as t ON uv.visitYearMonth = t.d_yearmonthnum
ORDER BY c.c_name, c.c_mktsegment, uv.visitYearMonth ASC;
```

This query improves on our previous one in a couple of ways. 

* We are querying the clickstream.uservisits_parquet1 table instead of clickstream.uservisits_csv10. These two tables contain the same data set, but they have been processed in different ways. The table clickstream.uservisits_parquet1 contains data in parquet format. Parquet is a columnar format, and yields I/O benefits for analytical workloads by providing compression and efficient retrieval of the attributes that are selected by the queries. Furthermore, the “1” vs “10” suffix indicates that all the data for each partition is stored in a single file instead of ten files like the CSV data set. The latter case has less overhead involved in processing large scans and aggregations.

* The aggregation work has been pushed down to Redshift Spectrum. When we analyzed the query plan previously, we observed that Spectrum is used for scanning. When you analyze the above query, you will see that aggregations are also performed at the Spectrum layer.

2. Re-run the SVL_S3QUERY_SUMMARY query:
```sql
select elapsed, s3_scanned_rows, s3_scanned_bytes, 
  s3query_returned_rows, s3query_returned_bytes, files, avg_request_parallelism 
from svl_s3query_summary 
where query = pg_last_query_id() 
order by query,segment;
```

You obtain the following results:

|elapsed|s3_scanned_rows|s3_scanned_bytes|s3query_returned_rows|s3query_returned_bytes|files|avg_request_parallelism|
|---|---|---|---|---|---|---|
|1990509|66270117|531159030|9|72|9|0.88|

The statistics reveal the source of some of the performance improvements:

* The bytes scanned is reduced even though the same number of rows are scanned as a result of compression.
* The number of rows returned is reduced to 9 from ~66.3M. This results in only 72 bytes returned from the Spectrum layer versus 795MBs. This is the result of pushing the aggregation down to the Spectrum layer. Our data is stored at the day-level granularity, and our query rolls that up to the month-level. By pushing the aggregation down to the Spectrum fleet, we only need to return 9 records that aggregate ad revenue up to the month level so that they can be joined with the required dimension attributes.

3. Run the query again with EXPLAIN:

The query plan should include an *S3 Aggregate* step, which indicates that the Spectrum layer offloads the aggregation processing for this query.

```
QUERY PLAN
XN Merge  (cost=1000094008880.16..1000094008880.18 rows=7 width=78)
  Merge Key: c.c_name, c.c_mktsegment, uv.visityearmonth
  ->  XN Network  (cost=1000094008880.16..1000094008880.18 rows=7 width=78)
        Send to leader
        ->  XN Sort  (cost=1000094008880.16..1000094008880.18 rows=7 width=78)
              Sort Key: c.c_name, c.c_mktsegment, uv.visityearmonth
              ->  XN Hash Join DS_DIST_ALL_NONE  (cost=94008878.97..94008880.06 rows=7 width=78)
                    Hash Cond: ("outer".customer = "inner".c_custkey)
                    ->  XN Hash Join DS_DIST_ALL_NONE  (cost=93971378.97..93971379.61 rows=7 width=48)
                          Hash Cond: ("outer".visityearmonth = "inner".d_yearmonthnum)
                          ->  XN Subquery Scan uv  (cost=93971346.13..93971346.42 rows=23 width=16)
                                ->  XN HashAggregate  (cost=93971346.13..93971346.19 rows=23 width=16)
                                      ->  XN Partition Loop  (cost=93969358.63..93970506.13 rows=112000 width=16)
                                            ->  XN Seq Scan PartitionInfo of clickstream.uservisits_parquet1  (cost=0.00..17.50 rows=112 width=8)
                                                  Filter: ((customer <= 3) AND (visityearmonth >= 199810) AND (visityearmonth >= 199810))
                                            ->  XN S3 Query Scan uservisits_parquet1  (cost=46984679.32..46984689.32 rows=1000 width=8)
                                                  ->  S3 Aggregate  (cost=46984679.32..46984679.32 rows=1000 width=8)
                                                        ->  S3 Seq Scan clickstream.uservisits_parquet1 location:"s3://redshift-spectrum-datastore-parquet1" format:PARQUET  (cost=0.00..37587743.45 rows=3758774345 width=8)
                          ->  XN Hash  (cost=32.82..32.82 rows=7 width=36)
                                ->  XN Subquery Scan t  (cost=0.00..32.82 rows=7 width=36)
                                      ->  XN Unique  (cost=0.00..32.75 rows=7 width=18)
                                            ->  XN Seq Scan on dwdate  (cost=0.00..32.43 rows=64 width=18)
                                                  Filter: (d_yearmonthnum >= 199810)
                    ->  XN Hash  (cost=30000.00..30000.00 rows=3000000 width=38)
                          ->  XN Seq Scan on customer c  (cost=0.00..30000.00 rows=3000000 width=38)
```