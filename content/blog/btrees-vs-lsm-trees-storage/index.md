---
title: "B-Trees vs. LSM Trees: The Ultimate Storage Showdown"
date: "2026-06-10T09:00:00.000Z"
description: "Choosing between a B-Tree and an LSM Tree isn't just about speed; it's a fundamental trade-off between read latency, write throughput, and the longevity of your hardware."
---

If you’re building a database, you eventually have to answer one question: how do you store the data on the disk? For thirty years, the answer was almost always a **B-Tree**. It’s the engine behind PostgreSQL, MySQL, and Oracle. But in the last decade, a new contender called the **Log-Structured Merge-Tree (LSM Tree)** has taken over the world of high-throughput distributed systems like Cassandra, RocksDB, and Bigtable.

Choosing between them isn't about which one is "better." It’s about which trade-offs you’re willing to live with. This is the world of the **RUM Conjecture**, which states that you can only optimize for two of three things: **R**eads, **U**pdates, or **M**emory (Space).

### B-Trees: The Art of the In-Place Update

B-Trees are the "traditional" way to store data. They treat the database as a series of fixed-size pages (usually 16KB). When you update a row, the B-Tree finds the page on the disk, reads it into memory, changes the bits, and writes it back to the exact same spot. 

**The Advantage: Low Read Amplification**
Because data is stored in a balanced, sorted tree, a point lookup is incredibly fast. You start at the root, follow three or four pointers, and you have your data. You almost never have to read more data than you actually asked for. This makes B-Trees the undisputed king of point reads and range scans.

**The Downside: The Random Write Penalty**
The "in-place" nature of B-Trees is their Achilles' heel. Every update requires a random I/O seek. On modern SSDs, this is better than it used to be, but it’s still much slower than a sequential write. Furthermore, B-Trees suffer from **Write Amplification**. To update a single 100-byte row, you have to write an entire 16,000-byte page. That’s a 160x overhead for a tiny change.

### LSM Trees: The Power of the Never-Ending Log

LSM Trees take the opposite approach. They never go back and change what they’ve already written. Every write is a sequential append to a log and an update to an in-memory buffer (the Memtable). Periodically, these buffers are flushed to disk as immutable **SSTables**.

**The Advantage: Massive Write Throughput**
Because all writes are sequential, LSM Trees can ingest data as fast as the hardware can move bits. They transform the "chaotic" random writes of your application into a "disciplined" stream of data. This is why LSM Trees power almost every time-series and logging database on the planet. They are also much friendlier to SSDs, as they minimize the wear-and-tear caused by constant random updates.

**The Downside: The "Hidden" Read Cost**
The cost of fast writes is slow reads. To find a single key, an LSM Tree might have to check the Memtable, then check a Bloom Filter, and then potentially search through five or six different SSTables across different "levels" of the tree. This is high **Read Amplification**. Even with Bloom Filters, range queries are difficult because the database has to merge results from multiple sorted files on the fly.

### The Comparison: RUM in Action

| Feature | B-Tree (Postgres/MySQL) | LSM Tree (RocksDB/Cassandra) |
| :--- | :--- | :--- |
| **Strategy** | In-place Updates | Append-only Logs |
| **Optimized For** | Read Latency (R) | Write Throughput (U) |
| **Space Usage** | Moderate (Fragmentation) | High (Shadowing/Tombstones) |
| **Write Pattern** | Random I/O | Sequential I/O |
| **Main Bottleneck** | Disk Seek / Locking | Compaction / Background I/O |

```text
ASCII Storage Layout:
B-Tree: [ Root ] -> [ Internal ] -> [ Leaf Page (Mutable) ]
                                          ^
                                          | (Overwrite)

LSM Tree: [ Memtable ] -> [ L0 SSTable ] -> [ L1 SSTable ] -> [ L2 SSTable ]
                                          ^
                                          | (Merge/Compact)
```

### Space Amplification: The Silent Disk Eater

B-Trees lose space to **Internal Fragmentation** (pages are rarely 100% full). LSM Trees lose space to **Shadowing**. When you update a key in an LSM Tree, the old version stays on the disk until a background "Compaction" process cleans it up. If you have a high-update workload, an LSM Tree can easily take up twice as much disk space as the actual data size.

### Conclusion: Which One Wins?

If your application is a traditional "Online Transactional Processing" (OLTP) system where users are reading their profiles and making small updates, **B-Trees** are almost always the right choice. Their predictable read performance is hard to beat.

But if you’re building an ingestion pipeline, a heavy logging system, or a globally distributed NoSQL store where write speed is the only thing that matters, the **LSM Tree** is the only way to go. 

The history of database engineering is the history of choosing which part of the RUM conjecture you’re willing to sacrifice. We haven't found the "perfect" data structure yet, and until we do, we’ll keep trading read speed for write throughput, one byte at a time.
