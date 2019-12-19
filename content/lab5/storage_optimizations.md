---
title: "Storage Optimizations"
weight: 55
---


Redshift Spectrum performs processing through large-scale infrastructure external to your Redshift cluster. It is optimized for performing large scans and aggregations on S3; in fact, with the proper optimizations, Redshift Spectrum may even out-perform a small to medium size Redshift cluster on these types of workloads. There are two important variables to consider for optimizing large scans and aggregations:
* **File size and count** - As a general rule, use files sizes between 50-500MB for non-splittable files, this is optimal for Redshift Spectrum. However, the number of files operating on a query is directly correlated with the parallelism achievable by a query. There is an inverse relationship between file size and count: the bigger the files, the fewer files there are for the same dataset. Consequently, there is a trade-off between optimizing for object read performance, and the amount of parallelism achievable on a particular query. Large files are best for large scans as the query likely operates on sufficiently large number of files. For queries that are more selective and for which fewer files are operating, you may find that smaller files allow for more parallelism.
* **Data format** Redshift Spectrum supports various data formats. Columnar formats like Parquet can sometimes lead to substantial performance benefits by providing compression and more efficient I/O for certain workloads. Generally, format types like Parquet should be used for query workloads involving large scans, and high attribute selectivity. Again, there are trade-offs as formats like Parquet require more compute power to process than plaintext. For queries on smaller subsets of data, the I/O efficiency benefit of Parquet is diminished. At some point, Parquet may perform the same or slower than plaintext. Latency, compression rates, and the trade-off between user experience and cost should drive your decision. In most cases, formats like Parquet is optimal.

To help illustrate how Spectrum performs on these large aggregation workloads, let’s consider a basic query that aggregates the entire 3.7B+ record dataset on Redshift Spectrum, and compared that with running the query exclusively on Redshift: 
```sql
SELECT uv.custKey, COUNT(uv.custKey)
FROM <your clickstream table> as uv
GROUP BY uv.custKey
ORDER BY uv.custKey ASC
```

In the interest of time, we won’t go through this exercise in the lab; nonetheless, it is helpful to understand the results of running this test.

For the Redshift-only test case, the clickstream data is loaded, and distributed evenly across all nodes (even distribution style) with optimal column compression encodings prescribed by Redshift’s ANALYZE command. 

The Redshift Spectrum test case utilizes a Parquet data format with one file containing all the data for a particular customer in a month; this results in files mostly in the range of 220-280MB, and in effect, is the largest file size for this partitioning scheme. If you run tests with the other datasets provided, you will see that this data format and size is optimal and will out-perform others by 60X+.

Take heed that the presented quantifications shouldn’t be applied generically as performance differences will vary depending on scenario. Instead take note of the testing strategy, the evidence, and the characteristics of the workload where Spectrum is likely to yield performance benefits.

The chart below compares the query execution time for the two scenarios. The results indicate that you will need to pay for 12 X DC1.Large nodes to get performance comparable to using Spectrum with the support of a small Redshift cluster in this particular scenario.  Also, note that the performance for Spectrum plateaus in the chart above. If the query involved aggregating data from more files, we would see a continued linear improvement in performance as well.


![Chart 1: Native Redshift vs. Spectrum Agg Performance by Node Count](/images/SpectrumAggCompare.png)
Chart 1: Native Redshift vs. Spectrum Agg Performance by Node Count