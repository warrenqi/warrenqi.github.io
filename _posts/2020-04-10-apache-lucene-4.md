---
layout: post
title:  "Paper: Apache Lucene 4.0"
date:   2020-04-10
categories:
---

Original paper [http://opensearchlab.otago.ac.nz/paper_10.pdf](http://opensearchlab.otago.ac.nz/paper_10.pdf)

## Summary

This is an overview of Lucene and specifically the updates to version 4. In a way, this paper distills Lucene's rich feature set into just a handful of pages. A sort of index of index. Lucene version 4 has significant improvements that I find directly applicable to my system at work. I work on a sharded, distributed search/retrieval system at Booking.

This paper inspired me to start a small demo project that searches song lyrics: [https://github.com/warrenqi/lucene-lyrics-www](https://github.com/warrenqi/lucene-lyrics-www)

## Preface

Lucene is a library and toolbox for search. Version 4 adds significant new features and improvements to the core components, which was already very rich. For example, staring in version 1.2, Lucene already had wildcards, edit distance matching, and full boolean operators. Version 3.6 added regex, custom scoring functions based on value of field.

Importantly, Lucene makes index formats ("Codecs") separate from the storage layer.

## S4 Lucene components

1. Analysis of incoming content AND queries
2. Indexing and storage
3. Searching
4. Ancillary modules (everything else, e.g. result highlighting)

## S4.1 Analysis

Mostly 3 tasks chained together
1. Optional character filtering and normalization (e.g. remove diacritics/glyphs)
2. Tokenization (delimiting)
3. Token filtering (e.g. stemming, stopword removal, n-gram creation)

## S4.2 Index and storage

There are many mechanical processes in this component.
1. A Document consists of 1 or more fields of content. These need to be stored.
2. Indexing of the documents is lock-free and highly concurrent.
3. Near Real Time indexing (new in version 4).
4. Segmented indexing with merging (and pluggable merge policies).
5. Abstractions to allow for different posting list data structures.
6. Transactional support.
7. Support for different scoring models.

## S4.3 Querying

Lucene provides a full toolbox of query parsers, as well as a full query parsing framework for developers to write their own query parsers.

## S4.4 Ancillary features

1. Result highlighting (snippets)
2. Faceting
3. Spatial search
4. Document grouping by key
5. Auto-suggest

## S5.1 Foundational pieces

1. Text analysis
    This is relevant for full text search.
    * Many attributes can be associated to a token.
    * A token value is also an attribute - it is in fact, the main "term" attribute.
    * Other example: part-of-speech tags
    * Language support and analysis is _very_ feature rich.

2. Finite State Automata
    Mostly relevant for improved full text search


## S5.2 Indexing

A key part I take away is Lucene's implementation of incremental update of index data. The actual storage of index data is abstracted to the "Directory API". There's a separation of "in-flight" data versus "persisted" data. Updates to index data are split into extents, named as the all-important **Segments**, and are periodically merged into larger segments.

Lucene assigns internal integer IDs to documents it scans. These IDs are ephemeral - they are used for identifying document data within a single particular **Segment**, and they're changed upon Segment compactions.

Two broad types of fields in a document - but a field can belong to _either or both_:
1. Fields carrying content to be indexed/inverted.
2. Fields carrying content to be simply stored


## S5.3 Incremental Index Updates

Also known as: how to handle extremely large indexes: divide and conquer.

Periodically, in-memory Segments are flushed to persistent storage (using the Codec and Directory abstractions), upon configurable threshold.

The IndexWriter class is responsible for processing index updates. It uses a pool of DocumentWriters that create new in-memory Segments. As new documents are added (to the end of the index usually), an index compaction/merge runs periodically, and it reduces the total number of Segments that comprise the whole index. **Note** A single large index is still split into multiple Segments. Version 4 makes the distinction of marking Segments as immutable.

I'm just going to quote this section from the paper because it's so well written:

>Each flush operation or index compaction creates a new commit point, recorded in a global index structure using a two-phase commit. The commit point is a list of segments (and deletions) comprising the whole index at the point in time when the commit operation was successfully completed.

>In Lucene 3.x and earlier, some segment data was mutable (for example, the parts containing  deletions or field normalization weights), which negatively affected the concurrency of writes and reads -- to apply any modifications the index had to be locked, and it was not possible to open  the index for reading until the update operation completed and the lock was released.

I'm not sure why Lucene 3 had to lock writes and reads here. The authors probably thought it through though, or the performance would not have improved enough versus fully implementing immutable segments, continuing below:

>In Lucene 4.0 the segments are fully immutable (write-once), and any changes are expressed either as new segments or new lists of deletions, both of which create new commit points, and the updated view of the latest version of the index becomes visible when a commit point is recorded using a two-phase commit. This enables lock-free reading operations concurrently with updates, and  point-in-time travel by opening the index for reading using some existing past commit point.


## S5.3.2 Concurrent IndexReaders

As relevant to multiple Segment writers, a user typically obtains an IndexReader from either a commit point (meaning: data flushed to disk), or directly from an Index**Writer**, which includes in-memory Segments.

For performance reasons, deletion is only actually removed during segment merge/compactions. This skews some global statistics slightly.

IndexReader implements the composite pattern: an IndexReader is actually a list of sub-Readers for each segment.

## S5.4 Codec and Directory API

This is the exciting stuff.

1. Experience an inverted index in Four-Dee (4D) !
    * We solve problems by using another layer of abstraction. 
    * Kidding aside, this is a brilliant idea. Lucene 4 Codec abstraction opens up different inverted index compression techniques (see RoaringBitmaps). This is possible by viewing an inverted index as a logical, 4 dimensional table consisting of the axis: (1) field, (2) term, (3) document, (4) position.
2. Codec API improves performance by allowing environment customizations
    * Online write-time sharding/pruning, speeding up reads using Bloom filters
3. Directory API
    * Lucene uses the file abstraction.
    * Flexible enough to be adopted from in-memory, to KV stores, to SQL stores, to HDFS.

## S5.5 Searching

This section is extremely interesting and relevant for full text search. Although, in interest of time, and because I'm now working on a search system that overwhelmingly deals with exact matches, I'm going to defer notes on this section for later.