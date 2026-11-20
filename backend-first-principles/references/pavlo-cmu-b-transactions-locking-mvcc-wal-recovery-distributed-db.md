# cmu-2 — cross-cutting principles

1. **The definition of "correct" with concurrency: the result must equal some serial order.** It does not have to be arrival order. Conflicts are pairs of operations on the same object where at least one is a write (RW, WR, WW), plus phantoms and write skew. If the dependency graph has no cycle, the schedule is serializable. [Pavlo #17]
2. **Read-modify-write is the canonical bug.** Two clients read a balance of 100, both write 75, and one update is lost. Fix it with a single atomic conditional UPDATE, or `SELECT ... FOR UPDATE` in a short transaction, or SERIALIZABLE plus retry. [#17, #18]
3. **Correctness first, speed second.** Retrofitting correctness onto a fast system is much harder than optimising a correct one. Even Postgres shipped a serializable bug for years. [#17, #22]
4. **Never trust isolation labels or defaults.** Postgres defaults to READ COMMITTED, MySQL to REPEATABLE READ. Oracle's "SERIALIZABLE" is really snapshot isolation. Postgres silently upgrades READ UNCOMMITTED. Snapshot isolation allows write skew. [#17, #20]
5. **Your application must handle aborts.** The DBMS can kill a transaction because of a deadlock, a validation failure or a timeout. Catch the error and retry, and keep transactions short: locks are held until commit, so no external API or LLM calls inside them. [#17, #18]
6. **The database cannot roll back the outside world.** Emails, payments and supplier calls made inside a transaction persist after it aborts. Record intent first, act after commit, make the action idempotent, and reconcile. [#17, #24]
7. **A commit acknowledgement can be lost after a successful commit.** Clients need a way to ask whether the commit happened, such as idempotency keys or status lookups. 2PC participants do the same: on recovery they ask the coordinator. [#17, #21, #24]
8. **Locking discipline matters more than locks.**
   - Strong strict 2PL means holding every lock until commit.
   - Deadlocks are unavoidable with locking. Handle them by detection (waits-for graph plus a victim) or prevention (wait-die or wound-wait).
   - In your own code, acquire resources in a consistent order.
   [#18]
9. **Pick lock granularity for the job.** Intention locks let one table lock replace a billion row locks. Use table-level locks for bulk loads. `FOR UPDATE SKIP LOCKED` turns Postgres into a safe multi-worker job queue. [#18]
10. **MVCC: readers do not block writers, but versions cost space.** Long-running or idle-in-transaction sessions pin garbage collection and cause bloat. Postgres append-only MVCC amplifies writes across every index, and VACUUM does not return space to the OS. Delta storage (MySQL, Oracle) is the better design. [#20]
11. **The write-ahead rule.** Log before data, and flush the log before acknowledging. This allows STEAL/NO-FORCE: commit costs one sequential append instead of random page writes. Group commit amortises fsync over about 5ms. Check that synchronous commit or fsync on commit is actually enabled; often it is not by default. [#21, #22]
12. **Shadow-copy then atomic swap.** Write the new version elsewhere and atomically flip one pointer or rename. This gives instant recovery at a runtime cost. It is ideal for file outputs and small single-writer systems (LMDB, SQLite journal). [#17, #21]
13. **Make recovery idempotent with sequence numbers.**
    - Stamp each unit of state with the last operation applied (pageLSN), and skip anything at or below it.
    - Log compensations (CLRs) so that a crash during rollback resumes rather than repeats.
    - Write "end" markers. Update the "last good checkpoint" pointer only after the checkpoint is complete.
    - Use fuzzy checkpoints that do not stop the world.
    [#22]
14. **Bound replay with checkpoints.** Choose frequency by recovery-time tolerance vs runtime I/O, often triggered by log volume rather than time. [#21, #22]
15. **The log is a reusable stream** for replication, change data capture (Debezium → Kafka) and audit. Do not dual-write to two systems. [#21]
16. **Distributed means slow and unreliable messages.** A garbage-collection pause looks like death. Shared-nothing gives locality. Shared-disk (S3 plus stateless compute plus caches) gives elasticity and is now dominant for analytics. [#23]
17. **Partition on the key queries filter or join on.** Hash partitioning suits point lookups, range partitioning suits scans. Consistent or rendezvous hashing minimises data movement when nodes change. Replicate small reference tables everywhere. [#23, #24]
18. **Replication is a latency/consistency dial.** Synchronous means no loss and higher latency. Asynchronous means a window of lost commits and stale replica reads. CAP covers partitions; PACELC adds latency vs consistency in normal operation. Choose per data type: money needs consistency, feeds can lag. [#23, #24]
19. **Commit agreement.** 2PC needs every participant and blocks on failure. Paxos and Raft need a majority and use leader leases. Use an existing Raft system; never build consensus yourself, and never use last-writer-wins for money. [#24]
20. **In analytics, data movement is the cost.**
    - Prune before reading (sparse indexes, zone maps or min-max, Bloom filters, Parquet footers).
    - Push predicates and projections down.
    - Broadcast small tables. Shuffle both sides only when unavoidable, and watch for skew.
    - Prefer predictable plans over clever ones; cardinality estimates are often wildly wrong.
    [#24, ClickHouse talk in #22, Firebolt talk in #18]

---

## Andy Pavlo (CMU Database Group) — #17 Concurrency Control Theory (https://www.youtube.com/watch?v=tMFAgvDViAI)
**A transaction is correct if the result equals some serial order of transactions. Concurrency control lets the database interleave work for speed while keeping that guarantee.**

- **Mental models / first principles**
  - Two separate problems make a "working" database unsafe: (1) a crash in the middle of a multi-step change, and (2) two clients changing the same data at once. Example: read balance $100, check that $25 is available, pay out, write back $75. If two clients sharing the account run this at the same moment, both write $75. The right answer is $50. This is the classic lost update from read-modify-write.
  - The simplest correct design: run one transaction at a time, copy the pages it touches, and swap a pointer at commit (shadow paging). It is safe and correct but has no parallelism. The whole field exists to keep that correctness while allowing interleaving.
  - Get correctness first and performance second. Adding correctness to a fast system later is very hard. Postgres still had a serializable-isolation bug fixed only a few years ago, after roughly 40 years of development.
  - The DB can only undo what it controls. Side effects inside a transaction (sending an email, calling an external API, "launching a missile") cannot be rolled back when it aborts.
  - Abstraction used for reasoning: a transaction is a sequence of reads and writes on named objects (A, B, ...), bounded by BEGIN and COMMIT/ABORT. The DB does not know what your application logic means.

- **Mechanisms**
  - **ACID**
    - **Atomicity** means all or nothing. An abort can come from the application or from the DBMS, for example on a conflict or lock timeout. The application must catch that error and decide whether to retry.
    - **Consistency** means integrity constraints (CHECK, NOT NULL, FK) that the application must declare. The DB cannot infer that "age < 0" is invalid.
    - **Isolation** means each transaction behaves as if it runs alone.
    - **Durability** means committed work survives crashes. Replication covers losing the machine itself.
  - Every single statement is a transaction. With autocommit on (the default in Postgres), the DB wraps each statement in an implicit BEGIN/COMMIT.
  - Atomicity is implemented by either:
    - **Logging (WAL):** record undo info (the old value) and redo info. Rule: the log record for a change must reach disk before the dirty page it describes. Normal operation is fast because log appends are sequential. Recovery is slower because the log must be replayed, so use checkpoints to cut replay length; fuzzy checkpoints let transactions keep running meanwhile. ARIES (IBM, early 90s) is the reference algorithm. Logs also double as audit trails; finance keeps about 7 years.
    - **Shadow paging:** copy pages and flip a pointer at commit. Normal operation is slower but recovery is instant, because uncommitted shadow pages are simply ignored. Used by LMDB, by SQLite in one mode, and by a 1970s Puerto Rican power utility that crashed several times a day. Pavlo's verdict: WAL almost always wins.
  - Physiological logging records byte-level diffs. Logical logging records the query and re-runs it, which takes as long as the original run.
  - Commit acknowledgement can be lost: the DB commits, then crashes before the client hears "OK". The transaction is still committed. The client must be able to find out what actually happened.
  - **Conflicts** are two operations from different transactions on the same object where at least one is a write:
    - Read-Write gives **unrepeatable reads**: reading A twice returns different values.
    - Write-Read gives **dirty reads**: reading uncommitted data that is later rolled back.
    - Write-Write gives **lost updates**, often from blind overwrites, e.g. ending with A from T2 and B from T1.
    - Further anomalies: **phantoms** (a range scan sees rows inserted or deleted) and **write skew** (under MVCC).
  - **Conflict serializability check**
    - Build a dependency (precedence) graph: one node per transaction, and an edge Ti→Tj when an operation in Ti conflicts with a later operation in Tj.
    - No cycle means the schedule is equivalent to a serial order, read off topologically. That order can differ from BEGIN order or even commit order, e.g. T2→T1→T3.
    - A cycle means the schedule is not serializable.
  - **View serializability** accepts more schedules (e.g. blind writes where only the last writer matters, or counting rows instead of summing). It needs knowledge of application meaning, so no real system implements it. The nesting is: serial ⊂ conflict-serializable ⊂ view-serializable ⊂ all schedules.

- **Decisions & trade-offs**
  - Pessimistic control (prevent conflicts up front with locks) suits workloads with many conflicts. Optimistic control (run freely and validate at commit) suits workloads where conflicts are rare.
  - WAL means fast runtime and slower recovery. Shadow paging means slow runtime and instant recovery.
  - Serializable does not guarantee arrival order. The DB may reorder concurrent transactions. Strict/external consistency (commit order matches real time, as in Spanner) is rare and expensive.
  - Eventual consistency (the NoSQL era, Bigtable-inspired) pushed correctness onto application developers. The industry largely reversed course. Spanner's paper says it is better for programmers to deal with performance problems from overusing transactions than to code around missing transactions.

- **Rules of thumb**
  - Treat read-check-write sequences (balance checks, seat or inventory counts) as one transaction, or use an atomic UPDATE ... WHERE balance >= x.
  - Application code must handle aborts and retry.
  - Do not trust an isolation label. Oracle's "SERIALIZABLE" is really snapshot isolation, which is weaker.
  - Never perform external side effects inside a DB transaction expecting rollback to undo them. Do them after commit, or make them idempotent and reconcilable.
  - **Escrow transactions** for hot counters: split inventory into per-region allotments (e.g. 50 seats East, 50 West) and coordinate only when an allotment runs out. Pavlo cites the Taylor Swift ticketing collapse as a hot-row contention example.

- **Failure modes & gotchas**
  - Two users run the same read-modify-write and one update is lost.
  - Interleaving a transfer with an interest calculation miscounts money: computing 6% interest while $100 is in flight loses $6.
  - A dirty read lets T2 commit work built on data T1 later aborts.
  - A long-running database replays a huge log after a crash if it never checkpoints.

- **Relevance to Sam's work**
  - Bookings and payments: the check-availability → pay → decrement pattern is exactly the lost-update example. The supplier API call is a non-rollbackable side effect, so record the intent in the DB first and reconcile afterwards.
  - Idempotency: commit acknowledgements can be lost, so clients need a way to check whether the commit happened, such as an idempotency key or a status lookup.
  - Pipelines: per-batch transactions give all-or-nothing loads. A crash mid-batch then leaves no half-written state.

## Andy Pavlo (CMU Database Group) — #18 Transactions with Two-Phase Locking + Firebolt talk (https://www.youtube.com/watch?v=drStlhNbfHI)
**Locks alone do not give serializability. The rule for when locks may be acquired and released does. Two-phase locking (2PL) is that rule, and deadlocks are the price.**

- **Mental models / first principles**
  - **Locks vs latches.**
    - Locks protect logical DB objects (database, table, page, tuple) and are held for the whole transaction. A central lock manager tracks them, and the DB must detect or prevent deadlocks because application queries are arbitrary.
    - Latches protect in-memory structures for microseconds and are embedded in the structure. Deadlock avoidance there comes from careful coding.
    - (OS "locks" are what DB people call latches.)
  - In real systems the transaction's future queries are unknown, so serializability must be enforced on-line. Static schedule analysis is not enough.
  - Locking without a protocol still breaks. Example: T1 locks A, reads, unlocks; T2 locks A, writes, unlocks; T1 relocks A and reads a different value. That is an unrepeatable read even though every access was locked.

- **Mechanisms**
  - **Lock modes.** Shared (S) is compatible with S. Exclusive (X) is compatible with nothing. Upgrades from S to X are allowed. Real systems have many more modes (Oracle row-exclusive, etc.).
  - **Lock manager.** A hash table of object → holders plus a wait queue. It is latched, so every lock request has real cost; this matters at 100k–1M txn/s. Lock state is not persisted, because after a crash all running transactions are dead anyway.
  - **2PL.**
    - Growing phase: acquire any locks.
    - Shrinking phase: the first release starts it, and no new locks may be acquired after that. Phases are per transaction.
    - 2PL guarantees conflict-serializable schedules.
    - Its flaw is cascading aborts: T2 reads what T1 wrote after T1 released its lock early. If T1 aborts, T2 must abort too. This wastes work and needs dependency tracking.
  - **Strong strict 2PL (also called rigorous).** Hold all locks until commit or abort, and release them atomically at commit. This is what real systems do, because SQL has no UNLOCK statement. It gives no cascading aborts, and undo is trivial: restore the last before-image.
    - Plain "strict" releases one class of locks early. Pavlo was unsure which class; textbooks differ.
  - **Deadlock detection.**
    - The lock manager keeps a waits-for graph and runs cycle detection periodically. Trade-off: checking very frequently burns CPU, while checking every 5 minutes leaves deadlocked transactions stuck for minutes.
    - Optimization: first look cheaply for 2-transaction cycles, and only run full cycle detection when needed.
    - **Victim selection** can be based on age (usually kill the newest; it keeps its original timestamp on restart so it is not starved), work done (do not kill the one that updated a billion rows), number of locks held, or the number of cascading aborts it would cause.
    - The victim releases its locks and undoes its changes. Then either the DB restarts it (stored procedure) or the application gets an exception and must retry.
  - **Savepoints.** Partial rollback to a savepoint can break a deadlock without killing the whole transaction. The application must set savepoints explicitly. A single statement can sometimes be undone and retried implicitly.
  - **Deadlock prevention** (no waits-for graph, only timestamps):
    - Wait-Die: an older requester may wait; a younger requester dies.
    - Wound-Wait: an older requester kills the holder; a younger requester waits.
    - All waiting goes in one direction by age, so cycles cannot form, much like latch crabbing always goes top-down.
    - A restarted transaction keeps its original timestamp, so it eventually becomes oldest and cannot starve.
    - Commercial systems (Oracle, DB2) offer both detection and prevention as tunable knobs. MySQL is simpler.
  - **Hierarchical locking.**
    - Locking a billion tuples individually costs more than updating them. Instead, lock at database, table, page, tuple or attribute level; a lock on a parent covers its children.
    - Tuple and table locks are universal. Attribute locks are rare (YugabyteDB). Early MongoDB took a database-wide write lock. SQLite allows a single writer.
    - **Intention locks** are hints on ancestors about what is held below:
      - IS: some descendant is held in S.
      - IX: some descendant is held in X.
      - SIX: the whole subtree is held in S and some descendant in X.
    - Protocol: take intention locks top-down, then the real S or X lock at the leaf.
    - Example: a reader takes IS on the table plus S on its tuple, a writer takes IX plus X, and they run in parallel. A full scan-and-update-one takes SIX plus X on one tuple; a concurrent single-tuple reader (IS) is fine, but a full-table S reader must wait.
    - **Lock escalation**: when you discover you need far more tuple locks than estimated, upgrade to a table-level lock.
    - DDL such as adding a column takes an X lock on the table.
  - **SQL hints.**
    - `SELECT ... FOR UPDATE` takes X locks during the read, for read-modify-write patterns. This avoids S→X upgrade failures and wasted reads. Postgres also has FOR SHARE, FOR NO KEY UPDATE and FOR KEY SHARE. MySQL has LOCK IN SHARE MODE.
    - `SKIP LOCKED` silently skips rows that are locked. Results are non-deterministic, but it is ideal for a DB-backed work queue: workers each grab the next unclaimed job without blocking each other.

- **Decisions & trade-offs**
  - Pessimistic 2PL forbids some schedules that are actually serializable. You give up parallelism for safety, which is the right default where money is involved.
  - Detection vs prevention: which is better depends on workload and hardware.
  - Lock granularity: coarse locks mean fewer lock-manager trips but less concurrency. Fine locks mean more concurrency but more overhead.
  - Most systems default to an isolation level below SERIALIZABLE. (Note from Pavlo: real banks use append-only ledgers plus summation rather than in-place balance updates.)

- **Rules of thumb**
  - Use FOR UPDATE on reads you will immediately update, such as a balance or an inventory row.
  - Build job queues on Postgres with `SELECT ... FOR UPDATE SKIP LOCKED LIMIT n`.
  - Acquire locks in a consistent global order in your own code to avoid deadlocks.
  - Always wrap transactions in retry-on-deadlock or retry-on-abort logic.
  - Keep transactions short. Locks are held to commit, so slow work inside a transaction (such as an LLM or external API call) blocks everyone else.

- **Firebolt guest talk (Benjamin Wagner) — "be wary of cardinality estimates"**
  - Context: an analytical, scale-out DB serving customer-facing apps. Queries take tens to hundreds of ms at hundreds to thousands of QPS, with repetitive query patterns. For them, predictable performance matters more than an optimal plan.
  - Unpredictability rarely comes from data or query changes. Load spikes should be handled with queueing and autoscaling. The usual culprit is an over-clever component, especially the planner, which can give 100–1000x gains but also the biggest regressions.
  - Their rules:
    - Use cardinality estimates only for join ordering, where the payoff is huge.
    - Avoid estimates elsewhere.
    - Let users pin join orders and aggregation strategies.
  - On Iceberg you often have only the row count. Magic selectivity formulas, such as the SQL Server heuristic, can give two tables identical estimates, so plans flip back and forth with small inserts.
  - Fix: propagate guaranteed lower and upper row-count bounds through the plan, and add a risk-minimizing pass that puts a side on the hash-join build side when its bounds prove it is small.
  - Reference: the VLDB test-of-time paper "How Good Are Query Optimizers, Really?"

- **Relevance to Sam's work**
  - Booking and inventory decrements: use SELECT ... FOR UPDATE or an atomic conditional UPDATE inside a short transaction, and never call the supplier API while holding the lock.
  - Pipeline job dispatch across parallel workers on one machine: a Postgres SKIP LOCKED queue gives safe concurrency without Redis.
  - Bulk loads should take a table-level lock rather than millions of row locks, which also means less RAM in the lock manager.

## Andy Pavlo (CMU Database Group) — #20 MVCC: Multi-Version Concurrency Control (https://www.youtube.com/watch?v=tUFha9-DuSk)
**MVCC never overwrites a row. It writes new versions stamped with timestamps, so readers see a consistent snapshot without blocking writers. The cost moves to version storage, garbage collection and index maintenance.**

- **Mental models / first principles**
  - MVCC is not a concurrency protocol by itself. It is a storage and versioning layer that you combine with timestamp ordering, OCC or 2PL.
  - Almost every modern DBMS uses some form of MVCC, even systems without multi-statement transactions.
  - A logical tuple (identified by primary key) maps to several physical versions. For any transaction, exactly one version is visible.
  - Key benefit: writers do not block readers, and readers do not block writers. In single-version 2PL, a shared lock blocks an exclusive lock.
  - Keeping all versions without garbage collection gives **time-travel queries** ("SELECT ... AS OF timestamp"). These are used for financial audits; SQL Server charges extra for it. Postgres had the feature in the 80s, then dropped it and added VACUUM because storage kept growing.

- **Mechanisms**
  - **Version metadata.** Each version has begin and end timestamps that mark when it is visible; an end of ∞ means it is the latest.
    - A transaction gets a timestamp at BEGIN and reads the version whose [begin, end) range contains it.
    - A write creates a new version with begin = its own timestamp and sets the old version's end to the same timestamp.
    - A separate **transaction status table** (active / committed / aborted) is needed, because timestamps alone do not say whether the creating transaction committed.
  - **Snapshot isolation (SI).** A transaction sees a consistent snapshot containing only transactions that committed before it started, so it never sees torn writes.
    - Write-write conflicts are resolved by **first writer wins**. The second writer aborts under basic timestamp ordering, or waits on the logical row's X lock under MVCC+2PL.
    - SI is susceptible to **write skew**. Jim Gray's marbles example: T1 turns all white marbles black while T2 turns all black marbles white. Under SI the colours just swap; a serial order would make them all one colour.
    - Write skew was missed by the 1992 SQL-standard isolation levels, which assumed 2PL. SI sits on a separate branch from read-committed → repeatable-read → serializable.
  - **Live demos.**
    - Postgres defaults to READ COMMITTED. After T1 commits, a blocked UPDATE in T2 re-reads the latest committed value, so 100 → 101 → 102 and no update is lost.
    - MySQL/InnoDB defaults to REPEATABLE READ: a plain SELECT keeps seeing the old snapshot after the other transaction commits.
    - Postgres stores xmin and xmax in each tuple. ctid is (page, slot), and new versions land in new slots.
    - Postgres treats READ UNCOMMITTED as READ COMMITTED; it upgrades you. Oracle does the opposite and downgrades a SERIALIZABLE request to SI.
    - Postgres deadlock detection (deadlock_timeout) killed one of two transactions that were cross-updating id 1 and id 2.
  - **Version storage options**
    - **Append-only** (Postgres): a full copy of the tuple goes into the same table space.
      - Oldest→newest chains are cheap to append but readers walk the chain to reach the latest version.
      - Newest→oldest chains give fast latest reads, but every update must repoint the indexes.
    - **Time-travel table** (SQL Server): the old full copy goes to a separate table and the main table is overwritten in place. GC then only touches the side table.
    - **Delta storage** (MySQL, Oracle: rollback/undo segments): store only the old values of changed columns, and overwrite the main tuple in place. Older versions are rebuilt by applying the deltas backwards. Much better when only 1 of many columns changes. Pavlo: this is the right way, and append-only is "the part of Postgres we hate the most".
  - **Garbage collection**
    - Keep a low watermark: the oldest active transaction timestamp. Versions invisible to everyone, and all versions from aborted transactions, can be reclaimed.
    - Tuple-level GC can be done two ways:
      - Background vacuum: Postgres keeps a dirty-page bitmap (visibility map) so it skips unchanged pages and avoids polluting the buffer pool.
      - Cooperative cleaning: readers prune old versions they walk past. This turns reads into physical writes. Rows nobody reads ("dusty corners") never get cleaned, so you still need a background vacuum.
    - Transaction-level GC: each transaction records the versions it invalidated, and they are reclaimed after commit once nothing can see them.
    - **Long-running transactions pin the watermark, so garbage cannot be collected.**
  - **Indexes**
    - The primary key index points at the head of the version chain. Changing the primary key is treated as delete + insert.
    - Secondary indexes can hold **physical pointers**, which must all be updated whenever the chain head moves, even if an unindexed column changed. Or they can hold **logical pointers** (MySQL stores the primary key), which adds an extra lookup but needs no index updates on a version change.
    - Uber's well-known blog post on moving from Postgres to MySQL came down largely to this write amplification on secondary indexes.
    - Indexes must allow duplicate physical keys across snapshots, e.g. delete then re-insert the same key while an old reader still needs the old version. Make them unique by adding version information or a record ID to the key.
    - Postgres puts multiple version entries in its indexes, which speeds scans but slows deletes.
  - **Deletes** use a deleted flag in the header or a tombstone version at the end of the chain. Physical removal is left to GC.
  - Postgres VACUUM marks space reusable but does not return it to the OS. Inserting a billion rows and then deleting them leaves the file bloated until VACUUM FULL or pg_repack.

- **Decisions & trade-offs**
  - Delta storage gives small writes and easy GC, but reading old versions costs reconstruction. Append-only is simple but causes bloat, index churn and vacuum load.
  - Newest→oldest chains suit OLTP that reads the latest version. Oldest→newest chains suit cooperative GC.
  - Logical vs physical index pointers: an extra lookup on reads vs index write amplification on every update.

- **Rules of thumb**
  - Know your DB's default isolation level: Postgres is READ COMMITTED and MySQL is REPEATABLE READ. They behave differently on read-modify-write.
  - Do not assume SI equals serializable. Guard invariants that span rows (e.g. "at least one seat or guide must stay available") with SELECT ... FOR UPDATE, an explicit constraint, or SERIALIZABLE.
  - Avoid long-open transactions and idle-in-transaction sessions. They block vacuum and cause bloat.
  - In Postgres, frequently updated wide rows with many indexes cause heavy write amplification. Keep hot mutable columns, such as status or counters, in narrow tables. Use pg_repack after mass deletes.

- **Failure modes & gotchas**
  - Write skew under SI or repeatable read.
  - A primary key deleted and re-inserted while old readers still exist.
  - Table bloat after bulk delete.
  - GC never reaches cold rows unless a background vacuum runs.

- **Relevance to Sam's work**
  - Bulk upserts on millions of rows (GeoNames/OSM) in Postgres create a full new tuple per update plus index entries. Prefer insert-into-new-table-then-swap, or COPY into staging and merge, then VACUUM/ANALYZE.
  - Booking status updates under READ COMMITTED are safe for single-row increments but not for checks across rows (e.g. room allotment sums). Use locks or SERIALIZABLE with retries.

## Andy Pavlo (CMU Database Group) — #21 Write-Ahead Logging + Shadow Paging (https://www.youtube.com/watch?v=CedEy54pe3g)
**Durability means "if I said committed, it survives a crash." Write-ahead logging (WAL) gets there with sequential log writes: log before data, and flush the log before acknowledging a commit.**

- **Mental models / first principles**
  - Setup: data lives on non-volatile storage. Changes happen in a volatile DRAM buffer pool. Disk I/O is page-granular, and hardware guarantees atomicity only for about 4KB, so an 8KB DB page can tear.
  - A commit acknowledgement is a promise. If the DB applied a change in memory, acknowledged it, and then lost power, the promise is broken.
  - Recovery has two halves:
    - What you do during normal operation so recovery is possible (this lecture).
    - What you do at restart (ARIES, next lecture).
  - Two primitives: **UNDO** removes effects of aborted or incomplete transactions. **REDO** reapplies effects of committed ones.
  - Commit-ack race: the DB may commit durably and then crash before the network acknowledgement arrives. The commit stands, and the client is supposed to check. (Pavlo: in practice almost nobody writes code that does this check.)

- **Mechanisms**
  - **Buffer pool policies**
    - **STEAL** means a dirty page of an uncommitted transaction may be evicted and overwrite disk. It then needs UNDO.
    - **NO-STEAL** forbids that. It is simple, but the transaction's working set must fit in RAM, so it cannot update a billion pages.
    - **FORCE** means all of a transaction's dirty pages are flushed before commit. That is lots of random I/O.
    - **NO-FORCE** does not flush pages at commit. It then needs REDO.
    - The eviction algorithm (LRU-K, ARC, ...) must respect whichever policy is in force.
  - **NO-STEAL + FORCE = shadow paging.**
    - There is a master page directory (committed state) and a shadow directory (the writer's copies). Commit = flush the shadow pages, then atomically overwrite one master pointer record.
    - Abort or crash means ignoring the shadow pages, so recovery is instant.
    - Costs: copying the page table or directory, random I/O, fragmentation, and garbage collection. System R abandoned it for these reasons. Sequential scans degrade as the file fragments.
    - Typically single-writer. LMDB swaps the B+tree root and copies only the path. CouchDB appends shadow pages and compacts. SQLite used it until 2010.
  - **SQLite rollback-journal mode.** Before modifying page P, copy the original P to a journal file and fsync. At restart, if the journal exists, copy its pages back over the DB file, then delete the journal. It is an undo-only, before-image log.
  - **WAL rule.**
    - A dirty page may not be written to disk until the log records describing its changes are on disk.
    - A transaction is not committed until all its log records, including COMMIT, are on disk.
    - This enables STEAL + NO-FORCE: commit costs one sequential log flush instead of random writes to N pages.
    - A log record holds <txnId, objectId, before-image (undo), after-image (redo)>, with BEGIN and COMMIT records around it.
    - The log buffer lives in memory, not the buffer pool. Uncommitted transactions' records can be flushed; without a COMMIT record they are undone at recovery.
  - **Group commit.**
    - An fsync takes about 1–5ms on SSD. If every commit fsyncs alone, throughput caps near 1000 txn/s.
    - Instead, batch commits over a short window (about 5ms) or until the buffer fills, using double or multiple log buffers so one fills while another flushes.
    - Rule of thumb: under 50ms per transaction is "good enough" for web workloads (ad auctions give about 40–50ms). HFT wants under 1ms and uses in-memory replicated systems.
  - **Log content types**
    - **Physical**: a byte diff at page and offset, for data and every index. Large.
    - **Logical**: the SQL statement itself. Small, since one statement can update a billion rows. But non-deterministic functions (NOW(), RANDOM()) must be replayed with the same values, concurrent execution order must be recorded, and recovery takes as long as the original query (a 1-hour query replays for an hour).
    - **Physiological** (most common): page id plus slot number plus a change description. The page can be reorganised internally and replay still works.
    - In-memory DBs often skip index logging and rebuild indexes at load.
  - **The WAL is a reusable stream.** Ship it to replicas (log shipping, where the replica "is always recovering") and to change data capture (CDC) tools such as Debezium → Kafka or Oracle GoldenGate, which feed warehouses and ETL. LSM trees still keep a WAL for the memtable and truncate it after SSTable flush. Note that logging happens redundantly at the DB, filesystem journal and SSD layers.
  - **Checkpoints** bound recovery so you do not replay a year of log.
    - Naive blocking checkpoint: pause all queries, flush the log, flush all dirty pages, write a CHECKPOINT record, resume.
    - At recovery: transactions committed before the checkpoint are ignored, those committed after are redone, and uncommitted ones are undone.
    - Problems: the system stalls, possibly writing terabytes of dirty pages, and you must still scan back to find transactions that were active. Fuzzy checkpoints (next lecture) fix both.
  - **Checkpoint frequency** is a pain-tolerance knob. Some systems use time (HFT about every 5 minutes); many use log volume (e.g. a checkpoint every ~512MB of log in MySQL-style systems), so an idle DB does no needless checkpoints.

- **Decisions & trade-offs**
  - STEAL/NO-FORCE (WAL) gives the fastest runtime and the slowest recovery. NO-STEAL/FORCE (shadow paging) gives a slow runtime and instant recovery. Pick WAL unless crashes are constant.
  - Logical logs are compact but replay slowly and need determinism. Physical logs are large but replay fast and mechanically.
  - Checkpoint often for fast recovery at the cost of runtime I/O, or rarely for the reverse.

- **Rules of thumb**
  - "Committed" is only as durable as the fsync setting. **Many systems do not have synchronous commit on by default**, leaving a few-ms window of loss. Set synchronous_commit=on (Postgres) or innodb_flush_log_at_trx_commit=1 (MySQL) for money and bookings.
  - Batch writes into fewer, larger transactions to amortise fsync. That is the client-side analogue of group commit.
  - Before writing your own "durable" file format, copy SQLite's pattern: write a journal or temp file, fsync, then atomically rename or swap a pointer.
  - Keep logs for 7 years only if regulation requires it (e.g. Sarbanes-Oxley for financial records).

- **Failure modes & gotchas**
  - Torn pages: a multi-page write is interrupted.
  - Acknowledging a commit before the log is flushed.
  - Logical replay with NOW() or RANDOM() gives different results.
  - Long-running idle transactions. Databricks auto-kills notebook transactions after 24h.
  - A blocking checkpoint stalls the whole system.

- **Relevance to Sam's work**
  - For long pipelines on a 16GB laptop, use the same idea: an append-only progress log (a checkpoint file of processed chunk IDs plus output offsets), written and fsynced before marking a chunk done. On restart, redo from the last checkpoint instead of from zero. Write outputs to a temp file and rename atomically, which is the shadow-paging idea, so a crash never leaves a half-written Parquet or CSV.
  - CDC (Debezium) is the clean way to stream changes from the bookings DB to analytics without dual-writes.

## Andy Pavlo (CMU Database Group) — #22 Database Crash Recovery (ARIES) + ClickHouse talk (https://www.youtube.com/watch?v=X2jc4qalNy0)
**ARIES recovers by replaying history first and then undoing what should not survive. Log sequence numbers (LSNs) and compensation log records (CLRs) make each step idempotent, so recovery can itself crash and restart safely.**

- **Mental models / first principles**
  - Goal at restart: keep every committed transaction's effects, remove every aborted or in-flight transaction's effects, and leave no torn writes.
  - ARIES (IBM DB2, 1992, about 70 pages) is the reference design. Its three ideas:
    - WAL with STEAL/NO-FORCE.
    - **Repeat history on redo**: redo everything, including losers and CLRs, even if it may already be on disk.
    - **Log the undos** as CLRs.
  - Pre-ARIES systems differed on whether to redo or undo first. ARIES fixes the order: Analysis → Redo → Undo.
  - Be paranoid first and optimise later. Redundant logging is deliberate. Postgres goes further: after each checkpoint, the first modification of a page logs a full page image, to survive torn pages.

- **Mechanisms**
  - **LSNs**
    - Each log record gets a globally unique, monotonically increasing LSN. LSNs give the physical order of changes; transaction timestamps give the logical order.
    - **pageLSN** (in the page header): the newest log record that modified the page.
    - **flushedLSN** (in memory): the highest LSN durably on disk.
    - **WAL rule as a check**: a page may be evicted only if pageLSN ≤ flushedLSN. Otherwise the buffer manager can force an early log flush.
    - **recLSN** (per dirty page, kept in the dirty page table): the oldest log record that dirtied the page since it was last flushed.
    - **lastLSN** (per transaction): the transaction's most recent record.
    - **MasterRecord** (on disk): the LSN of the last successful checkpoint BEGIN.
  - **Log record fields**: LSN, prevLSN (a per-transaction backward linked list, for speed rather than correctness), txnId, type, object, before value, after value, undoNextLSN.
  - **Commit**: write and flush COMMIT (possibly via group commit), acknowledge the client, then append TXN-END lazily. TXN-END means "nothing more will ever be needed for this transaction".
  - **Abort during normal operation**
    - Write ABORT and acknowledge immediately. Then walk the prevLSN chain backwards.
    - For each update, apply the inverse and write a **CLR** (before and after swapped, plus undoNextLSN = the next record to undo). Finish with TXN-END.
    - Under 2PL, keep the locks until the undo finishes.
    - CLRs need no immediate flush, and **CLRs are never undone**.
  - **Checkpoints**
    - Naive blocking checkpoints stall everything.
    - "Pause transactions mid-flight" checkpoints can write a page holding half of a transaction's changes, so the image is not consistent.
    - **Fuzzy checkpoints** write CHECKPOINT-BEGIN, then flush dirty pages while transactions continue. CHECKPOINT-END carries the **Active Transaction Table** (ATT: txnId, status Running / Committing / Undo-candidate, lastLSN) and the **Dirty Page Table** (DPT: pageId, recLSN) as of BEGIN. MasterRecord is updated only after END is durable.
  - **Recovery**
    1. **Analysis**: scan forward from MasterRecord's BEGIN to the end of the log, rebuilding the ATT and DPT.
       - TXN-END removes a transaction from the ATT.
       - Any other record adds its transaction, by default as an undo-candidate. COMMIT flips it to committed.
       - An unseen page touched by an update joins the DPT with recLSN set to that record.
    2. **Redo**: start at the smallest recLSN in the DPT and scan forward, reapplying updates and CLRs.
       - Skip a record if its page is not in the DPT, or its LSN is below the page's recLSN, or (after reading the page from disk) its LSN is at or below the pageLSN.
       - Otherwise apply it and set pageLSN = LSN. Write no new log records and do no flushing.
       - Committed transactions then get TXN-END.
    3. **Undo**: for transactions still marked undo-candidate, process their records in reverse LSN order across transactions using lastLSN and undoNextLSN. Write a CLR for each, then TXN-END.
  - **Crash during recovery**
    - Crash during Analysis: just redo the analysis, since it is read-only.
    - Crash during Redo: redo again, since redo is idempotent via pageLSN.
    - Crash during Undo: the existing CLRs are redone, and undoNextLSN resumes exactly where it stopped. No work is undone twice.
  - **Optimisations**
    - Do not flush at every TXN-END during redo.
    - Lazy undo: queue pending undos per page and apply them when the page is next touched, so the DB opens for business sooner.
    - Avoid long-running transactions; they make undo long.

- **Decisions & trade-offs**
  - Verbose logging (CLRs carry both values, full page images) trades disk space for robustness.
  - Checkpoint frequency trades runtime I/O against recovery time, as in lecture 21.

- **Rules of thumb (generalisable)**
  - Make every recovery step **idempotent**. Stamp each unit of state with the sequence number of the last operation applied, and skip operations at or below it.
  - Log compensations, rather than silently reverting. That way a crash during rollback resumes instead of double-applying.
  - Record "end" markers so restart knows which work units are fully finished and can be forgotten.
  - Persist a pointer to the last good checkpoint only after the checkpoint is complete.

- **ClickHouse guest talk (Robert Schulze) — data organisation and pruning**
  - Open-source columnar distributed OLAP DB in C++, built for append-heavy data (events, logs, time series, finance). Updates and deletes work but are slower.
  - **Parts**
    - Each INSERT writes an immutable "part": a file plus metadata, sorted by the user-chosen primary key. Inserts need no global synchronisation or uniqueness lookups, so they are disk-bound.
    - An async-insert buffer batches small inserts.
    - Background merges combine sorted parts with a k-way merge sort. This is LSM-like, but parts are flat rather than levelled, so scans parallelise over all parts and merges can pick any parts. No tombstones are used.
  - **Pruning**
    - Parts are split into **granules of 8,192 rows**, the smallest unit read from disk.
    - A **sparse primary index** stores only the first key of each granule, so about 1,000 entries cover about 8M rows. It is pinned in RAM. Queries binary-search the index, then load and filter only the candidate granules. This matters on object storage, where each access costs tens of ms.
    - **Projections**: alternative sort orders per part, chosen by the optimiser. They are powerful but double or triple storage.
    - **Skipping indices** (min-max, set of unique values, Bloom filter) annotate granules so whole blocks can be skipped. Cheap, but they depend on the data distribution.
  - Joins historically took a back seat because users denormalised into one fact table. Join reordering was added recently.

- **Relevance to Sam's work**
  - For resumable data pipelines, adopt LSN-style idempotency: each output partition records the last input chunk or offset applied, and restart skips work already done. Rollback steps get logged too.
  - For multi-million-row location and event analytics on a laptop, the ClickHouse lesson applies directly:
    - Sort data by the most common filter key (country/admin code, geohash, time).
    - Store it in row groups or granules with min/max stats, such as Parquet row groups or DuckDB zonemaps.
    - Prune before reading.
    - Prefer append plus periodic compaction over in-place updates.

## Andy Pavlo (CMU Database Group) — #23 Distributed Databases Pt. 1 (https://www.youtube.com/watch?v=IFLQBWY6dlE)
**Going distributed keeps every single-node problem and adds slow, unreliable communication. The core design choices are the architecture, partitioning, replication and coordination, and each one trades performance against elasticity and consistency.**

- **Mental models / first principles**
  - A parallel DB (one box, many cores) can assume fast, reliable communication. A distributed DB cannot.
    - Messages cross networks at 100–300ms worldwide, or seconds to minutes when lost.
    - Messages can be dropped.
    - A node can look dead while it is only pausing, for example a JVM or Go garbage collector freezing it for 10–30s.
  - Everything from single-node DBs (buffer pools, concurrency control, logging, optimisation) still applies. Distribution only makes each piece harder. Nodes may also differ in hardware.
  - Distribution is for performance (more resources) and for fault tolerance and availability.
  - SQL must stay location-transparent. The same query that runs on one node must work on the cluster, and the app sends one query and gets one result set back.

- **Mechanisms**
  - **Architectures**
    - **Shared-everything**: a single node, which is everything covered so far.
    - **Shared-nothing**: each node has its own CPU, memory and disk, and nodes talk only over the network (TCP; sometimes UDP or RDMA).
      - Best performance when queries are local to a node.
      - Hard to scale elastically: adding a node means physically moving data and updating the catalog.
      - This was the 1980s–2010s default (Stonebraker's 1986 argument).
    - **Shared-disk**: each node has its own CPU and memory, and the primary copy of the data lives in shared storage (S3, Azure Blob, GCS, distributed FS). Local SSD is only a cache.
      - Nodes are effectively stateless.
      - Adding a node is a catalog reassignment with no data movement.
      - Storage capacity is effectively unlimited ("your credit card runs out before S3 does").
      - Costs: remote-storage latency, and keeping cached copies coherent after updates. A writer must notify or invalidate other nodes' cached pages.
      - Most newer systems use it, especially OLAP (Snowflake, Databricks, Firebolt, Yellowbrick).
      - Snowflake's bet: let AWS make S3 better, put caches in front of it, and spend engineering effort on the query engine.
    - **Shared-memory** across nodes is essentially only found in high-performance computing.
    - Spanner blurs the line: shared-nothing nodes on top of the Colossus distributed FS.
  - **Push vs pull**
    - Push the query (a plan fragment) to the data, e.g. a local join computed where the partition lives, sending back partial results.
    - Or pull the data to the query.
    - The lines blur when storage can compute. S3 Select pushed filters into S3 to cut egress costs; it is no longer offered to new customers, which Pavlo thinks was a bad database decision.
    - Without storage compute, use metadata: Parquet footers, zone maps, and predicate/projection pushdown to fetch the minimum bytes. Parquet (about 2013) is now the de facto lake format; newer formats such as Vortex compete with it.
  - **Catalog**
    - Stores the schema plus the partition map and topology. It needs transactional guarantees too.
    - The client either asks the catalog where data lives, or sends the query to any node, which routes it.
    - Early MongoDB auto-sharding moved chunks without keeping the catalog in sync, causing duplicate or incorrect results, or it locked the whole DB.
  - **Partitioning (sharding)**
    - Partitions are physical in shared-nothing systems and logical ownership in shared-disk systems.
    - **Table partitioning**: one table per node. Good for isolating an insert-only log table.
    - **Replicating small dimension tables** to every node: e.g. about 35k ZIP codes, updated about four times a year.
    - **Horizontal partitioning**: the goal is balanced data and balanced load. A node with little data but hot queries is overloaded.
      - **Hash**: hash(key) mod N. Great for equality lookups on the key. Range scans and queries without the key must broadcast to all nodes. Changing N reshuffles almost everything.
      - **Range**: good for range scans, but needs rebalancing.
      - **Predicate**: WHERE-clause rules. Rare.
      - **Round-robin**.
    - **Consistent hashing** (Chord/MIT ~1997; Dynamo):
      - Keys hash onto a ring, and each key belongs to the next node clockwise. Adding a node only splits one neighbour's arc.
      - Virtual nodes (many points per physical node) smooth out skew.
      - Replicas are the next k nodes on the ring.
      - Used by Snowflake (assigning files to workers), Cassandra, Couchbase and Riak. DynamoDB later moved away from it.
    - **Rendezvous (highest-random-weight) hashing**: for each key compute hash(key‖nodeId) for every node and pick the highest. Adding a node moves only the keys it now wins. Consistent hashing is a special case.
  - **Replication**
    - Most "distributed" deployments are really a single primary plus replicas, e.g. Postgres with streaming replicas. Replicas give failover (leader election) and read offloading, but **reads on replicas can be stale**.
    - **Primary-replica**: all writes for an object go to its primary, which propagates them. Leader election (Paxos/Raft) runs when the primary dies.
    - **Multi-primary**: any node can write any object. Conflicts must be resolved, e.g. two transactions writing two objects each must not interleave.
    - **K-safety**: the minimum number of live copies required. Below it, the system stops accepting writes rather than risk losing data.
    - **Synchronous (strong) propagation**: the primary waits for replicas to flush and acknowledge before telling the client it committed. **Asynchronous (eventual)**: acknowledge immediately, which leaves a window where a committed write is lost if the primary dies. A middle ground waits for a quorum (e.g. 2 of 3).
    - **Propagation timing**: continuous (ship each change as it happens; most common) vs on-commit (batch everything at commit; no remote rollback needed but a longer commit).
  - **Coordinating distributed transactions**
    - **Centralized coordinator**: the classic TP monitor (BEA Tuxedo, Transarc from CMU's Andrew project, the X/Open XA standard). It holds the global lock table, and partitions ask it whether a commit is safe.
    - **Middleware router**: looks like a single DB and shards behind it, e.g. Google's sharded MySQL before Spanner, and Facebook's MySQL fleet.
    - **Decentralized**: any node can be the leader for a transaction and coordinates the commit with participants (details in lecture 24).
    - **Federated DBs** (one SQL front-end over heterogeneous DBs: Trino/Presto connectors, Postgres FDW) often degrade to SELECT * pulls because pushdown is not possible. Pavlo says they do not work well.
    - Transactions that touch a single node are easy. Multi-node transactions bring back 2PL/OCC, deadlocks and commit agreement, made worse across WAN regions.

- **Decisions & trade-offs**
  - Shared-nothing gives locality and speed. Shared-disk gives elasticity, simplicity and cheap storage.
  - Hash partitioning suits point lookups. Range partitioning suits scans.
  - Consistent and rendezvous hashing minimise data movement on membership changes.
  - Synchronous replication means no loss but latency bound by the slowest replica. Asynchronous means fast but can lose recent commits on failover.

- **Rules of thumb**
  - Pick the partition key that most queries filter on with equality. Otherwise every query scatters to all nodes.
  - Replicate small, rarely changing reference tables (countries, currencies, airport codes) everywhere to avoid distributed joins.
  - Send reads to replicas only where stale reads are acceptable. Read-your-writes flows (just booked → show booking) must hit the primary.
  - Expect GC pauses and slow nodes. Timeouts are not proof that a node is dead.

- **Relevance to Sam's work**
  - Sharding millions of location rows across parallel workers on one laptop works the same way. Hash-partition by a stable key (geonameid, country code) so each worker owns its partition with no shared state. Use rendezvous hashing if the worker count changes between runs.
  - Shared-disk thinking suits a laptop: data in Parquet on disk or S3, stateless compute (DuckDB/Polars) pulling only the needed row groups via footer stats.

## Andy Pavlo (CMU Database Group) — #24 Distributed Databases Pt. 2 (https://www.youtube.com/watch?v=pQh5fka3FC0)
**Distributed OLTP comes down to getting nodes to agree on commits: two-phase commit (2PC) needs everyone, Paxos and Raft need a majority. Distributed OLAP comes down to moving as little data as possible for joins, and shuffles are the main cost.**

- **Mental models / first principles**
  - The goal is still one logical database with ACID semantics. Distribution is the same problem "in hard mode": nodes die, and messages are late or lost.
  - The design assumes nodes you control are well-behaved (crash faults only). Malicious nodes would need Byzantine fault tolerance (BFT), as blockchains use. Pavlo: that is 5–10x slower and has no practical use outside crypto, so don't use blockchain databases.
  - An "atomic commit protocol" is the database name for what distributed-systems people call consensus over a replicated state machine. The state is the commit order of transactions, recorded in a log so replay gives the same state.
  - OLTP targets under 50ms per transaction. OLAP targets under about 1s and up to minutes. Each still needs a bit of the other's machinery.

- **Mechanisms**
  - **Two-phase commit** (credited to Jim Gray, who said it came from contract law)
    - The coordinator is the node the application commits through. Participants are only the nodes the transaction touched.
    - **Phase 1 (prepare)**: each participant validates using its local concurrency control and votes OK or ABORT.
    - **Phase 2 (commit or abort)**: the coordinator broadcasts the outcome. The client is told "committed" only after every participant acknowledges.
    - **One ABORT vote** sends everyone to phase 2 abort immediately, and the client is told at once. Cleanup happens later.
    - If a participant is unreachable (it crashed, or is in a 30s JVM GC pause, which a ping cannot detect, so heartbeats must go to the application layer), the coordinator waits until a timeout and then aborts.
    - If a node misbehaves in phase 2, that node is faulty. The transaction stays committed and a replacement replays the log.
    - Every inbound and outbound vote is written to a WAL, either the data log or a separate coordination log. Ideally it is flushed before replying; many systems skip that flush, which makes recovery longer but does not lose transactions.
    - **Recovery**: a participant that wakes in the prepared state asks the coordinator for the outcome. If the coordinator crashes, the simple rule is to abort and elect a new coordinator. No other transaction can commit until this one is resolved, so the protocol blocks.
    - **Optimisations**:
      - Early prepare voting: piggyback the vote on the last query. This needs a stored procedure, because the system must know it is the final query.
      - **Early acknowledgement** (common): tell the client "committed" once phase 1 passes unanimously, and finish phase 2 asynchronously.
  - **Paxos** (Lamport 1989; the paper is written as Greek-island archaeology and is famously unreadable; read "Paxos Made Live" from Google instead)
    - Roles are proposers and acceptors, plus learners who observe only.
    - It needs only a **majority** to accept, so the system stays live if a minority is down. A recovered node catches up from the log.
    - A newer proposal (n+1) pre-empts older ones. Competing proposers can starve each other, which exponential backoff helps.
    - **Multi-Paxos**: elect a leader with a lease. Only the leader proposes; leader election itself goes through Paxos.
    - Lease lengths: Spanner about 10s, TiDB about 10–15s, CockroachDB about 5 minutes. Short leases mean more election overhead; long leases mean longer stalls after a leader failure.
    - Gray and Lamport (2006) showed 2PC is a degenerate case of Paxos.
  - **Raft**: Paxos-like, but the leader must be the node with the most up-to-date log; it uses followers rather than learners. It is now more common because open-source implementations are plentiful. Viewstamped Replication (1988) and ZAB (ZooKeeper) are siblings.
  - Open point Pavlo deferred: under majority commit, a lagging participant cannot veto. So integrity checks such as constraints must happen before the commit protocol, or deterministic systems run validation first.
  - **CAP** (Brewer): during a network partition, choose Consistency or Availability.
    - Split-brain example: a primary in one data centre and a replica in another lose contact. The replica elects itself primary and both accept writes, which must be reconciled when the partition heals.
    - Transactional SQL and NewSQL systems choose C: without a quorum they reject writes, maybe still allowing reads.
    - NoSQL systems chose A and reconcile with last-writer-wins on timestamps, or vector clocks that return all versions for the client to merge (Riak). Pavlo: don't do that; application programmers cannot reason about it.
  - **PACELC** extends CAP: even without a partition, you trade Latency against Consistency. How long does the primary wait for distant replica acknowledgements, e.g. across the Atlantic? Use a timeout plus an abort, or accept transient inconsistency.
    - Choose per data type: money needs consistency, a social post can lag.
    - Facebook's old single-US-write-region trick: store the user's own fresh post in a cookie, so their own refresh shows it while replication catches up. That is read-your-writes faked at the application layer.
  - **Distributed joins** (local hash, sort-merge or nested-loop joins underneath; the goal is minimal data movement)
    1. One table partitioned on the join key, the other small and **replicated** everywhere: local joins, then a cheap union at the coordinator.
    2. Both tables **co-partitioned** on the join key: local joins, then union.
    3. **Broadcast join**: the smaller table is copied all-to-all, so every node holds it in full, then local joins. On shared-disk, have one node read from S3 and fan out, because S3 charges per read.
    4. **Shuffle join**: both tables are too big and partitioned on other keys, so both are repartitioned (hash or range) on the join key, then joined locally. This is MapReduce's shuffle, which databases did first as the "exchange/repartition" operator.
  - **Semi-join**: send only what is needed. Filter the dimension table (e.g. zip=15213), ship only the qualifying keys, or a **Bloom filter** of them, to the fact-table node, filter and project there (e.g. only price), then ship the smaller result. The optimiser does this automatically.
  - **Shuffle is the most expensive part of distributed OLAP.** It handles skew and rebalancing.
    - BigQuery shuffles at every pipeline breaker, using in-memory shuffle on specialised hardware. Shuffle stages act as checkpoints where the number of workers can be changed mid-query (e.g. 4 producers → 3 consumers), with spill to shared disk.
    - Pluggable open-source shuffle services exist (Apache Celeborn, Uniffle).
    - Because shuffle hides data location, operators can stay non-distribution-aware.

- **Decisions & trade-offs**
  - 2PC is simple but blocks when any node fails. Paxos and Raft are non-blocking with a majority, at the cost of complexity and leader leases.
  - Consistency vs availability during partitions. Consistency vs latency normally.
  - Broadcast when one side is small. Shuffle when both are large. Avoid both by co-partitioning on common join keys or replicating small dimension tables.

- **Rules of thumb**
  - Avoid distributed transactions where you can. Design partitioning so most transactions touch one node.
  - If you must coordinate, use an existing Raft-based system (etcd, CockroachDB, YugabyteDB, TiDB). Do not hand-roll consensus.
  - Never use last-writer-wins for money or inventory.
  - Push filters and projections down before any data crosses the network. Use Bloom filters for semi-joins.
  - Watch for skew: one hot partition idles the rest of the cluster.

- **Relevance to Sam's work**
  - **Bookings across a supplier API and your own DB form a distributed transaction with an untrusted participant.** 2PC is impossible, so use a saga-style flow:
    1. Write a PENDING booking with an idempotency key.
    2. Call the supplier with that key.
    3. Record CONFIRMED or FAILED.
    4. Reconcile periodically.
    These are the 2PC ideas (log every message, the participant asks the coordinator for the outcome after a crash) mapped onto API calls.
  - Joining a huge location table with small reference tables (countries, admin codes, supplier mappings) on one laptop: broadcast the small table (it fits in memory as a hash map) and stream the big one in chunks. For big × big joins, hash-partition both on the key to disk first (a manual shuffle), then join partition by partition so each pair fits in 16GB.
