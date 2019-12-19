---
title: "Table Maintainance - Vacuum"
weight: 27
---





## Table Maintenance - VACUUM
You should run the VACUUM command following a significant number of deletes or updates.  To perform an update, Amazon Redshift deletes the original row and appends the updated row, so every update is effectively a delete and an insert.  While, Amazon Redshift recently enabled a feature which automatically and periodically reclaims space, it is a good idea to be aware of how to manually perform this operation.  You can run a full vacuum, a delete only vacuum, or sort only vacuum.

Capture the initial space usage of the ORDERS table.
```sql
select col, count(*)
from stv_blocklist, stv_tbl_perm
where stv_blocklist.tbl = stv_tbl_perm.id and
stv_blocklist.slice = stv_tbl_perm.slice and
stv_tbl_perm.name = 'orders' and
col <= 5
group by col
order by col;
```
|col|count|
|---|---|
|0|352|
|1|352|
|2|64|
|3|448|
|4|64|
|5|128|

Delete rows from the ORDERS table.
```sql
delete orders where o_orderdate between '1992-01-01' and '1993-01-01';
```

Confirm that Redshift did not automatically reclaim space by running the following query again and noting the values have not changed.
```sql
select col, count(*)
from stv_blocklist, stv_tbl_perm
where stv_blocklist.tbl = stv_tbl_perm.id
and stv_blocklist.slice = stv_tbl_perm.slice
and stv_tbl_perm.name = 'orders' and
col <= 5
group by col
order by col;

```

Run the VACUUM command
```sql
vacuum delete only orders;
```

Confirm that the VACUUM command reclaimed space by running the follwoing quer again and noting the values have changed.
```sql
select col, count(*)
from stv_blocklist, stv_tbl_perm
where stv_blocklist.tbl = stv_tbl_perm.id
and stv_blocklist.slice = stv_tbl_perm.slice
and stv_tbl_perm.name = 'orders' and
col <= 5
group by col
order by col;
```
|col|count|
|---|---|
|0|320|
|1|320|
|2|64|
|3|416|
|4|64|
|5|128|

Note:  If you have a table with very few columns but a very large number of rows, the three hidden metadata identify columns (INSERT_XID, DELETE_XID, ROW_ID) will consume a disproportionate amount of the disk space for the table.  In order to optimize compression of the hidden columns, load the table in a single copy transaction where possible.  If you load the table with multiple separate COPY commands, the INSERT_XID column will not compress well and multiple vacuum operations will not improve compression of INSERT_XID.