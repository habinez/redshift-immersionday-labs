---
title: "Go Back in Time"
weight: 43
---

## Go Back in Time
In the next part of this lab, we will perform the following activities:
* Query historical data residing on S3 by create an external DB for Redshift Spectrum.
* Introspect the historical data, perhaps rolling-up the data in novel ways to see trends over time, or other dimensions.
* Enforce reasonable use of the cluster with Redshift Spectrum-specific Query Monitoring Rules (QMR).
	* Test the QMR setup by writing an excessive-use query.

**Note the partitioning scheme is Year, Month, Type (where Type is a taxi company). Here's a quick Screenshot:**

```javascript
https://s3.console.aws.amazon.com/s3/buckets/serverless-analytics/canonical/NY-Pub/?region=us-east-1&tab=overview
```
![](/images/canonical_year.png)

```javascript
https://s3.console.aws.amazon.com/s3/buckets/serverless-analytics/canonical/NY-Pub/year%253D2016/month%253D1/?region=us-east-1&tab=overview
```
![](/images/canonical_month.png)
```javascript
https://s3.console.aws.amazon.com/s3/buckets/serverless-analytics/canonical/NY-Pub/year%253D2016/month%253D1/type%253Dgreen/?region=us-east-1&tab=overview
```
![](/images/canonical_type.png)


### Create external schema (and DB) for Redshift Spectrum
Because external tables are stored in a shared Glue Catalog for use within the AWS ecosystem, they can be built and maintained using a few different tools, e.g. Athena, Redshift, and Glue. 
	
* Use the AWS Glue Crawler to create your external table adb305.ny_pub stored in parquet format under location s3://us-west-2.serverless-analytics/canonical/NY-Pub/.

	1. Navigate to the **Glue Crawler Page**. https://console.aws.amazon.com/glue/home?#catalog:tab=crawlers
	![](/images/crawler_0.png)
	1. Click on *Add Crawler*, and enter the crawler name *NYTaxiCrawler* and click *Next*.
	![](/images/crawler_1.png)
	1. Select *Data stores* as the source type and click *Next*.
	![](/images/crawler_2.png)
	1. Choose *S3* as the data store and the include path of *s3://us-west-2.serverless-analytics/canonical/NY-Pub*
	![](/images/crawler_3.png)
	1. *Create an IAM Role* and enter the name AWSGlueServiceRole-*RedshiftImmersion*.  
	![](/images/crawler_4.png)
	1. Select *Run on demand* for the frequency.
	![](/images/crawler_5.png)
	1. Click on *Add database* and enter the Database of *spectrumdb*
	![](/images/crawler_6.png)
	1. Select all remaining defaults. Once the Crawler has been created, click on *Run Crawler*.
	![](/images/crawler_7.png)
	1. Once the Crawler has completed its run, you will see a new table in the Glue Catalog. https://console.aws.amazon.com/glue/home?#catalog:tab=tables
	![](/images/crawler_8.png)
	1. Click on the *ny_pub* table, notice the recordCount of 2.87 billion. 
	![](/images/crawler_9.png)


* Now that the table has been cataloged, switch back to your Redshift query editor and create an external schema **adb305** pointing to your Glue Catalog Database **spectrumdb**


```sql
CREATE external SCHEMA adb305
FROM data catalog DATABASE 'spectrumdb' 
IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]'
CREATE external DATABASE if not exists;
```
{{% notice tip %}}
Run the query from the previous step using the external table instead of the direct-attached storage (DAS).

{{% /notice %}}

```sql
SELECT TO_CHAR(pickup_datetime, 'YYYY-MM-DD'),
COUNT(*)
FROM adb305.ny_pub
WHERE YEAR = 2016 and Month = 01
GROUP BY 1
ORDER BY 1;
```

  
### Add a Redshift Spectrum Query Monitoring Rule to ensure reasonable use
In Amazon Redshift workload management (WLM), query monitoring rules define metrics-based performance boundaries for WLM queues and specify what action to take when a query goes beyond those boundaries. Setup a Query Monitoring Rule to ensure reasonable use.

```javascript
https://docs.aws.amazon.com/redshift/latest/dg/cm-c-wlm-query-monitoring-rules.html
```
Take a look at SVL_QUERY_METRICS_SUMMARY view shows the maximum values of metrics for completed queries. This view is derived from the STL_QUERY_METRICS system table. Use the values in this view as an aid to determine threshold values for defining query monitoring rules.

```javascript
https://docs.aws.amazon.com/redshift/latest/dg/r_SVL_QUERY_METRICS_SUMMARY.html
```

Quick Note on QLM: The WLM configuration properties are either dynamic or static. Dynamic properties can be applied to the database without a cluster reboot, but static properties require a cluster reboot for changes to take effect. Additional info here:

```javascript
https://docs.aws.amazon.com/redshift/latest/mgmt/workload-mgmt-config.html
```
