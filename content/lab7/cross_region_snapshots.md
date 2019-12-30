---
title: "Cross Region Snapshots"
weight: 74
---

## Cross Region Snapshots
1. Navigate to your [Redshift Cluster list](https://console.aws.amazon.com/redshift/home?#cluster-list).  Select your cluster and click on *Backup* -> *Configure Cross-region snapshots*.

![](/images/ConfigureCRR_0.png)

2. Select the *Yes* radio button to enable the copy.  Select the destination region of *us-east-2*.  Because the cluster is encrypted you must establish a grant in the other region to allow the snapshot to be re-encrypted.  Select *No* for the Existing Snapshot Copy Grant.  Name the Snapshot Copy Grant with the value *SnapshotGrant*.
![](/images/ConfigureCRR_1.png)

3. To demonstrate the cross-region replication, initiate a manual backup.  Click on *Backup* -> *Take Snapshot*.
![](/images/Snapshot_0.png)

4. Name the snapshot *CRRBackup* and click *Create*.
![](/images/Snapshot_1.png)

5. Navigate to your list of [snapshots](https://console.aws.amazon.com/redshift/home?#snapshots:id=;cluster=) and notice the snapshot is being created. 

![](/images/Snapshot_2.png)

6. Wait for the snapshot to finish being created.  The status will be *available*.
![](/images/Snapshot_3.png)

7. Navigate to the [us-east-2 region](https://us-east-2.console.aws.amazon.com/redshift/home?region=us-east-2#snapshots:id=;cluster=) by select *Ohio* from the region drop down, or navigate to the following link.  Notice that the snapshot is available and is in a *copying* status. 

![](/images/Snapshot_4.png)
