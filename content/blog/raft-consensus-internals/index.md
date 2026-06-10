---
title: "Raft Consensus: How Computers Agree on the Truth"
date: "2026-06-10T09:00:00.000Z"
description: "agreement is easy when you’re a single machine, but when you’re a distributed cluster, reaching a consensus is a high-stakes game of politics and logic."
---

In a single-server world, consistency is easy. You write a value to a database, and the next person to read it gets that value. But we don't live in a single-server world anymore. We live in a world of distributed clusters where any node can crash, any network cable can be snipped, and any data center can lose power at any time.

How do you ensure that a cluster of five servers all agree on the current state of your application? This is the problem of **Consensus**. For years, the industry struggled with Paxos, an algorithm so complex that even its inventors admitted it was hard to implement. Then came **Raft**. 

Raft was designed with one primary goal: **Understandability.** It breaks the consensus problem into three distinct, manageable pieces: **Leader Election**, **Log Replication**, and **Safety.**

### The Strong Leader Model

Raft operates on the principle that the easiest way to manage a group is to have a strong leader. At any given time, every node in a Raft cluster is in one of three states:
1.  **Follower:** Passive. They just listen and do what they're told.
2.  **Candidate:** A follower that has timed out and is trying to become the new leader.
3.  **Leader:** The source of truth. All client requests must go through the leader.

Time is divided into logical **Terms**—monotonically increasing integers that act as a clock for the cluster. Every term starts with an election.

### Leader Election: The Randomized Timeout

How does the cluster decide who the leader is? Every follower has an **Election Timeout**. Here’s the trick: the timeout is **randomized** (usually between 150ms and 300ms). 

If a follower doesn't hear a "heartbeat" from the leader before its timer runs out, it becomes a Candidate. It increments its term, votes for itself, and sends a `RequestVote` RPC to everyone else. If it gets a majority of votes, it becomes the Leader.

The randomization is critical. Without it, every server would time out at the exact same time, they would all vote for themselves, and you'd have a "split vote" forever. By randomizing the clock, Raft ensures that one server will almost always time out first and win the election.

```text
ASCII Election Flow:
[S1: Follower] --- (Timeout!) ---> [S1: Candidate]
[S1] --(RequestVote)--> [S2, S3, S4, S5]
[S2, S3, S4] --(Vote: Yes)--> [S1]
[S1] --(Majority!) ---> [S1: LEADER]
```

### Log Replication: The Heartbeat of Truth

Once a leader is elected, it starts taking orders from clients. But it doesn't just execute them. It appends the command to its **Log** and broadcasts an `AppendEntries` RPC to all followers.

A command is only considered **Committed** once a majority of followers have successfully written it to their own logs. Once it’s committed, the leader applies it to its local **State Machine** (the actual database or application logic) and returns a success to the client.

Raft ensures that all logs stay in sync through a consistency check. When the leader sends a new log entry, it also sends the index and term of the entry *immediately preceding* the new one. If the follower doesn't have a match at that index, it rejects the update. The leader then retries with an earlier index until they find a common point in history. It’s like a conversation where you say, "Okay, we agree up to page 42, now let's talk about page 43."

### Safety: The Up-to-Date Rule

Consensus is useless if a node with an old, stale log can become the leader and overwrite the current truth. Raft prevents this with a simple safety rule: **A voter will only grant its vote if the candidate’s log is at least as up-to-date as its own.**

This ensures that the leader of any term must contain all the committed entries from all previous terms. The "truth" can never be lost, even if the leader crashes immediately after committing a value.

### Raft in the Real World: etcd and Consul

Raft isn't just a theoretical paper. It is the foundation of the modern cloud.
-   **etcd:** The primary data store for **Kubernetes**. Every time you scale a deployment or create a service, you are sending a command to a Raft leader in an etcd cluster. etcd uses a high-performance Go implementation of Raft to ensure your cluster state is always consistent.
-   **Consul:** Used for service discovery and configuration. Consul allows you to choose your consistency level—you can ask for "Consistent" reads (which always go to the leader) or "Stale" reads (which any node can answer), allowing you to trade off safety for lower latency.

### Conclusion

Raft proves that in software engineering, simplicity is a feature. By taking the "black box" of distributed consensus and turning it into a clear, logical progression of elections and log appends, Raft made reliable distributed systems accessible to everyone. It’s a reminder that the best algorithms aren't the ones that are the most "clever," but the ones that we can actually reason about and implement correctly. When you’re trusting a system with your most critical data, you don't want magic—you want Raft.
