---
title: "Configure Client Tool"
weight: 8
---

## Configure Client Tool
* See [Prerequisites](../prerequisites) for more details on downloading and installing [SQL Workbench/J](http://www.sql-workbench.net) and the [Redshift JDBC Driver](https://docs.aws.amazon.com/redshift/latest/mgmt/connecting-to-cluster.html). 
* Launch SQL Workbench/J and navigate to [File | Manage Drivers].
* Select "Amazon Redshift" and set the driver Library location to where you downloaded the Redshift JDBC Driver. Click Ok.
![](/images/Library.png)
* Navigate to [File | Connect Window] to create a new connection profile and modify the following settings and once complete click on the "Test Connection" button.
  * Name - "LabConnection"
  * Driver - Amazon Redshift (com.amazon.redshift.jdbc.Driver)
  * URL - Find this by navigating to the [Cluster List](https://console.aws.amazon.com/redshift/home?cluster-details:#cluster-list:), selecting your cluster, and copying the JDBC URL.  
  ![](/images/JDBCUrl.png)
  * Username - [Master user name]
  * Password - [Master user password]
  * Autocommit - Enabled
  
![](/images/Connection.png)
