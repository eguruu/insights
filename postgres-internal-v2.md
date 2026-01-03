### PostgreSQL Internals Cheatsheet  

#### Part I: Isolation and MVCC

* **Isolation Levels**

  **Read Uncommitted**  
  Allows dirty reads (rarely used). PostgreSQL implements it as Read Committed – true dirty reads are not possible.

  **Command**:
  ```sql
  SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
  SHOW transaction_isolation;
  ```

  **Demonstration** (behaves like Read Committed):
  ```sql
  -- Session 1
  BEGIN;
  UPDATE accounts SET balance = 999999 WHERE id = 1;  -- Uncommitted

  -- Session 2 (Read Uncommitted)
  SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
  SELECT balance FROM accounts WHERE id = 1;  -- Still sees old committed value

  -- Session 1
  ROLLBACK;
  ```

  **Read Committed**  
  Default; each statement sees only committed data (non-repeatable reads possible).

  **Command**:
  ```sql
  SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL READ COMMITTED;
  SHOW transaction_isolation;
  ```

  **Demonstration** (non-repeatable read):
  ```sql
  -- Session 1
  BEGIN;
  SELECT balance FROM accounts WHERE id = 1;  -- Sees 1000

  -- Session 2
  UPDATE accounts SET balance = 2000 WHERE id = 1;
  COMMIT;

  -- Session 1
  SELECT balance FROM accounts WHERE id = 1;  -- Now sees 2000
  COMMIT;
  ```

  **Repeatable Read**  
  Snapshot per transaction (phantom reads possible).

  **Command**:
  ```sql
  SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  SHOW transaction_isolation;
  ```

  **Demonstration** (repeatable but phantom possible):
  ```sql
  -- Session 1
  BEGIN ISOLATION LEVEL REPEATABLE READ;
  SELECT count(*) FROM accounts WHERE category = 'premium';  -- Sees 10

  -- Session 2
  INSERT INTO accounts (category, balance) VALUES ('premium', 5000);
  COMMIT;

  -- Session 1
  SELECT count(*) FROM accounts WHERE category = 'premium';  -- Still sees 10
  COMMIT;  -- May raise serialization failure only if conflict detected
  ```

  **Serializable**  
  True serializability via Snapshot Isolation + conflict detection (SSI since v9.1).

  **Command**:
  ```sql
  SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  SHOW transaction_isolation;
  ```

  **Demonstration** (serialization failure):
  ```sql
  -- Session 1
  BEGIN ISOLATION LEVEL SERIALIZABLE;
  SELECT sum(balance) FROM accounts WHERE active = true;

  -- Session 2
  BEGIN ISOLATION LEVEL SERIALIZABLE;
  UPDATE accounts SET active = false WHERE balance < 100;
  COMMIT;

  -- Session 1
  COMMIT;  -- ERROR: could not serialize access due to read/write dependencies
  ```

* **MVCC Basics**  
  Readers don't block writers, writers don't block readers. UPDATE/DELETE creates new tuple versions.

  **Command** (view tuple versions):
  ```sql
  SELECT ctid, xmin, xmax, * FROM accounts WHERE id = 1;
  ```

  **Demonstration** (new version creation):
  ```sql
  UPDATE accounts SET balance = balance + 100 WHERE id = 1;
  SELECT ctid, xmin, xmax, balance FROM accounts WHERE id = 1;  -- Two versions visible temporarily
  ```

* **Pages and Tuples**  
  Pages: Fixed 8KB blocks. Tuples: Rows with headers (xmin, xmax, ctid, etc.).

  **Command** (page header):
  ```sql
  CREATE EXTENSION pageinspect;
  SELECT * FROM page_header(get_raw_page('accounts', 0));
  ```

  **Command** (tuples on page):
  ```sql
  SELECT lp, lp_flags, lp_len, t_xmin, t_xmax, t_ctid
  FROM heap_page_items(get_raw_page('accounts', 0));
  ```

* **Snapshots**  
  Track active transactions to determine visible versions.

  **Command** (current snapshot):
  ```sql
  SELECT txid_current_snapshot();
  ```

  **Command** (snapshot components):
  ```sql
  SELECT txid_snapshot_xmin(txid_current_snapshot()) AS xmin,
         txid_snapshot_xmax(txid_current_snapshot()) AS xmax,
         unnest(txid_current_snapshot()) AS active_xids;
  ```

* **Page Pruning and HOT Updates**  
  HOT: Update without index update if new version fits same page. Pruning removes dead tuples.

  **Command** (likely HOT update):
  ```sql
  UPDATE accounts SET note = 'HOT test' WHERE id = 1;  -- note not indexed
  ```

  **Command** (check HOT/pruning evidence):
  ```sql
  SELECT lp, lp_flags, t_ctid FROM heap_page_items(get_raw_page('accounts', blkno));
  ```

* **Vacuum and Autovacuum**  
  Reclaim space from dead tuples, update visibility map.

  **Command** (manual vacuum):
  ```sql
  VACUUM (VERBOSE) accounts;
  ```

  **Command** (monitor vacuum):
  ```sql
  SELECT pid, phase, heap_blks_scanned, num_dead_tuples
  FROM pg_stat_progress_vacuum;
  ```

  **Command** (autovacuum status):
  ```sql
  SELECT relname, n_dead_tup, last_autovacuum, autovacuum_count
  FROM pg_stat_user_tables WHERE relname = 'accounts';
  ```

* **Freezing**  
  Prevents transaction ID wraparound by setting xmin to FrozenXID.

  **Command** (table freeze age):
  ```sql
  SELECT relname, age(relfrozenxid) AS frozen_age
  FROM pg_class WHERE relkind = 'r'
  ORDER BY age(relfrozenxid) DESC LIMIT 10;
  ```

  **Command** (database freeze age):
  ```sql
  SELECT datname, age(datfrozenxid) FROM pg_database;
  ```

* **Rebuilding**  
  CLUSTER/VACUUM FULL rewrite tables/indexes to defragment.

  **Command** (full rewrite):
  ```sql
  VACUUM FULL accounts;
  ```

  **Command** (reorder by index):
  ```sql
  CLUSTER accounts USING accounts_pkey;
  ```

  **Command** (monitor CLUSTER):
  ```sql
  SELECT phase, heap_blks_scanned FROM pg_stat_progress_cluster;
  ```

#### Part II: Buffer Cache and WAL

* **Buffer Cache**  
  Shared memory for pages (shared_buffers). Clock-sweep eviction (usage_count 0–5).

  **Command** (install extension):
  ```sql
  CREATE EXTENSION pg_buffercache;
  ```

  **Command** (buffers for table):
  ```sql
  SELECT count(*) AS buffers, count(*) FILTER (WHERE isdirty) AS dirty
  FROM pg_buffercache b
  JOIN pg_class c ON b.relfilenode = c.relfilenode
  WHERE c.relname = 'accounts';
  ```

  **Command** (usage_count distribution):
  ```sql
  SELECT usagecount, count(*) FROM pg_buffercache GROUP BY usagecount ORDER BY usagecount;
  ```

* **Write-Ahead Log (WAL)**  
  Ensures durability. Changes logged before data pages.

  **Command** (current WAL position):
  ```sql
  SELECT pg_current_wal_lsn();
  ```

  **Command** (WAL generated since startup):
  ```sql
  SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0'));
  ```

  **Command** (list recent WAL segments):
  ```sql
  SELECT name, pg_size_pretty(size) FROM pg_ls_waldir()
  WHERE name ~ '^[0-9A-F]{24}$' ORDER BY modification DESC LIMIT 10;
  ```

  **Command** (checkpoint stats):
  ```sql
  SELECT checkpoints_timed, checkpoints_req, buffers_checkpoint
  FROM pg_stat_bgwriter;
  ```

* **WAL Modes (wal_level)**

  **Command** (current level):
  ```sql
  SHOW wal_level;
  ```

  **Command** (related settings):
  ```sql
  SELECT name, setting FROM pg_settings
  WHERE name IN ('wal_level', 'wal_buffers', 'full_page_writes');
  ```

#### Part III: Locks

* **Relation-Level Locks**

  **Command** (acquire table lock):
  ```sql
  LOCK TABLE accounts IN SHARE ROW EXCLUSIVE MODE;
  ```

  **Command** (view relation locks):
  ```sql
  SELECT relation::regclass, mode, granted, pid
  FROM pg_locks WHERE locktype = 'relation' AND relation = 'accounts'::regclass;
  ```

* **Row-Level Locks**

  **Command** (acquire row lock):
  ```sql
  BEGIN;
  SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
  ```

  **Command** (inspect row locks):
  ```sql
  CREATE EXTENSION pgrowlocks;
  SELECT pid, locktype, mode, locked_row_ctid FROM pgrowlocks('accounts');
  ```

* **Miscellaneous Locks**

  **Command** (advisory lock):
  ```sql
  SELECT pg_advisory_xact_lock(100, 42);
  ```

  **Command** (view advisory locks):
  ```sql
  SELECT classid, objid, pid, mode FROM pg_locks WHERE locktype = 'advisory';
  ```

* **Locks in Memory**

  **Command** (waiting heavyweight locks):
  ```sql
  SELECT locktype, relation::regclass, mode, pid FROM pg_locks WHERE NOT granted;
  ```

  **Command** (simple blocking detection):
  ```sql
  SELECT blocking.pid AS blocking_pid, waiting.pid AS waiting_pid
  FROM pg_locks blocking
  JOIN pg_locks waiting ON blocking.relation = waiting.relation
  WHERE blocking.granted AND NOT waiting.granted;
  ```

#### Part IV: Query Execution

* **Stages**  
  Parser → Planner → Executor

  **Command** (full execution plan):
  ```sql
  EXPLAIN (ANALYZE, BUFFERS, VERBOSE, TIMING)
  SELECT * FROM accounts WHERE id = 1;
  ```

* **Statistics**

  **Command** (collect stats):
  ```sql
  ANALYZE accounts;
  ```

  **Command** (view column stats):
  ```sql
  SELECT attname, null_frac, n_distinct, most_common_vals
  FROM pg_stats WHERE tablename = 'accounts';
  ```

* **Table Access Methods**

  **Command** (force Seq Scan):
  ```sql
  SET enable_indexscan = off;
  EXPLAIN SELECT * FROM accounts WHERE id = 1;
  ```

* **Index Access Methods** (Index-only scans)

  **Command** (enable index-only scan):
  ```sql
  VACUUM accounts;  -- Updates visibility map
  EXPLAIN (ANALYZE) SELECT id FROM accounts WHERE id IS NOT NULL;
  ```

* **Join Methods**

  **Command** (disable hash/merge join):
  ```sql
  SET enable_hashjoin = off;
  SET enable_mergejoin = off;
  EXPLAIN SELECT * FROM accounts a JOIN transactions t ON a.id = t.account_id;
  ```

* **Other** (Sorting & Hashing)

  **Command** (force external sort):
  ```sql
  SET work_mem = '64kB';
  EXPLAIN (ANALYZE) SELECT * FROM accounts ORDER BY note;
  ```

  **Command** (hash aggregate):
  ```sql
  EXPLAIN (ANALYZE) SELECT category, count(*) FROM accounts GROUP BY category;
  ```

#### Part V: Types of Indexes

| Index Type | Use Case | Key Features | Ordering | Duplicates | Nulls | Creation Command | Inspection Command |
|------------|----------|--------------|----------|------------|-------|------------------|--------------------|
| **Hash** | Equality only | Fast equality on large tables | No | Yes | No | `CREATE INDEX accounts_balance_hash ON accounts USING HASH (balance);` | `EXPLAIN SELECT * FROM accounts WHERE balance = 1000;` |
| **B-Tree** (default) | Range, equality, ORDER BY | Most versatile | Yes | Yes | Partial | `CREATE INDEX accounts_balance_btree ON accounts (balance);` | `SELECT * FROM bt_page_items('accounts_balance_btree', 1);` |
| **GiST** | Geometric, ranges, custom | Lossy, flexible operators | Partial | Yes | Yes | `CREATE INDEX accounts_geom_gist ON accounts USING GIST (location);` | `SELECT * FROM gist_page_items('accounts_geom_gist', 1);` |
| **SP-GiST** | Unbalanced (prefixes, IPs) | Space-partitioned | No | Yes | Yes | `CREATE INDEX accounts_phone_spgist ON accounts USING SPGIST (phone);` | — |
| **GIN** | Arrays, JSONB, full-text | Inverted index | No | Yes | Yes | `CREATE INDEX accounts_tags_gin ON accounts USING GIN (tags);` | `EXPLAIN SELECT * FROM accounts WHERE tags @> '{premium}';` |
| **BRIN** | Very large sequential tables | Block summaries, tiny size | Yes | Yes | Yes | `CREATE INDEX accounts_ts_brin ON accounts USING BRIN (created_at);` | `SELECT * FROM brin_page_items('accounts_ts_brin', 1);` |

**Tips**:
```sql
-- Partial index
CREATE INDEX accounts_active_balance ON accounts (balance) WHERE active;

-- Expression index
CREATE INDEX accounts_lower_name ON accounts (lower(name));

-- Monitor unused indexes
SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;
```
