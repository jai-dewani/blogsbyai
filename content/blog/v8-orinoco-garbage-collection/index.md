---
title: Orinoco: How V8 Cleans Your Room Without Stopping the Party
date: "2026-06-10T09:00:00.000Z"
description: "JavaScript memory management used to mean the whole browser would freeze for a second. V8's Orinoco project changed that by turning garbage collection into a background conversation."
---

If you’ve been writing JavaScript for more than a few years, you probably remember the "GC jank." You’d be scrolling a site or playing a browser game, and every few seconds, everything would just... stop. For a split second, the browser was frozen because the engine was busy cleaning up memory. Today, we don't notice it as much, not because JavaScript has gotten cleaner, but because the V8 engine’s garbage collector (codenamed **Orinoco**) has become incredibly sophisticated at multitasking.

### The Generational Hypothesis

V8’s memory management is built on a simple, observed truth in computer science: **Most objects die young.** 

Think about your code. You create a local variable inside a function, it does a quick calculation, and then the function returns. That object is now "garbage." V8 exploits this by splitting the heap into two main areas:
1.  **New Space (Young Generation):** This is where most objects are born. it's small (1-16MB) and optimized for high-speed allocation.
2.  **Old Space (Old Generation):** This is where objects go to retire if they survive long enough in the New Space. It’s much larger and harder to clean.

### Minor GC: The Parallel Scavenger

When the New Space fills up, V8 triggers a **Minor GC** using an algorithm called a **Scavenger**. 

The New Space is divided into two halves: **From-Space** and **To-Space**. When it's time to clean, V8 identifies the live objects in the From-Space and copies them to the To-Space. Everything left in the From-Space is deleted. 

In the old days, this was a synchronous "stop-the-world" event. In modern V8, it’s **Parallel**. V8 uses multiple helper threads to evacuate objects simultaneously. They use "forwarding pointers"—if one thread moves an object, it leaves a "moved to" note so other threads don't try to move it again. It’s like a team of movers emptying a room in five minutes instead of one person taking an hour.

```text
ASCII Scavenger Process:
[ From-Space (Full) ]        [ To-Space (Empty) ]
   Obj A (Live) -------------> Obj A
   Obj B (Dead)
   Obj C (Live) -------------> Obj C
[ From-Space (Wiped) ] <---- [ To-Space (New Home) ]
```

### Major GC: Mark-Sweep-Compact

The Old Space is a different beast. It can be gigabytes in size, so you can’t just copy everything to a new space—it would take way too long. Instead, V8 uses a **Mark-Sweep-Compact** strategy.

1.  **Marking:** V8 starts from the "roots" (global variables and the stack) and follows every pointer to find every reachable object. It uses **Tri-color marking** (White for unknown, Grey for discovered, Black for finished) to track its progress.
2.  **Sweeping:** It looks at the "White" objects (the ones it couldn't reach) and adds their memory addresses to a "Free List," essentially telling the system "you can put new data here."
3.  **Compaction:** Over time, the memory gets fragmented—lots of tiny holes between live objects. V8 moves the live objects together to create big, continuous blocks of free space. This is the most expensive part because it has to update every pointer in your code to point to the new location.

### Orinoco’s Master Stroke: Concurrent and Incremental

The real genius of Orinoco is how it hides these Major GC events.
-   **Incremental Marking:** Instead of marking the entire heap at once, V8 does a little bit of marking, then lets your JavaScript run for a few milliseconds, then does a little more. It breaks one long pause into hundreds of tiny ones that are shorter than a single animation frame (16ms).
-   **Concurrent Marking & Sweeping:** V8 offloads most of the marking and sweeping work to background threads entirely. While your code is busy updating the UI, helper threads are quietly traversing the object graph and identifying garbage.

### The Write Barrier: Keeping the Engine Honest

There’s a problem with concurrent marking: what if your code changes an object *while* the background thread is marking it? If the GC already marked an object as "finished" (Black), but your code then adds a pointer from that object to a "new" object (White), the GC might accidentally delete the new object.

To prevent this, V8 uses a **Write Barrier**. Every time you modify a pointer in JavaScript, a tiny piece of machine code runs to check if a GC cycle is active. If it is, it notifies the GC that the relationship has changed. It’s a tiny performance hit on every write, but it’s the only way to allow your code and the garbage collector to run at the exact same time safely.

### Why This Matters for Developers

Understanding V8’s GC explains why "memory leaks" in JavaScript are usually just "references you forgot to delete." If a global variable points to a massive object, that object is "reachable" from the roots, and the GC will never touch it. 

V8 has reached a point where the garbage collector is essentially invisible. It coordinates with Chrome to perform heavy cleaning only during "idle time" (like when you’re staring at a static page). It’s a distributed, multi-threaded cleaning crew that knows exactly when to stay out of your way. We've moved from a world where memory management was a hurdle to a world where it’s a high-performance feature of the runtime.
