---
title: "Optimizing With Partitions"
weight: 54
---

In this section, you will learn about partitions, and how they can be used to improve the performance of your Redshift Spectrum queries.  Partitioning is a key means to improving scan efficiency. Previously, we ran the glue crawler which created our external tables along with partitions. Navigate back to the [Glue Catalog](https://console.aws.amazon.com/glue/home?#catalog:tab=tables).  Click on the *uservisits_csv10* table. The columns *customer* and *visityearmonth* are set as the partition keys.

![](/images/crawler_11_clickstream.png)

If you have interest in understanding the details of how partitions were setup, refer to the [documentation](http://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-external-tables.html).  You may explore the S3 buckets that are serving our Redshift Spectrum datasets by navigating to the following location: 
[s3://redshift-immersionday-labs/data/clickstream/uservisits_csv10/](https://s3.console.aws.amazon.com/s3/buckets/redshift-immersionday-labs/data/clickstream/uservisits_csv10/)

The entire 3.8 billion-row dataset is organized as a collection of large files where each file contains data exclusive to a particular customer and month in a year. This allows you to partition your data into logical subsets by customer and year/month as exemplified above. With partitions, the query engine can target a subset of files:
* Only for specific customers
* Only data for specific months
* A combination of specific customers and year/months

Take note that the right choice of partitions is dependent on your workload. Partitions should be selected based on the primary queries you want to optimize, and your data profile. For those implementing their own clickstream analytics, a partition scheme like year/month/region often makes sense. The choice of using customer in the partition scheme isn’t optimal for a use case where there is a very large number of customers, and little data for each one. The data set, and scheme used in this example is a practical one for scenarios like a multi-tenant ad-tech platform, or an IoT platform. In these cases, there are a moderate number of customers (tenants), and a lot of data per customer.

1. Observe the effects of leveraging partitioning on our query by running the following query.
```sql
SELECT c.c_name, c.c_mktsegment, t.prettyMonthYear, SUM(uv.adRevenue)
FROM clickstream.uservisits_csv10 as uv
RIGHT OUTER JOIN customer as c ON c.c_custkey = uv.customer
INNER JOIN 
  (SELECT DISTINCT d_yearmonthnum, (d_month||','||d_year) as prettyMonthYear 
   FROM dwdate 
   WHERE d_yearmonthnum >= 199810) as t ON uv.yearMonthKey = t.d_yearmonthnum
WHERE c.c_custkey <= 3
GROUP BY c.c_name, c.c_mktsegment, t.prettyMonthYear, uv.yearMonthKey 
ORDER BY c.c_name, c.c_mktsegment, uv.yearMonthKey  ASC
```

The join condition from the previous query has been modified. Instead of joining on the synthetic key, *custKey*, we use the partition key, *customer*, that we created as part of the data modeling process. This query should run approximately **2X faster** than the previous. 

2. Run the same Redshift Spectrum query again, but with EXPLAIN. Unlike before, you should see a Filter clause as part of the PartitionInfo scan that indicates partition pruning is executed as part of the query plan:

```
->  XN Seq Scan PartitionInfo of clickstream.uservisits_csv10 uv  (cost=0.00..12.50 rows=334 width=4)
         Filter: ((customer <= 3) AND (subplan 4: (customer = $2)))
```

3. Re-run the SVL_S3QUERY_SUMMARY query:
```sql
select elapsed, s3_scanned_rows, s3_scanned_bytes, 
  s3query_returned_rows, s3query_returned_bytes, files, avg_request_parallelism 
from svl_s3query_summary 
where query = pg_last_query_id() 
order by query,segment;
```

You should observe the following results:

|elapsed|s3_scanned_rows|s3_scanned_bytes|s3query_returned_rows|s3query_returned_bytes|files|avg_request_parallelism|
|---|---|---|---|---|---|---|
|113561847|1898739653|3.34084E+11|66270117|795241404|2520|9.71|


Note that *s3_scanned_rows* reveals that the rows scanned has been halved when compared with the previous query. This explains why our query ran roughly twice as fast.

The results are due to the fact that our data is evenly distributed across all customers, and by querying 3 of 6 customers with our customer partition key, the database engine is able to intelligently scan the subset of data containing customers 1,2 and 3 instead of the entire data set. However, the scan is still very inefficient, and we can benefit from utilizing our year/month partition key as well.

4. Run the query below:
```sql
SELECT c.c_name, c.c_mktsegment, t.prettyMonthYear, SUM(uv.adRevenue)
FROM clickstream.uservisits_csv10 as uv
RIGHT OUTER JOIN customer as c ON c.c_custkey = uv.customer
INNER JOIN (
  SELECT DISTINCT d_yearmonthnum, (d_month||','||d_year) as prettyMonthYear 
  FROM dwdate 
  WHERE d_yearmonthnum >= 199810) as t ON uv.visitYearMonth = t.d_yearmonthnum
WHERE c.c_custkey <= 3
GROUP BY c.c_name, c.c_mktsegment, t.prettyMonthYear, uv.yearMonthKey 
ORDER BY c.c_name, c.c_mktsegment, uv.yearMonthKey  ASC
```

The join condition from the previous query has been modified. Instead of joining on the synthetic key, *yearMonthKey*, we use the partition key, *visitYearMonth*. Our latest query utilizes both customer and time partitions and if you run this query a few times, you should see execution time in the range of 8s, which is a **22.5X improvement** on our original query! 

4. Re-run the SVL_S3QUERY_SUMMARY query:
```sql
select elapsed, s3_scanned_rows, s3_scanned_bytes, 
  s3query_returned_rows, s3query_returned_bytes, files, avg_request_parallelism 
from svl_s3query_summary 
where query = pg_last_query_id() 
order by query,segment;
```

Upon reviewing the statistics for this query, you should observe that Redshift Spectrum scans and returns the exact number of rows (66,270,117) required to compute the query.

|elapsed|s3_scanned_rows|s3_scanned_bytes|s3query_returned_rows|s3query_returned_bytes|files|avg_request_parallelism|
|---|---|---|---|---|---|---|
|7124877|66270117|11660676734|66270117|795241404|90|5.87
