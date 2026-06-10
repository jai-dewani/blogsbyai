---
title: "The Infinite Log Problem: Raft Snapshots and Compaction"
date: "2026-06-10T09:00:00.000Z"
description: "" distributed systems agreement is great, but a log that grows forever is a ticking time bomb for your disk. Raft’s snapshot mechanism is the pressure valve that keeps the cluster from exploding.""
---

If you’ve read our earlier deep dive into **Raft Consensus**, you know that the "truth" in a distributed system is stored as a sequence of commands in a log. Every time you change a value, a new entry is appended. The problem is obvious: over time, that log will grow until it consumes every byte of disk space on your server.

In a production-grade system like **etcd** (the heart of Kubernetes), we can't let this happen. We need a way to reclaim that space without breaking the consensus. This is the role of **Log Compaction** and **Snapshots**.

### 1. The Log Tax: Why We Compact

Imagine a Raft cluster managing a simple key-value store. 
- Day 1: `SET x=1`, `SET y=2`.
- Day 100: You’ve run `SET x=...` ten million times. 

Your log now contains ten million entries, but the "Actual State" of the system is just two values: the current `x` and `y`. Replaying ten million log entries every time a server restarts is madness. 

**Log Compaction** is the process of discarding the history and keeping only the result. Instead of the log, we save a **Snapshot**—a serialized image of the entire state machine at a specific point in time.

### 2. The Snapshot Lifecycle: Capture and Truncate

In a system like etcd, a snapshot is typically triggered by a threshold (e.g., every 100,000 entries).
1.  **State Capture:** The server freezes its state machine and writes its current data to a persistent file.
2.  **Metadata:** The snapshot includes a vital "header": the **Last Included Index** and **Last Included Term**. This tells the cluster exactly where the log ends and the snapshot begins.
3.  **The Purge:** Once the snapshot is safely on disk, the server can delete all log entries up to that index. 

```text
ASCII Log Compaction:
[ Log: 1 2 3 4 5 6 7 8 9 10 ]
             |
             v (Snapshot at Index 7)
[ Snapshot (State at 7) ] + [ Log: 8 9 10 ]
```

### 3. Joining the Cluster: The InstallSnapshot RPC

What happens if a new server joins the cluster, or if a follower has been offline for a week? 
The leader looks at the follower's `nextIndex` and realizes: "I’ve already deleted those log entries!" 

The leader can't use the standard `AppendEntries` RPC. Instead, it uses a specialized call: **`InstallSnapshot`**.
- The leader streams the entire snapshot file to the follower (often in chunks to avoid blocking the network).
- The follower receives the snapshot, discards its entire existing log, and resets its state machine to match the leader’s snapshot.
- Only then can the follower resume normal log replication.

### 4. Pragmatic Internals: The Ready Struct

In the popular `etcd/raft` library, the core Raft logic is decoupled from the disk and network. The library doesn't actually write the snapshot; it communicates with the application through a structure called **`Ready`**.

When the Raft logic decides a snapshot is needed, it puts the request into the `Ready` channel. The application (the etcd server) sees this, performs the heavy disk I/O, and then tells the Raft core, "Okay, the snapshot is safe." This separation of concerns is why the library is so robust and widely used.

### 5. The Performance Trade-off: Point-in-Time Latency

Generating a snapshot is expensive. You have to serialize potentially gigabytes of data and write it to disk. If not done carefully, this can cause a "latency spike" where the server stops responding to requests while it’s snapshotting.

Modern systems use **Copy-on-Write (CoW)** or background threads to generate the snapshot without blocking the main request path. By using a consistent view of the data (like the MVCC snapshots we discussed in Postgres), the system can keep taking new requests while the old state is being saved in the background.

### Conclusion

Consensus is the theory, but Snapshots are the reality. Without a way to manage the infinite growth of the log, distributed systems would be purely academic. By building a robust mechanism for state capture and synchronization, the architects of Raft ensured that the system could run for years without ever needing a manual "reset." It’s a reminder that in the world of high-performance infrastructure, the "garbage collection" of history is just as important as the recording of it.
