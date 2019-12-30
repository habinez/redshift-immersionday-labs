---
title: "Loading Data"
weight: 24
---


## Loading Data
A COPY command loads large amounts of data much more efficiently than using INSERT statements, and stores the data more effectively as well.  Use a single COPY command to load data for one table from multiple files.  Amazon Redshift then automatically loads the data in parallel.  For your convenience, the sample data you will use is available in a public Amazon S3 bucket. To ensure that  Redshift performs a compression analysis, set the COMPUPDATE parameter to ON in your COPY commands. To COPY this data you will need to replace the [Your-AWS_Account_Id] and [Your-Redshift_Role] values in the script below.

```sql
COPY region FROM 's3://redshift-immersionday-labs/data/region/region.tbl.lzo'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

COPY nation FROM 's3://redshift-immersionday-labs/data/nation/nation.tbl.'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

COPY customer from 's3://redshift-immersionday-labs/data/customer/customer.tbl.'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

COPY orders from 's3://redshift-immersionday-labs/data/orders/orders.tbl.'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

COPY part from 's3://redshift-immersionday-labs/data/part/part.tbl.'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

COPY supplier from 's3://redshift-immersionday-labs/data/supplier/supplier.json' manifest
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

COPY lineitem from 's3://redshift-immersionday-labs/data/lineitem/lineitem.tbl.'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

COPY partsupp from 's3://redshift-immersionday-labs/data/partsupp/partsupp.tbl.'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;
```
If you are using 4 dc2.large clusters nodes, the estimated time to load the data is as follows, note you can check timing information on actions in the performance and query tabs on the redshift console:
* REGION (5 rows) - 20s
* NATION (25 rows) - 20s
*	CUSTOMER (15M rows) – 3m
* ORDERS - (76M rows) - 1m
* PART - (20M rows) - 4m
*	SUPPLIER - (1M rows) - 1m
* LINEITEM - (600M rows) - 13m
*	PARTSUPPLIER - (80M rows) 3m

Note: A few key takeaways from the above COPY statements.
1. COMPUPDATE PRESET ON will assign compression using the Amazon Redshift best practices related to the data type of the column but without analyzing the data in the table. 
1. COPY for the REGION table points to a specfic file (region.tbl.lzo) while COPY for other tables point to a prefix to multiple files (lineitem.tbl.)
1. COPY for the SUPPLIER table points a manifest file (supplier.json)