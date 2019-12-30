---
title: "What Happened in 2016"
weight: 42
---

In the first part of this lab, we will perform the following activities:
* Load the Green company data for January 2016 into Redshift direct-attached storage (DAS) with COPY.
* Collect supporting/refuting evidence for the impact of the January, 2016 blizzard on taxi usage.
* The CSV data is by month on Amazon S3 ([link](https://s3.console.aws.amazon.com/s3/buckets/us-west-2.serverless-analytics/NYC-Pub/green/?region=us-west-2&tab=overview&prefixSearch=green_tripdata_2016)). Here's a quick screenshot from the S3 console: 

![](/images/green_2016.png)

* Here's Sample data from one file which can be previewed directly in the S3 [console](https://s3.console.aws.amazon.com/s3/object/us-west-2.serverless-analytics/NYC-Pub/green/green_tripdata_2013-08.csv?region=us-west-2&tab=select):

![](/images/green_preview.png)
	
	
### Build your DDL 
Create a schema `workshop_das` and table `workshop_das.green_201601_csv` for tables that will reside on the Redshift compute nodes, AKA the Redshift direct-attached storage (DAS) tables.

<details><summary>Hint</summary>
<p>
	
```sql
CREATE SCHEMA workshop_das;

CREATE TABLE workshop_das.green_201601_csv 
(
  vendorid                VARCHAR(4),
  pickup_datetime         TIMESTAMP,
  dropoff_datetime        TIMESTAMP,
  store_and_fwd_flag      VARCHAR(1),
  ratecode                INT,
  pickup_longitude        FLOAT4,
  pickup_latitude         FLOAT4,
  dropoff_longitude       FLOAT4,
  dropoff_latitude        FLOAT4,
  passenger_count         INT,
  trip_distance           FLOAT4,
  fare_amount             FLOAT4,
  extra                   FLOAT4,
  mta_tax                 FLOAT4,
  tip_amount              FLOAT4,
  tolls_amount            FLOAT4,
  ehail_fee               FLOAT4,
  improvement_surcharge   FLOAT4,
  total_amount            FLOAT4,
  payment_type            VARCHAR(4),
  trip_type               VARCHAR(4)
)
DISTSTYLE EVEN 
SORTKEY (passenger_count,pickup_datetime);
```



### Build your Copy Command 
* Build your copy command to copy the data from Amazon S3. This dataset has the number of taxi rides in the month of January 2016.


```sql
COPY workshop_das.green_201601_csv
FROM 's3://us-west-2.serverless-analytics/NYC-Pub/green/green_tripdata_2016-01.csv'
IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]'
DATEFORMAT 'auto'
IGNOREHEADER 1
DELIMITER ','
IGNOREBLANKLINES
REGION 'us-west-2'
;
```

* Determine how many rows you just loaded.

	
```sql
select count(1) from workshop_das.green_201601_csv;
--1445285
```

{{% notice tip %}}

The `[Your-Redshift_Role]` and `[Your-AWS-Account_Id]` in the above command should be replaced with the values determined at the beginning of the lab.

{{% /notice %}}
### Pin-point the Blizzard 
In this month, there is a date which had the lowest number of taxi rides due to a blizzard. Can you find that date?

```sql
SELECT TO_CHAR(pickup_datetime, 'YYYY-MM-DD'),
COUNT(*)
FROM workshop_das.green_201601_csv
GROUP BY 1
ORDER BY 1;
```