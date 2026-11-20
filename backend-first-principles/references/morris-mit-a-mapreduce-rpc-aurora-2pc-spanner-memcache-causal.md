# mit-1 — cross-cutting principles

All from Robert Morris, MIT 6.824 (2020). L1 = Intro/MapReduce, L2 = RPC & Threads, L10 = Aurora, L12 = Distributed Transactions, L13 = Spanner, L16 = Memcached at Facebook, L17 = COPS.

1. **Don't distribute unless one machine can't do it.** Distribution adds concurrency bugs, partial failure and consistency problems. Scaling one tier just moves the bottleneck to the next (web servers, then DB, then network). [L1, L16]
2. **At scale, failure is constant, not rare.** 1000 machines at one failure/year each gives about 3 failures/day. Failures are also correlated (same rack, same hardware batch), so spread replicas and restore redundancy fast, e.g. Aurora's 100-way parallel re-replication of small segments. [L1, L10]
3. **Write the consistency spec before replicating or caching:** what does a read return after a write? Strong (linearizable/external) consistency is easiest to program against and costs communication. Eventual consistency is cheap and full of anomalies. Product needs set the bar: bounded staleness plus read-your-writes was enough for Facebook. [L1, L13, L16, L17]
4. **Durability discipline: log before you promise.** Make the intent or log record durable before replying "committed", voting YES, or sending COMMIT. Keep undo info (old values) if partial work can reach storage, and redo info for committed work. Recovery truncates at the first log gap. [L10, L12]
5. **Ship small deltas, not big state.** Aurora's 35x came from sending tens-of-bytes log records instead of 8 KB pages, because the network was the bottleneck. MapReduce moved computation to the data. Find the scarce resource (usually network or I/O) and spend cheaper ones (CPU, redundant work) to save it. [L10, L1]
6. **Quorums: R + W > N, pick the highest version (never vote), proceed after the fastest W/R.** Slow or dead replicas are tolerated implicitly, and tail latency stops being set by your slowest node. [L10]
7. **Atomic multi-party commit = 2PC, which is correct but slow and *blocking*.** Once a participant has voted YES it can neither abort nor commit on a timeout. Use it only inside one administrative domain, never across organizations. Make the coordinator and participants replicated (Paxos/Raft) to remove the blocking (Spanner). [L12, L13]
8. **Before you promise, a timeout means abort. After you promise, you must wait or reconcile.** This applies to any external call whose outcome is unknown after a timeout: query status, don't blindly retry or assume failure. [L12]
9. **Every operation gets a unique ID, and every handler must be idempotent.** Duplicate COMMITs, re-sent deletes and retried RPCs must be harmless. Pure, deterministic tasks (MapReduce) can simply be re-run. [L12, L16, L1]
10. **Serializability via two-phase locking: hold locks until commit/abort.** Releasing early exposes intermediate or phantom values. Pessimistic locking suits high conflict; optimistic concurrency control suits low conflict. Watch lock ordering to avoid deadlock. [L12, L2]
11. **Give reads a cheaper path.** Most workloads are read-dominated. Use multi-version storage plus one snapshot timestamp per logical read for consistent, lock-free reads, with replicas waiting until caught up to that timestamp ("safe time"). Read replicas and caches lag, so route must-be-fresh reads to the source of truth. [L13, L10, L16]
12. **Clocks lie.** Never order events by raw wall clocks across machines. Use Lamport clocks (max(seen+1, now) plus node-ID tiebreak), sequence numbers, or bounded-uncertainty time with commit-wait (TrueTime). [L13, L17]
13. **Cache = load shield.** Look-aside cache: on write, update the DB *then delete* the key; never set from writers (racing sets cause permanent staleness). Use leases/single-flight against thundering herds and stale sets, warm-up for cold caches, and a fallback pool when a cache node dies. Once hit rate is high, the cache is load-bearing. [L16]
14. **Partition vs replicate is the core resource decision.** Partitioning is RAM- and storage-efficient but doesn't help a hot key, and has N² fan-out. Replicating fixes hot keys and fan-out but stores less distinct data. Most real systems mix both. [L16, L10]
15. **Order writes so readers never see a reference before its referent.** Write the object, then the pointer/list. Read the pointer, then the object. Causal consistency automates this; for data plus permission invariants you need a snapshot or transaction. Last-writer-wins silently loses concurrent updates. [L17]
16. **Concurrency in code: bound it, give state a single owner, plan termination, run a race detector.** Prefer one owner plus message passing over shared memory with locks. Use fixed worker pools, not a thread per item. A "check-then-act" sequence must be atomic. Racy code passes tests and then fails in production. [L2]
17. **Batch pipelines: chunk, pure map per chunk, partition by key to local disk, reduce per key.** This is the MapReduce shape that survives crashes (re-run a chunk) and limited RAM (spill to disk). It is awkward for multi-stage pipelines, which modern dataflow systems address. [L1]

## Robert Morris (MIT 6.824) — Lecture 1: Introduction (https://www.youtube.com/watch?v=cQP8WApzIQQ)
**Distributing a system buys parallelism and fault tolerance at the price of concurrency, partial failure and consistency headaches, so only distribute when one machine genuinely cannot do the job.**

- **Mental models / first principles**
  - A distributed system is cooperating computers talking over a network to do one coherent task. Four reasons to build one: performance via parallelism (more CPUs, RAM, disk arms), fault tolerance (a second machine takes over), problems that are physically spread out (bank A in New York, bank B in London), and security isolation (untrusted code talks over a narrow protocol).
  - First rule: if a single computer can solve it, do that. Distribution is never simpler.
  - Three sources of difficulty: concurrency (timing-dependent interactions), partial failure (some machines or links die while others carry on; a single box is mostly "works or doesn't"), and the performance gap (1000 machines rarely give 1000x without careful design).
  - The goal is abstractions that hide distribution: an interface that behaves like a local file system or key/value table but runs on many fault-tolerant machines. This is rarely fully achieved.
  - The three infrastructure categories are storage, computation and communication. Storage gets the most attention because it is a clean, well-understood abstraction.

- **Mechanisms**
  - **Scalability.** The ideal is linear: 2x machines gives 2x throughput, because buying hardware is cheaper than paying engineers to optimise code. Typical web-app path: add stateless web servers and split users between them. This works until the shared database becomes the bottleneck. More web servers then stop helping, and you must redesign storage (sharding), which is real design work. Scaling mostly moves the bottleneck somewhere else.
  - **Failure at scale.** If each machine fails about once a year, 1000 machines give roughly 3 failures a day. At scale, rare events become constant, so fault handling has to be part of the design, not an afterthought.
  - **Availability vs recoverability.** Available means the system keeps serving correctly under a defined set of failures (e.g. one of two replicas down). Recoverable is weaker: it stops serving, but once repaired it carries on correctly with nothing lost. That requires persisting state. Good available systems are also recoverable when failures exceed what they tolerate.
  - **Tools.** (1) Non-volatile storage (checkpoints/logs on disk or flash). It is slow to update, so much of high-performance design is about writing to it less often. (2) Replication. Its standing danger is replicas drifting out of sync.
  - **Consistency example.** Two replicas both hold k=20. A client sends put(k,21) to replica 1, then crashes before sending to replica 2. Later gets may return 21 or 20, and a fail-over could serve a week-old value. You need a written spec for what get returns. *Strong consistency* means a get sees the most recently completed put, which needs a lot of communication (read or write every copy). *Weak consistency* allows stale reads but is far cheaper.
  - Replicas should fail independently (not on the same rack or power cable, ideally different cities). That puts them 10–30 ms apart, which makes strong consistency expensive, since one put can take tens of milliseconds. This is why weak schemes are popular.
  - **MapReduce (Google, 2004).** Problem: multi-terabyte jobs (web index, link analysis) took weeks on one machine, and hand-writing distributed code for each job wasted engineers. Solution: the programmer writes only a pure `map(k,v)` that emits intermediate key/value pairs, and a `reduce(k, values[])`. The framework handles distribution, data movement and failure. Word count: map emits (word,"1"); reduce emits len(values). A *job* is made of many map and reduce *tasks*.
  - Execution: a master assigns map tasks to workers. Map output goes to the worker's **local disk**, partitioned by key. In the **shuffle**, each reduce worker fetches its keys from every map worker, which turns row-wise storage into column-wise. Reduce output is written to GFS, which splits files into 64 MB chunks across servers and keeps 2–3 replicas.
  - Network was the 2004 bottleneck. About 2000 machines shared a single root switch, giving each machine roughly 50 Mbit/s. Tricks to avoid the network: run GFS and MapReduce on the same machines, schedule each map on the machine that already holds its input chunk (data locality), and write map output locally. The shuffle can't be avoided: a sort moves its full input size (e.g. 10 TB) across the network, and replicating the output to GFS costs another network write.
  - Modern datacenters have many root switches and much more bandwidth, so later MapReduce stopped bothering with locality. (note: Google has since replaced MapReduce with successors such as Flume/Dataflow-style pipeline systems.)

- **Decisions & trade-offs**
  - Scaling hardware vs engineer time: horizontal scaling is a money-for-time trade, but it hits a shared bottleneck (usually the DB).
  - Strong vs weak consistency: correctness and simplicity for the application vs latency and communication cost, especially across geographic distance.
  - MapReduce's restriction is that maps must be independent, pure functions. It is great for "map, shuffle by key, reduce" problems and awkward for long multi-stage pipelines, which have to be chained as separate jobs (e.g. iterative PageRank). Later systems let you declare a whole pipeline and optimise it. Batch (finite input) vs streaming is another axis.

- **Rules of thumb**
  - Try everything on one machine first.
  - Find the bottleneck; expect adding capacity in one tier to shift it to the next.
  - Assume something is always broken at scale; design to mask failures from application code.
  - Write down semantics (what does a read return?) before you replicate or cache.
  - Move computation to the data, not data to the computation, when bandwidth is scarce.
  - Keep intermediate results on local disk, and make tasks pure and deterministic so they can simply be re-run.

- **Failure modes & gotchas**
  - A client that crashes mid-way through updating replicas leaves them divergent.
  - Fail-over can expose stale data long after the write.
  - Correlated failures (shared rack or power) defeat replication.
  - Distributed bugs are timing-dependent and hard to debug, so start early and test heavily.

- **Relevance to Sam's work**
  - On a 16GB laptop, the MapReduce model (chunked input, pure per-chunk map, partition by key to disk, then reduce per key) is exactly how to process GeoNames/OSM-scale data. Spill intermediate partitions to local disk instead of holding them in RAM, and make each chunk task re-runnable so a crash only costs that chunk.
  - Pure, deterministic tasks make retries safe, which is the same idea as idempotency for API calls.
  - The lesson that "scaling moves the bottleneck" applies to supplier integrations too: adding concurrency to API calls quickly hits provider rate limits or the DB.
  - The partial-write consistency example maps directly onto writing a booking to two systems (e.g. your DB plus a supplier). A crash between the two writes leaves them disagreeing unless you design for it.

## Robert Morris (MIT 6.824) — Lecture 2: RPC and Threads (https://www.youtube.com/watch?v=gA4YXUJX7t8)
**Threads let you write each concurrent activity as plain sequential code, but shared state then needs a locking strategy, coordination primitives and a bound on concurrency, and races hide in code that "always works".**

- **Mental models / first principles**
  - The course uses Go because it has good thread/lock support, a convenient RPC package, type and memory safety, garbage collection, and a small language. GC matters especially with threads: without it you have to hand-roll reference counting to know when the last thread has finished with a shared object.
  - A thread (goroutine) is its own program counter, registers and stack inside a shared address space. Go multiplexes many goroutines onto fewer OS threads, so scheduling happens at two levels.
  - Three reasons to use threads:
    1. **I/O concurrency.** Have many outstanding RPCs or network requests at once, one thread per request, each blocking independently.
    2. **Multi-core parallelism.** CPU-bound work spread over cores, up to the number of physical cores.
    3. **Convenience / background work.** E.g. a goroutine that loops "sleep 1s, ping workers" so a master can detect dead workers.
  - **Alternative: event-driven (async) programming.** One thread runs a loop that waits for any event (packet, timer) and looks up per-client state in a table. Pros: low per-activity overhead. Thread stacks are KB-sized, so a million threads means real memory and scheduler cost, while an event loop holds only small state per client. Cons: you must chop logic into callback pieces, which is harder to write, and a single loop gives no CPU parallelism. A hybrid is one event loop per core.
  - Processes are OS-isolated address spaces with no shared memory. Threads inside one process share memory.

- **Mechanisms**
  - **Race.** `n = n + 1` compiles to load, add, store. Two threads can both load 0 and both store 1, losing an update. Aligned 32-bit stores are typically atomic; byte stores and read-modify-write increments generally aren't, unless you use explicit atomic instructions.
  - **Mutex.** Lock/Unlock makes a multi-step sequence atomic with respect to other holders of the *same* lock. The language has no idea which data a lock protects; that mapping exists only in the programmer's head and has to be a deliberate, documented strategy.
  - **Lock placement.** Hiding the lock inside a data structure's methods is tidy. The costs are overhead when the data isn't actually shared, and deadlock risk when two lock-holding structures call each other. Fixing that often means lifting the locks up into the caller.
  - **Coordination primitives.**
    - Channels: send data between threads, and the receiver waits for it.
    - Condition variables: "kick" a thread that might be waiting.
    - WaitGroup: Add/Done/Wait for a known number of goroutines.
  - **Deadlock.** T1 holds lock A and wants B; T2 holds B and wants A. The program then hangs without crashing. "Program just stops doing anything" is the cue to check for deadlock.
  - **Web crawler case study.** Requirements: avoid cycles (remember URLs already fetched or started), fetch in parallel (pages are slow, so raise concurrency until pages/sec stops rising, i.e. the network is saturated), and detect termination (often the hardest part).
    - *Serial version:* DFS with a shared `fetched` map. Go maps are reference types, so recursion shares one map.
    - *Naive `go crawl(...)`:* the function returns immediately and main exits, so only one page is printed. You must wait for children.
    - *Shared-state version:* the fetched map is guarded by a mutex so check-then-set (lines 43–44) is atomic. Without the lock, two threads could both see "not fetched" and both fetch. Each level uses a WaitGroup: Add(1) before spawning, Done in the goroutine, Wait at the end. Use `defer wg.Done()` so Done runs even if the goroutine exits abnormally.
    - *Loop-variable capture gotcha:* a closure that reads the loop variable `u` sees it change on later iterations. Pass `u` as an argument to get a private copy. (note: Go 1.22+ made loop variables per-iteration, which removes this specific bug for new code, but the general closure-captures-variable lesson stands.) Variables captured by closures escape to the heap, and GC frees them later.
    - *Channel version:* a single master owns a private map, so there is no shared memory and no locks. Each worker fetches one page and sends exactly one message (its URL list) back on a channel. The master counts outstanding workers and stops when the count reaches 0, which is a clean way to detect termination. It has to seed the channel first or the master blocks forever. Channels are internally locked, so concurrent send and receive is safe. Use `select` for non-blocking receives.
  - **Race detector (`go run -race`).** It keeps shadow memory recording which thread last read or wrote each location, plus lock/sync events. It flags a write followed by a read with no synchronisation in between, even if the bad interleaving never actually happened. Limitation: it only sees code paths that actually ran, so it needs good test coverage.

- **Decisions & trade-offs**
  - Shared memory plus locks vs message passing over channels. Not sharing at all (a single owner of the state, communicating by messages) is usually simpler and avoids whole classes of bugs.
  - Threads vs event loop: programmer time vs memory and CPU overhead at very high client counts.
  - Thread-per-task vs a fixed worker pool: unbounded spawning can explode (a real web crawl could create billions of goroutines). A pool of N workers pulling from a queue bounds memory.

- **Rules of thumb**
  - Decide, for each piece of shared data, which lock protects it. Hold the lock across the whole read-check-write.
  - Prefer not sharing. Give state a single owner and send messages to it.
  - Always bound concurrency. Tune it up until throughput plateaus, then stop.
  - Plan termination and waiting explicitly (WaitGroup, outstanding counter). Put Done in a `defer`.
  - Run the race detector in testing. Absence of observed failures proves nothing.
  - Background loops (heartbeats, pollers) are cheap, but make sure they exit, or they pile up.

- **Failure modes & gotchas**
  - Racy code passes tests repeatedly and then fails at the customer, because the window is a few instructions wide.
  - Forgetting to wait for goroutines means work silently doesn't happen.
  - Closures capturing a mutating variable.
  - Lock ordering cycles cause deadlock.
  - Unbounded thread creation exhausts memory.
  - A goroutine dying before Done leaves Wait hanging forever.

- **Relevance to Sam's work**
  - Calling supplier APIs (Viator, RateHawk) in parallel is exactly the crawler problem: I/O concurrency, a "seen" set (dedupe), a concurrency bound (worker pool / semaphore tuned to the rate limit), and clean termination. In Python asyncio/ThreadPoolExecutor, use a bounded pool and a single owner (or a lock) for shared dedupe sets and caches.
  - On a 16GB laptop the per-thread memory point is directly relevant. Spawning one task per row of millions of rows will blow up RAM, so stream rows into a fixed-size worker pool.
  - Check-then-act races, like "is this booking already created? if not, create it", are the in-process version of the idempotency problem. Make the check and the insert atomic (lock, DB unique constraint, or upsert).
  - A master/worker split where workers return results over a queue and only the master mutates state is a clean pattern for chunked GeoNames/OSM processing with multiprocessing.

## Robert Morris (MIT 6.824) — Lecture 10: Cloud Replicated DB, Aurora (https://www.youtube.com/watch?v=jJSh54J1s5o)
**Aurora gets about 35x more throughput than mirrored MySQL by shipping only small redo-log records (not 8–16 KB pages) to six replicas across three availability zones, and by waiting for any 4 of 6 acks (a quorum) instead of all of them.**

- **Mental models / first principles**
  - **History behind the design.**
    - EC2 with locally attached disk: fine for stateless web servers, since you just spin up another. Bad for databases, because the data dies with the host. S3 snapshots lose everything written since the last snapshot.
    - EBS: a network "disk" backed by two servers using chain replication, which survives host death. But only one instance can mount a volume, it sends a lot of data over the network, and both replicas sit in the same availability zone (AZ), so a whole-datacenter loss loses data.
    - RDS (paper fig. 2): every page and log write is mirrored to a second AZ, giving 4 copies. Fault tolerant, but slow, because large page writes cross AZ links and you wait for all copies.
  - **How a transactional DB works (essential background).**
    - A transaction groups operations so they appear atomic to others and to crashes. Data items are locked until commit.
    - The on-disk B-tree data pages and the write-ahead log (WAL) are separate. Updates first land in a page cache.
    - Log records (txn id, item, old value, new value) plus a commit record must reach disk *before* the DB replies "committed". Dirty pages are flushed lazily, which batches many updates per page.
    - Recovery uses *redo* (re-apply committed new values not yet in the pages) and *undo* (use old values to roll back uncommitted changes that already reached disk). Storing both old and new values is what allows pages from long uncommitted transactions to be written early.
    - The key size asymmetry: a transfer changing two integers produces tens-of-bytes log records but 8 KB+ page writes.
  - **Aurora's two tricks:**
    1. Send only log records to storage. Storage is application-specific and understands MySQL redo records, so it is no longer a generic block device.
    2. Quorum writes, so slow or dead replicas are ignored.

- **Mechanisms**
  - **Fault-tolerance goals.** Write with one AZ dead. Read with one AZ dead plus one more server down (AZ outages can last days, during which another failure is likely). Ride out transiently slow servers. Re-replicate fast, because failures are correlated (same hardware batch).
  - **Classic quorums (late-70s idea).** N replicas, write to W, read from R, with R + W > N so every read set overlaps every write set in at least one server.
    - You can't pick the right answer by majority vote, since only one replica in the read set may be current. Use **version numbers** and take the highest.
    - Send to all N and proceed once the fastest W (or R) answer. Slow or dead servers are tolerated implicitly, with no need to decide who is up.
    - If you can't reach a quorum, keep retrying.
    - Tuning: R=1, W=N gives fast reads; W=1, R=N gives fast writes. Both extremes lose fault tolerance on the other operation.
    - Aurora uses N=6 (2 per AZ, 3 AZs), W=4, R=3. With 3 servers lost you can still read and rebuild replicas, but you can't write until replication is restored.
  - **Writes are log appends.** A transaction can be acknowledged only when its log records, and all earlier ones, are on a write quorum. Recovery can't restore a later transaction without the earlier ones.
  - **Storage servers** keep old page versions plus a per-page chain of pending log records. They apply the records lazily when someone reads the page. Six CPUs do redundant redo work, a trade of cheap CPU for scarce network.
  - **Reads normally aren't quorum reads.** The DB server tracks, per storage server, the highest *contiguous* log number it has acked, and reads a page from one server that is up to date.
  - **Quorum reads are used only in DB-server crash recovery.**
    1. A new instance finds the first log number missing everywhere (e.g. 103).
    2. It tells every server to discard entries from that point on. These can't belong to committed transactions, because commit required a write quorum.
    3. It then scans for transactions that have update records but no commit, and writes undo records for them.
  - **Sharding.** Data is split into 10 GB segments, each with its own 6-replica *protection group* (PG). Each log record goes only to the PGs holding the pages it modifies.
  - **Fast re-replication.** A storage server holds many 10 GB segments from about 100 different databases, each with a disjoint set of peers. When it dies, ~100 different servers each copy one segment from a surviving replica, all in parallel. That takes ~10 s instead of the ~1000–10,000 s it would take to push 10 TB through one 10 Gb/s link.
  - **Read replicas (up to 15).** They read pages directly from storage and cache them, and receive the main DB's log stream to update their caches. They lag by milliseconds. They must not apply uncommitted transactions, and must not see a B-tree in the middle of a rebalance. *Mini-transactions* and the VDL mark sequences of log records that storage exposes all-or-nothing. All writes go through one writer, because sequential log numbering needs a single sequencer.

- **Decisions & trade-offs**
  - General-purpose vs co-designed storage. Generality enables reuse; blurring the DB/storage boundary bought 35x. The paper doesn't separate how much came from quorums vs log-only shipping.
  - Chain replication or wait-for-all is simple but stalls on the slowest replica. Quorums tolerate stragglers at the cost of version tracking and more replicas.
  - Single writer plus many read replicas: easy log ordering, with read scale via eventually-fresh replicas (staleness is acceptable for most web reads, roughly 100:1 read:write).
  - Placing many small segments with disjoint peer sets turns recovery into a parallel job.

- **Rules of thumb**
  - Find the real bottleneck. Here it was network bytes, not CPU or disk. Ship deltas or intents (logs), not full state (pages).
  - Never acknowledge a commit until its log is durable on a quorum, and all prior entries are too.
  - Use version numbers, not voting, to pick the latest value.
  - Plan for whole-datacenter loss and for correlated failures. Restore redundancy quickly.
  - Keep both undo and redo information if partial work can reach durable storage.

- **Failure modes & gotchas**
  - Waiting on all replicas means tail latency equals your slowest server.
  - Readers seeing half-applied multi-step structural changes (B-tree splits).
  - Read replicas serving uncommitted data.
  - Gaps in the log after a crash: truncate at the first gap.

- **Relevance to Sam's work**
  - **Booking flows:** the WAL discipline (record intent durably, commit only once durable, then apply) is the pattern for an in-house booking ledger. Write a "booking intent" row before calling the supplier, mark it committed when confirmed, and have a recovery job that rolls forward or compensates pending intents after a crash.
  - **Data pipelines on 16GB:** "ship the log, not the page". Process and store diffs or changesets of GeoNames/OSM updates instead of rewriting full datasets, and apply them lazily.
  - Read replicas and caches lag, so decide which reads (e.g. availability or price at checkout) must hit the source of truth.
  - Splitting work into many small independent segments (e.g. 10 GB-ish partitions) makes both parallel processing and recovery from a failure cheap.

## Robert Morris (MIT 6.824) — Lecture 12: Distributed Transactions (https://www.youtube.com/watch?v=aDp99WDIM_4)
**Sharded data needs two mechanisms for correct multi-record transactions: concurrency control (two-phase locking) for isolation, and atomic commit (two-phase commit) for all-or-nothing behaviour despite partial failure. 2PC is correct but slow and blocking.**

- **Mental models / first principles**
  - Data gets sharded for load and space (half the bank accounts on server 1, half on server 2), but some operations, like a transfer, span shards. The goal is to hide this from the programmer through the transaction abstraction (begin ... end).
  - **ACID.**
    - Atomic: all writes happen or none, despite failures.
    - Consistent: app invariants (not the focus here).
    - Isolated: serializable.
    - Durable: once committed, it survives crashes, which means it is on non-volatile storage.
  - **Serializability.** A concurrent execution is correct if its results (DB state *and* outputs) match *some* one-at-a-time order of the same transactions.
    - Example: x=y=10, T1 moves $1 from y to x, T2 prints x,y. The only legal outputs are (11,9) when T1 runs first, and (10,10) when T2 runs first; the final state is 11/9 either way. Interleavings that print 10,9 or 11,10 are illegal.
    - Multiple results can be legal. Programmers can write transactions as if alone. Disjoint transactions can run truly in parallel, e.g. on different shards.
  - **Aborts** must be supported (missing record, overdraft, deadlock victim), which means undoing partial modifications.

- **Mechanisms**
  - **Concurrency control families.**
    - Pessimistic (locking): wait on conflict. Best when conflicts are frequent.
    - Optimistic (OCC): run without locks into a temporary area, validate at commit, abort and retry on conflict. Best when conflicts are rare, since it avoids lock overhead.
  - **Two-phase locking (2PL).** (1) Acquire a record's lock before reading or writing it. (2) Hold *all* locks until commit or abort, never releasing early.
    - Why hold until the end: releasing early lets another transaction slip into the middle, producing non-serializable results. Releasing a write lock early also lets others read a value that disappears if you abort (a "phantom" 11 that never really existed).
    - Locks are usually per-record (per row).
  - **Deadlock under 2PL.** T1 locks x then y while T2 locks y then x. The DB detects this via a waits-for cycle or a timeout, aborts one transaction and undoes it.
  - **Two-phase commit (2PC).** A transaction coordinator (TC) runs the transaction and sends gets/puts to participants (each holds a shard and a lock table). Every message carries a unique transaction ID chosen by the TC.
    1. At the end, the TC sends PREPARE to all participants.
    2. Each participant votes YES (it can commit) or NO (it aborted, hit a deadlock, crashed and forgot, or has a missing record).
    3. If all vote YES, the TC sends COMMIT; otherwise ABORT. Participants apply or undo, release locks, and ACK.
    4. The TC replies to the client.
  - **Crash rules (the important part).**
    - *Participant crashes before voting YES:* it may forget the transaction. When later asked to PREPARE, it says NO.
    - *Participant must force a log record before voting YES:* new values plus the locks held, written to its on-disk log. After a crash it restores the prepared state, including the locks, and waits for the outcome.
    - *TC must log the decision (txn id + commit/abort) before sending any COMMIT.* On restart it re-sends the outcome. Participants must therefore handle duplicate COMMITs. A participant that already finished and forgot the transaction simply ACKs again.
    - A TC that crashes before deciding can just abort. If asked about a transaction it doesn't know, it says "abort".
  - **Timeouts.**
    - TC waiting for votes: it may abort unilaterally, since nothing can have committed yet.
    - Participant waiting for PREPARE: it may abort unilaterally.
    - **Participant that has voted YES and is waiting for the outcome must BLOCK**, holding its locks indefinitely. It can neither abort (another participant may already have committed) nor commit (another may have voted NO). This blocking window is 2PC's defining weakness.
  - **Forgetting state.** The TC can drop a transaction after all ACKs arrive. Participants can drop it after applying and ACKing.
  - **Correctness rests on one entity (the TC) making the decision.** Participants don't negotiate with each other.
  - **Cost.** Multiple message rounds, plus synchronous log forces at each participant (before YES) and at the TC (before COMMIT), all while locks are held. At 10 ms per mechanical-disk append that is about 100 txn/s without cleverness.
  - **2PC vs Raft.** They look alike (leader plus acks) but solve opposite problems.
    - Raft: every replica does the *same* thing, only a majority is needed, so it gives high availability.
    - 2PC: each participant does a *different* part, all must do theirs, so it gives atomicity with low availability (any crash can stall everything).
    - Combine them: make the TC and each participant a Raft-replicated group. You get availability (any one server can fail) plus atomic multi-shard commit. This is the structure of Spanner and of lab 4's shard migration.

- **Decisions & trade-offs**
  - Pessimistic vs optimistic: chosen by conflict rate.
  - 2PC only within one trust and administrative domain (one datacenter or organization). Never across organizations: you would be letting another company's crash hold your locks. Inter-bank transfers don't use 2PC.
  - Avoid needing 2PC: if the store only supports single-record transactions, no 2PC is needed. Design data so related records live on the same shard.
  - Shrink the blocking window: keep the prepare-to-commit phase as fast as possible, and use specialised variants.

- **Rules of thumb**
  - Define correctness first (enumerate the legal serial outcomes), then check implementations against it.
  - Hold locks until commit or abort.
  - Write to stable storage *before* promising anything (vote YES, send COMMIT, reply to client).
  - Make every message carry a transaction ID, and make every handler idempotent (duplicate COMMIT/ABORT is safe).
  - Before you've promised, a timeout means abort. After you've promised, you must wait.

- **Failure modes & gotchas**
  - Half-applied transfers when a client or server dies between shards.
  - A participant discovering an invalid state late (overdraft) after another has already applied its part. PREPARE exists to catch exactly this.
  - Blocking with locks held cascades into other transactions stalling.
  - Lost messages: retries must be safe.
  - Deadlocks from inconsistent lock ordering.

- **Relevance to Sam's work**
  - **Bookings across suppliers.** A multi-leg trip (hotel via RateHawk + tour via Viator) is a cross-organization transaction, exactly where the teacher says 2PC must not be used. Use instead: local durable state machines, provider "hold/prebook" steps as a pseudo-prepare where available, then confirm, plus compensations (cancel/refund) if a later leg fails (saga-style).
  - **Idempotency.** Give each booking a unique ID generated before the first call and logged durably. Make "confirm" and "cancel" handlers tolerate duplicates. Use the provider's idempotency key or partner reference where one exists.
  - **The "after YES, must wait" lesson.** If the supplier call timed out *after* you sent "confirm", you don't know the outcome. Don't assume failure and rebook. Query status or reconcile.
  - **Pipelines.** When updating master location data in two places (DB + search index), log the intent first and make the apply step re-runnable.

## Robert Morris (MIT 6.824) — Lecture 13: Spanner (https://www.youtube.com/watch?v=4eW5SWBi7vs)
**Spanner offers serializable, externally consistent transactions over geo-replicated data by (1) running 2PC over Paxos-replicated participants so a crashed coordinator can't block anyone, and (2) giving read-only transactions lock-free, local-replica reads using multi-version snapshots, timestamps from bounded-uncertainty clocks (TrueTime), and commit-wait.**

- **Mental models / first principles**
  - Motivation: Google's ads DB was manually sharded over many MySQL/Bigtable instances (painful resharding), with no cross-shard transactions. The workload was dominated by read-only transactions (billions vs millions of read-write in table 6). It needed serializability *and* external consistency.
  - **External consistency (≈ linearizability):** if T1 finishes committing before T2 starts in real time, T2 must see T1's writes. In other words, no stale reads. Programmers find this the easiest model; weaker models push anomaly-handling onto app code.
  - **Layout.** Data is sharded by key. Each shard is replicated across several datacenters, and each shard's replicas form an independent Paxos (Raft-like, leader-based) group replicating a log. Sharding gives parallel throughput. Geo-replication gives disaster tolerance and copies near users. A majority quorum means one slow or flaky datacenter doesn't stall progress.
  - Two challenges: (1) a local replica may be in the lagging minority, which risks stale reads; (2) transactions span shards, so they need distributed transactions.

- **Mechanisms**
  - **Read-write transactions = 2PL + 2PC, where each participant is a Paxos group.**
    1. The client picks a unique transaction ID.
    2. It does all reads first. A read goes to the shard's Paxos leader, which takes a read lock (locks live only on the leader).
    3. It computes, then sends all writes at the end, choosing one participating group as the coordinator (TC).
    4. Each participant leader logs PREPARE through Paxos (waiting for a majority), then votes YES to the TC.
    5. The TC logs COMMIT through Paxos, then tells the other groups. They log the commit, apply the writes and release locks.
    - Because the TC's state lives in a replicated log, if the TC leader dies a follower takes over, sees the decision and finishes. This removes 2PC's indefinite blocking. The teacher calls this a really big deal, because blocking otherwise makes 2PC unacceptable at scale.
    - Cost: many cross-datacenter messages. About 100 ms per transaction with replicas on both US coasts (table 6), about 14 ms with nearby datacenters (table 3). Throughput is still high because non-conflicting transactions run in parallel across shards.
  - **Read-only transactions.** These read from the *local* replica, take no locks, skip 2PC and need no coordinator. This gives about 10x lower latency, and they don't slow writers, since writers never wait on readers' locks.
  - **Why "just read the latest value" fails.** T3 reads x after T1, then (slowly) reads y after T2. It sees x from T1's state and y from T2's state, which matches no serial order.
  - **Snapshot isolation.** Each transaction gets a timestamp: read-write at commit time, read-only at start. Storage is **multi-version**: every write adds (key, ts, value) instead of overwriting. A read at ts returns the newest version with timestamp ≤ ts, so all of a transaction's reads behave as if they happened at the same instant. The equivalent serial order is timestamp order.
    - Reading an "old" value is legal when the writer was concurrent with the reader. External consistency only constrains transactions that finished before the reader started.
    - Cost: storage and bookkeeping for multiple versions. Old versions must be garbage-collected, at minimum back to the oldest running transaction (the GC policy isn't stated).
  - **Safe time.** Paxos leaders send log entries in increasing timestamp order. A replica asked to read at ts=15 that has only seen entries up to 13 *waits* until it has seen ts ≥ 15. This fixes the lagging-minority problem at the cost of possible delay.
  - **Clock problem.**
    - A timestamp that is too *large* is safe but slow, because the read waits on safe time.
    - A timestamp that is too *small* (clock behind or slow) misses recently committed writes, which **violates external consistency**.
    - Perfect sync is impossible. Time comes from UTC labs via GPS/radio/NTP, to per-datacenter time masters, to servers. Uncertainty comes from unknown propagation delay, request/response round-trip ambiguity (a 1 s RTT means the true time could be anywhere within that 1 s), and local oscillator drift between syncs (milliseconds).
  - **TrueTime.** `TT.now()` returns an interval [earliest, latest] guaranteed to contain the true time.
    - **Start rule:** a transaction's timestamp = TT.now().latest (at start for read-only, at commit start for read-write).
    - **Commit wait:** a read-write transaction must not commit, make its writes visible or release locks until TT.now().earliest > its timestamp, i.e. until its timestamp is definitely in the past.
    - Together these guarantee that any transaction starting after T1 commits gets a strictly larger timestamp and therefore sees T1. Example: T1 gets [1,10] so picks ts=10, waits until earliest ≥ 11, then commits. T2 starts later, gets [10,12], picks 12 and reads T1's write.
    - Wait length scales with clock uncertainty, which is why tight clock sync (GPS plus time masters) pays off directly in latency.
  - **Separating the two guarantees.** Snapshot isolation alone gives serializable read-only transactions. Synchronized timestamps plus commit-wait add external consistency. Snapshot isolation without good clocks doesn't, because different machines pick timestamps.

- **Decisions & trade-offs**
  - Locks plus 2PC for writers (correct, slow, cross-datacenter) vs snapshot reads for readers (fast, local, lock-free). This is optimising for a read-dominated workload.
  - Replicating the coordinator trades extra Paxos rounds for not blocking.
  - Multi-version storage trades space and GC for lock-free consistent reads (plus "read as of yesterday" snapshot reads).
  - Clock-uncertainty budget vs commit latency.
  - Open-source descendant: CockroachDB.

- **Rules of thumb**
  - Characterise the read:write ratio and give reads a separate, cheaper path when possible.
  - To read multiple items consistently without locks, pin *one* snapshot timestamp per logical read and read every item "as of" it.
  - Never trust wall clocks for ordering unless you know their error bound. If you must, pick timestamps conservatively and wait out the uncertainty.
  - A replica should refuse to answer for a time it hasn't caught up to.
  - Make the coordinator's decision durable and replicated so failover can finish in-flight work.

- **Failure modes & gotchas**
  - Non-serializable "read latest" audits.
  - A slow or behind clock gives silent stale reads.
  - Multi-version storage growing without GC.
  - Long safe-time or commit-wait delays when clocks are poorly synced.
  - Read-write latency across continents (~100 ms each).

- **Relevance to Sam's work**
  - **Consistent snapshots for pipelines:** when a job reads several tables or files that are being updated (master locations, supplier catalogues), read them all "as of" a single version or timestamp (e.g. a dated snapshot partition) so joins don't mix states.
  - **Versioned records:** keeping (id, valid_from, value) rows rather than overwriting gives reproducible runs and easy rollback for location datasets. Budget the disk and GC old versions.
  - **Don't order events by machine clocks across systems** (e.g. supplier webhook timestamps vs your server time). Use sequence numbers or the source-of-truth's version, or allow for clock skew.
  - For booking confirmation, the external-consistency idea means: after a booking commits, any later read (the user's "my trips" page) must see it. Route such reads to the primary or a caught-up replica, not a lagging cache.

## Robert Morris (MIT 6.824) — Lecture 16: Cache Consistency: Memcached at Facebook (https://www.youtube.com/watch?v=Myp8z0ybdzM)
**At scale a cache exists mainly to shield a slow database from load, not to cut latency. Once you depend on it, every cache failure, cold start or invalidation storm threatens the DB, and racing writers can leave stale data cached forever unless you use invalidation plus leases.**

- **Mental models / first principles**
  - **The standard website evolution:**
    1. One box (web server + PHP + MySQL).
    2. CPU runs out first, so add many stateless front-ends over one DB. Stay on one DB as long as possible, because multiple DBs bring sharding logic in app code and distributed transactions.
    3. The DB runs out, so shard MySQL by key. App code must know the shard map, resharding is painful, cross-shard transactions need 2PC, and it's expensive.
    4. Add a cache layer (memcached), which gives at least 10x more reads/sec than MySQL on the same hardware. Websites are read-heavy, so reads run out before writes, and a hot key can't be fixed by further sharding.
  - **Look-aside vs look-through.** Memcached is look-aside: the front-end checks the cache; on a miss it queries the DB itself and then `set`s the result. The cache knows nothing about the DB, which matters because front-ends often cache *derived* values (processed or aggregated query results, partial HTML). Only app code knows how to recompute them. The cost is that consistency becomes the application's problem.
  - **Fragility.** With a 99% hit rate the DB sees about 1% of reads. Lose a cache server or the hit rate, and DB load can jump 50x. Design must ensure the DB never sees anything close to full load.
  - **Facebook's consistency needs are relaxed:** a few seconds of staleness is fine. Two things are not: (1) data stale *indefinitely* (hours or days), and (2) users not seeing *their own writes*.

- **Mechanisms**
  - **Read path:** `v = get(k)` (client hashes k to choose a memcached server). If nil, run SQL, then `set(k, v)`.
  - **Write path:** send to DB, then `delete(k)` in memcached.
    - The DB also emits deletes asynchronously by tailing its commit log (McSqueal) to local caches and, through replication, to the other region's caches.
    - The front-end's own delete exists so the writer reads its own write.
    - Delete *after* the DB write. Deleting first lets another reader refill the cache with the old value before the DB update lands.
  - **Why invalidate rather than update (set new value):** two clients increment x. C1 writes x=1, C2 writes x=2, then C2 sets cache=2 and C1 (late) sets cache=1. Now the cache disagrees with the DB indefinitely. Delete is idempotent and always safe.
  - **Regions.** A primary region (West Coast) takes all writes. Secondaries hold full replicas via MySQL async log replication (lag of seconds). Reads are always local. Reasons: users are near a datacenter (coast-to-coast is about 50 ms), and one page needs hundreds of lookups, so remote reads would be fatal. Writes from a secondary pay the cross-country trip, which is fine at a high read:write ratio.
  - **Partition vs replicate (the core resource decision).**
    - *Partition (shard by key hash):* RAM-efficient (10 servers × 1 GB = 10 GB distinct data). Doesn't help a single hot key. Each front-end talks to every server, giving N² TCP connections and incast.
    - *Replicate:* parallel serving of hot keys and fewer connections, but 10 servers × 1 GB = only about 1 GB distinct data.
    - Facebook does both. Within a region the DB is sharded, not replicated. Memcached is organised in **clusters**, each a set of front-ends plus its own sharded memcached servers. Clusters replicate one another's hot data.
    - Keep clusters small to limit N² connections, incast congestion (hundreds of simultaneous replies dropping packets) and the cost of a single huge datacenter network.
    - A shared **regional pool** caches unpopular keys once, saving RAM.
  - **Cold start of a new cluster.** An empty cache gives 100% misses, so sending half the traffic to it could raise DB load from 1% to 50% of reads (50x). Fix: a cold cluster, on a local miss, reads from a *warm* cluster's memcached, fills its local cache, and only then falls back to the DB. This runs for a few hours.
  - **Thundering herd.** Deleting a very hot key makes hundreds of front-ends miss at once and all hit the DB for the same row. **Leases:** on a miss, memcached gives the *first* requester a lease token. Others are told to wait briefly and retry. The lease holder reads the DB and sets with the token. Leases time out in case the holder dies.
  - **Memcached server failure.** Front-ends would fall through to the DB (possibly about 1M reads/s). The **gutter pool** is a small set of normally idle servers. On a timeout, the client retries against the gutter (hashing the key), and on a gutter miss reads the DB and fills the gutter. Deletes are not sent to gutter servers, since any gutter could hold any key and sending them would double delete traffic. Instead gutter entries expire very quickly. Automated replacement of the failed server runs in the background.
  - **Stale-set race (the key consistency bug).**
    1. C1 misses and reads v1 from the DB.
    2. C2 writes v2 and deletes k (the DB also deletes).
    3. C1, slow, finally does set(k, v1).
    - Result: stale *forever*.
    - Fix with the same leases: a delete *invalidates any outstanding lease* on that key, and a set with an invalidated lease is ignored. If the delete arrives after the set, the stale value lives briefly and then the late delete removes it. Either way there is no permanent staleness.

- **Decisions & trade-offs**
  - Look-aside (generic, app-controlled, cache derived data) vs look-through or integrated (easier consistency). Facebook's complexity stems from bolting together independent pieces. The teacher points to Yahoo PNUTS as an integrated design that gets multi-site replication with consistency.
  - Invalidate vs update on write.
  - Partition vs replicate, at every tier.
  - Full replication across regions (fast local reads, simple) vs partitioning across regions (cheaper writes and storage, but remote reads).
  - Weak but bounded staleness plus read-your-writes, as a deliberate product-driven consistency target.

- **Rules of thumb**
  - Decide what staleness the product tolerates, and separately guarantee read-your-own-writes and no unbounded staleness.
  - On write: update the source of truth first, then delete the cache entry. Never set derived data from a writer.
  - Protect the backing store: leases or single-flight on misses, warm-up for new caches, a fallback pool for failed caches, and alerts when hit rate drops.
  - Treat the cache as load-bearing infrastructure once hit rate is high.
  - Make deletes idempotent and safe to repeat or broadcast.
  - Bound fan-out (N² connections, incast) by grouping into clusters.

- **Failure modes & gotchas**
  - A slow reader setting an old value after a delete gives permanent staleness.
  - Concurrent update-sets reorder.
  - Delete-before-write races.
  - Hot key deletion causes a DB stampede.
  - Cache node loss or a new empty cluster floods the DB.
  - Gutter servers holding stale data if not expired fast.
  - Async replication lag means a secondary-region user may not see their write unless special handling exists. (note: the paper adds a "remote marker" mechanism for this, which the lecture doesn't cover.)

- **Relevance to Sam's work**
  - **Supplier API caching (availability, prices, static content from Viator/RateHawk):** use look-aside caching with TTLs. Use single-flight or lease logic so a cache expiry on a popular product doesn't fire 100 concurrent supplier calls and blow the rate limit. This is exactly the thundering herd, with the supplier API in the DB's place.
  - **Prices and availability at checkout** must bypass or revalidate the cache, because the product's staleness tolerance there is close to zero. Browsing pages can tolerate seconds to minutes.
  - **Avoid the stale-set race in Python services:** on a cache miss, attach a version or ETag, and refuse to write back if the key was invalidated meanwhile (e.g. Redis WATCH or compare-and-set with a version).
  - **For a laptop-scale pipeline,** the partition-vs-replicate thinking applies to lookup tables. Shard a huge GeoNames lookup by key across worker processes (RAM-efficient), rather than giving every worker a full copy (16 GB disappears quickly). Replicate only small, hot tables.

## Robert Morris (MIT 6.824) — Lecture 17: COPS, Causal Consistency (https://www.youtube.com/watch?v=fR_NB714EAI)
**Can a geo-replicated store do both reads and writes purely locally and still give sane semantics? Eventual consistency gives speed but anomalies. Causal consistency (COPS) fixes "I saw B, which depended on A, but A is missing" by shipping per-write dependency lists and delaying remote application until dependencies are visible, at the cost of metadata, cascading waits and no real transactions.**

- **Mental models / first principles**
  - **Setting:** several datacenters, each holding a full sharded copy, with a read-dominated web workload. Spanner (Paxos across datacenters) and Facebook memcache (all writes via the primary region) both make *writes* cross datacenters. COPS asks for **local writes too**: lower client latency, and the system keeps working when other datacenters are slow or down. The approach is to get performance first, then see what consistency can be had.
  - **Anomalies aren't bugs if the spec allows them**, but they make correct-looking code wrong.
  - **Canonical anomaly:** `put(photo)` then `put(list += photo)`. Under eventual consistency the list update can reach another datacenter before the photo, so a reader sees the list entry but `get(photo)` fails. The defensive-coding workaround is to retry or skip missing items.

- **Mechanisms**
  - **Strawman 1, eventual consistency.**
    - A write goes to the local shard server, which replies immediately and asynchronously streams the write to the same shard in other datacenters.
    - Highly parallel, since each shard pushes independently.
    - Variations: cheap writes with expensive reads (read all datacenters), or quorum read/write. Real-world examples: Amazon Dynamo and Cassandra.
    - Guarantee: if writes stop, all replicas converge. There is no ordering.
  - **Picking a winner among concurrent writes** needs version numbers that every datacenter orders identically.
    - Wall-clock timestamps with a unique datacenter/server ID in the low bits break ties.
    - A fast clock lets an earlier write beat a later one, or blocks others until clocks catch up.
    - **Lamport clocks** fix this: each server tracks Tmax (highest version seen), and new version = max(Tmax+1, wallclock). Versions stay roughly real-time but never go backwards relative to anything observed.
  - **Conflicting writes and last-writer-wins.** COPS uses last-writer-wins, which is deterministic but loses information. Two concurrent increments from 10 both write 11; two shopping-cart adds should union. Better tools are transactions, mini-transactions (atomic increment or test-and-set), or app-specific merge functions. These are generally hard, and weakly consistent systems just live with it.
  - **Strawman 2, eventual consistency plus barriers.** `put` returns a version, and `sync(k, v)` waits until *all* datacenters have k at version ≥ v.
    - Writer: put(photo), sync, then put(list).
    - Reader: read list first, then photo. If you see the later item, the earlier one is guaranteed present.
    - Slow writes. Blocks if a datacenter is down, so real systems sync to a majority and read from an overlapping quorum.
    - Arguably acceptable, since Spanner and Facebook already pay cross-datacenter cost on writes. But it puts the burden on the programmer to place syncs and order reads.
  - **Log-server approach.** Each datacenter funnels all writes through one log that ships in order. Order is preserved without client waits, but the single log server becomes a throughput bottleneck as shards grow. COPS rejects it.
  - **COPS (non-transactional).**
    - The client library keeps a **context**: the (key, version) of everything it has read or written.
    - `put(k, v)` sends the dependencies along. The local shard assigns a version (Lamport), stores the value and replies immediately, then forwards (k, v, version, deps) to the other datacenters.
    - At the remote datacenter, the shard for k asks the local shards for each dependency, "is key x at version ≥ n visible?", and **holds the write until all deps are installed**.
    - Result: any reader who sees Z v3 is guaranteed to see X ≥ v2 and Y ≥ v4. This fixes the photo example automatically.
    - Dependency tracking is automatic in the library, so the programmer just does gets and puts.
    - Optimisation: after a put, the context collapses to just that put's version, since dependencies are transitive and remote datacenters already won't install it until its own deps are present.
  - **Definition of causal consistency.**
    - Dependencies come from (a) program order within a client thread (put→put, get→put) and (b) reads-from (a get depends on the put that wrote the value). They are transitive.
    - If B depends on A and a client sees B, that client must subsequently see A (or newer).
    - Unrelated writes carry no ordering obligations, which allows parallelism. Real-time order between independent clients imposes nothing, unlike linearizability.

- **Decisions & trade-offs (and why causal consistency isn't widely deployed)**
  - **Cascading waits:** a write can wait on a dependency that is itself waiting on another, and a network partition can block application indefinitely. Mere slowness hurts. Follow-up papers mitigate this.
  - **Only tracked causality counts.** A phone call or email ("go look, I updated it") is invisible, so users can still see "stale" data.
  - **Some invariants need multi-key consistent snapshots.** Example: a photo list plus its ACL. With either read order you can get an old ACL with a new list, or a new ACL with an old list, and leak a private photo. COPS-GT (get transactions) returns dependencies so the client can detect mismatched versions and refetch.
  - **Per-user context is hard to maintain** when a browser hits different web servers.
  - **Metadata overhead** (storing and shipping dependencies) may be significant at millions of ops/s.
  - Only very limited transactions are possible, while users increasingly want real transactions. In practice people choose eventual or strong consistency.

- **Rules of thumb**
  - Know your store's consistency model and design read and write order around it. Write the referenced object first and the reference second. Read the reference first and the referenced object second.
  - Use version numbers that are monotonic and totally ordered (Lamport plus node ID). Don't rely on raw wall clocks.
  - Last-writer-wins silently drops concurrent updates. For counters and sets, use atomic ops, CRDT-like merges or transactions.
  - When several keys must be mutually consistent (data plus permissions), you need a snapshot or transaction, not just causal order.

- **Failure modes & gotchas**
  - Dangling references (list entry before object).
  - Lost increments under last-writer-wins.
  - Fast clocks hijacking versions.
  - sync/barrier blocking on a dead datacenter.
  - Unbounded remote waits under partition.
  - ACL/data skew leaking private data.
  - Out-of-band causality not honoured.

- **Relevance to Sam's work**
  - **Ordering writes across stores in pipelines:** write the child object (e.g. location row, image, product record) before the parent index or list that references it. Readers should tolerate or retry missing referents. This is the cheap version of causal ordering for any eventually consistent store (S3, search indexes, caches).
  - **Supplier webhooks and events can arrive out of order.** Carry a monotonic version per entity (supplier timestamp plus sequence, or Lamport-style max(seen+1, now)) and apply only newer versions. Never let last-writer-wins on booking status silently drop a cancel that arrived "late".
  - **Booking state should use explicit state transitions** (pending → confirmed → cancelled), not blind overwrites, so concurrent updates are detected rather than lost.
  - Permission-like checks (who can see a trip or an itinerary) need a consistent read of both the data and the permission, from one snapshot or the primary.
