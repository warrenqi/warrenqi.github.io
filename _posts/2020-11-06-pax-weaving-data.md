---
layout: post
title:  "Weaving Relations for Cache Performance (Partition Data Across - PAX) - reading notes"
date:   2020-11-06
categories:
---

Paper link: [https://research.cs.wisc.edu/multifacet/papers/vldb01_pax.pdf](https://research.cs.wisc.edu/multifacet/papers/vldb01_pax.pdf)

Paper published: 2001

# Impact:

PAX, as this paper originally describes in 2001, is the underlying idea for RCFiles, ORC, and Parquet. [https://en.m.wikipedia.org/wiki/RCFile](https://en.m.wikipedia.org/wiki/RCFile)


# Summary

Traditional database systems organize records one of two ways

1. **Row-oriented** a.k.a N-nary Storage Model slotted pages, or
2. **Column-oriented** a.k.a Decomposition Storage Model. 

This paper describes a new record organization model that is *more cache-optimized than NSM*, and at the same time, more efficient than DSM *when the number of columns increases*. This paper evaluates this new model, called Partition Attributes Across (PAX).


# Main Result(s):

This paper demonstrates that PAX, when implemented on a prototype database system (Shore), outperforms NSM in terms of CPU cache efficiency by reducing misses 50-75% when dataset fits within memory. PAX outperforms DSM as the number of query attributes increases.



# Content:

## S1. Introduction

1. NSM - stores records contingously starting from the beginning of each disk page, and uses an offset table (slot) at the end of each page to locate the beginning of each record. If a query uses only a fraction of the record's data (one column/attribute), the CPU cache behavior of NSM is poor: only a fraction of the data transferred to the cache is useful to the query: the item that the query processing algorithm requests and the transfer unit between the memory and the processor are typically not the same size. Loading the cache with useless data (a) wastes bandwidth, (b) pollutes the cache.
  
2. DSM - Proposed in 1985 to minimize the unnecessary IO of NSM. DSM partitions the dataset vertically into *n* sub-relations, each of which is connected by a key. Each attribute value is accessed only when the corresponding attribute is needed. Queries spanning multiple attributes, however, must spend additional time to join each attribute/column together.

3. PAX - For a given relation, PAX starts by storing data on disk page as NSM does. Within each page, PAX introduces the idea of a *"minipage"* which groups all the values of a particular attribute together, in a columnar fashion. During a sequential scan (e.g., to apply a predicate on a fraction of the record), PAX fully utilizes the cache resources, because on each cache miss, a group of a single attribute’s values are loaded into the cache together, reconstructing a record. PAX performs a *mini-join* among minipages, which incurs minimal cost because it does not have to look beyond the page - maintaining high data locality.

## S2. Background and related work

By 2001, it's clear that CPU execution speed was outpacing IO speed, and to unlock the full performance of the processor, computer systems have to more carefully avoid stalls. In a survey of OLTP systems, 50-90% of CPU stall times were due to data cache misses.

The N-ary Storage Model (NSM) has been the traditional storage format for OLTP systems. The dataset is broken into disk pages, where each page stores records sequentially. Each record is inserted into the first available free space of the page. Records have fixed length and variable length attributes, so at the end of every page, a pointer is written to point to the beginning of each record.

NSM is simple and straightforward, but its cache behavior is not optimal when a query executes on a subset of the attributes in a record - which is often the case. Because CPU caches work in blocks, each CPU cache miss will bring into the cache all values adjacent to the selected attribute value, even though none of this extra data will be useful in the computation.

The Decomposition Storage Model (DSM) vertically stripes a relation into columns, each containing only the values of a single attribute. Each attribute/column is scanned completely independently. DSM offers a high degree of spatial locality when sequentially accessing the values of one attribute. Intuitively, DSM works well when a query utilizes a small number of attributes in a relation, but its performance deteriorates when the query involves more attributes. This is because the database system must join all attributes together, and these joins become expensive as the number of attributes and dataset size increases beyond cache or main memory capacity.

## S3. PAX

The motivation for PAX is to keep the attribute values of each record on the same page as in NSM, while devising a more cache-friendly algorithm for data inside the page.

PAX vertically partitions the records within each page, grouping these values together as "minipages". When using PAX, each record resides on the same page as it would reside using NSM, but values of each attribute are grouped together. This requires a "minijoin" with records within the page, but no joins that span across pages.

## S3.2 Design

Each newly allocated PAX page contains a page header, and as many minipages as the degree of the relation. The PAX page header contains the number of attributes, the attribute sizes (for fixed length attributes), offsets to the beginning of the minipages, the current number of records on the page and the total space available on the page.

The structure of each PAX minipage is determined as follows:
  
  - Fixed-length attribute values are stored in F-minipages. At the end of each F-minipage there is a presence bit vector (bitmap) with one entry per record that denotes null values for nullable attributes.
  - Variable-length attribute values are stored in V-minipages. V-minipages are slotted, with pointers to the end of each value. Null values are denoted by null pointers.

## S4. Implementation and algorithms for data operations

### 1. Loads and Inserts:

PAX allocates pages in a similar way that NSM does. The difference is when PAX writes values within each minipage. When variable length values are present, minipage boundaries are adjusted to accommodate records. If a record fits in the page, but a particular attribute value does not fit in the current minipage, PAX requires a reorganization of the page structure by moving minipage boundaries.

At the end of writing a record, PAX calculates the position of each attribute value on the page, stores the value, and updates the presence bitmaps and offset arrays accordingly.

### 2. Updates:

When implemented in Shore, PAX pages can shrink the same way an NSM page would. In case of expanding a record, when a record grows bigger than its current page, the record is moved to another free page. PAX updates an attribute value by computing the offset of the attribute in the corresponding minipage, and updating the presence bitmaps.

### 3. Queries

- Scans

PAX invokes one scan operator per attribute involved in the request query. Each operator sequentially reads values from the corresponding minipage, using computed offsets.

- Joins

Joins in Shore receive input on top of two scan operators, each reading one relation.

## S5. Evaluation

The authors of this paper implemented NSM, DSM, and PAX on top of the Shore database. The authors selected a dataset of memory-resident, fixed-length numeric attributes.

The selection query were variations of this:

```
select avg(a_p)
from R
where a_q > Low AND a_q < High
```

It is important to point out that PAX targets **optimizing cache behavior, and not affect IO** - the unit of disk page does not change over NSM, so theoretical disk IO workload remains the same, and DSM still has an advantage over PAX when it comes to IO optimality over small number of columns.

When compared to NSM when each record is equal or greater than cache block size, NSM incurs a miss every record, but PAX incurs a miss every four records (using the author's CPU, 32 bytes per cache block, 8 bytes per attribute value). This translates to correspondingly higher execution performance.


