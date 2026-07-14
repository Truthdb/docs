# TruthDB: Staged Plan for Relational (SQL-Server-like) Functionality

## Context

TruthDB today implements only the start of Elasticsearch-like functionality (in-memory indices with snapshot+WAL persistence). The goal is full relational database functionality with SQL Server as the functional reference, delivered as a staged plan where **every stage compiles, passes tests, and is demonstrable end-to-end**, and the existing search functionality never regresses. The docs/ repo is deliberately disregarded.

## Decisions (all confirmed by user, 2026-07-11)

1. **Scope: full SQL Server parity, all planned in detail** (revised from the earlier "core + full SQL surface" answer): core relational engine (tables/types, CRUD, joins, aggregates, secondary indexes, ACID transactions), constraints (PK/FK/UNIQUE/CHECK/DEFAULT/NOT NULL), views, subqueries, prepared statements, **and** stored procedures/triggers/UDFs, security (logins/users/roles/permissions), backup & restore, and replication/HA — the last four as detailed stages 15–18, not outline.
2. **Hand-rolled T-SQL-flavored parser** — no sqlparser-rs.
3. **Paged on-disk table storage from day one** in the existing data region.
4. **TDS early, and TDS is *the* SQL interface**: real SQL Server tools (pymssql/go-mssqldb, later sqlcmd, SSMS query window) become the daily test harness as soon as basic SQL works. The native protocol stays a minimal dev vehicle (JSON text in `CommandResp`) and keeps carrying the ES-search surface; no native "protocol v2" is built.
5. **Recovery: full ARIES** — steal/no-force buffer management, undo logging, CLRs, fuzzy checkpoints. No transaction-size limits.
6. **Table layout: clustered index = table** (SQL Server faithful): a table with a PK is stored as a B+ tree on the PK; secondary indexes carry PK locators. PK-less tables are heaps with RID locators.
7. **Concurrency: both mechanisms, locking first** — lock-based 2PL engine first (READ UNCOMMITTED / READ COMMITTED / REPEATABLE READ / SERIALIZABLE), then a dedicated stage adds a row-version store → RCSI and SNAPSHOT. End state: all five SQL Server isolation levels.
8. **Spilling in scope (SQL Server faithful)**: sort/hash operators spill to temp disk space rather than erroring.
9. **Collations: per-column via icu4x** — full plumbing (per-column collation, `COLLATE`, precedence rules) with comparison rules from the pure-Rust icu4x library (incl. Swedish/Finnish rules). Justified dependency, like rustls for TDS TLS.

**Stated assumptions (veto any):** Volcano pull-iterator executor; rule-based planner only (no cost model/histograms in detailed scope); SQL-Server-compatible error numbers from day one (102, 208, 245, 515, 547, 1205, 2627, 3902…); GO is a client-side batch separator; MARS permanently rejected; rustls for TDS TLS; single database per instance (`USE truthdb` accepted) and single `dbo` schema in detailed scope; TDS auth from config-file users only until Stage 16 migrates them into catalog logins with hashed passwords; native protocol v1 messages byte-frozen.

## Verified current state (grounding)

- Crates: `truthdb` (server bin), `crates/truthdb-core` (engine + storage), `truthdb-proto`, `truthdb-net`, `truthdb-cli`, `truthdb-bench`. Linux-only; Docker for build/test on macOS (per CLAUDE.md).
- **Protocol**: 8-byte BE header (len u32 | msg_type u16 | flags u16) + bincode payload, TCP 9623; msg types 1–6 only. Everything rides `CommandReq{id, command: String}` → `CommandResp{id, ok, message: String}`. Unknown msg types silently dropped (`dispatcher.rs:83`) hanging clients; 8 MiB frame cap.
- **Engine** (`truthdb-core/src/engine.rs`): text commands `create index` / `insert document` / `search` (ES DSL: match/term/bool). All state in BTreeMaps; persistence = full-state bincode snapshot + JSON WAL event replay.
- **Storage** (`storage.rs`, `storage_layout.rs`, `direct_io.rs`, `allocator.rs`): single preallocated file, 4096 B pages, O_DIRECT + io_uring (`DirectFile`, sync positioned I/O). Regions: header | superblocks A/B | WAL ring (256 MiB–1 GiB, CRC'd entries) | data | metadata (unused) | allocator bitmap | snapshot descriptors ×2 | reserved. `WAL_ENTRY_TYPE_COMMIT=2` reserved and already parsed; superblock reserves `metadata_root`/`data_root`.
- **Critical constraints found**:
  - Data region is monopolized by snapshot A/B *half-region slots* — `write_checkpoint` frees/reallocates half the region every checkpoint (`storage.rs:673-684`). Relational pages would be clobbered; fixing this is task #1.
  - Allocator bitmap durable only at checkpoint; between-checkpoint allocations need WAL logging.
  - WAL appends read-modify-write shared tail pages (`DirectFile::write_all_at`) — torn rewrite can destroy previously-fsynced entries (latent durability bug).
  - One fsync + superblock write **per WAL append** (`storage.rs:416-417`).
  - One global `Arc<Mutex<Engine>>`; synchronous direct-I/O inside async tasks (blocks the tokio reactor); no session state anywhere.

## Cross-cutting architecture

- **Crate/module layout**: new `crates/truthdb-sql` (pure: lexer, parser, AST with spans, type system, `Datum`, 3VL evaluator, binder via `CatalogProvider` trait, logical planner, SQL error codes) · `truthdb-core/src/relstore/` (pages, buffer pool, B+ tree, heap, row/key codecs) · `truthdb-core/src/wal/` + `txn/` (ARIES log manager, transaction/lock managers) · `truthdb-core/src/rel/` (catalog, physical planner, executors, session, results) · new `crates/truthdb-tds` (TDS frontend).
- **Command routing**: legacy ES commands all require a `{` JSON body after their prefix — that shape routes to the frozen ES path; everything else is SQL. `CREATE INDEX ix ON t(c)` has no `{`, so no collision. Router regression tests lock both grammars.
- **ARIES essentials**: every WAL record carries `{lsn, prev_lsn(txn chain), txn_id, redo + undo payloads}`; page headers carry `page_lsn`; dirty-page table + active-txn table maintained; restart = analysis → redo (repeat history) → undo with CLRs; fuzzy checkpoints (begin/end records + DPT/ATT snapshots) bound recovery time and WAL-ring reclamation.
- **Key encoding**: B+ tree keys are order-preserving byte strings (sign-flipped BE ints, IEEE total-order floats, NULL-first). String keys use **icu4x collation sort keys** as the comparable form with the original string stored in the leaf value (sort keys aren't stable across ICU upgrades — document index rebuild on major icu4x bumps).
- **LSN** = unwrapped u64 WAL byte position (current `wal_state.tail` semantics).
- **Temp space** ("tempdb-lite"): allocator extents flagged temporary, never WAL-logged, freed wholesale on restart — backs spills and the version store.

## Stages

> **Status legend:** ✅ done · 🟡 partial (some sub-items unbuilt — see the `*(gap: …)*` note) · ❌ not started. Marks verified bullet-by-bullet against the code on 2026-07-14; see **[Implementation notes](#implementation-notes)** at the bottom. Shipped through Stage 12 (PRs #43–#70).

### Stage 1 — Storage substrate v2 (reclaim data region, buffer pool, ARIES-ready WAL)
`FILE_VERSION = 2` with in-place v1→v2 upgrade (rebuild bitmap marking only the live snapshot extent; no data movement).
- ✅ Delete the snapshot half-slot scheme: search snapshots become ordinary allocator extents (descriptor already stores arbitrary offset/len); free the previous extent once the new one is durable.
- ✅ Allocator: next-fit cursor, 64-page `allocate_extent()`, live in-memory instance, WAL-logged alloc/free, temp-extent flag; bitmap persisted at checkpoint.
- ✅ `DirectFile`: `read_page_into` / `write_page_from` / `write_pages_from` with caller-owned aligned frames; second file handle so log writes don't serialize behind page flushes.
- ✅ WAL rework: in-memory tail-page log buffer, flush = whole-page images (fixes the torn-tail RMW bug), lazy superblock (rewrite on checkpoint + every N MiB), recovery scans forward past `superblock.wal_tail` via CRC + seq_no monotonicity. New ARIES record framing (`lsn/prev_lsn/txn_id/redo/undo`) under `WAL_ENTRY_TYPE_REL=3`; search events keep type 1 unchanged.
- ✅ Buffer pool: configurable (default 64 MiB), pin/dirty/clock eviction, **steal allowed** (WAL-before-data is the only flush constraint); 32-byte page header (`page_lsn`, xxh64 checksum, type, `object_id`, `page_no` self-identity).
- ✅ Files: new `relstore/{mod,page,buffer_pool}.rs`, `wal/{mod,log_buffer,records}.rs`; changed `storage.rs`, `storage_layout.rs`, `allocator.rs`, `direct_io.rs`.
- ✅ Exit: v1-fixture upgrade test; torn-tail crash test; WAL-wrap/lazy-superblock reconstruction test; buffer-pool property tests; search CLI smoke (create/insert/search/restart/search) green.

### Stage 2 — Relational storage core: B+ trees, heaps, row codec, ARIES restart
No SQL yet; demonstrable via temporary debug commands (`table create/insert/scan/update/delete` through the command router).
- ✅ Slotted pages shared by all structures; **clustered B+ tree** (leaf = full row, internal = separators; sibling links; splits logged physiologically with FPI-on-split for idempotent redo) and **heap** (RID = page/slot, forwarding stubs) for PK-less tables.
- ✅ Row codec (`relstore/row.rs`): null bitmap | fixed section | var offsets | var data; T-SQL on-disk encodings (`relstore/types.rs`): INT family, BIT, FLOAT/REAL, DECIMAL(p≤38,s) as i128, DATE/TIME/DATETIME2 (SQL Server tick encodings), UNIQUEIDENTIFIER, [N]VARCHAR/VARBINARY. In-row cap ~3,900 B until overflow pages (Stage 14). Key codec per cross-cutting rules (collation-sort-key ready).
- 🟡 **Full ARIES restart** (`relstore/recovery.rs`): analysis/redo/undo with CLRs; fuzzy checkpoints (begin/end + DPT/ATT) replacing the WAL-usage-threshold logic for relational data; `wal_head` advances to min(checkpoint redo LSN, oldest active txn begin). Search snapshot/replay integrated into the combined checkpoint so truncation is safe for both subsystems. *(gap: checkpoints are quiescent — skip while any txn is active — not fuzzy begin/end+DPT/ATT)*
- ✅ Autocommit statement = implicit txn (begin/commit records); statement rollback via the undo chain.
- ✅ Catalog *storage*: fixed-object-id system tables as clustered trees, root page in `superblock.metadata_root`.
- ✅ Exit: kill-and-recover matrix (committed durable, uncommitted undone, crash *during* recovery re-runs cleanly — CLR idempotence); torn-page repair via FPI; B+ tree vs BTreeMap oracle property tests; split-crash tests; mixed search+relational WAL replay ordering test.

### Stage 3 — SQL front door: CREATE TABLE, INSERT, SELECT (via CLI)
- ✅ `truthdb-sql` crate: lexer + recursive-descent parser (spans everywhere), binder against `CatalogProvider`, 3VL evaluator. Grammar: `CREATE TABLE` (types INT/BIGINT/SMALLINT/TINYINT/BIT/FLOAT/VARCHAR(n)/NVARCHAR(n), NULL/NOT NULL, PRIMARY KEY → clustered tree; PK-less → heap), `DROP TABLE`, `INSERT ... VALUES`, `SELECT [TOP n] ... [WHERE] [ORDER BY]`, expressions (arithmetic, comparisons, AND/OR/NOT, IS NULL), `--`/`/* */`, `;` separators. PK duplicate → error 2627. *(note: binder lives in truthdb-core, not behind a `CatalogProvider` trait)*
- 🟡 Catalog semantics: `sys.tables`/`sys.columns` (queryable — the demo), `CatalogCache` with DDL version counter; bootstrap from hardcoded schemas. *(gap: no DDL version counter)*
- ✅ Volcano executors: clustered/heap scan, filter, project, in-memory sort, limit, insert. Command router + `spawn_blocking` (stops reactor blocking now, cheap). *(note: `spawn_blocking` superseded by the Stage 12 worker pool)*
- ✅ CLI: SQL completion on `;` (string/comment/bracket-aware), `GO` submits buffer, aligned ASCII table rendering, `(N rows affected)`, `Msg <n>, Level <s>` errors; JSON envelope `{"kind":...}` in `CommandResp.message`.
- ✅ Exit: create/insert/select/restart/reselect via CLI; parser corpus goldens (`truthdb-sql/tests/corpus/{ok,err}/*.sql`); 3VL truth-table tests.

### Stage 4 — TDS gateway 1: plaintext (pymssql, go-mssqldb `encrypt=disable`)
Real tools become the daily harness from here.
- ✅ New crate `crates/truthdb-tds`, listener on 1433 (`[tds] enabled`, addr/port config + packaging). One TDS connection = one engine session.
- ✅ Packet layer (8-byte header, EOM reassembly, negotiated packet size + ENVCHANGE ack); PRELOGIN (ENCRYPT_NOT_SUP, MARS off); LOGIN7 (UCS-2LE, password de-obfuscation, `[tds.auth]` users) → LOGINACK/ENVCHANGE/INFO/DONE; SQLBatch (consume ALL_HEADERS) → COLMETADATA/ROW/DONE(_MORE/_COUNT)/ERROR token streams. ERROR maps 1:1 from `SqlError` (number compatibility pays off).
- ✅ TYPE_INFO codecs for the Stage-3 type set (INTN/BITN/FLTN/BIGVARCHR+collation bytes/NVARCHAR); grow codecs as types land in later stages. Byte-fixture tests for every codec.
- 🟡 Basic Attention (cancel) handling: acknowledge + abort current statement (hook: `ExecCtx::check_cancelled()` polled in every operator). *(gap: attention is acknowledged only; no mid-statement abort, no `check_cancelled` hook)*
- ✅ Exit: CI conformance job — scripted go-mssqldb (Go) + pymssql (Python) doing create/insert/select/error-path against a live server in containers; login-failure test (18456). *(note: pytds substituted for pymssql)*

### Stage 5 — Single-table SQL completeness + collations
- 🟡 `UPDATE ... SET`, `DELETE` (materialize target RIDs/keys before mutating — Halloween avoidance), `DEFAULT`, `IDENTITY(seed,incr)` + `SCOPE_IDENTITY()`, NOT NULL (515). *(gap: `SCOPE_IDENTITY()` not implemented)*
- ✅ Types completed: DECIMAL (SQL Server result-precision formulas), DATETIME2/DATE, UNIQUEIDENTIFIER; `CAST`/`CONVERT`, `CASE`, `LIKE` (`%_[]`), `IN`, `BETWEEN`, `ISNULL`, `COALESCE`, string/number/date functions, documented implicit-conversion ladder.
- 🟡 **Collation plumbing**: per-column collation in `sys.columns`, `COLLATE` clause, precedence rules; icu4x-backed comparison + sort keys; SQL Server collation names mapped to ICU locales (`Finnish_Swedish_CI_AS` included); default database collation configurable. *(gap: no icu4x sort keys or precedence rules; comparison is basic; default collation hardcoded — the headline collation story is largely unbuilt)*
- ✅ Exit: DECIMAL goldens cross-checked against SQL Server docs; LIKE corpus; Swedish å/ä/ö ordering tests; identity continuity across restart; TDS type codecs extended + fixture-tested (DECIMAL/DATETIME2 are the fiddly ones).

### Stage 6 — Sessions, transactions, locking; driver transactions over TDS
The big architectural stage — do it before more SQL piles onto the global mutex.
- 🟡 **Engine actor**: replace `Arc<Mutex<Engine>>` with `EngineHandle` (mpsc of `EngineCall`) served by an engine-owned thread(+later pool); bounded sinks stream result rows; sessions live engine-side in a `SessionManager` (txn state, SET options; prepared statements later). Connection drop → rollback + lock release; idle/txn reaper. *(gap: results fully materialized in `BatchOutcome`, not streamed via bounded sinks; no idle/txn reaper)*
- 🟡 SQL: `BEGIN TRAN`/`COMMIT`/`ROLLBACK`, `@@TRANCOUNT`, `SET XACT_ABORT`, T-SQL nesting semantics, statement-level atomicity (statement failure keeps the txn unless XACT_ABORT/severity≥17), DDL in explicit txns disallowed for now (3902/3903 errors). *(gap: `SET XACT_ABORT` parsed but never read; a statement error always dooms the txn; no severity≥17 distinction)*
- ✅ Txn manager on ARIES (commit = flush-to-LSN; rollback = walk undo chain emitting CLRs); lock manager (IS/IX/S/X; Database/Table keys now, Row keys Stage 12; FIFO queues; 5 s wait timeout → abort youngest). Isolation: READ UNCOMMITTED/READ COMMITTED/REPEATABLE READ/SERIALIZABLE, lock-based (SERIALIZABLE correct via table locks until key-range locks).
- 🟡 **TDS Transaction Manager requests** (TM_BEGIN/COMMIT/ROLLBACK_XACT + ENVCHANGE 8/9/10 descriptors, validated in ALL_HEADERS) — `db.BeginTx()` / `setAutoCommit(false)` work as soon as transactions exist. *(gap: ALL_HEADERS skipped, not validated; transaction descriptor not checked)*
- ✅ Exit: two-driver-session isolation/blocking demos; disconnect-mid-txn rollback; commit-then-crash durable / uncommitted-then-crash undone; lock conflict matrix; go-mssqldb BeginTx matrix.

### Stage 7 — Secondary indexes and the planner
- 🟡 Nonclustered B+ indexes (leaf = key + PK locator, or RID for heaps), `CREATE [UNIQUE] INDEX ... (col [ASC|DESC])`, `DROP INDEX`, UNIQUE constraints (2601), blocking backfill; index maintenance in DML executors. *(gap: UNIQUE only via `CREATE UNIQUE INDEX`; no inline `UNIQUE` constraint in `CREATE TABLE`)*
- 🟡 Planner: constant folding → predicate pushdown → projection pruning → sargability against index key prefixes; rule order unique-seek > seek > range > scan; row counts as tie-breakers only. `IndexSeek/IndexScan/KeyLookup` executors. **`SET SHOWPLAN_TEXT ON`** returns the plan as a rowset. *(gap: no projection pruning, no row-count tie-breaks; KeyLookup folded into IndexSeek)*
- 🟡 Exit: key-codec order property tests (incl. collation keys); plan goldens; A/B harness (identical results with/without indexes). *(gap: collation-key order tests missing; index keys are binary, not collation sort keys)*

### Stage 8 — Joins, aggregation, DISTINCT — spill-capable from the start
- 🟡 `INNER/LEFT/RIGHT/FULL/CROSS JOIN`, comma-FROM, `GROUP BY`/`HAVING`, `COUNT/SUM/AVG/MIN/MAX` (+DISTINCT), `SELECT DISTINCT`, ORDER BY ordinals/expressions, ambiguity errors (209); binder scope stack with T-SQL alias-visibility rules. *(gap: ambiguity emits 207, not 209)*
- 🟡 Operators: block nested-loop join; hash join on equijoins (RIGHT→LEFT swap, FULL via matched-bits); hash aggregate; **external merge sort and grace-hash partitioning spilling to temp extents** when the per-query memory budget is exceeded (SQL Server tempdb-spill behavior). Int AVG truncates like T-SQL. Join order as written. *(gap: only naive nested-loop join + in-memory agg/sort — NO hash join, hash aggregate, external sort, grace-hash, or memory budget; "spill-capable" goal unmet)*
- ✅ Start the hand-rolled **sqllogictest-style runner** (`tests/slt/*.slt`, in-process against `Session::execute_batch`). *(note: drives `Engine::sql_batch`)*
- 🟡 Exit: NULL-heavy join/agg slt matrices; spill tests (tiny budget forces spill, results identical, temp space reclaimed on restart). *(gap: spill tests absent — no spill mechanism exists)*

### Stage 9 — TDS 2: TLS (rustls) + RPC/prepared statements (sqlcmd, go-mssqldb default encrypt)
- 🟡 Tunneled TLS handshake inside PRELOGIN packets (drive `rustls::ServerConnection` manually, then switch to a TLS stream) — the riskiest TDS artifact; fixture-test against captured client handshakes. `[tds] encryption = off|optional|required` + cert paths. *(gap: TLS + cert paths done; no `off|optional|required` mode — opportunistic only)*
- 🟡 Engine prepared-statement machinery (per-session `PreparedStatement{text, param_decls, bound plan, catalog_version}`, lazy rebind after DDL) fronted by RPC: `sp_executesql`, `sp_prepare/sp_execute/sp_prepexec/sp_unprepare` ProcIDs, RETURNVALUE/DONEPROC/DONEINPROC tokens, TYPE_INFO param decoding; `sp_cursor*` politely rejected. *(gap: only `sp_executesql`+TYPE_INFO done; no PreparedStatement/rebind, no `sp_prepare/execute/prepexec/unprepare`, no RETURNVALUE/DONEPROC/DONEINPROC tokens)*
- 🟡 Client-compat shims: `SET QUOTED_IDENTIFIER/ANSI_NULLS/TEXTSIZE` as session options; intrinsics `@@VERSION`, `@@SPID`, `@@TRANCOUNT`, `DB_NAME()`, `SUSER_SNAME()`; `sp_describe_first_result_set`. *(gap: `sp_describe_first_result_set` absent)*
- 🟡 Exit: CI matrix — go-mssqldb `encrypt=true trustservercertificate=true` parameterized CRUD; go-sqlcmd scripted `.sql` runs with output diffing; ODBC sqlcmd (mssql-tools18) flavor; rebind-after-DDL test. *(gap: go-mssqldb CRUD + pytds-TLS in CI; no go-sqlcmd/ODBC/JDBC, no rebind test)*

### Stage 10 — Constraints & ALTER TABLE
- 🟡 `FOREIGN KEY ... REFERENCES` (NO ACTION), `CHECK` (passes on TRUE or UNKNOWN), named constraints, `ALTER TABLE ADD/DROP CONSTRAINT`, `ALTER TABLE ADD` column (metadata-only when nullable/defaulted), `INSERT ... SELECT`. *(gap: `ALTER TABLE ADD <column>` not implemented — parser defers it)*
- 🟡 Enforcement in DML executors: FK child probe via parent PK; parent-side probe via child FK index if present else scan (documented cliff). Error 547 with constraint names. Catalog: `sys.foreign_keys`, `sys.check_constraints`, `sys.default_constraints`. *(gap: parent-side probe always scans children — child-FK-index path deferred)*
- 🟡 Exit: constraint slt matrix (NULL-FK skip-enforcement trap included); ALTER + old-row reads; metadata durability. *(gap: ADD-column old-row-read uncovered since ADD column is missing)*

### Stage 11 — Subqueries, views, variables
- 🟡 Derived tables, scalar subqueries (512 on >1 row), `IN (SELECT)`, `[NOT] EXISTS` (semi/anti hash joins uncorrelated; per-row `SubqueryExec` correlated — correct, slow, honest), non-recursive CTEs. *(gap: correlated subqueries not implemented — they error 207; uncorrelated derived/scalar/IN/EXISTS/CTE done)*
- 🟡 `CREATE/DROP VIEW` (source in `sys.sql_modules`, inline expansion at bind, depth-capped cycles, read-only); batch variables `DECLARE @v` / `SET` / `SELECT @v =`; `EXEC sp_executesql` as T-SQL text (same machinery as the RPC path). *(gap: view source in `TableDef.view_query`, no `sys.sql_modules`; no `EXEC sp_executesql` text path; view-over-view rejected, not expanded)*
- 🟡 Exit: correlated-subquery slt fixtures; view-over-view; parser corpus growth. *(gap: no correlated-subquery fixtures — feature absent; view-over-view slt only asserts rejection)*

### Stage 12 — Concurrency scale-out: worker pool, group commit, row locks
- 🟡 Engine actor becomes router + fixed worker-thread pool; shared services get internal synchronization (catalog RwLock, sharded buffer-pool latches, partitioned lock manager, sharded txn table). Search state behind one RwLock on the same pool. *(gap: worker pool + search RwLock done, but a single coarse `Mutex<StorageFile>` — no sharded buffer-pool latches, partitioned lock manager, or sharded txn table)*
- ✅ **Group commit** (`wal/log_writer.rs`): dedicated log-writer thread; committers wait on a `flushed_lsn` watermark; one fsync serves every commit in the window — with Stage 1's lazy superblock this fully retires fsync-per-append. *(note: log-writer fsyncs a duped WAL fd off the storage lock; commits/fsync ≫ 1 shows on slow disks — ~1 on tmpfs)*
- 🟡 Row locks (`LockKey::Row`) + IX intents, escalation at ~5000 locks; waits-for-graph deadlock detector (200 ms, youngest victim, error 1205); key-range locks on B+ leaves for true SERIALIZABLE. *(gap: deadlock detector done — but runs synchronously at park + 5 s timeout backstop, not a 200 ms poll; row locks, ~5000 escalation, and key-range locks all missing — locking is still table-granular)*
- 🟡 Exit: 16-thread bank-transfer invariant test with random aborts; guaranteed-deadlock test; bench shows commits/fsync ≫ 1; heartbeats answered during large scans. *(gap: transfer/deadlock/commits-per-fsync tests present; no heartbeats-during-scan test)*

### Stage 13 — Version store: RCSI and SNAPSHOT isolation
- ❌ Row versioning: updated/deleted rows copy their prior version to a version store in temp extents; rows carry a version pointer + txn timestamp; visibility by snapshot LSN; background cleanup once no snapshot needs a version.
- ❌ `ALTER DATABASE ... SET READ_COMMITTED_SNAPSHOT ON` and `ALLOW_SNAPSHOT_ISOLATION ON` database options; `SET TRANSACTION ISOLATION LEVEL SNAPSHOT`; write-write conflict detection under SNAPSHOT (error 3960).
- ❌ Exit: readers-don't-block demos under RCSI; snapshot write-conflict tests; version cleanup under load; all five isolation levels covered in the slt/driver matrix.

### Stage 14 — SSMS query window + large values + ops hardening
- ❌ SSMS scope = query window: catalog probes (`SERVERPROPERTY`, `sys.databases`, minimal `sys.configurations`), `USE truthdb` ENVCHANGE, accurate per-statement DONE_COUNT, `SET NOCOUNT`; JDBC smoke in CI; manual SSMS checklist file.
- ❌ Overflow page chains for values > in-row cap (VARCHAR(MAX)-class) + TDS PLP encoding for (MAX) types; offline file-growth procedure via CLI.
- ❌ Exit: JDBC + go-mssqldb full matrix green; 10 MiB value round-trip + crash-recovery; SSMS checklist pass.

### Stage 15 — T-SQL programmability: stored procedures, triggers, UDFs
- ❌ Grammar: `BEGIN...END`, `IF/ELSE`, `WHILE/BREAK/CONTINUE`, `RETURN [expr]`, `TRY/CATCH`, `THROW`, `RAISERROR` (printf subset); `CREATE/ALTER/DROP PROCEDURE` (param defaults, `OUTPUT`), `EXEC [@rc =] name` (positional/named/OUTPUT args); `CREATE FUNCTION` scalar + inline TVF + multi-statement TVF; `CREATE TRIGGER ... AFTER {INSERT,UPDATE,DELETE}` with `inserted`/`deleted` and `UPDATE(col)`. Deferred with justification: INSTEAD OF triggers, GOTO, cursors.
- ❌ Execution: **AST interpreter + per-statement cached plans** (what SQL Server does). Body parsed into a `ProcBody` control-flow tree; leaf statements bound/planned lazily, cached, stamped with catalog version (reuses prepared-statement rebind machinery). Engine `ProcCache` by object_id; per-session frame stack (`@@NESTLEVEL` cap 32 → error 217, recursion within cap); `@@ERROR`/`@@ROWCOUNT` maintained by the dispatch loop.
- ❌ Error semantics faithful: severity 11–19 catchable, ≤10 informational, ≥20 kills the connection; `XACT_ABORT`/doomed transactions give `XACT_STATE() = -1` (CATCH may only ROLLBACK) riding ARIES statement/txn undo. Build the severity/abort truth table first — it's the subtle core.
- ❌ Triggers: fire once per statement (incl. 0-row, faithful), creation order, `inserted`/`deleted` rowsets spill to temp extents, run inside the statement's txn (`ROLLBACK` in trigger → 3609); nested-triggers ON / RECURSIVE_TRIGGERS OFF defaults. Scalar UDFs run per-row (no inlining — documented perf cost); side-effect restrictions enforced in binder.
- ❌ Catalog: `sys.procedures/triggers/trigger_events/parameters` over `sys.objects`; objects gain an owner `principal_id` now (Stage 16 hook) and a stubbed `PermissionChecker` trait. TDS: RPC-by-name to user procs, RETURNSTATUS + RETURNVALUE tokens, exact DONEINPROC/DONEPROC counts.
- ❌ Exit: TRY/CATCH × severity × XACT_ABORT slt matrices; trigger suites (0-row fire, rollback-in-trigger, nesting caps, spill path); OUTPUT/return-status conformance from real drivers; kill-mid-trigger crash test (statement fully undone).

### Stage 16 — Security: logins, users, roles, permissions
- ❌ Authentication: `CREATE/ALTER/DROP LOGIN ... WITH PASSWORD`, **PBKDF2-HMAC-SHA256** (210k iterations, salted, versioned blob) using the crypto provider rustls already brings into the tree (zero new compiled deps; deliberately not SQL Server's hash format — security beats faithfulness, documented). Verification off the engine-actor thread. First boot migrates `[tds.auth]` config users into `sys.server_principals` and always ensures `sa`; config auth is then dead. In-memory per-login/IP login backoff; 18456 with faithful state codes.
- ❌ Principals: **both levels kept** (logins + database users, `dbo`↔`sa`) — SSMS and `SUSER_SNAME()`/`USER_NAME()` semantics assume it. Server role `sysadmin` (bypass-all); fixed db roles `db_owner/db_datareader/db_datawriter/db_ddladmin`; user-defined roles with nesting (cycle-checked). Effective membership = cached bitset with a **security version counter** bumped by security DDL.
- ❌ Permissions: `GRANT/DENY/REVOKE {SELECT,INSERT,UPDATE,DELETE,EXECUTE,REFERENCES,ALTER}` on objects + database-scoped `CREATE ...` grants; DENY beats GRANT across user + transitive roles. Column-level grants and `WITH GRANT OPTION` deferred (documented complexity cliff).
- ❌ Enforcement **in the binder**: plans carry a `RequiredPermissions` set + security version, revalidated on reuse when the version moves (closes the stale-plan bypass). **Ownership chaining implemented** (skip check when owner(A)==owner(B)) — makes the grant-EXECUTE-only app pattern work.
- ❌ Catalog: `sys.server_principals/sql_logins/database_principals/database_role_members/database_permissions`; `SUSER_SNAME`, `USER_NAME`, `IS_SRVROLEMEMBER`, `IS_ROLEMEMBER` intrinsics.
- ❌ Exit: PBKDF2 test vectors; DENY/GRANT truth-table slt across nested roles; ownership-chain scenarios (proc→view→table); revoke-then-reuse-cached-plan test; driver login conformance incl. backoff timing; config-migration upgrade test.

### Stage 17 — Backup & restore
- ❌ **Online fuzzy full backup** (ARIES-bracketed): force fuzzy checkpoint → `redo_start_lsn`; register a hold in a new **`LogTruncationGate`** (truncation floor = min over checkpoint/backup/log-backup/replication-slot holds — built here, reused by Stage 18); copy allocated pages in physical order (checksum-verify each, retry, buffer-pool fallback for torn reads); a concurrent **log follower** streams WAL entries into the backup so the hold trails rather than pinning the window; capture `backup_end_lsn`; fail clean (3013-style) if ring pressure would overwrite unshipped log. Search snapshot region copied as a raw extent — same LSN space, so log coverage is automatic.
- ❌ Recovery models: `ALTER DATABASE ... SET RECOVERY {FULL|SIMPLE}`. FULL holds ring truncation until `BACKUP LOG` copies `[last_log_backup_lsn, end]` to archive files (copy-out-before-truncate); no log backup + full ring ⇒ 9002 stall, faithful and documented.
- ❌ **Restore is offline via CLI** (`truthdb-cli restore --full f.bak [--log ...] [--stopat TS]`): rebuild a fresh file from page images + allocation map, ARIES redo of embedded+archived log, PITR stop at commit timestamp (commit records gain a timestamp field), undo in-flight txns. `RESTORE VERIFYONLY/HEADERONLY/FILELISTONLY` online over TDS; full T-SQL `RESTORE DATABASE` returns "instance must be offline" (single-db instance — documented simplification).
- ❌ Format `TDBBAK1`: self-describing header, allocation map, page-run blocks, 64 KiB log blocks, xxh64 on every block; `WITH INIT / CHECKSUM (default on) / STATS / COPY_ONLY`; compression codec byte reserved (lz4_flex later); DIFFERENTIAL/striping/encryption deferred. Minimal `sys.backupset` history table (file is self-describing, history loss non-fatal).
- ❌ Exit: backup-under-concurrent-write-load → restore → logical-dump diff; kill-mid-backup harmless; PITR between two marked commits; log-chain gap detection (4305); VERIFYONLY on bit-flipped fixtures; FULL-model ring-stall test.

### Stage 18 — Replication / HA (physical WAL shipping)
- ❌ **Physical, not logical**: standby applies ARIES redo verbatim → byte-identical replica, search subsystem replicated for free. Availability-group-lite: topology via config + CLI, not the enormous AG DDL (documented).
- ❌ Transport: replication listener (default 9624), TLS mandatory (rustls reuse), shared-secret auth; messages `Hello{node_id, cluster_uuid, epoch, last_received_lsn}` / `LogData` / `FlushAck{flushed_lsn, applied_lsn}` / `Heartbeat` / `Promoted`.
- ❌ Standby lifecycle: seeded from a Stage 17 backup restored `--standby`; **perpetual incremental redo** (restart redo refactored into a resumable LSN-range function with restartpoints). Per-standby **persistent slots** hold ring truncation via the `LogTruncationGate`; `max_slot_retain_bytes` exceeded ⇒ slot invalidated, standby must reseed.
- ❌ **D1 (async + manual failover)**: log-writer hands flushed ranges to per-standby sender threads; `truthdb-cli promote` (T-SQL alias `ALTER DATABASE ... FAILOVER`): drain, finish redo+undo, bump the new **epoch** field in the superblock, open read-write. Epoch fencing rejects lower-epoch streams; a diverged old primary must reseed (no pg_rewind analog — documented). No auto-failover/quorum in scope.
- ❌ **D2 (sync + readable standby)**: per-replica `SYNCHRONOUS_COMMIT` — group commit additionally waits for `FlushAck ≥ batch_end_lsn` (pending queue in the log-writer); ack timeout ⇒ replica NOT_SYNCHRONIZED, commits proceed (availability-first, documented, with a strictness knob). Readable standby: redo writes pre-images into the standby's **local** version store; reads are SNAPSHOT-only at the last-applied-commit LSN (same approach as SQL Server readable secondaries; depends on Stage 13).
- ❌ Monitoring: `sys.dm_repl_replica_states` / `sys.dm_repl_slots` virtual tables + `truthdb-cli repl-status`.
- ❌ Exit: two-process CI harness — standby kill/reconnect resumes from `last_received_lsn`; primary kill → promote → slt verification; sync-commit fault injection (delayed acks block commits, then timeout demotion); slot invalidation under ring pressure; epoch-fencing rejoin refusal; lag-metric assertions.

### Stage 19 — Future outline (not planned in detail)
❌ Multiple databases/schemas, cursors, INSTEAD OF triggers, GOTO, savepoints (`SAVE TRAN`), cost-based optimizer + histogram statistics, recursive CTEs, updatable views, FK CASCADE, MARS (recommend: never), BULK INSERT, columnstore, column-level grants / `WITH GRANT OPTION` / `EXECUTE AS`, DIFFERENTIAL/striped/encrypted backups, auto-failover + client routing + logical replication, and **full-text convergence**: reimplement the ES DSL as `CREATE FULLTEXT INDEX` + `CONTAINS()` over relational tables, then retire the legacy in-memory `IndexState`.

## Search-engine coexistence (every stage)

Search WAL events keep `entry_type=1` JSON payloads and existing replay; relational records use new entry types; recovery demultiplexes by type from one ring. Search snapshots keep the descriptor mechanism (placement changes to allocator extents in Stage 1). Combined checkpoint ensures `wal_head` never passes a record either subsystem needs. The ES DSL grammar is frozen; router regression tests guard both surfaces. Search stays reachable only via the native protocol until full-text convergence.

## Testing & verification strategy

- **Every stage**: `cargo fmt --check`, `clippy -D warnings`, `cargo test --workspace` in Linux containers (macOS: Docker wrappers per CLAUDE.md); scripted end-to-end smoke (CLI early, real TDS drivers from Stage 4); search-regression smoke.
- **Parser corpus** (Stage 3+): golden `.sql` → AST/error files; every bug becomes a corpus file.
- **Crash/recovery harness** (Stage 2+): kill-without-checkpoint, reopen, assert exactly-committed state; ARIES-specific: undo-with-CLRs, crash-during-recovery, torn-page FPI repair.
- **slt runner** (Stage 8+): hand-rolled sqllogictest-format files driving `Session::execute_batch` in-process.
- **Plan goldens** (Stage 7+): SHOWPLAN_TEXT snapshots.
- **TDS conformance** (Stage 4+): CI jobs with real drivers (pymssql, go-mssqldb, go-sqlcmd, ODBC sqlcmd, JDBC) against a live server; byte-fixture tests for every packet/token/type codec.
- **Collation tests** (Stage 5+): Swedish/Finnish ordering goldens; sort-key/index-order consistency.
- **Backup/replication harnesses** (Stage 17+): backup-under-write-load round-trip diffs, PITR fixtures; two-process primary/standby CI harness (kill/reconnect, promote, epoch fencing, sync-commit fault injection).

## Key risks

1. **Full ARIES is the hardest subsystem** (undo chains, CLRs, fuzzy checkpoints, ring-WAL interaction) and it lands early (Stages 1–2) because everything sits on it. Mitigation: exhaustive crash-harness matrix before any SQL lands; keep the old WAL path behind a test flag until crash tests pass.
2. **Stage 6 engine-actor refactor** touches every request path — sequenced while locks are table-granular and interleavings coarse.
3. **TDS-early means codec churn**: TYPE_INFO codecs grow with each type stage; byte fixtures per codec keep this mechanical. Tunneled TLS-in-PRELOGIN (Stage 9) is ~200 careful lines — fixture-test against captured handshakes.
4. **Collation-aware keys** couple icu4x sort keys to index layout — original values stored in leaves; index rebuild documented for icu4x major upgrades.
5. **DECIMAL arithmetic and datetime encodings** are fiddly pure-code sinks — goldens cross-checked against SQL Server documentation.
6. **Interface churn between SQL and storage**: `TableAccess`/`IndexAccess` traits + row/key codecs co-signed at Stage 2 exit; everything above is insulated.
7. **TRY/CATCH × statement-atomicity semantics** (Stage 15): which errors doom a transaction is the subtlest T-SQL behavior to replicate — build the severity/abort truth table first and test it exhaustively.
8. **Resumable-redo refactor** (Stage 18) touches restart correctness, and standby redo-generated row versioning is the hardest replication piece — both isolated behind the D1/D2 split.

## Implementation notes

*Added 2026-07-14. The stage/bullet marks above (✅ / 🟡 / ❌) were verified bullet-by-bullet against the actual code in `truthdb/` — git history plus a per-stage source audit — not from memory. The plan text is the original **target**; this section records where the shipped build **diverges** from it. Everything through Stage 12 is merged (PRs #43–#70).*

**One-line status:** a working, transactional, concurrent SQL-Server-compatible engine — real ARIES storage/recovery, TDS wire protocol, most of the single-table + join/aggregate SQL surface, and the concurrency scale-out. Stages 1–8 are the core (done, with the gaps below); 9–12 are partial; 13–19 are unstarted.

### Notable divergences (what the 🟡 marks add up to)

- **Query engine has no hash operators or spill (Stage 8).** Joins, `GROUP BY`, `DISTINCT`, and `ORDER BY` work, but only via naive nested-loop join + fully in-memory aggregation/sort. There is **no** hash join, hash aggregate, external merge sort, grace-hash partitioning, or per-query memory budget — so the plan's "spill-capable from the start" goal is unmet. Large joins/sorts are slow and memory-bound.
- **Concurrency is coarse (Stage 12).** The worker pool, group commit, and waits-for-graph deadlock detector are in, but all storage access still funnels through a single `Mutex<StorageFile>` — none of the planned sharded buffer-pool latches / partitioned lock manager / sharded txn table. Locking is table-granular: **no row locks, intent-lock escalation, or key-range locks**, so concurrent writers to the same table serialize even on different rows and SERIALIZABLE leans on table locks. Group commit's win ("commits/fsync ≫ 1") shows only on slow disks — ~1 on fast/tmpfs storage. The deadlock detector runs the graph synchronously when a batch parks (plus a 5 s timeout backstop), not the planned 200 ms poll.
- **Collation is basic (Stage 5).** `COLLATE` and per-column collation are plumbed, but comparison is **not** icu4x-sort-key based — no Swedish/Finnish ordering, no collation precedence rules, default collation hardcoded. Index keys are binary, not collation sort keys (Stage 7).
- **Checkpoints are quiescent, not fuzzy (Stage 2).** ARIES analysis/redo/undo + CLRs are correct, but a checkpoint is skipped while any transaction is active rather than the planned fuzzy checkpoint (begin/end + DPT/ATT snapshots). Correct, but the WAL can grow under a long-running transaction.
- **Prepared statements are minimal (Stage 9).** `sp_executesql` (parameterized queries) works; the `sp_prepare`/`sp_execute`/`sp_prepexec`/`sp_unprepare` handle family, per-session `PreparedStatement` caching + rebind-after-DDL, and RETURNVALUE/DONEPROC/DONEINPROC tokens are not built.

### Smaller deferred items (self-contained follow-ups)

- Correlated subqueries (Stage 11) — uncorrelated forms work; a correlated one errors 207.
- `ALTER TABLE ADD <column>` (Stage 10).
- `SCOPE_IDENTITY()` (Stage 5).
- `SET XACT_ABORT` is parsed but not honored; a statement error always dooms the transaction, no severity≥17 distinction (Stage 6).
- Views are stored in `TableDef.view_query`, not `sys.sql_modules`; view-over-view is rejected rather than expanded; no `EXEC sp_executesql` text path (Stage 11).
- Planner: no projection pruning, no row-count tie-breaks, `KeyLookup` folded into `IndexSeek`; UNIQUE only via `CREATE UNIQUE INDEX`, not an inline constraint (Stage 7).
- TDS Attention (cancel) is acknowledged but does not abort a running statement; `ALL_HEADERS` is skipped, not validated (Stages 4, 6). No `sp_describe_first_result_set` (Stage 9). No idle/txn reaper; result rows are materialized in `BatchOutcome`, not streamed via bounded sinks (Stage 6). No DDL version counter (Stage 3).

### Testing substitutions

- `pytds` substituted for `pymssql`; the slt runner drives `Engine::sql_batch` (not the literal `Session::execute_batch`); TDS CI uses `encrypt=disable` and covers go-mssqldb + pytds, not the full go-sqlcmd / ODBC / JDBC matrix the later stages call for.

### Architecture notes

- The binder lives in `truthdb-core`, not behind the planned `CatalogProvider` trait in `truthdb-sql`.
- Stage 3's `spawn_blocking` execution model was superseded by the Stage 12 worker-thread pool.
- Search (ES) coexistence holds: search WAL events keep `entry_type=1`, relational records use type 3, one ring demultiplexed by type; the ES DSL is still reachable only via the native protocol.
