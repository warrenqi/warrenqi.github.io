---
layout: post
title:  "Paper: The Tail at Scale"
date:   2020-03-28
categories:
---

Original paper [https://research.google/pubs/pub40801/](https://research.google/pubs/pub40801/)

## Summary

This was a very practical paper for me because the techniques were directly applicable to my work. I work on a sharded, distributed search/retrieval system at Booking. I have blended in my notes and some applicable details.

## Preface

A "coordinator" service that fans out calls to N number of "data" nodes will have to tolerate high tail latency. To counter this effect we can use redundancy: at a high level, by making redundant calls to replicas of the same data node, obtaining the first/fastest response, and discarding/ignoring the slower replicas.

Scenario:
A root service makes calls to 21 data nodes. Suppose each data node has 99 percentile latency of 10ms.
The probability of all 21 calls responding within 10ms is:

    (0.99)^21 = 0.81


## Techniques for minimizing tail latency

1. Keep low level queues short, so higher level policies take effect more quickly. For example: Google File System keep few operations outstanding in the OS disks queue. Instead, this service maintains its own priority queue of pending disk requests. This technique allows servers to serve interactive requests before older requests for batch operations.

2. Reduce Head Of Line (HOL) blocking. For example: Google Web search uses time-slicing to allow interleaved execution of short running requests. See HTTP/2. Prevent a single large expensive query from adding latency to many smaller/cheaper queries.

3. Manage background activities, synchronize/orchestrate disruption. For example: compaction in log-oriented storage, garbage collection. Explore this alternative: schedule and synchronize GC pauses at minimal load times to minimize tail latency during high load.

4. Caching: it does not directly address tail latency -- unless entire dataset fits in cache.


## Embracing tail latency

1. Hedged requests. Issue the same request to multiple replicas, use the fastest response; discard, or cancel the remaining outstanding requests. Naive implementations of this method yields unacceptable overhead. Further refinement usually reduces tail latency effects, with minimal overhead.

A typical approach is to defer sending a second request until waiting a period of time == the 95th percentile latency. This limits overhead to approximately 5%.

A Google experiment: 100 node fanout, 10ms delay hedged requests, 999 percentile reduced from 1800ms to 74ms, with just 2% increase in overhead requests.


2. Tied requests. This is an attempt to further improve (1), by observing that: to permit more aggresive use of hedged requests, without large resource consumption, we require faster cancellation of requests.

Examine a server queue before execution. Mitzenmacher: "allowing a client to choose between 2 servers based on queue depths at enqueue time -- exponentially improves load balancing performance over uniform random."

Tied requests: servers perform cross-server status updates, peer-to-peer. A coordinator sends the same request to 2 servers, each tagged with the identity of the other server. When a servers starts executing a request, it sends a cancellation message to its "tied" peer. The other server can either immediately abort, or deprioritize the queued request.

It is useful for the client to introduce a small delay of 2x the average network message delay (approx 1ms) between sending the 1st and 2nd requests.

Google's implementation: serving requests that is strictly not cached in memory & must be read from disk. Replication factor is 3. The results are effective in both an isolated environment and a busy environment.


3. Alternative tied-request: probe remote queues first, then submit to least loaded server -- this approach can be less effective than simultaneous tied-requests because: (A) load changes between probe and request; (B) request service times are difficult to estimate; (C) clients can create temporary hot spots when all clients pick the same server.


## Operational notes on production

1. Canary requests: prevent a bad request from nuking all nodes in a fanout. Google IR systems employ "canary requests": a root server sends a request to one or two leaf servers, and only query remaining nodes if canary responds within resonable time. Otherwise, the system flags the request as dangerous, and prevents further execution. This guards against errors, and DOS attacks.

2. Mutations: for services that require consistent updates, the most commonly used techniques are quorum-based: they write to multiple replicas; they inherently help tail-tolerance.