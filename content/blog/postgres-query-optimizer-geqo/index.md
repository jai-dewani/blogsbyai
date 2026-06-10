---
title: "Why Postgres Thinks Your Join is a Traveling Salesman Problem"
date: "2026-06-10T09:00:00.000Z"
description: ""When your SQL query gets too complex, PostgreSQL stops trying to be perfect and starts acting like an evolving organism.""
---

We've all written that one monster SQL query with twelve or thirteen joins and just hoped for the best. Most of the time, we treat the database like a black box that just works. We assume that because we used the correct syntax, the engine will find the most efficient way to fetch our data. But if you've ever noticed your query plan suddenly shift from "optimized" to "weirdly inconsistent" as you add more tables, you've probably run into the Genetic Query Optimizer (GEQO). It’s one of the most fascinating and misunderstood parts of the Postgres internals. This is the part of the source code where the database stops being a rigid calculator and starts acting like a biological simulator.

### The Combinatorial Explosion

The core problem is the sheer math of joins. If you have a simple query with three tables, there are only a few ways to join them. But as you add more tables, the number of possible join orders explodes exponentially. It’s not just a linear increase. By the time you hit 12 tables, the number of permutations is over 479 million. If Postgres tried to exhaustively check every single one of those combinations—what we call the "System R" approach or dynamic programming—your query would take longer to plan than it would to actually execute. 

Your database would spend minutes just thinking about how to run a query that might only take a few seconds to finish. This is the classic "Planning vs. Execution" trade-off. To avoid this "planning explosion," Postgres sets a default threshold called `geqo_threshold` at 12 tables. Once you cross that line, it switches gears entirely.

```text
Permutation Math:
- 3 Tables: 6 orders
- 5 Tables: 120 orders
- 8 Tables: 40,320 orders
- 10 Tables: 3,628,800 orders
- 12 Tables: 479,001,600 orders (GEQO Kicks in here)
```

### The Genetic Petri Dish

Instead of trying to find the absolute perfect plan, GEQO treats your query like a population of organisms in a digital petri dish. It treats the problem of join ordering as a variation of the Traveling Salesman Problem (TSP). The goal is to find the shortest path through all the tables while minimizing the "cost" of the joins. Each possible join order is encoded as a "chromosome," which is basically just a string of table IDs. 

The algorithm starts with a random set of these join orders—a starting population. Then, it lets them "evolve" through successive generations. The "fitness" of each plan is determined by the standard Postgres cost estimator. It looks at table statistics, index availability, disk I/O costs, and CPU costs for nested loops or hash joins. The "fittest" plans—the ones with the lowest estimated cost—survive to the next generation, while the expensive ones are discarded.

### Edge Recombination Crossover

To manage this evolution, GEQO uses a specialized technique called Edge Recombination Crossover (ERX). In a standard genetic algorithm, you might just swap parts of two chromosomes. But for join ordering, you can't just swap numbers because each table must appear exactly once. ERX is designed to preserve the "edges" or the specific pairs of tables joined together in the parent plans.

Imagine two parent plans:
Parent A: 1-2-3-4-5
Parent B: 4-3-1-5-2

The ERX algorithm builds an adjacency list for each table based on these parents and then constructs a child plan that attempts to keep high-value edges together. If Table 1 and Table 2 were joined efficiently in Parent A, the algorithm tries to keep them adjacent in the child. This mimics how successful traits are passed down in biological evolution.

```text
ASCII Visualization of ERX (Simplified):
Parent A: [A-B] [B-C] [C-D]
Parent B: [D-A] [A-C] [C-B]

Adjacency List for A: B, D, C
Adjacency List for B: A, C
...
Child: Starts at a random node, picks the neighbor with the fewest further neighbors.
Goal: Preserve the 'neighborhoods' that yielded low-cost joins in the parents.
```

### The Cost of Non-Determinism

This leads to a weird side effect that can drive DBAs crazy: non-determinism. Because GEQO uses a randomized genetic algorithm, the same monster query might result in a slightly different execution plan every time you run it. You might run an `EXPLAIN ANALYZE` and get a great plan, then run the exact same query five minutes later and see it crawl to a halt because the genetic algorithm happened to settle on a slightly worse "good enough" plan during that specific run. 

It’s not looking for the *best* plan anymore; it’s looking for a plan that is statistically likely to be efficient enough to run without crashing the server. This is why for mission-critical queries with many joins, some developers choose to use explicit `JOIN` syntax and set `join_collapse_limit` to 1, effectively forcing the database to use the order written in the SQL, bypassing the optimizer's creativity entirely.

### Tuning the Effort

If you’re working on modern hardware with plenty of memory and fast NVMe drives, that default threshold of 12 might actually be too low for your needs. Many experienced DBAs bump it up to 14 or even 16. The reasoning is simple: the cost of a sub-optimal "genetic" plan is often much higher than the extra 50 or 100 milliseconds of planning time required for an exhaustive search. 

You can adjust this by changing the `geqo_threshold` in your `postgresql.conf` file. You can also tweak the `geqo_effort` setting, which controls how many generations the algorithm runs and how large the population is. 

| Parameter | Impact | Trade-off |
|-----------|--------|-----------|
| `geqo_threshold` | When to switch to GEQO | Higher = More exhaustive search, longer planning |
| `geqo_effort` | Population size and generations | Higher = Better plans, much longer planning |
| `geqo_pool_size` | Number of individuals in the population | Higher = More diversity, more memory usage |

### The Philosophy of Pragmatism

The existence of GEQO is a reminder that engineering is often the art of being "good enough." We've built a system that recognizes its own limits. When the math becomes too big for even a computer to solve in a reasonable timeframe, we fall back on the oldest trick in the book: evolution. 

It might be messy, and it might be inconsistent, but it’s the only way to keep the lights on when the data gets truly complex. The next time you write a join that spans half your schema, just remember that somewhere deep in the Postgres source code, your query is busy competing for survival. We aren't just calculating data; we're simulating life to find the path of least resistance.
