---
layout: post
title:  "JVM Metaspace - Classloader memory allocation"
date:   2021-08-12
categories:
---

- Article link: [https://blogs.sap.com/2021/07/16/jep-387-elastic-metaspace-a-new-classroom-for-the-java-virtual-machine/](https://blogs.sap.com/2021/07/16/jep-387-elastic-metaspace-a-new-classroom-for-the-java-virtual-machine/)



# Commentary:

JEP 387 “Elastic Metaspace” – a new classroom for the Java Virtual Machine

This article discusses one of my favorite topics - Java off-heap memory. Specifically, the metaspace.


# Novel Idea:

This article introduces a new way for JVM to allocate memory for the Metaspace, a region of memory that JVM uses to load class objects. In the past, the Metaspace was managed with a slab allocator, but under stressful patterns of class loading and unloading, the slab allocator caused high fragmentation, and its allocation granularity was not tunable. This article, and the corresponding Java feature, enables more robust and elastic management of the Metaspace.


# Prior Work:

1. [https://bugs.openjdk.java.net/browse/JDK-8198423](https://bugs.openjdk.java.net/browse/JDK-8198423)
2. [https://spring.io/blog/2015/12/10/spring-boot-memory-performance](https://spring.io/blog/2015/12/10/spring-boot-memory-performance)
3. [http://trustmeiamadeveloper.com/2016/03/18/where-is-my-memory-java/](http://trustmeiamadeveloper.com/2016/03/18/where-is-my-memory-java/)

# Content:


## Off-Heap Memory and the Metaspace

The JVM manages the following in off-heap:

1. Thread stacks
2. GC control structures
3. Interned Strings
4. CDS archives and text segments
5. JIT-compiled code (the code cache)
6. and many many other things

> All these data live outside the Java heap, either in C-Heap or in manually managed mappings. Colloquially named off-heap – or, somewhat more incorrectly, native – memory, the combined size of these regions can surpass that of the heap itself.

A java class is removed – unloaded – only if its loading class loader dies. The Java Specification defines this:

> “A class or interface may be unloaded if and only if its defining class loader may be reclaimed by the garbage collector ” [5].

## Before Java 8: The Permanent Generation

Before Java 8, class metadata lived in PermGen, which was managed by JVM GC. JEP 122 removed the PermGen: two factors coincided, (1) The advent of JRockit JVM, and (2) the Sun-JVM group internal roadmap.


_Java 8_

JEP 122 shipped with JVM 8. The first Metaspace was implemented with several features:

> Better suited for bulk-release data is a so-called Arena Allocator [9]. In its simplest form, an arena is a contiguous region: on each new allocation request, an allocation top pointer is pushed up to make space for the new allocation. This technique is primitive but very fast. It is also very memory-efficient since waste is limited to padding requirements. We pay for this efficiency by not being able to track individual allocations: allocations are not released individually but only as a whole, by scrapping the arena.

> The Metaspace is also, in its heart, an arena allocator. Here, an arena is not bound to a thread like a thread stack or a TLAB. Instead, a class loader owns the arena and releases it upon death.

From first glance, this allocation pattern is a perfect match for class loading/unloading - leaf/related classes are allocated close to the parent classes, and the root node is de-allocated last when all children objects are unloaded, resulting in a whole slab de-allocation.

## Metaspace Problems

- Fixed chunk sizes

> For a start, Metaspace chunk management had been too rigid. Chunks, which came in various sizes,  could never be resized. That limited their reuse potential after their original loader died. The free list could fill up with tons of chunks locked into the wrong size, which Metaspace could not reuse.

- The lack of elasticity

The first Metaspace also lacked elasticity and did not recover well from usage spikes.

> When classes get unloaded, their metadata are not needed anymore. Theoretically, the JVM could hand those pages back to the Operating System. If the system faces memory pressure, the kernel could give those free pages to whoever needs it most, which could include other areas of the JVM itself. Holding on to that memory for the sake of some possible future class loads is not useful.

> But Metaspace retained most of its memory by keeping freed chunks in the free list. To be fair, a mechanism existed to return memory to the OS by unmapping empty virtual space nodes. But this mechanism was very coarse-grained and easily defeated by even moderate Metaspace fragmentation. Moreover, it did not work at all for the class space.

From the JDK ticket: "So, we could get metaspace OOMs even in situations where the metaspace was far from exhausted"

- High per-classloader overhead

> In the old Metaspace, small class loaders were disproportionally affected by high memory overhead. If the size of your loaders hit those “sweet spot” size ranges, you paid significantly more than the loader needed. For example, a loader allocating ~20K of metadata would consume ~80K internally, wasting upward of 75% of the allocated space.

These amounts are tiny but quickly add up when dealing with swarms of small loaders. This problem mostly plagued scenarios with automatically generated class loaders, e.g., dynamic languages implemented atop Java.


## Java 16: Metaspace evolved


- Buddy Allocation

Very simplified, buddy allocation in Metaspace works like this:

1. Classloader requests space for metadata; its arena needs and requests a new chunk from the chunk manager.
2. The chunk manager searches the free list for a chunk equal or larger than the requested size.
3. If it found one larger than the requested size, it splits that chunk repeatedly in halves until the fragments have the requested size.
4. It now hands one of the splinter chunks over to the requesting loader and adds the remaining splinters back into the free list.

- Deallocation of a chunk works in reverse order:

1. Classloader dies; its arena dies too and returns all its chunks to the chunk manager
2. The chunk manager marks each chunk as free and checks its neighboring chunk (“buddy”). If it is also free, it fuses both chunks into a single larger chunk.
3. It repeats that process recursively until either a buddy is encountered that is still in use, or until the maximum chunk size (and by that, maximum defragmentation) is reached.
4. The large chunk is then decomitted to return memory to the operating system.

> Like a self-healing ice sheet, chunks splinter on allocation and crystallize back into larger units on deallocation. It is an excellent way of keeping fragmentation at bay even if this process repeats endlessly, e.g., in a JVM which loads and unloads tons of classes in its life time.

# Results

- Code
[https://github.com/openjdk/jdk/commit/7ba6a6bf003b810e9f48cb755abe39b1376ad3fe](https://github.com/openjdk/jdk/commit/7ba6a6bf003b810e9f48cb755abe39b1376ad3fe)


