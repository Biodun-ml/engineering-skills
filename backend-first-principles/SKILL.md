---
name: backend-first-principles
description: First-principles backend engineering playbook distilled from 127 lectures (Sriniously's "Backend from First Principles", Hussein Nasser, CMU databases with Andy Pavlo, MIT 6.824, Martin Kleppmann's distributed systems course, Arpit Bhayani, ByteByteGo). Use whenever designing, building, reviewing or debugging a backend or data-moving system — APIs, request pipelines, auth, databases and schemas, transactions, caching, queues and background jobs, webhooks, third-party/supplier API integrations, payments and bookings, idempotency, retries, concurrency, rate limits, scaling, replication, consistency, deployment, observability — even if the user doesn't say "backend" or "first principles". Also use for architecture decisions and code reviews of server-side code.
---

# Backend from first principles

This skill turns 127 lectures into working judgment. The SKILL.md holds the
procedure and the principles that come up again and again; `references/` holds detailed,
cited notes for every video (paraphrased, with numbers, mechanisms and teacher corrections).

**How to use it**
1. Follow the design procedure below for any new system or significant change.
2. Run the relevant checklists against the design or code.
3. If a `references/` folder sits next to this file, use it when a topic needs depth (e.g. MVCC bloat, Raft, webhook signing, multipart uploads):
   `references/INDEX.md` lists every video with file and line number, or
   `grep -n -i "keyword" references/*.md`. Each file starts with a cross-cutting principles
   block, then one section per video ending in "Relevance to Sam" (Sam's context: travel-tech
   supplier integrations such as Viator/RateHawk, booking flows, multi-million-row location
   pipelines on a 16 GB laptop). Use those sections only when the task is in that context.
   If it isn't there, work from this file and ask the user for the notes pack
   (`backend-first-principles-full.zip`) when you need a teacher's detailed treatment.
4. Explain the *why* to the user: name the underlying constraint (physics, failure, concurrency,
   durability) rather than citing a rule. When you use a specific idea from the notes, mention
   which teacher it came from so the user can go back to the video.

---

## 1. The design procedure (for anything non-trivial)

1. **Write the workflow first.** Who calls what, in what order, with what data. Gather the
   questions stakeholders will ask. Cut to an MVP. Box diagrams come last. *(Nasser)*
2. **Put numbers on it.** Reads/s, writes/s, payload sizes, data volume, growth, latency
   budget per hop, external quotas. Physics: light in fibre ≈ 200,000 km/s, so a
   cross-world round trip is ~100–200+ ms before any work; DRAM ≪ SSD ≪ network in latency;
   random I/O ≫ sequential. *(Sriniously, Pavlo, Nasser)*
3. **Classify each workload.** Waiting (I/O-bound: APIs, DB, disk) → concurrency (async,
   bounded pools). Computing (CPU-bound) → parallel processes on share-nothing partitions.
   Beyond a modest bounded pool, more threads make I/O-bound work slower (memory, context switches). *(Sriniously #22, Nasser)*
4. **Decide the consistency need per data type, not globally.** Money, inventory, bookings:
   strong, atomic, conditional writes. Feeds, counters, caches, search, analytics: eventual,
   with explicit merge rules. Write down "what does a read return after a write?" before you
   replicate or cache anything. *(Morris, Kleppmann, Pavlo)*
5. **Find the sources of truth and the boundaries.** One authoritative store per fact.
   Everything crossing a boundary (user input, supplier responses, webhooks, files) is
   untrusted and is validated where it enters. *(Sriniously #9, #20)*
6. **Design the failure paths before the happy path.** For every remote call: success,
   failure, and *unknown* (timeout). For every step: what if we crash right here? *(Kleppmann
   1.3/2.1, Morris "most awkward moment to fail")*
7. **Choose the simplest thing that meets the numbers.** One machine, one database, a
   monolith, an index before a cache, a bigger box before Kubernetes. Distribute or add
   microservices/Kafka/sharding only when the measurements force it. Observability is the one
   thing to add from day one. *(Sriniously Scaling, Kleppmann 1.1, Morris L1, Arpit/PayPal)*
8. **Plan for change.** Live data/schema moves in reversible stages: expand → backfill from a
   snapshot → shadow-read and diff → flip reads → flip writes → contract. *(Arpit Stripe/GitHub)*
   Versioned reversible migrations, backward-compatible schemas and
   messages (rolling deploys run two versions at once), feature flags, an exit path from
   vendors. *(Sriniously #12, #30, PaaS)*

---

## 2. Principles that recur across all the teachers

### Failure and uncertainty
- **Timeouts mean "unknown", not "failed".** The work may have happened. For bookings and
  payments, resolve the unknown by querying status with your own reference, then reconcile —
  never blindly retry a non-idempotent call. *(Kleppmann, Morris L12, Nasser timeouts)*
- **Assume at-least-once everywhere; make effects idempotent.** Lost acks, retries, queue
  visibility timeouts and failovers all create duplicates. Exactly-once is an *effect*:
  at-least-once delivery + dedupe on a stable ID (idempotency key, webhook delivery ID,
  request ID) enforced by a unique constraint **in the same transaction as the side effect**
  (work-then-record double-processes; record-then-work skips). When the effect is external
  (a charge, a supplier booking) you can't share a transaction: insert the key as a pending row
  first, pass the same key/reference to the vendor, and resolve "unknown" by status query.
  *(all teachers; Sriniously #14, #29)*
- **The database can't roll back the outside world.** Emails, charges and supplier bookings
  made inside a transaction survive its abort. Record intent first (pending row / outbox in
  the same transaction), act after commit, verify, flip status, and run a reconciliation/
  sweeper job for stuck rows. *(Pavlo #17, Sriniously Object Storage + Webhooks)*
- **Across organisations there is no atomic commit.** 2PC blocks and needs everyone's
  cooperation; use sagas: make one side reversible (cancel/refund), persist each decision
  before acting, drive forward with retries, compensate on failure. *(Kleppmann 7.1, Morris L12)*
- **You can't tell slow from dead.** Timeouts decide liveness, never safety. Leases bound a
  deposed leader; epochs/fencing tokens reject zombie workers that wake up after a pause.
  Distributed locks are not mutexes — protected work must still be idempotent. *(Kleppmann
  6.x, Morris GFS/ZooKeeper)*
- **Retries need limits, exponential backoff, jitter, retryable-vs-not classification, and
  circuit breakers.** Synchronised retries turn a blip into a thundering herd. Honour
  Retry-After on 429/503. *(Arpit PayPal, ByteByteGo, Nasser)*
- **Failure at scale is constant and correlated.** Remove single points of failure, degrade
  (read-only, cached, feature off) rather than go down, chunk work so a crash costs one chunk.
  *(Morris L1/Aurora, Arpit Aurora, Sriniously #16)*

### Correctness under concurrency
- **Check-then-act on shared state is a race** — even in single-threaded async code if an
  `await` sits between check and act. Make it one atomic step: conditional `UPDATE ... WHERE
  version = $v`, `SELECT ... FOR UPDATE` in a short transaction, a unique constraint, a Lua
  script, or SERIALIZABLE + retry. *(Sriniously #22, Pavlo #17–18, Nasser ACID)*
- **Read-modify-write = compare-and-set in a retry loop** with randomised backoff.
  *(Morris ZooKeeper)*
- **Know your isolation level.** Postgres defaults to READ COMMITTED, MySQL to REPEATABLE
  READ; snapshot isolation permits write skew; labels lie (Oracle "SERIALIZABLE" = SI).
  Applications must catch aborts/deadlocks and retry. *(Pavlo #17, #20)*
- **Keep transactions short.** No external API or LLM calls while holding locks. Acquire
  resources in a consistent order. `FOR UPDATE SKIP LOCKED` makes Postgres a safe multi-worker
  job queue. *(Pavlo #18)*
- **Don't order events across machines by wall clock.** Use sequence numbers, versions or
  logical clocks; use the monotonic clock for durations/timeouts/rate limits; UTC only for
  timestamps that leave the machine. *(Kleppmann 3.x–4.1, Morris Spanner/COPS)*
- **Last-writer-wins silently loses data.** Merge per field, keep tombstones, reference by
  stable IDs not positions. *(Kleppmann 7.3/8.1, Morris COPS)*
- **Never trust event order or payloads as current state.** Treat webhooks/notifications as
  signals: re-fetch from the source of truth, or apply only if newer by version. To mirror
  another system, pull its ordered log with a stored cursor. *(Sriniously #29)*

### Durability and data
- **Log before you promise.** Make the record durable (fsync / majority-replicated) before
  replying "done", voting yes, or showing it to anyone. `write()` is not durable. Batch many
  operations per fsync. Check synchronous_commit / fsync settings are really on for money.
  *(Morris L10/L12, Raft; Pavlo #21)*
- **The log is the source of truth; state is derived.** Deterministic processing of an
  ordered, append-only log rebuilds any state; snapshot + log suffix bounds replay; the log
  doubles as CDC/replication/audit. Don't dual-write to two systems. *(Kleppmann 5.3,
  Morris Raft, Pavlo #21)*
- **Shadow-copy then atomic swap** for files and outputs: write new elsewhere, flip one
  pointer/rename last. Multi-part state needs a commit marker. *(Pavlo #21, Morris ZooKeeper)*
- **Push correctness into the schema.** NOT NULL by default, UNIQUE, CHECK, enums, FKs with
  deliberate ON DELETE, DECIMAL / integer minor units for money (never floats), TIMESTAMPTZ.
  App validation is the second layer. *(Sriniously #12, Pavlo #03)*
- **Index for the queries you run.** WHERE/JOIN/ORDER BY of hot queries; composite column
  order matters (prefix rule); Postgres doesn't auto-index FK columns (MySQL/InnoDB does); each index taxes every write; keep
  predicates sargable (no functions on indexed columns); confirm with `EXPLAIN ANALYZE`.
  Random UUIDv4 keys scatter B-tree inserts (worst on clustered InnoDB) — prefer time-ordered
  IDs (UUIDv7/ULID) or an internal sequential key. *(Sriniously #12/#20, Pavlo #08/#15, Nasser)*
- **MVCC readers don't block writers, but long/idle transactions pin garbage** → Postgres
  bloat. Prefer snapshots over locks for consistent exports, but in Postgres a long snapshot also holds back VACUUM: run big exports on a replica or in keyset-paginated chunks, and set `idle_in_transaction_session_timeout`. *(Pavlo #20, Kleppmann 8.2)*
- **Durability is not backup.** Replication protects from hardware, not from your own bad
  delete. Versioning + separately credentialed copies. *(Sriniously Object Storage)*

### Performance and scale
- **Measure before fixing.** Timing logs, traces, `EXPLAIN ANALYZE`, profilers. Report p50/
  p95/p99 together; set timeouts/SLOs from the tail; run at 60–80% utilisation because latency
  explodes near 100%. *(Sriniously Scaling, Nasser percentiles)*
- **Database fix order:** remove N+1 → add evidence-based indexes → pool connections →
  then cache. *(Sriniously Scaling)*
- **Connections are expensive, stateful resources.** Pool and cap them (total across all
  instances comfortably < `max_connections`, leaving headroom for admin/migrations), reuse with keep-alive/HTTP2, always release in `finally`.
  "Works for N requests then hangs" = leaked pooled resource. *(Nasser, Sriniously Vercel 3)*
- **Respond fast, finish async.** Persist + enqueue, return an id/pending, let workers do
  slow parts with backoff; workers ack, have visibility timeouts, small chunked tasks.
  *(Sriniously #14, Scaling; Nasser)*
- **Caching is an invalidation problem.** Cache-aside by default: on write, update DB then
  *delete* the key (don't set from writers). TTL + eviction always; scan-resistant policies
  for bulk reads; single-flight/leases against stampedes; a high-hit-rate cache is
  load-bearing. Cache supplier responses to protect quotas, but never data that must be live
  at decision time (price/availability at booking). *(Morris Memcache, Sriniously #13, Pavlo #04)*
- **Batch and stream.** Batching cuts calls by an order of magnitude; stream large bodies
  instead of buffering (20 × 100 MB uploads ≈ 2 GB RAM → OOM); never relay big bytes through
  the app server (presigned URLs/CDN); multipart, parallel, resumable transfers.
  *(Arpit, Sriniously Object Storage)*
- **Replicas give read scale and stale reads.** Route a user's reads after their write to the
  primary (read-your-writes). Partition on the key queries filter/join by; consistent hashing
  moves ~K/N keys; split hot keys. *(Nasser, Sriniously Scaling, ByteByteGo, Pavlo #23)*
- **Count I/Os, read only what you need.** Sequential beats random; columnar/Parquet +
  projection/predicate pushdown + zone maps for analytics; manage larger-than-RAM data with a
  bounded working set and explicit chunking. *(Pavlo #03–#06, #13)*

### Operating it
- **Stateless, disposable processes.** Any instance can die anytime without data loss; files
  go to object storage, state to the DB; no sticky sessions. *(Sriniously Twelve-Factor, DevOps)*
- **Graceful shutdown:** on SIGTERM fail readiness, stop intake, drain with a bounded timeout
  under the orchestrator grace period, finish/roll back transactions, close resources in
  reverse order. *(Sriniously #19)*
- **Readiness ≠ liveness.** A liveness probe that touches a slow DB restarts healthy pods in
  waves. *(Sriniously #30)*
- **Config outside code, validated at boot — fail fast.** Secrets in env/secret manager, never
  in git (rotate any that touched it). *(Sriniously #17, Twelve-Factor)*
- **Observability = structured logs + metrics + traces**, with request/trace IDs on every
  line, propagated through context into outbound calls; OpenTelemetry; alert on actionable
  SLO signals. *(Sriniously #18)*
- **Deadlines travel with the request.** Context carries a deadline and cancellation into every
  DB and supplier call; a timeout should actually cancel work, not just stop waiting.
  *(Sriniously #10, Nasser timeouts, Morris L2)*
- **Test the logic, not the plumbing.** Functional core + thin imperative I/O shell; mock your
  own wrappers, not vendor SDKs; size tests by what they touch (process/machine/network); real
  DB via containers; never `sleep` in tests; coverage shows untested lines, mutation testing
  shows weak assertions. *(Sriniously #26)*
- **Small, frequent, reproducible deploys;** immutable artifacts stamped with the commit;
  stop the bleeding (rollback/flag off) before root-causing. *(Sriniously #30, PaaS)*

---

## 3. Checklists

Run the ones that apply; report gaps with the reason they matter.

**API / request pipeline** *(Sriniously #5, #6, #9, #10, #11, #16, #28; Nasser HTTP)*
- [ ] Order: CORS → request ID/logging → authN → authZ → bind/validate/transform → handler →
      service → repository; global error handler last.
- [ ] Handlers own HTTP; services own business logic (no HTTP types); repositories one query,
      one shape.
- [ ] Every input validated (body, query, path, headers *and* third-party responses).
- [ ] Correct methods/status codes (400/401/403/404/409/422/429 vs 5xx); empty list = 200 [].
- [ ] Lists paginated, deterministically sorted, sort columns whitelisted; keyset pagination
      for large/deep lists.
- [ ] Non-idempotent creates (POST bookings/payments) accept an Idempotency-Key.
- [ ] Errors mapped centrally; no stack traces, raw DB errors or user enumeration leaked.
- [ ] OpenAPI spec as the contract; generated clients; contract tests.

**Auth & security** *(Sriniously #8, #20; Nasser TLS)*
- [ ] Authorise at the point of data access: owner-scoped queries, default deny, 404 not 403
      for others' resources; acting user from auth context, never the payload.
- [ ] Sessions (random ≥128-bit, HttpOnly/Secure/SameSite) for users; short-lived JWT +
      refresh only if truly needed; API keys/HMAC for machines; OAuth2/OIDC authorization code
      + PKCE (PKCE is current guidance, not from the teacher).
- [ ] Passwords: Argon2id/bcrypt with tuned cost; generic errors; equalised timing.
- [ ] Parameterised SQL, argument-array exec, escaped HTML — code and data never mixed.
- [ ] Webhooks verified by HMAC over raw bytes, constant-time compare, replay window.
- [ ] Outbound calls to user-supplied URLs: block private ranges, no redirects (SSRF).
- [ ] Layered rate limits (IP, account, global); reserve capacity for critical traffic, shed
      low-priority work first, decide fail-open vs fail-closed. *(Arpit Stripe, Kleppmann file)*
- [ ] TLS 1.3; never disable certificate checks.

**Database & transactions** *(Sriniously #12; Pavlo; Nasser)*
- [ ] Constraints carry invariants; money is DECIMAL/minor units; times are TIMESTAMPTZ.
- [ ] Migrations versioned, reversible, backward compatible with the running version.
- [ ] Every read-modify-write is atomic (conditional update / FOR UPDATE / CAS).
- [ ] Isolation level known and chosen; aborts and deadlocks caught and retried.
- [ ] No external calls inside transactions; transactions short.
- [ ] Indexes match hot queries; EXPLAIN checked; no N+1.
- [ ] Pool sized against max_connections across all instances; leaks impossible (`finally`).

**External APIs, suppliers, payments, bookings** *(Kleppmann, Morris L12, Arpit Stripe/PayPal,
Sriniously #14/#29, Pavlo #17/#24)*
- [ ] Wrapped behind your own interface; tests mock the wrapper; sandbox/contract tests check
      the real vendor.
- [ ] Every call has connect + read timeouts; the timeout branch is "unknown" → status query /
      reconciliation, not "failed".
- [ ] Idempotency keys sent where the vendor supports them; your own dedupe table otherwise.
- [ ] Intent recorded (pending row/outbox) before the call; result verified; sweeper for stuck.
- [ ] Saga with compensations (cancel/refund) for multi-party flows; never 2PC across orgs.
- [ ] Per-supplier token bucket + concurrency cap; bulk syncs yield to live bookings.
- [ ] Backoff + jitter + retry budget + circuit breaker; 429/Retry-After honoured.
- [ ] Cache supplier content with TTL; re-check price/availability live at booking time.
- [ ] Inbound webhooks: ack fast, enqueue, dedupe by delivery ID, re-fetch current state.

**Async jobs & pipelines (incl. limited-RAM machines)** *(Sriniously #14, #22, Object Storage;
Morris MapReduce; ByteByteGo; Arpit Aurora/Bitcask; Pavlo)*
- [ ] Jobs idempotent and re-entrant; small, chained/fanned-out units; failures retry one unit.
- [ ] Ack after success; visibility timeout > real runtime; after N failed attempts (poison or retries exhausted) move to a DLQ and alert on its depth — never DLQ on a first transient error.
- [ ] Bounded queues and bounded worker pools (no thread per item); backpressure.
- [ ] Big data processed in chunks that fit a fixed memory budget; spill/partition to disk;
      checkpoint progress durably so a crash resumes, not restarts.
- [ ] Outputs written to a temp location then atomically renamed; deterministic transforms so
      any output can be rebuilt from inputs.
- [ ] Columnar formats (Parquet) + pushdown for analytical reads; read only needed columns.
- [ ] Concurrency for I/O (async/thread pool), processes for CPU (mind the Python GIL unless on a free-threaded build).

**Caching** *(Sriniously #13, Scaling; Morris Memcache; Pavlo #04)*
- [ ] Clear freshness requirement per key; TTL + eviction; invalidate by delete-after-write.
- [ ] Stampede protection (single-flight/lease/jittered TTL); cold-start plan.
- [ ] Hit rate monitored; behaviour when the cache is down is acceptable.

**Reliability & operations** *(Sriniously #16–#19, #30; Nasser timeouts)*
- [ ] Timeouts set explicitly at every layer (connect, read, queue-wait, processing, pool).
- [ ] Graceful shutdown; readiness vs liveness; health endpoint returns the commit hash.
- [ ] Config validated at boot; secrets not in code.
- [ ] Structured logs with request IDs; metrics with p50/p95/p99; traces across services.
- [ ] SLOs and error budget defined; alerts are actionable.
- [ ] Backups/versioning tested by restoring.

**Distributed/replicated systems** *(Kleppmann, Morris, Pavlo #23–24)*
- [ ] Is distribution actually required? (One machine is simpler and often enough.)
- [ ] Consistency model written down per data type; linearizable path for money.
- [ ] Majority quorums (2F+1) counted out of all nodes; no two-node "failover".
- [ ] R + W > N gives read-after-write, but not linearizability without read repair.
- [ ] Write the referent before the reference (object, then pointer) so readers never see a
      dangling pointer; reply only after the write is durable/replicated. *(Morris COPS, Raft)*
- [ ] Leases/epochs/fencing against split brain and zombies.
- [ ] Use an existing consensus system (etcd/ZooKeeper/managed DB); never hand-roll Raft.
- [ ] Control plane (membership/config) separated from data plane.

---

## 4. Failure modes to hunt for in reviews

- Retrying a timed-out POST that creates a booking/charge → duplicates.
- Sending an email/charge inside a DB transaction that later aborts.
- `if balance >= x: balance -= x` across two statements or an `await`.
- Webhook handler that trusts payload order, or processes before dedupe.
- Cache set by writers (racing sets → permanent staleness); no TTL; cached prices used at checkout.
- Read replica used right after a write → user doesn't see their change.
- Connection/pool leak on an error path; pool size × instances > DB limit.
- Unbounded `asyncio.gather` / thread-per-item against a rate-limited API.
- Loading a whole file/table into memory; `SELECT *` for analytics.
- Wall-clock timestamps used to order events from different machines.
- Liveness probe that depends on the database.
- Floats for money; naive timestamps; OFFSET pagination on huge tables.
- Microservices/Kafka/Kubernetes introduced before any measurement demanded them.

---

## 5. Where to look in `references/`

| Need | File (see INDEX.md for line numbers) |
|---|---|
| Routing, auth, request layers, caching, security, testing, logging, OpenAPI | `sriniously-a-*` |
| HTTP, Postgres, concurrency, task queues, object storage, config, serverless, PaaS | `sriniously-b-*` |
| Scaling/performance, Twelve-Factor, CDN/edge, real-time (SSE/WebSockets), search, serialization, API pitfalls | `sriniously-c-*` |
| DevOps, REST design, error handling, webhooks, validation, graceful shutdown, object storage pt 1 | `sriniously-d-*` |
| Storage engines, buffer pools, LSM, columnar/Parquet, B+trees, query execution & optimizers | `pavlo-cmu-a-*` |
| Transactions, 2PL, MVCC, WAL, crash recovery (ARIES), distributed databases, 2PC/Raft | `pavlo-cmu-b-*` |
| MapReduce, RPC & threads, Aurora, distributed transactions, Spanner, Memcache at Facebook, causal consistency | `morris-mit-a-*` |
| GFS, primary-backup, Raft, ZooKeeper, linearizability, chain replication | `morris-mit-b-*` |
| Clocks, broadcast, replication, quorums, consensus, 2PC, CRDTs; Kafka exactly-once, Stripe rate limiter/migrations, PayPal herd, GitHub sharding, Bitcask, load balancers | `kleppmann-distsys-and-arpit-case-studies.md` |
| OSI/TCP/UDP/TLS/HTTP versions, proxies, timeouts, ACID, keys & indexes, pub/sub, sync vs async, percentiles; system design cases (rate limiter, chat, crawler, URL shortener, KV store, YouTube, Kafka) | `nasser-protocols-db-fundamentals-and-bytebytego-system-design.md` |

Lines marked "(note: …)" in the references are corrections or caveats to what a teacher said.
Prefer the corrected version.
