---
title: "Native Redshift versus Redshift with Spectrum"
weight: 58
---

At this point, you might be asking yourself, why would I ever not use Spectrum? Well, you still get additional value from loading data into Redshift. In fact, it turns out that our last query runs even faster when executed exclusively in native Redshift. Running a full test is beyond the time we have for the lab, so let’s review test results that compares running the last query with Redshift Spectrum versus exclusively with Redshift on various cluster sizes.

As a rule of thumb, queries that aren’t dominated by I/O and involve multiple joins are better optimized in native Redshift.  Furthermore, the variability in latency in native Redshift is significantly lower. For use cases where you have tight performance SLAs on queries, you may want to consider using Redshift exclusively to support those queries.

On the other hand, when you have the need to perform large scans, you could benefit from the best of both worlds: higher performance at lower cost. For instance, imagine we needed to enable our business analysts to interactively discover insights across a vast amount of historical data. 
