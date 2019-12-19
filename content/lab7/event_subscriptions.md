---
title: "Event Subscriptions"
weight: 72
---




## Event Subscriptions
1. Navigate to your Redshift Events page.  Notice the *Events* involved with creating the cluster.  
```
https://console.aws.amazon.com/redshift/home?#events:cluster=
``` 
![](/images/Events.png) 

2. Click on the *Subscriptions* tab and then click on the *Create Event Subscription* button.
![](/images/CreateSubscription_0.png) 

3. Create a subscription for *any* severity *management* notification on *any cluster*.   Notice the types of event on the right you will recieve a notification for.
![](/images/CreateSubscription_1.png)

4. Name the subscription *ClusterManagement* and click *Next*.
![](/images/CreateSubscription_2.png)

5. Select the *Create New Topic* tab and enter the topic name *ClusterManagement*.  Add your email address and click *Add Recipient*.  Finally, click *Create*.
![](/images/CreateSubscription_3.png)

6. You will recieve an email shortly.  Click on the *Confirm subscription* link in the email.
![](/images/ConfirmSubscriptionEmail.png)

7. The link should take you to a final confirmation page confirming the subscription.
![](/images/SubscriptionConfirmed.png)