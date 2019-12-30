---
title: "Query JSON data using Redshift Spectrum"
weight: 85
---


1. Login to Redshift and create external schema

```sql
CREATE external SCHEMA nested_json
FROM data catalog DATABASE 'nested-json' 
IAM_ROLE 'arn:aws:iam::[Your-AWS-Account_Id]:role/[Your-Redshift_Role]'
CREATE external DATABASE if not exists;
```

2. Run the following query to view customer name, address and comments

```sql
SELECT cust.c_name, cust.c_address, cust.c_comment
FROM nested_json.cusnested_json cust
ORDER BY cust.c_name;
```

You will see the following output. Notice how c_comment key was not present in customer_2 and customer_3 JSON file. This demonstrates that the format of files could be different and using the Glue crawler you can create a superset of columns – supporting schema evolution. The files which have the key will return the value and the files that do not have that key will return null.
![](/images/lab8_query1.png)

Filter the data by nationkey and address:

```sql
SELECT cust.c_name, 
  cust.c_nationkey, 
  cust.c_address
FROM nested_json.cusnested_json cust
WHERE cust.c_nationkey = '-2013'
  AND cust.c_address like 'AAA%';
```
![](/images/lab8_query1.1.png)



3. Query the Order struct and check how many orders each customer has:
```
Order array<
  struct<
    o_orderstatus:String, 
    o_totalprice:Double,
    o_orderdate:String,
    o_order_priority:String,
    o_clerk:String,
    o_ship_priority:Int,
    o_comment:String
  >
>
```

```sql
SELECT  cust.c_name, count(*)
FROM nested_json.cusnested_json cust,
     cust.orders.order co  
GROUP BY cust.c_name
ORDER BY cust.c_name;
```
![](/images/lab8_query2.png)

4. Query the Orders array to flatten or un-nest the Order details. Notice how the scalar in an array is queried using alias (e.g. co.o_totalprice).  Struct data type is queried using the dot-notation (e.g. cust.c_name).

```sql
SELECT cust.c_name,
           co.o_orderstatus,
           co.o_totalprice,
           to_date(co.o_orderdate, 'YYYY-MM-DD'),
           co.o_order_priority,
           co.o_clerk,
           co.o_ship_priority,
           co.o_comment  
FROM nested_json.cusnested_json cust,
           cust.orders.order co;
```
![](/images/lab8_query3.png)

5. Further un-nest lineitems by using a left join.

```sql
SELECT cust.c_name,
       to_date(co.o_orderdate, 'YYYY-MM-DD'),  
       litem.l_linenumber,
       litem.l_quantity,
       litem.l_extendedprice,
       litem.l_discount,
       litem.l_tax,
       litem.l_returnflag,
       litem.l_linestatus,
       to_date(litem.l_shipdate, 'YYYY-MM-DD'),
       to_date(litem.l_commitdate, 'YYYY-MM-DD'),
       to_date(litem.l_receiptdate, 'YYYY-MM-DD'),
       litem.l_shipinstruct,
       litem.l_shipmode,
       litem.l_comment,
FROM nested_json.cusnested_json cust
LEFT JOIN cust.orders.order co on true
LEFT JOIN co.lineitems.lineitem litem on true	
;
```

6. Find the retail price for each customer

```sql
SELECT cust.c_name,
  sum(litem.p_retailprice)
FROM  nested_json.cusnested_json cust
LEFT JOIN cust.orders.order co on true
LEFT JOIN co.lineitems.lineitem litem on true	
GROUP BY cust.c_name;
```

7. Aggregating nested data with subqueries

```sql
SELECT cust.c_name, 
       (SELECT COUNT(*) FROM cust.orders.order o) AS ordercount,
       (SELECT COUNT(*) FROM cust.orders.order o, o.lineitems.lineitem l) as lineitemcount
FROM nested_json.cusnested_json cust
ORDER BY c_name;
```
![](/images/lab8_query4.png)