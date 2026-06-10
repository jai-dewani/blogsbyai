---
title: Why Your Database Writes to a Log Before It Writes to a Table
date: "2026-06-27T11:30:00.000Z"
description: "Traditional databases like to update things in place, but high-performance systems like RocksDB and Cassandra treat your data like a never-ending diary."
---

If you’ve ever wondered how a database can handle millions of writes per second without breaking a sweat, the answer usually involves a data structure called a Log-Structured Merge-Tree (LSM Tree). Most of us grew up with B-Trees, where the database finds a specific spot on the disk and updates it "in place." That works fine for small workloads, but when you’re dealing with the scale of a company like Facebook or Netflix, random writes to a disk are a death sentence for performance. LSM Trees fix this by turning every write into a sequential append.

The core idea is simple: never go back and change what you’ve already written. When a new piece of data comes in, the database does two things. First, it appends the change to a Write-Ahead Log (WAL) on the disk. This is just a safety net; if the power goes out, the log is how the database remembers what it was supposed to do. Second, it adds the data to an in-memory sorted structure called a Memtable. Because it's in RAM, this is incredibly fast.

But RAM is expensive and finite. Once the Memtable gets full, the database "freezes" it and flushes the whole sorted block to the disk as a Sorted String Table (SSTable). These files are immutable—they never change. If you update a key, the database doesn't find the old version and swap it out; it just writes a new version to a new SSTable. This is why LSM Trees are "log-structured." The database is basically a series of increasingly larger files that represent the history of your data.

The catch, of course, is reading. If you want to find a specific key, it might be in the current Memtable, or it might be buried in one of fifty different SSTables on the disk. To keep things from slowing down, LSM systems use a probabilistic tool called a Bloom Filter. It’s a tiny bit of metadata that can tell the database "this key definitely isn't in this file," allowing it to skip unnecessary disk reads.

Eventually, you end up with too many files and too many old versions of the same data. That’s where "compaction" comes in. A background process wakes up, grabs a few SSTables, and merges them into a single, larger file, throwing away the old versions and the "tombstones" (delete markers) along the way. It’s a constant trade-off between write speed, read speed, and disk space—what engineers call the RUM Conjecture. We’re basically trading a bit of background maintenance for the ability to write data as fast as the hardware will allow.
