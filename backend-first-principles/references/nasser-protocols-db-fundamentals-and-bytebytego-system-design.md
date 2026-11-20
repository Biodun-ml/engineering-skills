# hussein-bytebytego — cross-cutting principles

1. **Ask first: is the thread computing or waiting?** If it's waiting on I/O (APIs, disk, DB), use async on one thread or event loop. If it's computing, use separate processes on share-nothing partitions of the data. Use shared-memory multithreading only when you must, because locks and races are expensive to get right. [Hussein: Async vs Multithreading vs Multiprocessing; Web servers]
2. **Connections are expensive, stateful resources.** Each one costs TCP and TLS handshakes plus memory on both ends. Reuse them with keep-alive, pools and HTTP/2 multiplexing. Cap their number. Always release them in `finally` or a context manager. "Works for N requests then hangs" usually means a leaked pooled resource. [Hussein: HTTP crash course; TCP vs UDP; TLS; Postgres connection leak]
3. **Set every timeout explicitly, and know which one fired.** The main ones are connect, request-read, queue-wait, processing and response (plus pool-acquire). A timeout only means someone stopped waiting. The work may still be running, and real cancellation has to be built on purpose. [Hussein: Timeouts; Postgres connection leak]
4. **Blind retries make outages worse.** A request that timed out is often still being processed. Retrying multiplies load and causes thundering herds. Retries need limits, backoff and jitter, circuit breakers, and above all operations that are safe to repeat. [Hussein: Timeouts; ByteByteGo: 7 Concepts (circuit breaker)]
5. **Assume at-least-once delivery everywhere and make consumers idempotent.** Publisher acks get lost, consumers crash before committing offsets, and inbox redeliveries happen until an ack arrives. Exactly-once is costly. Idempotency keys and dedupe-by-id are the practical answer, especially for bookings and payments. [Hussein: Pub/Sub; ByteByteGo: 20 Concepts, Kafka, Chat System]
6. **Accept fast, process later.** Persist and enqueue the work, return an id or "pending" status at once, and let workers do the slow parts with backend retries. Clients poll, long-poll or get a push. Chained synchronous calls across services break as soon as any link fails. [Hussein: Sync vs Async workloads; Sync vs Async apps; Pub/Sub; ByteByteGo: YouTube]
7. **Check-then-act on shared state is a race.** Examples are a balance check followed by a debit, rate-limit counters across servers, and availability checks before booking. Make the check and the write one atomic step: a transaction with the right isolation or `SELECT ... FOR UPDATE`, a conditional UPDATE, a Redis Lua script, or a unique constraint. [Hussein: ACID; ByteByteGo: Rate Limiter]
8. **Pick isolation and consistency per data type, not globally.** Money and inventory need atomic transactions and strong reads. Counters and feeds can be approximate or eventually consistent. Every step up in strength costs throughput. Know your database's default isolation level (usually read committed). [Hussein: ACID; ByteByteGo: 7 Concepts, Key-Value Stores]
9. **Replicas give read scale but also stale reads.** Async replication is fast and eventually consistent. Sync replication is consistent and slow. After a user writes, send that user's reads to the primary (read-your-writes), as YouTube had to. [Hussein: ACID; Sync/Async workloads]
10. **Durability is a dial, so turn it down only for data you can regenerate.** Commit with fsync of the WAL is safe and slow. `synchronous_commit=off` or relying on the OS cache is fast but can lose recent commits. That is fine for reloadable staging data and never fine for bookings. [Hussein: Sync/Async workloads; ACID]
11. **Physical layout drives query cost.** Clustered (primary) keys order the table, so range scans are cheap. Secondary indexes add a heap hop per row. Random UUID primary keys on clustered tables wreck insert locality, so prefer sequential or time-ordered ids. [Hussein: Primary vs Secondary Key]
12. **Partition work by the key you need to serialise on, and watch for skew.** Use per-host queues for politeness, per-entity Kafka keys for ordering, and consistent hashing so adding nodes moves only about K/N keys. Split hot keys with a compound key, such as entity plus a hash. [ByteByteGo: Web Crawler, Kafka, Key-Value Stores, 7 Concepts]
13. **Big jobs should be a DAG of small, independent, checkpointed units.** Use chunked, checksummed, resumable transfers. Process segments in parallel. Retry only what failed, and record progress so a crash resumes instead of restarting. [ByteByteGo: YouTube, Web Crawler; Hussein: Timeouts (queue)]
14. **Rate-limit with a token bucket: capacity sets the burst, refill sets the sustained rate.** Enforce it in middleware with shared, atomic state, return 429 with retry information, and decide whether to fail open or closed. As a client, honour 429/Retry-After and run your own per-destination bucket. [ByteByteGo: Rate Limiter, 7 Concepts, Web Crawler]
15. **Fail fast rather than slow.** A circuit breaker (closed, open, half-open) stops callers piling threads and connections onto a sick dependency and gives it time to recover. [ByteByteGo: 7 Concepts; Hussein: Pub/Sub]
16. **Measure latency with percentiles, not averages.** Report p50, p95 and p99 together, set timeouts and SLOs from the tail, and distrust a single headline percentile. Alert on composite signals tied to SLOs. [Hussein: Percentile Tail Latency; ByteByteGo: 7 Concepts]
17. **Put cross-cutting network concerns at a proxy or gateway layer.** That covers TLS termination, routing and canaries, auth, rate limits, caching and aggregation. Remember the proxy is itself a client with its own timeouts and retries (504s), and it must be highly available. [Hussein: Proxy vs Reverse Proxy; Timeouts; ByteByteGo: 20 Concepts]
18. **Know which layer a failure lives in.** DNS, then TCP, then TLS (SNI/ALPN), then HTTP status. A cold request pays DNS + TCP + TLS before any server work, so cache DNS and reuse connections. Encrypt in transit, prefer TLS 1.3 (one round trip, forward secrecy), and never disable certificate checks. [Hussein: OSI; google.com walkthrough; TLS; HTTP]
19. **Design by writing, and start from numbers.** Write the user workflow, gather stakeholder questions, cut an MVP, write a technical overview and per-component specs (inputs, interfaces, computation, outputs), and draw box diagrams last. Size reads/s, writes/s and storage before choosing technology. Add complexity (microservices, Kafka, sharding) only when the evidence demands it. [Hussein: Design Process; Backend Fundamentals; ByteByteGo: URL Shortener, 20 Concepts, 7 Concepts, Kafka]

---

## Hussein Nasser — Asynchronous vs Multithreading and Multiprocessing Programming (The Main Difference) (https://www.youtube.com/watch?v=0vFgKr5bjWI)
**A thread waiting on I/O is doing nothing; async reclaims that idle thread, multithreading adds threads that share memory, multiprocessing adds isolated processes that can spread across machines.**

- **First principles: what "blocked" means.** Every process starts with one thread. When that thread computes (e.g. checking primes) it is genuinely busy on a CPU. When it reads a disk, calls a REST endpoint or talks to a printer, the real work is done elsewhere (the I/O controller, the remote server, the OS); the thread just sits on a file descriptor waiting. In synchronous code everything after that call is stuck too, including UI handling in a single-threaded app.
- **Multithreading.** Spin up more threads inside the same process. They share the process's memory and resources, which is the root problem: concurrent access to the same memory causes races, so you need mutexes/locks ("thread safety"), much like row locking in a database. Threads can run truly in parallel on multiple cores. Hussein's (admittedly opinionated) view: very hard to get right, needs a deep understanding of the OS and CPU, and often not worth the pain; cites SQLite's author disliking threads.
- **Asynchronous (single-threaded, non-blocking).** Node.js is the reference example. Instead of waiting, the thread hands the request to whoever does the work (I/O controller, network stack), registers a callback, and immediately moves on. When the work completes, an event is queued; an **event loop** repeatedly checks "is there code to run / a completed callback?" and runs it. No shared-memory races because there is one thread.
  - Ergonomics evolved: nested callbacks ("callback hell", AJAX era ~2003–2004) -> promises (`then/catch` chains) -> `async/await`, which is syntactic sugar that makes non-blocking code read like sequential code.
- **Multiprocessing.** Rather than threads sharing memory, run separate processes, each with its own memory. They coordinate through IPC: sockets on localhost ports, a shared store such as Redis, etc. Key advantage: the same design scales from one machine to many (containers are a more isolated version of the same idea; Hadoop splits work similarly).
  - Example: brute-forcing an MD5 hash. Split the candidate space (dictionary A–M to one process, N–Z to another) so each worker searches independently. A reverse proxy (e.g. nginx with Lua) can even act as the splitter that farms ranges out to workers/containers.

**Decisions & trade-offs**
- I/O-bound work (waiting on APIs, disk, DB): async on one thread is the cheapest, simplest win; no locks needed.
- CPU-bound, embarrassingly parallel work: multiprocessing; partition the input so workers share nothing.
- Multithreading: only when you really need shared in-memory state with parallel CPU, and you are prepared to reason about locks and races.

**Rules of thumb**
- Ask first: is my thread computing or waiting? Waiting -> async. Computing -> processes.
- Prefer share-nothing partitioning over shared memory + locks.
- Use async/await syntax to keep non-blocking code readable.

**Failure modes**
- Shared mutable state across threads -> race conditions, deadlocks, subtle bugs.
- Callback nesting -> unreadable code; use promises/await.
- A CPU-heavy task on a single async thread will still block the event loop (implied by the model: async only helps when the thread is idle-waiting).

**Relevance to Sam**
- Supplier API calls (Viator, RateHawk, etc.) are classic I/O waits: run many concurrently with async (e.g. Python `asyncio`/`aiohttp`) rather than one thread per call.
- Large row-level transforms (GeoNames matching) are CPU-bound: chunk the data and fan out to processes (`multiprocessing`/`ProcessPoolExecutor`). On 16GB RAM remember each process has its own memory copy, so cap worker count and chunk size rather than loading the whole dataset into every worker.

## Hussein Nasser — Frontend and Backends Timeouts (https://www.youtube.com/watch?v=2GAQVXGT_Zw)
**Timeouts exist to free resources and bound waiting; there are at least five distinct ones along a request's life, and giving up waiting is NOT the same as cancelling the work.**

- **First principles.** A timeout says "stop waiting after T". Reasons: release memory/CPU/connections, blunt denial-of-service attempts that try to make you hold resources, and surface operations that breach an SLA (e.g. endpoint should take 3s, so time out at 5s and log it). Stopping the wait and cancelling the work are orthogonal: the server may keep processing after the client walks away.
- **Everyone is both client and backend.** A reverse proxy/CDN is a backend to the browser and a client to the origin; the same timeouts apply at every hop.

**The five timeouts**
1. **Connection timeout (client side).** How long to wait for the TCP 3-way handshake plus the TLS handshake that people usually lump into "connection". On expiry, stop or retry later.
2. **Request read timeout (backend side).** The server receives a byte stream, decrypts it, and parses until it finds a complete request (for HTTP/1.1: method/path/version ... terminating blank line). Clients can be slow (uploads, big POST bodies) or malicious: **Slowloris** sends one byte at a time to pin server resources. Set this based on what you know of your traffic; if your requests are GETs of ~1–2 KB including cookies and JWTs, a second or two is plenty. HTTP 408 (Request Timeout) corresponds to this case (note: the transcript says "403", which is Forbidden; 408 is the correct code).
3. **Wait (queue) timeout (backend side).** Execution doesn't have to be immediate: the reader can push the request onto a queue and a worker pool pops it. That adds overhead but makes load measurable and controllable (queue depth, time-in-queue, prioritisation of important requests). Cap how long a request may sit unpicked; tell the client when it was dropped. Guidance: short requests -> execute synchronously; long-running ones -> queue.
4. **Usage / processing timeout (backend side).** How long a request may consume workers/CPU (huge sorts, big DB reads) before you kill it. Should vary by request type. HTTP has no standard status code specifically for "processing exceeded".
5. **Response timeout (client side).** How long the caller waits for a reply. Browsers don't impose one (e.g. on server-sent events); you can implement it with `AbortController`/abort signal on `fetch`. Aborting only stops your waiting.

**Cancellation mechanics**
- Closing the connection sends a TCP FIN; the kernel sees it, but your app only learns if it listens for that event and has bookkeeping mapping connection -> in-flight requests -> where they are running (DB query, loop, etc.). Your processing loops must check a cancel flag.
- Heavy-handed alternative: one request per thread/process and kill it on disconnect; works but respawning threads/processes is costly and lengthens queue waits for others.
- Logically, the clean way to cancel is a separate cancel request (HTTP/2 has stream reset built in).
- Don't assume the framework cancels for you. It's fine to consciously decide "not a problem for my use case" and keep it simple, as long as it's a decision.

**Proxies and gateway timeouts**
- Through a CDN/reverse proxy, your connection and read timeouts are with the proxy. The proxy then becomes a client to the origin with its own connection and response timeouts; failures show up as **504 Gateway Timeout** (couldn't connect, or connected and sent but no reply in time).
- Proxy can retry N times or fail fast; it's configuration.

**Biggest gotcha: retries on timeout.** A timed-out request is usually still running. Retrying adds a duplicate, the overloaded service gets 2x, 3x, 4x load across the whole fleet, and when it recovers it is immediately crushed again (thundering herd). Hussein cites the 2020 AWS Kinesis outage as this pattern.

**Rules of thumb**
- Set all five explicitly; know your request size/duration profile and set tight values.
- Per-request-type processing budgets.
- Retries need limits, backoff, and the operation must be safe to repeat.
- Log timeouts as SLA signals.

**Relevance to Sam**
- Supplier APIs (Viator, RateHawk, Coras): set connect and response timeouts on every HTTP client; never rely on defaults (Python `requests` has no default timeout).
- Bookings/payments: a response timeout does not mean the booking failed. Before retrying, use an idempotency key or query the supplier's booking status; otherwise you risk double bookings/double charges.
- Long jobs (bulk matching, enrichment) belong on a queue with wait and processing timeouts, not in a synchronous HTTP request.

## Hussein Nasser — How to Become a Good Backend Engineer (Fundamentals) (https://www.youtube.com/watch?v=V3ZPPPKEipA)
**Learn concepts, not tools: protocols, web servers, databases, proxies, caching, messaging, frameworks, message formats, security. Sample them all, then go deep on one.**

- **Mental model.** Every technology exists because something before it fell short. Learn the "why" (Kafka exists for X, gRPC exists because REST/JSON had limitation Y) and the tool becomes obvious. If you can't find a reason a thing exists, question it or drop it. Nobody is an expert in all of this; breadth to taste, then depth in what pulls you.

**The fundamentals map (with the "why" for each)**
1. **Communication protocols.** TCP vs UDP (connection-oriented vs connectionless; why TCP has slow start and setup cost), HTTP/1.1 vs 2 vs 3 (QUIC), WebSockets (bidirectional: chat, games), gRPC. Practical payoff: explaining why an API is slow, e.g. a loop that opens and closes a TCP connection per call, or 700 simultaneous connections.
2. **Web servers.** Their single job is serving static or dynamic content. Topics: caching, ETags (so client caches stay correct), edge servers, threading model (single-threaded non-blocking like Node vs multi-threaded like Apache/Tomcat). nginx is both web server and proxy.
3. **Database engineering.** ACID (each letter is a deep field; isolation alone could be a career), relational vs NoSQL (NoSQL relaxes some guarantees to scale), index structures down to LSM trees.
4. **Proxies / reverse proxies.** Load balancers, caching layers, TLS termination, L4 vs L7 proxying, service meshes (which are just proxies). His bet: proxies grow in importance with microservices.
5. **Caching.** When to cache, in-memory caches, stateful vs stateless caching, invalidation is one of the hardest problems.
6. **Messaging systems.** Queues vs pub/sub (Kafka). Systems now talk to systems, so coordination needs messaging. Preference for tools that do one thing simply; multi-purpose brokers are harder to debug.
7. **Web frameworks / API styles.** Express, Django etc.; REST is an architecture, gRPC a protocol.
8. **Message formats.** XML (human-readable, heavy for machines) -> JSON (smaller, JS-native) -> Protocol Buffers (minimise bytes on the wire for high-volume traffic).
9. **Security.** TLS, MITM and replay attacks, secrets handling (don't leak DB credentials), firewalls/open ports, DoS, SNI.

**Rules of thumb**
- Language matters less than fundamentals; they transfer.
- Ask "why is it like this and not like that?" and form defensible opinions.
- Full-stack is valid, but specialists get hired for deep problems (e.g. a write-optimised B-tree).
- Never call yourself an expert; it stops growth.

**Relevance to Sam**
- Directly relevant as the outline for the skill itself: protocols -> servers -> DB/ACID -> proxies -> caching -> messaging -> formats -> security.
- Connection reuse (HTTP sessions/pools) is the first thing to check when bulk supplier calls are slow.
- For large internal data exchange, binary/columnar formats beat JSON on bandwidth and RAM.

## Hussein Nasser — Hyper Text Transfer Protocol Crash Course - HTTP 1.0, 1.1, HTTP/2, HTTP/3 (https://www.youtube.com/watch?v=0OrmKCB0UrQ)
**HTTP is a layer-7 request/response protocol riding on a transport; each version exists to cut the cost of connections and the waiting between requests.**

- **Mental model.** Client-server, request-response. The client is usually a browser but just as often Python, a mobile app, or another backend. HTTP is the lingua franca because every stack speaks it. Underneath, TCP is a "smart bus": it carries the bytes reliably and in order but knows nothing about what they mean. Layers nest like a Russian doll: L1 light/electricity -> L2 frames with MAC addresses -> L3 IP addresses -> L4 ports (TCP/UDP) -> encryption/session -> the HTTP text.
- **Anatomy of a request:** URL (path + query), method, headers, optional body.
  - Method semantics map loosely to SQL: GET ~ SELECT, POST ~ INSERT, PUT ~ UPDATE, DELETE ~ DELETE (plus OPTIONS etc.). GET carries no body (parameters go in the URL); POST carries a body (e.g. an image with its content-type).
  - Headers: content type, cookies, `Host` (lets a layer-7 proxy route to the right backend), accept/encoding, cache directives, user agent.
- **Anatomy of a response:** status code, headers, body. 200 OK, 201 Created, 204 No Content, 302 Found (redirect), 404 Not Found. If you write your own service you own the status-code semantics, and clients depend on them being correct.
- **Demo takeaways.** Browser devtools Network tab shows the first HTML request followed by a cascade of CSS/JS/image requests that the browser discovers by parsing; ETags allow caching so repeat requests are skipped. Spinning up a static server (`npx http-server` on 8080) and calling it with `fetch` shows HTTP is just a protocol any client can speak.
- **HTTPS.** Same as HTTP but a TLS handshake comes first so both sides end up with the same symmetric key; after that, requests/responses are encrypted on the wire.

**Version evolution (why each change)**
- **HTTP/1.0 (1996).** One TCP connection per request: open, request, response, close. Rationale: RAM was tiny (~64 MB machines), and each open connection costs memory on both ends. Broke down once pages pulled many images/scripts: each resource pays TCP setup and TCP slow start. Responses were buffered whole before sending.
- **HTTP/1.1 (1997, dominant ~20 years).** Persistent connections via keep-alive: reuse one connection for many requests -> much lower latency. ETag-based caching. Chunked transfer encoding: stream parts of a response as soon as they exist (the slowly-rendering image of dial-up days). **Pipelining** (send several requests without waiting) was effectively abandoned because responses must come back in order, causing head-of-line blocking; browsers disable it.
- **HTTP/2 (from Google's SPDY; dominant by ~2015).** Multiplexing: many concurrent requests interleaved as streams on one TCP connection, removing HTTP-level head-of-line blocking. Header compression. Binary framing (note: the transcript says it uses Protocol Buffers; that's wrong, HTTP/2 has its own binary frame format and HPACK header compression). Server push (note: since deprecated/removed in major browsers). Effectively always over TLS in browsers. Protocol selection is done inside the TLS handshake via **ALPN** (the client lists supported protocols, the server picks), which replaced the older NPN.
- **HTTP/3 = HTTP over QUIC.** Keeps HTTP/2's features but swaps TCP for QUIC, which runs on UDP and re-implements reliability and congestion control in user space, avoiding TCP's in-order delivery penalty and slow handshakes. (note: the video calls it experimental; HTTP/3 was standardised as RFC 9114 in 2022 and is widely deployed. Also note HTTP/2 still suffers head-of-line blocking at the TCP layer when a packet is lost; that is the main motivation for QUIC.)
- Statelessness: HTTP looks stateless to the app even though TCP underneath is stateful; you can kill and restart a server and the client just reconnects.

**Decisions & trade-offs**
- Connection per request = simple, memory-light, but slow. Persistent connection = fast, but holds memory/file descriptors on the server.
- Buffering vs streaming: streaming gets first bytes out sooner.
- Pipelining vs multiplexing: ordered responses create head-of-line stalls; independent streams don't.

**Rules of thumb**
- Always reuse connections (keep-alive, connection pools, one HTTP/2 connection for many calls).
- Return correct status codes; clients and proxies act on them.
- Use ETags/cache headers so unchanged resources aren't re-sent.
- Stream large responses instead of building them fully in memory.

**Relevance to Sam**
- When hammering a supplier API with thousands of calls, use a single `requests.Session` / `httpx.Client` (keep-alive) or an HTTP/2-capable client; opening a fresh TLS+TCP connection per call is pure overhead and may trip rate limits.
- Streaming/chunked responses and paging matter on 16GB: process large downloads (e.g. GeoNames dumps, supplier catalogues) incrementally rather than buffering whole bodies.

## Hussein Nasser — My Process of Designing and Architecting Software (https://www.youtube.com/watch?v=TMamxmcurAQ)
**Write in order to think: workflow (plain language) -> stakeholder questions -> MVP -> technical design overview -> per-component specs -> only then simple box diagrams.**

- **First principle.** You don't think first and then write; you write and the thinking appears. Writing persists your thoughts outside your head so you can revisit and "re-index" them. The first draft is a thinking tool, not the deliverable. (Borrowed from writers like Seth Godin; he used it for his books before applying it to software.)
- **Environment.** Block 2–3 uninterrupted hours (less yields little), no meetings/email/notifications, distraction-free full-screen editor (WriteMonkey, FocusWriter).

**The process**
1. **Workflow document (non-technical).** Describe how the software is used end-to-end in exhaustive detail, "stating the obvious" down to button clicks and "the backend receives this". Stating the obvious frees creativity and, crucially, surfaces questions, because every workflow choice drives backend design and algorithms (ordered list or recommendations? read receipts? end-to-end encryption vs TLS termination at a central server?).
2. **Compile questions and meet stakeholders.** Get decisions, finish the workflow. It describes the long-term vision (where it should be in ~10 years) and becomes readable by non-technical people.
3. **Extract the MVP** from the full workflow into its own document: what ships first.
4. **Design overview (technical).** A technical rendering of the workflow: UI/UX, frontends (web, mobile), APIs, backend, databases, reverse proxies, config management, storage engine choices (row vs column store, LSM vs B-tree), eager vs lazy approaches, how it scales. Everything should trace back to a workflow item; otherwise ask whether you're inventing it. Exceptions are legitimate technical-only items (async jobs, health checks), but even those ultimately serve users (e.g. shaving latency). Guard against choosing the "latest greatest" tech for emotional reasons. Iterate with teammates.
5. **One document per component.** For each box: what it is, what it interfaces with, what it computes, what it outputs (ordering system, UI component, messaging system...). Together these form the spec.
6. **Diagrams last.** Plain boxes, arrows and labels (Google Slides is fine); skip UML notation. Possibly one diagram per complex component. Schedule periodic team reviews.

**Rules of thumb**
- No diagrams until the prose is done; writing exposes gaps diagrams hide (RFCs are mostly prose).
- Every technical component should be justifiable by a user workflow.
- Separate the vision document from the MVP document.
- Per component: inputs, interfaces, computation, outputs.

**Failure modes**
- Jumping to diagrams or tech choices before the workflow is clear.
- Tech chosen for novelty, not need.
- Unasked stakeholder questions turning into rework later.

**Relevance to Sam**
- Good template for Trameter features (booking flow, Drive Button, supplier integrations): write the user workflow first, list questions for stakeholders (e.g. what happens if the supplier confirms late?), extract an MVP, then write per-component specs (ingestion, matching, booking service) with explicit inputs/outputs. This also fits well with an AI-assisted build: the prose spec becomes the prompt.

## Hussein Nasser — Percentile Tail Latency Explained (95%, 99%) Monitor Backend performance with this metric (https://www.youtube.com/watch?v=3JdQOExKtUY)
**Min, max and mean all hide the shape of latency; percentiles tell you what fraction of requests finished under a given time.**

- **Why not min/max/average.** Min = 1 ms says nothing about the rest. Max = 300 ms can't distinguish "all requests ~299 ms" from "one outlier". Average is polluted by a few pathological requests (a handful of 10-minute requests drag a mostly 1–2 ms service's mean up).
- **Definition.** p95 = 30 ms means 95% of requests completed in 30 ms or less. The upper percentiles (p99, p99.9) are the "tail".
- **Mechanism (nearest-rank).** Sort the latencies ascending. Index = ceil(p x N), counting from 1. Example with 10 samples (1 ms ... 120 ms): p75 -> 0.75x10 = 7.5 -> 8th value = 11 ms; p90 -> 9th value = 100 ms; p99 -> 10th = 120 ms. Needs hundreds to thousands of samples to be meaningful; 10 is just for illustration.
- **Reading claims critically.** A single impressive number (p99.99 = 10 ms) is an approximation and can mislead. Ask for several: if p95 = 9 ms and p75 = 8 ms, the distribution is flat and the tail claim is less impressive than it sounds. Compare multiple percentiles to see the shape.

**Trade-offs / costs**
- Exact percentiles require sorting all samples; at millions of requests that's expensive (could push sorting to a DB). (note: production monitoring systems use streaming approximations such as histograms/HDR histograms or t-digest instead of storing and sorting every value; also percentiles can't be averaged across servers, merge histograms instead.)

**Rules of thumb**
- Report p50, p95, p99 (and max) together; set SLOs on tail percentiles, not averages.
- Be suspicious of a single headline percentile.
- Make sure the sample is large enough.

**Relevance to Sam**
- Track supplier API latency per supplier as p50/p95/p99; set your client timeouts just above a healthy p99 rather than guessing. A rising p99 with a flat p50 is an early warning of a degraded supplier.
- For batch pipelines, per-chunk processing-time percentiles spot skewed chunks (one huge country file) that stall a parallel job.

## Hussein Nasser — Primary Key vs Secondary Key - What you probably didn't know (https://www.youtube.com/watch?v=PQcIfqE85Co)
**A primary (clustered) key physically organises the table around that key; a secondary index is a separate ordered structure that points back into an unordered heap, costing an extra hop per row.**

- **The heap.** In a row store, table data lives in an area on disk (Postgres calls it the heap). Without a clustering key, rows are just appended in insertion order; no order is maintained.
- **Primary key = clustered index / index-organised table (Oracle "IOT").** The table itself is kept sorted by the key, so the table is effectively the index. Inserting 1, 8, then 2 means 2 must sit between 1 and 8. Real databases don't shift rows; they leave free space in pages for expected values, but maintaining order still costs on writes (page splits etc.).
  - Payoff: range queries. "Give me keys 1–9" is roughly one sequential read because the rows sit together. Also the full row is right there with the key.
- **Secondary index.** Table stays a "jumbled" heap; a separate B-tree holds the column values in order with pointers (row ids/tuple ids). A lookup searches the index, collects row ids, then jumps into the heap to fetch the actual rows (especially for `SELECT *`), which is random I/O.
- **Platform behaviour.**
  - MySQL (InnoDB): every table is clustered on a primary key; if you don't define one, a hidden sequential one is created.
  - Oracle and SQL Server: clustering is optional (IOT / clustered index).
  - Postgres: all indexes are secondary; tables are heaps. (note: Postgres has a `CLUSTER` command that reorders a table once by an index, but the order is not maintained on later writes. Also Hussein jokingly calls heap-organised tables "HOT"; in Postgres HOT actually means Heap-Only Tuple updates, a different concept.)

**Decisions & trade-offs**
- Clustered: fast range scans and key lookups, slower/more expensive inserts when keys arrive out of order.
- Random primary keys (UUIDv4/GUID) on a clustered table are bad: inserts land all over the B-tree, causing page splits, poor cache locality, and lost benefit from the buffer pool. Sequential keys are fine.
- Secondary index: cheap to have several, but each lookup returning many rows pays heap hops.

**Rules of thumb**
- On MySQL/InnoDB use a sequential/monotonic primary key (auto-increment, or time-ordered IDs such as UUIDv7/ULID) rather than random UUIDs. (note: in InnoDB secondary indexes store the primary key, not a row id, so a wide PK also bloats every secondary index.)
- Design primary keys around your common range queries.
- Knowing "primary key" implies physical ordering helps you predict query cost.

**Relevance to Sam**
- Big location tables (~6M GeoNames rows): if you load them into MySQL or SQL Server, cluster on a sequential id or on the key you range-scan (e.g. country + admin code), not a random hash. In Postgres, consider `CLUSTER` once after bulk load and index-only-friendly covering indexes for hot lookups.
- Booking/order tables: random UUIDs are convenient for external ids, but keep an internal sequential key for clustering.

## Hussein Nasser — Proxy vs Reverse Proxy Server Explained (https://www.youtube.com/watch?v=SqqrOspasag)
**Forward proxy: the server doesn't know the real client. Reverse proxy: the client doesn't know the real server.**

- **Forward proxy mechanism.** The client is configured to send all outbound traffic to the proxy; the TCP connection is client<->proxy, and the proxy makes the request to the destination on the client's behalf. To the destination, the proxy is the client.
  - Use cases: caching shared content for many users; anonymity (but note it just moves who knows your identity, from the destination to the proxy); logging (what sites are hot, who browses what); blocking sites (corporate phishing lists, ISPs, governments); microservice sidecars that take a simple local request (plain HTTP/1.1) and upgrade it to HTTPS/HTTP/2/gRPC towards other services.
- **Reverse proxy mechanism.** The client thinks `google.com` is the destination, but it's a front that forwards to one of many backends.
  - Use cases: **load balancing** (L3/L4/L7 variants, a deep topic); **caching** at the edge; **ingress/routing** (Kubernetes ingress: `/pictures` -> pictures service, `/comments` -> comments service, i.e. content-based routing rather than balancing); **canary deployments** (rule sends e.g. 3–8% of requests to a new version, as Instagram/YouTube do for experiments).
  - Examples: HAProxy, nginx, Envoy, Linkerd, Caddy; some act as both web server and reverse proxy.

**Common questions**
- **Both at once?** Yes: a service-mesh sidecar (Linkerd, Istio, Envoy) is a proxy for the app's outbound calls and a reverse proxy for its inbound ones. It moves networking concerns (timeouts, retries, circuit breaking, TLS) out of app code.
- **Proxy vs VPN for privacy.** A VPN works at a lower layer; with HTTPS sites it sees only the domains you visit. A **TLS-terminating** proxy decrypts your traffic and re-encrypts with its own certificate, so it sees everything. Check the padlock: the certificate should belong to the real site, not the proxy. Either way you transfer trust to the operator.
- **Only HTTP?** No: SOCKS proxies (low-level tunnelling), HTTP proxies that tunnel via the `CONNECT` method (end-to-end TLS preserved, safer), or TLS-terminating proxies (appropriate for your own reverse proxies, not for your browsing).

**Rules of thumb**
- Put cross-cutting network concerns (TLS termination, retries, timeouts, routing, canaries) in the proxy layer, not every service.
- TLS termination belongs at a reverse proxy you control; distrust forward proxies that terminate TLS.
- Remember the proxy is a client to your backend: it has its own timeouts and retries (see the timeouts video).

**Relevance to Sam**
- Outbound to suppliers: a single egress/forward proxy or client wrapper can centralise logging, caching of static supplier content (catalogues, location lists), rate limiting and a stable egress IP for suppliers that allow-list IPs.
- Inbound: a reverse proxy (nginx/Caddy) in front of Trameter APIs handles TLS, routing by path, and canary rollouts of new booking-flow versions.

## Hussein Nasser — Publish-Subscribe Architecture (Explained by Example) (https://www.youtube.com/watch?v=O1PgqUqZKTA)
**"REST vs message queue" is really request-response vs publish-subscribe: request-response is simple for two parties but breaks on long chains of services; pub/sub decouples them at the cost of hard delivery guarantees and broker complexity.**

- **Request-response model.** Client always initiates; it waits (possibly asynchronously, spending no CPU) for the reply. Elegant, simple, the web runs on it.
- **Where it breaks: the upload pipeline example.** Upload -> compress -> format (480p/720p/1080p/4K) -> notify subscribers (only once the video is actually consumable). Done with chained synchronous calls, each service waits on the next; the client waits on the whole chain. Any network error mid-chain breaks everything and leaves you not knowing what finished. Adding a copyright-check service that also needs the compressed file means rewiring the topology; services multiply connections and know about each other.
- **Request-response pros:** elegant and simple for two parties; stateless HTTP allows horizontal scaling of identical receivers behind a load balancer (note: "stateless" doesn't apply to e.g. a DB SQL request-response, which is very stateful).
- **Cons:** poor fit for multiple receivers; high coupling ("software should have social anxiety": we want services oblivious of each other); both ends must be online simultaneously; teams compensate with chained timeouts, retries and circuit breakers (or service meshes / client libraries like Twitter's Finagle), which add load and complexity.

**Pub/sub mechanism**
- A middle layer (broker / message queue / streaming platform) holds named **topics** (Kafka), **queues** (RabbitMQ), or **channels** (Redis). Anyone can publish and subscribe.
- Upload service receives the video (still request-response with the client, a hybrid is fine), publishes "raw video" to a topic over the fast local network, then immediately tells the client "done". Client can disconnect.
- Compress service subscribes to raw-video, publishes compressed-video; format service subscribes to that and publishes per-resolution topics; notification service subscribes to whichever resolution should trigger the alert; copyright service just subscribes to compressed-video with no changes to anyone else.
- **Pros:** scales to many distinct consumers with different needs; ideal for microservices (avoids the everything-connects-to-everything mesh; the broker becomes the one dependency); loose coupling so you can change one piece safely; consumers can be offline and catch up because messages are stored.
- **Cons:**
  - **Publish delivery:** the publisher needs the broker's ack. No ack -> retry -> possible duplicate. Needs idempotency, and it's unclear whether publisher or broker should own dedup.
  - **Consume delivery (the hard part):** how does the broker know the consumer received and processed a message? Acks can themselves be lost; this is genuinely hard (exactly-once is an area of research).
  - **Single point of failure** at the broker (mitigated with replication).
  - **Complexity of delivery models:**
    - *Push* (RabbitMQ over AMQP, Redis over RESP; both need a persistent two-way connection): near real-time, but broker must track who is online and hold messages for offline consumers; **back pressure**: a fast producer floods a slow consumer, forcing the broker to buffer, time out or throttle per consumer. Also network saturation and broker retries.
    - *Polling*: consumer repeatedly asks "anything for me?"; wastes requests and network on empty replies.
    - *Long polling* (Kafka): consumer's request is held open until data exists; costs only memory while waiting, consumer controls its pace, solving back pressure, at some loss of real-time immediacy.

**Decisions & trade-offs**
- Two services, quick operation -> request-response.
- Long-running multi-step workflows, fan-out to several independent consumers, consumers that may be down -> pub/sub.
- Push for latency; pull/long-poll to let consumers control rate.

**Rules of thumb**
- Acknowledge the client as soon as the work is durably handed off, not when it's complete.
- Assume at-least-once delivery: make every consumer idempotent and every publish deduplicable.
- Plan back pressure explicitly.

**Relevance to Sam**
- Booking flow: accept the booking request, persist + publish "booking requested", respond with a booking id and pending status; separate consumers call the supplier, take payment, send confirmation emails. A supplier outage then delays work instead of failing user requests.
- Data pipeline: stages (download -> clean -> match -> enrich -> load) can communicate via topics/queues so a slow stage applies back pressure rather than exhausting RAM. Even locally, bounded queues between process pools give the same effect on a 16GB laptop.
- Duplicate publishes/consumes are normal: use idempotency keys (booking reference, row hash) so a re-run doesn't double-book or double-insert.

## Hussein Nasser — Relational Database ACID Transactions (Explained by Example) (https://www.youtube.com/watch?v=pomxJOFVcQs)
**A transaction is a unit of work; atomicity and isolation together produce consistency, durability makes commits survive crashes, and every strengthening costs performance, so decide what you can give up.**

- **Transaction.** One or more queries treated as one unit, bounded by BEGIN and ended with COMMIT (persist) or ROLLBACK (abort). Example: transfer $100 from account 1 ($1000) to account 2 ($500): SELECT balance (>= 100?), UPDATE acct1 -100, UPDATE acct2 +100, COMMIT. Before commit, changes are "dirty".
- **Atomicity.** All queries succeed or all are rolled back. Without it, a crash after debiting acct1 but before crediting acct2 leaves $900 + $500: $100 vanished. On restart the database must roll back the partial work.
- **Isolation.** Can my in-flight transaction see changes made by other concurrent transactions? It's a knob. Lack of isolation produces **read phenomena**:
  - *Dirty read*: reading another transaction's uncommitted change, which may later roll back. Sales report example: line items sum to $130 but a concurrent uncommitted +5 units makes the total query return $155.
  - *Non-repeatable read*: reading the same value twice in one transaction and getting different committed values (a legitimate commit happened in between). Happens in real code: e.g. reading rows for detail lines then aggregating them in a second query.
  - *Phantom read*: a range query picks up a row inserted and committed by another transaction mid-way (new product row makes the total $140 vs lines $130). Hard to prevent with row locks because the row didn't exist to be locked.
  - *Lost update*: two transactions modify the same value; one overwrites the other before commit.
- **Isolation levels (fastest/weakest -> slowest/strongest).**
  - *Read uncommitted* (note: the video mislabels this "read committed" at first): no isolation; all phenomena possible.
  - *Read committed*: each query sees data committed as of that query's start. No dirty reads, but non-repeatable and phantom reads possible. Default in most databases and often good enough.
  - *Repeatable read*: the transaction sees only data committed before the transaction started. Implemented either with **versioning / snapshots** (MVCC: you read "version 0" while others write v1, v2) or with **shared read locks** that block writers (more expensive). Phantoms possible in the textbook table, but versioning implementations can avoid them. (note: Hussein cites Cassandra and RocksDB here; the mainstream examples are Postgres and MySQL InnoDB, which use MVCC. Postgres's repeatable read is snapshot isolation and does prevent phantoms.)
  - *Serializable*: as if transactions ran one after another; no phenomena; slowest. Databases parallelise where they safely can; serialization failures must be retried (note: added). Applications sometimes implement it themselves with explicit locks.
- **Consistency, two kinds.**
  1. *Consistency in data*: user-defined invariants (sum of balances, like counter equals rows in likes table), enforced by keys/foreign keys and as a by-product of atomicity + isolation. Some systems choose to tolerate drift: Instagram like counts on a 5M-like post or YouTube subscriber counts shown as "6.2K" are approximations; nobody verifies, and performance wins.
  2. *Consistency in reads*: after a commit, does every later reader see it? With one server, yes. Add read replicas (leader/follower, async replication) and **both relational and NoSQL systems** can serve stale values. "Eventual consistency" just means a later read will catch up. YouTube's Vitess (sharding over MySQL) mentioned as a scaling example.
- **Why NoSQL exists.** Relational databases were designed (1970s) for one big machine and are hard to scale horizontally. NoSQL systems relaxed some guarantees (consistency, schema, referential integrity, sometimes isolation/transactions) to scale out by adding nodes.
- **Durability.** Once committed, data survives power loss (written to non-volatile storage). Pure in-memory caches like Memcached skip durability by design. (note: Redis does offer optional persistence via RDB snapshots/AOF; and modern MongoDB has secondary indexes and multi-document transactions, unlike what the video implies.)

**Decisions & trade-offs**
- Each step up in isolation reduces concurrency/throughput. Pick the weakest level that preserves your invariants.
- Approximate counters are fine for likes/views; never for money or inventory.
- Read replicas buy read scale at the cost of stale reads; route read-your-own-write paths to the primary.

**Rules of thumb**
- Wrap multi-statement changes that must stay in step (debit + credit, booking + inventory decrement) in one transaction.
- Know your DB's default isolation (Postgres/Oracle/SQL Server: read committed; MySQL InnoDB: repeatable read).
- Reports that run several queries and must agree -> repeatable read/snapshot.
- Check-then-act (SELECT balance then UPDATE) is a race under read committed: use `SELECT ... FOR UPDATE`, a conditional `UPDATE ... WHERE balance >= 100`, or serializable with retry (note: added detail).

**Relevance to Sam**
- Bookings/payments: the check-availability -> reserve -> charge sequence is the classic lost-update/double-booking risk. Use row locks or conditional updates and a unique idempotency key so retries can't create a second booking.
- Bulk loads: load a chunk inside one transaction so a crash mid-chunk leaves no half-written batch (atomic per chunk, restartable from the last committed chunk).
- If reads go to replicas, a user who just booked may not see the booking immediately; read from the primary for confirmation pages.

## Hussein Nasser — Synchronous and asynchronous workloads are everywhere (https://www.youtube.com/watch?v=Vf47Jyt_hqc)
**Sync = caller blocked until the work fully completes; async = caller gets a placeholder (promise/job id/"ok") immediately and the real result later. The same trade (speed now vs certainty now) appears in code, backends, replication, commits and the OS disk cache.**

- **Definitions.** Synchronous: you are blocked for the entire operation, unblocked only when it fully completes. Asynchronous: the call returns at once with a promise/future; the work runs elsewhere (idle time on the same thread as in Node, another thread, another process, another machine) and you're notified/called back.

**Five places it shows up**
1. **Programming.** Old VB6/C# code froze the UI on blocking calls (hacks like `DoEvents`). Then callbacks (unreadable nesting) -> promises -> async/await (code looks blocking, thread is free to handle clicks, timers, animation while waiting on a REST call).
2. **Backend execution / "asynchronous services".** Async on the client side doesn't make the backend async: the server still executes the request while the connection waits. Failures then fall on the client to retry, which doesn't scale. Solution: the backend enqueues the job (a millisecond operation), returns a job/queue id immediately, and a worker executes later; the client polls ("done yet?"), long-polls, or receives a push. Failures are retried on the backend without bothering the client.
3. **Message queues / pub-sub.** YouTube upload: return the video id right away, then compress, transcode (480p–4K), content-ID/copyright checks via topics (e.g. Kafka) consumed by different workers. Doing it synchronously would make the upload fail if any step fails.
4. **Database replication.**
   - *Synchronous replication*: the primary applies the write, sends it to all replicas, waits for every success, then acknowledges the client. Strong consistency, slow writes.
   - *Asynchronous replication*: acknowledge once the primary's WAL write succeeds; replicas catch up later. Fast writes, stale reads.
   - Real case (YouTube 2006–07, told by Sugu Sougoumarane, creator of Vitess): users updated their profile, refreshed, read from a replica, saw the old data. Fix: for a client that just wrote (identified by IP), route its reads to the primary ("sticky"/read-your-writes). Cheap and effective.
5. **Commits (Postgres `synchronous_commit`).** A commit promises durability: if success is returned, pulling the power a millisecond later must not lose it. Synchronous commit waits until the WAL is physically on disk. Turning it off makes commits return before the flush: much faster, but a crash can lose recently "committed" transactions. Hussein disables it on a test database that is reloaded daily, where loss doesn't matter. (note: in Postgres, turning `synchronous_commit` off risks losing the last few hundred ms of commits but not corruption; turning `fsync` off risks corruption. Also, his claim that Postgres writes all transaction data to disk as it goes is loose; what's flushed at commit is the WAL, data pages are written later by checkpoints/background writer.)
6. **OS page cache / fsync.** `write()` usually lands in the OS cache, which batches and flushes later (ms to hundreds of ms); reads are served from that cache so it looks written. Batching turns 10 updates to the same page into 1 physical I/O and spares SSD wear. Databases can't accept that risk for the WAL, so they force it to disk with `fsync` (or bypass the cache); data/index pages can be lazier because the WAL can rebuild them.

**Decisions & trade-offs**
- Sync: simple, strongly consistent, slower, failures surface to the caller.
- Async: fast, scalable, but you must handle "accepted but not done", polling/notification, stale reads, and possible loss if acknowledged before durable.
- Choose per use case: likes count stale is fine; bank balance isn't.

**Rules of thumb**
- Long-running backend work -> enqueue, return a job id, let clients poll/subscribe.
- After a user's own write, serve their reads from the primary (read-your-writes).
- Only relax durability (`synchronous_commit=off`, no fsync) for data you can regenerate.

**Relevance to Sam**
- Bulk loading millions of rows into a local Postgres for transformation: setting `synchronous_commit=off` (and using `COPY`, unlogged staging tables) for reloadable staging data can massively speed up loads; never do it for bookings.
- Supplier booking calls: expose your booking endpoint as an async service (return a booking id, status "pending"), do the supplier call in a worker with retries, and let the frontend poll or receive a webhook.
- If you add read replicas, route post-booking confirmation reads to the primary.

## Hussein Nasser — Synchronous vs Asynchronous Applications (Explained by Example) (https://www.youtube.com/watch?v=N5Ky-mz6n-8)
**A synchronous client waits for the answer before running the next line; an asynchronous client hands over a callback and keeps going. The visible difference is whether the user is frozen.**

- **Synchronous client.** Calls the function and sits there; code below can't run until the result returns. Example: a video upload app that greys out and freezes the whole UI until the upload finishes, then prints "success" with the URL.
  - Pro: trivially readable top-to-bottom code.
  - Con: blocked UI, bad UX; users today don't tolerate it.
- **Asynchronous client.** Makes the call, registers a callback ("call this when done, success or failure"), and continues. The upload example immediately gets back a video URL while the bytes are still uploading, so the user can edit title, description and tags meanwhile (how YouTube behaves).
  - Mechanism behind the instant URL: the server quickly creates a record and returns its id first (that request itself is short and synchronous); the heavy upload/processing keeps running in the background.
  - Pro: responsive UI, more work done in parallel.
  - Con: code is harder to follow (callbacks), now much eased by promises and async/await (ES6+), which read like synchronous code.

**Rules of thumb**
- Return an identifier as early as possible (create the record first), then do the heavy work; lets users and downstream code proceed.
- Use async/await rather than raw callbacks for readability.
- Keep long work off the UI/request thread.

**Relevance to Sam**
- Same "id first, work later" pattern for bookings and long data jobs: create the booking/job row, return its id, process in the background, expose status. Also useful for Trameter frontends that call slow supplier searches: render the page and stream/poll results in rather than blocking.

## Hussein Nasser — TCP vs UDP Crash Course (https://www.youtube.com/watch?v=qqRYkcta6IE)
**TCP buys reliability, ordering and fairness with headers, waiting and per-connection state; UDP is a thin, stateless, fire-and-forget datagram layer you build only what you need on top of.**

- **Addressing.** Both are layer-4 transports. IP address identifies the machine (or VM/container); port identifies the application on it (80 HTTP, 25 SMTP, 23 telnet, 22 SSH). IP + port = an endpoint.

**TCP (Transmission Control Protocol) features, i.e. pros**
- **Acknowledgments:** physical links are unreliable (cut cables, interference, flipped bits), so segments carry sequence numbers and the receiver acks them.
- **Guaranteed delivery:** no ack or corrupted data -> retransmit until acknowledged.
- **Connection-based:** a handshake establishes a connection whose state lives on both client and server (needed to track acks, retransmits, ordering).
- **Congestion control:** like freeway on-ramp metering lights in LA, TCP holds back when the network is busy so it isn't flooded.
- **Ordered delivery:** packets take different routes and arrive out of order; TCP numbers and reorders them.

**TCP costs, i.e. cons**
- Bigger headers (at least ~20 bytes) on every segment, even a 1-byte payload -> more bandwidth; hurts on 2G/3G.
- Slower: waiting for acks, congestion windows, reordering, retransmits.
- **Stateful:** stateless means you can restart the server and clients carry on unaffected. With TCP, killing the server kills the connection ("connection closed by foreign host" in the telnet demo); in-flight data is lost and clients must reconnect.
- **Server memory per connection:** each connection consumes kernel memory and must be tracked, which caps concurrent connections.
- **DoS exposure:** an attacker opens connections and never completes/acks them (SYN-flood style); the server holds half-open state until timeouts, multiplied by millions of attempts (Yahoo was hit like this in the early 2000s).

**UDP (User Datagram Protocol)**
- Cons: no acks, no retransmission, only a checksum to detect corruption (drop it or pass it up); connectionless (no handshake, the server doesn't know who's talking, so firewalls often block UDP and a known IP:port can be flooded); no congestion control (keeps sending regardless); no ordering.
- Pros: small header (8 bytes) -> less bandwidth; faster (no waiting); **stateless** (server restart is invisible to clients, which simply keep sending); easy horizontal scaling (round-robin across many identical UDP servers since no connection is pinned).
- Apps can layer "reliable UDP" (their own acks/ordering only where needed), as games in low-bandwidth regions do; QUIC is the big modern example.

**Demo notes (Node.js)**
- TCP: `net.createServer`, socket `on('data')`; telnet connects, receives "hello"; "hi" arrived as 4 bytes (h, i, CR 13, LF 10).
- UDP: `dgram.createSocket('udp4')`, `on('message')`; `echo hi | nc -u -w1 127.0.0.1 8081` sends with no connection; server can restart between sends without the client noticing.

**Decision guide**
- TCP: anything where every byte and order matter: databases (SQL statements and results), chat/text, HTTP/1–2, file transfer. Header overhead is negligible for small text messages.
- UDP: real-time media and streaming (a garbled frame is tolerable, waiting isn't), games, DNS (small stateless lookups that scale easily). (note: the transcript expands DNS as "dynamic name server"; it's Domain Name System, and DNS falls back to TCP for large responses/zone transfers.)

**Rules of thumb**
- Connections are a resource: each holds memory; pool and reuse them, and cap them.
- Statefulness limits restartability and scaling; keep app servers stateless where you can.
- Handshake costs mean short-lived connections are expensive.

**Relevance to Sam**
- Every supplier/DB call rides TCP: opening a new connection per row in a big loop pays a handshake (plus TLS) each time. Use pooled sessions and batch requests.
- Too many concurrent connections from a 16GB laptop (or to a Postgres server) exhausts memory/file descriptors; bound concurrency with semaphores/pool sizes.

## Hussein Nasser — The OSI Model - Explained by Example (https://www.youtube.com/watch?v=7IS7gigunyI)
**Each layer wraps the one above in its own header (Russian dolls) on the way out and strips it on the way in; knowing which layer adds what tells you what each device and attacker can see.**

- **Setup.** A LAN with a router, machines with private IPs (10.0.0.x) and MAC addresses (48-bit network-card ids). A phone at 10.0.0.5 requests `http://10.0.0.3:80/` from a web server.

**Sending side, top to bottom**
- **L7 Application:** the browser builds the HTTP GET request as a big string: method, path, host, headers, cookies (no body for GET).
- **L6 Presentation:** encryption if used (TLS); otherwise pass-through. After this the bytes are opaque to lower layers.
- **L5 Session:** associates the data with a session/connection so many concurrent connections to the same server can be told apart. (note: in TCP/IP practice there's no separate session-layer id; a connection is identified by the 4-tuple of source IP, source port, destination IP, destination port.)
- **L4 Transport:** splits the data into **segments**, each tagged with destination port (80) and an auto-chosen source port, plus sequence numbers so the receiver can reorder.
- **L3 Network:** wraps segments into **packets** with source and destination IP. Best effort; no error checking of its own, its job ends when the packet reaches the IP.
- **L2 Data link:** cuts packets into **frames** with source and destination MAC and basic error detection. If the sender knows only the IP, **ARP** (Address Resolution Protocol) asks "who has 10.0.0.3?" and caches the MAC.
- **L1 Physical:** bits encoded into electricity, light, or radio. The big enabler: the ability to sneak 1s and 0s into a signal.

**Receiving side, bottom to top**
- On a shared medium (Wi-Fi, hubs) every device receives every frame; the network card discards frames not addressed to its MAC. (note: switched Ethernet forwards frames only to the right port; Wi-Fi remains shared.) Tools like Wireshark keep everything instead, which is why unencrypted traffic on public Wi-Fi is readable.
- L2 strips frame headers and reassembles; L3 confirms the IP is "me" (else routing applies); L4 hands the payload to whichever app listens on port 80 and remembers the source IP:port to reply to; L5 matches the connection; L6 decrypts; L7 the web framework (e.g. Express) finally sees a GET.
- Terminology: segment (L4) -> packet (L3) -> frame (L2); one segment may span several packets/frames.

**Rules of thumb**
- Map every problem to a layer: DNS/HTTP/app errors (L7), TLS (L6), ports and connection limits (L4), routing/IP (L3), Wi-Fi/MAC (L2).
- Encrypt in transit (TLS/VPN); anything below the encryption layer can be sniffed in plaintext otherwise.
- Load balancers and proxies are described by the layer they read (L4 sees IP/port; L7 sees HTTP).

**Relevance to Sam**
- Debugging supplier integrations: a connection refused/timeout is L3/L4 (firewall, IP allow-list, port), a TLS error is L6, a 4xx/5xx is L7. Knowing which tells you whether to call the supplier's network team or fix your request.
- Never send API keys or payment data over plain HTTP, especially from shared networks.

## Hussein Nasser — Transport Layer Security, TLS 1.2 and 1.3 (Explained by Example) (https://www.youtube.com/watch?v=AlE5X1NlHgg)
**TLS's handshake exists to give client and server the same symmetric key without ever sending it; TLS 1.3 forces ephemeral Diffie-Hellman, which is both faster (fewer round trips) and safer (forward secrecy) than TLS 1.2's RSA key transport.**

- **Plain HTTP baseline.** Port 80 over TCP; open connection, GET, server processes (the latency you as an engineer try to minimise), response headers and body, connection kept alive in HTTP/1.1. HTTP itself is stateless; TCP underneath is stateful.
- **HTTPS.** Identical, but port 443 and a handshake before the first request. Goal: both sides hold the same **symmetric** key (e.g. AES), because symmetric crypto is far faster than asymmetric. Afterwards the GET and the response are encrypted; TCP just carries opaque bytes. Encryption/decryption adds some processing cost; know it's there.

**TLS 1.2 handshake**
- ClientHello: supported key-exchange methods (RSA, Diffie-Hellman...) and symmetric ciphers (AES, DES, Blowfish...); varies with how old the client is.
- Server picks algorithms and sends its **certificate** (contains its public key plus identity info).
- With RSA key exchange: client generates the pre-master secret (input to the symmetric key), encrypts it with the server's public key, sends it. Server decrypts with its private key. Both derive the key; "finished"; start encrypted traffic.
- **Problems:** (1) The key material itself travels on the wire (encrypted). If the server's private key ever leaks, every recorded past session can be decrypted. That has happened: Heartbleed (unpatched OpenSSL) let attackers read server memory, including private keys. (2) Many round trips before the first useful byte; slow. (3) Too many options, including insecure ones; security depends on configuration. (note: Hussein counts "four round trips"; a full TLS 1.2 handshake is 2 round trips on top of TCP's.)

**Diffie-Hellman intuition (colour analogy)**
- Three ingredients: client's private value (never sent), server's private value (never sent), a public value. Combining all three yields the shared secret.
- Combining a private value with the public one produces something safe to send, because separating them again is computationally infeasible.
- So each side sends "public + my private"; each side adds its own private to what it received; both arrive at the same secret, which never crossed the wire.
- **Ephemeral** DH (DHE/ECDHE): the private values are generated fresh per session and thrown away, so stealing the server's long-term key later can't decrypt old sessions (forward secrecy).

**TLS 1.3 handshake**
- Removes the options: key exchange is always ephemeral Diffie-Hellman; only the symmetric cipher is negotiated.
- Client sends, in its first message, its public parameter and its combined (public+private) value. Server generates its private value, computes the secret immediately, replies with its own combined value; client computes the same secret. One round trip and encrypted data can flow. (note: the video says both "one round trip" and "two round trips"; TLS 1.3 is 1-RTT for a new connection, with optional 0-RTT resumption that carries replay risks. The server still sends its certificate in 1.3, now encrypted, to authenticate itself; the video skips authentication.)
- SSL and TLS 1.0/1.1 are obsolete and insecure.

**Rules of thumb**
- Prefer TLS 1.3; disable SSL, TLS 1.0/1.1; avoid static-RSA key exchange in 1.2 (use ECDHE suites).
- Protect and rotate private keys; patch TLS libraries quickly.
- Reuse connections so you pay the handshake once (keep-alive, pooling, session resumption).

**Relevance to Sam**
- Every HTTPS call to a supplier pays TCP + TLS setup. In bulk jobs (thousands of availability/price calls), connection reuse saves a full round trip or more per call, which can dominate total runtime when suppliers are geographically far from Lagos.
- Don't disable certificate verification to "fix" TLS errors in Python clients; fix the CA bundle instead.

## Hussein Nasser — What are web servers and how do they work (with examples httpd and nodejs) (https://www.youtube.com/watch?v=JhpUch6lWMw)
**A web server is software that accepts TCP connections and answers HTTP requests with static files or generated content; the interesting engineering is in how it uses threads and memory per connection.**

- **What it serves.** "Web content" = anything sent over HTTP: HTML, images, PDFs, CSS, JS, JSON from APIs.
  - **Static content:** files on disk (HTML, images, CSS, even JS files). Drop a file in a folder and the server maps it to a URL. Highly cacheable; changes are detected with ETags.
  - **Dynamic content:** generated per request by backend code (DB queries, per-user feeds like Facebook, per-region look). Caching is tricky because each response can differ.
- **Ready-made vs build-your-own.** Ready-made: Apache httpd (tiny C binary, ~361 KB package), IIS (ASP.NET), Tomcat (Java, heavy but does much more), lighttpd, `http-server`. Build-your-own with frameworks: Node.js/Express, Python Tornado. They give you the raw listen/request/response loop, and you implement routing, file serving, caching, status codes yourself.

**How a request is handled**
- Server listens on a port (80 HTTP, 443 HTTPS by default). Client opens TCP (+TLS); the server allocates memory for a **socket per client**. That memory reservation is what DoS attacks exploit.
- The request (GET /index.html) is processed: from cache, disk, or DB; then status code + headers + body go back. With keep-alive, the next request reuses the connection (fast), but idle open connections still hold server memory.
- **Single-threaded blocking server (simplest model).** One process, one thread. Many sockets can be open and idle in memory, but only one request runs at a time. While a POST is being processed, a newly connected client gets a socket but waits. Long-running work needs a cut-off (he calls it circuit breaking) so the thread is released.
- **Thread-per-request (Apache/Tomcat).** Spawns a thread per request/connection; hence config knobs like max threads / max connections, bounded by memory.
- **Alternative scaling.** Run many simple single-threaded servers (e.g. containers) behind a load balancer; Hussein finds this simpler than one complex multi-threaded server. No settled right answer.

**Demos**
- Apache on a Raspberry Pi (Debian): `apt-get install apache2`, serves `/var/www/html` on port 80; `chown` the folder to edit. Adding `main.js` referenced from `index.html` shows a second request on the same kept-alive connection. Apache handles ETags automatically: unchanged file -> **304 Not Modified** (client sent `If-None-Match`); after editing the file -> new ETag and full 200 response.
- Express: `app.listen(8080)` (ports below 1024 need root). With no routes, everything is 404 "Cannot GET". Add `app.get('/', (req,res)=>res.sendFile(path.join(__dirname,'index.html')))` (use absolute paths to avoid path-traversal); `app.get('/sup', ...)` returns text; you control `res.status(...)` freely (201 Created, 404, even 418 I'm a teapot). Freedom means responsibility: caching, correct codes, security are on you.

**Decisions & trade-offs**
- Ready-made server: caching, ETags, static serving, hardening for free; less control.
- Framework/own server: total control, but you must reimplement caching, status semantics, and safe file serving.
- Concurrency model: single-threaded (simple, needs scaling out) vs thread-per-request (simple code, memory-bound) vs event-loop non-blocking (efficient for I/O).

**Rules of thumb**
- Serve static assets from a real web server/CDN, not from your app code.
- Cap max connections/threads to what memory allows.
- Use absolute paths and whitelists when serving files.
- Return honest status codes; 304 + ETag saves bandwidth.

**Relevance to Sam**
- When exposing a Trameter data or booking API, put nginx/Caddy in front for static files, TLS and caching; keep the Python app for dynamic logic.
- On a 16GB machine, a thread-per-request server with high max-threads can exhaust memory; prefer async frameworks (FastAPI/uvicorn) for I/O-heavy endpoints that wait on suppliers.

## Hussein Nasser — What happens when you type google.com into your browser and press enter? (Detailed Analysis) (https://www.youtube.com/watch?v=dh406O2v_1c)
**A single Enter key triggers: URL-vs-search parsing, protocol/port choice (HSTS), layered DNS lookup over UDP through ARP and NAT, a TCP handshake, a TLS 1.3 handshake with SNI and ALPN, then HTTP/2 streams and browser parsing that fans out more requests.**

Context: fresh machine and browser, first visit ever, late-2019 browser.

1. **Typing.** Each keystroke may trigger autocomplete from history, a local index, or a call to the default search engine.
2. **URL or search term?** The browser parses "google.com" and decides it's a URL to visit.
3. **Protocol and port.** No scheme was typed. Defaulting to HTTP enables man-in-the-middle and SSL-stripping attacks, so browsers ship an **HSTS** (HTTP Strict Transport Security) preload list: if the domain is on it, go straight to HTTPS on 443; otherwise HTTP on 80.
4. **DNS (the most involved step).** Lookup order: browser DNS cache -> OS (cache and the hosts file) -> configured resolver. Classic DNS is unencrypted UDP on port 53, so your ISP/employer can see every domain you visit (not page contents if HTTPS and no TLS-terminating proxy). **DoH** (DNS over HTTPS) and **DoT** (DNS over TLS) encrypt it; tension between network admins (who want to see/separate DNS traffic) and privacy advocates; Hussein leans DoH. It was off by default then.
   - Packet path, with example addresses (client 10.0.0.2, MAC AA; gateway 10.0.0.1, MAC FF; resolver 1.1.1.1; router public IP 44.1.2.4): build an IP packet dst 1.1.1.1:53, src 10.0.0.2:random port. Resolver isn't in the local subnet, so the frame goes to the **gateway's MAC** (learned via ARP). The router does **NAT**: rewrites source to its public IP, records a NAT-table entry (internal ip:port <-> external), forwards. The reply (google.com = 4.1.2.3) comes back to the router's public IP/port, which uses the NAT table to forward it to 10.0.0.2.
5. **TCP connection.** 3-way handshake to 4.1.2.3:443 from a new random source port; same gateway/ARP/NAT path; a second NAT entry. Variations: a configured HTTPS/SOCKS proxy changes the destination to the proxy; with HTTP/1.1 the browser opens ~6 parallel connections per host (note: the video calls this pipelining; it's really just multiple connections, pipelining is a different, abandoned feature).
6. **TLS 1.3 handshake (1 round trip).** ClientHello carries: the client's Diffie-Hellman key share (public value + combined value), supported symmetric ciphers, **ALPN** (I speak h2, maybe h3), and **SNI** (the hostname, because one IP may host many domains and the server must pick the right certificate). Server computes the shared secret, picks the cipher, replies with its key share, the matching certificate and protocol choice. Client derives the same secret. Beware downgrade attacks. Chrome may use QUIC with Google instead.
7. **HTTP request.** GET / with Host header and any cookies; under HTTP/2 the headers are compressed, framed as a binary stream, encrypted with the symmetric key, and sent over the existing TCP connection.
8. **Server side.** Google's load balancer: an L4 balancer just forwards the stream to a backend; an L7 balancer terminates TLS and inspects the HTTP request. Server builds a response: Content-Type, Set-Cookie (tracking), HTML body; compresses, encrypts, sends to the router's public IP:port; NAT maps it back.
9. **Browser rendering.** Decrypt, read **Content-Type** and act on it (parse HTML, render image, execute JS). Missing content types led browsers to "MIME sniff" the body, a source of attacks; servers now send `X-Content-Type-Options: nosniff`. Parsing the HTML discovers CSS/JS/images; with HTTP/2 each gets its own stream on the one connection, in parallel. **HTTP/2 push** could send them before being asked (note: since deprecated in major browsers). With HTTP/1.1, parallelism comes from ~6 connections.

**Rules of thumb**
- Latency budget of a cold first request: DNS + TCP + TLS + request + processing. Caches (DNS, connections, TLS sessions) remove most of it on later requests.
- Always set correct Content-Type and `nosniff` on APIs.
- Use HSTS on your own domains.
- SNI is why many HTTPS sites can share one IP.

**Relevance to Sam**
- Mental checklist for "this supplier call is slow on first request": DNS resolution, TCP connect, TLS handshake, then server time. Reusing a session/connection pool removes the first three for subsequent calls; Python clients also benefit from caching DNS.
- When calling suppliers through corporate proxies or VPNs, remember the proxy changes the connection path (and possibly TLS termination).

## Hussein Nasser — backend leaking Postgres database connections - what I did to fix it (https://www.youtube.com/watch?v=KGbwkbaCwss)
**A backend that hangs after N requests is often a connection pool leak: connections checked out for a transaction were never released, so later requests queue forever waiting for the pool.**

- **Symptom.** A to-do API (Express + Postgres via `pg`) served exactly 4 requests, then every request hung. Browser devtools showed requests "pending"; in production nginx (reverse proxy) returned a timeout after its configured 60 s, which told him the backend was reachable but not answering, rather than down. Restarting fixed it for another 4 requests: a consistent, reproducible failure, which is what makes debugging easy.
- **Root cause.** For a transaction he did `client = await pool.connect()`, `BEGIN`, queries, `COMMIT`, returned the response, but never called `client.release()`. The pool (max = 4) believed all 4 connections were still in use; request 5 waited in the pool queue indefinitely. Node is a single long-lived process, so nothing reset the pool's state.
- **Why pools exist.** Without a pool, each request opens a new DB connection: TCP handshake + Postgres auth/connection setup + TLS if enabled, then tears it down. Pools pre-create ("preheat") a bounded set and lend them out. Never create DB connections inside the request path.
- **Lucky detail.** He found it fast only because max was set to 4. With the default (around 10–20; he recalls 20) the leak might have gone unnoticed until production load. (note: node-postgres's default `max` is 10.)

**Fixes and safeguards**
- Always release the client, ideally in a `finally` block so errors also release it.
- Configure pool timeouts: connection-acquire timeout (`connectionTimeoutMillis` in node-postgres) so a request fails fast instead of hanging; idle timeout (how long unused connections live); statement/query timeouts.
- For single statements that don't need a transaction, use `pool.query(...)`: it checks out a connection, runs, and returns it automatically.
- **Gotcha:** don't build a multi-statement transaction from consecutive `pool.query` calls. Each call may run on a different connection and auto-commits individually; BEGIN/ROLLBACK won't cover the others, so it isn't atomic. Transactions require one dedicated client from `pool.connect()`.
- Also noted: an unbounded `SELECT *` of all to-dos is a bad query; paginate/limit.

**Rules of thumb**
- "Works for N requests, then hangs" -> suspect leaked pooled resources (DB connections, HTTP connections, file handles, semaphores).
- Set explicit, small pool sizes in dev to surface leaks early.
- Every acquire has a guaranteed release (try/finally, context managers).
- Time out on pool acquisition, not just on queries.

**Relevance to Sam**
- Python equivalents: SQLAlchemy/psycopg pools. Use `with engine.begin() as conn:` or `with pool.connection() as conn:` so connections return even on exceptions; set `pool_timeout`, `pool_size`, statement timeouts.
- Parallel pipelines: each worker process needs its own pool (connections can't be shared across forked processes), and total connections = workers x pool size must stay under Postgres `max_connections`.
- Same leak pattern applies to supplier HTTP sessions and semaphores guarding rate limits: always release in `finally`.

## ByteByteGo — 20 System Design Concepts You Must Know - Final Part (https://www.youtube.com/watch?v=uq-JpclPQV4)
**Six architecture building blocks (microservices, authn/authz, service discovery, API gateway, message queues, WebSockets), each solving a coordination problem and each introducing its own operational cost.**

- **Microservices.** Split an app into small services per business capability (accounts, catalogue, cart, payments, fulfilment) talking over defined APIs. Pros: independent teams, per-service tech stacks, independent deploys and scaling. Cost: function calls become network calls that can fail, time out, or return surprises, so you need serious monitoring, logging and error handling. **Rule: only when complexity justifies the coordination overhead; small teams/simple apps usually shouldn't.**
- **Authentication and authorization.** "Who are you?" and "what may you do?". Token-based auth: log in once, get a token, attach it to later requests so credentials aren't rechecked each time. Authorization: role-based (viewer/editor roles) or attribute-based policies (user attributes, resource attributes, context). Distributed challenge: consistent enforcement across services; propagate tokens between services and/or use a central policy engine.
- **Service discovery.** In the cloud, instances appear, die and move; hardcoded addresses break. A registry: services register on start; callers look up current addresses. Implementations: DNS-based, dedicated registries, orchestrator-built-in (e.g. Kubernetes). Hard parts: registry unavailability and stale entries for crashed services (health checks/heartbeats).
- **API gateway.** Single entry point. Without it, clients must know every service and each re-implements auth, rate limiting, error handling (tight coupling). The gateway routes, authenticates, rate-limits, transforms formats and **aggregates** (one mobile call -> profile service + activity service -> one combined response). Often adds retries, circuit breakers, response caching, detailed logging. Trade-off: it becomes critical infrastructure; must be highly available and fast or it's a bottleneck/SPOF.
- **Message queues.** Asynchronous communication; producer and consumer needn't be up at the same time (decoupled in time); consumers process at their own pace. Delivery guarantees:
  - *At-most-once*: never duplicated, may be lost.
  - *At-least-once*: never lost, may be duplicated.
  - *Exactly-once*: most complex; each message processed once.
  - Cost: handling partial failures and data integrity across multiple queue-connected components.
- **WebSockets.** HTTP request-response means the server can't speak first, so real-time apps (chat, games) poll wastefully. WebSocket upgrades an HTTP connection to a persistent, bidirectional channel: server pushes immediately, lower latency, fewer requests; an event-driven server can hold many connections. Costs: connections are stateful and pinned to one server (harder load balancing); proxies, firewalls and network blips break them; need reconnection logic, heartbeats, and fallbacks.

**Rules of thumb**
- Choose the combination of patterns your requirements need; not every system needs all.
- Treat every network hop as fallible: timeouts, retries, observability.
- Assume at-least-once and make consumers idempotent (the practical default).
- Gateways and registries must be redundant.

**Relevance to Sam**
- Trameter scale probably doesn't justify many microservices yet; a modular monolith plus a queue for slow supplier work is often the sweet spot.
- An API gateway layer is the natural home for per-supplier auth tokens, rate limits and response caching when Trameter exposes or consumes many APIs.
- Use at-least-once queues for booking events with idempotency keys rather than chasing exactly-once.

## ByteByteGo — 7 System Design Concepts Explained in 10 Minutes (https://www.youtube.com/watch?v=Qd9tJ3H_hPE)
**Reliability means continuing to work correctly while failures happen: CAP choices, eventual consistency, load balancing, consistent hashing, circuit breakers, rate limiting, monitoring.**

- **CAP theorem.** Under a network partition, a distributed system must choose between Consistency (all nodes see the same data) and Availability (every non-failing node answers). Partition tolerance isn't optional. Examples: Google Spanner chooses consistency (synchronised atomic clocks, linearizable transactions; majority side keeps reads/writes, minority loses write availability). DynamoDB chooses availability (keeps accepting writes, eventual consistency, last-writer-wins on timestamps) (note: this describes DynamoDB global tables / original Dynamo design; single-region DynamoDB also offers strongly consistent reads). Banking -> consistency; social feeds -> eventual consistency.
- **Eventual consistency.** If updates stop, replicas converge. Writes complete without waiting for all replicas (faster, more available); e.g. Amazon's cart accepts items even with some servers unreachable. Conflict resolution options: last-write-wins (simple, can silently lose data), CRDTs (merge mathematically regardless of order), application-defined merge functions (business rules). Convergence is usually milliseconds in normal operation.
- **Load balancing.** L4 (IP/port; fast, little intelligence) vs L7 (HTTP headers, URLs, content; smarter, costlier). Algorithms beyond round-robin: least connections, least response time. Balancers run as primary/secondary with heartbeats for failover. Consistent hashing can pin a client to a server for session state.
- **Consistent hashing.** Problem: with `hash(key) mod N`, changing N remaps almost every key. Solution: put nodes and keys on a hash ring; a key belongs to the first node clockwise; replicate to the next N-1 nodes clockwise. Adding/removing a node moves only ~K/N keys (K keys, N nodes), just from neighbours. Used by DynamoDB and Cassandra.
- **Circuit breaker.** Stops cascading failures. States: *closed* (normal, counting failures), *open* (fail immediately without calling the dependency, letting it recover and saving your resources), *half-open* (after a timeout let a few test requests through; success -> closed, failure -> open). Netflix popularised it (Hystrix). Insight: fast failures beat slow failures that tie up threads/connections.
- **Rate limiting.** Algorithms: token bucket (tokens refill at a fixed rate to a cap; allows bursts while bounding average), leaky bucket (process at a constant rate; queue or drop excess), fixed window counter (simple; boundary problem lets a client double its rate around window edges), sliding window (rolling count smooths boundaries). Distributed: counters must be shared, e.g. Redis with Lua scripts to atomically increment and set TTL. Tiered limits (authenticated vs anonymous; stricter when suspicious).
- **Monitoring/observability.** Four signal types: metrics (time series: CPU, request rate, errors), logs (structured events with context), traces (a request's path across services), events (deploys, config changes). Telemetry volume is huge (TBs/day at scale). Alerting: static thresholds break when traffic shifts; use anomaly detection and composite alerts (high CPU AND rising errors AND slow responses) to cut noise. Aim alerts at SLOs that reflect user experience.

**Rules of thumb**
- Decide C vs A per data type: money/inventory consistent, counters/feeds eventual.
- Prefer CRDTs or business merge rules over blind last-write-wins where data loss matters.
- Wrap every external dependency in timeout + circuit breaker.
- "Start simple, measure everything, add complexity only with clear evidence of need."

**Relevance to Sam**
- Supplier integrations are the textbook circuit-breaker case: if RateHawk/Viator degrade, open the breaker and serve cached results or "temporarily unavailable" instead of piling up hung requests.
- You are usually on the receiving side of suppliers' rate limiters: implement a client-side token bucket per supplier so bulk jobs stay under quota and don't get banned.
- Consistent hashing is a handy trick for sharding big local jobs too (stable assignment of keys such as GeoNames ids to worker partitions when you change worker count).

## ByteByteGo — Design a Web Crawler: FAANG Interview Question (https://www.youtube.com/watch?v=6u25GckPhLU)
**A crawler is BFS plus four things that make it work at scale: per-host politeness queues, prioritisation, deduplication (URL-seen and content-seen), and distribution with checkpointing.**

- **Sizing.** 1 billion pages/month ≈ 400 pages/second. Requirements: fast and distributed, polite (never overwhelm a site), prioritised, and fault-tolerant ("on the internet something always goes wrong").
- **Naive BFS and why it fails.** Seed URLs -> queue -> fetch -> extract links -> append. Breaks because most links point back to the same host (Wikipedia linking Wikipedia), so you hammer one site, hit its rate limits and get blocked; and all URLs are treated as equally valuable.
- **Politeness.** Group URLs by host. You can't have one queue per website (millions), so use a fixed pool of a few thousand queues and map host -> queue by hashing: all URLs of one site land in one queue. Worker threads pull from queues with delays between requests to the same host.
- **Prioritiser and frontier.** Incoming URLs are scored (page popularity, update frequency, inbound links) before entering the **frontier** (the to-crawl set); high-value URLs are scheduled sooner. Large crawlers may use adaptive ML models.
- **Deduplication.** *URL-seen* store prevents fetching the same link twice. *Content-seen* detects mirrored/reposted pages by hashing text or structure.
- **Processing loop.** Downloader -> parser (validate HTML, extract text and links) -> link extractor (relative -> absolute URLs) -> URL filter (drop images/videos/disallowed domains) -> prioritiser -> frontier. Repeat.
- **Scaling concerns.** Crawlers distributed across regions, each owning part of the frontier, ideally near target servers; keeping politeness consistent across distributed crawlers is hard; DNS lookups are slow, so cache them aggressively; **checkpoint** so a crashed crawler resumes where it stopped.

**Rules of thumb**
- Rate-limit per destination host, not just globally.
- Hash-partition work by the key you need to serialise on (host).
- Deduplicate on both identity (URL) and content (hash).
- Checkpoint long-running jobs.
- Cache DNS for high-volume outbound fetching.

**Relevance to Sam**
- Directly maps to supplier and content ingestion: bulk-fetching from many suppliers/websites (e.g. Drive Button scanning blog articles, scraping attraction data) needs per-host queues and delays, URL-seen and content-hash dedup, and checkpoints so a crash at row 3,000,000 doesn't restart from zero.
- On a 16GB laptop, keep the frontier and seen-sets on disk (SQLite/RocksDB, or a Bloom filter for URL-seen) instead of in Python sets.

## ByteByteGo — FAANG System Design Interview: Design A Chat System (WhatsApp, Facebook Messenger, Discord, Slack) (https://www.youtube.com/watch?v=okrR1KXNLtA)
**Hybrid protocols (HTTP to send, WebSocket to receive), stateless vs stateful service layers, a per-user inbox with acks for guaranteed delivery, a presence directory for routing between chat servers, and heartbeats for online status.**

- **Scope.** 1:1 and group chats (<=100 members), guaranteed delivery, messages stored for offline users only for a limited time (not a forever archive), online presence; grows from thousands to millions of users.
- **Client-server protocol.** HTTP POST is fine for sending. Receiving is the problem because HTTP is client-initiated: polling wastes requests (mostly empty); long polling helps; **WebSocket** (upgrade after an HTTP handshake to a persistent channel) lets the server push instantly but is harder to scale. Chosen: HTTP for sending (scales easily behind load balancers), WebSocket for receiving.
- **Three layers.** *Stateless services* (auth, profiles, message send via REST, behind load balancers, scale by adding servers). *Stateful chat servers*: each client holds one WebSocket to one chat server; a modern server holds tens to hundreds of thousands of concurrent connections, which sets server count. *Third-party*: Apple APNs / Google FCM push notifications for offline users (don't reinvent this).
- **Inbox pattern (guaranteed delivery).** Online recipient: deliver over WebSocket, device acks, done (no storage). Offline or failed delivery: store in recipient's inbox (a personal queue) for a limited time. On reconnect the chat server drains the inbox in order. A message is removed only after the device **acks**; no ack -> retry later. (This is at-least-once delivery; the client should dedupe by message id.)
- **Routing between chat servers.** Alice's server asks a **presence service** (directory: user -> current chat server). If Bob is online, direct RPC from Alice's server to Bob's server, which pushes over Bob's socket (lowest latency; Discord uses direct server-to-server). If offline, write to inbox and trigger a push notification. Service discovery lets servers find each other.
- **Group chat fan-out.** On send, look up online members and RPC each; offline members get inbox entries. Fine up to ~100 members since cost grows with group size.
- **Presence via heartbeats.** Don't trust the WebSocket appearing alive (backgrounded app, sleeping device). Client sends a ping every 30 s; server records last-seen; no ping for 60 s -> offline. Tolerates brief dropouts (tunnels, lifts).
- **Scaling bottlenecks.** First: connections (add chat servers). Then the database, as inboxes constantly fill and drain: shard by user id. Multi-region chat servers for latency, at the cost of cross-region routing.
- **Further topics.** Ordering across servers (message ids, timestamps, vector clocks); end-to-end encryption key exchange (harder in groups); media (compression, CDN, progressive loading); read receipts and typing indicators (broadcasting typing status can overwhelm a system); rate limiting and abuse prevention.

**Rules of thumb**
- Split stateless (easy to scale) from stateful (connection-holding) components.
- Persist only what's undelivered; delete on ack.
- Use heartbeats with a grace window rather than raw connection state for liveness.
- Use managed push services for offline delivery.

**Relevance to Sam**
- The ack-then-delete inbox is a good model for delivering booking confirmations or supplier webhooks reliably: store pending notifications, retry until acknowledged, dedupe by id.
- Heartbeat + timeout logic applies to monitoring long-running pipeline workers (mark a worker dead after missed heartbeats and reassign its chunk).

## ByteByteGo — How Does a URL Shortener Work? (https://www.youtube.com/watch?v=HHUi8F_qAXM)
**Back-of-envelope sizing, then two ID strategies (hash-and-truncate vs counter + base62), a cache-first read path, and replicas/sharding for a read-heavy workload.**

- **Estimation.** 100M new URLs/day ≈ 1,000+ writes/s; reads ~10x -> 10,000+ redirects/s. 10 years -> ~365 billion URLs; ~36 TB just for URLs (~100 bytes each).
- **Key length.** Alphabet [0-9a-zA-Z] = 62 symbols. 62^1 = 62, 62^2 ≈ 3,800, 62^3 ≈ 238,000, ..., 62^7 ≈ 3.5 trillion > 365 billion. So 7 characters.
- **Approach 1: hash + truncate.** MD5 the long URL (32 hex chars), keep the first 7. Collisions happen; on collision, perturb the input and retry. Every creation needs a DB lookup to check the 7 chars aren't taken. (Note: a bonus of hashing is that the same long URL maps to the same short code.)
- **Approach 2: counter + base62.** Assign the next integer and convert to base62 by repeated division by 62, reading remainders in reverse. Example: 11157 -> remainders 59, 55, 2 -> digits (2, 55, 59) -> "2TX"-style 3-char code (exact letters depend on alphabet ordering). No collisions, no existence checks.
  - Trade-offs: need a unique ID generator across multiple servers (distributed counter / ID service is its own problem); sequential codes are guessable, enabling enumeration of links.
- **Redirect path (the hot path).** Cache lookup -> hit: redirect. Miss: read DB, populate cache, redirect. Uses **301 Moved Permanently**; browsers cache it and may skip the shortener next time. (note: that caching means fewer hits and lost click analytics; many shorteners use 302/307 when they need per-click analytics.)
- **Scaling.** A single DB handles ~10k lookups/s; add read replicas, then shard. Sharding brings its own issues: even distribution, routing to the right shard, shard failure, rebalancing when adding servers.
- **Real-world extras.** Rate limiting against spam, click analytics, malicious-URL blocking.
- **Same patterns elsewhere.** ID generation (Instagram photo ids), metadata caching (Netflix), data splitting (Uber trips).

**Rules of thumb**
- Start design with numbers: writes/s, reads/s, storage over the retention period; they pick the architecture.
- Read-heavy -> cache-aside in front of the DB, then replicas, then shards.
- Prefer collision-free ID generation (counters/ID services) over hash-and-check when you control the key.
- Be aware of the privacy cost of sequential, guessable ids.

**Relevance to Sam**
- Base62-encoded sequential ids are a neat, compact way to create public booking references or shareable trip links (pair with a random component if guessability matters).
- Hash-and-truncate with collision checks is exactly the pattern for generating stable ids for deduplicated places/locations; for millions of rows, use a long enough hash (e.g. 64-bit) that collisions are practically impossible rather than checking each one.

## ByteByteGo — How Key value Stores Work (Redis, DynamoDB, Memcached)? (https://www.youtube.com/watch?v=Dwt8R0KPu7k)
**A key-value store is a giant dictionary; distributing it across thousands of machines forces consistent hashing for placement, replication for survival, a CAP choice, conflict resolution (vector clocks) and gossip for failure detection.**

- **Model.** `put(key, value)` / `get(key)`, e.g. `user:12345:cart` -> cart items. Amazon-scale: terabytes per region, billions of pairs, millions of accesses per second, so data must be spread over thousands of servers. Question: which server holds a given key?
- **Modulo hashing fails.** `hash(key) % N` remaps nearly every key when N changes; adding one machine means moving almost all data.
- **Consistent hashing.** Servers placed at points on a ring (e.g. A=100, B=1000, C=5000). A key hashes to a point (750) and belongs to the first server clockwise (B). Adding D at 500 moves only keys in 101–500 from B to D; everything else stays.
- **Replication.** If B dies its keys vanish, so store each key on the owner plus the next two servers clockwise (C and A). Survives node loss.
- **Conflicts.** Two family members add to the same cart simultaneously via different replicas -> two versions. CAP: during network problems you choose consistency (refuse requests; banks) or availability (serve possibly stale data; most web apps would rather show an old cart than none). Big systems choose **eventual consistency**.
- **Vector clocks.** Version metadata recording which server modified the data (and its counter), so the system can tell whether one version descends from another or they truly conflict. On conflict: auto-merge (union of cart items), ask the user, or last-write-wins (risk losing data).
- **Failure detection via gossip.** All-to-all pings with 1,000 servers ≈ a million connections; doesn't scale. Each server keeps a membership list and periodically shares it with a few random peers; news of a dead node spreads epidemically through the cluster.
- Not covered: storage engines, data-centre failures, performance tuning.

**Rules of thumb**
- Use consistent hashing (with replicas to the next N-1 nodes) whenever the node set changes.
- Pick conflict-resolution semantics per data type; mergeable data (sets like carts) can be merged automatically.
- Use gossip/heartbeats rather than full-mesh health checks at scale.

**Relevance to Sam**
- Redis/Memcached as a cache in front of supplier APIs (key = supplier + query params, value = response, with TTL) cuts latency and keeps you under supplier rate limits.
- Mergeable-set thinking helps when combining location records from multiple sources (GeoNames + OSM + supplier data): define merge rules per field instead of last-write-wins.

## ByteByteGo — Rate Limiter System Design: Token Bucket, Leaky Bucket, Scaling (https://www.youtube.com/watch?v=YXkOdWBwqaA)
**Use a token bucket stored in Redis, enforced in middleware, with atomic Lua scripts so concurrent servers can't lose counts.**

- **Requirements.** Configurable rules (e.g. 100 requests/min/user); on excess return **HTTP 429** with headers for remaining quota and reset time; overhead under ~3 ms p95 per check; highly available; shared by multiple servers.
- **Fixed window counter.** Per-user counter reset each minute; reject past 100.
  - Where to store it: a database adds a query to every request and can overload the thing you're protecting; per-server memory is fast but each server counts separately (100 on A + 100 on B = 200). **Redis**: shared, in-memory, with increment and auto-expiry primitives.
  - Flaw: boundary bursts. 100 requests in the last 10 s of minute 1 + 100 in the first 10 s of minute 2 = 200 in 20 s, every boundary.
- **Token bucket (industry standard; AWS, Stripe).** Tokens refill at a steady rate up to a capacity; each request consumes one; empty bucket -> reject. Two knobs: **capacity** = max burst, **refill rate** = sustained throughput. Capacity 100, refill 100/min allows 100 instantly but exactly 100/min long-term; quiet periods bank tokens for legitimate bursts, and boundary gaming doesn't work.
- Alternatives that also fix boundaries: sliding window log, sliding window counter, leaky bucket (constant drain rate). Token bucket is the best simplicity/effectiveness balance for most cases.
- **Placement.**
  - Client-side: can't be trusted; malicious clients bypass it.
  - Server-side in app code: full control, but mixed with business logic and duplicated per service.
  - Middleware (API gateway, reverse proxy, dedicated service): separate from business logic, one place to manage policy, more moving parts. Usually best; if you already have a gateway doing auth, put it there.
- **Architecture.** Rules live in a config service (premium 1,000/hour, free 100/hour). Middleware caches rules, keeps bucket state in Redis; on request: identify user, load bucket, if token available decrement and forward, else 429 + retry-after info.
- **Race condition at scale.** Servers A and B both read 3, both decide OK, both write 4: one count lost. Fix: **atomic** read-check-decrement via a Redis Lua script (executes as one indivisible unit).
- **Open questions mentioned:** multi-region latency, hot keys (few users generate most traffic), updating rules without restarts, and **fail open vs fail closed** when Redis/the limiter is down.

**Rules of thumb**
- Separate burst (capacity) from sustained rate (refill) explicitly.
- Shared state + concurrency -> atomic operations (Lua, `INCR`, transactions), never read-modify-write from app code.
- Always tell clients when to retry (429 + reset/Retry-After headers).
- Decide fail-open/fail-closed deliberately.

**Relevance to Sam**
- As a client of supplier APIs, honour their 429s and `Retry-After`, and run your own token bucket per supplier (in-process for a single job; Redis if several workers/processes share a quota) so parallel workers don't collectively exceed the limit.
- The read-check-write race is the same bug class as double booking; the cure is the same: make the check and the update one atomic operation.

## ByteByteGo — System Design: Design YouTube (https://www.youtube.com/watch?v=jWRW2xGMqSw)
**Keep giant files off your API servers (pre-signed URLs + chunked multipart uploads), turn processing into a DAG of parallel segment tasks, and serve adaptive-bitrate segments from CDNs.**

- **Scale and requirements.** YouTube ingests ~500 hours of video per minute. A 10-min 4K video is 1.5 GB (compressed) to 30 GB (ProRes). Uploads must be resumable; playback must adapt in real time (e.g. 25 -> 2 Mbps without stopping). Scope: upload and streaming only.
- **Uploads via pre-signed URLs.** You could proxy uploads through API servers (streaming, 30-minute timeouts, partial-upload handling) but that wastes them on pass-through bytes. Instead the API issues a temporary signed URL granting direct write to blob storage; the client uploads straight there.
- **Multipart/chunked upload.** Client splits into 5–10 MB chunks (small enough for slow links, big enough to limit overhead), fingerprints each with SHA-256, uploads ~6 in parallel; failed chunks retry individually, enabling resume. Same pattern as Dropbox/Google Drive/backup tools.
- **Transcoding needs.** Inputs vary (iPhone HEVC, Android H.264, ProRes). Outputs must play on old phones, 2018 smart TVs, stale browsers: resolutions 2160p down to 240p; codecs H.264 (universal, more bandwidth), VP9 (less bandwidth, not on old devices), AV1 (least bandwidth, needs strong hardware); containers MP4 (compatibility) and WebM (web). One upload -> 15–20 output files.
- **DAG workflow.** Model processing as a directed acyclic graph: nodes are steps, edges are dependencies; acyclic guarantees completion. Split video at keyframes (every 2–10 s; independently decodable) into segments; split video, audio, metadata into separate paths; fan segment x resolution tasks to hundreds of workers; thumbnails and subtitle extraction on dedicated workers. Hours sequentially -> minutes in parallel.
- **Adaptive bitrate streaming.** Player downloads few-second segments, not one file. A primary manifest lists renditions; each rendition's media manifest lists segment URLs. Player measures recent download speed and picks the rendition per segment; switches are seamless. HTTP **range requests** fetch byte ranges, enabling instant seeking. Segments served from **CDN** edges near viewers (Tokyo viewer served from Tokyo).
- **Deeper topics.** Hot (viral) videos: metadata caching and DB hotspot avoidance. Cost: transcode popular formats first, on-demand transcoding for rare formats, cold storage migration. Latency: start processing segments as they arrive, pipelining upload and processing.

**Rules of thumb**
- Let clients talk to object storage directly with short-lived signed URLs; keep app servers for logic.
- Chunk + checksum + parallel + per-chunk retry = resumable large transfers.
- Model batch processing as a DAG of independent units to parallelise and retry at fine granularity.
- Precompute the common variants; produce rare ones lazily.

**Relevance to Sam**
- The DAG-of-chunks model is exactly how to structure the multi-million-row location pipeline: split by natural boundaries (country/admin region files), process chunks independently in a worker pool, retry failed chunks only, and record completion per chunk (checkpointing). Orchestrators like Airflow/Prefect/Dagster formalise this, but a simple manifest table works on a laptop.
- Chunked downloads with checksums and resume apply to big source dumps (GeoNames, OSM extracts, GTFS feeds).
- Pre-signed URLs are the right way to let partners/creators upload media (e.g. Riselist or Drive Button assets) without proxying bytes through your API.

## ByteByteGo — System Design: Why is Kafka Popular? (https://www.youtube.com/watch?v=7_wkWQ9rB5I)
**Kafka is a replicated, partitioned, append-only log: it decouples producers from consumers, absorbs spikes, and allows replay, at the cost of operational complexity, per-partition-only ordering, and throughput-over-latency tuning.**

- **Why used.** Decoupling (producers and consumers evolve independently), buffering traffic spikes that would swamp direct calls, replaying events for debugging and recovery. LinkedIn, Netflix, Uber.
- **Structure.** Messages are appended to a **partition** = append-only log file on disk. Partitions live on **brokers**; brokers form a cluster. Partitions are grouped into **topics** (payments, clicks, uploads). A message has key, value, timestamp, optional headers. The **key picks the partition**: same key -> same partition -> ordered; no key -> spread for balance. One broker handles hundreds of thousands of messages/s; network bandwidth usually saturates before CPU or disk.
- **Partitioning strategy decides scalability.** Bad key -> **hot partition**. Example: key by movie id and a blockbuster's Friday-night premiere floods one partition. Fix: compound key (movie id + hash of user id) spreads load while keeping each user's events ordered. Time-based partitioning suits logs (easy retention) but complicates real-time aggregation.
- **Offsets.** Consumers track position per partition with offsets and commit them back to Kafka periodically. Commit too early -> crash loses messages (marked done before processed). Commit too late -> reprocessing duplicates.
- **Consumer groups.** Each partition is consumed by exactly one consumer in a group; parallelism is capped by partition count. On consumer failure, Kafka **rebalances** its partitions to survivors automatically.
- **Delivery guarantees.** At-most-once (fast, may lose), at-least-once (no loss, duplicates), exactly-once (possible, complex, slower; worth it for financial or critical pipelines; needs producer and consumer configuration).
- **Durability.** Each partition has a leader (handles reads/writes) and followers replicating it; a follower takes over if the leader fails. Typical replication factor 3 (survive one broker loss safely). Producers can require acknowledgement from all in-sync replicas (`acks=all`): safest, slower.
- **Patterns.** Uber reportedly streams driver locations through Kafka for real-time surge pricing, partitioned by geography so regions scale independently. **Event sourcing**: append every state change as an event and rebuild current state by replay; full audit trail.
- **When not to use.** Optimised for throughput, not latency (batching/buffering adds delay), so not for request-response. Ordering only within a partition; global order means one partition and no parallelism; most systems accept partial (per-key) order. Significant operational burden.

**Rules of thumb**
- Choose partition keys for both ordering needs and even load; watch for skew.
- Commit offsets after processing (at-least-once) and make processing idempotent.
- Replication factor 3, `acks=all` for data you can't lose.
- Need per-entity ordering? Key by entity id.

**Relevance to Sam**
- Kafka is likely overkill operationally for Trameter today; but its ideas transfer: an append-only event table (bookings_events) for audit and replay; key-based partitioning of work so one entity's updates are processed in order; idempotent consumers; "commit after processing" for any resumable batch job.
- For partitioned parallel processing of location data, avoid hot partitions (e.g. one massive country): split big keys with a secondary hash, like the compound-key trick.

