---
title: "Why Your Database Writes to a Log Before It Writes to a Table"
date: "2026-06-10T09:00:00.000Z"
description: ""Traditional databases like to update things in place, but high-performance systems like RocksDB and Cassandra treat your data like a never-ending diary.""
---

If you’ve ever wondered how a database can handle millions of writes per second without breaking a sweat, the answer usually involves a data structure called a Log-Structured Merge-Tree (LSM Tree). Most of us grew up with B-Trees, which are the industry standard for traditional relational databases like MySQL or Postgres. In a B-Tree, the database finds a specific spot on the disk and updates it "in place." 

### The B-Tree Bottleneck

The problem with B-Trees is random I/O. When you update a record, the database has to jump to a specific location on the disk. For modern SSDs and NVMe drives, sequential writes are significantly faster than random ones. When you’re dealing with the scale of a company like Facebook, Netflix, or a massive IoT pipeline, random writes to a disk are a death sentence for performance. 

LSM Trees fix this by turning every single write—no matter where it belongs in the sorted order—into a sequential append at the end of a file.

### The Write Path: WAL and Memtable

The core philosophy of an LSM Tree is simple: never go back and change what you’ve already written. When a new piece of data comes in, the database doesn't search for an existing record to overwrite. Instead, it does two things simultaneously.

1. **Write-Ahead Log (WAL):** It appends the change to a sequential file on the disk. This is purely a safety net. Since writes to the end of a file are extremely fast, this ensures that the data is "durable"—if the power goes out two milliseconds later, the log is how the database remembers what it was supposed to do during the recovery process.
2. **Memtable:** The database adds the data to an in-memory sorted structure known as a Memtable (typically a SkipList or a Red-Black Tree). Because it's stored in RAM, this is incredibly fast. You get an immediate "success" response without waiting for a complex disk update.

```text
ASCII Write Path:
[Client Write] ----> [Write-Ahead Log (Disk - Sequential)]
               |
               ----> [Memtable (RAM - Sorted)]
```

### The Flush: From RAM to SSTables

RAM is expensive and finite. You can't just keep adding data to the Memtable forever. Once the Memtable reaches a threshold (usually 64MB or 128MB), the database "freezes" it. It becomes immutable, and a new, empty Memtable is created for incoming writes. 

A background thread then takes that frozen Memtable and flushes the whole sorted block to the disk as a Sorted String Table (SSTable). These files are permanent and never change. If you update a key, the database doesn't find the old version and swap it out; it just writes a new version of that key to a brand new SSTable. 

This is why LSM Trees are "log-structured." Your database isn't a single monolithic table; it's a series of increasingly larger files that represent the history of your data over time.

### The Read Path: Bloom Filters and Indices

The real challenge comes when you want to read. If you want to find the value for "user_123," it might be in the current Memtable, or it might be buried in any one of fifty different SSTables on your disk. Checking every single file would be devastating for performance. To solve this, LSM systems use a probabilistic tool called a **Bloom Filter**.

A Bloom Filter is a tiny bit of metadata stored in memory that can tell the database with 100% certainty if a key **definitely isn't** in a specific file. 
- If the Bloom Filter says "No," the database skips that file entirely.
- If it says "Maybe," the database checks a sparse index to find the exact block within the SSTable and reads it.

### Compaction: Organizing the Library

Eventually, you end up with too many files and too many old, obsolete versions of the same data. This is where "Compaction" comes in. A background process wakes up, grabs a few SSTables, and merges them into a single, larger file. During this merge, it throws away the older versions of keys and processes "tombstones" (special markers used to indicate that a piece of data has been deleted).

There are two main strategies for compaction:
| Strategy | Goal | Use Case |
|----------|------|----------|
| **Size-Tiered** | Low write cost | High-ingestion (Cassandra) |
| **Leveled** | Low read cost, less space | Read-heavy (RocksDB) |

### The RUM Conjecture

This architecture represents a fundamental trade-off in computer science known as the RUM Conjecture (Read, Update, Memory). You're trading background CPU work and disk space (the compaction process) for the ability to write data as fast as your hardware will allow. 

In an era where we need to ingest millions of events per second, the LSM Tree has become the backbone of the modern high-performance web. It treats your data like a never-ending diary, recording every thought as it happens, and only worrying about organizing the library when it has a free moment.
