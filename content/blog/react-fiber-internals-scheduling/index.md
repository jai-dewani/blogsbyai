---
title: React is No Longer Recursive (and Why That Matters)
date: "2026-06-26T10:00:00.000Z"
description: "React Fiber rewritten the core engine to move away from the call stack and toward a linked list, making your UI interruptible for the first time."
---

We usually think of React as a declarative way to build UI, but under the hood, it’s a massive scheduling machine. Before React 16, the engine (the "Stack Reconciler") was a fairly straightforward recursive process. If you triggered a state change, React would walk down your component tree, figure out what changed, and update the DOM in one single, synchronous shot. The problem was that you couldn't stop it. If you had a huge component tree, the browser's main thread would be blocked until React was finished, leading to the dreaded "janky" UI where typing in an input felt like pulling teeth.

React Fiber changed everything by moving away from the JavaScript call stack and building its own "virtual stack" using a linked list. A "Fiber" is just a plain object that represents a unit of work. Because it’s a linked list (with `child`, `sibling`, and `return` pointers) instead of a recursive function call, React can now stop in the middle of a render, yield control back to the browser to handle a user click or an animation frame, and then come back right where it left off.

The engine uses a technique called Double Buffering. It maintains a "Current" tree (what you see on the screen) and a "Work-in-Progress" tree (what it's currently building). While React is busy crunching numbers in the background to build the new tree, the user still sees the old one. Once the work is done, React just swaps a single pointer to make the work-in-progress tree the current one. This ensures you never see a "partial" update where half the page is new and the other half is old.

The real magic is in the "Lanes" model. React assigns a priority level to every update. A user typing in a box gets a high priority lane, while a data fetch in the background gets a low priority lane. If React is busy rendering a massive list and you start typing, it pauses the list, handles your keystrokes, and then resumes the list. It's the difference between a train that has to reach the station before anyone can get off, and a car that can pull over whenever it needs to. We’re no longer fighting the browser; we’re cooperating with it.
