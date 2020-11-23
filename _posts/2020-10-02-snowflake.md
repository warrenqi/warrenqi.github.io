---
layout: post
title:  "Snowflake Data Warehouse reading notes"
date:   2020-10-02
categories:
---


**Snowflake Data Warehouse Reading Notes**
============================================


## Summary

Snowflake (2016) is a cloud-native data warehouse that's optimized for large analytical workloads, and provides ACID transactions, over structured and semi-structured data. One key innovation Snowflake offers is what it calls its "multi-cluster, shared data" architecture which improves upon the shared-nothing architecture. Within this blueprint, this paper talks about the three major components: (1) the data storage, (2) the elastic Virtual Warehouse, (3) the central coordination and management system called Cloud Services.

## Impact:

Perhaps the most interesting impact is Snowflake, the product, has matured into a complete company providing data warehouse Platform as a Service (Paas). Snowflake Inc has a market cap of $80 billion today.

In more concrete terms, one value that Snowflake provides is a warehouse platform that's not strictly tied to one of the three major cloud providers (AWS, Google, Microsoft). Meanwhile, Snowflake is also more feature complete, and has more security capabilities built-in, compared to some of the more obvious open source systems like Impala and Presto.

Some high level features of Snowflake advertised: elastic scaling and high availability, accommodates structured and semi-structured/schema-less data, point-in-time travel, and end-to-end security.


## S1. Introduction

Cloud providers such as Amazon, Google and Microsoft are now providing platform-as-a-service; the key value they provide are: shared infrastructure, economies of scale, flexiblity, and capability. But traditional data warehouse software predates the cloud - they were designed with small physical clusters of machines in mind, poorly fitting modern cloud platforms.

Data has changed as well. Today's data are increasing sourced from different origins, increasingly without schema. Traditional data warehousing systems struggle here.

Some open source systems aim to tackle these issues, but they are difficult to deploy without significant engineering effort. Snowflake aims to **capture this market share, where a business workload can benefit from the cloud infrastructure, but are not served by open source systems.**

>"Snowflake is not based on Hadoop, PostgreSQL or the like. The processing engine and most of the other parts have been developed from scratch."

### Snowflake differentiates itself by the following

1. Software-as-a-service deploy and run
2. Relational ANSI SQL support and ACID transactions
3. Support for semi-structured data in JSON and Avro
4. Elastic scale up and down
5. High availability and durability
6. Cost efficient (same as 4)
7. Fine grained access control


## S2. Storage vs. Compute

First we critique the popular layout for high performance data warehouse systems: _shared-nothing, where every node has the same responsibilities, over mostly uniform, commodity hardware._ A pure shared-nothing architecture couples the compute resources with storage resources.

1. While the hardware is uniform, the workload is not. A system configured for high IO is poorly fit for complex queries that demand higher compute, and vice versa. This leads to inefficency.
2. Membership changes causes large data movement, usually leading to significant performance impact, limiting elasticity and availability.
3. Expanding on 2, online upgrades are harder to roll out when every node is impacted.

These points are tolerable in an onprem system, as is the case with Presto. But in the cloud, node failures are more frequent and performance can vary more significantly. And as a product offering, Snowflake needed to bring online upgrades as a capabiilty. Thus, Snowflake separates storage and compute with two loosely coupled, independently scalable sub-systems: **Snowflake has a custom built, caching, shared-nothing Compute engine, and Storage is outsourced to Amazon S3.**

## S3. Architecture

There are 3 major components:
- Storage
- Compute, called "Virtual Warehouses"
- Coordination/management

### Storage

Snowflake chose to use out-of-the-box AWS S3 storage, and understand its tradeoffs, rather than implementing its own HDFS-style of storage.

AWS S3 had the following drawbacks:
- Higher latency compared to local storage
- Higher CPU overhead corresponding to every IO request
- Objects could only be overwritten in full, not appendable

And the following benefits:
- Supports GET requests for ranges of a file
- High availability and durability
- Effectively infinite space for spilling temporary data

Snowflake implemented the following to counterbalance the tradeoffs: (1) local caching, (2) latency skew resilience logic in the compute layer, and (3) a PAX-style physical storage format for the data in each table. And (4), perhaps worthy of emphasis for paging:

>"Storing query results in S3 enables new forms of client interactions and simplifies query processing, since it removes the need for server-side cursors found in traditional database systems."

### Virtual Warehouses (Compute Units)

The Snowflake equivalent of a compute cluster is the Virtual Warehouse. VWs are effectively state-less, dynamically scalable. Each individual query runs on exactly one VW. Long running queries are retried from the beginning, without checkpointing (similar to early Google F1 descriptions).

Each worker node in a VW holds an LRU cache of table files, and the PAX philosophy is passed here:

>"the cache holds file headers and individual columns of files, since queries download only the columns they need."

Here's an interesting insight: Snowflake query optimizer/coordinator "remembers" a mapping of worker node to data they've recently cached by consistent hashring.

>"To improve the hit rate and avoid redundant caching of individual table files across worker nodes of a VW, the query optimizer assigns input file sets to worker nodes using consistent hashing over table file names [31]. Subsequent or concurrent queries accessing the same table file will therefore do this on the same worker node."

This technique allows Snowflake to avoid eagerly replacing cache contents when a node fails, or the VW resizes - the underlying node LRU cache will eventually correct itself. This simplifies the system.

When one Snowflake VW worker finishes its work early, it can _file steal_ additional work from peer workers for the duration and scope of a current query. This helps alleviate straggler nodes.


### Execution Engine

Snowflake has its own, custom built SQL execution engine. The highlights are:

1. Columnar - integrated and utilizes the optimality of underlying physical data layout
2. Vectorized - Snowflake avoids intermediate results, and instead processes data in a pipeline fashion, in batches of a few thousand rows.
3. Push-based - avoids control flow logic within tight loops for better cache efficiency

Snowflake makes use of several additional optimizations outside its execution engine: there no need for a buffer pool - Snowflake simply spills out to "storage" to accommodate large workloads. This trades off pure speed for more capability, and relies on the (outsourced) storage layer to speed up over time.

Snowflake query execution, with its underlying assumption of S3, can consider the dataset to be a fixed set of immutable files.


### Cloud Services (Coordinator)

The Cloud Services component emcompasses the query optimizer, transaction manager, and access-control mechanisms of Snowflake. This component is replicated to tolerate transient failures.

- Query Management and Optimization

CS governs the early stages of each query: parse, resolving objects/file handles, access control, and plan optimization. The optimizer is based on Cascades, without indexes, and automatic statistics building. The optimizer output is distributed to all worker nodes assigned to the query. Cloud Services monitors and tracks the state of the query to collect performance counters, and retry in case of node failures.

- Concurrency Control

Snowflake is designed for analytical workloads first and foremost: large reads, bulk or trickle inserts, and bulk updates. Snowflake implements ACID transactions via Snapshot Isolation level of consistency.

Snapshot isolation built on top of MVCC is a natural choice for Snowflake, almost a direct consequence of using S3 immutable storage. **Snowflake keeps track of file changes in a global key-value metadata store.**

- Pruning

Due to Snowflake's storage choice, and the target market use pattern, Snowflake forgoes maintaining indices. For Snowflake, indices are too costly, and the requirement of random access is too big of a problem to solve.

Alternatively, **Snowflake uses min-max pruning**, aka small materialized aggregates, zone maps, and data skipping.

>"Here, the system maintains the data distribution information for a given chunk of data, in particular minimum and maximum values within the chunk. Depending on the query predicates, these values can be used to determine that a given chunk of data might not be needed for a given query. "

This technique works well for sequential access across large spans of data, and it does not involve overhead of updating indices when bulk loading.


## S4. Extra Features

One section to highlight in exclusive Snowflake features is the ability to optimize data layout for schema-less serialized data: as opposed to Apache Impala and Google Dremel (BigQuery), Snowflake aims to achieve both the flexibility of a schema-less database and the performance of a columnar relational DB. It does so by performing statistical analysis of the data within a single table file (stored PAX style). When Snowflake sees a frequently accessed path within a file, it tries to infer its type, and retrofit it into the optimized compressed columnar format like a native relational data. This also works with the min-max pruning.






