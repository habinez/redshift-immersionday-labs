---
title: "Create a Single Version of Truth"
weight: 44
---

## Create a Single Version of Truth
In the next part of this lab, we will demonstrate how to create a view which has data that is consolidated from S3 via Spectrum and the Redshift direct-attached storage.

### Create a view 
Create a view that covers both the January, 2016 Green company DAS table with the historical data residing on S3 to make a single table exclusively for the Green data scientists. Use CTAS to create a table with data from January, 2016 for the Green company. Compare the runtime to populate this with the COPY runtime earlier.

<details><summary>Hint</summary>
<p>

```sql
CREATE TABLE workshop_das.taxi_201601 AS 
SELECT * FROM adb305.ny_pub 
WHERE year = 2016 AND month = 1 AND type = 'green';
```

</p>
</details>

Note: What about column compression/encoding? Remember that on a CTAS, Amazon Redshift automatically assigns compression encoding as follows:

* Columns that are defined as sort keys are assigned RAW compression.
* Columns that are defined as BOOLEAN, REAL, or DOUBLE PRECISION data types are assigned RAW compression.
* All other columns are assigned LZO compression.

```javascript
https://docs.aws.amazon.com/redshift/latest/dg/r_CTAS_usage_notes.html 

```

```sql
ANALYZE COMPRESSION workshop_das.taxi_201601
```

Here's the output in case you want to use it:

|Column|Encoding|Est_reduction_pct|
|---|---|---|
|vendorid|zstd|79.46|
|pickup_datetime|zstd|33.91|
|dropoff_datetime|zstd|34.08|
|ratecode|zstd|61.75|
|passenger_count|zstd|61.23|
|trip_distance|zstd|73.34|
|fare_amount|bytedict|85.61|
|total_amount|zstd|75.28|
|payment_type|zstd|68.76|
|year|zstd|91.13|
|month|zstd|91.13|
|type|zstd|89.23|


### Complete populating the table 
Add to the January, 2016 table with an INSERT/SELECT statement for the other taxi companies.

<details><summary>Hint</summary>
<p>

```sql
INSERT INTO workshop_das.taxi_201601 (
  SELECT * 
  FROM adb305.ny_pub 
  WHERE year = 2016 AND month = 1 AND type != 'green');
```

</p>
</details>

### Remove overlaps in the Spectrum table 
Now that we've loaded all January, 2016 data, we can remove the partitions from the Spectrum table so there is no overlap between the direct-attached storage (DAS) table and the Spectrum table.

<details><summary>Hint</summary>
<p>

```sql
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='fhv'); 
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='green'); 
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='yellow'); 
```

</p>
</details>


### Create a view with no Schema Binding
Create a view **adb305_view_NYTaxiRides** from **workshop_das.taxi_201601** that allows seamless querying of the DAS and Spectrum data.

<details><summary>Hint</summary>
<p>

```sql
CREATE VIEW adb305_view_NYTaxiRides AS
  SELECT * FROM workshop_das.taxi_201601
  UNION ALL 
  SELECT * FROM adb305.ny_pub
WITH NO SCHEMA BINDING;

```

</p>
</details>

### Is it Surprising this is valid SQL?

* Note the use of the partition columns in the SELECT and WHERE clauses. Where were those columns in your Spectrum table definition?
* Note the filters being applied either at the partition or file levels in the Spectrum portion of the query (versus the Redshift DAS section).
* If you actually run the query (and not just generate the explain plan), does the runtime surprise you? Why or why not?

```sql
EXPLAIN 
SELECT year, month, type, COUNT(*) 
FROM adb305_view_NYTaxiRides 
WHERE year = 2016 AND month IN (1) AND passenger_count = 4 
GROUP BY 1,2,3 ORDER BY 1,2,3;
```

```sql
QUERY PLAN 
XN Merge  (cost=1000090025653.20..1000090025653.21 rows=2 width=48)
  Merge Key: derived_col1, derived_col2, derived_col3
  ->  XN Network  (cost=1000090025653.20..1000090025653.21 rows=2 width=48)
        Send to leader
        ->  XN Sort  (cost=1000090025653.20..1000090025653.21 rows=2 width=48)
              Sort Key: derived_col1, derived_col2, derived_col3
              ->  XN HashAggregate  (cost=90025653.19..90025653.19 rows=2 width=48)
                    ->  XN Subquery Scan adb305_view_nytaxirides  (cost=25608.12..90025653.17 rows=2 width=48)
                          ->  XN Append  (cost=25608.12..90025653.15 rows=2 width=38)
                                ->  XN Subquery Scan "*SELECT* 1"  (cost=25608.12..25608.13 rows=1 width=18)
                                      ->  XN HashAggregate  (cost=25608.12..25608.12 rows=1 width=18)
                                            ->  XN Seq Scan on t201601_pqt  (cost=0.00..25292.49 rows=31563 width=18)
                                                  <b>Filter: ((passenger_count = 4) AND ("month" = 1) AND ("year" = 2016))</b>
                                ->  XN Subquery Scan "*SELECT* 2"  (cost=90000045.00..90000045.02 rows=1 width=38)
                                      ->  XN HashAggregate  (cost=90000045.00..90000045.01 rows=1 width=38)
                                            ->  XN Partition Loop  (cost=90000000.00..90000035.00 rows=1000 width=38)
                                                  ->  XN Seq Scan PartitionInfo of adb305.nytaxirides  (cost=0.00..15.00 rows=1 width=30)
                                                       <b> Filter: (("month" = 1) AND ("year" = 2016))</b>
                                                  ->  XN S3 Query Scan nytaxirides  (cost=45000000.00..45000010.00 rows=1000 width=8)
                                                        ->  S3 Aggregate  (cost=45000000.00..45000000.00 rows=1000 width=0)
                                                              ->  S3 Seq Scan adb305.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..37500000.00 rows=3000000000 width=0)
                                                                  <b> Filter: (passenger_count = 4)</b>
```

* Now include Spectrum data by adding a month whose data is in Spectrum

```sql
EXPLAIN 
SELECT year, month, type, COUNT(*) 
FROM adb305_view_NYTaxiRides 
WHERE year = 2016 AND month IN (1,2) AND passenger_count = 4 
GROUP BY 1,2,3 ORDER BY 1,2,3;

```

```sql
QUERY PLAN
XN Merge  (cost=1000090029268.92..1000090029268.92 rows=2 width=48)
  Merge Key: derived_col1, derived_col2, derived_col3
  ->  XN Network  (cost=1000090029268.92..1000090029268.92 rows=2 width=48)
        Send to leader
        ->  XN Sort  (cost=1000090029268.92..1000090029268.92 rows=2 width=48)
              Sort Key: derived_col1, derived_col2, derived_col3
              ->  XN HashAggregate  (cost=90029268.90..90029268.90 rows=2 width=48)
                    ->  XN Subquery Scan adb305_view_nytaxirides  (cost=29221.33..90029268.88 rows=2 width=48)
                          ->  XN Append  (cost=29221.33..90029268.86 rows=2 width=38)
                                ->  XN Subquery Scan "*SELECT* 1"  (cost=29221.33..29221.34 rows=1 width=18)
                                      ->  XN HashAggregate  (cost=29221.33..29221.33 rows=1 width=18)
                                            ->  XN Seq Scan on t201601_pqt  (cost=0.00..28905.70 rows=31563 width=18)
                                                 <b> Filter: ((passenger_count = 4) AND ("year" = 2016) AND (("month" = 1) OR ("month" = 2))) </b>
                                ->  XN Subquery Scan "*SELECT* 2"  (cost=90000047.50..90000047.52 rows=1 width=38)
                                      ->  XN HashAggregate  (cost=90000047.50..90000047.51 rows=1 width=38)
                                            ->  XN Partition Loop  (cost=90000000.00..90000037.50 rows=1000 width=38)
                                                  ->  XN Seq Scan PartitionInfo of adb305.nytaxirides  (cost=0.00..17.50 rows=1 width=30)
                                                       <b> Filter: (("year" = 2016) AND (("month" = 1) OR ("month" = 2)))</b>
                                                  ->  XN S3 Query Scan nytaxirides  (cost=45000000.00..45000010.00 rows=1000 width=8)
                                                        ->  S3 Aggregate  (cost=45000000.00..45000000.00 rows=1000 width=0)
                                                              ->  S3 Seq Scan adb305.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..37500000.00 rows=3000000000 width=0)
                                                                  <b> Filter: (passenger_count = 4)</b>
```

```sql
EXPLAIN 
SELECT passenger_count, COUNT(*) 
FROM adb305.ny_pub 
WHERE year = 2016 AND month IN (1,2) 
GROUP BY 1 ORDER BY 1;
```

```sql
QUERY PLAN
XN Merge  (cost=1000090005026.64..1000090005027.14 rows=200 width=12)
  <b>Merge Key: nytaxirides.derived_col1</b>
  ->  XN Network  (cost=1000090005026.64..1000090005027.14 rows=200 width=12)
        Send to leader
        ->  XN Sort  (cost=1000090005026.64..1000090005027.14 rows=200 width=12)
              <b>Sort Key: nytaxirides.derived_col1</b>
              ->  XN HashAggregate  (cost=90005018.50..90005019.00 rows=200 width=12)
                    ->  XN Partition Loop  (cost=90000000.00..90004018.50 rows=200000 width=12)
                          ->  XN Seq Scan PartitionInfo of adb305.nytaxirides  (cost=0.00..17.50 rows=1 width=0)
                               Filter: (("year" = 2016) AND (("month" = 1) OR ("month" = 2)))
                          ->  XN S3 Query Scan nytaxirides  (cost=45000000.00..45002000.50 rows=200000 width=12)
                                <b> ->  S3 HashAggregate  (cost=45000000.00..45000000.50 rows=200000 width=4)</b>
                                      ->  S3 Seq Scan adb305.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..30000000.00 rows=3000000000 width=4)
```

```sql
EXPLAIN 
SELECT type, COUNT(*) 
FROM adb305.ny_pub 
WHERE year = 2016 AND month IN (1,2) 
GROUP BY 1 ORDER BY 1 ;
```

```sql
QUERY PLAN
XN Merge  (cost=1000075000042.52..1000075000042.52 rows=1 width=30)
  <b>Merge Key: nytaxirides."type"</b>
  ->  XN Network  (cost=1000075000042.52..1000075000042.52 rows=1 width=30)
        Send to leader
        ->  XN Sort  (cost=1000075000042.52..1000075000042.52 rows=1 width=30)
              <b>Sort Key: nytaxirides."type"</b>
              ->  XN HashAggregate  (cost=75000042.50..75000042.51 rows=1 width=30)
                    ->  XN Partition Loop  (cost=75000000.00..75000037.50 rows=1000 width=30)
                          ->  XN Seq Scan PartitionInfo of adb305.nytaxirides  (cost=0.00..17.50 rows=1 width=22)
                               Filter: (("year" = 2016) AND (("month" = 1) OR ("month" = 2)))
                          ->  XN S3 Query Scan nytaxirides  (cost=37500000.00..37500010.00 rows=1000 width=8)
                              <b>  ->  S3 Aggregate  (cost=37500000.00..37500000.00 rows=1000 width=0)</b>
                                      ->  S3 Seq Scan adb305.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..30000000.00 rows=3000000000 width=0)
```