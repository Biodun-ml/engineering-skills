# cmu-1 — cross-cutting principles
(Source: Andy Pavlo, CMU 15-445 Intro to Database Systems, Fall 2025 lectures #03, #04, #05, #06, #08, #13, #15.)

1. **Count I/Os, not instructions; prefer sequential over random.** Storage is block-addressable (touching 1 byte costs a 4KB+ block), SSD is ~100x+ slower than DRAM, and random I/O is far worse than sequential. Design every algorithm to read big contiguous chunks and batch/sort random accesses. [#03 Storage, #04 Buffer Pools]
2. **Manage data larger than memory deliberately.** Budget memory up front, keep a bounded working set, and move fixed-size chunks between disk and RAM yourself; if the working set exceeds RAM nothing saves you. [#03, #04]
3. **Temporal locality: do all the work on data while it is loaded.** Don't re-read a page you could have finished with in one pass; ride a tuple as far up the pipeline as possible; let concurrent scans share one pass. [#04, #05 scan sharing, #13 pipelines]
4. **The application knows its access pattern better than the OS.** mmap and the OS page cache cause uncontrolled writeback, stalls, scattered error handling and double buffering; own your cache, prefetching and I/O scheduling. [#04]
5. **Durability is paranoid work.** fsync can fail and older Linux lied after failure (fsyncgate); hardware may hold writes in battery-backed caches; pages larger than the atomic hardware page can tear. Treat fsync failure as fatal, checksum pages, write log before data. [#03, #04]
6. **Use indirection to keep changes local.** Slot arrays and logical record IDs let data move inside a page without updating indexes; never let application code depend on physical row IDs. [#03, #05]
7. **Caches must be scan-resistant and remember history.** Plain LRU/CLOCK are flushed by one big scan; LFU forgets time. Use LRU-K, two-region (probation/protected) lists, ARC, ghost lists and separate pools or ring buffers for bulk scans. [#04, #05]
8. **Pick the storage layout from the workload.** Row/heap for OLTP point access and whole-row updates; index-organized for primary-key lookups; LSM (append + compaction, tombstones) for write/ingest-heavy or append-only storage; columnar/PAX (Parquet/ORC) for analytics. HTAP rarely wins both. [#05, #06]
9. **Append, then compact.** Turning updates into sequential appends of versioned records plus a background k-way merge that keeps the newest version per key (and honours delete tombstones) makes writes fast, but costs write amplification and compaction I/O that competes with foreground work. [#05 LSM; #08 B-epsilon trees]
10. **Read only what you need.** Columnar layouts, projection pushdown, late materialization and zone maps (per-block min/max stats) let queries skip columns and whole blocks. Never SELECT * for analytics. [#06, #13, #15]
11. **Compress with domain-aware encodings and compute on compressed data.** Dictionary (order-preserving), RLE (after sorting), bit packing with patches, delta (+RLE), bitmaps for low cardinality; keep values fixed-width so offsets work; lossless unless the user explicitly accepts loss. [#06]
12. **Exact numbers need exact types.** Floats can't represent 0.1+0.2 exactly; money and anything reconciled must use fixed-point DECIMAL/NUMERIC or integer minor units, accepting the CPU cost. [#03]
13. **B+trees are the default index: high fan-out, sorted leaves, O(log n), range and prefix capable.** Composite key order matters (prefix rule); make non-unique keys unique with a tiebreaker; when fetching via a secondary index, sort record IDs by page first; hash indexes only serve equality. [#08, #13]
14. **Batch your execution.** Tuple-at-a-time (iterator) is simple and good for OLTP; vectorized batches of ~1K rows remove per-row call overhead and enable SIMD, which is why every modern analytic engine (DuckDB, Snowflake, ClickHouse) uses it. Specialize/JIT hot expressions, fold constants, dedupe subexpressions. [#13]
15. **Beware mutating what you iterate (Halloween problem).** An update that moves rows ahead of the scan cursor processes them again; snapshot IDs or track processed items. [#13]
16. **Filter early, project early, join small-into-big, never cross join.** Correct plan choice moved a toy query from ~2M I/Os to ~37. Keep predicates sargable (no functions/arithmetic on indexed columns) and confirm with EXPLAIN; optimizers miss "obvious" rewrites like val+0. [#15]
17. **Rewrites must be provably equivalent.** Optimizers rely on relational-algebra rules; NULLs, outer joins, casts and overflow are where "obvious" rewrites go wrong. [#15]
18. **Deterministic budgets beat wall-clock budgets when reproducibility matters.** SQL Server bounds optimization by number of transformations, so plans don't change with machine load. [#15]
19. **Background maintenance always competes with foreground work.** Dirty-page flushing, LSM compaction and lazy B+tree merges all need throttling policies; there is no universally correct setting. [#04, #05, #08]
20. **Constants matter, not just big-O.** A replacement policy, encoding or expression evaluator that is asymptotically fine but 10x slower per operation loses in practice. [#04, #13]

## Andy Pavlo (CMU Database Group) — #03 Database Storage: Files, Pages, Tuples (https://www.youtube.com/watch?v=PRLXdIMJhOg)
**A disk-oriented database is just files of fixed-size pages, with slotted pages and record IDs giving indirection so data can move without breaking pointers.**

**Mental models / first principles**
- A DBMS is a stack of layers with narrow APIs: disk/storage manager -> buffer pool manager -> access methods -> operators/execution -> planner/optimizer. Each layer gives guarantees to the one above. Build and reason bottom-up.
- "Disk-oriented" architecture: the authoritative copy lives on non-volatile, block-addressable storage (SSD, HDD, S3); you can only operate on data after pulling it into volatile, byte-addressable DRAM. The core job is to give the illusion of more memory than you have: manage a database far bigger than RAM (8GB RAM, terabyte DB) by moving pages back and forth cleverly and avoiding stalls.
- Block addressability means touching one byte costs a whole block (typically 4KB). Every design choice flows from minimising disk reads/writes and maximising sequential over random I/O.
- If the working set of the running queries exceeds memory, nothing saves you; design so the common case fits.

**Mechanisms**
- Latency ladder (Jim Gray / Jeff Dean table): DRAM ~100ns; SSD ~16-50us (two-plus orders of magnitude slower); HDD and network storage far worse; tape absurd. Gray's trick: read "ns" as "seconds" — L1 = 1 s, tape = ~31 years.
- Random vs sequential I/O: random read on a good SSD ~80-100us; sequential throughput is vastly higher. Example: MySQL's doublewrite buffer writes all dirty pages sequentially first, then does the random in-place writes in the background, so foreground work isn't blocked on random I/O.
- Three page notions: hardware page (~4KB, the largest unit the device writes atomically), OS page (4KB, huge pages 2MB-1GB), database page (512B-16KB: SQLite/Oracle/DB2/RocksDB/WiredTiger 4KB; Postgres/SQL Server 8KB; MySQL up to 16KB). If the DB page is larger than the atomic hardware page, a crash can tear it — extra steps are needed before declaring a write durable.
- Page size trade-off: read-heavy/scan-heavy workloads prefer bigger pages (each fetch brings more useful rows); write-heavy prefer small (4-16KB) because updating one byte forces rewriting the whole page.
- Heap file = unordered collection of pages; API: create/get/write/delete page and iterate all pages. Locate a page by arithmetic (offset = page_id x page_size) for a single file, or via a page directory (a mini-database tracking which file/location holds each page, plus free-space info and empty pages).
- Page directory must be kept in sync with data pages; in memory strictly, on disk it can be looser because it can be rebuilt from page metadata after a crash — the trade-off is runtime speed vs recovery time.
- Page header: page size, checksum, software version (layout migrations), visibility/transaction info, compression/encoding, sometimes the schema (Oracle makes pages self-contained), and summaries/sketches (e.g. counts of values so a scan can skip the page).
- Slotted pages: header + slot array growing forward, tuple bytes growing backward from the end; full when they meet. The slot array maps slot -> offset, so tuples can be compacted or moved inside the page without touching indexes. Naive "append fixed-length tuples and keep a count" breaks on deletes and variable-length data.
- Record ID = (file, page, slot); Postgres ctid is 6 bytes, Ingres 4 bytes; usually derived not stored. SQLite stores a monotonic rowid and uses it as the hidden key. Applications must never rely on record IDs — they change on compaction/vacuum.
- Tuple = header (null bitmap, visibility info) + attributes in declared order; schema lives in the catalog, not per-tuple (another reason not to mix tables in a page). Values are word-aligned (64-bit) via padding; reordering columns for packing exists in research but is rare.
- Types: ints/floats are native C types (IEEE-754 for floats); timestamps typically micro/milliseconds since Unix epoch; varchars inline with length prefix, or pointer to overflow pages when large.
- Floats are inexact: 0.1+0.2 printed at high precision != 0.3. Fixed-point NUMERIC/DECIMAL (Postgres stores a struct with scale/sign/weight plus digit array) is exact but addition is a long branchy routine — much slower than one CPU instruction.
- NULLs: (1) per-tuple null bitmap in header — standard for row stores; (2) sentinel value (e.g. INT_MIN means null) — common in column stores/in-memory DBs; (3) per-attribute flag byte — wastes space due to alignment, don't do it.
- Large values: overflow pages (Postgres TOASTs values >2KB though pages are 8KB; MySQL/SQL Server around half a page), chained if needed, optionally compressed. "German strings": keep length + a prefix inline next to the pointer so predicates like prefix matches can be resolved without fetching the overflow page. External value storage (Oracle BFILE, SQL Server FILESTREAM) stores a URI to a file outside the DB, without the DB's durability/transaction guarantees. Old Gray study suggested ~128KB+ blobs go outside (note: heuristic is dated).

**Decisions & trade-offs**
- Off-the-shelf filesystems (ext4 etc.) are good enough; custom raw-device filesystems (Oracle ASM) buy ~15% at huge engineering cost.
- Let the DBMS, not the OS, schedule its reads/writes — the OS "will always do a bad job" for DB access patterns.
- Replication of pages is not the storage manager's job: either below it (RAID, S3) or above it (logical replicas).
- Compaction in-page is cheap once the page is in memory; reading it from disk is the real cost. Postgres doesn't compact on update, SQL Server does.

**Rules of thumb**
- Count I/Os, not CPU instructions, for disk-resident data. Prefer sequential writes; batch random ones in the background.
- Add an indirection layer (slot array, logical IDs) so physical moves stay local ("small blast radius").
- Checksum every page so corruption is detected on read.
- Never use floating point for money; use fixed-point decimal (or integer minor units) and accept the CPU cost.

**Failure modes & gotchas**
- Torn writes when DB page > atomic hardware page.
- Lost/corrupt page directory makes recovery slow or impossible.
- Relying on physical row IDs (ctid/rowid) in application code.
- Endianness when copying raw DB files between architectures (SQLite sidesteps by storing as strings).
- Collation/encoding of strings (ASCII vs UTF-8/16) affects sorting and comparisons.

**Relevance to Sam**
- Payments/bookings: store prices and FX amounts as DECIMAL/NUMERIC or integer minor units, never float; rounding drift breaks reconciliation with suppliers.
- 16GB laptop: the whole lecture is about managing data larger than RAM — chunk by page-like blocks, read sequentially, keep only the working set resident.
- Big text blobs (supplier descriptions, JSON payloads) inflate rows; keep them in a separate table/column or external store and keep a short prefix/hash inline for filtering.
- File formats like Parquet are open, cross-system formats, unlike proprietary DB files — better for pipeline interchange.

## Andy Pavlo (CMU Database Group) — #04 Memory Management & Buffer Pools (https://www.youtube.com/watch?v=8-2yv4z0VZc)
**The DBMS must own its memory: a buffer pool with pinning, a smart replacement policy, background flushing and its own I/O scheduling beats delegating to the OS (mmap, page cache) every time.**

**Mental models / first principles**
- Two levers: spatial control (place data that is used together close together on disk so reads are sequential) and temporal control (once a page is in memory, do as much work on it as possible before giving it up; never re-read the same page twice when you could have done both jobs in one pass).
- The buffer pool is the DBMS's cache/virtual-memory system, built in user space because the DBMS knows the queries, transactions and dependencies and the OS doesn't. "The DBMS can always make better decisions than the OS" is a recurring theme.
- Ideally allocate all memory at startup (e.g. 80GB of a 100GB box) and never go back to the OS; out-of-memory should surface at boot, not mid-query. Use pools for everything heap-allocated: data pages, indexes, sort/join scratch (may spill to disk), log buffers. Don't co-locate other heavy processes on a DB box.

**Mechanisms**
- Buffer pool = array of fixed-size frames (frame size = page size). The page table is an in-memory hash map page_id -> frame plus metadata: dirty flag, pin/reference count, access-tracking info. The page directory (on disk) maps page_id -> disk location; the page table needn't be durable (some serverless systems persist it to prewarm after restart).
- Fetch: look up page table; hit -> pin and return pointer; miss -> pick a free frame (or evict), take a latch on the slot so no one else claims it, read from disk, install, unpin when the caller says it's done. A pinned page can never be evicted; pointers stay valid until released.
- Locks vs latches: locks protect logical objects (tuples, tables) for transactions, have a lock manager, deadlock detection and rollback because users do dumb things (open a txn and go for coffee). Latches are short-term mutexes protecting internal data structures; no one rescues you, so the engineer must avoid deadlocks.
- Why not mmap: (1) OS may write dirty pages at any time, breaking write-ahead-log ordering (mlock prevents eviction but not writeback); (2) threads stall on page faults with no way to do other work; (3) I/O errors arrive as signals anywhere in the code instead of at one read call; (4) OS page-table structures and their latches are slower than a purpose-built one. MongoDB started on mmap and abandoned it (bought WiredTiger); SQLite's mmap is off by default.
- Replacement policy goals: good choices, fast (if picking a victim costs more than an I/O, just do the I/O), tiny metadata.
  - LRU (1965): evict least recently accessed (linked list or timestamps).
  - CLOCK (1969): approximate LRU with one reference bit per frame and a sweeping hand that clears bits and evicts the first frame with bit 0; resumes where it left off; only runs when a frame is needed. Linux uses a multi-hand variant.
  - Both suffer sequential flooding: one full scan pollutes the cache and evicts hot pages (e.g. a hot key everyone reads). Scan-heavy OLAP may actually want MRU.
  - LFU (1971): access counts — O(log n) upkeep and no notion of time (a burst at night keeps a cold page forever).
  - LRU-K (early 90s; SQL Server, early Postgres): keep last K access times per page; evict the page with the largest gap between now and its K-th most recent access (infinity if fewer than K accesses; tie-break by oldest access). Ghost list remembers history of evicted pages so a returning hot page isn't relearned from scratch. Use a logical counter, not wall-clock time (clocks jump back).
  - MySQL approximate LRU-K: one list split into young and old regions; new pages enter at head of old region; only a second access promotes to young. One-off scans fall off the old tail.
  - ARC (IBM; ZFS, DB2; briefly Postgres, patent worries): recent and frequent lists plus ghost lists, with an adaptive parameter p that grows whichever list the workload rewards — self-tunes between recency and frequency as workloads shift. Roughly state of the art.
- Further tricks: per-query ring buffer for big sequential scans (Postgres) so scans don't flood the shared pool; priority hints (keep B+tree root pages; don't double-count a read-then-update within one transaction — SQL Server does this).
- Dirty pages: evicting one requires writing it (and its log record first) — slow. Background writer / page cleaner flushes dirty pages proactively so eviction finds clean victims; must be throttled so it doesn't steal I/O from queries (Postgres triggers on thresholds; enterprise systems tune aggressively).
- I/O scheduling: Linux I/O priority is per-process, so it can't tell a critical query's read from a background task's. DBMSs keep their own I/O queue: sort requests by page/disk location to turn random into sequential I/O, prioritise log writes and transactional work, and set the OS scheduler to the simplest mode.
- OS page cache double-buffers every page (kernel copy + user-space copy). Use O_DIRECT (direct I/O) to bypass it. Postgres is the famous holdout (recommends only ~25% RAM for shared_buffers); Postgres 18 adds async I/O as a step toward direct I/O (direct I/O needs async I/O first because you lose page-cache hits).
- fsyncgate (2018): write() only reaches a kernel buffer; fsync() forces it to disk. Linux used to mark pages clean after a failed fsync, so the common "retry fsync in a loop" pattern reported success while data was lost. Postgres, MySQL and WiredTiger were affected; now fixed, but the lesson is that a failed fsync means you cannot trust the data — crash and recover from the log.

**Decisions & trade-offs**
- Exact LRU vs CLOCK: precision vs metadata and update cost; constants matter (1ms vs 10ms), not just big-O.
- Recency vs frequency: LRU floods on scans; LFU forgets time; LRU-K/ARC combine them at more bookkeeping.
- Aggressive background flushing: faster evictions vs I/O stolen from foreground work.
- Direct I/O: no double caching and full control vs losing free OS caching (needs async I/O and a good own cache).

**Rules of thumb**
- Pin while using; unpin as soon as done; never hold pins across user think time.
- Isolate scans from the hot working set (ring buffer, separate pool, "old" region).
- Remember history of evicted items (ghost entries) for adaptive caches.
- Use logical clocks for ordering decisions.
- Sort pending I/O by location before issuing.
- Treat fsync failure as fatal, not retryable. Assume anything that can fail will, and hardware/OS may lie (battery-backed caches, old fsync semantics).

**Failure modes & gotchas**
- mmap-based storage engines: uncontrolled writeback, unpredictable stalls, scattered error handling.
- Sequential flooding evicting hot pages.
- LFU keeping a briefly-popular item forever.
- Counting multiple accesses from one transaction as popularity.
- Double buffering with the OS page cache wastes half your RAM.
- Running other memory-hungry processes alongside the DB causes thrashing.

**Relevance to Sam**
- Any in-process cache for supplier API responses or geocoding lookups (e.g. Viator/RateHawk product data) should be scan-resistant: an LRU will be wiped by one bulk sync job. Use separate caches for bulk/batch paths vs live user paths, or a two-region (probation/protected) design like MySQL's or ARC/LRU-K (libraries like Caffeine/TinyLFU follow this idea).
- 16GB laptop pipelines: budget memory up front (fixed chunk sizes, bounded queues) instead of letting the OS swap; process data in one pass while it is loaded (temporal control); sort keys before random lookups into big files to get sequential I/O.
- When writing files that must be durable (exports, booking records), write -> fsync -> rename, and treat fsync errors as hard failures.
- Memory-mapping huge files (e.g. via numpy memmap/pyarrow) is convenient for read-only analytics but unpredictable for read-write stores — know the trade-off.

## Andy Pavlo (CMU Database Group) + Joseph (SingleStore) — #05 Log-Structured Database Storage + SingleStore talk (https://www.youtube.com/watch?v=2_sTdS4h-bY)
**Three ways to lay out a table — tuple-oriented heap, index-organized, and log-structured (LSM) — trade read cost against write cost; LSMs turn every write into a sequential append and pay later in compaction.**

**Buffer pool optimizations (finishing lecture 4)**
- Multiple buffer pools: per database/table/page type (DB2 lets you go per table/index/blob). Gains: policies tuned to each access pattern and less latch contention. Route a page by object ID in the record ID, or hash(page_id) mod N (MySQL). Invariant: a page lives in exactly one pool, never two copies.
- Prefetching: the DBMS knows the query plan, so it reads ahead. The OS can only prefetch the next sequential pages; the DBMS can prefetch the non-contiguous leaf pages a B+tree range scan will touch (e.g. pages 3 and 5, not 4).
- Scan sharing / synchronized scans: a second query scanning the same table attaches to the running cursor, consumes the same pages, then wraps around for the part it missed. Done at storage level, unlike result caching. In DB2, SQL Server, Teradata, Postgres. Oracle only matches byte-identical SQL text (case/whitespace differences break it).

**Tuple-oriented heap recap and its costs**
- Read: index gives record ID -> page directory -> fetch page -> slot -> tuple (two lookups: index then heap).
- Insert: find a page with free space (directory tracks it), else allocate one.
- Update in place: if the new version doesn't fit the page even after in-page compaction, it becomes delete + insert elsewhere.
- Problems: fragmentation (wasted gap between slot array and data), useless I/O (fetch a 16KB page to change 8 bytes), batch updates touching many scattered pages, and incompatibility with append-only storage (HDFS, Google Colossus, S3-style object stores don't allow in-place overwrite).

**Index-organized storage**
- Store the tuples themselves in the leaf nodes of the index (B+tree, skip list, trie) keyed by primary key; leaf pages keep a sorted key->offset array plus tuple data. One traversal + binary search finds the row; no separate heap lookup. SQLite and MySQL InnoDB work this way by default; Oracle/SQL Server offer it per table.

**Log-structured storage (LSM tree, O'Neil et al. 1996, successor to log-structured file systems)**
- Only two ops: PUT(key, value) and DELETE(key) (a tombstone). No in-place overwrite on disk.
- Writes go to an in-memory MemTable (skip list/B+tree; in-place updates allowed there; a separate write-ahead log makes it durable). When it reaches ~hundreds of MB, scan it in key order and write an immutable SSTable (sorted string table); only the latest version of each key survives. SSTables are ordered newest -> oldest.
- Reads: check MemTable; then SSTables newest-first; first hit wins. To avoid probing every file, keep in-memory summaries: per-file key ranges, Bloom filters (point lookups; false positives possible), range filters. Values are whole serialized tuples, so a partial update is a read-modify-write unless it's a blind write.
- Deletes are cheap: append a tombstone without finding the old data; readers must respect it.
- Compaction: background k-way merge (like sort-merge) of SSTables, keeping only the newest entry per key and dropping overwritten/deleted data. Triggered by size, file count, overlap, etc.; no universally correct policy; competes with foreground queries for I/O.
  - Leveled compaction (RocksDB default): L0 files overlap (straight from MemTable); each lower level is larger and has non-overlapping key ranges, so a lookup touches at most one file per level. Cost: write amplification — merging into a level can rewrite most of it (10GB read + 10GB written).
  - Universal/tiered compaction: single tier, merge adjacent time-ordered runs into bigger ones; less write amplification, faster ingest; good when queries want recent data (time series), worse for arbitrary key lookups.
- RocksDB (Meta's fork of Google's LevelDB, by Jeff Dean et al.) is the common embedded engine; the first thing Meta did was rip out mmap for a real buffer manager. Many systems (CockroachDB etc.) started on RocksDB then wrote their own.
- Core trade-off: LSM = fast, sequential inserts/updates/deletes; slower reads plus compaction cost and write amplification (a row rewritten many times though updated once). Heap = row written once, but random in-place updates.

**SingleStore talk: OLTP vs OLAP**
- OLTP: many small concurrent reads/writes, millisecond SLAs, strict ACID; row-oriented B-trees/skip lists (Postgres, Oracle, SQL Server, CockroachDB, Spanner).
- OLAP: huge scans and aggregations over TB-PB, minutes to hours; column stores (Snowflake, ClickHouse, Redshift, BigQuery, Vertica). Column stores scan dramatically faster but updates are slow; no column store matches a row store on point queries.
- SingleStore = "column store with tricks": a row-store segment holds hot rows in front of the column store, column encodings made seekable, secondary hash indexes giving row offsets, and the row store manages locking — so it can finish both analytic (TPC-H/TPC-DS) and transactional (TPC-C) benchmarks. Also offers in-memory row-store tables (formerly MemSQL).

**Rules of thumb**
- Pick storage by workload: read-mostly with point lookups -> heap or index-organized; write/ingest-heavy, append-only storage, or high churn -> LSM; analytics scans -> columnar.
- Keep any two copies of the same page/object from diverging: one owner per item.
- Isolate long scans; let concurrent scans share I/O.
- With LSMs, budget for compaction I/O and disk headroom; tune leveled vs tiered by read vs write priority.

**Failure modes & gotchas**
- Write amplification and compaction storms stealing I/O; space amplification until compaction runs.
- Reading many L0 files when compaction falls behind.
- Bloom filter false positives cost extra reads; range queries need range filters.
- Hard-coded limits lurk (Oracle 1000 columns; Postgres 2^16-ish columns; record ID widths).

**Relevance to Sam**
- Ingesting millions of rows (GeoNames updates, supplier catalog syncs) is an LSM-shaped problem: append new records with a version/timestamp, compact (keep latest per key) as a separate batch step, rather than doing random in-place updates on a big file. This is exactly the pattern for Parquet/Delta-style "append + merge" pipelines.
- Tombstones: represent supplier deletions as explicit delete records so downstream merges propagate them instead of silently keeping stale products.
- Merge-on-read: when combining many partial outputs (chunks processed in parallel), a k-way merge of sorted files keeping the newest entry per key is memory-cheap and fits a 16GB laptop.
- Bloom filters/key-range summaries per chunk file let you skip files during dedup/matching lookups.
- Booking/payment state is OLTP (row store, ACID); analytics on bookings or location data is OLAP (columnar: DuckDB/Parquet). Don't force one engine to do both.

## Andy Pavlo (CMU Database Group) — #06 Column-Store Databases (https://www.youtube.com/watch?v=yWnToWrskXE)
**For analytics, store each attribute contiguously (columnar/PAX, as in Parquet/ORC) and compress with DBMS-aware schemes so queries read only needed columns and can run on compressed data.**

**Mental models / first principles**
- Workloads: OLTP = many small reads/writes on few rows (post a comment, add to cart); OLAP = complex read-heavy queries over lots of existing data (joins, aggregates, window functions); HTAP = trying to do both in one system (SingleStore, TiDB) — often neither best-in-class. Quad chart: write-heavy+simple = OLTP; read-heavy+complex = OLAP. Most queries, even in OLTP apps, are reads.
- The relational model is logical; nothing forces a tuple's attributes to be stored together. Physical layout can be chosen per workload while SQL stays the same. Switching an analytic workload to a column store took one telco's queries from days to minutes.
- I/O is the bottleneck; compression trades CPU for fewer bytes read, increasing useful tuples per I/O.

**Mechanisms**
- NSM / row store: all attributes of a tuple contiguous in a slotted page. Great for point lookups, inserts, updates, whole-tuple reads. Bad for analytics: an OLAP query filtering on hostname and aggregating on last_login still drags every other column (often >50% wasted bytes) through I/O and the buffer pool.
- DSM / pure column store: each attribute's values for all tuples stored together in fixed-length arrays; one header/null bitmap per column page instead of per tuple. Tuple reconstruction via fixed-length offsets (the i-th value in each column belongs to tuple i). Embedding explicit tuple IDs per value is a bad idea (extra storage + index); DATAllegro did this, Microsoft paid ~$200M and threw the code away.
- Variable-length values are made fixed-length by dictionary compression (pad-to-max is wasteful).
- DSM benefits: minimal I/O, tight branch-free loops over contiguous same-type data (CPU-friendly, SIMD), far better compression (homogeneous values). Costs: slow point queries and inserts/updates/deletes (one row = N column writes; stitching tuples back from N pages).
- PAX (Partition Attributes Across, Ailamaki 2001) — what nearly all "column stores" actually do, and what Parquet, ORC, and newer Nimble (Meta) and Vortex use: horizontally split rows into row groups (Parquet default ~1M rows; ORC by size ~100MB), and inside each row group store column chunks. Per-row-group metadata (encoding, dictionaries) plus a file footer written last (you don't know stats until done) holding offsets. Readers parse footer -> jump to wanted column chunks. OLAP "pages" are MBs (S3-object sized), not KBs.
- Gotcha: Parquet's footer is Thrift/protobuf-like and must be deserialized whole (10MB of metadata to read 10 bytes); newer formats use FlatBuffers for zero-copy partial access.
- Compression goals for a DBMS: fixed-length outputs (to keep offset addressing), delay decompression as late as possible (operate on compressed data), lossless only (lossy, e.g. downsampling 1-second temps to 10-minute buckets, only on explicit user request; approximate query processing is a separate thing).
- Granularity: block/page (general-purpose), tuple (rare — Terark/SQL Server), attribute (Postgres TOAST compresses big values), columnar (what column stores use).
- Naive block compression (gzip, zstd, LZ4, Snappy; Oracle's OZIP): opaque to the DBMS — must decompress to read. MySQL InnoDB compressed pages: stored at 1/2/4/8KB, plus a per-page mod log absorbing blind writes without decompressing; reads that need values decompress to 16KB and apply the mod log. Postgres has no page compression, only TOAST.
- Columnar encodings (composable/multiplicative):
  - RLE: (value, start offset, run length) triplets; needs sorted data — alternating values make it bigger than raw. Sorting the table by one column helps that column and may hurt others; choosing the best sort order is NP-hard.
  - Bit packing: store the bits actually needed (32-bit ints holding small values -> 8 bits: 256 -> 64 bits). Patching / Redshift "mostly8": outliers get a sentinel and live in a patch table.
  - Bitmap encoding: one bitmap per distinct value; superb for low cardinality with fast bitwise ops; awful for high cardinality (zip codes). Roaring bitmaps compress sparse bitmaps.
  - Delta encoding: store differences from previous value (or frame of reference from a global min); combine with RLE (timestamps +1 repeated): 320 -> 128 -> 96 bits in the example.
  - Dictionary encoding (most common; ORC dictionary-encodes everything): map values to small fixed-length integer codes with a two-way dictionary. Order-preserving dictionaries (sorted) let range/prefix predicates run on codes: evaluate LIKE 'And%' on the small dictionary, convert to code range BETWEEN 10 AND 20, then scan integers. DISTINCT queries may need only the dictionary. Inserts that fall between existing codes force re-encoding — acceptable because OLAP data is mostly immutable.

**Decisions & trade-offs**
- Row vs column vs PAX by workload; PAX gives column-store scan benefits while keeping a tuple's attributes nearby for reconstruction.
- Encoding choice depends on data: sorted/low-cardinality -> RLE/bitmap; small ints -> bit packing; monotone series -> delta (+RLE); strings -> dictionary.
- CPU vs storage/I/O is the core compression trade-off.

**Rules of thumb**
- Analytical queries should never SELECT *; read only needed columns.
- Sort data before writing columnar files to maximize RLE/dictionary gains and min/max skipping.
- Prefer integer comparisons on codes over string comparisons.
- Never store raw passwords in the DB; hash/encrypt in the application.

**Failure modes & gotchas**
- RLE on unsorted, alternating data increases size.
- Bitmap encoding on high-cardinality columns explodes.
- Non-order-preserving dictionaries break range predicates on codes.
- Patching sentinel collisions: a real value equal to the sentinel must also go in the patch table.
- Point lookups and per-row updates on column stores are slow; batch writes.

**Relevance to Sam**
- GeoNames (~6M rows) and supplier catalogs on a 16GB laptop: keep them in Parquet (PAX) and query with DuckDB/Polars, selecting only needed columns; this is the single biggest memory and speed win.
- Sort before writing Parquet (e.g. by country_code, then admin codes / name) so dictionary and RLE compress well and row-group min/max stats let readers skip row groups.
- Choose row-group sizes so one group fits comfortably in RAM when processing in chunks.
- Use categorical/dictionary types (pandas category, Polars Categorical, Arrow dictionary) for repeated strings like country codes, feature classes, supplier names; joins/filters on codes are much faster and smaller.
- Booking/payment state belongs in a row store (Postgres); analytics and matching runs belong in columnar files. Don't try to run both on one engine without a reason.

## Andy Pavlo (CMU Database Group) — #08 B+Trees: The Best Data Structure in the World (https://www.youtube.com/watch?v=u7ii_Lvm9rM)
**A B+tree is a page-sized, high-fanout, self-balancing index over a sorted linked list of leaf nodes; it supports point, range and prefix lookups in O(log n) with few I/Os, which is why nearly every CREATE INDEX gives you one.**

**Recap: dynamic hash tables (end of lecture 7)**
- Static hash tables must rebuild (usually doubling) when full — expensive at scale. Chained hashing can degrade to a linear scan.
- Extendible hashing: a directory indexed by the first `global depth` bits of the hash; buckets have local depth. On overflow, split only that bucket; double the (small) directory only when needed. Many directory slots can share a bucket.
- Linear hashing: a split pointer walks buckets in order; on any overflow, chain the overflowing bucket and split the bucket at the split pointer, rehashing it with key mod 2n (keys go either to the same bucket or the new one). Lookup: if h(key) mod n < split pointer, use mod 2n. Eventually every bucket is split; pointer wraps. Deletes can shrink, but implementations are lazy about it to avoid split/merge thrash.
- Hash tables are mainly for internal structures (page table, join hash tables), not general indexes: they can't do range scans or lookups on a prefix of a multi-column key.

**Mental models / first principles**
- "B-tree" family = balanced trees. Original B-tree (Bayer & McCreight at Boeing, ~1970; "B" likely = Boeing); "The Ubiquitous B-Tree" (Comer, 1979). Real systems implement a B+tree plus the B-link tree additions (Lehman & Yao, CMU 1981: sibling pointers for concurrency). Postgres's "nbtree" is a Lehman-Yao B+tree that relaxes balance. When a DBMS says B-tree it almost always means B+tree.
- A B+tree is scaffolding on top of a sorted linked list (the leaves): it lets you jump into the list instead of scanning it. Skip lists play the same role.
- One node ~ one page (4-16KB), so fan-out is in the hundreds; real trees are only ~4-5 levels deep even for billions of keys. It turns random I/O into few page reads and sequential leaf scans.

**Mechanisms**
- Properties: M-way (up to M children, M-1 keys per inner node); perfectly balanced (all leaves same depth); every node except root at least half full; O(log n) search/insert/delete.
- Inner nodes: alternating child page-IDs and separator keys (guideposts only). Leaves: keys + values (record IDs, whole tuples in index-organized tables like MySQL InnoDB/SQLite, or primary keys for MySQL secondary indexes), with prev/next sibling pointers. NULLs grouped first or last (Postgres: last).
- Physical node layout: header (level, slot count, prev/next, high key), keys stored as a separate sorted array and values as a parallel array, so binary search touches only keys (like a column store).
- B-tree vs B+tree: B-tree stores values in inner nodes too (less space, but lookups/scans bounce up and down = random I/O, and worse under concurrency). B+tree values live only in leaves; separator keys of deleted entries may remain in inner nodes harmlessly; existence is always confirmed at a leaf.
- Insert: descend to leaf; if room, insert in sorted order. If full, split: new leaf, move upper half of keys, push (copy) the middle key up as separator; recursively split parents; splitting the root grows height by one. Untouched subtrees can stay on disk — only page IDs change.
- Delete: remove from leaf; if still >= half full, done. Else try to borrow from an adjacent rich sibling (update parent separator), else merge with a sibling and delete the parent's separator, recursing; root collapse shrinks height. Only look at immediate siblings (more would mean more latches).
- Composite keys (a, b, c) with per-column sort order and NULLS FIRST/LAST: supports full-key lookups, prefix lookups (a, or a and b: descend then scan leaves until the prefix changes) and, in Oracle and newer Postgres, skip scans on suffix-only predicates (scan all leaves, parallelizable). Predicate order in SQL doesn't matter to a decent optimizer.
- Duplicate keys (non-unique indexes, MVCC versions): correct approach is to append the record ID to the key to make it unique (hidden from users; prefix lookups still work). Overflow chains of leaf pages hanging "down" from a leaf are a bad idea because later splits must drag the whole chain into memory.
- Clustered index / clustered table: heap physically sorted in index order (free with index-organized storage; Postgres CLUSTER sorts once and doesn't maintain it). If index order differs from heap order, naively following record IDs causes repeated random page reads. Optimization: scan the index first, collect record IDs, sort by page ID, then fetch each page once (Postgres bitmap heap scan; can intersect ID lists from multiple indexes).
- Design choices: node size depends on storage speed (fast memory -> small nodes; slow HDD -> up to ~1MB); inner nodes may be bigger than leaves. Strict half-full merging yields ~69-70% occupancy; delaying merges avoids split/merge churn (hence Postgres's "non-balanced" tree).
- Variable-length keys: store pointers to keys in the tuple (TimesTen did this; random memory access makes it slow — don't); variable-size nodes (needs a custom allocator; academic only); padding to max length (wasteful); standard choice: slotted-page-style key offset array within the node, with overflow for huge keys.
- In-node search: linear scan (fine; small, in memory), SIMD comparisons of 4+ fixed-width keys per instruction, binary search (common), interpolation search (only for dense, uniform keys; rare).
- Optimizations: pointer swizzling — when a child page is pinned in memory, store its raw memory pointer instead of a page ID to skip buffer-pool lookups (must never evict a parent while a child is swizzled, and invalidate on eviction). Write-optimized B-epsilon / fractal trees (TokuDB; SplinterDB, RelationalAI, others): each inner node has a mod log buffering inserts/deletes; lookups check logs on the way down (a buffered delete ends the search); full logs flush down a level. Gets LSM-like write batching inside a B-tree, but complex (e.g. leaf scans must consult buffers above), so rare.
- Further reading: Goetz Graefe's books on modern B-tree techniques.

**Decisions & trade-offs**
- Hash index: O(1) point lookups only. B+tree: point, range, prefix, ordered output, slightly more I/O for points.
- B+tree vs LSM: B+tree reads are predictable; LSM writes are cheaper; B-epsilon trees sit between.
- Eager vs lazy rebalancing: correctness is the same; laziness trades space for fewer reorganizations.

**Rules of thumb**
- Put the most selective / always-present column first in a composite index; an index on (a,b) serves queries on a, and a+b, not efficiently on b alone (unless skip scan).
- For range lookups through a secondary index into a heap, sort record IDs by page before fetching.
- Make non-unique keys unique by appending a tiebreaker ID rather than chaining duplicates.
- Keep index keys short and fixed-width where possible (better fan-out, SIMD-friendly).

**Failure modes & gotchas**
- Index order != table order -> random I/O storms on range scans.
- Duplicate-key overflow chains make splits pathological.
- Random-key inserts (e.g. random UUIDs) spread writes across all leaves, causing many splits and poor cache locality; monotonically increasing keys always hit the rightmost leaf (hot spot under concurrency) — root/rightmost pages deserve buffer-pool priority (see lecture 4).
- Concurrent splits/merges are the hard part (covered next lecture with latch crabbing).

**Relevance to Sam**
- Supplier/booking tables in Postgres: composite indexes like (supplier_id, external_product_id) give both exact matching and "all products for a supplier" prefix scans; idempotency keys for booking requests should have a UNIQUE B+tree index so duplicate submissions fail fast.
- Prefer time-ordered IDs (UUIDv7/ULID) over random UUIDv4 for large insert-heavy tables to keep B+tree inserts localized.
- In pipelines: matching millions of rows against a reference set is a sorted-merge or B+tree-style problem — sort both sides by key and walk them (sequential) rather than doing random lookups.
- Range lookups on coordinates (lat/lon bounding boxes) use B+trees poorly on two dimensions; use a composite key on a geohash/H3 cell for prefix-range scans, or a spatial index.

## Andy Pavlo (CMU Database Group) — #13 Query Execution Part 1 (https://www.youtube.com/watch?v=E-UUd6cB57w)
**A query plan is a DAG of operators split into pipelines; how data moves between operators (one tuple, all tuples, or batches; pulled from the top or pushed from the bottom) and how scans and expressions are executed decide whether the engine suits OLTP or OLAP.**

**Mental models / first principles**
- Pipeline = a chain of operators a tuple can flow through without stopping. Pipeline breaker = an operator that needs all its input before emitting anything (hash-join build side, sort, full aggregation). Goal from the 1970s onward: once you've paid to fetch a tuple, ride it as far up the pipeline as possible.
- Logical data flow (bottom-up arrows) is not the same as physical execution (most systems pull from the top).
- A processing model has two parts: control flow (who tells an operator to run) and data flow (how results move), plus what is passed (whole tuples = early materialization; minimal columns + record IDs = late materialization).

**Mechanisms**
- Iterator / Volcano model (Graefe): every operator implements Open/Next/Close; Next returns one tuple or EOF. Root's Next calls children recursively; a hash join drains its left child to build the table, then pulls from the right child to probe. Like Python generators/yield. Used by Postgres, MySQL, SQLite, classic Oracle/SQL Server/DB2. Pros: simple, easy to debug, streaming, LIMIT is trivial (stop calling Next). Cons: one virtual function call per tuple per operator — expensive at billions of tuples.
- Materialization model (MonetDB, 1990s; also HyriseV1, RavenDB): each operator emits its entire output at once. Needs operator fusion (push filters/limits/projections down into scans) to avoid passing a billion rows to a filter that keeps ten. Fine for OLTP/in-memory with small results; falls apart on large intermediates.
- Vectorized/batch model (MonetDB/X100 -> VectorWise -> its author co-founded Snowflake; now every modern OLAP engine): Next returns a batch (~1024-2048 tuples; DuckDB ~1024). Amortizes call overhead, gives tight branch-free loops over arrays, enables SIMD/GPU kernels. Design question: refill sparse batches after filtering or pass them up sparse.
- Plan direction: pull (top-down Next; easy, LIMIT-friendly, most common) vs push (bottom-up; a scheduler runs pipeline tasks in dependency order, each writing to an output buffer the scheduler hands to dependent tasks; separates control flow from data flow, enables better placement/scheduling and fusing within pipelines). Push + vectorized: HyPer, Umbra, CedarDB, DuckDB (switched from pull), ClickHouse, Firebolt, Snowflake. Old row engines (Oracle, SQL Server, DB2) bolt on separate columnar/vectorized engines rather than mixing models in one engine.
- Access methods (plan leaves): sequential scan (fallback; cursor over pages), index scan, multi-index scan.
- Making sequential scans fast: compression/encoding, prefetching, scan sharing, buffer pool bypass, parallelism, clustering/sorting, late materialization, materialized views (hard to maintain incrementally; Postgres needs manual REFRESH), result caching (rare), data skipping, code specialization.
- Data skipping: (1) approximate queries (sampling-based APPROX_COUNT etc., lossy — only when the user asks; never for account balances); (2) zone maps / small materialized aggregates (Moerkotte 1998): per-block min/max/count/null-count/distinct; skip a block if the predicate can't match (WHERE val > 600 with block max 400). Universal, cheap, always worth it; Parquet/ORC/Vortex all carry these stats.
- Index choice depends on selectivity: for age<30 AND dept='CS', use whichever index returns fewer rows; hash indexes can't serve range predicates. Choosing is the optimizer's job (next lecture).
- Multi-index scan (Postgres bitmap scan, MySQL index merge): probe several indexes, intersect record-ID sets for AND, union for OR, fetch remaining rows, apply residual predicates (e.g. country='US').
- Modifications reuse scan operators beneath INSERT/UPDATE/DELETE operators; INSERT accepts tuples from any child (INSERT ... SELECT, CREATE TABLE AS). UPSERT = scan + update-or-insert; TRUNCATE = drop/recreate.
- Halloween problem (IBM System R, 1976, found on a Friday afternoon): an UPDATE scanning an index on salary moves updated rows ahead of the cursor and updates them again (give everyone under $1100 a $100 raise -> some get raised repeatedly). Fix: track which logical tuples this statement already modified (side set or a marker in the tuple header; MVCC handles it later).
- Expression evaluation: WHERE/JOIN predicates are expression trees walked per tuple using an execution context (current tuple, query parameters, schema). Tree walking is slow (pointer chasing, branches). Faster: specialize to a direct function (val == 1), vectorize it over batches (auto-vectorization/SIMD), or JIT-compile. Demo: Postgres on 50M rows, scan with WHERE: ~4.0s; with LLVM JIT of the WHERE clause ~2.7s (including ~0.38s compile cost; Postgres's cost model decides when JIT is worth it); DuckDB single-threaded columnar vectorized push engine with no JIT: under 1s.
- HyPer/Umbra-style two-stage compilation: emit fast-to-generate machine code to start immediately, compile an optimized version with LLVM in the background, swap it in if ready.
- Expression rewrites: constant folding (UPPER('wu tang') once, not per row), common subexpression elimination (shared subtrees computed once).
- Prepared statements: PREPARE plan once, EXECUTE with parameters; saves optimization time but a cached plan can be bad for some parameter values. Postgres switches to a generic plan after ~5 executions.

**Decisions & trade-offs**
- OLTP: iterator or materialization model, index scans; small results.
- OLAP: vectorized model; push vs pull is an engineering-tolerance call.
- JIT: pays compile time up front; only worth it for long-running queries.
- Approximate answers: speed vs exactness — user decides, never the system.

**Rules of thumb**
- Push filters and projections as close to the scan as possible (operator fusion / predicate pushdown).
- Process in batches of ~1K rows, not one at a time, for CPU-bound work.
- Keep min/max stats per block/file so readers can skip.
- When an update can move rows within the structure you're iterating, track what you've already processed.
- Hoist constants and repeated subexpressions out of per-row loops.

**Failure modes & gotchas**
- Per-tuple function call overhead in iterator engines on large scans.
- Materializing huge intermediates.
- Halloween problem in updates that alter the indexed key.
- Prepared/cached plans that are wrong for skewed parameter values.
- Assuming a row-store with JIT can match a purpose-built columnar engine for analytics.

**Relevance to Sam**
- Python pipelines: per-row loops (iterrows, apply) are the iterator model at its worst. Use vectorized frames (Polars/DuckDB/NumPy/Arrow) or process chunks of thousands of rows; for streaming large files, a generator of batches gives bounded memory plus vectorized speed.
- DuckDB is a vectorized push engine — ideal on a 16GB laptop for joins/aggregations over Parquet; it beats Postgres-style engines by large factors for analytical scans.
- Write Parquet with row-group statistics and sort by common filter columns so zone maps skip data.
- Halloween-style bugs show up in pipelines that mutate the collection they iterate (e.g. re-queueing records for re-matching, or updating a table while paging through it with OFFSET/keyset on the column being changed). Snapshot the IDs to process first, or keep a processed-set.
- Upsert pattern for supplier syncs = scan/match then update-or-insert; make it idempotent with a unique key so retries don't double-apply.
- Constant folding/CSE apply to pipeline code too: precompile regexes, normalize lookup tables once, cache repeated transforms.

## Andy Pavlo (CMU Database Group) — #15 Database Query Optimizers Part 1 (https://www.youtube.com/watch?v=b53huOGcsZ8)
**The optimizer turns a bound logical plan into a cheap, provably equivalent physical plan by applying relational-algebra transformation rules and searching the (exponential) space of alternatives, using heuristics and/or a cost model.**

**Mental models / first principles**
- Pipeline: SQL -> parser (tokens -> abstract syntax tree) -> binder (resolve table/column/function names to catalog object IDs, e.g. Postgres OIDs; error on unknown names; hard with nested queries, self-joins, duplicate names) -> logical plan (what, not how) -> optimizer (+ cost model fed by catalog statistics and constants) -> physical plan -> execution engine.
- Worked example (employee JOIN department WHERE dept name = 'Toy'): literal translation with cross product + filters + temp files = ~2M I/Os; inner join with block nested loop = ~54K; sort-merge join = ~7K; pipelining (no temp files) = ~3K; push the filter below the join, use the index on department name, swap join sides, index nested-loop join = ~37 I/Os. Same answer, five orders of magnitude apart.
- Equivalence is non-negotiable: a fast wrong plan is worthless. Rules come from relational algebra equivalences (SQL's outer joins and NULL semantics complicate this).
- Join ordering is NP-hard; the number of n-way binary join trees grows explosively (Catalan-number style). Most queries have no joins; the tail is long (seen: 1500-table joins from generated dashboards; LLM-generated SQL makes weird queries common).
- Optimizers work one query at a time (multi-query optimization is basically research-only; DBT-style DAGs hint at it).
- Physical properties (especially sort order) must be tracked: a sort-merge join, an ORDER BY, or an index may already produce the required order.

**Mechanisms**
- Three components: transformations (logical->logical, logical->physical), search strategy, cost model (next lecture). The only exact cost estimate is running the plan; MongoDB literally races candidate plans and caches the winner — simple, breaks for complex queries.
- Equivalence rules: split conjunctive predicates into separate filters so each can move independently; reorder filters (cheap/selective first); selections commute with everything except outer joins; inner joins and cross products are commutative and associative.
- Expression rewrites: transitivity (x=3 AND y=x -> y=3), constant folding (1+1 -> 2), evaluate deterministic functions on constants once.
- Always-apply logical rules (no cost model needed): split conjunctions; predicate pushdown to just above the scan/join that supplies the columns; turn cross product + equality predicate into inner join (never do a cross join unless explicitly asked); projection pushdown to drop unneeded columns early; limit pushdown where legal. Rules can be many-to-one or one-to-many; some are one-way, others need history for backtracking. SQL Server has ~480-500 transformation rules; Microsoft is using LLMs to discover new valid rules and to prove LLM-rewritten queries equivalent via the optimizer.
- Prepared statements complicate planning: parameters unknown at plan time.
- Heuristic/rule-based optimizers (INGRES 1970s, Oracle until the early 90s, MongoDB, most new systems unless they borrow Postgres or Apache Calcite): easy to build and debug; rely on magic constants (e.g. how much faster sequential vs random I/O is) and fall apart beyond ~2-way joins. Postgres's rules are hard-coded if/else, which makes extension hard. Oracle once marketed "use FROM-clause order" as a "semantic optimizer".
- Cost-based search (IBM System R, 1970s): apply rules, enumerate alternatives, cost each, prune; may accept temporarily worse plans to reach better ones (like escaping a local minimum). Can be staged (single-table access paths, then joins, then nested queries) or holistic.
- Stopping conditions: wall-clock budget; cost below a threshold (e.g. 10% better than a known plan); no improvement for a while; search space exhausted. SQL Server counts transformations applied instead of time so the same query always yields the same plan regardless of machine load (must still reach at least one complete plan).
- Access path selection: logical scan -> sequential scan (always possible fallback), index scan, multi-index scan. "Sargable" (search-argument-able) predicates map directly onto an index.
- Demo: Postgres uses the index for ORDER BY val LIMIT 20 but not for ORDER BY val + 0 (it matches on the expression syntactically), even with NOT NULL and fresh statistics; SQL Server handles it. DuckDB chose sequential scans (even a self-join of two scans) over its index for these queries and still ran in ~10ms vs ~1s in Postgres (columnar + parallel, not a fair comparison).
- CAST semantics differ across systems (4.2 or 4.8 to INT: error vs truncate vs round) — rewrites must respect each system's semantics.
- Search direction: bottom-up / forward chaining (start from leaves and build up; System R, Postgres; HyPer/Umbra best-in-class, DuckDB copied HyPer's) vs top-down / backward chaining (start from the desired result and decompose; SQL Server's Cascades-style, best top-down). Can seed search with known plan shapes (e.g. star schema: fact table + dimensions).
- System R dynamic programming: choose best access path per table; then for each level of the join tree, enumerate join algorithms for each pair/next table, keep only the cheapest way to produce each intermediate result, and pick the cheapest full path. Restricted to left-deep trees (no bushy or right-deep) to shrink the space; Postgres still avoids bushy plans. No physical-property awareness originally — sort order was hacked in via cost penalties. Not guaranteed optimal.
- Query hints (MySQL, Oracle, SQL Server; Postgres only via pg_hint_plan extension; Postgres philosophy: fix the optimizer instead) override join order/algorithms/indexes. Double-edged: they freeze a plan based on today's data, and break silently when data or staff change.

**Decisions & trade-offs**
- Heuristics: fast and predictable, poor on multi-join queries. Cost-based: better plans, needs statistics and a cost model, more engineering.
- Search-space pruning (left-deep only, no cross products) trades optimality for tractable planning time.
- Optimization time vs execution time: spend ~1ms on trivial queries, more on complex ones; budget with deterministic counters if reproducibility matters.
- Hints: immediate fix vs long-term brittleness.

**Rules of thumb**
- Filter early, project early, join on keys, never cross join.
- Write predicates in sargable form: compare the bare indexed column to a constant (no arithmetic, functions or casts on the column, e.g. not val+0, LOWER(email), DATE(ts)) or create an expression index.
- Check plans with EXPLAIN (and EXPLAIN ANALYZE/BUFFERS in Postgres) instead of assuming.
- Always verify that a rewrite preserves semantics (NULLs, outer joins, casts, overflow).

**Failure modes & gotchas**
- Optimizers miss "obvious" rewrites (val+0) and pick scans.
- Plans for prepared statements can be wrong for some parameters.
- Huge machine-generated joins blow up planning.
- Hints rot as data evolves.
- Wall-clock-limited planners give nondeterministic plans under load.

**Relevance to Sam**
- Supplier/booking queries in Postgres: keep WHERE clauses sargable (e.g. created_at >= X AND created_at < Y instead of DATE(created_at) = X; store normalized lowercase emails or add an expression index) so indexes are actually used; inspect with EXPLAIN ANALYZE.
- In pandas/Polars/DuckDB pipelines, apply the same always-on rules manually or via lazy APIs: filter and select columns before joins (Polars lazy and DuckDB do pushdown automatically; eager pandas does not).
- Join order matters on a 16GB machine: join the small, filtered side into the big side (build hash tables on the smaller input); avoid accidental cross products from non-unique join keys (check key uniqueness before merging GeoNames with supplier location tables).
- When asking an LLM to write SQL for big tables, verify plans: generated SQL often has odd shapes that defeat simple optimizers.

