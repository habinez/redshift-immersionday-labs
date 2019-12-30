---
title: "Configure Client Tool"
weight: 8
---

## Configure Client Tool

* See [Prerequisites](../prerequisites) for more details on downloading and installing [DBeaver](https://dbeaver.io/) and the [Redshift JDBC Driver](https://docs.aws.amazon.com/redshift/latest/mgmt/connecting-to-cluster.html). 
* Launch DBeaver and navigate to [Database | New Database Connection].
* Search for "Redshift" and set the driver Library location to where you downloaded the Redshift JDBC Driver. Click Ok.
![](/images/Library-dbeaver.png)
* Select the presented option and click "Next"
* Find this by navigating to the [Cluster List](https://console.aws.amazon.com/redshift/home?cluster-details:#cluster-list:), selecting your cluster, and copying the JDBC URL. The URL is formed by as {host}:{port}/{database}  
  ![](/images/JDBCUrl.png)
  * Host: [Cluster host name from endpoint]
  * Port: -[5439]
  * User - [Master user name]
  * Password - [Master user password]

![](/images/JDBCURL-dbeaver.png)

* Click on the "Test Connection" button.

* Click Finish
