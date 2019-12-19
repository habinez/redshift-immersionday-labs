---
title: "Plan for the Future"
weight: 46
---


In this final part of this lab, we will compare different strategies for maintaining more recent or *HOT* data within Redshift direct-attached storage, and keeping older *COLD* data in S3 by performing the following steps:
* Allow for trailing 5 quarters reporting by adding the Q4 2015 data to Redshift DAS:
	* Anticipating that we’ll want to ”age-off” the oldest quarter on a 3 month basis, architect your DAS table to make this easy to maintain and query.
	* Adjust your Redshift Spectrum table to exclude the Q4 2015 data.
* Develop and execute a plan to move the Q4 2015 data to S3.
	* What are the discrete steps to be performed?
	* What extra-Redshift functionality must be leveraged?
	* Simulating the extra-Redshift steps with the existing Parquet data, age-off the Q4 2015 data from Redshift DAS 	and perform any needed steps to maintain a single version of the truth.

* There are several options to accomplish this goal. Anticipating that we’ll want to ”age-off” the oldest quarter on a 3 month basis, architect your DAS table to make this easy to maintain and query. How about something like this?

```sql
CREATE OR REPLACE VIEW adb305_view_NYTaxiRides AS
  SELECT * FROM workshop_das.taxi_201504 
UNION ALL 
  SELECT * FROM workshop_das.taxi_201601
UNION ALL 
  SELECT * FROM workshop_das.taxi_201602
UNION ALL 
  SELECT * FROM workshop_das.taxi_201603
UNION ALL 
  SELECT * FROM workshop_das.taxi_201604
UNION ALL 
  SELECT * FROM adb305.ny_pub
WITH NO SCHEMA BINDING;
```
	
* Or something like this? Bulk DELETE-s in Redshift are actually quite fast (with one-time single-digit minute time to VACUUM), so this is also a valid configuration as well:

```sql
CREATE OR REPLACE VIEW adb305_view_NYTaxiRides AS
   SELECT * FROM workshop_das.taxi_current
UNION ALL 
  SELECT * FROM adb305.ny_pub
WITH NO SCHEMA BINDING;
```

* If needed, the Redshift DAS tables can also be populated from the Parquet data with COPY. Note: This will highlight a data design when we created the Parquet data

**COPY with Parquet doesn’t currently include a way to specify the partition columns as sources to populate the target Redshift DAS table. The current expectation is that since there’s no overhead (performance-wise) and little cost in also storing the partition data as actual columns on S3, customers will store the partition column data as well.**

* We’re going to show how to work with the scenario where this pattern wasn’t followed. Use the single table option for this example
	
```sql
CREATE TABLE workshop_das.taxi_current 
DISTSTYLE EVEN 
SORTKEY(year, month, type) AS 
SELECT * FROM adb305.ny_pub WHERE 1 = 0;
```

* And, create a helper table that doesn't include the partition columns from the Redshift Spectrum table.

```sql
CREATE TABLE workshop_das.taxi_loader AS 
  SELECT vendorid, pickup_datetime, dropoff_datetime, ratecode, passenger_count, 
  	trip_distance, fare_amount, total_amount, payment_type 
  FROM workshop_das.taxi_current 
  WHERE 1 = 0;
```

### Parquet copy continued

* The population could be scripted easily; there are also a few different patterns that could be followed.  Below is a script which issues a seperate copy command for each partition where the **type=green**.  Once complete, seperate scripts would need to be used for other **type** partitions.

```sql
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2015/month=10/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2015/month=11/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2015/month=12/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=1/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=2/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=3/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=4/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=5/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=6/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=7/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=8/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=9/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=10/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=11/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=12/type=green' IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]' FORMAT AS PARQUET;
```

```sql
INSERT INTO workshop_das.taxi_current 
  SELECT *, DATE_PART(year,pickup_datetime), DATE_PART(month,pickup_datetime), 'green' 
  FROM workshop_das.taxi_loader;
```

```sql
TRUNCATE workshop_das.taxi_loader;
```

### Redshift Spectrum can, of course, also be used to populate the table(s).

```sql
DROP TABLE IF EXISTS workshop_das.taxi_201601;
CREATE TABLE workshop_das.taxi_201601 AS SELECT * FROM adb305.ny_pub WHERE year = 2016 AND month IN (1,2,3); 
CREATE TABLE workshop_das.taxi_201602 AS SELECT * FROM adb305.ny_pub WHERE year = 2016 AND month IN (4,5,6);
CREATE TABLE workshop_das.taxi_201603 AS SELECT * FROM adb305.ny_pub WHERE year = 2016 AND month IN (7,8,9);
CREATE TABLE workshop_das.taxi_201604 AS SELECT * FROM adb305.ny_pub WHERE year = 2016 AND month IN (10,11,12);
```

### Adjust your Redshift Spectrum table to exclude the Q4 2015 data

**Note for the Redshift Editor users:** Adjust accordingly based on how many of the partitions you added above.

```sql
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=10, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=10, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=10, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=11, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=11, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=11, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=12, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=12, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=12, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=2, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=2, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=2, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=3, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=3, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=3, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=4, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=4, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=4, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=5, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=5, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=5, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=6, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=6, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=6, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=7, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=7, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=7, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=8, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=8, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=8, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=9, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=9, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=9, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=10, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=10, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=10, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=11, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=11, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=11, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=12, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=12, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=12, type='green');
```

* Now, regardless of method, there’s a view covering the trailing 5 quarters in Redshift DAS, and all of time on Redshift Spectrum, completely transparent to users of the view. What would be the steps to “age-off” the Q4 2015 data?

	1. Put a copy of the data from Redshift DAS table to S3. What would be the command(s)?
		* UNLOAD
	1. Extend the Redshift Spectrum table to cover the Q4 2015 data with Redshift Spectrum.
		* ADD Partition.
	1. Remove the data from the Redshift DAS table:
		* Either DELETE or DROP TABLE (depending on the implementation).

