---
title: "Launch Redshift Cluster"
draft: true
weight: 7
---

## Launch Redshift Cluster
Navigate to the **Amazon Redshift Dashboard** and click on the "Launch Cluster" button.  
```
https://console.aws.amazon.com/redshift/home#cluster-list:
```
* Cluster Details - Enter values as appropriate for your organization.  Note the Master user password as you will not be able to retrieve this value later.
![](/images/ClusterDetails.png)
* Node Configuration - Modify the Cluster type to Multi Node and set the Number of compute nodes to 2.
![](/images/NodeConfiguration.png)
* Additional Configuration - Choose the VPC, Subnet Group, VPC Security group, and Role which you identified or created earlier.
![](/images/AdditionalConfiguration.png)  
![](/images/AssignRole.png)