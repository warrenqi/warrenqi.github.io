---
layout: post
title:  "AWS Aurora reading notes"
date:   2020-09-04
categories:
---

Paper link:[https://www.allthingsdistributed.com/files/p1041-verbitski.pdf](https://www.allthingsdistributed.com/files/p1041-verbitski.pdf)

# Summary

Aurora's goal is to increase throughput when moving MySQL to a cloud native environment. AWS engineers revised and revamped the storage and network layers of MySQL, to better utilize the characteristics of a distributed file system and AWS infra/layout.


# Main Result(s):

Aurora keeps the "front-end" of a single master MySQL, but with the new storage and network backend, system throughput is significantly improved. Replication delay is very low (in milliseconds); Read throughput scales horizontally up to the maximum limit of 15 read replicas. Write throughput, however, is limited to single master vertical scaling. The limit of total data stored is 64TB (as of 2020).

# Impact:

The most significant impact Aurora brings is increased throughput over a default MySQL installation in AWS cloud: lower replication lag, less dilated latency distribution, higher read-scale-out throughput, shorter time to recover master.

The key insight is to keep Aurora single master, and re-implement the read replicas around an ever-forward-rolling redo log. This sidesteps the complex problem of distributed consensus in other distributed databases (e.g. Spanner).


## S1. Introduction

1. MySQL is built with local disks in mind. This assumption does not hold well when operating MySQL in the cloud, where the storage may be mounted further away from the compute. When this happens, latency and performance suffers.

2. In AWS, we have a true distributed file system, that abstracts away the failures of individual disks. This, combined with other key characteristics of AWS foundational pieces, makes good motivation to rethink the way a cloud installation of MySQL should operate.

3. Distributed synchronization protocols (like 2PC) are difficult to make performant in large distributed environments. This is a motivating factor to "eliminate multi-phase synchronization" by pushing the limits of a single master writer.


## S2. Replication to avoid correlated failures

1. Aurora aims to use a quorum-based design, but a typical 2/3 quorum is inadequate for AWS scale. It is easy to see why: Assume DataCenters A, B, and C. If DC C fails, we will break quorum with any concurrent failure in DC A or DC B, due to the simplest scenarios like maintenance - we would lose 2/3 copies. Aurora needs to tolerate a DC failure as well as "concurrently occurring background noise failures".

2. The chose quorum design is
    - writes can continue after one entire AZ failed, but no more.
    - reads can continue after one entire AZ failed plus one additional node, but no more.
    - total quorum votes is 6: so writes require 4/6 and reads require 3/6.

3. The breaking up of storage into 10GB segments and distributing them into replicas across AWS makes this act like partitioning the data on top of a distributed file system:

>"We instead focus on reducing MTTR to shrink the window of vulnerability to a double fault. We do so by partitioning the database volume into small fixed size segments, currently 10GB in size. These are each replicated 6 ways into Protection Groups (PGs) so that each PG consists of six 10GB segments, organized across three AZs, with two segments in each AZ. A storage volume is a concatenated set of PGs, physically implemented using a large fleet of storage nodes that are provisioned as virtual hosts with attached SSDs using Amazon Elastic Compute Cloud (EC2). The PGs that constitute a volume are allocated as the volume grows. We currently support volumes that can grow up to 64 TB on an unreplicated basis."

Is the slightly lower quorum requirement for read to help make recovery faster? It would seem so.


## S3. Porting the extraneous IO to be cloud-native


1. MySQL issues multiple concurrent local disk writes, before it issues these multiple writes across the network to a read replica, which then issues its own set of multiple local writes.

2. These include: the redo log (the diff between before-page and after-page), the statement binlog, the modified data pages, a second temp write (called a double-write, a good description here [https://www.percona.com/blog/2006/08/04/innodb-double-write/](https://www.percona.com/blog/2006/08/04/innodb-double-write/) and here [http://enjoydatabases.blogspot.com/2016/08/innodb-doublewrite-buffer.html](http://enjoydatabases.blogspot.com/2016/08/innodb-doublewrite-buffer.html) ) to prevent torn data pages, and the metadata files.

3. The traditional MySQL replication, when operated in a distributed environment, is akin to a 4/4 quorum.

4. With Aurora, only the redo log is replicated across to read replicas, and the distribution is done using the semantics of a distributed and highly available file system, S3. When 4/6 replicas acknowledge, the primary can consider the log records durable.

5. Read replicas can asynchronously and continuously apply the redo log on top of their buffer caches to materialize the updated records.

## S4. Read replicas asynchronously process the primary's redo log

1. This paper only provides a sketch of the underlying implementation.
2. The master can assign a monotonically increasing ID - the Log Sequence Number - to each log record.

>"The logic for tracking partially completed transactions and undoing them is kept in the database engine, just as if it were writing to simple disks."

3. We don't need a distributed consensus protocol. This is not a Raft log.

4. Read Replicas can gossip with each other to fill in gaps up to the maximum LSN.
  - The storage layer determines the max LSN that guarantees availability of all prior log records (Volume Complete LSN, or VCL). During storage recovery, every log record with an LSN larger than the VCL must be truncated.
  - This must be further truncated by the database's durability criterium:

>"The database can, however, further constrain a subset of points that are allowable for truncation by tagging log records and identifying them as CPLs or Consistency Point LSNs. We therefore define VDL or the Volume Durable LSN as the highest CPL that is smaller than or equal to VCL and truncate all log records with LSN greater than the VDL. For example, even if we have the complete data up to LSN 1007, the database may have declared that only 900, 1000, and 1100 are CPLs, in which case, we must truncate at 1000. We are complete to 1007, but only durable to 1000."

  - Reads can be made consistent by finding the Minimum Read Point LSN across all nodes. Each node in the cluster can compute this.

# Follow ups:

There isn't a clear description of the dual-master Aurora yet. Is it implemented as a hot spare? Or does it use a synchronization mechanism like 2PC or just sync replication between the two master copies? Would be interesting to find out if the synchronization incurs performance penalty in trade of higher availability.

