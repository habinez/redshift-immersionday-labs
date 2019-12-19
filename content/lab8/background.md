---
title: "Background"
weight: 81
---

## Background
Nested data support enables Redshift customers to directly query their nested data from Redshift through Spectrum.  Customers already have nested data in their Amazon S3 data lake.  For example, commonly java applications often use JSON as a standard for data exchange. Redshift Spectrum supports nested data types for the following format
* Apache Parquet
* Apache ORC
* JSON
* Amazon Ion

**Complex Data Types** 

*Struct* - this type allows multiple values of any type to be grouped together into a new type. Values are identified by a *Field Name* and *Field Type*.  In the following example, the *Name* field is a struct which has two nested fields of the *string* type.

```javascript
{Name: {Given:"John", Family:"Smith"}}
{Name: {Given:"Jenny", Family:"Doe"}}
{Name: {Given:"Andy", Family:"Jones"}}
```
```javascript
Name struct<Given: string, Family: string>
```

*Array* - this type defines a collection of an arbitrary number of elements of a certain type.  In the following example, the *Phones* field is an array of elements with the *string* type.

```javascript
{Phones: ["123-457789"]}
{Phones: ["858-8675309","415-9876543"]}
{Phones: []}
```
```javascript
Phones array<string>
```

Create even more complex data types by (deeply) nesting complex data types like struct, array or map.
```javascript
{
    Orders: [ 
        {Date: "2018-03-01 11:59:59", Price: 100.50}, 
        {Date: "2018-03-01 09:10:00", Price: 99.12} 
    ]
}

{Orders: [] }

{Orders: [ {Date: "2018-03-02 08:02:15", Price: 13.50} ]}
```
```javascript
Orders array<struct<Date: timestamp, Price: double>>
```

