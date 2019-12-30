---
title: "Infer JSON Schema"
weight: 82
---

We will create AWS Glue crawler to infer the JSON dataset

1. Navigate to the [Glue Crawler Page](https://console.aws.amazon.com/glue/home?#catalog:tab=crawlers).  Click on *Add crawler*.
![](/images/lab8_crawler01.png)

2. Name the crawler *nested-json* and click *Next*
![](/images/lab8_crawler02.png)

3. Select *Data Stores* as source type and click *Next*
![](/images/lab8_crawler03.png)

4. Set the data store as *s3*, select the radio *Specified path in another account*, and enter the path *s3://redshift-immersionday-labs/data/nested-json*. 

![](/images/lab8_crawler04.png)

5. Click – *No* for *Add another data store* and click *Next*
![](/images/lab8_crawler05.png)

6. Select *Create an IAM role*, specify the name of the role as below and click *Next*
![](/images/lab8_crawler06.png)

7. Select *Run on demand* for the frequency and click *Next*
![](/images/lab8_crawler07.png)

8. Click *Add database* to create an new AWS Glue database
![](/images/lab8_crawler08.png)

9. Specify database name as *nested-json* and click *Create* 
![](/images/lab8_crawler09.png)

10. Specify a table prefix of *cus* and click *Next*
![](/images/lab8_crawler10.png)

11. Review all settings and click *Finish*
![](/images/lab8_crawler11.png)

12. We have now created the crawler, click on *Run it now*. The crawler will automatically infer the schema of the JSON datasets.
![](/images/lab8_crawler12.png)

13.When the crawler finishes, you will see the crawler in *Ready* status and you will see *Tables added* as *1*
![](/images/lab8_crawler13.png)
