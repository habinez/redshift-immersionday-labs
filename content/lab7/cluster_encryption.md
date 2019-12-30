---
title: "Cluster Encryption"
weight : 73
---

## Cluster Encryption
Note: This portion of the lab will take ~45 minutes to complete based on the data loaded in [LAB 2 - Creating Redshift Clusters](../../lab2/cloudformation).  Please plan accordingly.

1. Navigate to your [Redshift Cluster list](https://console.aws.amazon.com/redshift/home?#cluster-list).  Select your cluster and click on *Cluster* -> *Modify Cluster*.

![](/images/ModifyCluster.png)

2. Select *KMS* for the database encryption and then click *Modify*.
![](/images/EnableKMS.png)

4. Notice your cluster enters a *resizing* status.  The process of encrypting your cluster is similar to resizing your cluster using the classic resize method.  All data is read, encrypted and re-written. During this time, the cluster is still avialable for read queries, but not write queries.
![](/images/Resizing.png)

5. You should also receive an email notification about the cluster resize because of the event subscription we setup earlier.
![](/images/ResizeNotification.png)
