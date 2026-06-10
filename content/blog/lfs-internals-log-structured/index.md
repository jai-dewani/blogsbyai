---
title: "The Log-Structured File System: Writing Everything as a Log"
date: "2026-06-10T09:00:00.000Z"
description: "Traditional file systems waste time waiting for the disk to spin. LFS proved that if you treat your entire hard drive like a single, never-ending log, you can achieve nearly 100% write efficiency."
---

If you’ve read our earlier deep dives into **LSM Trees** (the engine of modern databases) or **ZFS** (the unkillable file system), you’ve seen a recurring theme: **Append-Only is faster than Update-In-Place.** 

This idea didn't start with modern NoSQL databases. It was born in 1991 at Berkeley with the **Log-Structured File System (LFS)**. Designed by Mendel Rosenblum and John Ousterhout, LFS was a radical departure from the "Fast File System" (FFS) of the 1980s. It recognized a simple truth: CPU and memory were getting faster, but the physical physics of spinning disks (seek time) were not. 

The only way to make storage keep up was to stop moving the disk head and start writing data as a continuous, high-speed stream.

### 1. The Core Philosophy: The Log is the File System

In a traditional file system, every file has a fixed home. If you update a 4KB block in a file, the OS calculates the exact physical location on the disk and overwrites it. 

LFS abandons this. In LFS, **there are no fixed locations.** All modifications—data blocks, metadata, directory entries, and inodes—are buffered in memory and written sequentially to the tail of a giant log. 
- **The Result:** Instead of ten small, random writes (which would require ten slow disk seeks), LFS performs one single, massive sequential write. It utilizes nearly 100% of the disk's bandwidth.

### 2. The Floating Inode: Where did my file go?

If a file's data blocks keep moving to the end of the log, how does the system find them? In FFS, inodes live in a fixed table. In LFS, the **Inodes float.** 

To solve this, LFS introduced the **Inode Map (imap)**. This is a table that maps an inode number to its current physical address on the disk. 
- **Checkpoints:** To find the imap itself, LFS maintains a fixed-location **Checkpoint Region (CR)** at the start of the disk. This region contains pointers to the latest chunks of the imap. Every 30 seconds, LFS updates this region, providing a consistent "snapshot" of the entire system.

### 3. The Segment Cleaner: Paying the Log Tax

Writing to a log is fast, but it creates a problem: "garbage." When you update a file, the old version of its blocks are still on the disk, but they are no longer reachable. If you don't clean them up, you'll eventually run out of space.

LFS manages this through **Segments**. The disk is divided into large, fixed-size chunks (e.g., 1MB). A background process called the **Cleaner** (the original Garbage Collector) is responsible for reclaiming space:
1. It reads a few partially-full segments into memory.
2. It identifies which blocks are still "live" by checking the current imap.
3. It writes those live blocks into a new, compact segment at the end of the log.
4. It marks the old segments as "empty" and ready to be reused.

```text
ASCII LFS Layout:
[ Segment 1: Full ] [ Segment 2: Partially Dead ] [ Segment 3: Active Log ]
       |                      | (Cleaning)                |
       v                      v                           v
[ Seg 1 ] [ Seg 2 (Wiped) ] [ Seg 3 ] [ Seg 4: Compacted Live data from Seg 2 ]
```

### 4. Crash Recovery: Roll-Forward

Recovery in LFS is orders of magnitude faster than the traditional `fsck`. Instead of scanning the entire disk for inconsistencies, LFS simply finds the last valid **Checkpoint Region**. 

Because LFS uses an append-only log, the system can even "roll forward" from the last checkpoint by scanning the log tail to recover data that was written in the seconds just before the power went out.

### 5. The Legacy of LFS

LFS was highly controversial in the 90s because the cleaner overhead could be high for certain workloads. However, the architecture proved to be prophetic.
- **SSDs:** Modern Solid State Drives are fundamentally log-structured at the firmware level (the **FTL** we discussed earlier). They must erase blocks before writing, so they write everything as a log and clean up garbage in the background—exactly like LFS.
- **Databases:** Almost all modern high-performance databases (RocksDB, Cassandra, InfluxDB) use log-structured storage to achieve their throughput.

### Conclusion

The Log-structured File System is a masterclass in predicting the future. By realizing that physical disk seeks were the ultimate bottleneck, Rosenblum and Ousterhout created an architecture that optimized for the most expensive resource. While LFS didn't replace FFS on our desktops, it provided the blueprint for the entire modern world of high-performance storage. It’s a reminder that sometimes the best way to handle a slow resource is to stop pretending it’s fast and start working around its physical limits.
