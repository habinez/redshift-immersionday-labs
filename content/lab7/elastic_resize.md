---
title: "Elastic Resize"
weight: 75
---

## Elastic Resize
Note: This portion of the lab will take ~15 minutes to complete based on the data loaded in [LAB 2 - Creating Redshift Clusters](../../lab2).  Please plan accordingly.
1. Navigate to your Redshift Cluster list.  Select your cluster and click on *Cluster* -> *Resize*.  Note, if you don't see your cluster, you may have to change the *Region* drop-down.
![](/images/Resize_0.png)

2. Ensure the *Elastic Resize* radio is selected.  Choose the *New number of nodes*, and click *Resize*.
![](/images/Resize_1.png)

3. When the resize operation begins, you'll see the Cluster Status of *prep-for-resize*.
![](/images/Resize_2.png)

4. When the operation completes, you'll see the Cluster Status of *available* again.
![](/images/Resize_3.png)