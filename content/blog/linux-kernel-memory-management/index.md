---
title: "The Buddy and the Slab: How Linux Manages Your RAM"
date: "2026-06-10T09:00:00.000Z"
description: ""Linux memory management is a multi-layered masterpiece that balances the needs of high-level processes with the brutal reality of hardware fragmentation.""
---

If you’ve ever looked at a memory graph in your dashboard and wondered how the Linux kernel keeps track of billions of bytes without losing its mind, you’re looking at one of the most sophisticated pieces of engineering in existence. The kernel doesn't just "allocate memory." It manages a complex hierarchy of abstractions, from virtual address spaces to physical page blocks, using two main workhorses: the **Buddy System** and the **Slab Allocator**.

### The Virtual Illusion

Every process on your system thinks it owns the entire memory of the computer. It sees a flat, continuous range of addresses from zero to several terabytes. This is **Virtual Memory**. In reality, your RAM is a fragmented mess of physical pages.

The hardware (the **MMU**) and the kernel work together to maintain **Page Tables**. These are hierarchical maps that translate a virtual address into a physical one. Modern 64-bit Linux uses a 4-level or 5-level page table system. It’s like a giant phone book where you look up a name (the virtual address) to find a number (the physical page). If the number isn't there, the CPU triggers a "Page Fault," and the kernel has to go find some physical RAM to back up that virtual promise.

### The Buddy System: Managing the Physical Blocks

At the lowest level, the kernel needs to allocate chunks of physical RAM. But if you just allocate bytes wherever you find them, you'll eventually end up with "External Fragmentation"—lots of tiny holes of free memory that are too small to be useful.

To fix this, Linux uses the **Buddy System**. It organizes free physical memory into lists of blocks, where each block size is a power of two (4KB, 8KB, 16KB, etc.). 
- **Allocation:** If you need 16KB but the kernel only has a 32KB block, it splits that block into two 16KB "buddies." One goes to you, and the other goes into the 16KB free list.
- **Freeing:** When you give that 16KB back, the kernel checks if its "buddy" is also free. If it is, they "coalesce" back into a single 32KB block. 

This simple recursive logic ensures that large contiguous blocks of memory are preserved whenever possible.

```text
ASCII Buddy System Split:
[ 32KB Block ] 
      |
      v (Split)
[ 16KB Buddy A ] [ 16KB Buddy B ]
      |
      v (Split A)
[ 8KB A1 ] [ 8KB A2 ]
```

### The Slab Allocator: Managing the Small Stuff

The Buddy System is great for large chunks, but it only works in units of "Pages" (usually 4KB). What if the kernel needs to create a tiny 128-byte object, like a file descriptor or a process structure (`task_struct`)? Using a whole 4KB page for a 128-byte object would be a massive waste (Internal Fragmentation).

The **Slab Allocator** (specifically the **SLUB** implementation in modern Linux) sits on top of the Buddy System. It requests a few pages from the Buddy System and carves them into a "Slab"—a pre-formatted array of equal-sized slots for a specific type of object.

There are dedicated slabs for common kernel objects. When the kernel needs a new `inode`, it doesn't calculate where to put it; it just asks the "inode cache" for the next free slot in its current slab. It’s incredibly fast because the memory is already allocated and the cache is "hot."

### SLAB vs. SLUB vs. SLOB

Linux has experimented with different slab designs over the years:
1.  **SLAB:** The original design. It used complex queues and per-CPU caches to minimize locking, but it became too heavy as core counts exploded.
2.  **SLUB:** The modern standard. It simplified the metadata by moving it into the `page` structure itself. It scales better on many-core (NUMA) systems and is the default in almost all modern distributions.
3.  **SLOB:** A "Simple List Of Blocks" designed for tiny embedded systems with very little RAM. It was recently removed from the kernel because SLUB has become efficient enough even for small devices.

### The Lifecycle of a Memory Request

When you call `malloc` in your application:
1.  The C library (`glibc`) asks the kernel for a large chunk of virtual memory via a syscall like `brk` or `mmap`.
2.  The kernel updates your process's **Page Tables** but doesn't actually give you physical RAM yet (Lazy Allocation).
3.  The first time you *write* to that memory, the CPU triggers a **Page Fault**.
4.  The kernel wakes up, looks at the **Buddy System** for a free physical page, and maps it into your page table.
5.  If the request was for a small kernel-internal object, the kernel bypasses this and uses the **Slab Allocator** directly.

### Conclusion

Linux memory management is a study in pragmatic trade-offs. The Page Tables provide isolation, the Buddy System provides contiguous physical blocks, and the Slab Allocator provides high-speed object management. It’s a system that manages to be both generic enough to run on a toaster and specialized enough to power the world's fastest supercomputers. The next time your application runs without a hitch, remember the silent "buddies" in the kernel making sure every byte is exactly where it needs to be.
