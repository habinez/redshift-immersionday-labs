---
title: "Review JSON Schema"
weight: 83
---


## Review JSON Schema
Navigate to the Glue Catalog and click on the *cusnested-json* table. 

```
https://console.aws.amazon.com/glue/home?#catalog:tab=tables
```
Click – *Edit Schema* and review the schema created by the crawler.
![](/images/lab8_table1.png)

The JSON dataset contains struct, array columns. 
![](/images/lab8_table2.png)

Note: The Crawler created a superset of the columns in the table definition. Customer_1.JSON file has the c_comment column but customer_2.JSON and customer_3.JSON does not have the c_comment column.
