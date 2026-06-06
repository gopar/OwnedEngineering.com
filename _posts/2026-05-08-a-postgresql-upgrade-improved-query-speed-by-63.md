---
title: A PostgreSQL Upgrade Improved Query Speed By 63%
date: 2026-05-08
categories: [postgresql]
tags: []
description: How an overdue upgrade unlocked faster performance.
---

We upgraded PostgreSQL 12 → 17 and TimescaleDB `1.7.5` → `2.21.0`.\
No code changes, and no new indexes.

Result:
- Cold queries: ~41% faster
- Warm queries: ~63% faster

## Why the upgrade
Originally this task of upgrading was in the backlog for over 2 years because it was deemed "low priority" at the
time. What finally caused change was that since the PostgreSQL version was 12, it was now labeled as a security risk by
the internal security team.

## How we measured results
Since the project is using Django, we ended up using a pre-existing `manage.py` bench-marking script that did a
"good-enough" job of fetching important and related models that the company cared about.

For those who have done database bench-marking, know that this isn't a "clean" bench-marking method (Python garbage
collecting, Django ORM caching, network latency, etc can all affect results), but for this project it served as more
than enough for our purposes.

Also we tried to circumvent as much noise as we could on the database server side by syncing/flushing OS cache,
restarting the server between runs, etc.

Anyways, here are the actual numbers of the upgrade (multi-step):

<figure markdown="1">
  <figcaption><strong>Table 1:</strong> Benchmark Results (seconds)</figcaption>

| Name        | PG 12 - 1.7.5 | PG 12 - 2.10.3 | PG 15 - 2.10.3 | PG 15 - 2.21.0 | PG 17 - 2.21.0 |
|-------------|---------------|----------------|----------------|----------------|----------------|
| Linear Warm |          8.42 |           6.17 |           3.14 |           3.30 |           3.14 |
| Linear Cold |         34.34 |          22.25 |          19.66 |          20.33 |          20.20 |

</figure>

As you can see, we got most of our speed boost when we went from PG 12 with TimescaleDB `1.7.5` -> PG 15 TimescaleDB
`2.10.3`

The benchmark did include random cold/warm numbers, but since we didn't have a way to properly test the randomness (aka
a seed of some sort), the numbers are not as reliable so they are not shown here.

## Multi-step upgrade process
For those readers who are curious why a multi-step happened, I will explain.

Our bottleneck for upgrading was TimescaleDB. The highest version we could go to was `2.10.3` without leaving PG 12.
Next we upgraded PG 12 -> PG 15 while keeping the same TimescaleDB version. Third we upgrade to `2.21.0` while being in
PG 15. Finally we upgrade PG 15 -> PG 17. And per each upgrade doing the benchmarks as we go (plus other house keeping)

So in the end, upgrade path looked like this:
- Upgrade TimescaleDB `1.7.5` -> `2.10.3`
- Upgrade PG 12 -> PG 15
- Upgrade TimescaleDB `2.10.3` -> `2.21.0`
- Upgrade PG 15 -> PG 17

See [TimescaleDB Documentation](https://www.tigerdata.com/docs/self-hosted/latest/upgrades/major-upgrade)

##  Takeaway
This post shows the upsides of upgrading your database (security patches + performance) and shows that you don't need
perfect instrumentation to create benchmarks as long as you are benchmarking the correct things (In this case what was
most important was the fetching of models and their relations)

<!-- * Linkedin -->
<!-- #+begin_quote -->
<!-- We upgraded PostgreSQL 12 → 17 and Timescale 1.7.5 → 2.21.0. -->

<!-- No code changes. -->
<!-- No new indexes. -->

<!-- Result: -->
<!-- - Cold queries: ~41 faster -->
<!-- - Warm queries: ~63% faster -->

<!-- Optimization doesn't always mean optimizing query :) -->

<!-- More details at ..... -->
<!-- #+end_quote -->
