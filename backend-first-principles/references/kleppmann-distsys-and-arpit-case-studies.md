# kleppmann-arpit — cross-cutting principles

1. **Every remote call has three outcomes: success, failure, and unknown.** A timeout means "maybe it happened". Design the unknown branch explicitly: query by your own reference, reconcile, or retry safely. Never treat a timeout as a definite failure for bookings or payments. (Kleppmann 1.3 RPC, 2.1 Two generals, 2.4 Fault tolerance)
2. **Exactly-once is an effect, not a delivery:** at-least-once retries + idempotent operations or durable dedup by (client ID, sequence/idempotency key). Model state as sets or "set to X" rather than increments/decrements. A retry must be recognisable as the *same* request. (Kleppmann 5.1 Replication; Arpit, Kafka exactly-once)
3. **Across companies you cannot get atomic certainty; build compensations and reconciliation.** Make one side reversible (refund, cancel), persist your decision before acting on it, then drive it to completion with retries. Real 2PC blocks if the coordinator dies; external partners use sagas plus reconciliation jobs. (Kleppmann 2.1, 7.1 Two-phase commit; Arpit, Kafka transactions)
4. **Fence zombies with epochs/terms.** Any worker, leader or lock holder can pause (GC, swapping, network) and wake up believing it still owns the job. Give logical owners a stable ID and a monotonically increasing epoch; reject writes with stale epochs; always defer to the higher term. (Kleppmann 6.1 Consensus, 6.2 Raft; Arpit, Kafka transactional.id)
5. **Timeouts decide liveness, never safety.** You cannot tell a slow node from a dead one (FLP; eventually-perfect failure detectors). Correctness must not depend on bounded latency or on clocks agreeing. (Kleppmann 2.3 System models, 2.4, 6.1)
6. **Use the right clock.** Monotonic clock for durations, timeouts, backoff and rate limits; UTC time-of-day only for timestamps that leave the machine; NTP can step or panic, and leap seconds have taken down fleets. Never order causally related events from different machines by wall-clock time; use version numbers, sequence numbers, or logical clocks (Lamport for total order, vector clocks to detect concurrency). (Kleppmann 3.1, 3.2, 3.3, 4.1; Spanner 8.2)
7. **Retries need exponential backoff + jitter + caps**, plus classification of retryable vs non-retryable errors; synchronized retries turn overload into cascading failure, and a DLQ is for genuine poison messages only. (Arpit, PayPal thundering herd; Arpit, Stripe rate limiter)
8. **Protect capacity with layered limits and prioritised shedding:** per-tenant token bucket, separate concurrency caps for expensive operations, reserve capacity for critical traffic, shed low-priority work first, fail open if the limiter breaks. As an API client, mirror this: per-supplier rate + concurrency limits, and pause bulk syncs before bookings. (Arpit, Stripe rate limiter)
9. **Batch whenever you can, and prefer pull when fan-out explodes.** Batching cuts calls by an order of magnitude at the cost of a bounded delay; per-node push queues multiply cost (M × N). (Arpit, Atlassian SNS/SQS cache invalidation; PayPal)
10. **Pick consistency per data type.** Linearizable conditional writes (single authoritative row with compare-and-swap, or consensus) for money, inventory and bookings; eventual consistency with explicit merge rules for caches, search, content and offline clients. Quorums (w + r > n) give read-after-write but are not linearizable without read repair; strong consistency costs round trips, a leader bottleneck, and availability under partitions (CAP). (Kleppmann 5.2, 7.2, 7.3)
11. **Last-writer-wins silently loses data.** Merge at field level, keep tombstones with timestamps for deletes so missed deletes don't resurrect records, and use vector clocks or CRDTs when concurrent edits must all survive. Reference items by stable unique IDs, never by shifting positions or row order. (Kleppmann 5.1, 7.3, 8.1)
12. **Deterministic processing of an ordered log is a superpower.** Same inputs in the same order give the same state (state machine replication). Keep append-only input logs and deterministic transforms so any output can be rebuilt; when order isn't guaranteed, make updates commutative and idempotent. (Kleppmann 5.3, 4.2/4.3 broadcast; Arpit, Bitcask)
13. **Append-only + checksummed records + an in-memory index of key → offset** gives fast, crash-safe storage; a torn tail record is detectable and truncatable. The limit is RAM for keys, so estimate key-index size before you rely on it on a 16GB machine. (Arpit, Bitcask)
14. **Migrate live data in reversible, verified stages:** build new alongside, backfill in parallel from a snapshot (not the live primary), shadow-read and diff, flip reads, flip writes while keeping the old store updated for rollback, cut over only when replication lag is tiny, delete old last. Enforce logical domain boundaries in code (lint cross-domain joins) before moving anything physically. (Arpit, Stripe migrations; GitHub sharding)
15. **Design for constant failure and correlated faults.** At fleet scale something is always broken; plan for "a whole zone + one more node" (Aurora 6 copies, write 4/read 3); degrade to read-only rather than go down; remove single points of failure; beware the same bug or event hitting every node at once (2012 leap second). (Arpit, Aurora §2; Kleppmann 2.4, 3.1, 5.2)
16. **Small fixed-size chunks shrink blast radius and repair time** and enable parallel recovery and load rebalancing (Aurora's 10 GB segments repaired in ~10 s). The same logic applies to pipelines: chunk work so a crash costs one chunk, and checkpoint progress durably. (Arpit, Aurora §2; Kleppmann 2.3 crash-recovery)
17. **Keep compute stateless and push work towards the data.** Stateless workers can be killed or replaced; ship small deltas/logs, not whole objects; any synchronous fan-out is as slow as its slowest leg. (Arpit, Aurora §1 storage-compute separation)
18. **Nothing is magic: understand the internals of the parts you lean on.** Load balancers take minutes to scale (shift traffic gradually, pre-warm); consistent hashing is a sorted-array lookup that answers ownership but moves no data; LRU caches get wiped by bulk scans unless they are scan-resistant (InnoDB midpoint insertion). (Arpit, Atlassian LB outage; consistent hashing; MySQL midpoint insertion)
19. **Keep it simple and stay on one machine if you can.** Distribution opens a Pandora's box; delete microservices that only add a hop; shard only when forced, and then by clear ownership boundaries. (Kleppmann 1.1; Arpit, PayPal; GitHub sharding)
20. **Long reads should use snapshots, not locks.** MVCC gives consistent, lock-free reads for exports and backups; causality can flow through humans, so pass versions/tokens explicitly when order matters across hops. (Kleppmann 8.2 Spanner)

---

## Martin Kleppmann — Distributed Systems 1.1: Introduction (https://www.youtube.com/watch?v=UEAMfLPZZhE)
**A distributed system is many computers cooperating over an unreliable network with no shared memory; only go distributed when you must.**
- Mental model: concurrency inside one process has a shared address space (pass a pointer, the other thread can follow it). Across machines there is no shared memory, so a pointer/reference is meaningless to the receiver; all sharing must happen by sending serialised messages over a network.
- Lamport's joke definition captures the core pain: a machine you never knew existed can fail and break yours. Kleppmann's working definition: multiple devices (phones, servers, cars) communicating over some network to jointly achieve a task; the field studies how to coordinate them.
- Four reasons to distribute:
  1. Inherent distribution (messaging between two phones cannot be single-machine).
  2. Reliability: if one node fails or reboots, others take over its work.
  3. Performance/latency: intercontinental round trips cost ~100 ms from speed of light alone, so put data near users.
  4. Scale: problems too large for any single machine (CERN-style million-core analysis).
- Costs: the network may drop, delay or be interfered with (weak Wi-Fi, unplugged cable, overload, DoS); processes may crash; both happen non-deterministically at any moment. The goal is **fault tolerance**: the system keeps serving even when parts are faulty.
- Rule of thumb (strongly emphasised): if a problem fits on one computer, keep it on one computer. Distribution opens a "Pandora's box"; open it only when one of the four reasons forces you to.
- Relevance for Sam: on a 16GB laptop, the right answer for many pipelines is a single well-tuned process (chunking, parallelism within the box) rather than a distributed setup. Third-party API integrations, however, are unavoidably distributed systems: every supplier call (Viator, RateHawk) is a message over an unreliable network to a machine you do not control, so all the failure reasoning in later lectures applies.
- Further reading named: van Steen & Tanenbaum (applied), Cachin/Guerraoui/Rodrigues (theory), Kleppmann's *Designing Data-Intensive Applications* (distributed databases for industry engineers), Bacon & Harris (OS link).

## Martin Kleppmann — Distributed Systems 1.2: Computer networking (https://www.youtube.com/watch?v=1F3DEq8ML1U)
**The only primitive in distributed systems is "node A sends a message to node B"; channels differ only in latency and bandwidth.**
- Abstraction: every participant is a *node*; every interaction reduces to sending a message. Wi-Fi, cellular, undersea fibre, satellite are all just message channels.
- "Hard drives in a van" is a legitimate network: AWS-style shipping of a ~50 TB box takes a day or two (huge latency) but works out to roughly 1 Gbit/s effective bandwidth; send more boxes for more bandwidth. Lesson: for bulk data moves, latency and bandwidth are separate axes, and physical transfer can beat the internet.
- Orders of magnitude to remember:
  - Latency: same data centre / home LAN ~1 ms or less; intercontinental ~100 ms; van ~1–2 days.
  - Bandwidth: poor 3G ~100 kbit/s, good 3G ~1 Mbit/s, home broadband ~10 Mbit/s (varies), van ~1 Gbit/s.
- The web as a distributed system: browser (client) sends a GET request carrying the URL path to the server named in the URL; server responds with HTML/image/PDF etc. Two messages: request and response. Headers carry client version (User-Agent), accepted formats (Accept), compression, language.
- Layering: the conceptual HTTP request/response sits on TCP, which splits messages into packets (max ~1500 bytes on typical networks), handles back-and-forth, and reassembles on the receiver; TLS adds encryption. In distributed-systems reasoning a "message" is a logical unit that may span many packets; leave packetisation to the network layer.
- Practical tools shown: an HTTP-level proxy capture (Charles) to see request/response messages; a packet capture (Wireshark) to see individual TCP packets. Useful when debugging supplier API integrations: inspect what was actually sent and received rather than trusting the client library.
- Relevance: when pulling large datasets (GeoNames/OSM dumps), think in bandwidth terms; when calling supplier APIs, think in latency terms (each round trip to a far-away supplier region costs ~100 ms, so batch requests and run them concurrently).

## Martin Kleppmann — Distributed Systems 1.3: RPC (Remote Procedure Call) (https://www.youtube.com/watch?v=S2osKiqQG9s)
**A remote call can be made to look like a local function, but it is fundamentally different: it can be lost, delayed, or half-executed.**
- Running example: online shop calls a third-party payment service (which in turn talks to the card network and issuing bank, maybe 3-D Secure). In code it looks like `paymentService.processPayment(card, amount, currency)` then "fulfil order if success". The implementation is not in your program; it sits across the internet, owned by another company. That is exactly Trameter's shape with Viator/RateHawk.
- Mechanism: an RPC framework/middleware generates a **stub** with the same signature as the remote function. The stub **marshals** (encodes) arguments into bytes (JSON or a binary format), sends message m1; the RPC server unmarshals, invokes the real function, marshals the return value into m2, which the client unmarshals and returns.
- **Location transparency** is the ideal (remote call indistinguishable from local) and it is a lie. Remote calls have failure modes local calls never have:
  - request lost: retry? but then you may charge the card twice;
  - message delayed but eventually arrives;
  - server crashes mid-execution, after partially doing the work, so no response comes back.
  Kleppmann's verdict: they look similar but are "simply two different things".
- History to recognise: Sun RPC (1980s, used for NFS), CORBA and Java RMI (1990s object middleware, now dead), gRPC (Google, 2015), REST over HTTP (today's default; enthusiasts say it is a different philosophy, but functionally it is still RPC).
- REST/AJAX pattern: build args, `POST` with JSON body and Content-Type header to `/payments`; success handler checks status code then parses JSON; a separate catch handler fires on network error/timeout (e.g. nothing within 30 s). Key point: after a timeout **you do not know whether the request got through**, so the error path must account for "maybe happened".
- Microservices / SOA: big enterprises split systems into services that call each other over the data-centre network via RPC, often across languages (COBOL, C++, Java, newer). Interop uses an **IDL** (e.g. Protocol Buffers in gRPC): language-neutral message types (`PaymentRequest{card, amount int32, currency enum}`) and service definitions (`rpc ProcessPayment(PaymentRequest) returns (PaymentStatus)`), from which client and server stubs are generated in each language.
- Rules of thumb:
  - Treat every external call as having three outcomes: success, definite failure, and **unknown** (timeout). Design the unknown branch explicitly.
  - Do not blindly retry non-idempotent operations (charges, bookings); make them safe to retry first (idempotency keys, dedupe on the server; see later lectures and the Stripe/Kafka videos).
  - Use typed schemas/IDLs at integration boundaries so type mismatches surface at build time, not in production.
- Relevance: supplier booking calls are textbook RPC over unreliable networks; a timed-out "create booking" may have created a booking. Always be able to look up by your own reference afterwards.

## Martin Kleppmann — Distributed Systems 2.1: The two generals problem (https://www.youtube.com/watch?v=MDuWnzVnfpI)
**Over a lossy channel, two parties can never reach certainty that they both will act; real systems survive by making actions reversible or checkable later.**
- Purpose of system models: algorithms are only correct relative to stated assumptions about what can fail. This lecture and the next are thought experiments that motivate those assumptions.
- Setup: two armies must attack together (one alone loses; neither is fine). They communicate only by messengers who may be captured. A sender learns a message arrived only by getting a reply.
- Core ambiguity: general 1 sends "attack 10 Nov" and hears nothing. Either the request was lost (general 2 knows nothing) or the reply was lost (general 2 has committed). These look identical to the sender but are completely different states at the receiver.
- Strategy analysis:
  - G1 always attacks and spams messengers: if all are captured, G1 attacks alone.
  - G1 attacks only on receiving a reply: now G2 is in G1's old position (it only knows G1 attacks if its reply arrives), so G2 needs an ack of the ack, and so on infinitely.
  - Result: **no common knowledge** is achievable with finitely many messages. Retries raise probability, never certainty.
- Applied: shop must dispatch goods iff the payment service charges the card; the RPC between them is the messenger. Strictly impossible to guarantee. Real commerce works because of **second-level safeguards**:
  - Make one side revocable: charging is reversible via refund, so the payment service can charge, and if the shop cannot ship (out of stock), it refunds and apologises.
  - Reconcile after the fact: when the network recovers, the shop asks the payment service "did you actually charge that card?" and acts on the answer.
- Rules of thumb for bookings/payments:
  - Do not seek atomic certainty across two companies; design for compensating actions (refund, cancel booking) and **reconciliation jobs** that query the counterparty's state.
  - Order operations so that the reversible step happens first, or so the irreversible one can be compensated.
  - Every outbound request should carry an ID you can later query by, to resolve "unknown" outcomes.

## Martin Kleppmann — Distributed Systems 2.2: The Byzantine generals problem (https://www.youtube.com/watch?v=LoGx_ldRBU0)
**When some participants may lie, honest nodes can still agree only if fewer than one third are malicious (n >= 3f+1).**
- Setup: n generals, reliable messaging (easier than two generals) but up to f generals are traitors that lie, send contradictory messages, and may collude (harder). Honest generals do not know who is malicious. Goal: all honest generals agree on the plan; nothing can be required of the malicious ones.
- Why it is hard: G3 receives "attack" from G1 and "G1 told me retreat" from G2. Either G2 is lying, or G1 sent contradictory orders and G2 is honest. G3 cannot distinguish the two.
- Result: tolerating f malicious nodes requires at least **3f+1** total. Three generals with one traitor is unsolvable; f=1 needs 4, f=2 needs 7.
- Digital signatures help (G2 can prove what G1 actually said), but do not make the problem easy.
- Practical relevance: shop, payment service and customer must agree on an order; trust is asymmetric. The shop must treat customers as potentially fraudulent; the payment service may not trust the shop (stolen-card storefronts); the shop may trust the payment service. Byzantine thinking is a model for these "parties who do not fully trust each other" settings, though the symmetric version is a simplification.
- Rules of thumb:
  - Inside your own infrastructure you usually assume crash faults, not lies; at boundaries with customers and partners assume input can be malicious and verify (signed webhooks, server-side price checks, never trust client-sent amounts).
  - Do not trust a supplier/partner's claim blindly when money is involved; reconcile against an authoritative source.
- Name origin: "Byzantine" meant excessively complicated/devious long before computing; no historical claim about the empire implied.

## Martin Kleppmann — Distributed Systems 2.3: System models (https://www.youtube.com/watch?v=y8f7ZG_UnGI)
**Every distributed algorithm rests on three explicit assumptions (network, node, timing); if reality is weaker than your assumption, the algorithm's guarantees vanish.**
- Networks fail for mundane and bizarre reasons: buffer overflow under load, operator unplugs the wrong cable, sharks biting subsea fibre (Google filmed it), a cow kinking a fibre lying on the ground.
- **Network models** (point-to-point, bidirectional links):
  - *Reliable*: every sent message is received, nothing fabricated; reordering allowed. Strongest, easiest to program against.
  - *Fair-loss*: each send has non-zero probability of arriving; retry enough and it eventually gets through (no bound on how long).
  - *Arbitrary*: an active adversary may drop, modify, replay, spoof (coffee-shop Wi-Fi owner). Realistic for the public internet.
  - *Network partition*: nodes up, link between groups down for a finite but possibly long period.
- **Upgrading links**:
  - Fair-loss to reliable = **retry until acknowledged + deduplicate at receiver**. Price: unbounded delay.
  - Arbitrary to (almost) fair-loss = TLS (integrity, authenticity, confidentiality). It cannot help if the adversary blocks everything forever.
  - This is the theoretical basis for "retries + idempotent receiver" in every API integration.
- **Node models**:
  - *Crash-stop*: once crashed, gone forever (phone in the toilet, destroyed hardware).
  - *Crash-recovery*: may crash and restart; in-memory state lost, stable storage (disk) survives. Can still die permanently.
  - *Byzantine*: deviates arbitrarily from the algorithm, possibly disguising it.
  - A node is *faulty* (crashed / deviating) or *correct*; other nodes cannot directly tell which.
- **Timing models**:
  - *Synchronous*: bounded message latency and bounded execution speed. Very strong, usually false.
  - *Asynchronous*: no timing assumptions at all (a message may take 20 years; a node may pause anywhere).
  - *Partially synchronous*: mostly synchronous, occasionally asynchronous for unknown periods; you cannot tell which state you are in. The realistic compromise; some problems are unsolvable in pure asynchrony.
- Why synchrony assumptions break in practice:
  - Network: lost message + retry; partition (minutes to days); congestion/queueing; a switch reconfiguration once held packets in buffers for over a minute, even inside a well-run data centre.
  - Nodes: context switches, priority inversion, **stop-the-world GC pauses** (can last minutes with large heaps), **page faults, especially when memory is tight**. Real-time OSes give scheduling guarantees, but general systems do not run on them.
- Rules of thumb:
  - Write down your network/node/timing model before designing a protocol. Assuming crash-recovery when Byzantine nodes exist, or synchrony when the system is partially synchronous even for ~10 s, voids all guarantees.
  - Never let correctness depend on "this call returns within X ms". Timeouts are for liveness (giving up), not for safety (deciding something definitely did not happen).
  - Persist anything you need after a crash; in-memory progress is gone on restart (crash-recovery model). For long pipelines, checkpoint progress to disk so a restart resumes rather than redoes.
- Relevance to a 16GB laptop: memory pressure causes swapping/page faults that freeze a process for seconds; a worker that is "paused" may still hold a lock or lease and resume believing it is valid. Keep chunk sizes well under RAM to avoid swapping, and do not rely on wall-clock deadlines for correctness.

## Martin Kleppmann — Distributed Systems 2.4: Fault tolerance (https://www.youtube.com/watch?v=43TDfUNsM3E)
**High availability comes from tolerating a bounded number of faults with no single point of failure; failure detection by timeout is inherently imprecise.**
- Availability = fraction of time a service works correctly. 99% allows ~3–4 days/year downtime; 99.9% ~9 hours/year; fixed-line telephone network targets five nines (very conservative engineering).
- **SLO** = your internal target, e.g. 99.9% of requests answered correctly within 200 ms, measured over one day. **SLA** = contract with customers about expected service level.
- Fault tolerance is always bounded: "keeps working while fewer than half the nodes are crashed" (1 of 3, 2 of 5). No system tolerates everything failing at once.
- **Single point of failure (SPOF)**: a component whose failure takes down the whole system. Goal: remove any one component and still work.
- **Failure detector** (really a fault detector): usually a timeout on a ping/heartbeat. No reply within T means "suspected dead".
- Why timeouts lie: no reply may be because the request was lost, the reply was lost, either was delayed, the node is in a long GC pause and will answer in a minute, or it really crashed. Indistinguishable to the sender.
- A **perfect** failure detector exists only with synchrony + crash-stop + no Byzantine nodes. In realistic (partially synchronous) systems the best is **eventually perfect**: may wrongly suspect a live node for a while (false positive), may take a while to notice a real crash (false negative), but eventually suspects a node iff it has really crashed. That is enough to build useful algorithms (consensus, later).
- Rules of thumb:
  - Define SLOs in terms of both success rate and latency percentile over a window; track supplier APIs against their SLAs.
  - When a supplier API "times out", classify it as *suspected* failure, not confirmed; route to retry/reconcile, not to "mark booking failed".
  - Choose timeouts as a trade-off: short = more false suspicions (and duplicate work), long = slow detection.
  - Hunt for SPOFs: a single cron box, a single API key, a single DB without backup.

## Martin Kleppmann — Distributed Systems 3.1: Physical time (https://www.youtube.com/watch?v=FQ_2N3AQu0M)
**Physical clocks are imprecise hardware tied to a messy definition of time (UTC with leap seconds); software that assumes clean time breaks.**
- Uses of time: scheduler slices (~10 ms), timeouts, failure detectors, retry back-off, performance measurement/profiling, log timestamps, database event times (when a purchase happened), **expiry** (cache TTLs, DNS TTLs e.g. ~7000 s, TLS certificate validity windows), and ordering events across nodes.
- Physical vs logical clocks: physical clocks count seconds/time of day; logical clocks (next lecture) count events. In distributed systems a "clock" is something you ask for a timestamp.
- **Quartz clocks**: lab-grown crystal cut by laser to resonate at a set frequency (piezoelectric). Errors from manufacturing variation and temperature (stable around 20–25 C, frequency drops quadratically away from that, so hot server rooms drift more). Drift measured in ppm: **1 ppm ≈ 32 s/year**; typical quartz ~20 ppm; rule of thumb < 50 ppm.
- **Atomic clocks**: caesium transition ~9.19 GHz defines the SI second. Expensive but purchasable. **GPS** satellites carry atomic clocks and broadcast time + position; a receiver gets accurate time, but data centres need a roof antenna because of shielding.
- Definitions of time: GMT (astronomical, sun at Greenwich meridian, averaged) vs TAI (international atomic time, ~hundreds of synchronised atomic clocks). Earth's rotation wobbles (tides, earthquakes), so **UTC = TAI corrected by leap seconds** to stay aligned with astronomy. Time zones are offsets from UTC.
- **Leap seconds**: may be inserted or removed, usually 30 June or 31 Dec, decided months ahead by astronomers. Removal: 23:59:58 jumps straight to 00:00:00. Insertion: 23:59:59 then 23:59:60.
- Representations:
  - Unix time: seconds since 1 Jan 1970, which ignores leap seconds (so not truly UTC).
  - ISO 8601: human-readable Y-M-D h:m:s. Converting between them needs the Gregorian leap-year rules (div by 4, not by 100, unless by 400) and, strictly, the 20+ leap seconds since 1970. Most software ignores leap seconds.
- Incident (the puzzle): **30 June 2012** leap second triggered a Linux kernel bug causing livelock (100% CPU, no progress) across many companies simultaneously; rebooting did not help, resetting the system clock did.
- Mitigation: **leap smearing**: spread the extra second over ~24 h (12 h either side) by slightly slowing clocks, so there is no discontinuity. A pragmatic hack because legacy software cannot be fixed.
- Rules of thumb:
  - Store timestamps in UTC; convert to local time only for display (travel bookings span time zones, so keep the zone/offset explicitly with any local datetime like a departure time).
  - Use a monotonic clock for durations/timeouts, never time-of-day.
  - Correlated failures are the dangerous ones: the same bug hitting every node at the same instant defeats replication.
  - Expect clocks on different machines (and supplier servers) to disagree; do not compare timestamps from different hosts for ordering without allowing for skew.

## Martin Kleppmann — Distributed Systems 3.2: Clock synchronisation (https://www.youtube.com/watch?v=mAyW-4LeXZo)
**NTP can only estimate clock skew (assuming symmetric latency), may jump your clock, and may silently give up; measure durations with a monotonic clock.**
- Drift (rate error) produces **skew** (difference between two clocks at the same instant). In asynchronous/partially synchronous networks skew can be reduced but never to zero.
- **NTP** (PTP also exists): clients query servers with accurate sources. **Strata**: stratum 0 = atomic/GPS source, stratum 1 = server attached to it, stratum 2 syncs from stratum 1, etc.
- Statistical tricks: query several servers and discard outliers (one misconfigured server); sample each server repeatedly over minutes to filter random delay (does not fix systematic asymmetry). Good network: skew within a few ms; busy Wi-Fi / loaded internet: much worse.
- **Skew estimate mechanism**: client sends at t1 (client clock); server receives at t2, replies at t3 (server clock); client receives at t4.
  - Round-trip network delay δ = (t4 − t1) − (t3 − t2).
  - One-way latency is unmeasurable without synced clocks, so assume symmetry: response took δ/2.
  - Server clock at t4 ≈ t3 + δ/2; skew θ = t3 + δ/2 − t4 = ((t2 − t1) + (t3 − t4))/2.
- Correction policy:
  - |θ| < 125 ms: **slew** (speed up or slow down by at most 500 ppm, i.e. 0.5 ms/s) to converge over minutes. Example: a client crystal running ~45 ppm fast gets slowed by ~45 ppm.
  - Larger: **step** (jump forwards or backwards).
  - Very large (e.g. ~1000 s): **panic**, refuse to adjust, leave it to an operator. Your clock may then be wrong by half an hour or more, silently.
- Gotcha: `end = currentTimeMillis() − start` can be **negative or hugely inflated** if NTP steps the clock mid-measurement (also leap seconds).
- **Time-of-day vs monotonic clocks**:
  - Time-of-day (Java `currentTimeMillis`, Linux `CLOCK_REALTIME`): relative to the 1970 epoch, comparable across machines if synced, needed for things like certificate expiry checks; can jump.
  - Monotonic (Java `nanoTime`, `CLOCK_MONOTONIC`; Python `time.monotonic()`): arbitrary origin (e.g. boot time), meaningless across machines, but never jumps (slewing still applies). The correct choice for elapsed time, timeouts, rate measurements.
- Rules of thumb:
  - Durations, timeouts, retry back-off, rate limiting windows within a process: monotonic clock.
  - Timestamps that leave the machine or are persisted: time-of-day UTC, and monitor NTP skew if correctness depends on it.
  - Do not assume a supplier's server time matches yours; when an API returns "expires_at", allow a skew margin.

## Martin Kleppmann — Distributed Systems 3.3: Causality and happens-before (https://www.youtube.com/watch?v=OKHIdpOAxto)
**Physical timestamps cannot reliably order cause and effect; the happens-before partial order can.**
- Motivating bug: A posts "moon is made of cheese" (m1) to B and C. B replies "no it isn't" (m2) to A and C. Network delay lets C receive m2 before m1, so C sees a reply before the thing it replies to.
- Why timestamps do not fix it: even with NTP, **clock skew between A and B can exceed the one-way network delay**, so B's reply can carry an earlier timestamp than A's original (t2 < t1). Timestamp order then contradicts causal order.
- **Happens-before (Lamport)** over events (local step, send, receive). a → b iff:
  1. a and b on the same node and a executed first (each node is single-threaded with a total local order; treat each thread as a node if needed);
  2. a is the send and b the receive of the same message (messages assumed unique, e.g. sender ID + sequence number, or a long random ID);
  3. transitivity: a → c and c → b.
- It is a **partial order**: for any a, b exactly one of a → b, b → a, or a ‖ b (**concurrent**). Concurrent means neither knew of the other, not "same instant".
- Visual rule: a → b iff there is a path from a to b moving forward along node timelines and along message arrows. Example: a→f via two messages; a ‖ e because reaching e would require going backwards on node C.
- **Causality**: if a → b, a *may* have caused/influenced b (information could flow). If a ‖ b, neither caused the other. Analogy: light cones in physics; events outside each other's light cones cannot influence each other. In systems, messages play the role of light.
- **Causal order**: any total ordering consistent with happens-before (if a → b, a comes first); concurrent events can be in either order.
- Rules of thumb:
  - Never use wall-clock timestamps from different machines to decide "which came first" for causally related events (e.g. booking created vs. booking cancelled webhooks from a supplier; message replies). Use sequence numbers, version numbers, or explicit "in reply to / previous version" references.
  - Give every message/event a unique ID (node ID + counter or UUID) so duplicates and causality can be tracked.
  - Watch spelling: *causal*, not *casual*.

## Martin Kleppmann — Distributed Systems 4.1: Logical time (https://www.youtube.com/watch?v=x-D8iFU1d-o)
**Logical clocks count events, not seconds: Lamport clocks give a causality-consistent total order; vector clocks additionally detect concurrency.**
- Requirement for any logical clock: if a → b then T(a) < T(b). Logical timestamps are counters, unrelated to elapsed seconds.
- **Lamport clock** (Lamport ~1978):
  - Each node holds integer t = 0. On any local event, t += 1.
  - On send: increment, attach t to the message.
  - On receive of (t', m): t = max(t, t') + 1, then deliver.
  - Example: A's events get 1, 2 (send m1 with 2); B receives: max(0,2)+1 = 3; next event 4 (send m2 with 4); C receives: 4+1 = 5.
  - Properties: a → b implies L(a) < L(b), but **L(a) < L(b) does not imply a → b** (could be concurrent). Different events can share a timestamp.
  - Make unique: pair (L, nodeID); within a node timestamps never repeat.
  - **Total order**: compare by L, break ties by node name. Every pair of distinct events is ordered, and the order is consistent with causality. This is the basis for total-order broadcast and for "last writer wins" tie-breaking later.
- **Vector clock**:
  - n known nodes; timestamp is a length-n vector, all zeros initially. Entry i = number of events observed from node i.
  - Local event at node i: T[i] += 1. Send: increment and attach T. Receive T': element-wise max(T, T'), then T[i] += 1, deliver.
  - Example: A: <1,0,0>, <2,0,0> (send). B receives: <2,1,0>, then <2,2,0> (send). C had <0,0,1>; receives: max then increment = <2,2,2>.
  - Meaning: a vector summarises the set of events that happened before (and including) this event: "first 2 events of A, first 2 of B, none of C".
  - Comparison: T ≤ T' iff every element ≤; T < T' iff ≤ and not equal; otherwise **incomparable**. ≤ corresponds to subset of event sets.
  - Properties (both directions): V(a) < V(b) iff a → b; V(a) = V(b) iff same event; incomparable iff a ‖ b. So vector clocks *compute* happens-before.
- Trade-off: Lamport = one integer, cheap, total order, but cannot detect concurrency. Vector = O(n) per message, needs known membership, but detects concurrent updates (conflicts).
- Rules of thumb:
  - Use a per-entity version counter (a degenerate Lamport clock) to reject stale updates: "apply update only if version > stored version". Useful when processing out-of-order supplier webhooks or parallel pipeline workers writing the same record.
  - Use vector clocks / version vectors where you must detect conflicting concurrent edits (multi-device editing, multi-leader replication) instead of silently overwriting.

## Martin Kleppmann — Distributed Systems 4.2: Broadcast ordering (https://www.youtube.com/watch?v=A8oamrHf_cQ)
**Group messaging guarantees form a ladder: reliable < FIFO < causal < total order (and FIFO-total); stronger ordering costs waiting.**
- Broadcast/multicast = one node sends to a whole group. Group assumed fixed here (real systems must handle join/leave). Must be fault tolerant: one faulty node must not halt delivery among the rest.
- Hardware LAN multicast exists but not over the public internet; we build broadcast on top of point-to-point unicast.
- Parallel to links: **best-effort broadcast** (may not reach everyone, esp. if sender crashes) vs **reliable broadcast** (retransmission ensures eventual delivery to all correct nodes; no time bound, asynchronous/partially synchronous model).
- Terminology: application *broadcasts*, algorithm *sends/receives* point-to-point messages, then *delivers* to the application, possibly after a **holdback** delay to enforce ordering. A node always delivers its own broadcasts to itself (matters for total order).
- Orderings:
  - **FIFO**: messages from the same sender delivered in send order everywhere; messages from different senders in any order. Can deliver a reply before the message it replies to (if from different senders).
  - **Causal**: if broadcast(m1) → broadcast(m2), every node delivers m1 first; concurrent messages any order, and different nodes may deliver concurrent messages in different orders.
  - **Total order** (atomic broadcast): all nodes deliver all messages in the same order (which order is decided somehow). A sender may have to hold back its own message until earlier-ordered messages arrive.
  - **FIFO-total order**: total order plus per-sender FIFO. Strictly stronger than causal.
- Hierarchy: FIFO-total ⊃ causal ⊃ FIFO ⊃ reliable ⊃ best-effort (each satisfies the guarantees of those below).
- Rules of thumb:
  - Choose the weakest ordering your correctness needs; each step up adds latency and coordination.
  - Per-key FIFO is usually what you need for event streams (all updates for booking X in order). Kafka-style partitioning by key gives exactly this, not global order.
  - If replicas must apply the same updates to reach the same state, you need total order (see state machine replication).

## Martin Kleppmann — Distributed Systems 4.3: Broadcast algorithms (https://www.youtube.com/watch?v=77qpCahU3fo)
**Build broadcast in layers: a reliability layer (re-forwarding or gossip), then an ordering layer (sequence numbers, dependency vectors, or a sequencer).**
- Naive reliable broadcast (sender unicasts to everyone with retransmission) fails: if A reaches C, the copy to B is lost, and A crashes before retrying, C has delivered m1 and B never will. Inconsistent.
- **Eager reliable broadcast**: on first receipt of a message, every node re-sends it to all others. Survives sender crashes; cost O(n²) messages per broadcast.
- **Gossip / epidemic protocols**: each node forwards to a few randomly chosen peers (e.g. fan-out 3); duplicates are possible; with high probability everyone receives it (30 nodes in ~6 rounds in the demo). Robust to crashes and loss; many tunables (when to stop forwarding).
- **FIFO algorithm**: per node: `sendSeq` counter, `delivered[]` vector (count delivered per sender), `buffer` holdback queue. Broadcast (i, sendSeq, m) via reliable broadcast, then increment. On receive: add to buffer; repeatedly deliver any buffered message from sender j whose seq == delivered[j], then delivered[j] += 1.
- **Causal algorithm**: same state. On broadcast: `deps = copy(delivered); deps[i] = sendSeq`; send (i, deps, m); increment. On receive: buffer; deliver any message whose `deps ≤ delivered` (vector-clock comparison), then delivered[sender] += 1.
- **Total order, option 1: leader/sequencer**: nodes send to a single leader over FIFO links; the leader FIFO-broadcasts to everyone; the leader's send order is the total order. Problem: leader crash halts everything; changing leaders safely (never two leaders) requires consensus (lecture 6).
- **Total order, option 2: Lamport timestamps**: deliver in ascending (Lamport, node) order. To know no smaller timestamp can still arrive, use FIFO links and wait until you have heard from every node with a higher timestamp. Not fault tolerant: one crashed node stops everyone's progress.
- Rules of thumb:
  - The holdback-buffer + per-sender sequence number pattern is the standard way to restore order from an unordered/at-least-once channel (e.g. webhook events with sequence numbers: buffer and apply in order; detect gaps).
  - Dedup is required whenever you re-forward or retry (eager broadcast and gossip both deliver duplicates to the network layer).
  - A single sequencer is simple and gives total order but is a SPOF; any "fix" by electing a new one needs consensus or you risk split brain.

## Martin Kleppmann — Distributed Systems 5.1: Replication (https://www.youtube.com/watch?v=mBUCF1WGI_I)
**Replicating changing data under retries and lost messages requires idempotence or dedup, tombstones for deletes, per-key logical timestamps, and a deliberate choice between last-writer-wins and keeping conflicts.**
- Why replicate: survive crashes, hardware faults, partitions and planned reboots (rolling restarts one node at a time); spread heavy read load. Static data is trivial; changing data is the hard part.
- RAID-1 mirrors disks under a single controller; distributed replication lacks that single coordinator and faces unreliable networks across locations.
- **Retry double-apply bug**: client sends "increment likes"; request applied, ack lost; client retries; counter goes 12,300 to 12,302. Real example: a Twitter account showing **following −20**, very likely a repeated decrement on unfollow retries.
- Fixes:
  - **Deduplication**: requires storing request IDs durably (crash-recovery model), which can be a lot of data.
  - **Idempotence**: f(f(x)) = f(x). Increment is not idempotent; "add user ID to set" is. Model likes/follows as a set of IDs, and derive the count.
- Delivery semantics:
  - At-most-once: send once, no retry.
  - At-least-once: retry until acked, may apply multiple times.
  - **Exactly-once (effect)** = at-least-once retries + idempotent operation, or + durable dedup.
- **Limits of idempotence**: client 1 likes (f), ack lost; client 2 unlikes (g); client 1 retries f. The sequence f, g, f leaves it liked, contradicting user intent (liked then unliked). A retry is causally the *old* request, not a new one.
- **Lost-delete ambiguity**: "add then remove" with the remove lost to replica B leaves B {x}, A {}. Identical replica state to "add lost to A". Intents differ, but replicas cannot tell.
- Solution: **logical timestamps per key + tombstones**. Client attaches t1 to add (store x → (true, t1)); remove gets t2 > t1 and is stored as x → (false, t2), a **tombstone**, not a physical delete. Tombstones may be garbage-collected later.
- **Anti-entropy**: replicas periodically compare data and, per key, keep the version with the newer timestamp, converging.
- Concurrent writes (two clients set x = v1 and x = v2; replicas receive them in different orders): arrival order cannot decide, or replicas diverge.
  - **Lamport timestamps → last-writer-wins (LWW)**: keep highest timestamp, silently discard the other concurrent write(s). Fine where losing concurrent writes is acceptable.
  - **Vector clocks**: concurrent writes have incomparable timestamps; keep both as **siblings/conflict** and let the application merge and write back. No data loss, more complexity.
  - Databases differ in which they choose; pick per use case.
- Rules of thumb:
  - Prefer state-setting ("set status = CANCELLED", "add ID to set") over delta operations ("decrement count") on any path that can be retried.
  - Use soft deletes/tombstones with timestamps when syncing data between systems (e.g. location master data or supplier inventory deltas), so a missed delete does not resurrect a record and a late "add" does not undo a later delete.
  - Carry an idempotency key with every retried request so the retry is recognisably the same request, not a new one.
  - LWW silently loses data; never use it for money or bookings without understanding that.

## Martin Kleppmann — Distributed Systems 5.2: Quorums (https://www.youtube.com/watch?v=uNxl3BFcKSA)
**With n replicas, require w write acks and r read responses with w + r > n so every read overlaps the latest write; majority quorums tolerate a minority of failures.**
- Reliability arithmetic (assuming independent faults, which is optimistic since real faults correlate): with per-node unavailability p, P(at least one down) → 1 as n grows (more replicas = something is always broken), but P(all down) = pⁿ and P(half down) both fall exponentially. So design for "works while a majority is up".
- **Read-after-write consistency** violated without quorums: write v1 reaches B only (A down), later read hits A only (B down), returns stale v0. Writing/reading *all* replicas fixes it but then one failure blocks everything.
- **Quorum rule**: write succeeds after w acks; read uses r responses; require **w + r > n** so read and write sets intersect in at least one replica. Example n=3, w=2, r=2. n=5: w=3, r=3.
- Reader gets possibly differing values; picks the one with the newest (logical) timestamp.
- **Majority quorums**: odd n, w = r = ⌊n/2⌋+1. 2/3 tolerates 1 down; 3/5 tolerates 2; 4/7 tolerates 3.
- **Read repair**: if the reader sees a stale replica (or one that did not answer), it writes the newest value back to it, **reusing the original timestamp** (it is a re-delivery of the old write, not a new write). Complements background anti-entropy.
- If w + r ≤ n, reads can miss the latest write.
- Rules of thumb:
  - Read-after-write for a user matters: after a user saves something (a booking, a profile), read it back from the primary or a replica known to have it, not an arbitrary lagging replica.
  - Remember correlated failures (same rack, same bug, same leap second) break the independence assumption behind these numbers.

## Martin Kleppmann — Distributed Systems 5.3: State machine replication (https://www.youtube.com/watch?v=mlWOQuO55PE)
**Deterministic replicas that apply the same updates in the same order end up identical; with weaker ordering you need commutative (and, at the weakest, idempotent) updates.**
- **State machine replication (SMR)**: broadcast every update via FIFO-total-order broadcast; on delivery, apply it with **deterministic** logic. Same start state + same inputs in same order = same final state. The update can be arbitrarily complex (a whole transaction with business logic) as long as it is deterministic.
- Uses: some distributed databases replicate transactions this way; blockchains are SMR (the chain = total-order-broadcast sequence decided by consensus; state = ledger).
- Cost: same as total order broadcast. A replica cannot apply its own update immediately; it must wait for the agreed order. Needs a fault-tolerant total order protocol (next lecture).
- **Passive (leader-follower) database replication**: only the leader runs read-write transactions (read-only may run on followers). Transactions may execute concurrently but **commit in a total order**; each commit appends a record of data changes to the commit log, which is shipped in order to followers who apply it. Same SMR principle.
- Weaker broadcast, stronger requirements on updates:
  - **Causal broadcast**: concurrent updates must **commute** (f then g = g then f).
  - **Reliable broadcast** (no ordering, exactly-once delivery): *all* updates must commute.
  - **Best-effort broadcast** (loss and duplicates possible): updates must commute and be **idempotent**, and tolerate loss.
- Rules of thumb:
  - Make pipeline steps deterministic (no wall clock, no random, no unordered dict/set iteration affecting output) so that replaying the same input log reproduces the same output. This gives you rebuildability: keep an append-only log of inputs and re-derive state.
  - If you cannot guarantee order (parallel workers, unordered queues), design updates to be commutative (set-union, max, counters as per-source sums) and idempotent.

## Martin Kleppmann — Distributed Systems 6.1: Consensus (https://www.youtube.com/watch?v=rN6ma561tak)
**Consensus = automatic, split-brain-free leader change; it is equivalent to total order broadcast and works by two quorum votes (elect the leader, then approve each decision).**
- Context: SMR needs total order broadcast; a single leader/sequencer is a SPOF. **Manual failover** (operator promotes a new leader) is fine for planned maintenance (move the leader before rebooting) but painful for unplanned crashes ("3 a.m. phone call").
- **Consensus problem**: nodes propose values; all decide on one of the proposed values; once decided, never changes. Run consensus once per slot to get total order broadcast; the two problems are **formally equivalent**.
- Algorithms: **Paxos** (single value), **Multi-Paxos** (sequence), **Raft**, **Viewstamped Replication**, **ZAB** (ZooKeeper Atomic Broadcast). Raft/VR/ZAB provide total order broadcast directly.
- System model chosen by all of these: **partially synchronous, crash-recovery**, non-Byzantine.
  - **FLP impossibility**: in an asynchronous system, even with crash-stop failures only, no deterministic consensus algorithm is guaranteed to terminate. Hence timeouts (partial synchrony) are needed for progress.
  - Crucial split: timing affects **liveness** (when messages get delivered) but never **safety** (same order everywhere). Weird delays can stall a correct consensus system, but not corrupt it.
  - Byzantine-fault-tolerant consensus exists (some blockchains) but is more complex and less efficient.
- Leader election mechanics (Raft):
  - Suspect leader failure via timeout (only suspected; could be a network issue).
  - **Split brain** (two nodes both acting as leader) corrupts or loses data; must be prevented.
  - **Term** = integer incremented at each election. Each node votes **at most once per term**; a leader needs a quorum (e.g. 3 of 5, tolerating 2 down). So at most one leader *per term* (an election may also fail).
  - But leaders from *different* terms can coexist: old leader 1 (term t) partitioned away still thinks it leads while 2 and 3 elect node 2 in term t+1.
- Fix: a leader cannot decide alone. **Second quorum round**: "may I deliver m as the next message in term t?" Followers say yes only if they have not seen a higher term. Only with a quorum of yeses is m decided. A deposed leader cannot get a quorum, so cannot commit.
- Rules of thumb:
  - Never implement your own leader election or distributed lock on top of timeouts alone; use a proven consensus system (etcd, ZooKeeper, a Raft library) or a single DB with transactions.
  - Any "leader" or "lock holder" must have its actions checked by others against a term/epoch (fencing token); otherwise a paused/partitioned old leader will act after being replaced.
  - Separate in your head what must never go wrong (safety) from what may be delayed (liveness); timeouts should only influence the latter.

## Martin Kleppmann — Distributed Systems 6.2: Raft (https://www.youtube.com/watch?v=uXEYuDwm7e4)
**Raft is total order broadcast via a replicated log: one elected leader per term appends entries, followers accept only consistent prefixes, and an entry is committed once a majority stores it.**
- Node roles: **follower, candidate, leader**, always within a term. Start-up and crash recovery always begin as follower. Timeout without hearing from a leader → candidate. Majority of votes → leader. Seeing any **higher term** → immediately step down to follower ("peaceful transfer of power"). Election timeout without quorum → retry with a higher term.
- **Durable vs volatile state**: `currentTerm`, `votedFor`, `log`, `commitLength` must be on stable storage (survive crashes); `currentRole`, `currentLeader`, `votesReceived`, `sentLength`, `ackedLength` are volatile and reset on recovery. Lesson: be explicit about which state must survive a crash.
- **Randomised election timeouts** prevent all nodes starting elections simultaneously (e.g. after a common restart).
- **Log**: array of (message, term-of-leader-when-appended).
- **Election**: candidate increments term, votes for itself, sends VoteRequest(id, term, logLength, lastLogTerm). Voter: adopt higher term (become follower, clear votedFor); grant vote iff term matches, it has not voted for someone else this term, and the candidate's log is at least as up to date: **lastLogTerm greater, or equal and log at least as long**. Prevents an outdated node becoming leader and losing committed entries. Candidate collects voter IDs in a *set* (duplicate responses are harmless: idempotent counting); leader when |votes| ≥ ⌈(n+1)/2⌉.
- **Broadcasting**: non-leaders forward the message to the leader over a FIFO link (gives FIFO-total order). Leader appends (msg, term), counts its own ack, calls `ReplicateLog` to each follower. ReplicateLog also runs periodically as **heartbeat**, retransmission, and commit propagation.
- **ReplicateLog**: split log at `sentLength[follower]` into prefix (assumed present) and suffix (new); send LogRequest(leaderId, term, prefixLen, prefixTerm, leaderCommit, suffix).
- **Follower receive**: adopt higher term / accept leader. `logOk` iff follower log length ≥ prefixLen and (prefixLen = 0 or follower's entry at prefixLen−1 has term = prefixTerm). **Log matching invariant**: same index and same term implies identical logs up to there, so one term comparison verifies the whole prefix.
- **AppendEntries**: if existing entries overlap the suffix and the term at the last comparable index differs, **truncate** to prefixLen (those entries were uncommitted, from an old leader; committed entries are never truncated). Append only entries not already present (re-sends are idempotent). If leaderCommit > commitLength, deliver the newly committed entries to the application in order.
- Follower replies LogResponse(id, term, ack = prefixLen + |suffix|, success).
- **Leader on response**: higher term → step down. Success and ack ≥ previous ackedLength (handles reordered responses) → update sent/acked, try committing. Failure → `sentLength −= 1` and resend (walk back to fill gaps; could be optimised).
- **CommitLogEntries**: while some index at commitLength is acked by a majority (including leader), deliver it and increment commitLength; followers learn via the next leaderCommit.
- Rules of thumb drawn from Raft:
  - Monotonic epochs (terms) plus "always defer to the higher epoch" is a general pattern for safe handover of ownership (workers, schedulers, leases).
  - Count acknowledgements in sets keyed by sender, not counters, so duplicate messages are harmless.
  - Acks expressed as "I have everything up to N" (cumulative) are robust to reordering: just take the max.
  - Do not write this yourself; use etcd/Consul/ZooKeeper or a managed DB. Understanding it tells you what guarantees those give.

## Martin Kleppmann — Distributed Systems 7.1: Two-phase commit (https://www.youtube.com/watch?v=-_rdWB9hN1c)
**Atomic commit across nodes needs unanimous votes; 2PC achieves it but blocks if the coordinator dies, and a fault-tolerant variant routes votes through total order broadcast.**
- "Consistency" is overloaded: ACID-C (database satisfies application invariants, e.g. a course with students must have a lecturer); read-after-write consistency; replica consistency (same state when? what do reads return?). Many distinct consistency models.
- **Atomicity**: all of a transaction's updates commit durably, or none do. Foundation for invariants. **Atomic commitment problem**: a transaction spanning several nodes must commit on all or abort on all; if any participant crashes, all abort.
- Atomic commit vs consensus:
  - Consensus may pick *any* proposed value; atomic commit must commit iff *every* participant votes yes, abort if any votes no.
  - Consensus (Raft) tolerates a minority of failures; atomic commit tolerates none (one crash aborts).
- **Two-phase commit (2PC)** (not to be confused with two-phase locking, which is about serialisable isolation):
  1. Client runs the transaction (ID T1) across nodes as usual.
  2. On commit, request goes to a **coordinator**, which sends **prepare** to all participants. Each writes all changes durably, checks constraints, and votes yes/no. **Yes is a binding promise** that it can commit later, no backing out.
  3. Coordinator decides (commit iff all yes), writes decision to disk, sends commit/abort; participants finish and release locks.
- Failure handling:
  - Participant crash/timeouts → coordinator aborts.
  - Coordinator crash: after restart it reads its decision from disk and must honour it (some participants may already have committed); no record → abort.
  - **Blocking problem**: if the coordinator dies after prepares but before announcing the decision, participants are stuck holding locks, unable to commit or abort, until the coordinator recovers (could be long if hardware died).
- **Fault-tolerant 2PC via total order broadcast**: each participant broadcasts its vote (with set of participants R) by TOB to all participants. A failure detector (any node, even the client) that suspects a participant broadcasts a **"no" vote on its behalf**. Because TOB delivers in the same order everywhere, every node counts only the **first** vote per participant; commit when the set of yes-voters equals R, abort on the first no. All reach the same decision even if a slow participant and a suspicious detector race.
- Relevance to bookings/payments:
  - You cannot 2PC with external suppliers or payment providers; they do not expose prepare/commit. Instead use **sagas**: sequence of local steps with compensating actions (hold/charge, book, on failure cancel/refund), plus reconciliation. (note: sagas not covered in the lecture; standard industry alternative.)
  - Some supplier APIs do offer a two-step "hold/pre-book then confirm" flow; treat the hold like a "prepare" with an expiry, and persist your decision before sending confirm so a crash mid-way can be resumed.
  - Persist the coordinator's decision before acting on it, and make the "send decision" step idempotent and retryable.

## Martin Kleppmann — Distributed Systems 7.2: Linearizability (https://www.youtube.com/watch?v=noUNH3jDLC0)
**Linearizability makes a replicated system look like one copy with atomic operations in real-time order; plain quorums are not enough, but quorum reads with synchronous read repair (ABD) or total order broadcast (for compare-and-swap) are.**
- Definition (intuition): every operation takes effect atomically at some instant between its start and finish; if op B starts after op A finished (real time), B must observe A's effect. Overlapping operations may take effect in either order. Defined purely from the client's view, independent of implementation.
- "Strong consistency" is the vague popular term; use linearizability. Also applies to multi-core CPUs: per-core caches act like replicas, so a write on one core may be invisible to another without memory barriers.
- Not the same as **serializability** (transaction isolation: equivalent to some serial order of transactions).
- Not the same as happens-before: happens-before is defined via messages; linearizability via real time, even between clients that never communicate.
- **Quorums are not linearizable**: client 1's write of v1 reaches A quickly, B and C slowly. Client 2 reads A, B → v1. Then client 3 reads B, C → v0. Client 3 started after client 2 finished but saw older data. Violation, despite every operation satisfying w + r > n.
- **Fix (ABD algorithm)**: reader, before returning, writes the newest value (with original timestamp) to replicas that were stale/unknown and waits until it is on a quorum. Gets = quorum read + read repair; sets = blind (unconditional) quorum writes. Linearizable, but concurrent blind writes still overwrite each other.
- **Linearizable compare-and-swap (CAS)**: quorum algorithms cannot do it; use **total order broadcast**: broadcast `get(x)` or `cas(x, old, new)` and act only when delivered; every replica applies them in the same order (SMR), so results agree. Gets are also routed through TOB to be linearizable.
- Rules of thumb:
  - Things like "only one booking per seat/room", unique usernames, account balances, inventory decrements need linearizable CAS/conditional writes (a single primary DB row with `UPDATE ... WHERE version = ?` / `SELECT ... FOR UPDATE`, or a consensus store). Eventually consistent stores cannot enforce them.
  - Reading from async replicas or caches is not linearizable; route "read-your-decision" reads to the primary.
  - Linearizability costs latency (extra round trips) and availability under partitions (see 7.3).

## Martin Kleppmann — Distributed Systems 7.3: Eventual consistency (https://www.youtube.com/watch?v=9uCP3qHNbWw)
**Strong consistency costs latency, throughput and availability under partitions (CAP); eventual consistency keeps every replica usable locally at the price of conflicts that must be merged.**
- Costs of linearizability: ~two round trips per request (slow); a single Raft leader bottlenecks writes (scalability); needs a reachable quorum, else no reads or writes (availability).
- Example: calendar on laptop and phone. Offline (airplane mode), laptop changes the event time to 10 am, phone changes the title. After reconnect the time change survives and the title change is silently lost: **last-writer-wins at the whole-object level**. Consistent in the end, but data lost.
- **CAP theorem** in practice: during a network partition, the minority side must either refuse requests (stay linearizable, lose availability) or serve from local state (stay available, lose linearizability). A forced choice, not a design preference.
- **Eventual consistency**: reads served from local state; updates propagate eventually; if updates stop, replicas eventually converge. Weak: says nothing if updates never stop.
- **Strong eventual consistency (SEC)**: (1) every update applied on one correct replica is eventually applied on all; (2) **replicas that processed the same set of updates are in the same state**, even if applied in different orders. Needs commutative updates (causal/reliable broadcast replication).
- Benefits: operations never wait on network, always fast, keep working offline/partitioned. Background sync via FIFO/causal/reliable broadcast.
- Cost: concurrent updates to the same item create **conflicts**; resolve by LWW (lossy) or smarter merge algorithms (Kleppmann's research: CRDTs, see 8.1).
- **Ladder of assumptions** (strongest to weakest):
  1. Atomic commit (2PC): must reach **all** participants, partially synchronous.
  2. Consensus = total order broadcast = linearizable CAS (proved equivalent): **quorum** + partial synchrony (FLP).
  3. Linearizable get/set (ABD): quorum, but **fully asynchronous** (no timeouts or failure detectors).
  4. Eventual / strong eventual consistency, causal/FIFO/reliable/best-effort broadcast: **no waiting** for any other node, no timing assumptions; local updates applied immediately.
- Rules of thumb:
  - Pick per data type: money, inventory, booking state need linearizable/conditional writes on a single authority; search indexes, caches, analytics, location master data, content can be eventually consistent.
  - Merge at field level, not object level, to avoid the calendar-style loss (time and title edits are independent fields).
  - Offline-capable clients (mobile apps for travellers) imply eventual consistency; plan the merge rules up front.

## Martin Kleppmann — Distributed Systems 8.1: Collaboration software (https://www.youtube.com/watch?v=OqqliwwG0SM)
**Offline-capable collaborative apps converge via CRDTs (commutative merges, op-based or state-based) or operational transformation; the trick is to avoid identifiers (like indexes) whose meaning shifts under concurrent edits.**
- Setting: many users each hold a local replica (browser/app), edit any time even offline, sync later. Concurrent edits must converge (strong eventual consistency) and ideally lose no data.
- Calendar fix: merge at the field level. Laptop's title change ("Lecture 1") and phone's time change (10 am) both survive.
- **Op-based LWW map CRDT**: state = set of (timestamp, key, value). Read: find the entry for key. Set: new globally unique Lamport timestamp (counter + node ID), **reliable broadcast** (key, value, t). On delivery: if no existing entry for key or existing t' < t, replace it. Application is commutative, so any delivery order converges. Calendar: keys = field names.
- **State-based LWW map CRDT**: update local set directly and broadcast the **whole state**; receivers **merge** = union, then keep only highest-timestamp triple per key. Merge must be **commutative, associative, idempotent** (a join-semilattice).
- Op-based vs state-based trade-off:
  - Op-based: small messages; requires reliable broadcast (missing op = permanent divergence).
  - State-based: large messages (entire state); tolerates loss and duplication (any later state supersedes earlier ones), can use best-effort broadcast, and fits quorum + anti-entropy replication too.
- **Google Docs demo**: two windows; network cut; "1 2" typed in one and "2 3" in the other; reconnect yields "1 2 3" automatically.
- **Index problem**: doc "BC". A inserts "A" at 0 → "ABC"; B inserts "D" at 2 → "BCD". Applying B's op literally at A (insert D at 2) gives "ABDC" ≠ "ABCD". Index 2 changed meaning due to a concurrent insert.
- **Operational transformation (OT)**: on receiving an op, transform it against concurrent local ops: T(insert(D,2), insert(A,0)) = insert(D,3); an insert further right does not shift an earlier one. Concurrency detected e.g. with vector clocks. Downside: generally needs **total order broadcast** among users (and the full algorithm is fiddly).
- **Text CRDT with fractional positions**: give each character a unique ID = (rational position in (0,1), node ID). Insert between neighbours at (p1+p2)/2 (B=0.5, C=0.75, A=0.25, D=0.875). Ties (same position from concurrent inserts) broken by node ID. Requires **arbitrary-precision** numbers (floats run out of precision). Delete by broadcasting (position, nodeID), not index. Uses **causal broadcast** so a delete is never delivered before the insert it targets; otherwise ops commute. Efficiency (metadata per character) is an active research area.
- Rules of thumb:
  - Reference things by stable unique IDs, never by positions/offsets that shift (row numbers, list indexes, array offsets in a file being concurrently modified). Relevant to large data pipelines: key records by stable IDs (GeoNames ID, supplier product code), not row order.
  - If you need sync across devices or regions with offline edits, design data as CRDT-friendly: per-field LWW, sets with add/remove tags, counters per node.
  - State-based sync (send full snapshot, merge idempotently) is simpler and loss-tolerant when data is small; op-based is cheaper on bandwidth but needs reliable, sometimes causal, delivery.

## Martin Kleppmann — Distributed Systems 8.2: Google's Spanner (https://www.youtube.com/watch?v=oeycOVX70aE)
**Spanner combines classic pieces (Paxos SMR, 2PL, 2PC) with one novelty: TrueTime clock-uncertainty intervals plus commit-wait give causally consistent timestamps, enabling lock-free consistent snapshot reads.**
- Goals at global scale (millions of nodes): serializable transactions, linearizable reads/writes, **sharding** (each node replicates a subset), atomic commit across shards.
- Classic building blocks: SMR within a shard via **Paxos** (similar to Raft); **two-phase locking** for serializability (shared locks for reads, exclusive for writes, held until commit); **two-phase commit** across shards.
- Problem: big read-only transactions (e.g. a full **database backup**) under 2PL would hold shared locks on everything for hours, blocking all writes. Unacceptable.
- **Consistent snapshot**: a view consistent with causality. If T1 → T2 and the snapshot includes T2's writes, it includes T1's; and vice versa.
- **MVCC** (also in PostgreSQL, MySQL): each read-write transaction gets commit timestamp t_w; writes create new object versions tagged with t_w (old versions kept). A read-only transaction with snapshot timestamp t_r reads, per object, the newest version with timestamp ≤ t_r. No locks; concurrent writes ignored.
- Requirement: T1 → T2 ⇒ ts(T1) < ts(T2).
  - Physical clocks fail (skew).
  - **Lamport clocks fail** too: user sees T1's result on replica A, then triggers T2 on replica B; causality flows through the human, and no message carries the timestamp from A to B.
- **TrueTime**: returns an interval [earliest, latest] guaranteed (with high probability) to contain real time, accounting for RTT to the time server, drift, and other errors. On commit, the transaction **waits out the uncertainty δ** (still holding locks), then commits and releases. Any transaction starting after it in real time therefore gets a non-overlapping, strictly later interval.
- Keeping δ small: **atomic clocks + GPS receivers in every data centre**; every node syncs its quartz clock with the local time server **every 30 s**. Right after sync, uncertainty ≈ 1 ms (local RTT). Between syncs it grows at an assumed worst-case drift of **200 ppm** → ~6 ms over 30 s (sawtooth). **Average uncertainty ≈ 4 ms**, so average commit wait ≈ 4 ms, far less than a ~100 ms intercontinental round trip.
- Result: lock-free, causally consistent snapshot reads, e.g. backups without disrupting writes.
- Rules of thumb:
  - Use MVCC snapshot reads (e.g. Postgres `REPEATABLE READ` read-only transactions, or `pg_dump`) for long exports/analytics on live data instead of locking tables.
  - Causality can travel through humans and outside channels (a user reading a page then clicking), invisible to message-based logical clocks. If ordering matters across such hops, pass a token/version explicitly (e.g. include the last-seen version in the next request).
  - When you rely on time, quantify the uncertainty and design for it (wait it out, or add safety margins) rather than assume clocks agree.

## Arpit Bhayani — Bitcask Explained: A Log-Structured fast KV Store Used By Uber (https://www.youtube.com/watch?v=0WI_tJRUMqM)
**Bitcask = append-only data files on disk + an in-memory hash index (keydir) of every key; one memory lookup and one disk seek per read, sequential writes only, but the entire key set must fit in RAM.**
- Context: embedded (in-process, no network hop) KV store built as a storage backend for **Riak** (transcript says "React"; note: it is Riak, the distributed DB), which Uber ran in production. Every database reduces to efficient on-disk key-value access (row ID → row); B+ trees, LSM trees and Bitcask are alternative answers.
- First principle (from log-structured file systems): **make writes sequential, allow reads to be random.** Disk does minimum work per operation.
- Layout: a directory of append-only files. Exactly one **active** file receives writes; when it reaches a size limit it becomes **immutable** and a new active file starts (avoid unbounded files and file-system limits).
- **Record format**: `CRC | timestamp | key_size | value_size | key | value`. Fixed-width headers let a reader know exactly how many bytes to read in one go (a delimiter like `\n` would need multiple reads and breaks if the value contains it). E.g. 2-byte size fields allow lengths up to 2^16. **CRC over the record** detects torn/partial writes after a crash (e.g. a crash mid-way through a 100 MB value): on recovery, if CRC fails, truncate the file at that point since later bytes are untrustworthy.
- **Keydir** (in-memory, dense index): key → (file ID, offset, size, timestamp). Every key present; miss means not found (no disk access).
- Operations:
  - GET: keydir lookup → open file → seek offset → read fixed header → read key_size + value_size bytes → return value. One memory lookup + one random disk read.
  - PUT: build record, compute CRC, append to active file, update keydir.
  - DELETE: append a **tombstone** record and remove key from keydir. Delete is also an append.
- **Merge/compaction**: iterate immutable files (never the active one), keep only records that keydir says are the latest version, drop overwritten values and tombstoned keys, write compacted files. Disk usage follows a sawtooth (grows, drops at merge).
- Strengths: saturates disk IOPS (does the bare minimum per op), trivial to implement (Arpit says it can be built in minutes; good learning exercise and foundation for understanding LSM trees).
- Limitation: **keys are memory-bound** (whole key space in RAM); values are disk-bound.
- Relevance for Sam's 16GB laptop:
  - For large lookups (e.g. millions of GeoNames IDs → record offsets), the "append-only data file + in-RAM dict of key→offset" pattern lets you random-access a multi-GB file while holding only keys in memory. Estimate RAM: (avg key size + ~50–100 bytes Python dict overhead) × number of keys; 6M keys may need ~1 GB in Python.
  - Append-only + checksum per record is a robust format for crash-safe pipeline checkpoints/output: a killed process leaves at most one torn tail record, detectable via CRC/length.
  - If keys do not fit in RAM, move to an LSM-based store (RocksDB/LevelDB) or SQLite (B-tree).

## Arpit Bhayani — Cache Invalidation using SNS + SQS at Atlassian, and then they moved away (https://www.youtube.com/watch?v=fJNyTNP71MM)
**Push-based per-node invalidation fan-out (SNS → one SQS queue per pod) explodes in cost and throttling at scale; batching cuts messages, and switching to pull (batched invalidations on S3, polled by sidecars) removes the per-node queue entirely.**
- Context: Atlassian's **Tenant Context Service (TCS)**: DynamoDB source of truth, ingestion workers writing to it, TCS API serving reads. Each consumer service runs pods with a **sidecar** that fetches TCS data and caches it **locally per pod** (no central cache), so the main app stays focused on serving requests.
- Original invalidation flow: on write, ingestion worker publishes an invalidation to an **SNS topic** (one per TCS client); SNS fans out to **one SQS queue per pod**; the sidecar consumes and evicts locally; next read misses and re-fetches from TCS API.
- Problems:
  - One SNS topic per client and **one SQS queue per compute node**: operational sprawl, hits AWS account limits.
  - Fan-out = M topics × N queues per invalidation; SQS bills per message. Under heavy write load, invalidation volume spikes and **account-level SQS throttling** kicks in.
- Fix 1, **batching** (lowest-hanging fruit): buffer invalidations for a window (e.g. 1 s) and send one message containing many. 10 invalidations/s → 10× fewer messages, smoother bursts. Cost: added invalidation delay (staleness up to the window). Still leaves one queue per node.
- Fix 2, **push → pull**: ingestion writes batched invalidation records to **S3**; sidecars periodically poll S3, read the batch, and evict. No per-node queues; cost scales with number of batches × pollers rather than per-key fan-out.
- Takeaways Arpit stresses:
  - Infrastructure cost can explode non-linearly as clients/nodes grow, even for "simple" designs.
  - The day-zero naive solution is fine, but revisit long-running services periodically against cost, latency, durability SLAs.
  - Be willing to re-architect when cost/convenience/maintainability degrade (but not every few months for fun).
- Rules of thumb:
  - Whenever you can batch, batch (API calls, DB writes, queue messages); trade a bounded delay for an order-of-magnitude reduction in calls.
  - Push gives low latency but sender bears fan-out cost; pull lets consumers control rate and scales with consumers, at the cost of polling delay. Choose based on how fresh the data must be.
  - Cache invalidation staleness = batch window + poll interval; make that explicit and acceptable for the data (supplier metadata: fine; prices/availability at checkout: re-validate against source instead).
  - For Trameter-style supplier data caches, prefer TTL + periodic pull of change lists over per-item push notifications unless freshness truly demands it.

## Arpit Bhayani — Consistent Hashing and Its Implementation From First Principles (https://www.youtube.com/watch?v=Jvimt56KPnk)
**Consistent hashing is a few-line ownership-lookup algorithm (first node clockwise of hash(key) in a sorted array), not a service; it minimises ownership changes when nodes join or leave, but does not move data for you.**
- Problem it solves: **data/request ownership** in a group of nodes: "given this key/request, which node owns it?" Applies to database shards, caches, load balancers routing requests, stateless API routing.
- What it is not: a silver bullet, a microservice, or a data-migration mechanism. It only answers ownership; moving data when ownership changes is your problem and may be expensive.
- Baseline: `hash(key) mod n`. Changing n (4 → 5) reassigns almost every key (in Arpit's example ~10 of 12 keys move). Ownership churn = mass data movement / cache misses.
- **Ring**: hash nodes (or place them deliberately) onto a hash space ring; a key belongs to the **first node clockwise** (position ≥ hash(key)), wrapping around. Adding N5 between N4 and N2 only moves keys in (N4, N5] from N2 to N5; all other ranges unchanged. Minimal ownership change under elastic topology.
- **Implementation**: never allocate the hash space (2^128/2^256 slots) or build a linked-list ring. Keep a **sorted array of node positions**; for a key, find the first position > hash(key), wrapping to index 0. That is it.
  - Lookup is "first element greater than x in a sorted array": linear scan for a handful of nodes (5-node cluster: don't bother with more), **binary search** (Python `bisect`) for more, or balanced tree / skip list if nodes change often.
  - Alternatives for larger scale: **Jump consistent hash** (Google), **rendezvous (HRW) hashing**, **Maglev hashing** (Google's Maglev paper).
- Rules of thumb:
  - Always think through the implementation of an algorithm; it dispels hype and gives clarity.
  - The routing logic runs wherever the topology is known (API server, LB, proxy, client library).
  - Where you need fine-grained control over placement, or data migration is costly, consistent hashing may not fit; consider explicit directory/lookup-table sharding (see GitHub sharding notes).
  - (note: production rings usually add *virtual nodes* (many points per physical node) to even out load; not discussed in the video.)
- Relevance: when parallelising a pipeline across workers or partitions (e.g. splitting millions of locations into files by key), a stable hash of a stable ID (not Python's per-process randomised `hash()`; use md5/xxhash) guarantees the same record always lands in the same partition across runs.

## Arpit Bhayani — How Amazon Aurora Achieves Durability with Quorum Writes (Aurora paper §2) (https://www.youtube.com/watch?v=EaWlpn24eAs)
**At fleet scale failures are constant background noise, so Aurora keeps 6 copies across 3 AZs (write quorum 4, read quorum 3) to survive "an AZ + 1" and shards storage into 10 GB segments to make repair fast.**
- Durability = once a write is acknowledged, it can be read. Instance lifetime ≠ storage lifetime: compute can be stopped, resized or fail without losing data (motivates storage/compute separation; see the §1 video).
- Scale changes the question from *if* to *when*: if one machine fails ~once/year, a fleet of 365 averages a failure per day. Faults range from transient network lag, reboots, to permanent loss of disk, node, rack, leaf/spine switch, or a whole data centre/AZ.
- **Quorum basics**: V copies; read needs V_r responses, write needs V_w acks.
  - **V_r + V_w > V**: every read overlaps the latest write.
  - **V_w > V/2**: any two writes overlap, so conflicting writes are detected. Counterexample: V=10, V_w=2, V_r=9 satisfies the first rule, but two conflicting writes can be acked by disjoint pairs {A,B} and {I,J}.
- Why the textbook **2-of-3 quorum is inadequate at scale**:
  - All 3 in one AZ: AZ outage = unavailable.
  - 2 in AZ1, 1 in AZ2: AZ1 outage = lost quorum.
  - 1 per AZ in 3 AZs: an AZ outage plus *one more* unrelated node failure (likely at scale; **double faults**) = lost quorum.
- **Aurora design: "AZ+1"**: V=6, two copies in each of 3 AZs, **V_w=4, V_r=3**.
  - One AZ down: 4 copies remain, reads and writes both work.
  - One AZ down + one more node: 3 remain, **writes unavailable but reads still served**. Write availability is sacrificed before read availability.
- **Segmented storage**: volume (up to 64 TB) split into fixed **10 GB segments**, each replicated 6 ways = a **protection group**. Segment (not disk) is the unit of failure and repair.
  - **Mean time to repair** for a segment ≈ **10 s on a 10 Gbps link** (copy from surviving replicas). Losing a quorum would require two failures in the same protection group plus an AZ failure inside that 10 s window: sufficiently unlikely even at Amazon scale.
  - **Operational simplicity**: to deal with a hot disk/node, mark its segments bad and let quorum repair migrate them; same mechanism for rolling OS/security patches (take out one node at a time, re-replicate in parallel). Self-healing.
- Rules of thumb:
  - Evaluate fault tolerance against correlated failures (whole AZ/rack/region) plus one independent failure, not just one node.
  - Small fixed-size chunks/partitions shrink blast radius and repair time and enable parallel recovery and heat management. Same principle for big pipelines: process in fixed-size chunks so a failure costs one chunk's rework, not a full rerun.
  - It is acceptable to degrade to read-only rather than go fully down.

## Arpit Bhayani — How GitHub Sharded Their Databases Without Downtime and Broke Their Monolith (https://www.youtube.com/watch?v=Tq1fif3rcnQ)
**First enforce logical boundaries in code (virtual partitioning with schema domains, query linters, transaction alerts), then move each domain to its own cluster via snapshot + replication + a sub-second, read-only cutover behind a proxy.**
- Starting point: one huge MySQL primary with read replicas, **ProxySQL** in front for pooling/routing, scaled vertically (no premature sharding). Reached ~**950k queries/s**. Pain: no ownership boundaries (anyone queried/joined anything), **noisy neighbours** (a surge in PR creation hurt commit browsing), one outage hit everyone, cross-domain transactions (e.g. stars and comments in one transaction).
- Why not shard earlier: nobody shards on day zero; by the time you need to, the code is a tangle of cross-table joins.
- **Tier 1, virtual partitioning**:
  - `schema_domains.yaml` groups tables into logical domains (gists: gists, gist_comments, starred_gists; repositories: issues, pull_requests, repositories; users...).
  - **Query linters** in dev/test raise exceptions (with helpful messages) on cross-domain joins/queries; exemptions require an explicit code comment, so counting exemptions measures readiness of a domain.
  - In production, **transactions** spanning domains are captured, analysed and **alerted on** (not blocked), since after separation they would lose atomicity.
  - Shared/polymorphic tables (e.g. reactions on issues, PRs, comments) are split per domain or duplicated (e.g. read-mostly copies); exempted from linting.
  - A domain with zero violations is self-sufficient and ready to move.
- **Tier 2, physical move of a domain (gists, cluster A → B)**:
  1. Snapshot gist tables from A; load into B's primary and B's replicas; B replicas follow B primary.
  2. Make **B primary a replica of A primary** (MySQL replication tracked via **GTID**, a monotonically increasing global transaction ID).
  3. Point the application's gist connection at **ProxySQL-B**, which initially forwards everything to ProxySQL-A. Traffic path is now in place; only routing changes later.
  4. Wait until replication lag A→B is tiny (< ~1 s).
  5. **Cutover**: set A primary read-only (gist requests fail briefly with 5xx); poll until B's GTID has caught up with A's; stop replication; reconfigure ProxySQL-B to route to B's primary. Whole cutover **< 100 ms**.
  6. Repeat per domain (repositories, users...).
- Rules of thumb:
  - Make boundaries explicit and machine-checked before splitting anything physically; measure progress via counted violations.
  - Put an indirection layer (proxy, connection-per-domain) in place early so a later move is a config change.
  - Zero-*ish* downtime = keep replica in sync, start cutover only when lag is minimal, freeze writes briefly, verify catch-up by position/ID, then flip.
  - Practise migrations locally (Arpit urges building a prototype with Docker + MySQL replication).
- Relevance for Sam: keep pipeline stages and datasets in clear domains (locations, suppliers, transit) with no cross-domain joins hidden in code, so any one can be moved to its own DB/service later. For dataset swaps (new master location table), build the new version alongside, sync, then flip a pointer (view/alias) atomically.

## Arpit Bhayani — How Kafka Actually Achieves Exactly-Once Semantics (https://www.youtube.com/watch?v=iRhDCIzhm4A)
**Kafka's exactly-once is layered: idempotent producer (producer ID + per-partition sequence numbers), transactional IDs with epochs to fence zombie producers, and two-phase commit across partitions via a transaction coordinator. It is about writes into Kafka, not your consumer's side effects.**
- Default is **at-least-once**: if the broker's ack is lost, the producer retries and the broker appends a duplicate (a two-generals variant). A lost request is safe to retry; a lost ack is not, without help.
- Myth-busting: the guarantee concerns **delivery/storage**, not arbitrary **processing**. Kafka cannot know whether your consumer charged a card or sent an email. (note: Kafka EOS also covers read-process-write *within Kafka* (consume → produce with offsets committed in the same transaction, read by `read_committed` consumers); external side effects still need your own idempotency.)
- **Layer 1, idempotent producer** (`enable.idempotence=true`):
  - Broker assigns each producer instance a **producer ID (PID, int64)**; producer stamps each batch with a **per-partition, monotonically increasing sequence number**.
  - Broker tracks recent sequence state per producer/partition (Arpit: the last 5 batches). Next expected seq → append; already-seen seq → drop as duplicate; gap (e.g. expecting 8, got 10) → reject as out of order. (note: the gap case raises an error rather than being silently ignored.)
  - Broker deduplicates by **(PID, seq)**, not by payload: two identical events sent back to back are legitimately both kept.
  - Batch header carries PID, producer epoch, base sequence, records.
- **Layer 2, zombie fencing via `transactional.id`**:
  - Failure: producer stalls (long GC pause / partition); a replacement starts with a **new PID** and re-sends; the zombie wakes and flushes its delayed writes. Broker sees two independent PIDs and appends duplicates.
  - Fix: the application assigns a stable logical **transactional.id** (e.g. payment-producer-1..3). On `initTransactions`, the coordinator bumps an **epoch** for that ID. Writes carrying an older epoch are rejected (ProducerFenced-style exception) and the zombie shuts itself down.
- **Layer 3, transactions across partitions (2PC)**:
  - Coordinator chosen by hash(transactional.id) mod 50 partitions of the internal `__transaction_state` topic; that partition's leader is the coordinator.
  - Commit: coordinator appends **PREPARE_COMMIT** to its log (an immutable, durable decision), then sends **transaction markers** (COMMIT/ABORT control records) to every partition leader involved, then writes COMPLETE_COMMIT. If anything crashes after the decision is logged, the coordinator retries until all partitions have the marker.
  - Arpit's point: an append-only log needs no "reservation"; the decision is just a durable log entry everyone must honour. (note: data records are appended before the commit; `read_committed` consumers simply skip records of open/aborted transactions.)
- Reusable patterns (Arpit highlights the transactional-ID idea):
  - **(client ID, sequence number)** dedup at the receiver = idempotent writes without inspecting payloads. Mirror this for supplier/payment calls with idempotency keys.
  - **Stable logical identity + epoch/fencing token** neutralises zombie workers after failover. For a pipeline with restartable workers, give each logical worker a fixed ID and bump an epoch in the DB on start; reject writes from older epochs.
  - Log the decision durably first, then drive it to completion with retries (same as 2PC coordinator recovery in Kleppmann 7.1).
  - For booking/payment consumers reading from a queue, exactly-once *effect* = at-least-once consumption + idempotent handler keyed by event ID (store processed IDs in the same DB transaction as the side effect).

## Arpit Bhayani — How MySQL's Midpoint Insertion Strategy Prevents Cache Pollution (https://www.youtube.com/watch?v=DMfr9jMLnz0)
**A one-off full scan would flush a plain LRU cache; InnoDB splits its LRU list into young (5/8) and old (3/8) parts and inserts new pages at the midpoint, so only pages touched again get promoted.**
- Latency ratios Arpit quotes: if RAM access = x, SSD ≈ 4x and HDD ≈ 80x (note: these are his rough relative figures; real gaps for random access are often orders of magnitude larger, so treat them as directional).
- Databases move data in **pages** (4/8/16 KB; InnoDB default 16 KB). Reading one row brings its whole page into memory; a 4 KB page with 1 KB rows holds 4 rows.
- Locality: **spatial** (nearby rows likely accessed next, so caching whole pages pays) and **temporal** (recently used pages likely reused). Hence the **buffer pool**: cached pages managed by **LRU** (doubly linked list; hit moves page to head; evict from tail).
- Miss path: read page from disk, insert into cache, evict tail if full.
- **Problem: cache pollution by sequential scans**: a full table scan (e.g. dumping a table) makes every scanned page "most recent", pushing out the entire hot working set; the scanned pages are never used again; subsequent queries all miss and hit disk.
- **Midpoint insertion**: list = **young sublist (head, ~5/8)** + **old sublist (tail, ~3/8)**. New pages enter at the **head of the old sublist** (the midpoint). Eviction from the old tail. A page accessed again while in the old sublist is promoted to the head of the young sublist (a "second chance"). A full scan therefore churns at most the old 3/8; the hot young set survives.
- Observability/tuning: `SHOW ENGINE INNODB STATUS` → BUFFER POOL AND MEMORY (pages made young / not young, young-making rate, buffer pool size, free buffers, database pages). `innodb_old_blocks_pct` (default **37** ≈ 3/8) sizes the old sublist. (note: `innodb_old_blocks_time` also delays promotion of pages touched repeatedly within a short window during a scan; not mentioned in the video.)
- Rules of thumb:
  - Any LRU cache that serves both a hot working set and occasional bulk scans needs scan resistance (midpoint/2Q/segmented LRU, or bypass the cache for bulk jobs).
  - Running big exports/analytics on the same DB that serves live traffic can degrade it by evicting hot pages; run them off a replica or at quiet times.
  - On a 16GB laptop: OS page cache behaves similarly; streaming a huge file once can evict other useful cached data. For in-process caches (e.g. `functools.lru_cache` over lookups while streaming millions of rows), size them deliberately and keep one-off keys out.

## Arpit Bhayani — How PayPal Beat the Thundering Herd Problem and Fixed Their Architecture (https://www.youtube.com/watch?v=pFBCgFzS2W8)
**Synchronized retries amplify overload into cascading failure; adding random jitter to backoff de-synchronises them. And delete microservices that are just unnecessary hops.**
- **Thundering herd**: an overloaded service fails requests; clients retry; retry load stacks on original load; more failures; more retries. Plain exponential backoff (1, 2, 4, 8, 16, 32 s) still fails if all clients failed at the same moment, because they retry at the same moments.
- Braintree (PayPal) disputes flow: merchants call **Disputes API** via SDK; traffic is very bursty. API enqueues a job (ActiveJob, Rails' Sidekiq/Celery-like abstraction) on **SQS**; a worker makes a synchronous HTTP call to a **Processor Service**, which stored disputes in its DB; a cron (every few minutes/hours) batched them into zip files and pushed them via **SFTP** to external payment processors.
- Symptoms: cascading failures; job queue build-up; overloaded Processor Service; connection failures; synchronized retries; jobs exhausting retries and landing in the **dead-letter queue (DLQ)**. A DLQ is for poison messages needing human attention and is closely monitored; flooding it with transient failures means humans must triage what was actually retryable.
- **Fix 1, jitter**: random variation added to backoff to spread retries over a window.
  - **Full jitter**: sleep = random(0, backoff).
  - **Equal jitter**: sleep = backoff/2 + random(0, backoff/2) (guaranteed minimum wait plus randomness).
  - Even a crude random 0–5 s spread helps; the point is to break convergence and flatten peaks.
- **Fix 2, remove the unnecessary microservice**: the Processor Service only buffered in a DB and batched on a cron. Moved batching + submission into the worker itself: worker consumes from queue, batches, submits to processors. One less network hop, one less thing to overload.
- Arpit's lessons:
  - Simple fixes (jitter) can solve seemingly complex problems.
  - Microservices are an anti-pattern in most cases; design as if you were the only engineer who must build, run and monitor it. "Simple systems scale."
- Rules of thumb for Sam:
  - Every retry loop against suppliers/APIs: exponential backoff + jitter + cap on attempts + cap on max delay; honour `Retry-After`/429 headers.
  - Distinguish retryable (timeouts, 429, 5xx) from non-retryable (4xx validation) errors before retrying; send only genuinely poison messages to a DLQ.
  - Retries of booking/payment calls must carry idempotency keys (see Kafka/Kleppmann notes).
  - Batch submissions to external systems when they accept batches.
  - Prefer a worker that does the whole job over chaining thin services.

## Arpit Bhayani — How Stripe Migrates Millions of Records and Schema with Zero Downtime and No Data Inconsistency (https://www.youtube.com/watch?v=FtJPvgMsn-Q)
**Online migrations use dual writes in four steps (dual-write + backfill, move reads with shadow comparison, move writes but keep the old store updated for rollback, then delete old), with the backfill run on a snapshot in parallel.**
- Why schemas evolve: cleaner abstractions, and new/evolved requirements. The hard part is not the ALTER but doing it while keeping **consistency** (never an incorrect state) and **availability** (no downtime).
- Scale arithmetic (Stripe's numbers): 100M subscription objects at even 1 s each sequentially = 100M s ≈ **3 years**. So migration must be parallel/distributed.
- Concrete case: Stripe (MongoDB-based "DocDB") initially embedded **one subscription inside each customer document**; requirements changed to many active subscriptions per customer; target: separate `subscriptions` collection.
- **Step 1, dual write + backfill**:
  - Writes go to the old store and are propagated **asynchronously** to the new one, so request latency does not double.
  - Backfill old records: naive paginated "everything updated before T" queries against production would hurt live traffic. Instead take a **snapshot** of the old collection and run a **MapReduce**-style distributed job over the snapshot to transform and write into the new collection.
  - Then verify nothing was missed (completeness check).
- **Step 2, move reads**: switch read paths to the new store, but run **shadow comparison**: also read from old, compare, and **alert on mismatches**. This is what proves correctness before trusting it.
- **Step 3, move writes**: writes go to the **new** store and propagate asynchronously back to the **old** one, so a rollback to old remains possible (plan B, and plan B for plan B). Code changes span thousands of lines; keep each change simple and modular to build confidence; ensure no write path is missed.
- **Step 4, delete old**: only after confirming 0% traffic, no stray code paths, configs or connection strings still use the old store; otherwise a forgotten path breaks (e.g. servers failing to start when a removed cluster's connection fails).
- Rules of thumb:
  - Never run heavy backfill/export queries against the live primary; use a snapshot or replica.
  - Parallelise transformations; estimate total time = items × per-item time / workers before starting.
  - Shadow-read and diff before cutting over; alert, don't assume.
  - Keep the old path reversible until the new path has run cleanly for a while.
  - Verify completeness explicitly (counts, checksums per partition).
- Relevance for Sam: replacing a master location dataset or a supplier mapping table follows the same pattern: build new table alongside, backfill in parallel chunks from a snapshot file, run both and diff results (e.g. match rates per country), flip reads, keep old for rollback, then drop.

## Arpit Bhayani — Inside Stripe's Rate Limiter Architecture (https://www.youtube.com/watch?v=Oxy-6MAiYPw)
**Stripe layers four independent defences (request-rate limiter, concurrent-request limiter, fleet-usage load shedder, worker-utilisation load shedder) and always prefers degrading low-priority traffic over an outage.**
- Goals: availability under unpredictable load; **fairness across tenants** (one merchant's burst must not hurt others); low-priority traffic must not eat capacity needed by critical traffic; predictable latency.
- Why simple approaches fail: one global per-request throttle lacks context; load shedding without prioritisation hurts UX; a single layer is not enough.
- Failure mode being prevented: **cascading failure**: high load → higher latency → client timeouts → client retries → even higher load.
- Design principles:
  - Multiple independent protection layers.
  - Prefer **service degradation over outages**.
  - **Fail open**: if the rate limiter itself is down, let requests through rather than rejecting everything.
  - **Load shedding**: drop low-priority work so high-priority work has resources (like scheduled power cuts to households to keep industry running).
- The four tiers:
  1. **Request rate limiter**: N requests/second per user/tenant via **token bucket** (refilled each second; one token per request); stored in Redis. Limits can be raised per tenant temporarily (e.g. a merchant's flash sale).
  2. **Concurrent request limiter**: caps requests **in flight** at once (e.g. at most 20 concurrently), independent of rate. Protects expensive endpoints (Arpit's analogy: allow 1,000 req/s overall but at most ~10 concurrent video-transcode jobs).
  3. **Fleet usage load shedder**: classify requests as **critical** vs **non-critical**; always reserve at least **20% of fleet capacity for critical** traffic; reject non-critical traffic beyond 80% even if spare capacity exists at the moment. Deliberately leaves capacity unused because, for a payment processor, failing a critical request is unacceptable.
  4. **Worker utilisation load shedder** (last resort, per server, rarely triggered, used in major incidents): traffic buckets in priority order: critical methods > POSTs > GETs > **test-mode** traffic. Under overload, shed from the lowest bucket up; restore gradually as the incident resolves.
- Lesson: engineering is an enabler of business priorities; decide explicitly what matters most and shed everything else first.
- Rules of thumb for Sam (as a *client* of rate-limited supplier APIs and as an API provider):
  - Client side: apply your own token bucket per supplier (respect their published limits), plus a **concurrency semaphore** (e.g. asyncio.Semaphore) per supplier; these are separate limits.
  - Prioritise: booking/checkout calls > availability/price checks > bulk catalogue syncs. When a supplier throttles, pause the bulk sync, not the bookings.
  - Fail-open vs fail-closed is a per-component decision; for a non-essential limiter, fail open.
  - Serving your own API (e.g. Drive Button, Tereka): per-tenant token bucket + concurrent limit on expensive endpoints (LLM calls, big searches) protects you on cheap infrastructure.

## Arpit Bhayani — Load Balancers are not Magic: Dissecting an Atlassian Outage (https://www.youtube.com/watch?v=KwGyA6B7qrI)
**A managed load balancer is itself a scaled fleet that takes minutes to grow; Atlassian's blue-green deploy pointed full production traffic at a brand-new, cold ELB, which collapsed until it scaled up ~5 minutes later.**
- Setup (2019): clients → AWS **Elastic Load Balancer** → API servers → database. During the incident API servers looked healthy (normal CPU/memory, no error logs) but **requests reaching the LB dropped to zero**; it self-recovered after ~5 min with no changes.
- AWS's diagnosis: newly created LB nodes inside the ELB were dying of **OOM**. An ELB is many LB servers behind one DNS name/IP, scaled horizontally with load.
- LB scaling is slow: adding an internal LB node takes ~**1–7 minutes** before it serves. LBs cope with gradual **surges** but not sudden **spikes** (flash sale, marketing push, a cricket match turning exciting).
- Root cause: Atlassian's deploys were **blue-green including the ELB**: new API servers *and* a **new ELB**, then a **DNS flip** to the new ELB. The new ELB started with ~2 nodes (Arpit's illustration: 2 × 10k rps = 20k capacity) while 100k rps instantly moved to it, so ~80k rps failed until the ELB scaled out. Not "warm".
- Fix: **reuse the same load balancer** and do a **rolling** replacement of API servers behind it; no recurrence.
- Takeaways:
  - LBs (and any autoscaled managed service) have scale-up latency; if expecting spikes, **pre-warm** (ask the cloud provider) or keep capacity ahead of demand.
  - Understand how your infrastructure components work internally (scale-out time, warm-up) instead of treating them as black boxes.
  - A green dashboard for your app servers does not mean users are served; monitor at the edge (LB request count, synthetic checks).
- Rules of thumb for Sam:
  - When cutting traffic over to new infrastructure (new server, new DB, new API key/account with a supplier), shift gradually (weighted routing, canary) rather than all at once; new components start cold (empty caches, small pools, low quotas).
  - Bulk jobs that suddenly hammer a supplier API or a freshly started DB look like a spike to them; ramp up concurrency gradually.

## Arpit Bhayani — What are L4 Load Balancers and how do they work? (https://www.youtube.com/watch?v=RcarDmgWezY)
**An L4 load balancer routes on transport-level info only (IPs, ports, TCP/UDP) without reading payloads; in pass-through mode it rewrites and forwards packets on one end-to-end TCP connection, in proxy mode it terminates the client connection and opens its own to a backend.**
- Purpose of any LB: spread load across backends and hide infrastructure elasticity from clients (machines added or removed transparently), giving reliability, scalability and availability.
- OSI framing: **L4** = transport (TCP/UDP): sees source/destination IP and port, protocol; does not inspect data. **L7** = application (HTTP, WebSocket...): reads URL, headers, cookies; can rewrite URLs, add/remove headers, route by path (nginx as reverse proxy). The distinction is *how deeply it looks into the data*.
- L4 examples: AWS Network Load Balancer, HAProxy in TCP mode, Linux iptables/nftables.
- **Pass-through mode**: a single TCP connection from client to backend; LB (e.g. 192.168.1.100) rewrites packet destination to a backend (…101 or …102) using only the 4-tuple + primitive metadata. Algorithms: random, hash of source IP or 4-tuple. Responses go back through the LB; with **masquerading** (SNAT) the backend's IP is hidden behind the LB's.
  - iptables demo: PREROUTING DNAT rules with `--probability 0.5` to send half of port-80 traffic to 101, the rest to 102; POSTROUTING MASQUERADE.
  - Very cheap and fast, but little flexibility.
- **Proxy mode**: client's TCP connection terminates at the LB, which opens a second connection to the chosen backend. Enables health checks, **least-connections**, **weighted round robin**, latency-aware routing, connection-level observability (RTTs, drops), TCP-level rate limiting/retries by source IP. Still does not read the payload. Examples: HAProxy (TCP mode), Envoy, nginx `stream`.
- Choosing: proxy mode when you need connection-level observability, smarter routing, or limits/retries; pass-through when raw throughput and simplicity matter.
- Arpit's advice: prototype with 3–4 containers (HAProxy config: `balance roundrobin`, server lines with `check`) to understand it.
- Relevance: you rarely run an LB on a laptop, but knowing L4 vs L7 explains behaviours like long-lived connections (WebSockets, HTTP keep-alive pools to suppliers) sticking to one backend under L4 hashing, and why L7 is needed for path-based routing or header-based tenant routing.

## Arpit Bhayani — What the heck is Storage-Compute Separation? (Aurora paper §1) (https://www.youtube.com/watch?v=DA5W8tO_7Nw)
**Cloud databases separate a stateless compute tier from a replicated storage tier, trading a network hop (and costlier transactions) for elastic scaling, cheap idle cost and simple operations; Aurora cuts the network cost by shipping only redo logs to storage.**
- Distributed cloud DBs sell **resiliency** (losing a node doesn't affect availability; easy replacement) and **scalability** (absorb rising load without heavy ops). Almost all do two things: **decouple storage from compute**, and **replicate (partitioned) storage across nodes**. Examples: Aurora, CockroachDB, Spanner, Dynamo.
- Traditional: one VM runs MySQL/Postgres with its own disk (compute and storage coupled). Separated: compute nodes accept the query, parse, plan, fetch from storage nodes over the network, then join/aggregate and respond; storage nodes handle persistence, replication, durability and fast I/O.
- Why: **storage is cheap, compute is expensive**. With separation you pay for compute only when queries run, which enables **serverless databases** (Neon, acquired by Databricks; PlanetScale for MySQL). Coupled nodes must run forever.
- Operational simplicity: misbehaving storage node → bring up another and copy from replicas; compute nodes are **stateless** → just replace them; adding read capacity, failover, scaling all become easy.
- Costs ("no win-win"):
  - **Network becomes the bottleneck**: compute pulls data from storage to join (some DBs push joins down), so bandwidth, TCP connection limits, open-file limits can bite.
  - Storage nodes need high, well-provisioned IOPS.
  - **Fan-out writes**: latency is bounded by the **slowest replica** you wait for; stricter guarantees cost throughput/latency.
  - **Transactions get expensive**: commits spanning machines need 2PC-like coordination (locks on multiple nodes, extra round trips), and long distributed transactions are fragile under failures.
  - At scale there is constant background noise of hard and soft failures; retries are always happening.
- **Aurora's approach** (detail in later sections): built on MySQL; storage is independent, fault tolerant, **self-healing**, spans multiple data centres/AZs; avoids performance variance from failures in any tier; **ships only redo log records** to storage (far less network I/O, so much higher throughput than standard MySQL); side benefits: **continuous backups** (logs streamed constantly), near-instant crash recovery without checkpointing, and asynchronous background work that does not stall foreground queries. Arpit contrasts this with dumps on traditional DBs that may stall traffic. (note: `pg_dump` uses an MVCC snapshot and does not block ordinary reads/writes in Postgres; it blocks only conflicting DDL. Heavy dumps still consume I/O and cache.)
- Rules of thumb:
  - Keep compute stateless and state in a durable store; then compute can be killed, replaced or scaled freely (applies to pipeline workers too: workers read chunks from storage, write results back, hold nothing important in memory).
  - Ship small deltas/logs rather than whole pages/objects across the network when replicating.
  - Push filters and aggregation close to the data to cut transfer (e.g. query Parquet with DuckDB column/row-group pruning instead of loading whole CSVs into pandas on a 16GB machine).
  - Every synchronous fan-out inherits the latency of its slowest leg; set per-leg timeouts and consider quorum-style "first k of n" waits.

