---
title: "The Last Resort: Inside the Linux OOM Killer"
date: "2026-06-10T09:00:00.000Z"
description: "When your server runs out of memory, it doesn't just crash. It starts making high-stakes, life-and-death decisions about which process to sacrifice to save the system."
---

If you’ve ever managed a Linux server, you’ve likely seen the message: `Out of memory: Kill process 1234 (mysqld)`. It’s a gut-punch for any DBA, but it’s also a sign that your kernel just did its job. 

The **OOM (Out-of-Memory) Killer** is the Linux kernel’s final line of defense. When physical RAM and swap are completely exhausted, the kernel is faced with a catastrophic choice: let the entire machine freeze, or kill a specific process to reclaim memory. 

The way the kernel chooses that "victim" is not random. It uses a calculated, heuristic-driven "Badness" function to decide which process is the most deserving of termination.

### 1. The Trigger: The Failed Allocation

The OOM Killer doesn't run in the background. It is triggered by the **Page Allocator**. 

When a process asks for more memory (via `malloc` or a page fault), the kernel tries to find a free page. If it can't, it goes through a desperate cycle of reclaim:
1.  **Discard Page Cache:** It throws away your cached files from RAM.
2.  **Swap:** It pushes cold data to the disk.
3.  **Compaction:** It tries to move data around to create large contiguous blocks.

If all of this fails, the allocator calls `out_of_memory()`. The machine is now in a "Lifeboat" situation—someone has to be thrown overboard.

### 2. The Badness Function: oom_score

The kernel iterates through every process and assigns it an **`oom_score`**. You can see this value for any running process at `/proc/[pid]/oom_score`. 

The scoring formula (post-2.6.36) is simple and brutal:
-   **Baseline:** The score is the percentage of the system's allowed memory that the process is currently using. If your app uses 50% of the RAM, its score starts at **500**.
-   **Max Score:** The maximum possible score is **1000**.
-   **Privilege Discount:** If a process has root-level privileges (`CAP_SYS_ADMIN`), its score is reduced by 30 points. The kernel assumes root processes are "more important" to system stability.

### 3. Manual Intervention: oom_score_adj

As a sysadmin or developer, you can influence the OOM Killer’s decision using the **`oom_score_adj`** file. 
- **The Range:** You can write a value from **-1000 to +1000**.
- **The Immune:** Writing **-1000** makes the process completely unkillable. The OOM Killer will skip it and look for the next highest score. This is commonly done for critical system daemons like `sshd` or `syslogd`.
- **The Target:** Writing **+1000** makes the process the first choice for termination, even if it’s using very little memory.

### 4. Cgroups: Isolation in the Lifeboat

In a world of Docker and Kubernetes, the OOM Killer has a more localized role. When you set a memory limit on a container, the kernel creates a **Memory Cgroup**. 

If the container exceeds its limit, the **Cgroup OOM Killer** fires. It only looks at the processes *inside* that specific container. It will kill the hungriest process in the container to bring it back under its limit, leaving the rest of the host system (and other containers) completely untouched.

```text
ASCII OOM Selection:
[ System RAM: 16GB (FULL) ]
      |
[ Victim Search ]
      |-- Chrome (4GB) -> Score: 250
      |-- JVM (8GB)    -> Score: 500
      |-- Postgres (root, 8GB) -> Score: 470 (500 - 30 discount)
      |
[ Winner: JVM ] --(SIGKILL)--> [ RAM RECLAIMED ]
```

### 5. Overcommit: The Root of the Problem

Why does the OOM Killer even need to exist? Why does the kernel allow apps to ask for more memory than it has? This is called **Optimistic Overcommit**. 

Linux assumes that when an app asks for 1GB of RAM, it won't actually use all of it immediately. By "over-promising" memory, Linux can run more applications simultaneously. You can change this behavior using `vm.overcommit_memory`:
- **0 (Heuristic):** The default. Be optimistic but use common sense.
- **1 (Always):** Always say yes. (Dangerous!)
- **2 (Never):** Only allow allocations that can be backed by physical RAM + swap. This prevents OOM situations but can make applications crash with "Out of Memory" errors even when RAM is technically available.

### Conclusion

The OOM Killer is a reminder of the brutal reality of physical constraints. In an ideal world, every application would manage its own memory perfectly. In the real world, memory is a shared and finite resource. The OOM Killer is the kernel's way of ensuring that a single misbehaving application cannot take down the entire city. It is a harsh, pragmatic gatekeeper—the ultimate arbiter of system survival.
