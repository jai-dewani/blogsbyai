---
title: The Math of Collaboration: A Deep Dive into CRDTs
date: "2026-06-10T09:00:00.000Z"
description: "How do tools like Figma and Google Docs allow thousands of people to edit the same data at the same time without a central server deciding who is 'right'? The answer lies in the elegant mathematics of CRDTs."
---

If you’ve ever tried to build a collaborative app, you’ve hit the wall of **Conflict Resolution**. What happens when two users change the same word at the exact same millisecond? In the old days, we relied on "Locking" (only one person can edit at a time) or "Last Writer Wins" (the second person’s work is simply deleted). Both are terrible user experiences.

**CRDTs (Conflict-free Replicated Data Types)** changed the game. They are a family of data structures that allow multiple people to edit the same data offline or concurrently, and then merge their changes automatically without any conflicts. It’s not magic; it’s a masterclass in set theory and abstract algebra.

### 1. Strong Eventual Consistency (SEC)

The core goal of a CRDT is **Strong Eventual Consistency**. In a traditional system, you need a central server to act as the "Source of Truth." In a CRDT system, there is no single source of truth. Instead, every replica is guaranteed to arrive at the exact same state if they have received the same set of updates, regardless of the **order** in which those updates arrived.

### 2. The Two Flavors: State vs. Operation

There are two primary ways to implement a CRDT:

**A. State-based (CvRDT):**
In this model, replicas synchronize by sending their **entire state** to each other. When a replica receives a remote state, it uses a `merge()` function to combine it with its local data.
-   **The Secret Sauce:** The merge function must be a **Join-Semilattice**. This is a fancy way of saying it must be **Commutative** (order doesn't matter), **Associative** (grouping doesn't matter), and **Idempotent** (merging the same thing twice changes nothing).

**B. Operation-based (CmRDT):**
Instead of sending the whole state, replicas broadcast only the **operations** (e.g., `add "X" at position 5`).
-   **The Catch:** This requires a reliable messaging layer that ensures operations are delivered "exactly once" and in "causal order."

### 3. The Counter Problem: G-Counters and PN-Counters

You might think a simple counter is easy, but it’s the perfect example of why CRDTs are necessary. If two nodes both increment a counter from 0 to 1, and then merge, a naive merge would result in `1`. But the "correct" result should be `2`.

-   **G-Counter (Grow-only):** Each node maintains its own integer in an array: `[node1: 1, node2: 1]`. The "value" is the sum of the array. To merge, you just take the **maximum** of each index. This satisfies all the semilattice properties.
-   **PN-Counter (Positive-Negative):** Since you can't "decrement" a G-Counter (it would violate monotonicity), a PN-Counter uses two G-Counters: one for additions and one for subtractions. The final value is `Sum(Additions) - Sum(Subtractions)`.

### 4. The Challenge of Deletion: Tombstones

In a collaborative text editor, deleting a character is a nightmare. If I delete a word while you are editing it, how does the system know where your edits should go?

CRDTs solve this with **Tombstones**. When you delete an item, it isn't actually removed from the internal memory. Instead, it’s marked with a "dead" flag. This ensures that the item still exists in the "coordinate system" of the document, so other concurrent operations can find their bearings.
-   **The Cost:** Tombstones are the "garbage" of the CRDT world. They consume memory forever. Modern libraries like **Yjs** use highly optimized structures to compress these tombstones, making CRDTs performant enough for massive documents.

```text
ASCII CRDT Merge logic:
[ User A State: {X: 5, Y: 2} ]       [ User B State: {X: 3, Y: 8} ]
              |                             |
              +----------( Sync )-----------+
                             |
                   [ Merged Result ]
              { X: max(5, 3), Y: max(2, 8) }
                     { X: 5, Y: 8 }
```

### 5. Why Not Operational Transformation (OT)?

You might have heard of **Operational Transformation**, the technology behind Google Docs. 
- **OT** requires a central server to re-order every single keystroke. It is incredibly complex to implement (the "concurrency control" logic is thousands of lines of code with endless edge cases).
- **CRDTs** are decentralized. They work perfectly in peer-to-peer environments (like Figma or local-first apps). The complexity is moved into the data structure itself, which is mathematically proven to be correct.

### Conclusion

CRDTs represent a shift in how we think about "truth" in computing. We are moving from a world of "one server knows all" to a world where "everyone knows a piece, and the math brings us together." By embracing commutativity and idempotency, we can build apps that feel truly collaborative and work seamlessly across the planet. It’s a reminder that sometimes the most difficult engineering problems are solved not with more servers, but with better math.
