---
layout: post
title:  "Google Dremel - a Decade Later"
date:   2021-06-14
categories:
---

- Paper link: [https://research.google/pubs/pub49489/](https://research.google/pubs/pub49489/)

## Novel Idea:

Related to [Dremel 2010]({% post_url 2021-04-29-dremel-original %}), this paper examines what Dremel ideas withstood the test of time, and what ideas changed as Dremel evolved into BigQuery.


## Main Result(s):

Most of the original Dremel system design withstood the test of time: some inspired major industry trends and are now considered best practices. Here are the top 5:

1. SQL - as the de facto API and language of data retrieval and query.
2. Disaggregated compute and storage - decoupling storage from compute, allowing separate scaling of either resource, and separate optimization paths (see #5).
3. In situ data analysis - Dremel's use of a distributed file system to share data access allowed an interoperable ecosystem of services to flourish.
4. Serverless compute - elastic scale up and down of compute was at its infancy in 2010, but is pervasive today.
5. Columnar storage - Dremel introduced a novel encoding for nested data that generalized columnar physical layout to relational and semi-structured data.


## Prior Work:

Since the 2010 Dremel paper, multiple papers advanced the state of data analysis, as cited:

- [Storing and querying tree structured data in Dremel](https://research.google/pubs/pub43119/)
- [Reordering rows for better compression](https://arxiv.org/abs/1207.2189)
- [Riffle: shuffle service for large scale data analytics](https://collaborate.princeton.edu/en/publications/riffle-optimized-shuffle-service-for-large-scale-data-analytics)


## Content:


### S2. SQL

Dremel reintroduced SQL for data analysis at Google. Before, the conventional wisdom at Google was "SQL doesn't scale", and the tradeoff was either scalability, or ease of use. Dremel combined both.

Dremel initially avoided query-time joins, relying on Google's internal support for denormalized, but often hierarchical data - the nested structure of protobuf, together with Dremel's ability to process it in situ, fit Google's use case well.

> SQL finally became pervasive at Google, across widely used systems such as Dremel, F1, and Spanner, and other niche systems such as PowerDrill [24], Procella [15], and Tenzing [16].

SQL functionality in Dremel has been expanded in recent years, in particular with joins. Distributed joins across large datasets remain an active area of research. Dremel introduced a new shuffle join architecture that leverages the latest research, with Google's internal network optimizations.


### S3. Disaggregation of storage and compute

_Storage_

Dremel started on a cluster of special hardware. In 2009, Dremel started migration of compute towards Borg, the inception of storage disaggregation/decoupling, along with the move of storage to GFS. The move was challenging: 

> Harnessing query latency became an enduring challenge for Dremel engineers, which we cover in more detail in Section 7. It took a lot of fine-tuning of the storage format, metadata representation, query affinity, and prefetching to migrate Dremel to GFS. Eventually, Dremel on disaggregated storage outperformed the local-disk based system both in terms of latency and throughput for typical workloads.

_Memory_

Dremel had an initial implementation of distributed joins modeled after the MapReduce shuffle operation, spilling intermediate results from local RAM to local disk. However, as most everyone knows by now (2021), this was a bottleneck.

1. With such colocation, it is not possible to efficiently mitigate the quadratic scaling characteristics of shuffle operations as the number of data producers and consumers grew.
2. The coupling inherently led to resource fragmentation and stranding and provides poor isolation. This became a major bottleneck in scalability and multi-tenancy as the service usage increased.

> After exploring alternatives, including a dedicated shuffle service, in 2014 we finally settled on the shuffle infrastructure which supported completely in-memory query execution [4]. In the new shuffle implementation, RAM and disk resources needed to store intermediate shuffle data were managed separately in a distributed transient storage system.

The new shuffle implementation:

- Reduced the shuffle latency by an order of magnitude.
- Enabled an order of magnitude larger shuffles.
- Reduced the resource cost of the service by more than 20%.


### S4. In Situ Data Analysis

> In situ data processing refers to accessing data in its original place, without upfront data loading and transformation steps. In their prescient 2005 paper [22], Jim Gray et al. outlined a vision for scientific data management where a synthesis of databases and file systems enables searching petabyte-scale datasets within seconds. They saw a harbinger of this idea in the MapReduce approach pioneered by Google, and suggested that it would be generalized in the next decade.

The transition to in situ analytics required 3 ingredients:

1. Consuming data from a variety of data sources
2. Eliminating traditional ETL-based data ingestion from an OLTP system to a data warehouse
3. Enabling a variety of compute engines to operate on the data.

First, the data needed to be formatted in a way that enables interoperability between different systems. At Google, that format was Protobuf:

> A self-describing storage format in GFS enabled interoperation between custom data transformation tools and SQL-based analytics. MapReduce jobs could run on columnar data, write out columnar results, and those results could be immediately queried via Dremel.

Second, the ecosystem of different analytical tools allowed federation of queries (similar to Presto): 

> In some cases, including remote file systems such as Google Cloud Storage 11 and Google Drive, 12 we read the files directly. In other cases, including F1, MySQL, and BigTable, we read data through another engine’s query API. In addition to expanding the universe of joinable data, federation allows Dremel to take advantage of the unique strengths of these other systems.


### S5. Serverless Compute

Some key points in the dynamic utilization of compute resources in Dremel/BigQuery:

1. Centralized scheduling: more fine-grained, better isolation and utilization of resources.
2. New Shuffle Persistence Layer: decoupled schedule and execution of different stages of each query; at each checkpoint of a query, the scheduler can dynamically preempt workers.
3. Flexbile Execution DAG evolution: BigQuery implements a more flexible execution plan than that described in the original 2010 paper.
4. Dynamic execution plan: for queries on data where the cardinality estimates are wrong, Dremel/BigQuery allows the query plan to dynamically change during runtime, managed by the central query coordinator, and checkpointed by the shuffle persistence layer.


### S6. Columnar Nested Data

From a computer science perspective, the Dremel model of encoding data is perhaps most interesting.

Dremel influenced or inspired the encoding of nested data on a columnar layout. This was evident in the development of Parquet, ORC, and Apache Arrow. ORC takes a slightly different approach to encoding than Dremel/Parquet does, and depending on data pattern, the compression efficiency differs.

> The main design decision behind repetition and definition levels encoding was to encode all structure information within the column itself, so it can be accessed without reading ancestor fields. Indeed, the non-leaf nodes are not even explicitly stored. However, this scheme leads to redundant data storage, since each child repeats the same information about the structure of common ancestors. The deeper and wider the structure of the message, the more redundancy is introduced.

> With repetition/definition levels it is sufficient to only read the column being queried, as it has all required information. In 2014, we published efficient algorithms [3] for Compute, Filter and Aggregate that work with this encoding. With length/presence encoding, it is also necessary to read all ancestors of the column. This incurs additional I/O, and while ancestor columns usually are very small, they could require extra disk seeks.

The encoding and filtering/querying of columnar data has been embedded into a library called Capacitor. It makes the following optimizations accessible to not just Dremel, but other Google systems such as Spanner and MapReduce.

- Partition and predicate pruning: maintain statistics about values in each column.
- Vectorization.
- Skip-indexes: within a Capacitor block, column values are split into segments which the header points to; when querying with high selectivity, these indexes point directly to the value segments.
- Predicate reordering: Capacitor uses heuristics to run the most selective filters first, even if its evaluation could be more complex.

_Row Reordering_

Here's an optimization that was hard to imagine:

> RLE in particular is very sensitive to row ordering. Usually, row order in the table does not have significance, so Capacitor is free to permute rows to improve RLE effectiveness.

> Unfortunately, finding the optimal solution is an NP-complete problem [29].

> short RLE runs give more benefit for long strings than longer runs on small integer columns. Finally, we must take into account actual usage: some columns are more likely to be selected in queries than others, and some columns are more likely to be used as filters in WHERE clauses. Capacitor’s row reordering algorithm uses sampling and heuristics to build an approximate model.


