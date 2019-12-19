---
title: "Redshift Spectrum Request Accelerator"
weight: 57
---

Redshift Spectrum Request Accelerator (SRA) is automatically and transparently enabled, significantly improving the performance of queries against data in Amazon S3.

1. Run the following query against data in year 1992. This query should run in under 10 seconds. Modify the filter and run it again for year 1993.
```sql
SELECT c.c_name, c.c_mktsegment, uv.visitYearMonth, uv.totalRevenue
FROM (
  SELECT customer, visitYearMonth, SUM(adRevenue) as totalRevenue
  FROM clickstream.uservisits_parquet1
  WHERE customer <= 3 and visitYearMonth between 199201 and 199212
  GROUP BY  customer, visitYearMonth) as uv
RIGHT OUTER JOIN customer as c ON c.c_custkey = uv.customer
ORDER BY c.c_name, c.c_mktsegment, uv.visitYearMonth ASC;
```
2. Now let's run the query against data in three years, from 1992 to 1994. This query will access about 3x more data but it takes less than 3x time. This is because SRA caches the Spectrum result for year 1992 and 1993. this query can reuse it instead of processing them again.

3. Inspect the SVL_S3QUERY_SUMMARY again by running the query:
```sql
select query, starttime, elapsed, s3_scanned_rows, s3_scanned_bytes,
  s3query_returned_rows, s3query_returned_bytes, files, avg_request_parallelism 
from svl_s3query_summary 
order by query desc,segment;
```

You should observe the results below. Note that this query scans nearly 1.1B records.

|query|starttime|elapsed|s3_scanned_rows|s3_scanned_bytes|s3query_returned_rows|s3query_returned_bytes|files|avg_request_parallelism|
|---|---|---|---|---|---|---|---|---|
|207376|2019-11-06 21:16:54.594125|2303302|1102046642|8848259230|299|117806|144|5.2|
|207325|2019-11-06 21:12:53.917315|923104|265401542|2130891630|72|28368|36|5.3|
|207312|2019-11-06 21:12:22.648347|1616744|265426602|2131090671|72|28368|36|4.9|

This is a lot of data to process, yet the query runs in under 10 seconds. This is substantially faster than the queries we ran at the start of the lab, which queried the same dataset. The difference in performance is a result of the improvements we made to the data format, size and pushing the aggregation work down to the Spectrum layer.

