# TruthDB Roadmap

Last updated: 2026-07-24

What is built and what comes next. Legend: ✅ built and tested · ❌ not
started. Items under "Ahead" are in no committed order; each area gets its own
detailed plan document before work on it starts.

## Built

### The shared log

- ✅ Single-file storage on io_uring with an internal space allocator.
- ✅ Write-ahead log with group commit; the log is the source of truth, all
  other state is derived and rebuildable.
- ✅ ARIES-style crash recovery (redo/undo) and checkpoints.
- ✅ Backup: full backups, log backups, point-in-time restore, FULL/SIMPLE
  recovery models with copy-out-before-truncate log archiving.
- ✅ Physical replication: WAL shipping over TLS with authenticated handshake,
  replication slots, synchronous commit, readable standbys, standby
  restartpoints, and manual promotion with epoch (timeline) fencing.
- ✅ Replication monitoring: `sys.dm_repl_slots`, `sys.dm_repl_replica_states`,
  and a CLI status command.

### The engines

- ✅ The relational engine: SQL Server-compatible SQL dialect and TDS wire
  protocol (SSMS and JDBC clients work unmodified), ACID transactions with
  row-versioned isolation, constraints, security.
- ✅ Multiple databases as naming namespaces: `CREATE`/`DROP DATABASE`,
  `USE`, three-part `db.dbo.object` resolution and cross-database queries,
  real `sys.databases` rows, `DB_ID`/`DB_NAME` — plus the container tag on
  page-scoped log records, the groundwork for filtered subscriptions and
  per-namespace retention.
- ✅ The search engine: full-text search with an Elasticsearch-style query
  DSL, running as a second engine over the shared log — the working proof of
  the parallel-engines architecture.

## Ahead

### Search convergence

- ❌ `CREATE FULLTEXT INDEX` and `CONTAINS()` over relational tables; retire
  the legacy standalone search surface.

### The vector engine

- ❌ Collections of embeddings with approximate-nearest-neighbor indexes,
  quantization, and filtered search; specialized storage layout owned by the
  engine.
- ❌ Integration with relational rows (embeddings beside the data they
  describe, same transactions and backups) surfaced through SQL and the
  search DSL; possibly a dedicated wire protocol for high-throughput ingest
  and query.


### The streams engine

- ❌ Topics and durable subscriptions as filtered views over the shared log;
  change-data-capture from relational tables.
- ❌ Tiered log retention: the in-place log as hot tier, archived segments as
  warm tier (generalizing the recovery-model log archiver), per-namespace
  retention policies.
- ❌ Kafka wire-protocol compatibility, so existing producers and consumers
  connect unmodified.

### The ledger engine

- ❌ Double-entry accounts and transfers with TigerBeetle-style semantics:
  linked event chains committing atomically, domain-isolation rules enforced
  by the engine.
- ❌ Atomicity with the relational engine: a table update and a ledger
  transfer in one transaction.

### The temporal vertical

- ❌ `AS OF` queries: consistent point-in-time reads across tables and
  engines from one clock.
- ❌ Bitemporal views (valid time and transaction time) for pension
  administration and audit workloads.
- ❌ Deep-history retention built on the tiered log.
