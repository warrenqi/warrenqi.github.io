---
layout: post
title:  "Highly Available Transactions: Virtues and Limitations paper overview"
date:   2021-12-12
categories:
---

- Paper link: [https://www.vldb.org/pvldb/vol7/p181-bailis.pdf](https://www.vldb.org/pvldb/vol7/p181-bailis.pdf)
- Related to [Google Spanner]({% post_url 2021-08-20-spanner-2012 %}) and [Amazon Aurora]({% post_url 2020-09-04-aws-aurora %})

# Commentary:

There are interesting analogies between distributed systems and memory models of multi-threaded programming languages, in the context of modeling system/program behavior when there are concurrent modifications to data.

This paper focuses on **isolation levels**, which defines how a (DB) system behaves when multiple clients modify the same piece of data. The highest isolation level - Serializable - dictates that a system should execute concurrent client-issued transactions "as if" the clients executed them in some serial order, one at a time. [Wikipedia on Isolation](https://en.m.wikipedia.org/wiki/Isolation_(database_systems))

Isolation guarantees are different from **consistency levels**, which, in the context of distributed systems, mostly apply to how replicas/nodes in a (DB) system converge to a consistent state or "consensus", and the order/sequence that these replicas converge in. [Wikipedia on Consistency Models](https://en.m.wikipedia.org/wiki/Consistency_model)


# Summary of isolation levels

Taken from the paper with extra remarks:

```
Highly Available:
    Read Uncommitted (RU)
    Read Committed (RC)
    Monotonic Atomic View (MAV)
    Item Cut (ICI) and Predicate Cut Isolation (PCI) -- not SI
    Monotonic Reads (MR) (a consistency level)
    Monotonic Writes (MW) (a consistency level)
    Writes Follow Reads (WFR) -- Lamport's "Happens-before" order

Sticky:
    Read Your Writes (RYW) (a consistency level)
    Pipelined RAM (PRAM)
    Causal (a consistency level)

Unavailable:
    Cursor Stability (CS)
    Snapshot Isolation (SI)
    Repeatable Read (RR)
    Recency
    Linearizability (a consistency level)
    One-Copy Serializability (1SR)
    Strong Serializability

```

## Read Committed

While most existing databases implement extended versions of Read Committed isolation to include guarantees like Monotonicity and Recency, the agnostic definition of RC is:

> Transactions should not access uncommitted or intermediate versions of data items. The condition known as "Dirty Writes" is prohibited.

There is a begin edge at the start of new writes, and an end edge when new writes are fully committed. RC dictates that no observers can observe an intermediate state between these two edges.

To implement RC in a HAT system, 2 options:

1. Clients can send writes to servers, who will not deliver new values to other readers until notified that the writes have committed.
2. Clients can buffer writes to servers, until writes are fully committed.

These types of implementations do not guarantee Recency. See [Granularity of Locks paper](https://web.stanford.edu/class/cs245/readings/granularity-of-locks.pdf)

## Writes Follow Reads

This is a property describing how someone would observe the ordering of events. It's equivalent to Lamport's "Happens-before" ordering: if `T1` occurs before `T2`, and if another observer/session observes `T2`, it must also observe the effects of `T1`.

[Lamport's definition](https://en.m.wikipedia.org/wiki/Happened-before)

Implementation: WFR ordering effectives dictates that upon revelation of a new value/write, the sequence of writes leading up to this write are also revealed.

> Force servers to wait to reveal new writes (say, by buffering them in separate local storage) until each write's respective dependencies are visible on all replicas. This mechanism effectively ensures that all clients read from a globally agreed upon lower-bound on the versions written.

This does not imply that transactions will read their own writes: in HA scenario (non-sticky), a client may have to switch servers, and issue its next requests against a partitioned, out-of-date server.


## Monotonic Atomic Reads

This property describes the _isolation effects_ of **atomicity**, but critically, adds monotonicity properties:

> Under MAV, once some of the effects of a transaction `Tx` are observed by another transaction `Ty`, then all other effects of `Tx` must be observed by `Ty`

This never sounded so different from Read Committed. MAV is described as stronger than RC but in practical terms the effects of an RC versus MAV transaction are very similar: 

> MAV disallows reading intermediate writes: observing all effects of a transaction implicitly requires observing the final (committed) effects of the transaction as well.

Implementation: the paper's sketched implementation of MAV describes it adds monotonicity properties on top of RC, in a distributed system with sharded replicas.

1. Replicas store all versions ever written to each data item. They gossip about versions they have observed, and construct a lower bound on the versions updated at every replica.
    * This can be represented by a vector clock
2. At the start of a transaction, clients choose a `read_timestamp` smaller or equal to `global_lower_bound`.
3. During transaction execution, replicas return/work on the latest version of each data item that is `<= read_timestamp`.
4. The DB system advances the `global_lower_bound` along transactional boundaries.

