### PostgreSQL Internals Cheatsheet

This cheatsheet summarizes key concepts from PostgreSQL internals, originally based on the structure of *PostgreSQL 14 Internals* by Egor Rogov (Postgres Professional). Core mechanisms remain highly relevant in the latest version (**PostgreSQL 18** as of January 2026), with only incremental changes since v14.

#### Part I: Isolation and MVCC

* **Isolation Levels**:
  * Read Uncommitted: Allows dirty reads (rarely used).
  * Read Committed: Default; each statement sees committed data (non-repeatable reads possible).
  * Repeatable Read: Snapshot per transaction (phantom reads possible).
  * Serializable: True serializability via Snapshot Isolation + conflict detection (SSI since v9.1).

  **Nested Commands**:
  ```sql
  SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  SHOW transaction_isolation;
  -- Demo non-repeatable read (Read Committed)
  -- Session 1: BEGIN; SELECT * FROM accounts WHERE id = 1;
  -- Session 2: UPDATE accounts SET balance = balance + 100 WHERE id = 1; COMMIT;
  -- Session 1: SELECT * FROM accounts WHERE id = 1;  -- Sees new value
  COMMIT;
  ```

* **MVCC Basics**:
  * Multi-Version Concurrency Control: Readers don't block writers, writers don't block readers.
  * Each transaction gets a snapshot (point-in-time view via xmin/xmax in tuple headers).
  * UPDATE/DELETE creates new tuple versions (old marked dead via xmax).
  * Visibility check: Tuple visible if created by committed tx before snapshot and not deleted after.

  **Nested Commands**:
  ```sql
  SELECT ctid, xmin, xmax, * FROM your_table LIMIT 5;
  -- Deeper inspection
  CREATE EXTENSION pageinspect;
  SELECT lp, t_xmin, t_xmax, t_ctid FROM heap_page_items(get_raw_page('your_table', 0));
  ```

* **Pages and Tuples**:
  * Pages: Fixed 8KB blocks.
  * Tuples: Rows with headers (xmin, xmax, ctid, etc.).

  **Nested Commands**:
  ```sql
  CREATE EXTENSION pageinspect;
  SELECT * FROM page_header(get_raw_page('your_table', 0));  -- Page header
  SELECT count(*) AS tuples_on_page FROM heap_page_items(get_raw_page('your_table', 0));
  ```

* **Snapshots**:
  * Track active transactions; determine visible versions.

  **Nested Commands**:
  ```sql
  SELECT txid_current_snapshot();  -- e.g., 1000:2000:1005,1010
  SELECT txid_snapshot_xmin(txid_current_snapshot()), txid_snapshot_xmax(txid_current_snapshot());
  -- Active backends
  SELECT pid, query FROM pg_stat_activity WHERE backend_xid IS NOT NULL;
  ```

* **Page Pruning and HOT Updates**:
  * Heap-Only Tuple (HOT): Update without index update if new version fits on same page.
  * Pruning: Remove dead tuples during updates.

  **Nested Commands**:
  ```sql
  -- Trigger potential HOT update
  UPDATE your_table SET non_indexed_col = non_indexed_col WHERE pk = 42;
  -- Check page for dead line pointers
  SELECT lp_flags FROM heap_page_items(get_raw_page('your_table', blkno));
  ```

* **Vacuum and Autovacuum**:
  * VACUUM: Reclaim space from dead tuples, update visibility map.
  * Autovacuum: Automatic cleanup to prevent bloat and transaction ID wraparound.

  **Nested Commands**:
  ```sql
  VACUUM (VERBOSE, ANALYZE) your_table;
  SELECT phase, heap_blks_scanned, heap_blks_vacuumed FROM pg_stat_progress_vacuum;
  -- Autovacuum stats
  SELECT relname, n_dead_tup, last_autovacuum FROM pg_stat_user_tables;
  ```

* **Freezing**:
  * Prevent tx ID wraparound; freeze old tuples (set xmin to FrozenXID).

  **Nested Commands**:
  ```sql
  SELECT relname, age(relfrozenxid) FROM pg_class WHERE relkind = 'r' ORDER BY age(relfrozenxid) DESC;
  SELECT datname, age(datfrozenxid) FROM pg_database;
  ```

* **Rebuilding**:
  * CLUSTER/VACUUM FULL: Rewrite tables/indexes to defragment.

  **Nested Commands**:
  ```sql
  VACUUM FULL your_table;
  CLUSTER your_table USING your_index;
  SELECT * FROM pg_stat_progress_cluster;
  ```

#### Part II: Buffer Cache and WAL

* **Buffer Cache**:
  * Shared memory for pages (shared_buffers param).
  * Clock-sweep algorithm for eviction (usage_count up to 5).
  * Double caching with OS page cache.
  * Pinning: Buffers pinned during use.

  **Nested Commands**:
  ```sql
  CREATE EXTENSION pg_buffercache;
  SELECT relname, count(*), count(*) FILTER (WHERE isdirty) AS dirty FROM pg_buffercache b JOIN pg_class c ON b.relfilenode = c.relfilenode GROUP BY relname;
  SELECT usagecount, count(*) FROM pg_buffercache GROUP BY usagecount;
  ```

* **Write-Ahead Log (WAL)**:
  * Ensures durability: Changes logged before applied to data pages.
  * Records: Minimal redo info (not full page unless full_page_writes=on).
  * Segments: 16MB files in pg_wal.
  * Buffers: wal_buffers (default ~1/32 of shared_buffers).
  * Checkpoint: Flush dirty buffers periodically; WAL replay for recovery.

  **Nested Commands**:
  ```sql
  SELECT pg_current_wal_lsn(), pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0'));
  SELECT * FROM pg_ls_waldir() ORDER BY modification DESC LIMIT 5;
  SELECT * FROM pg_stat_bgwriter;  -- Checkpoint stats
  ```

* **WAL Modes (wal_level)**:
  * minimal: No replication.
  * replica: Physical replication.
  * logical: Logical replication.

  **Nested Commands**:
  ```sql
  SHOW wal_level;
  SELECT name, setting FROM pg_settings WHERE name LIKE 'wal%';
  ```

#### Part III: Locks

* **Relation-Level Locks (table-level)**:
  * ACCESS SHARE (SELECT), ROW SHARE (SELECT FOR SHARE), ROW EXCLUSIVE (UPDATE/DELETE), etc.
  * Up to ACCESS EXCLUSIVE (DROP TABLE).

  **Nested Commands**:
  ```sql
  LOCK TABLE your_table IN ACCESS EXCLUSIVE MODE;
  SELECT relation::regclass, mode, granted FROM pg_locks WHERE locktype = 'relation';
  ```

* **Row-Level Locks**:
  * FOR UPDATE, FOR SHARE, etc.; taken on tuples during DML.
  * Deadlock detection via wait graph.

  **Nested Commands**:
  ```sql
  SELECT * FROM your_table FOR UPDATE;
  CREATE EXTENSION pgrowlocks;
  SELECT * FROM pgrowlocks('your_table');
  ```

* **Miscellaneous Locks**:
  * Advisory locks (application-level).
  * Predicate locks (for serializable isolation; on index ranges).

  **Nested Commands**:
  ```sql
  SELECT pg_advisory_lock(999);
  SELECT * FROM pg_locks WHERE locktype = 'advisory';
  ```

* **Locks in Memory**:
  * Spinlocks: Low-level CPU primitives.
  * Lightweight Locks (LWLocks): For shared structures (e.g., buffer mapping, WAL insert/write).
  * Heavyweight Locks: For relations/rows.

  **Nested Commands**:
  ```sql
  SELECT * FROM pg_locks WHERE NOT granted;  -- Waiting locks
  -- Simple blocking query
  SELECT blocking.pid AS blocking_pid, blocked.pid AS blocked_pid
  FROM pg_locks blocked JOIN pg_locks blocking ON blocked.relation = blocking.relation
  WHERE NOT blocked.granted;
  ```

#### Part IV: Query Execution

* **Stages**:
  * Parser → Planner (optimizer) → Executor.

  **Nested Commands**:
  ```sql
  EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT * FROM your_table;
  ```

* **Statistics**:
  * pg_statistic; ANALYZE collects for cost-based planning.

  **Nested Commands**:
  ```sql
  ANALYZE your_table;
  SELECT * FROM pg_stats WHERE tablename = 'your_table';
  ```

* **Table Access Methods**:
  * Seq Scan, Index Scan, Bitmap Heap Scan.

  **Nested Commands**:
  ```sql
  EXPLAIN SELECT * FROM your_table WHERE id = 42;  -- Observe scan type
  ```

* **Index Access Methods**:
  * Index-only scans (visibility map).

  **Nested Commands**:
  ```sql
  SELECT relname, relpages FROM pg_class WHERE relname = 'your_table';
  ```

* **Join Methods**:
  * Nested Loop (with/without index).
  * Hash Join.
  * Merge Join.

  **Nested Commands**:
  ```sql
  SET enable_hashjoin = off;  -- Force other joins
  EXPLAIN SELECT * FROM table1 JOIN table2 ON table1.id = table2.id;
  ```

* **Other**:
  * Sorting (external if large).
  * Hashing for aggregation/joins.

  **Nested Commands**:
  ```sql
  EXPLAIN (ANALYZE) SELECT col, count(*) FROM your_table GROUP BY col;
  ```

#### Part V: Types of Indexes

| Index Type | Use Case | Key Features | Ordering | Duplicates | Nulls | Nested Creation & Inspection |
|------------|----------|--------------|----------|------------|-------|-----------------------------|
| Hash | Equality (=) only | Fast for large equality lookups | No | Yes | No | `CREATE INDEX ON table USING HASH (col);`<br>`SELECT * FROM pg_stat_user_indexes WHERE indexrelname LIKE '%hash';` |
| B-Tree (default) | Range (<, >, =), ORDER BY, equality | Balanced tree; most versatile | Yes | Yes | Partial | `CREATE INDEX ON table (col);`<br>`SELECT * FROM bt_page_items('index_name', 1);` (pageinspect) |
| GiST | Geometric, full-text, custom (e.g., range types, spatial) | Generalized Search Tree; lossy | Partial | Yes | Yes | `CREATE INDEX ON table USING GIST (geom);`<br>`SELECT * FROM gist_page_items('index_name', 1);` |
| SP-GiST | Unbalanced data (e.g., phone numbers, URLs, IP) | Space-Partitioned GiST | No | Yes | Yes | `CREATE INDEX ON table USING SPGIST (col);` |
| GIN | Arrays, JSONB, full-text (inverted index) | Generalized Inverted; good for composite values | No | Yes | Yes | `CREATE INDEX ON table USING GIN (array_col);` |
| BRIN | Very large tables (block range) | Block Range INdex; small size, lossy | Yes | Yes | Yes | `CREATE INDEX ON table USING BRIN (col);`<br>`SELECT * FROM brin_page_items('index_name', 1);` |

**Tips**:
* Indexes add write overhead (WAL + space).
* REINDEX/CLUSTER for maintenance.
* Partial/Expression indexes for optimization: `CREATE INDEX ON table (col) WHERE condition;`

