---
title: Readers Never Block Writers: The Magic of Postgres MVCC
date: "2026-06-10T09:00:00.000Z"
description: "Postgres manages to handle thousands of concurrent users without locking your tables, but it pays for that speed by leaving a trail of 'ghost' data behind."
---

If you’ve ever used a database that used "table-level locking," you know the pain of a single slow report blocking every other user in the building. Postgres avoids this nightmare through a system called **Multi-Version Concurrency Control (MVCC)**. The core philosophy is deceptively simple: instead of forcing people to wait for a lock, Postgres just gives everyone their own version of the data. 

In a Postgres world, **readers never block writers, and writers never block readers.** But as we’ll see, this "magic" comes with a significant physical cost on your hard drive.

### The Hidden Columns: xmin and xmax

Every row (or "tuple") in a Postgres table has two hidden system columns that you never see in a standard `SELECT *`: **xmin** and **xmax**. These are the birth and death certificates of your data.

- **xmin:** The ID of the transaction that **inserted** this row.
- **xmax:** The ID of the transaction that **deleted** or **updated** this row. If the row is still live, `xmax` is 0.

When you run an `UPDATE` in Postgres, the database doesn't actually change the bits on the disk. Instead, it does two things:
1. It "marks" the old version as dead by setting its `xmax` to your current transaction ID.
2. It inserts a brand new version of the row with its `xmin` set to your current ID.

At any given moment, a single logical row might have five or six different physical versions sitting on the disk.

### The Snapshot: A Moment in Time

When your transaction starts, Postgres takes a **Snapshot**. This isn't a copy of the data; it’s just a list of which transaction IDs are currently active. 

Postgres uses this list and the `xmin`/`xmax` columns to decide which version of a row you’re allowed to see. If a row was inserted by a transaction that hasn't committed yet, you can't see it. If a row was deleted by a transaction that committed before you started, it’s invisible to you. It’s like everyone is walking through the same library, but because of the glasses they’re wearing, they see a different version of the books.

```text
ASCII MVCC Lifecycle:
[ Transaction 101 Starts ]
  INSERT -> Row A (xmin: 101, xmax: 0)

[ Transaction 102 Starts ]
  UPDATE -> Row A 
    - Old Row A (xmin: 101, xmax: 102) <-- Marked for death
    - New Row A (xmin: 102, xmax: 0)   <-- New life

[ Transaction 103 (Reader) Starts ]
  - Can see Old Row A (because 102 hasn't committed yet)
  - Cannot see New Row A
```

### The Cost: Dead Tuples and Table Bloat

The downside of never overwriting data is that your tables eventually fill up with "Dead Tuples"—old versions of rows that are no longer visible to *any* active transaction. If you update a million rows, you’ve just doubled the size of your table on disk, even if the "logical" amount of data stayed the same.

This is called **Bloat**. If you don't clean it up, your queries have to scan through millions of dead rows just to find the five live ones they actually need. Performance degrades, and your disk space disappears.

### The Necessary Evil: VACUUM

To handle this, Postgres has a background process called the **Autovacuum**. Its job is to wander through your tables, find the dead tuples that are truly gone, and mark their space as "reusable." 

Crucially, standard `VACUUM` doesn't give the space back to the operating system; it just makes it available for future Postgres inserts. It’s like erasing a page in a notebook so you can write on it again, rather than ripping the page out. (If you want the space back from the OS, you need `VACUUM FULL`, which locks the table and rewrites it from scratch—the very thing MVCC was designed to avoid).

### Transaction ID Wraparound: The Silent Killer

There’s one more quirk to MVCC. Transaction IDs are 32-bit integers, which means there are about 4 billion possible IDs. If your database runs for a long time and hits ID #4,000,000,001, it "wraps around" to zero. 

If this happens, old transactions suddenly look like they’re in the "future," and your data might vanish. To prevent this, the vacuum process also "freezes" old rows, marking them with a special bit that tells Postgres "this row is so old it’s visible to everyone forever."

### Why This Matters

Understanding MVCC is the key to tuning Postgres. It’s why long-running transactions are dangerous—they prevent the vacuum from cleaning up dead tuples because the vacuum doesn't know if that old transaction might still need to see them. 

Postgres is a masterclass in trade-offs. It gives you incredible concurrency and data integrity, but in exchange, it asks you to manage the physical reality of versioned data. It’s a reminder that in the world of high-performance databases, there is no such thing as a free lunch—only a very well-managed buffet.
