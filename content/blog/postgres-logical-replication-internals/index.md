---
title: "Decoding the Stream: How Postgres Logical Replication Powers CDC"
date: "2026-06-10T09:00:00.000Z"
description: "Logical replication is the key to selective data distribution and Change Data Capture (CDC), allowing you to transform raw database logs into a real-time stream of high-level business events."
---

If you’ve ever used a tool like Debezium to stream database changes into Kafka, or if you’ve synced specific tables between two different Postgres versions, you’ve used **Logical Replication**. 

While traditional **Physical Replication** is great for high availability (it creates an exact, byte-for-byte copy of your disk), it’s a blunt instrument. You can’t replicate just one table, you can’t replicate between different OS versions, and your replica must be read-only. **Logical Replication** breaks all these rules by decoding the database’s "diary"—the Write-Ahead Log (WAL)—into a stream of logical row changes.

### 1. The WAL: The Source of Truth

At the heart of every Postgres transaction is the **Write-Ahead Log (WAL)**. Every `INSERT`, `UPDATE`, and `DELETE` is first appended to this log for durability. In physical replication, the primary server just streams these raw bytes to the standby.

In logical replication, we use **Logical Decoding**. A dedicated process called the **WALSender** reads these raw bytes and passes them through a "Decoding Plugin" (usually the built-in `pgoutput`). This plugin understands the Postgres internal format and translates the raw disk block changes back into high-level operations: *"Transaction 101 updated the 'email' column of user 42."*

### 2. Replication Slots: The Bookmark

How does the primary server know which parts of the log a subscriber has already seen? It uses **Replication Slots**. 

A replication slot is a persistent "bookmark" on the primary. It ensures that the primary **never deletes** a WAL file until all associated slots have successfully consumed it. 
- **The Risk:** This is the most dangerous part of Postgres administration. If your subscriber (e.g., your CDC tool) crashes and stops consuming data, the replication slot will keep pinning WAL files. Your disk will fill up, and eventually, the primary database will crash. Monitoring `pg_replication_slots` is mandatory.

### 3. The Reorder Buffer: Reassembling the Story

Transactions in Postgres are interleaved. Transaction A might start, then Transaction B starts, then B commits, then A commits. The WAL records the events in the order they hit the disk, meaning the records for A and B are mixed together.

The logical decoder uses a **Reorder Buffer** to reassemble these fragments. It buffers the changes for each transaction in memory (or on disk if they are large) and only emits them to the subscriber once the transaction has **committed**. If a transaction aborts, the decoder simply throws away its buffered changes. This ensures the subscriber only ever sees a consistent, committed view of the data.

### 4. Replica Identity: Who are you?

To handle an `UPDATE` or `DELETE`, the subscriber needs to know *which* row to change. Since logical replication doesn't use physical disk addresses, it needs a logical identifier.

Postgres uses **REPLICA IDENTITY**. 
- By default, this is the **Primary Key**. 
- If you don't have a primary key, you can set it to `FULL`, which means Postgres will log the entire old version of the row so the subscriber can perform a search to find the match. This is accurate but adds significant overhead to your WAL size.

### 5. Publications and Subscriptions

Postgres implements this via a **Publish-Subscribe** model:
-   **Publication:** A set of tables on the primary that you want to share. You can even filter by row (e.g., `WHERE status = 'active'`) or by column (only send the `id` and `email`).
-   **Subscription:** A connection from a downstream database that pulls data from a publication. 

This model allows for complex topologies: you can consolidate data from ten different regional databases into one central data warehouse, or you can split a monolithic database into multiple microservices by replicating specific tables.

```text
ASCII Logical Replication Flow:
[ Primary DB ]
  |-- (Transaction) --> [ WAL ]
                           |
                     [ WALSender ] --(Logical Decoding)--> [ pgoutput ]
                                                              |
[ Network Stream ] <------------------------------------------+
       |
[ Subscriber DB ]
  |-- [ Apply Worker ] --> (Update Tables)
```

### Conclusion

Logical Replication transformed Postgres from a standalone relational database into a first-class citizen of the modern data stack. By exposing the internal WAL as a stream of logical events, it enabled the rise of **Change Data Capture (CDC)** and allowed developers to build reactive, event-driven architectures directly on top of their primary database. It’s a reminder that the most valuable data isn't just what is currently in your tables, but the *history* of how that data is changing.
