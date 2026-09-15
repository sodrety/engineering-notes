---
title: 'PostgreSQL-to-ClickHouse CDC Without the Usual Logical-Replication Pipeline'
description: 'WalShadow reads PostgreSQL physical WAL and writes ClickHouse-native blocks. The shorter path can reduce CDC latency, but it also moves correctness and compatibility closer to PostgreSQL internals.'
pubDate: 2026-09-15
---

Most PostgreSQL-to-ClickHouse CDC systems follow a familiar path:

```text
PostgreSQL
    ↓
Logical replication slot
    ↓
CDC connector
    ↓
Serialized events
    ↓
Kafka or another buffer
    ↓
Transform and normalize
    ↓
ClickHouse
```

That architecture is flexible, but every stage adds work and latency. ClickHouse's new `walshadow` project takes a different route: it reads PostgreSQL's physical WAL and turns the changes directly into ClickHouse-native blocks.

The result is faster in ClickHouse's benchmark. It also creates a tighter relationship with PostgreSQL internals.

## Why this matters now

ClickHouse announced WalShadow on September 10, 2026. In its benchmark, transactions became visible in ClickHouse after roughly 200 milliseconds. The replication process sustained about 289,000 rows per second, close to the source PostgreSQL rate of approximately 290,000 rows per second.

For comparison, the same test reported about 10 seconds of commit-to-visible latency and roughly 120,000 rows per second for PeerDB.

Those are vendor benchmarks, not production guarantees. The test used one table, same-region services, and `c8i.2xlarge` instances with eight vCPUs each. A workload with updates, deletes, large rows, multiple tables, frequent DDL, or remote regions could behave very differently.

The interesting part is the architecture, not the headline number.

## Physical WAL removes pipeline stages

PostgreSQL logical replication produces changes at the logical row level. That is useful because downstream systems can understand inserts, updates, deletes, table filters, and replication identities.

Physical WAL is lower-level. It records changes to PostgreSQL's storage structures. A physical WAL consumer cannot decode a tuple correctly without knowing the catalog state that existed when PostgreSQL wrote it.

WalShadow solves this with a shadow PostgreSQL process. The shadow process does not store ordinary user-table contents. It replays the relevant catalog and recovery state so WalShadow can understand table layouts, PostgreSQL types, relation rewrites, and schema changes over time.

<figure>
  <img src="https://assets.deram.my.id/assets/pg-wal.png" alt="Architecture diagram showing PostgreSQL physical WAL flowing to parallel row decoders, bounded queues, a batch builder, and parallel ClickHouse inserters, with a shadow PostgreSQL process providing schema history and ordering barriers for DDL and truncation." loading="lazy" decoding="async" />
  <figcaption>WalShadow's direct path keeps physical WAL for row decoding while a shadow PostgreSQL process supplies schema history. The orange boundary shows where ordering still matters.</figcaption>
</figure>

The data path can then decode table records outside the shadow database and send them directly to ClickHouse:

```text
                         ┌─────────────────────┐
                         │  Shadow PostgreSQL   │
                         │  catalogs + recovery│
                         └──────────┬──────────┘
                                    │ schema history
PostgreSQL physical WAL ────────────┼───────────────┐
                                    │               │
                                    ▼               ▼
                           Parallel decoders   Schema events
                                    │
                                    ▼
                             Batch builder
                                    │
                                    ▼
                       ClickHouse-native blocks
                                    │
                                    ▼
                           Parallel inserters
```

That removes several common stages:

- No logical decoding output plugin
- No Kafka requirement
- No JSON serialization step
- No separate normalization stage

The trade-off is that the replication engine now needs to understand PostgreSQL's physical storage and version-specific behavior.

## Parallelism creates an ordering problem

WalShadow uses separate worker pools for decoding and ClickHouse insertion. That is how it approaches the source PostgreSQL throughput without forcing one serial consumer to process every change.

But parallel processing means blocks can arrive out of order:

```text
Transaction A ── decode ─────────────── insert
Transaction B ── decode ── insert
```

Transaction B may reach ClickHouse before transaction A even if A committed first.

WalShadow attaches the source WAL position to each row through `_lsn`. That gives ClickHouse a source-side ordering signal and lets the destination identify newer versions of a row.

Some operations cannot safely run out of order. Schema changes and truncations introduce barriers. The system waits for preceding data to become durable before applying those operations.

This is the part that deserves more attention than the throughput chart. Low latency is useful only if the destination remains correct during:

- Updates to the same primary key
- Deletes
- Multi-row transactions
- DDL changes
- Truncation
- Restarts
- Source failover
- Rows whose PostgreSQL layout changed over time

An LSN column does not automatically define the correct analytical read model. The ClickHouse table design still needs an explicit policy for updates and deletes. `_lsn` gives the destination a source ordering signal; it does not replace the choice of table engine, merge behavior, or query semantics.

## The operational boundary moves

The usual logical CDC pipeline has more moving parts, but those parts are familiar. Kafka can buffer events. Connectors can transform them. Logical replication can work across PostgreSQL major versions and can replicate selected tables or columns.

Physical-WAL replication reduces the number of components, but it increases coupling to the source.

The current `walshadow` repository lists these requirements:

- PostgreSQL 16 or newer
- The source and WalShadow shadow instance must use the same PostgreSQL major version
- The PostgreSQL module is loaded through `shared_preload_libraries`
- The project is marked **Experimental** and **Development Preview**

That last point matters. This is not yet a drop-in replacement for a mature production CDC stack.

A managed PostgreSQL service may also limit the configuration required to load the module. Cross-major-version migrations, heterogeneous PostgreSQL environments, or pipelines that need substantial transformation may still be better served by logical CDC.

## A practical evaluation plan

Do not start by copying the reported 200 ms number. Start by testing whether the design fits your workload.

### 1. Define the freshness target

Measure the time between:

```text
PostgreSQL commit timestamp
        →
row available under the ClickHouse query contract
```

Use p50, p95, and p99 latency. "Sub-second" is not a useful SLO if the slowest transactions take several seconds to appear.

### 2. Test the source constraints

Confirm that you can run the required PostgreSQL version and preload module. Check how the system behaves when replication is delayed:

- How much WAL accumulates?
- Where is the retention limit?
- What happens if ClickHouse is unavailable?
- Can the replication process resume from its last durable position?

A CDC outage must not silently consume the PostgreSQL disk through unbounded WAL retention.

### 3. Test correctness before throughput

Use a production-like clone and exercise:

- Inserts, updates, and deletes
- Repeated updates to one key
- Large and toasted values
- Multi-row transactions
- Column additions, renames, and drops
- Table rewrites
- Truncation
- Replication restarts during a transaction
- PostgreSQL failover and timeline changes

Compare source and destination data using row counts, key samples, checksums where practical, and application-level queries.

### 4. Measure each queue

The architecture uses bounded queues and a shared payload budget to limit work in flight. Monitor:

- Decoder queue depth
- Batch-builder delay
- ClickHouse insertion backlog
- WAL position received
- WAL position decoded
- WAL position durably visible
- Memory usage and disk spill
- Source PostgreSQL CPU and I/O
- ClickHouse insert latency

A single "replication lag" number hides where the delay actually occurs.

### 5. Rehearse rollback

Keep the old CDC path or a recoverable source snapshot until the new path has passed a real cutover rehearsal.

Decide in advance:

- How to stop new writes
- How to identify the last consistent LSN
- How to handle rows already written to ClickHouse
- How to resume after a failed migration
- How to switch analytical reads back

## When this design makes sense

WalShadow is worth evaluating when you need:

- Very fresh analytical data
- A direct PostgreSQL-to-ClickHouse path
- High ingest rates
- Minimal transformation between source and destination
- A PostgreSQL environment where the same-major-version constraint is acceptable

A logical CDC pipeline remains the safer default when you need portability, mature operational tooling, cross-version replication, rich filtering, or extensive event transformation.

## Takeaway

Physical-WAL CDC is not simply "logical replication, but faster." It changes the boundary of the system.

You remove serialization, buffering, and normalization stages. In return, you take responsibility for PostgreSQL-version coupling, catalog history, ordering barriers, WAL retention, destination semantics, and recovery testing.

That can be a good trade for a low-latency analytical mirror. It is a poor trade if the team treats the benchmark as a product guarantee or skips the correctness work.

WalShadow's current status says exactly how to approach it: test it seriously, but treat it as a development preview until your own workload proves otherwise.

## References

- [ClickHouse: Introducing WalShadow](https://clickhouse.com/blog/introducing-walshadow) — announcement and benchmark, September 10, 2026
- [WalShadow repository](https://github.com/ClickHouse/walshadow) — project status, requirements, and build instructions
- [WalShadow architecture](https://github.com/ClickHouse/walshadow/tree/main/architecture) — shadow PostgreSQL, bounded parallel pipeline, and ordering model
- [PostgreSQL logical replication documentation](https://www.postgresql.org/docs/current/logical-replication.html) — logical versus physical replication concepts
