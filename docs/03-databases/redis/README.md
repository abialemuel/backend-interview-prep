# Redis

This section covers Redis (Remote Dictionary Server) for backend engineering
interviews. It targets the **Redis 8.x line** (Redis Open Source), with the
**Valkey** fork called out where it diverges.

## Scope and version

| Item                 | Value                                                                 |
|----------------------|-----------------------------------------------------------------------|
| Redis covered        | 8.x ("Redis Open Source"; 8.0 GA May 2025, point releases through 2026) |
| License (Redis 8+)   | Tri-license: **AGPLv3** (OSI-approved open source) / RSALv2 / SSPLv1  |
| Open-source fork     | **Valkey** (Linux Foundation) — BSD-3-Clause; 8.x/9.x releases        |
| Redis Enterprise     | Commercial: Active-Active (CRDT) geo-distribution, managed clustering |

> Licensing history — a genuine 2026 interview topic. In **March 2024** Redis Ltd.
> moved Redis 7.4+ from BSD-3 to dual SSPL/RSALv2 (not OSI open source), which
> triggered the **Valkey** fork (AWS, Google, Oracle, Ericsson, Snap under the
> Linux Foundation). In **May 2025**, Redis 8.0 added **AGPLv3** as a third
> license option, making Redis OSI open source again — and folded the former
> Redis Stack modules (JSON, Redis Query Engine, time series, probabilistic
> structures) plus the new **vector set** type into the core distribution.
> Valkey continues independently under BSD-3 with its own roadmap (notably
> enhanced I/O threading throughput, and in Valkey 9.0 atomic slot migration
> and multi-database cluster mode); most cloud "Redis-compatible" services
> (e.g., ElastiCache, Memorystore) now run Valkey. Core concepts, data
> structures, commands, and protocols here apply to both unless noted.

## Files in this section

| File                                  | Description                                                                       |
|---------------------------------------|-----------------------------------------------------------------------------------|
| `01-data-structures-and-use-cases.md` | Core data types, internal encodings, use cases, Redis 8 additions (vector sets, JSON, probabilistic), expiration, eviction, scripts vs functions, transactions. |
| `02-persistence-and-clustering.md`    | RDB, AOF, hybrid persistence, Sentinel, Cluster, Active-Active/CRDT, replication, ops concerns. |
| `03-interview-questions.md`           | 35+ interview questions graded junior/senior/staff with model answers.            |

## Recommended reading order

1. `01-data-structures-and-use-cases.md` — establish what Redis *is* and how
   data is represented internally. Most interview questions are rooted here.
2. `02-persistence-and-clustering.md` — how Redis survives restarts and scales
   out. Sentinel vs Cluster trade-offs show up constantly in system design.
3. `03-interview-questions.md` — use this at the end to self-test, or as a
   quick index of topics to revisit.

Redis often appears in interviews as the cache in front of a relational
database — pair this section with [MySQL](../mysql/README.md) or
[PostgreSQL](../postgresql/README.md) for the full caching + source-of-truth
story.

## How to run examples

All `redis-cli` examples assume a local Redis (or Valkey) server on the default
port `6379`. Start one with:

```bash
redis-server --port 6379 --save "" --appendonly no   # in-memory only, dev
```

Relevant cluster-level examples (e.g., `CLUSTER MEET`) are safe to skip if you
only have a single-node instance.
