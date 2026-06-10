---
title: "Speed, Simplicity, and SDS: The Redis Internal Masterclass"
date: "2026-06-10T09:00:00.000Z"
description: "Redis is often called 'just a cache,' but its internal data structures and single-threaded event loop are a masterclass in high-performance systems engineering."
---

When people talk about Redis, they usually focus on how fast it is. They quote benchmarks of millions of operations per second and move on. But the "why" behind that speed is where things get interesting. Redis doesn't just run fast; it’s designed from the ground up to minimize every possible source of latency, from locking overhead to memory fragmentation. 

The secret to Redis is its extreme pragmatism. It uses a single-threaded event loop and a collection of custom, memory-optimized data structures that would make a textbook computer scientist blush.

### The Reactor Pattern: One Thread to Rule Them All

The most controversial part of the Redis architecture is its single-threaded nature. In a world where every phone has eight cores, why would a high-performance database use only one? 

The answer is **Lock Contention**. In a multi-threaded database, you spend a massive amount of CPU cycles just managing locks to ensure two threads don't change the same value at the same time. Redis avoids this entirely. Because it's single-threaded, there are no locks. No context switching. No race conditions. It just processes commands one after another as fast as the CPU can possibly go.

It handles thousands of concurrent connections using an Asynchronous Event library called **AE**. It uses OS-native primitives like `epoll` on Linux or `kqueue` on macOS to "watch" all its sockets. The kernel tells Redis exactly which sockets have data waiting, and Redis processes them in a never-ending loop.

```text
ASCII Event Loop (AE):
[ Network Sockets ] --(epoll)--> [ Event Queue ]
                                       |
                                       v
                             [ Single Worker Thread ]
                                       |
                             [ Process Commands ]
                                       |
                             [ Write Responses ]
```

### SDS: Better Than C Strings

C strings (`char*`) are simple, but they’re also dangerous and slow. You don't know how long they are without scanning the whole memory block, and they’re not binary-safe because they end at the first null byte (`\0`). 

Redis uses its own string implementation called **SDS (Simple Dynamic Strings)**. An SDS string has a header that stores the length and the amount of allocated space. 
- **O(1) strlen:** Getting the length is a constant-time lookup.
- **Binary Safety:** You can store anything (images, Protobufs) because the null byte doesn't mean "the end."
- **Pre-allocation:** When you append to a string, Redis allocates more memory than it needs. If you add 1 byte, it might allocate 1024, so the next 1023 additions are effectively free.

### The Memory Sandwich: Listpacks and Quicklists

The most impressive part of Redis internals is how it adapts to the size of your data. If you have a small list of 10 items, Redis doesn't use a traditional linked list (which has massive pointer overhead). Instead, it uses a **Listpack** (or **Ziplist** in older versions).

A Listpack is a contiguous block of memory where all elements are shoved together one after another. There are no pointers. It’s incredibly cache-friendly because the CPU can read the whole list into its cache in one shot. 

But as your list grows, a Listpack becomes slow because you have to reallocate a huge block of memory every time you add an item. At that point, Redis automatically converts the list into a **Quicklist**. This is a hybrid: a doubly linked list where each node is a small, manageable Listpack. You get the fast head/tail access of a linked list with the memory efficiency of a Listpack.

### Incremental Rehashing: Avoiding the "Pause"

Every hash table eventually needs to grow (rehash). In most systems, this is a "stop the world" event where the entire table is copied to a new, larger one. If you have 10 million keys, your database might freeze for a second while this happens.

Redis avoids this with **Incremental Rehashing**. Every dictionary in Redis actually has *two* hash tables. When it’s time to grow, Redis starts moving buckets from the old table to the new one, but it only moves a few buckets at a time—every time a client sends a command. It spreads the "cost" of the rehash across thousands of individual requests, so the user never notices a latency spike.

### Why It Matters

Redis isn't fast by accident. It’s fast because it recognizes that the biggest bottleneck in modern computing isn't the CPU speed; it's the cost of moving data and managing complexity. By building a single-threaded system that uses custom, adaptive data structures, Redis proves that simplicity is the ultimate sophistication. It’s a reminder that sometimes the best way to handle millions of requests is to just do them one at a time, very, very carefully.
