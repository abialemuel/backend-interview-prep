# Redis

This section covers Redis (Remote Dictionary Server) for backend engineering
interviews. It focuses on the Redis 7.x line, in particular **Redis 7.4** and the
**Valkey** fork maintained by the Linux Foundation.

## Scope and version

| Item               | Value                                                       |
|--------------------|-------------------------------------------------------------|
| Redis covered      | 7.4 (latest 7.x release tracked here)                       |
| Open-source fork   | **Valkey** (Linux Foundation) — compatible fork from 7.4   |
| License (Redis 7.4)| **SSPL/RSALv2** (Server Side Public License + Redis Source Available License v2) |
| License (Valkey)  | BSD-3-Clause (the historical Redis license)                 |

> In March 2024 Redis Ltd. changed the license of Redis 7.4+ from BSD-3 to a
> dual SSPL/RSALv2 scheme, meaning Redis is *no longer OSI open-source*. Cloud
> providers and most modern distributions now ship **Valkey**, the BSD-licensed
> fork created by AWS, Google, Oracle, Ericsson and Snap under the Linux
> Foundation. Concepts, data structures, commands, and protocols discussed
> here apply to both Redis 7.x and Valkey, unless noted otherwise.

## Files in this section

| File                                  | Description                                                                       |
|---------------------------------------|-----------------------------------------------------------------------------------|
| `01-data-structures-and-use-cases.md` | Core data types, internal encodings, use cases, expiration, eviction, scripts, transactions. |
| `02-persistence-and-clustering.md`    | RDB, AOF, hybrid persistence, Sentinel, Cluster, replication, ops concerns.       |
| `03-interview-questions.md`           | 25+ interview questions grouped by Easy / Medium / Hard with model answers.       |

## Recommended reading order

1. `01-data-structures-and-use-cases.md` — establish what Redis *is* and how
   data is represented internally. Most interview questions are rooted here.
2. `02-persistence-and-clustering.md` — how Redis survives restarts and scales
   out. Sentinel vs Cluster trade-offs show up constantly in system design.
3. `03-interview-questions.md` — use this at the end to self-test, or as a
   quick index of topics to revisit.

## How to run examples

All `redis-cli` examples assume a local Redis (or Valkey) server on the default
port `6379`. Start one with:

```bash
redis-server --port 6379 --save "" --appendonly no   # in-memory only, dev
```

Relevant cluster-level examples (e.g., `CLUSTER MEET`) are safe to skip if you
only have a single-node instance.