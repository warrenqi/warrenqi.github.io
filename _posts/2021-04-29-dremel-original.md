---
layout: post
title:  "Google Dremel/BigQuery - reading notes"
date:   2021-04-29
categories:
---

### Commentary:
Dremel is Google's innovative approach to query massive amounts of data, utilizing thousands of machines, with an interactive latency that was previously not possible with MapReduce. The Dremel/BigQuery style of querying distributed data in-situ became a cited influence for many other distributed systems like [Snowflake]({% post_url 2020-10-02-snowflake %}). The Protobuf record description, and its record shredding/striping and reassembly algorithm, went on to influence open source implementations like Thrift, Avro, and Parquet.

- Paper link: [https://research.google/pubs/pub36632/](https://research.google/pubs/pub36632/)
- Related algorithm description from Parquet authors: [https://github.com/julienledem/redelm/wiki/The-striping-and-assembly-algorithms-from-the-Dremel-paper](https://github.com/julienledem/redelm/wiki/The-striping-and-assembly-algorithms-from-the-Dremel-paper)
- Also: [https://blog.twitter.com/engineering/en_us/a/2013/dremel-made-simple-with-parquet.html](https://blog.twitter.com/engineering/en_us/a/2013/dremel-made-simple-with-parquet.html)
- Date: 2006-2009

### Novel Idea:
Before Dremel, analytical queries running on large volumes of archival data typically relied on proprietary databases, or recently, MapReduce. Dremel is a new system that introduces a novel approach to query data in-situ, at interactive response times (a few seconds), by doing the following:

1. formatting the data in an optimized physical layout,
2. distributing the query processing across thousands of machines,
3. outsourcing the data storage to a distributed file system (GFS).

### Main Result(s):
Dremel is able to query and analyze petabytes of data, and return calculated results within a few seconds, in 2006 - at Google's scale. These were, and continue to be, industry-leading results.

### Prior Work:
BigQuery is influenced by a variety of prior research work in databases. However, many of those were non-public or not practically implemented.

The paper cites several sources as inspiration for the record shredding algorith. I also see influences of [PAX (Partition Attributes Across)]({% post_url 2020-11-06-pax-weaving-data %}) as well - the idea to strike the optimal balance between row and columnar storage is not new to the year 2000.


### Legacy:
It is easy to see the influence Dremel/BigQuery has on later systems. The most obvious is Snowflake, which directly cites Dremel. Similar systems on the Hadoop ecosystem include Apache Impala. Apache Parquet is an open source implementation of the physical data storage specification in Dremel. Apache Arrow is a continuation of that.

## Content:

## S2-3. Background and Data Model
The main problem Dremel tried to solve was interactive analytical querying of data at Google. The primary need was to get access to results quickly, ideally within a few seconds, while the raw data can span billions/trillions of rows.

The two main "ingredients" to make this happen are

1. A common storage layer of physical data, on a distributed file system, that is high performance, resilient to failures, that enables quick access to the subset of data relevant for processing, and at the same time, separated from the processing layer.
2. A shared storage format - consisting of a data model descriptor + serializer, namely Protobuf, and a physical data storage format, that is optimal for the nested Protobuf data structure.

Item 1 has been described in the GFS paper. Item 2 is the focus of Section 4.

## S4. Nested Columnar Storage

Records can be wide, but not every field is necessary for query processing:

> All values of a nested field, such as `A.B.C`, are stored continguously. `A.B.C` can be retrieved without reading `A.E`, `A.B.D`, etc.

In Protobuf, field values can be nested, optional, and repeated.

> "The challenge is how to preserve all structural information and be able to reconstruct records from an arbitrary subset of fields"

The idea is to compute and store two additional integer values to every field value, as the records are written.

1. Repetition Level

    - This is defined as "at what repeated field in the field path this value has repeated."
    - In other words: Repetition Level starts with 0 at the beginning of a new record, at the record root level. As we traverse the record via DFS, jot down this node's depth level by examining how many REPEATED fields exist to the root - call it rawRepLevel. If this node is a new occurence, use its parent's Repetition Level; otherwise, this node is repeating, thus, use the rawRepLevel.


2. Definition Level

    - This is defined as "how many fields in p that could be undefined are actually present in the record"
    - In other words: As we traverse the record by DFS, Definition Level specifies the lowest level, in the field's path, at which the value is still defined/not-NULL.

Here is a version of Java psuedocode. Also listed: [toy implementation](https://github.com/warrenqi/lucene-lyrics-www/blob/master/src/main/java/com/distraction/dremel/RecordStripe.java)

```
public void traverse(
    int curDef,
    int nonNullDef,
    int parentRep,
    int parentDisplayRep,
    Set<Field> seen) {

        int repLevel = this.type == REPEAT ? parentRep + 1 : parentRep;
        int repLvlDisplay = seen.contains(this.fieldpath) ? repLevel : parentDisplayRep;
        seen.add(this.fieldpath);

        if (isParentNode(this)) {
            Set<> nextLevelSeen = new Set<>(seen);
            int nextDefLevel = childrenAreNull(this) ? curDef : 1 + curDef;

            if (childrenAreNulls(this)) {
                for (child : this.children) {
                    child.traverse(
                        1 + curDef,
                        nextDefLevel,
                        repLevel,
                        repLvlDisplay,
                        nextLevelSeen
                    );
                }
            }
        } else {
            // a value node
            int defLevel = emptyValue(this) ? nonNullDef - 1 : curDef;

            writers.write(this.fieldpath, this.value, repLvlDisplay, defLevel);
        }
    }

```

Record reassembly is done via the FSM described in the paper. Apache Parquet has an open source implementation of effectively the same FSM.

## S5-6. Query Language and Execution
There are several advantages of converting the records into blocks of columnar physical format:

1. At the macro level: separate fields can be stored and processed on separate machines.
2. At the single machine level: sequential reads and writes.
3. At the file level: more efficient encoding/decoding of values; also, better compression and vectorization - values are always of the same type within a block.
4. Implicit skipping of empty values/NULLs.

As for Query Language, Dremel started with a derivative/subset of SQL, but as it evolved into BigQuery, it also added more and more features of standard SQL. This is motivated by wide adoption of SQL in industry.

### Tree Query Execution Architecture

Dremel fans out a query from a root node to leaf nodes. Non-leaf nodes compute the boundaries of where pieces of data are distributed, and splits the query into paritioned queries. Leaf nodes work on the actual data scans. Then, as the leaf nodes return sub-results back up the tree, non-leaf nodes aggregate the results.

There are several advantages of doing this:

1. Parallelization: effective partitioning of compute to where the data lives.
2. Connection pooling: any parent node can maintain connection to a pooled set of nodes, and optimize against connection churn.
3. Resilience against failure: at the largest distribution leaf nodes, any transient or permanent failure can be quickly failed over to replicas.


## Ideas for further work:
Google BigQuery team wrote a follow-up paper, called [Dremel - A Decade Later](https://research.google/pubs/pub49489/)
