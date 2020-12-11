---
layout: post
title:  "Presto: SQL on Everything - reading notes"
date:   2020-07-23
categories:
---

## Summary

Company: Facebook

Paper link: [https://prestosql.io/Presto_SQL_on_Everything.pdf](https://prestosql.io/Presto_SQL_on_Everything.pdf)

Paper published: 2018

Presto is a flexible OLAP engine: it executes queries ranging from user-facing applications with sub-second latency requirements, to multi-hour ETL jobs that aggregate or join TBs of data. Presto's main innovation is its Connector API, which lends itself access to high performance data source backends (streams, KV stores, RDBMS, HDFS). So while some other open source OLAP engines focused on optimizing queries for a specific environment (Impala, Spark), Presto decoupled the query execution optimization from the storage IO optimization, which is generalized.

## Main Result(s):

Facebook realized in early 2010s that SQL is the most accessible query language, but many systems had multiple incompatible SQL-like support. Presto unifies the "frontend" by providing ANSI SQL interface to different backends, while defining a connector interface that allows developers to integrate Presto with different backends via RPC endpoints. Presto is an extensible and flexible system. Presto's performance depends heavily on the storage backend it is connected to: Facebook for example made internal storage optimizations (Raptor) for Presto. Today, Presto is widely adopted in many organizations.

## Impact:

Presto was originally designed to enable interactive query over Facebook's data warehouse. It is  powering most of the analytical workload (interactive + offline) at Facebook. It is also widely used in other companies in the industry. Presto is available as a managed service as AWS Athena.



## S2. Motivating Use Cases

A. Interactive analytics over Facebook data is defined as: 100GB - 1TB compressed, need query results within seconds to minutes. Most of the data sits on HDFS. 50-100 Concurrent users.
B. Batch ETL: scheduled by workflow management, tuned for efficiency by data engineers, usually CPU and memory intensive aggregations or joins with large tables.
C. AB Testing: expected results in hours, with accuracy in mind. Also, users expect to do further drill-downs, so pre-aggregation of results is not suitable. Multiple joins with large datasets (users, devices). Query shapes are programmatically defined, so this narrows down the scope.
D. Advertiser Analytics: user-facing application; queries are highly selective (for individual advertisers); query shapes contain joins and aggregations, or window functions. Replication lag expected in minutes, but query volume is high and latency requirements are strict.

## S3. Architecture Overview

Presto cluster consists of a **single coordinator** and one or more worker nodes. Clients send SQL statements to the coordinator over HTTP.

A **split** in Presto is an addressable chunk of data in some external storage system. The coordinator evaluates and distributes a query plan into **tasks** for the workers. Splits are assigned with the tasks. Worker nodes fetch data from the assigned splits. Presto workers execute tasks in a way that's familiar to distributed computers (Spark, GFS/MapReduce): they cooperatively multi-task, they pipeline results from one stage to the next. When data shuffles between nodes, Presto tunes its buffering to minimize latency.

Presto offers many high-level plugins, the most important of which is the Connector API, composed of: (1) Metadata API, Data Location API, Data Source API, and Data Sink API.

## S4. System Design

### A. SQL Dialect
Presto implements most of ANSI SQL, plus a few extensions (lambda support, transforms, filter, reduce) on embedded complex types (maps, arrays).


### B. Client Interface, Parsing, and Planning
The Presto coordinator listens on HTTP from clients, and also has a JDBC client that interfaces with BI tools.
Parsing: Presto uses ANTLR parser to convert SQL statements into AST. Supported functions include subqueries, aggregations, and window functions.
Logical Planning: the AST is converted to an intermediate representation (IR) that represents **plan nodes**, each node representing a physical or logical operation, and the children of that node are its inputs.


### C. Query Optimization
The plan optimizer converts a logical plan into a more physically executable structure. The process greedily evaluates transformations until a fixed point. Presto uses well-known optimizing transforms such as predicate and limit pushdown, column pruning, decorrelation. Presto already considers table and column statistics for join strategy selection and join re-ordering. Further work on cost-based optimizer is ongoing.

1. The optimizer can take account of physical data layout optimizations: the Connector API can report locations and other data properties such as partitioning, sort, grouping, and indexes. In fact, a single piece of data can be provided by multiple physical layouts.

2. Predicate pushdowns: Presto leverages the most aggressive data retrieval optimization available, pulling only data out of the necessary shard if it's provided, and using indexes whenever possible.

    > "For example, the Developer/Advertiser Analytics use case leverages a proprietary connector built on top of sharded MySQL. The connector divides data into shards that are stored in individual MySQL instances, and can push range or point predicates all the way down to individual shards, ensuring that only matching data is ever read from MySQL. If multiple layouts are present, the engine selects a layout that is indexed on the predicate columns. Efficient index based filtering is very important for the highly selective filters used in the Developer/Advertiser Analytics tools."

3. Inter-node parallelism: (resembles MapReduce/Spark) parts of a plan that can be executed across different nodes in parallel are called **stages**. They are distributed to maximize parallel IO. When the engine shuffles data between stages, the query uses more resources. Thus, there are important considerations to minimize shuffles.

  - Data Layout: when doing large joins in the AB Testing use case, Presto leverages the fact that both tables in the join are partitioned on the same column, so it uses a co-located join strategy to eliminate large shuffles across nodes. Presto can also make use of indexes as nested loop joins.
  - Node properties: nodes can express *required* and *preferred* properties while participating in a shuffle.

4. Intra-node parallelism: when appropriate, Presto exploits single node multi-thread execution, for example hash joins. This also combats natural partition-skew, to a certain degree (Spark for example does not have an automatic mechanism to do the same).


### D. Scheduling

Presto internalizes the assignment, execution, and management of tasks in its cluster.
The first major theme of Presto scheduling is in its considerations for performance: maximizing parallelism, pipelining, buffering, minimizing shuffles.

Some definitions:
- A coordinator distributes plan **stages** to workers.
- A stage consists of **tasks**, where a task represents a single unit of processing.
- A task contains multiple **pipelines** within it.
- A pipeline is a chain of **operators** on the data, and the optimizer can determine the level of **local parallelism** in a pipeline. Pipelines are linked with in-memory shuffles.
- The coordinator links stages together with **shuffles**.
- Data streams stage to stage as soon as it's available.

Schedule decisions are made in two parts: (1) a logical ordering of what stages to schedule, (2) determine how many tasks should be scheduled and the physical nodes they run on.

1. Stage scheduling: Presto has two modes: *all-at-once* or *phased*. All-at-once starts all stages concurrently to minimize wallclock time. Phased mode runs an algorithm to find **strongly connected components** within the directed graph of data flow, which outputs a modified DAG that avoids deadlocks, and then execution may begin in topological order for maximum efficiency.

2. Task scheduling: Leaf Stages and Intermediate Stages.
- Leaf stage - these read data from connectors (in the form of *splits*). In shared-nothing configurations (Facebook Raptor), the scheduler may assign a leaf node to every worker node, to maximize IO parallelism. This exerts pressure on network bandwidth. Presto plugins at Facebook allows a mechanism to express a preference for rack-local versus rack-remote reads.
- Intermediate stage - these only read data from other stages.


3. Split scheduling:

    >"When reading from a distributed file system, a split might consist of a file path and offsets to a region of the file. For Redis, a split consists of table information, a key and value format, and a list of hosts to query"

Every task in a leaf stage must be assigned one or more splits to become eligible to run. Presto asks connectors to enumerate small batches of splits, and assigns them to tasks lazily. This minimizes time-to-first-result lag, decouples time-to-enumerate splits from query response time, allowing each worker to keep a smaller inflight queue.


### E. Query Execution

The core query execution parts of Presto takes inspiration from DB literature.

1. Local Data Flow:

A worker thread does compute on a data split within a driver loop. The Presto driver loop cites the original Volcano paper, with variations: Presto's loop is more friendly to yielding, aims for maximizing work in very **quanta**; also, the unit of in-memory data the driver loop operates on, also called a *page* in traditional DB literature, is columnar encoded. This is a difference from Impala (2013).

2. Shuffles:

When transporting data across nodes, workers use a long-pull, in-memory mechanism. This is unlike Spark.

Something interesting Presto does is using a continuous monitor-and-adjust feedback loop to tune the input/output buffer counts and depths, for optimal utilization and minimal latency: when overall utilization is high, it can tune down the concurrency level, increasing fairness, protecting against slow clients unable to consume at the output rate.

3. Writes:

Presto tries to strike a sweet spot between output write concurrency versus metadata overhead, using the buffer utilization threshold.


### F. Resource Management

Being its own distributed system, Presto exercises some freedom in the way it manages CPU and memory, taking lessons learned from other distributed systems (HDFS, Spark).

1. CPU scheduling:

Presto splits are allowed to run on a thread for a maximum *quanta* of one second, after which it yields the thread and returns to the queue. This is an effective technique to accommodate slow and fast queries on the same cluster. While it is always challenging to accomplish fair scheduling in practice with arbitrary workloads, the Presto scheduler is effective - it penalizes long running tasks.

2. Memory management:

- Memory Pools: Presto splits memory allocation into *user* and *system* pools. System memory allocation are side-effects of Presto implementation, e.g. shuffle buffers. User memory allocation are directly related to input/query. Presto kills queries exceeding memory limits.

To handle data skew and the skew of per-node memory usage (for example one data partition consumes 2x median memory), and allowing multiple concurrent queries to run within some memory overcommit capabilities, Presto uses two techniques - spilling and reserve pools.

- Spilling: Presto supports spilling to disk for hash joins and aggregations, but Facebook disables spilling for the sake of predictable latency.

- Reserved Pool: When the general memory pool (system + user) of a node is exhausted, the query is "promoted" to use the Reserved Pool of memory across all worker nodes. To prevent deadlocks, only a single query can enter the Reserved Pool across the entire cluster. Obviously, if a node has exhausted its general memory pool AND the Reserved Pool is occupied by some query, all other queries will stall on that node. This is an effective technique to combat skew stragglers spilling to disk, and the trade-offs are clear.

### G. Fault Tolerance

Presto relies on clients to retry failed queries. Presto coordinator is a singleton, but Facebook externally coordinates failover to standby coordinators (Zookeeper or Helix will do fine here).

## S5. Query Processing Optimizations

Presto made an intense effort to optimize query processing.

### A. Working with the JVM

Presto optimizes within the JVM: we know the G1 collector does not work well with large objects. Presto data structures on the critical path of query execution are implemented as flat arrays instead of objects and references.

  >"For example, the HISTOGRAM aggregation stores the bucket keys and counts for all groups in a set of flat arrays and hash tables instead of maintaining independent objects for each histogram."

### B. Code Generation

  1. Presto uses generated bytecode to evaluate constants, function calls, and references when running over data.
  
  2. Instead of a generic processing loop, Presto uses the semantics of its query execution:
  - It prevents cross-pollution of JIT compile due to queries being switched across quantas.
  - Because the engine is aware of the types involved in each computation within the processing loop of every single task pipeline, it can generate unrolled loops over columns with elimination of target type variance, making the JIT compiler job easier.
  - JIT compiler profiling of every task is independent

### C. File Format Features

Presto makes use of well-known optimizations in row-column optimized data formats, and extends those optimizations to how it processes data while in memory.


## Competitive work:

Leading open source competitors of Presto are: Hive (written by Facebook as a SQL interface over HDFS, generating MapReduce jobs), Spark, and Impala. Spark and Impala both integrate more tightly with their respective storage backends, whereas Presto was designed with backend adaptability in mind.

