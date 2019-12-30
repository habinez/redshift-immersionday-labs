---
title: "Load JSON data using Redshift Spectrum"
weight: 86
---

Let’s leverage Redshift Spectrum to ingest JSON data set in Redshift local tables. This is one usage pattern to leverage Redshift Spectrum for ELT. We will also join Redshift local tables to external tables in this example.

1. Create Redshift local staging tables.

```sql
DROP TABLE IF EXISTS  public.stg_customer;
create table stg_customer 
( c_custkey     integer not null,
  c_name        varchar(25) not null,
  c_address     varchar(40) not null,
  c_nationkey   integer not null,
  c_phone       char(15) not null,
  c_acctbal     decimal(15,2) not null,
  c_mktsegment  char(10) not null,
  c_comment varchar(117) not null)
  backup no;
  
DROP TABLE IF EXISTS  public. stg_orders;

create table stg_orders  
( o_orderkey       integer not null,
  o_custkey        integer not null,
  o_orderstatus    char(1) not null,
  o_totalprice     decimal(15,2) not null,
  o_orderdate      date not null,
  o_orderpriority  char(15) not null,  
  o_clerk          varchar(20) not null, 
  o_shippriority   integer not null,
  o_comment        varchar(100) not null)
backup no;

DROP TABLE IF EXISTS  public. stg_lineitem;
create table stg_lineitem 
( l_orderkey    integer not null,
  l_partname    varchar(50),
  l_supplyname  varchar(50),
  l_linenumber  integer not null,
  l_quantity    decimal(15,2) not null,
  l_extendedprice  decimal(15,2) not null,
  l_discount    decimal(15,2) not null,
  l_tax         decimal(15,2) not null,
  l_returnflag  char(1) not null,
  l_linestatus  char(1) not null,
  l_shipdate    date not null,
  l_commitdate  date not null,
  l_receiptdate date not null,
  l_shipinstruct char(25) not null,
  l_shipmode     char(10) not null,
  l_comment varchar(44) not null)
backup no;
```

2. Write the ELT code to ingest JSON data residing on s3 using Redshift Spectrum into Redshift local tables.

```sql
BEGIN TRANSACTION;

TRUNCATE TABLE public.stg_customer;
INSERT INTO public.stg_customer
(        c_custkey
       , c_name
       , c_address
       , c_nationkey
       , c_phone
       , c_acctbal
       , c_mktsegment
       , c_comment
)
SELECT row_number() over (order by cust.c_name),
       cust.c_name, 
       cust.c_address,
       cust.c_nationkey,
       cust.c_phone,
       cust.c_acctbal,
       cust.c_mktsegment,
       coalesce(cust.c_comment,'unk')
FROM nested_json.cusnested_json cust;

TRUNCATE TABLE public.stg_orders ;
INSERT INTO public.stg_orders 
(        o_orderkey
       , o_custkey
       , o_orderstatus
       , o_totalprice
       , o_orderdate
       , o_orderpriority
       , o_clerk
       , o_shippriority
       , o_comment
)
SELECT row_number() over (order by cust.c_name) 
       ,stgcust.c_custkey
       ,co.o_orderstatus
       ,co.o_totalprice
       ,to_date(co.o_orderdate, 'YYYY-MM-DD') 
       ,co.o_order_priority
       ,co.o_clerk
       ,co.o_ship_priority
       ,co.o_comment
FROM nested_json.cusnested_json cust, 
     cust.orders.order co,
     public.stg_customer stgcust
WHERE cust.c_name = stgcust.c_name;


TRUNCATE TABLE stg_lineitem;
INSERT INTO public.stg_lineitem 
(        l_orderkey
       , l_partname
       , l_supplyname
       , l_linenumber
       , l_quantity
       , l_extendedprice
       , l_discount
       , l_tax
       , l_returnflag
       , l_linestatus
       , l_shipdate
       , l_commitdate
       , l_receiptdate
       , l_shipinstruct
       , l_shipmode
       , l_comment
)

SELECT so.o_orderkey 
       , litem.p_name
       , litem.s_name
       , litem.l_linenumber
       , litem.l_quantity
       , litem.l_extendedprice
       , litem.l_discount
       , litem.l_tax
       , litem.l_returnflag
       , litem.l_linestatus
       , to_date(litem.l_shipdate, 'YYYY-MM-DD')
       , to_date(litem.l_commitdate, 'YYYY-MM-DD')
       , to_date(litem.l_receiptdate, 'YYYY-MM-DD')
       , litem.l_shipinstruct
       , litem.l_shipmode
       , litem.l_comment
FROM nested_json.cusnested_json cust, 
     cust.orders.order co,
     co.lineitems.lineitem litem,
     public.stg_orders so,
     public.stg_customer sc
WHERE to_date(co.o_orderdate, 'YYYY-MM-DD') = so.o_orderdate
    and co.o_totalprice = so.o_totalprice
    and so.o_custkey = sc.c_custkey
    and sc.c_name = cust.c_name
;

END TRANSACTION;
```

3. Query the counts in each of the tables.

```sql
SELECT 'customer', count(*) from stg_customer
UNION ALL
SELECT 'orders', count(*) from stg_orders
UNION ALL
SELECT 'lineitem', count(*) from stg_lineitem;
``` 
![](/images/lab8_query5.png)

4. Consider wrapping the ELT code in a Redshift stored procedure

```sql
CREATE OR REPLACE PROCEDURE sp_loadtpch(indate in date) as
$$
declare
  integer_var int;
begin

RAISE INFO 'running staging for date %',  indate;

TRUNCATE TABLE public.stg_customer;
INSERT INTO public.stg_customer
(        c_custkey
       , c_name
       , c_address
       , c_nationkey
       , c_phone
       , c_acctbal
       , c_mktsegment
       , c_comment
)
SELECT row_number() over (order by cust.c_name),
       cust.c_name, 
       cust.c_address,
       cust.c_nationkey,
       cust.c_phone,
       cust.c_acctbal,
       cust.c_mktsegment,
       coalesce(cust.c_comment,'unk')
FROM nested_json.cusnested_json cust;

GET DIAGNOSTICS integer_var := ROW_COUNT;
RAISE INFO 'rows inserted into stg_customer = %', integer_var;

TRUNCATE TABLE public.stg_orders ;
INSERT INTO public.stg_orders 
(        o_orderkey
       , o_custkey
       , o_orderstatus
       , o_totalprice
       , o_orderdate
       , o_orderpriority
       , o_clerk
       , o_shippriority
       , o_comment
)
SELECT row_number() over (order by cust.c_name) 
       ,stgcust.c_custkey
       ,co.o_orderstatus
       ,co.o_totalprice
       ,to_date(co.o_orderdate, 'YYYY-MM-DD') 
       ,co.o_order_priority
       ,co.o_clerk
       ,co.o_ship_priority
       ,co.o_comment
FROM nested_json.cusnested_json cust, 
     cust.orders.order co,
     public.stg_customer stgcust
WHERE cust.c_name = stgcust.c_name;

GET DIAGNOSTICS integer_var := ROW_COUNT;
RAISE INFO 'rows inserted into stg_orders = %', integer_var;

TRUNCATE TABLE stg_lineitem;
INSERT INTO public.stg_lineitem 
(        l_orderkey
       , l_partname
       , l_supplyname
       , l_linenumber
       , l_quantity
       , l_extendedprice
       , l_discount
       , l_tax
       , l_returnflag
       , l_linestatus
       , l_shipdate
       , l_commitdate
       , l_receiptdate
       , l_shipinstruct
       , l_shipmode
       , l_comment
)
SELECT so.o_orderkey 
       , litem.p_name
       , litem.s_name
       , litem.l_linenumber
       , litem.l_quantity
       , litem.l_extendedprice
       , litem.l_discount
       , litem.l_tax
       , litem.l_returnflag
       , litem.l_linestatus
       , to_date(litem.l_shipdate, 'YYYY-MM-DD')
       , to_date(litem.l_commitdate, 'YYYY-MM-DD')
       , to_date(litem.l_receiptdate, 'YYYY-MM-DD')
       , litem.l_shipinstruct
       , litem.l_shipmode
       , litem.l_comment
FROM nested_json.cusnested_json cust, 
     cust.orders.order co,
     co.lineitems.lineitem litem,
     public.stg_orders so,
     public.stg_customer sc
WHERE to_date(co.o_orderdate, 'YYYY-MM-DD') = so.o_orderdate
    and co.o_totalprice = so.o_totalprice
    and so.o_custkey = sc.c_custkey
    and sc.c_name = cust.c_name
;
	 
GET DIAGNOSTICS integer_var := ROW_COUNT;
RAISE INFO 'rows inserted into stg_lineitem = %', integer_var;
 
END;	  
$$ LANGUAGE plpgsql;
```

Execute the procedure
```sql
call sp_loadtpch(current_date);
```
