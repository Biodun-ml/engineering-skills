# mit-2 — cross-cutting principles

1. **The replication loop.** Performance -> sharding -> constant faults -> replication -> inconsistency -> consistency costs performance. Decide consciously where on that curve each piece of data sits; money/bookings need strong consistency, logs/analytics usually don't. [GFS, L3]
2. **You cannot tell "dead" from "unreachable".** Any failover that acts on silence alone risks split brain. Use an external authority (test-and-set/lease service, configuration manager) or a majority vote. [GFS L3; VMware FT L4; Raft L6; Chain replication L9]
3. **Leases bound the damage of a deposed leader.** A new primary may only be appointed after the old one's lease has expired; the old one stops serving on expiry by its own clock. [GFS L3]
4. **Majorities of 2F+1 tolerate F failures, and any two majorities overlap.** That overlap is what carries terms and committed data from one leader to the next. Count the majority out of all servers, not live ones; two nodes alone can't fail over safely. [Raft L6]
5. **Never externalise an effect before the cause is durable elsewhere (the output rule).** Reply/confirm only after the operation is replicated/committed (backup acked, or majority persisted). [VMware FT L4; Raft L6/L7]
6. **Duplicates are unavoidable at failover; plan for at-least-once.** Every scheme needs request IDs and a server-side dedup table that replays the original reply; clients/consumers must tolerate duplicate outputs. [GFS L3; VMware FT L4; ZooKeeper L8; Chain replication L9]
7. **Partial failures leave replicas divergent unless the protocol forbids it.** GFS lets failed appends land on some replicas (so readers see duplicates/gaps); making it strong needs dedup, mandatory application or removal of secondaries, two-phase commit, and tail resync on failover. [GFS L3]
8. **Replicated state machine: same start state + same inputs in the same order + determinism = same state.** Capture every source of non-determinism (time, randomness, thread interleaving) at the leader and ship the result. Ship inputs (small) rather than state (big), but keep state transfer/snapshots to bootstrap new replicas. [VMware FT L4; Raft L6]
9. **The log is the source of truth; state is derived.** Logs give a global order, hold tentative entries, enable retransmission, and rebuild state by replay; bound them with snapshot + suffix, and give lagging followers a full-snapshot catch-up path. [Raft L6/L7; GFS master log + checkpoints L3]
10. **Persist the minimum you can't reconstruct, and fsync it before you promise anything.** `write()` is not durable; fsync is ~10 ms on spinning disk (~100 ops/s ceiling), ~0.1 ms on SSD. Batch many operations per fsync. [Raft L7]
11. **Linearizability is the precise meaning of "behaves like one server".** One total order respecting real time; reads see the latest preceding write; no stale reads; all clients agree on write order. Test a history by finding an order or a cycle. [Raft L7; ZooKeeper L8]
12. **Read scaling vs freshness.** Leader-based replication gets slower as you add replicas. Serving reads from followers scales reads but makes them stale; ZooKeeper keeps writes linearizable plus per-client FIFO / read-your-writes via a monotonic zxid token, and `sync` for fresh reads on demand. [ZooKeeper L8]
13. **Multi-part state needs a commit marker and change notification.** Delete ready -> write parts -> create ready; readers watch ready so they detect torn reads. Equivalently: write a new version, flip a single pointer last. [ZooKeeper L8]
14. **Read-modify-write must be a conditional write (compare-and-set on a version) in a retry loop**, with randomised exponential backoff under contention. [ZooKeeper API L9]
15. **Avoid herd effects.** Wake one waiter per release (sequential-node queue), not all; jitter retries. [ZooKeeper locks L9]
16. **Distributed locks are not mutexes.** A crashed holder releases the lock mid-update; protected work must be idempotent or self-repairing (or only used as soft locks / leader election). [ZooKeeper locks L9]
17. **Separate control plane from data plane.** A small consensus-backed configuration manager decides membership and sharding; bulk data uses the cheapest suitable replication (chain, primary-backup). Chain replication is cheaper per leader and simple to recover, but every write is as slow as the slowest node; majority schemes tolerate stragglers. [Chain replication L9]
18. **Single masters are simple and eventually hit RAM, request-rate and manual-failover limits.** Watch metadata growth (many small files/items) in any central index. [GFS L3]
19. **Bound queues between fast producers and slow consumers.** Raft has no flow control: followers that ack faster than they apply grow logs until memory runs out. [Raft L6]
20. **Debugging method: ask "what's the most awkward moment to fail?" and "can this state actually arise?"** Construct the crash sequence before reasoning about the protocol's response. Assume an entry was committed if a client could have seen it. [VMware FT L4; Raft L6/L7]

## Robert Morris (MIT 6.824) — Lecture 3: GFS (https://www.youtube.com/watch?v=EpIgvowZr00)
**Big storage forces a loop: performance -> sharding -> constant faults -> replication -> inconsistency -> paying for consistency costs performance; GFS deliberately accepts weak consistency to stay simple and fast.**

**Mental models / first principles**
- The recurring design loop: you want aggregate throughput, so you shard data over many machines; with hundreds or thousands of machines something is always broken, so you need automatic fault tolerance; the strongest tool for that is replication; replicas drift unless carefully coordinated; coordination means extra messages and round trips, which costs the performance you started out wanting. Every storage system sits somewhere on this trade-off.
- "Strong consistency" intuition: the system should behave exactly like one single-threaded server holding one copy, executing requests one at a time, each read seeing all earlier writes. Even this ideal is ambiguous for concurrent writes: you can only say whether observed results are *consistent with some* serial order (if one reader sees value 2, every later reader must also see 2).
- A naive two-replica design (clients write to both servers independently) breaks immediately: servers can apply concurrent writes in different orders, so readers get different answers; and "read from server 1 unless it's down" lets a value silently flip on failover with no write having happened.
- GFS's stance (2003, heretical for academia): a storage system may return slightly wrong/stale/duplicated data if applications can cope. Acceptable for web crawls, search indexes, logs; not acceptable for things like ad billing or bank balances.

**Mechanisms**
- Architecture: one master (naming + chunk locations) and many chunk servers (actual bytes). Separating metadata from data lets each be designed independently.
- Files split into 64 MB chunks, 3 replicas by default. Each chunk stored as an ordinary Linux file on a chunk server.
- Master state: (1) filename -> array of chunk handles [persisted]; (2) chunk handle -> {replica servers [not persisted; rebuilt by asking chunk servers on reboot], version number [persisted], current primary [volatile], lease expiry [volatile]}.
- Persistence via an append-only log on disk plus periodic checkpoints. Log appends are cheap (batch many records into one sequential write) vs random b-tree updates; recovery = load latest checkpoint + replay the log tail. The master's disk write rate caps how fast metadata can change.
- Read: client sends (filename, offset) to master; master computes chunk index (offset / 64MB), returns handle + replica list; client caches this and reads from the nearest replica (same rack/switch). The client library splits reads that cross chunk boundaries.
- Record append (the main write path): client asks master for the last chunk. If no primary exists, master finds replicas whose version equals the master's recorded latest version, picks a primary, increments the version, persists it, informs primary/secondaries, and grants the primary a 60-second lease. Client pushes data to all replicas (actually along a chain chosen to avoid cross-switch bottleneck links), into a temp area. Then client tells primary to commit; primary serialises concurrent appends, picks the offset, writes locally and tells secondaries to write at the same offset. All say yes -> success; any failure -> error to client, and the client retries the whole append from scratch.
- Version numbers distinguish stale replicas (e.g. a server down for a week). If no server with the current version is reachable, the master waits rather than serve stale data. (Morris notes the paper is murky about whether the master persists the new version before or after telling replicas; neither order is obviously safe.)
- Leases prevent split brain: if the master cannot reach a primary, it may be a network partition, not a crash; the primary may still be serving clients. So the master must wait out the lease before appointing a new primary. Also clients cache primaries, and messages naming the old primary can be in flight, so "just tell clients about the new one" doesn't work.

**Consistency consequences**
- A failed append may have landed on some replicas but not others. Retries mean a record can appear twice, at different offsets, with padding/gaps; different replicas show different orders (e.g. A,B,C,B on one replica vs A,gap,C,B on another). If the client crashes before retrying, a record exists on some replicas forever and not on others.
- Guarantee: if append returned success, the record exists at the same offset on all replicas. Nothing stronger.
- Applications compensate: embed checksums and clear record boundaries, include unique IDs/sequence numbers to drop duplicates, or avoid concurrent appends to order-sensitive files (single writer, sequential).

**What it would take to get strong consistency** (preview of Raft labs)
- Primary-side duplicate request detection so retries don't create duplicates.
- Secondaries must execute what the primary tells them, or be removed from the replica set; no silently ignoring.
- Don't expose data to readers until all replicas have promised they can apply it: two-phase commit (prepare/promise, then commit).
- New primary after a crash must resynchronise the tail of operation history with secondaries.
- Either route all reads through the primary or give secondaries leases too, so stale replicas can't serve reads.

**Decisions & trade-offs**
- Single master: dead simple, globally consistent view of metadata. Cost: master RAM limits number of files/chunks; master CPU/disk caps request rate; failover was manual (tens of minutes). These were exactly the problems that hit Google later.
- Weak append semantics: high parallel throughput, simple code; cost pushed to every application.
- Optimised for large sequential reads/writes and throughput, not low latency or small random updates.
- Single data centre only; cross-region replication is much harder.

**Rules of thumb**
- Decide explicitly what consistency your consumers need; "eventually readable, may contain duplicates" is fine for logs and analytics, not for money.
- When storage can duplicate or reorder, make records self-describing: ID, checksum, length/boundary marker. Deduplicate downstream.
- Persist only what can't be reconstructed (GFS persists filename->chunks and versions; replica locations are rediscovered).
- Use an append-only log + checkpoints for durable metadata.
- Never appoint a new leader just because you can't reach the old one; wait for its lease to expire.

**Failure modes & gotchas**
- Network partition makes "unreachable" indistinguishable from "dead" -> split brain.
- Cached routing info (who is primary) can be stale even if just fetched.
- Partial failure of a multi-replica write leaves replicas divergent; retries create duplicates.
- Central metadata server becomes RAM and throughput bottleneck as file counts grow (many small files is the killer).

**Relevance to Sam's work**
- Append-only data pipelines (e.g. writing chunked outputs of millions of rows): GFS-style "at-least-once append" means you must dedupe by a stable record key and validate checksums/row counts per chunk.
- Supplier API integrations: a failed call may have partially succeeded on the far side (like a failed append that landed on some replicas). Retrying without an idempotency key/dedup creates duplicate bookings. The primary-side duplicate detection fix is the same idea as an idempotency key.
- 64MB-chunk + metadata-index design is a good template for chunked processing on a 16GB laptop: keep a small in-memory index of chunk -> location/status, keep bulk data in files, process chunks in parallel, checkpoint progress to an append-only log so a crash restarts from the last checkpoint.
- Many tiny files kill a single metadata master; batch small records into large chunks.

## Robert Morris (MIT 6.824) — Lecture 4: Primary-Backup Replication / VMware FT (https://www.youtube.com/watch?v=M_teob23ZzY)
**Keep a backup identical by feeding it the same inputs in the same order, never release output until the backup has the input that produced it, and use an external arbiter to stop both sides going live.**

**Mental models / first principles**
- Replication only defends against *fail-stop* faults: a machine that simply stops (power pulled, network cut, CPU overheats and halts). It does not defend against software bugs or hardware that computes wrong answers: both replicas will agree on the same wrong result. Checksums/ECC convert some silent corruption into detected (fail-stop) faults.
- Replication assumes *independent* failures. Correlated failures defeat it: identical machines from the same batch with the same defect, one data centre hit by power loss/earthquake/fire. Put replicas in separate failure domains if that matters.
- Whether replication is worth 2-3x hardware is an economic question: cost of downtime vs cost of the replica. A bank: yes; a course web server: no.
- Two approaches:
  - *State transfer*: primary ships its whole state (or changed parts) to the backup. Simple, robust to parallelism/non-determinism, but state is big.
  - *Replicated state machine (RSM)*: ship only the external inputs; if both start in the same state and apply the same inputs in the same order deterministically, they stay identical. Inputs are usually tiny compared to state (a GB database vs "write key 27"), so RSM is usually preferred, but it is more complex and relies on determinism.
- Even RSM needs state transfer to bootstrap a fresh replica after one dies. With two replicas, after one failure you are living on borrowed time; re-replicate fast.
- Level of replication: VMware FT replicates the entire machine (memory, registers), so it works for any unmodified software. Almost every other system (GFS and later papers) replicates at the *application level* (e.g. chunk table operations), which is far more efficient but must be built into the application.

**Mechanisms (VMware FT)**
- Primary and backup VMs on two physical machines, hypervisor (VMM) on each. Client packets reach the primary VMM, which delivers them to the guest and forwards a copy over the *logging channel* to the backup. Both execute; only the primary's output packets are actually sent; the backup VMM drops its outputs.
- Sources of non-determinism that must be captured: input packet contents *and* the exact instruction at which its interrupt fires; timer interrupts (~100/s); weird instructions (time-of-day, CPU ID, hardware random number). Multi-core interleaving is banned outright: the guest is single-core. (note: VMware's later multi-core FT reportedly moved to a state-transfer approach for this reason.)
- Log entry = (instruction count since boot, event type, data / result). The backup uses a CPU performance-counter feature ("interrupt me after N instructions") to inject the event at exactly the same instruction.
- The backup must never run ahead of the primary, so it only executes while it has at least one buffered event telling it where to stop next; it is always at least one event behind.
- Bounce buffers: the NIC DMAs into VMM-private memory; the VMM pauses the guest and copies the whole packet in at a recorded instruction, so the guest can't observe a partially-written packet at differing times.
- Failure detection: the logging channel is chatty (timer interrupts), so silence for ~1 s means the peer is presumed dead; the backup consumes all buffered log entries, then "goes live" (takes over the MAC/IP, starts emitting output).

**The output rule** (the key idea)
- Danger: primary replies "11" to an increment, crashes; the log entry to the backup was lost; backup takes over at 10 and later replies "11" again. A client saw a state that the survivor never had.
- Rule: the primary may not release an output until the backup has acknowledged receipt (buffering is enough, not execution) of all log entries up to that point. So if a client saw the reply, the backup has the input.
- Cost: every output waits a round trip to the backup. ~0.5 ms in adjacent racks, ~5 ms+ across cities; a microsecond-level service becomes millisecond-level. Throughput can be kept with many concurrent clients, but latency suffers. Holding output (not execution) softens it. Application-level replication can do better (e.g. read-only operations need not wait).
- Logging in the primary's own memory doesn't help: it disappears in a crash. The durable "log" is the backup's memory, hence the ack.

**Duplicates are unavoidable at cutover**
- If the primary crashes after releasing output, the backup replays the last input and emits the same reply again. Any replication system with failover either risks losing output or duplicating it; everyone chooses duplicates. FT gets away with it because the backup's TCP state is identical, so it emits the same sequence number and the client TCP stack discards the duplicate. Without TCP you need application-level sequence numbers/dedup on the client side.

**Split brain and the arbiter**
- A partition between primary and backup (both still talking to clients) makes each think the other is dead; both going live diverges state.
- Solution: an external test-and-set service (on the shared disk server). Whoever wants to go live must atomically flip the flag; the second requester is refused. Must be consulted on *every* go-live, because you can never distinguish "dead" from "unreachable". The arbiter itself must be replicated or it becomes the single point of failure.

**Rules of thumb / checklist**
- Ask always: "what is the most awkward moment for a failure, and what would the client observe?"
- Identify all non-determinism (time, randomness, thread scheduling, input arrival order) before claiming replicas stay in sync; make the leader decide and ship the result.
- Never externalise an effect (reply, payment confirmation, email) until the fact that caused it is durably recorded somewhere that survives the failure of the machine doing it.
- Design the client side to tolerate duplicate responses/effects.
- Failover needs an authority outside the two contestants.

**Failure modes & gotchas**
- Replicas drifting slightly leads to dangerous outcomes (e.g. a backup unaware of a lease the primary granted, then granting it again).
- Shared-hardware/shared-site correlated failures.
- Single-arbiter single point of failure.
- Synchronous replication across distance throttles latency.

**Relevance to Sam's work**
- Booking/payment flows are exactly the output-rule problem: persist the booking intent/state (DB commit, or a durable log) *before* telling the customer "confirmed" or calling the supplier; on retry/failover, expect duplicates and dedupe via idempotency keys (the analogue of TCP sequence numbers).
- Pipelines on one laptop: deterministic re-runs (fixed seeds, stable sort orders, no reliance on thread completion order) let you replay inputs to rebuild state, the RSM idea, instead of snapshotting huge in-memory state.
- If running a scheduler/worker on two machines, use a lock/lease in a shared store (Postgres advisory lock, Redis SET NX with expiry) as the test-and-set arbiter so two instances never both process the same job queue.

## Robert Morris (MIT 6.824) — Lecture 6: Fault Tolerance: Raft (1) (https://www.youtube.com/watch?v=64Zp3tzNbpE)
**Majority voting over an odd number of servers removes the single decision-maker that every earlier system hid in a corner, and lets a replicated log stay correct through crashes and partitions without split brain.**

**Mental models / first principles**
- MapReduce, GFS and VMware FT all replicate, but each relies on one entity (master, test-and-set server) to decide who is primary. That entity can't disagree with itself, which is its virtue, and it is a single point of failure, which is its flaw.
- Why two replicas can't fix it: build a replicated test-and-set on two servers. Rule "clients must reach both" -> less available than one server. Rule "reach whichever you can" -> under a partition, two clients each acquire the "lock" on different halves -> split brain. With two symmetric servers you must choose between fault tolerance and correctness.
- Pre-1990 workarounds: a network that physically cannot fail (expensive, carefully engineered) or a human tie-breaker (slow, itself a SPOF).
- The insight (Paxos and Viewstamped Replication, ~1990; widely deployed only ~15 years later): use an odd number of servers and require a *majority* for every step. At most one partition can contain a majority, so at most one side makes progress. 2F+1 servers tolerate F failures (3 tolerate 1, 5 tolerate 2). The majority is always counted out of *all* servers, not just live ones.
- The deeper reason it works: any two majorities overlap in at least one server. So a new leader's voters include someone who knew the previous term, and any committed entry is held by someone in every future majority.

**Mechanisms**
- Raft is a library under an application (e.g. a key-value store). Clients talk to the leader's application layer; the app calls `Start(cmd)`, which appends to the leader's log and returns the index/term it *would* commit at. Raft sends AppendEntries RPCs; once a majority (counting the leader) has the entry, it is committed. Raft then emits an ApplyMsg (index, command) on a channel; only then does the app execute and reply. All replicas apply the same commands in the same order, so (if deterministic) their state evolves identically.
- Followers learn the commit point via the leaderCommit field piggybacked on the next AppendEntries/heartbeat; no separate commit message. Followers applying late is off the client's critical path.
- Why logs matter: (1) the leader's numbered slots define the global order; (2) followers hold tentative, uncommitted entries that may be discarded; (3) the leader keeps entries to retransmit to lagging followers; (4) persisted to disk, the log lets a rebooted server rebuild its state by replay (later: snapshots/checkpoints to avoid full replay).
- After a total crash and reboot, no server knows the commit point. A new leader forces followers' logs to match its own; once matched on a majority, those entries are committed and everyone re-executes.
- Leader election: each server has an election timer reset by every AppendEntries. On timeout: increment term, vote for self, send RequestVote to others. Each server votes at most once per term; majority wins, so at most one leader per term. The winner announces itself implicitly by sending AppendEntries for that term (only a leader may). Followers mostly need the current term, not the leader's identity.
- Elections can fail by producing zero leaders: too few live servers, or a split vote. Randomised timeouts make split votes unlikely.

**Decisions & trade-offs**
- Leader vs leaderless: original Paxos had no leader; a leader allows agreement in one round of messages per request in the common case (~2x faster) and is easier to reason about.
- Election timeout tuning: minimum must be a few heartbeat intervals (heartbeat 100 ms -> minimum ~300 ms) so dropped heartbeats don't trigger elections. Maximum bounds recovery time (the system is frozen with no leader until a timer fires). The random spread must exceed the RPC round-trip (e.g. >10 ms) so the first candidate finishes collecting votes before the next timer fires. Pick a *fresh* random value every reset, or two unlucky servers can split votes forever.
- Raft has no flow control: if followers acknowledge faster than they execute, logs grow unboundedly until memory runs out; production systems add throttling/back-pressure.

**Failure modes & gotchas**
- Spurious elections happen with a healthy leader if the network is slow; the old leader may still think it's leader. It is harmless because, in a minority partition, it can never assemble a majority and so never commits or replies; clients just time out.
- One-way network failure (leader can send heartbeats but not receive) can stall the cluster; fix by having the leader step down when it stops getting heartbeat replies.
- Logs diverge transiently, e.g. leader crashes mid-broadcast. Example to reason through: slot 10 on all three servers, slot 11 on two, slot 12 has term 4 on one and term 5 on another. 10 and 11 might have been committed (held by a majority, and the leader may have replied to a client), so they must be kept; slot 12 could not have been committed, so either may be dropped. Rule: if an entry *could* have been committed you must preserve it, because the client may have seen success.
- Methodology: before asking "what does the protocol do with this state?", ask "can this state actually arise?" and construct the crash sequence that produces it.

**Rules of thumb**
- Never let a two-node setup auto-failover without an external arbiter; three nodes (or a third witness) is the minimum for safe automatic failover.
- Reply to a client only after the operation is durably committed on a majority.
- Commands applied through a replicated log must be deterministic.
- Persist anything you need to rejoin safely (log, current term, vote).

**Relevance to Sam's work**
- When using managed systems built on Raft/consensus (etcd, Consul, CockroachDB, Kafka KRaft, many cloud databases), know that writes block during leader election (hundreds of ms to seconds) and that a minority side refuses writes; integration code should retry with timeouts instead of treating that as data loss.
- The "log as source of truth, state as derived" model fits pipelines: keep an append-only log of applied operations (e.g. which supplier records were merged/edited) so any derived dataset can be rebuilt by replay, and checkpoint periodically to avoid replaying from zero on a 16GB machine.
- The "don't reply until committed" rule maps to booking confirmations; the "Start returns an index, apply comes later" split maps to async job queues where the request is accepted with an ID and the result is delivered when durable.
- Unbounded queue growth when consumers are slower than producers (Raft's missing flow control) is the same trap as an async pipeline producer outrunning workers on a small-RAM machine: bound queues and apply back-pressure.

## Robert Morris (MIT 6.824) — Lecture 7: Fault Tolerance: Raft (2) (https://www.youtube.com/watch?v=4r8Mz3MMivY)
**How a Raft leader repairs divergent logs, who is allowed to become leader, what must be fsynced and why that dominates performance, how snapshots bound log growth, and linearizability as the precise test of "correct".**

**Mechanisms: log repair**
- Every AppendEntries carries (prevLogIndex, prevLogTerm) for the slot before the new entries. A follower accepts only if its own entry at prevLogIndex has that term; otherwise it rejects. This consistency check prevents a follower from silently grafting new entries onto a different history.
- Leader keeps nextIndex[follower], initialised to the end of its own log after election. On rejection it decrements and resends with all entries from there onward. When a follower accepts, it deletes everything after the match point and replaces it with the leader's entries. The net effect: find the last point where logs agree, then overwrite the follower's suffix with the leader's.
- Discarding a follower's entries is safe because they were never on a majority, so no leader could have committed/executed them or replied to a client; the client will time out and resend.
- Fast backup (one RPC per conflicting *term* rather than per entry): on rejection the follower returns XTerm (term of conflicting entry), XIndex (first index of that term in its log), XLen (its log length). Leader: if it has no entries of XTerm, jump nextIndex to XIndex; if it does, jump to its own last entry of XTerm; if follower's log is just too short, jump to XLen. Needed because a follower down for a long time, or a stale leader in a minority partition piling up uncommitted entries, can be thousands of entries behind. (Binary search and other schemes also work.)

**Election restriction (who may lead)**
- Straw man "longest log wins" is wrong: a server with entries from terms 5,6,7 (never replicated) can be longer than two servers holding a committed term-8 entry; electing it would overwrite committed data.
- Actual rule (Raft §5.4.1): vote for a candidate only if its last log entry has a higher term than yours, or the same last term and a log at least as long. This favours candidates that heard from the most recent leader. Combined with majority overlap, the winner always holds every committed entry.
- Term numbers persist and propagate through votes, so after a crash the next term is guaranteed to be higher than any used before (e.g. 8, not 6), even if the logs alone would suggest otherwise.

**Persistence**
- Persistent state: log, currentTerm, votedFor. Everything else (commitIndex, lastApplied, nextIndex, matchIndex) is volatile and reconstructed by the leader through AppendEntries exchanges.
- Why persistence at all, when you can replace a dead server with a fresh one: because a cluster-wide power failure kills all of them at once, and you can't rebuild from nothing.
- votedFor must persist or a server could vote twice in one term after a reboot -> two leaders. currentTerm must persist or the cluster could reuse an old term number -> two different "term 6"s.
- On reboot, verify that persisted state is valid (not zeros/corruption) before rejoining.
- Ordering: a leader must persist a new entry before sending AppendEntries; a follower must persist before acknowledging, because the ack is a promise.
- `write()` does not mean durable; only `fsync()` guarantees the data is on the media. fsync is expensive.
- Cost numbers: spinning disk ~10 ms per synchronous write -> max ~100 ops/s if you fsync per operation. SSD ~0.1 ms (100x better). Battery-backed DRAM: millions/s. This synchronous-durability cost is the performance limiter in Raft, file systems and databases alike.
- Mitigation: batch. Accept many client requests, persist them in one fsync, then send AppendEntries. You can also defer persistence until just before you communicate with the outside world (send an RPC or reply).

**Snapshots / log compaction**
- Without compaction the log grows forever (millions of entries), consuming disk and making restart replay take hours.
- Application state is usually far smaller than the log (many writes to X collapse into one table entry). When the log exceeds a threshold, Raft asks the application for a snapshot as of a specific log index, persists (snapshot, index) plus the log suffix, and discards the prefix.
- On restart, Raft hands the latest snapshot to the application, which must be able to rebuild its in-memory state from it, then replays the remaining log.
- Problem: a lagging follower may need entries the leader already discarded. Option A: never discard past the slowest follower (but one follower down for a week blocks compaction forever). Raft's choice: InstallSnapshot RPC, sending the whole snapshot then resuming AppendEntries.
- Snapshot contents belong to the application; Raft only tracks the index. This breaks modularity somewhat (Raft and app must cooperate).
- RPCs can arrive late, duplicated or out of order; a follower should ignore an InstallSnapshot that is older than what it already has.

**Linearizability (definition of correct)**
- A history (client-observed requests with start and end times) is linearizable if there exists a total order of operations that (1) respects real time: if A finished before B started, A precedes B; and (2) every read returns the value of the most recent preceding write in that order.
- It is defined purely from outside observation, independent of implementation. Test: draw constraint arrows (real-time precedence, write-before-read-that-saw-it, read-before-next-overwrite); if they form a cycle, the history is not linearizable. Example: W(x=1) then W(x=2); a read returning 2 finishes before another read starts that returns 1 -> cycle -> violation (stale read).
- Concurrent (overlapping) operations may be ordered either way.

**Rules of thumb**
- When in doubt whether an entry was committed, assume it was if it might have been visible to a client.
- Persist only what you can't reconstruct, but fsync it before you promise anything.
- Batch durable writes; the fsync, not the CPU, is usually the bottleneck.
- Snapshot + suffix log is the standard recovery pair; make the application able to both emit and load a snapshot.
- Design catch-up paths for followers/consumers that fall arbitrarily far behind.

**Failure modes & gotchas**
- Mistaking `write()` for durability.
- Reusing a random election timeout or forgetting votedFor across restarts.
- Correct-but-too-slow implementations (one-entry-at-a-time backup) can fail real-time requirements.
- Stale, reordered RPCs (including old snapshots) must be detected and ignored.

**Relevance to Sam's work**
- Long-running pipelines over millions of rows: use the snapshot + log pattern. Periodically write a compact checkpoint (e.g. Parquet of current state + "processed through row/chunk N") and an append-only log of later changes, so a crash or OOM on the 16GB laptop resumes from the checkpoint, not from zero.
- When writing checkpoints, write to a temp file, fsync, then atomic rename; otherwise a crash can leave a half-written "valid-looking" file.
- Batch database writes (bulk inserts, one transaction per few thousand rows) for the same reason Raft batches fsyncs: per-row commits are bounded by sync latency.
- Use linearizability as the bar for booking/inventory state ("after I confirm, every subsequent read shows it"); use the cycle test to judge whether a cache or read replica can return stale availability or prices.
- Supplier sync jobs that fall far behind should have a "full resync / snapshot" path, not only incremental deltas (the InstallSnapshot idea).

## Robert Morris (MIT 6.824) — Lecture 8: ZooKeeper (and linearizability, continued) (https://www.youtube.com/watch?v=pbmyrNjzdDk)
**Adding replicas to a leader-based replicated system makes it slower, not faster; ZooKeeper buys read scalability by giving up linearizable reads, then restores usable reasoning with linearizable writes, per-client FIFO order, zxids, sync and watches.**

**Linearizability, more practice**
- Two ways to judge a history: produce a total order that respects real time and read-after-latest-write (proof it's linearizable), or find a cycle in the "must precede" graph (proof it isn't). Concurrent (overlapping) writes may be ordered either way, so "W(1) started first" does not mean 1 was applied first.
- Single global order: all clients must observe writes in the same order. If client A reads 2 then 1 while client B reads 1 then 2, it is not linearizable; this is the symptom of two replicas applying writes in different orders.
- No stale reads: W(x=1) completes, then W(x=2) completes, then a read returns 1 -> violation. This is why a lagging replica may not serve reads in a linearizable system.
- Retries and duplicate tables: servers keep a table keyed by unique client request ID storing the result, so a resent request is not re-executed and gets the original reply. Is returning an old cached reply OK? Yes, if you define the request's interval from first send to final receipt: the read was concurrent with later writes, so the older value is legal. Applications must not assume the value was current *at the moment the reply arrived*.
- Linearizability is a property of observed histories, not a design recipe; easy to have with one single-threaded server, very easy to violate in anything distributed.

**Performance problem with leader-based replication**
- In Raft/ZAB every request goes through the leader, which sends every entry to every follower. More servers = more leader work. Throughput falls as you add replicas. Commutative operations (different keys) don't help inside this framework (but sharding in other systems exploits it).
- Most real workloads are read-heavy. Sending reads to any replica would scale reads linearly, but a follower may be stale: it may not have been in the majority, may not have seen the commit, or may be partitioned from the leader without knowing.

**ZooKeeper's consistency contract (paper §2.3)**
- Writes are linearizable: all state-modifying ops behave as if executed one at a time in an order respecting real time.
- FIFO client order: each client's operations take effect in the order the client issued them, including asynchronous (pipelined) writes. Reads by a client observe log positions that never go backward, even when the client switches replicas, and a client's read observes that client's own preceding writes (read-your-writes).
- Mechanism: every log entry has a zxid. Replicas return the zxid with each read; the client remembers the highest zxid seen and sends it with later requests; a replica behind that point must wait (or refuse) until it catches up.
- Reads may be stale with respect to *other* clients' writes. That is the price of read scale-out (ZooKeeper's read throughput grows with replica count).
- `sync`: a pseudo-write that goes through the leader; a read issued after it must reflect state at least as new as the sync's log position. Gives fresh reads at the cost of a leader round trip; use only when needed.

**Pattern: atomic multi-file config update with a "ready" file**
- Writer (master): delete `ready`; write config files f1, f2, ...; create `ready`. Linearizable writes + FIFO order guarantee replicas apply them in exactly this order.
- Reader: check `ready` exists, then read f1, f2. Because reads march forward in the log, seeing `ready` means subsequent reads see the full new config.
- Trap: reader sees old `ready`, reads f1 (old), meanwhile writer deletes `ready` and rewrites, reader reads f2 (new) -> a torn mix. Fix: `exists(ready, watch=true)`. The replica guarantees the watch notification (ready deleted) is delivered *before* any read result reflecting a log position after the delete. So the reader learns to discard and retry before it can act on mixed data.
- Watches live in the replica's memory; if that replica crashes, the client receives a notification of the failure and must re-establish all watches and re-read.

**Decisions & trade-offs**
- Linearizable reads (all reads via leader, or leases) vs scalable reads (any replica, possibly stale). Choose per operation; ZooKeeper defaults to scalable and offers `sync` for freshness.
- A general-purpose coordination service (ZooKeeper) vs embedding a consensus library (Raft) in every app: the service packages the hard part once, at the cost of fitting your problem into its API.
- Replication for fault tolerance costs money (7 servers = 7x) and doesn't automatically yield 7x throughput.

**Rules of thumb**
- Tag every client request with a unique ID; servers dedupe by ID and replay the stored reply.
- Readers of multi-part state need a version marker plus a change notification, or they will read torn state.
- Carry a monotonic "last seen version" token on requests to get read-your-writes and monotonic reads across replicas.
- If you need the absolute latest value, route through the leader/primary or force a sync; otherwise assume reads may be stale.

**Failure modes & gotchas**
- Read replicas silently serving stale data (lag, missed commits, partition).
- Torn reads of configuration split across many keys.
- Lost watches after a replica failure.
- Assuming a delayed reply reflects the state at reply time.

**Relevance to Sam's work**
- Read replicas / caches in front of a bookings DB: fine for browsing inventory and content, not for "is this slot still available" or payment state; route those to the primary or re-check at commit time.
- Read-your-writes for users: after a customer books, their next page load must show the booking even if served from a replica; pass a version/timestamp token or pin that user's reads to the primary briefly.
- Supplier API retries: keep a dedup table of (idempotency key -> stored response) so a retried booking request returns the original confirmation instead of booking twice.
- Pipeline config/reference data spread across multiple files (e.g. mapping tables loaded by workers): publish atomically, e.g. write a new versioned directory then flip a single "current" pointer/manifest last; readers read the manifest once and use only that version. This is the ready-file pattern without needing watches.

## Robert Morris (MIT 6.824) — Lecture 9: More Replication, CRAQ (ZooKeeper API, mini-transactions, chain replication) (https://www.youtube.com/watch?v=IXHzbCuADt0)
**A coordination service is useful because of its API (exclusive create, versioned conditional writes, ephemeral and sequential nodes, atomic watches), which enables mini-transactions, locks and leader election; chain replication is a simpler, cheaper-per-leader replication topology that must be paired with a consensus-backed configuration manager.**
(note: despite the title, this lecture only introduces chain replication; the CRAQ-specific read-from-any-replica mechanism is mentioned as the goal but not explained in this video.)

**What ZooKeeper is for**
- A fault-tolerant, partition-safe implementation of the test-and-set arbiter VMware FT needed.
- Publishing config (e.g. current master's IP), electing a master, storing a master's small critical state so a successor can pick it up, worker registration, work queues for MapReduce-style jobs. One shared ZK cluster (5 or 7 nodes) typically serves many services in a data centre.
- Holds data in memory: great for ~100 MB of coordination data, terrible for 100 GB of application data.

**API (mechanisms)**
- Hierarchical names (znodes) purely as a namespace so unrelated apps don't collide; not a mountable filesystem.
- Znode types: regular (persistent); ephemeral (deleted automatically when the creating client's session stops heartbeating); sequential (ZK appends a monotonically increasing, never-reused number to the name).
- `create(path, data, flags)` is *exclusive*: exactly one of many concurrent creators gets success.
- `delete(path, version)` and `setData(path, data, version)` are conditional on the znode's current version.
- `exists(path, watch)` / `getData(path, watch)`: the check and the watch registration are atomic relative to the write stream, so no change can slip between them unnoticed.
- `getChildren(dir)` lists entries.

**Why plain put/get isn't enough**
- Counter increment via `x = get(k); put(k, x+1)` loses updates under concurrency (two clients both read 10, both write 11), and ZooKeeper's possibly-stale reads make it worse.

**Mini-transaction pattern (compare-and-set loop)**
- `loop { (x, v) = getData(f); if setData(f, x+1, v) succeeds: break }`. Reads may be stale, but the conditional write goes to the leader and fails if the version moved, so the read-modify-write is atomic once it succeeds. Same pattern gives test-and-set.
- Called mini-transactions: atomic on one item, not general multi-item transactions like a database.
- Under heavy contention (1000 clients), only one wins per round, giving O(n^2) total attempts. Mitigate with randomised exponential backoff (sleep that doubles its random range each failure; same idea as Raft's randomised election timers). Watches don't fix it, because all clients send their writes before any notification arrives.

**Locks**
- Simple lock: `create(lockfile, ephemeral)`; on failure `exists(lockfile, watch=true)`; if it still exists wait for the notification; retry. Correct because creates are serialised and exists+watch is atomic relative to the delete. But suffers the *herd effect*: every release wakes all waiters, all retry create -> O(n^2) RPCs. Non-scalable lock.
- Scalable lock (paper §2.4): `create(prefix, sequential|ephemeral)` -> e.g. f27; list children; if no lower number, you hold the lock; else `exists(next-lower, watch=true)` and wait; on wake, re-list (do not just re-watch), because your predecessor may have died while waiting (ephemeral node removed) rather than released. Each release wakes exactly one waiter: O(1) per handoff.
- Crucial caveat: distributed locks are not like mutexes. If a holder crashes mid-update, its ephemeral lock is released anyway and the next holder sees half-updated data. So either (a) every acquirer must detect and repair a predecessor's partial work (e.g. with the ready-file convention or ordered update steps), or (b) use them as soft locks where redoing work is harmless (MapReduce task claims, since tasks are idempotent), or (c) for leader election.
- In-process thread locks: if a thread dies holding a lock the invariants are broken; the safe move is to crash the program, not continue.

**Chain replication**
- Servers in a chain: head ... tail. Writes go to the head, each node applies and forwards, the tail replies to the client. Reads go to the tail only. Without failures it behaves like a single server, so it is linearizable.
- Simple failure states: a write is either committed (reached the tail, therefore on every node) or got only as far as some prefix of the chain. No Raft-style divergent-log puzzles.
- Recovery: head fails -> next node becomes head (writes that only reached the dead head were never acknowledged). Tail fails -> predecessor becomes tail (it knows everything the tail knew; writes it had become committed, so clients may retry and the head needs duplicate suppression). Middle node fails -> remove it and the predecessor resends recent writes to its new successor.
- vs Raft: the head sends each write once (Raft leader sends to every follower), and reads load the tail, not the head; so it scales further before the leader saturates. But every write waits on every node, so one slow replica (e.g. software install, GC, flaky disk) slows *all* writes; Raft only waits for a majority, tolerating stragglers and distant data centres.
- Chain replication cannot handle partitions on its own: if nodes decided themselves who's dead you'd get split brain. It's always paired with an external *configuration manager* (built on Raft/Paxos/ZooKeeper) that monitors liveness and issues the single authoritative chain membership; nodes must obey it, and if a successor seems dead the head keeps retrying until a new config arrives.

**Common architecture**
- Consensus-backed configuration manager (small, critical, partition-safe) + many data shards, each replicated with whatever scheme is most efficient for the workload (chain, primary-backup, Paxos). The config manager also decides sharding (which keys go to which group). Spanner-style systems use Paxos per shard instead.
- Partial connectivity (A and B both reach the manager but not each other) can still break things; a careful manager verifies members can reach each other.

**Rules of thumb**
- Use conditional (version-checked / compare-and-swap) writes for read-modify-write on shared state; retry with randomised exponential backoff.
- Avoid wake-everyone waiting patterns; queue waiters so each release wakes one.
- Treat a distributed lock as advisory/lease-like; design the protected work to be idempotent or recoverable.
- Separate "who is in charge / membership" (consensus) from "bulk data replication" (cheap scheme).
- Every scheme still needs client request dedup.

**Failure modes & gotchas**
- Lost updates from non-atomic read-modify-write.
- Thundering herd on lock release or retries.
- Lock holder dying mid-update leaves inconsistent data that the next holder trusts.
- Chain replication throughput dragged down by its slowest member.
- Self-appointed failover without an authority -> split brain.

**Relevance to Sam's work**
- Inventory/availability counters, booking caps and credit balances: implement as conditional updates (`UPDATE ... SET n = n-1, version = version+1 WHERE id = ? AND version = ?`, or `WHERE n > 0`), retrying on zero rows affected, never read-then-write in application code.
- Parallel workers over millions of rows (multiprocessing, or several machines): claim work items with an atomic conditional update or `SELECT ... FOR UPDATE SKIP LOCKED`; make tasks idempotent so a crashed worker's chunk can simply be redone, the MapReduce soft-lock idea.
- Retry storms against supplier APIs after an outage are the herd effect; add jittered exponential backoff and cap concurrency.
- For scheduled jobs on more than one host, use a lease/lock with expiry (ephemeral-node analogue) and write job outputs so a half-finished run is detectable (status marker written last).

