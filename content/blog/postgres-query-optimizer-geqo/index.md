---
title: Why Postgres Thinks Your Join is a Traveling Salesman Problem
date: "2026-06-25T09:00:00.000Z"
description: "When your SQL query gets too complex, PostgreSQL stops trying to be perfect and starts acting like an evolving organism."
---

We've all written that one monster SQL query with twelve or thirteen joins and just hoped for the best. Most of the time, we treat the database like a black box that just works. But if you've ever noticed your query plan suddenly shift from "optimized" to "weirdly inconsistent" as you add more tables, you've probably run into the Genetic Query Optimizer (GEQO). It’s one of the most fascinating and misunderstood parts of the Postgres internals, and it’s where the database stops being a rigid calculator and starts acting like a biological simulator.

The core problem is the math of joins. If you have a query with three tables, there are only a few ways to join them. But as you add more tables, the number of possible join orders explodes exponentially. By the time you hit 12 tables, the number of permutations is over 479 million. If Postgres tried to exhaustively check every single one of those combinations—what we call the "System R" approach—your query would take longer to plan than it would to actually execute. To avoid this "planning explosion," Postgres sets a default threshold (geqo_threshold) at 12 tables. Once you cross that line, it switches gears.

Instead of trying to find the absolute perfect plan, GEQO treats your query like a population of organisms. Each possible join order is encoded as a "chromosome." It starts with a random set of these join orders and then lets them "evolve." It uses a technique called Edge Recombination Crossover to take two "parent" join plans and create "offspring" that preserve the best parts of both. It calculates the "fitness" of each plan using its standard cost estimation—looking at table statistics and CPU costs—and keeps the cheapest ones.

This leads to a weird side effect: non-determinism. Because GEQO uses a randomized genetic algorithm, the same monster query might result in a slightly different execution plan every time you run it. It’s not looking for the *best* plan anymore; it’s looking for a "good enough" plan that it can find before you get bored and cancel the query. It’s a pragmatic trade off that most developers don't even realize is happening under the hood.

If you’re working on modern hardware with plenty of memory, that default threshold of 12 might actually be too low. Many DBAs bump it up to 14 or 16 to let the exhaustive planner work a little longer, because the cost of a sub-optimal "genetic" plan is often much higher than an extra 50 milliseconds of planning time. But the next time you write a join that spans half your schema, just remember that somewhere deep in the Postgres source code, your query is busy evolving like a digital Darwinian experiment.
