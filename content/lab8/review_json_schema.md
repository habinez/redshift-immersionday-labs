---
title: "Review JSON Schema"
weight: 83
---

1. Navigate to the [Glue Catalog](https://console.aws.amazon.com/glue/home?#catalog:tab=tables) and click on the *cusnested-json* table. \

2. Click – *Edit Schema* and review the schema created by the crawler.
![](/images/lab8_table1.png)

3. The JSON dataset contains struct, array columns. 
![](/images/lab8_table2.png)

{{% notice note %}}
The Crawler created a superset of the columns in the table definition. Customer_1.JSON file has the c_comment column but customer_2.JSON and customer_3.JSON does not have the c_comment column.
{{% /notice %}}