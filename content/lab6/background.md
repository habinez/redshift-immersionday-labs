---
title: "Background"
weight: 61
draft: true
---



When it comes to choosing a SQL-based database in AWS, there are many options. Sometimes it can be difficult to know which one to choose. For example, when would you use Amazon Aurora instead of [Amazon RDS PostgreSQL](https://aws.amazon.com/rds/postgresql/) or [Amazon Redshift](https://aws.amazon.com/redshift)? To answer this question, you must first understand the nature of the data workload and then evaluate other factors such as the quantity of data and query access patterns.

The design and capabilities of the different AWS services mean that each service has different strengths and excels at different workloads. This leads to trying to pick the right tool for the job, which can result in tradeoffs. But sometimes you don’t want to compromise.

This post explains how to use two services together—Amazon Redshift and Amazon RDS PostgreSQL—to avoid tradeoffs when choosing between a columnar data store and a row-based data store.

## Amazon Redshift
Amazon Redshift is a high-performance, petabyte-scale data warehouse service that excels at online analytical processing (OLAP) workloads. Databases such as RDS PostgreSQL or [Amazon Aurora](https://aws.amazon.com/rds/aurora/) typically store terabytes of data, and they excel at online transaction processing (OLTP) workloads.

Amazon Redshift uses a [columnar architecture](http://docs.aws.amazon.com/redshift/latest/dg/c_columnar_storage_disk_mem_mgmnt.html), which means the data is organized by columns on disk instead of row-by-row as in the OLTP approach. Columnar architecture offers advantages when querying a subset of the columns in a table by greatly reducing I/O. And because the data is stored by column, it can be highly compressed which further reduces I/O and allows more data to be stored and quickly queried.

RDS PostgreSQL uses a row-based architecture, which offers advantages when the workload is selecting, inserting, updating or deleting a small number of rows at a time, which is typical for OLTP.

Amazon Redshift also uses a [massively parallel processing](http://docs.aws.amazon.com/redshift/latest/dg/c_challenges_achieving_high_performance_queries.html) (MPP), shared-nothing architecture. This means that Amazon Redshift is designed to use all of the computing resources across many machines (called nodes) even when executing a single query. This provides excellent performance for analytical queries across a large number of rows. In contrast, most OLTP databases only use a subset of resources on one machine to process each query. This difference in architecture means that most OLTP databases can handle more concurrent queries because each query is typically less resource-intensive than those in Amazon Redshift.

Linking the high-performance power of Amazon Redshift with the feature-richness of RDS PostgreSQL is an attractive proposition because the two systems complement each other so well. How is it possible to link these two systems? An RDS PostgreSQL database is not an MPP database, but it does have features that enable multiple instances to be linked to one another.

Interestingly, Amazon Redshift was originally forked from PostgreSQL, which is why PostgreSQL drivers and API libpq work with Amazon Redshift. The combination of this PostgreSQL feature and Amazon Redshift compatibility lets the two systems be connected. This connection enables PostgreSQL to issue queries, and Amazon Redshift to return the results for processing to PostgreSQL.

Combining Amazon Redshift and RDS PostgreSQL provides the following benefits:

* Materialized views for cached copies of data that work well for high-concurrency dashboards
* High-concurrency, partitioned aggregate tables with block range indexes (BRIN).
* Procedural Language/PostgreSQL (PL/pgSQL) user-defined functions that can query Amazon Redshift by using dynamic SQL.
* Post-Amazon Redshift transformation, such as returning result sets as JSON.

![](/images/Redshift_pg_Image_1.PNG)

The diagram above shows how the connections flow between the end users and the databases. Optionally, you can connect directly to Amazon Redshift if needed. If that is the case, consider configuring pgbouncer-rr on an Amazon EC2 instance to simplify management of the two connections. The diagram below illustrates this solution:

![](/images/Redshift_pg_Image_2.PNG)

For further reading, check out Bob Strahan’s blog post [Query Routing and Rewrite: Introducing pgbouncer-rr for Amazon Redshift and PostgreSQL](https://blogs.aws.amazon.com/bigdata/post/Tx3G7177U6YHY5I/Query-Routing-and-Rewrite-Introducing-pgbouncer-rr-for-Amazon-Redshift-and-Postg) post.

RDS PostgreSQL includes two extensions to execute queries remotely. The first extension is the PostgreSQL foreign-data wrapper, postgres_fdw. The postgres_fdw module enables the creation of external tables. External tables can be queried in the same way as a local native table, However, the query is not currently executed entirely on the remote side because postgres_fdw doesn’t push down aggregate functions and limit clauses. When you perform an aggregate query through an external table, all the data is pulled into PostgreSQL for an aggregation step. This is unacceptably slow for any meaningful number of rows.

The second extension is dblink, which includes a function also called dblink. The dblink function allows the entire query to be pushed to Amazon Redshift. This lets Amazon Redshift do what it does best—query large quantities of data efficiently and return the results to PostgreSQL for further processing.


## Installation and setup
To set up this solution:
1. Launch an Amazon Redshift cluster.
2. Launch an RDS PostgreSQL (9.5+) instance in the same Availability Zone as the cluster in Step 1.
3. Configure the VPC security group for the Amazon Redshift cluster to allow an incoming connection from the RDS PostgreSQL endpoint.
4. Optional: load the Amazon Redshift sample data to run the queries included in this post.
5. Connect to the RDS PostgreSQL instance, and then run the following SQL code, replacing the <placeholders> with the values from your own instances:

```sql
CREATE EXTENSION postgres_fdw;
CREATE EXTENSION dblink;
CREATE SERVER foreign_server
        FOREIGN DATA WRAPPER postgres_fdw
        OPTIONS (host '<amazon_redshift _ip>', port '<port>', dbname '<database_name>', sslmode 'require');
CREATE USER MAPPING FOR <rds_postgresql_username>
        SERVER foreign_server
        OPTIONS (user '<amazon_redshift_username>', password '<password>');
```

For more information, see [dblink](http://www.postgresql.org/docs/9.5/static/contrib-dblink-function.html) in the PostgreSQL documentation.

## Basic querying
The dblink function requires you to pass in the SQL statement as a string and define the result set, as shown in the following query:
```sql
SELECT *
FROM dblink('foreign_server',$REDSHIFT$
    SELECT sellerid, sum(pricepaid) sales
    FROM sales 
    WHERE saletime >= '2008-01-01' 
    AND saletime < '2008-02-01' 
    GROUP BY sellerid 
    ORDER BY sales DESC
$REDSHIFT$) AS t1 (sellerid int, sales decimal);

```
In this example:

* The dblink function accepts the server connection (‘foreign_server’) that was created in the previous step.
* The SQL query is passed in as a string between double dollar quotes ($REDSHIFT$). Using dollar quotes makes reading and writing the query easier.
* The double dollar quotes are labeled REDSHIFT to help highlight the SQL that will be sent to Amazon Redshift.
* The query results are a recordset that you must name and for which you must specify the datatypes (AS t1(sellerid int, sales decimal). This enables further joining and processing.

The partial result set from this query is:
![](/images/Redshift_pg_Image_3.PNG)
