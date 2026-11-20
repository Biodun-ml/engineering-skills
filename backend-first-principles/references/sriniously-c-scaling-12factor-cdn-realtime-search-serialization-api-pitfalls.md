# sriniously-3 — cross-cutting principles

1. **Measure before you fix.** Find the bottleneck with timing logs, distributed tracing, `EXPLAIN ANALYZE` and profilers (CPU-bound work only) before adding caches or servers. In the teacher's example, a week of Redis work missed a synchronous 500 ms logging call. (Scaling Pt1, Pt2)
2. **Think in percentiles and headroom, not averages.** Track P50/P95/P99 against throughput. Latency rises sharply as utilisation nears 100%, and traffic is bursty, so run at ~60–80%. The P99 path is usually the money path (payments, bookings, external calls). (Scaling Pt1)
3. **Simplicity first; observability from day one.** Use a bigger box before Kubernetes, an index before Redis, a monolith before microservices. Microservices scale teams, not machines. The one exception to "keep it simple" is production-grade logs, metrics and traces from the start. (Scaling Pt2)
4. **Fix the database in order:** remove N+1 queries (batch / JOIN / eager load), add indexes based on evidence (composite column order matters; every index slows writes), pool connections (use an external pooler like PgBouncer when autoscaling), then cache. (Scaling Pt1)
5. **Caching is an invalidation problem.** Cache-aside is the default. Write-through trades write latency for no misses; write-behind risks inconsistency. TTL vs event-based invalidation; local vs distributed vs tiered caches. Watch the hit rate (~90% is good). At the HTTP layer, use `Cache-Control` (`private`/`public`, `max-age`, `s-maxage`) with ETag revalidation. (Scaling Pt1; Vercel Pt2)
6. **Stateless, disposable processes.** No sessions, files or caches tied to one instance. Never use sticky sessions. You must be able to kill any process at any time without data loss. Boot in seconds, handle SIGTERM by draining, and survive OOM kills (crash-only design). (Scaling Pt2; Twelve-Factor 6, 9)
7. **Idempotent, re-entrant jobs.** Interrupted work goes back to the queue and must be safe to run again: wrap it in a transaction or make it idempotent. This underpins graceful shutdown, retries and at-least-once delivery. (Twelve-Factor 9; Real-Time)
8. **Respond fast, finish async.** Do only what the user must see synchronously, then queue the rest (emails, external APIs, media processing, bulk deletes) to separately scaled workers. In the teacher's examples: 400 ms -> 100 ms, and 8 s -> instant. (Scaling Pt2; Twelve-Factor 8)
9. **Replicas and shards trade consistency for scale.** Read replicas cause read-your-write anomalies through replication lag, so route reads that follow a write to the primary. Sharding hinges on the shard key. Prefer managed or distributed databases over running your own. (Scaling Pt2)
10. **Distance is physics.** Fibre carries data at ~200,000 km/s, so a cross-world round trip costs ~100 ms. CDNs and edge PoPs cut this to a few ms for cacheable content. Edge compute suits light, data-free logic (auth rejection, locale, redirects); data-bound compute belongs near the database. (Vercel Pt2, Pt4; Scaling Pt2)
11. **Config and secrets live outside code; backing services are URLs.** Config goes in granular env vars (pass the "could we open-source this repo now?" test); use a secret manager as the team grows. Every database, cache, queue or third-party API is an attached resource swappable via config. (Twelve-Factor 3, 4)
12. **Reproducible builds and immutable releases.** One codebase, many deploys; declared and lock-pinned, isolated dependencies (containers are the strongest form); build -> release -> run with immutable releases for instant rollback; dev/prod parity (the SQLite-vs-Postgres `LIKE` trap); logs to stdout; one-off admin tasks shipped in the repo. (Twelve-Factor)
13. **Validate strictly at the boundary; design for the consumer.** Validate every input with enums, formats and ranges (security + documentation). Use sane defaults for optional params, `200 []` for empty lists vs `404` for a missing single entity, separate success and error schemas with machine-readable codes, deterministic pagination with `total`/`page`/`limit`/`totalPages`, and an OpenAPI spec generated from code feeding typed clients. (Don't be THAT; Serialization)
14. **Push only when it's worth it, and pay per event.** Polling cost scales with watchers. SSE (one-way, auto-reconnect via `Last-Event-ID`, used for LLM streaming) covers most push needs; WebSockets give bidirectional framed streams with ping/pong heartbeats. (Real-Time)
15. **Persistent connections are state.** Fan out across instances with pub/sub. Redis Pub/Sub is at-most-once; use Streams or Kafka for at-least-once. Sequence-number every message for catch-up, and plan for fan-out cost (events x subscribers) and reconnect storms. Per-connection capacity limits are FDs (runtime-dependent), client ephemeral ports (the 4-tuple) and ~10 KB RAM per connection. (Real-Time)
16. **Use the right engine for search.** Leading-wildcard `ILIKE` means full scans with no ranking or typo tolerance. Use an inverted index (Postgres FTS/trigram or Elasticsearch/Lucene) with BM25 and field boosting; map exact fields as `keyword`. Bulk-load in batches. (Elasticsearch)
17. **Choose a rendering/compute model by data freshness and load pattern.** SSG / ISR / SSR per route. Serverless suits bursty, event-driven work (cold starts, ~15-minute limits, statelessness); avoid it for latency-critical payment paths and long jobs. Autoscaling is reactive and has min/max cost traps. (Vercel Pt4; Scaling Pt2)

## Sriniously — 21.1 Backend Scaling and Performance Engineering: Part-1 (https://www.youtube.com/watch?v=z7kt_p44rjs)
**Measure before you optimise: latency is a distribution, utilisation is non-linear, and the real bottleneck is rarely where you assume.**

- **Mental models / first principles**
  - "Fast" = latency: the whole round trip from user action to rendered result (browser -> network -> server -> DB / third-party API -> response -> parse/render).
  - Latency is not one number. It varies per request (cache hit vs miss, idle vs busy server). Averages hide pain: if 99% finish in 50 ms and 1% take 5 s, the mean (~100 ms) looks healthy while, at 1M req/day, 10,000 users wait 5 s.
  - Use percentiles: P50 (median), P90 (10% of users see worse than this), P99 (1% see worse). Engineers focus on P95/P99 because the slowest paths are where the heaviest logic lives (payments, purchases, external calls, webhooks) — i.e. your most valuable users.
  - Throughput = requests per second/minute the system can sustain. Latency must be read together with throughput: 150 ms at 10 rps can become 2 s at 1,000 rps. Capacity questions ("can we survive Black Friday / an email campaign / a podcast mention?") need both.
  - Utilisation vs latency is not linear (ice-cream shop / highway analogies). Worker speed is unchanged under load, but queueing time explodes as utilisation approaches 100%. At ~50% traffic flows; at 80% slowdowns; at 90% behaviour becomes unpredictable; at 100% gridlock.
  - Traffic is bursty, never metronomic. Even 40–50% average load can spike past 100%.

- **Mechanisms**
  - Bottleneck = the specific component causing slowness. Teacher's story: a week spent adding Redis in front of a "slow" GET; timing logs then showed DB 10 ms, cache 5 ms, but a **synchronous call to a remote logging service ~500 ms**. Other hidden culprits: JSON/XML serialisation, huge response payloads over the network, external API calls inside loops.
  - Profilers sample a running app and record which functions run and for how long; flame graphs show call stacks — wider bar = more time, callees stacked on top. Good for CPU-bound work, weak for I/O-bound work.
  - Most backend slowness is I/O-bound (DB, external APIs, queues, files). For that use **distributed tracing** (observability): follow one request through every component with timestamps (e.g. 2 ms in business code, 800 ms in a DB query).
  - **N+1 queries**: one query for N items, then one query per item (e.g. 20 posts + 20 author lookups = 21 round trips). Each query pays network + (maybe) TCP setup + parse + plan + execute; 1,000 x 5 ms = 5 s. Fix: batch — collect IDs, fetch in one `IN`/JOIN query (2 queries regardless of N). ORMs hide this because loops look like normal code; use eager-load primitives (Django `select_related`/`prefetch_related`, Rails `includes`, joins in TypeORM/Prisma/Drizzle) and turn on SQL query logging in development.
  - **Indexes**: without one, the DB does a sequential (full-table) scan. A B-tree index keeps a sorted copy of a column's values plus row pointers; teacher's rough example: ~4 s scan over 1M rows vs <100 ms with an index. Costs: storage, and every INSERT/UPDATE/DELETE must also update every index, so indexing every column slows writes heavily.
  - Postgres indexes the primary key automatically. Add obvious foreign-key indexes (e.g. `author_id`) up front; add the rest only after traces show a slow query.
  - **Composite index** (`user_id, created_at`): column order matters — it serves queries on `user_id` alone or on both, but not `created_at` alone. **Covering index**: includes the columns a query needs, so the DB answers from the index without touching the table (bigger index).
  - `EXPLAIN ANALYZE` shows the executed plan; look for Seq Scan on big tables, add an index, re-run and confirm it now shows Index Scan.
  - **Connection cost**: each new DB connection = TCP 3-way handshake + auth + TLS negotiation + session state + server memory (MBs). Postgres typically allows only a few hundred connections (about 300–600 depending on size).
  - **Connection pooling**: borrow an open connection, run the query, return it. Internal (driver-level, per process) pools multiply with horizontal scaling: 3 instances x 150 = 450 possible connections against a DB that allows 300, so the DB falls over during a spike. An **external pooler (PgBouncer)** caps total connections across all instances; prefer it in production.
  - **Caching** (after queries, indexes and pooling are fixed): e.g. 800 ms query -> 50 ms cache hit.
    - Invalidation: time-based TTL (picking the right TTL is hard and per-endpoint) vs event-based (delete the key in the write handler; precise, but every write path must remember to do it or users see stale data).
    - Location: local in-process map (~instant, but N instances = N inconsistent copies) vs distributed (Redis/Memcached/Valkey; consistent, costs a network hop). **Tiered**: small local cache for the hottest keys in front of Redis.
    - Patterns: **cache-aside / lazy loading** (read cache -> on miss read DB, fill cache; on write delete the key) — the default. **Write-through** (write DB and cache together; never misses, slower writes). **Write-behind** (write cache, persist to DB asynchronously; fast, but a failed DB write leaves the two inconsistent).
    - Hit rate: ~90% is good, ~20% means something is wrong. Driven by TTL (long TTL raises hits but also staleness; 7 days defeats the point), cache size, and how well you understand access patterns.
  - **Vertical scaling** (scale up: more CPU, RAM, NVMe, 10 Gbps NIC): no architecture change; 2x CPU ≈ 2x requests; one big box usually costs less than two half-size ones and has less ops overhead. Limits: a hardware ceiling (largest instance available), a single point of failure, and no geographic spread (users far from the region pay latency).
  - **Horizontal scaling** (scale out: more instances): linear capacity in theory (1k rps/server x 5), redundancy, servers can sit in several regions. Costs: you now need a load balancer and an algorithm, state sync between servers, partition and conflicting-decision handling, health detection and re-admitting instances. Distributed systems swap one set of problems for another; they do not remove problems.

- **Decisions & trade-offs**
  - Write-through vs write-behind vs cache-aside: freshness vs write latency vs consistency risk.
  - Local vs distributed cache: speed vs consistency. Tiered gets most of both.
  - Index generosity: read speed vs write speed and storage.
  - Vertical first for simplicity; horizontal when you hit the ceiling or need high availability or geographic reach.

- **Rules of thumb**
  - Never guess; always measure (timing logs -> tracing -> EXPLAIN ANALYZE -> profiler for CPU work).
  - Track P50/P90/P99 against throughput, not averages.
  - Keep production at roughly 60–80% utilisation; keep 20%+ headroom for bursts.
  - Eliminate N+1 queries, index from evidence, pool connections (external pooler when autoscaling), then cache.

- **Failure modes**
  - Applying textbook fixes (add Redis, upgrade Postgres, add servers) to a problem you have not located.
  - Synchronous side work (logging, emails) on the request path.
  - Per-instance pools exhausting DB connections when autoscaling.
  - Forgotten invalidation on one write path -> stale data.

- **Relevance for Sam**
  - Supplier calls (Viator, RateHawk) inside loops are N+1 over HTTP: batch them, or run them concurrently with a bounded limit.
  - Trace the booking flow end to end. Its P99 path is the money path.
  - For million-row matching jobs, use `EXPLAIN ANALYZE` plus targeted composite indexes, and make bulk inserts before creating indexes to avoid index-maintenance cost on writes (inference from the index-write-cost point).
  - On a 16GB laptop, a small local LRU cache in front of a remote/expensive lookup is the "tiered cache" idea in miniature.

## Sriniously — 21.2 Backend Scaling and Performance Engineering: Part-2 (https://www.youtube.com/watch?v=sOhAopEwjH4)
**Scale out stateless app servers behind a load balancer. Handle the stateful database with replicas or shards. Use CDNs, queues and serverless where they fit. Only add complexity that measurements justify.**

- **Mental models / first principles**
  - **Statelessness** is what makes horizontal scaling work: any instance must give the same result for any request, and deleting one instance must change nothing. Anything that persists (sessions, uploaded files, cache, SQLite files) must live outside the instance: Redis for sessions, S3/R2/MinIO for files, Postgres/RDS for data. Example: a session held in instance A's memory gives a 401 when the next request lands on B.
  - Horizontal scaling therefore changes code and architecture. Vertical scaling only changes the instance size.
  - The database is the hard part because it is stateful: copies must stay consistent.
  - Physics sets a latency floor. Light in fibre travels about 200,000 km/s, so a Tokyo <-> N. Virginia round trip (~20,000 km) costs ≥ ~100 ms before any work. Add deserialisation, a 50–100 ms query and a ~200 ms external API, and the user sees 500–800 ms.
  - Microservices scale **teams**, not machines.

- **Mechanisms**
  - **Load balancer**: the single entry point that forwards each request to an instance and relays the response.
    - Round robin: rotating order. Fine when requests and servers are uniform. When cheap (~200–300 ms) and expensive (~2 s: external call + write to a heavily indexed table) requests are mixed, it can pile the heavy ones onto one server.
    - Weighted round robin: proportional to capacity (an 8 GB/4-core box gets 2x).
    - Least connections: send to the server with the fewest in-flight requests, so slow requests naturally steer traffic away. It also has a weighted variant.
    - Others: least response time, and resource-based (CPU/RAM).
    - **Health checks**: the LB polls a cheap endpoint (e.g. every second) expecting 200. On failure it stops routing user traffic to that instance but keeps probing, and re-adds the instance once it returns 200. Without this, users randomly hit 502/503.
  - **Read replicas**: one primary takes writes; replicas take SELECTs and can be placed near users. Most SaaS traffic is ~70–90% reads, so the primary handles ~30%.
    - Cost: **replication lag** (e.g. ~200 ms US->India). The classic bug is read-your-own-write: save the profile, the UI immediately refetches from a replica and shows the old value. That is dangerous for payments or invoices.
    - Mitigations: route reads that follow a write on the same entity to the primary; track lag and make reads wait; delay the client refetch.
  - **Sharding / partitioning**: split one huge table (e.g. billions of orders) across physical DB instances by a **shard key** (e.g. order date ranges, or per month). Each shard has fewer rows (faster queries) and you get more total capacity. A routing layer decides which shard to query. Picking the key is the hard part.
  - Modern distributed DBs (PlanetScale/Vitess, Neon, CockroachDB, YugabyteDB) and managed RDS/Cloud SQL handle replication, sharding and backups. Don't run your own DB infrastructure without deep expertise, but know the concepts so you can configure regions, replicas and backups.
  - **CDN**: edge PoPs near users cut latency from ~100 ms to ~2–3 ms and offload origin traffic.
    - Cache static assets (JS/CSS/HTML bundles, images, fonts, video) and some API responses (e.g. a slow-changing catalogue).
    - Purge by tag on change (Cloudflare cache tags).
    - Also absorbs DDoS. Without it, autoscaling under attack can turn into a huge bill (the teacher's example: $50k in a day).
  - **Edge compute** (Cloudflare Workers on V8 isolates): run small logic at the PoP. Good uses: reject unauthenticated requests in ~2–3 ms instead of a 100 ms round trip, locale/geo customisation, routing. Constraints: little CPU/RAM, no filesystem, no raw TCP. It complements origin servers; it does not replace them.
  - **Async processing / queues** (adopt early, not late): do the minimal synchronous work, respond, push a job to a queue (Redis/BullMQ, RabbitMQ, Kafka) and let workers (same codebase or a separately scaled one) finish.
    - Invite flow: ~100 ms DB work + ~300 ms email API -> respond after 100 ms and queue the email.
    - Account deletion across many large tables (~8 s synchronously) -> validate, respond, queue `delete_user`.
    - Video thumbnails, encoding and subtitles follow the same pattern.
    - Candidates: anything whose result the user doesn't need to see instantly (emails, notifications, media processing, bulk deletes).
  - **Monolith vs microservices**: a monolith is one deployable unit and easy to develop, test, refactor and deploy. Microservices help with:
    - deployment coupling between large teams (100–200+ devs);
    - scaling one hot module independently (e.g. payments vs notifications);
    - a different stack per module (an npm Markdown library vs Go/Rust image resizing at ~500 ms vs ~50 ms).
    - Costs: function calls become network calls (latency, failures, retries, timeouts), debugging needs distributed tracing across services, and each service owning its DB creates consistency problems.
  - **Serverful vs autoscaling vs serverless**:
    - Fixed servers force capacity planning: under-provision and you crash; over-provision and you pay (e.g. $5,000 vs $500 a week).
    - Autoscaling (e.g. add an instance at 70% memory) is reactive, boots take seconds to minutes, max-instances caps cut both ways (too low = outages, too high = $100k days), and you still pay the always-on minimum.
    - Serverless: you supply functions plus event triggers (an API gateway maps routes to functions) and pay per execution. Costs:
      - **cold starts** (mitigated by keep-warm pings, which erode the savings, and by fast runtimes: Firecracker microVMs for Lambda, V8 isolates at ~0–5 ms for Workers);
      - execution limits (Lambda ~15 min);
      - statelessness (no persistent DB TCP pools or websockets, so you need serverless DBs and poolers).
    - Avoid it for latency-critical banking/payment paths, long-running jobs and DB-connection-heavy apps. Good for sporadic, event-driven work: image/video processing, queue messages, file uploads, DB change events.

- **Decisions & trade-offs**
  - Every distributed solution trades consistency for scale or latency.
  - Edge vs origin: speed vs capability.
  - Serverless vs servers: cost at low or irregular load vs latency predictability.
  - Microservices only with clear answers on team size, divergent scaling needs and tech needs.

- **Rules of thumb (the teacher's closing list)**
  1. Start from the problem: measure with logs, metrics, traces and load tests before choosing a fix. Otherwise it's premature optimisation.
  2. Prefer simple: a big vertical box before Kubernetes, an index before Redis, a monolith before microservices. Every component is something else that can fail and must be monitored.
  3. Scale for the load you have plus reasonable headroom. Big-tech blog advice may not fit your system.
  4. Production-grade observability (Prometheus/Grafana, New Relic) from day one is the one exception to "keep it simple".
  5. Performance is a mindset built by watching real systems struggle.

- **Failure modes**
  - In-memory sessions or local file uploads behind a load balancer.
  - Round robin with skewed request costs.
  - Read-your-write anomalies on replicas.
  - Autoscaling too slow for 3–4 s bursts.
  - Uncapped autoscale or DDoS bills.
  - Cold starts on user-facing paths.

- **Relevance for Sam**
  - Supplier calls (RateHawk/Viator) and confirmation emails belong in queued workers, with the booking API replying once its own DB write is durable.
  - After a booking write, read from the primary (read-your-write), never from a lagging replica.
  - Big location tables are natural candidates for partitioning by country or region.
  - Serverless suits bursty batch enrichment jobs; it does not suit long matching runs (15-minute limits).

## Sriniously — 27. The Twelve-Factor App (https://www.youtube.com/watch?v=2_fcv3-XfN4)
**Twelve-Factor is the contract between your service and the platform that runs it. Follow it and any machine, platform or new teammate can run, scale and update the app without surprises.**

- **Mental models / first principles**
  - Written in 2011 by Heroku (Adam Wiggins, 12factor.net) from watching thousands of deploys. Heroku open-sourced it in Nov 2024 with AWS/Google maintainers. A community rewrite on the "next" branch keeps the 12 names but restates each as principles; it is still a draft as of the video.
  - The problem it fights is **software erosion**: hand-built "snowflake" servers (OS, runtime, DB and FTP'd code installed by hand, config edited over SSH) rot even when nobody changes the code — OS patches, a co-hosted app upgrading a shared library, the person who set it up leaving. Symptoms: "works on my machine" and multi-day onboarding.
  - Scope: network services (web apps, APIs) in any language or DB. It covers the app/platform interface, not general code quality (DRY, KISS).
  - Docker, Kubernetes, Render, Railway, Fly, Cloud Run and Vercel all assume these rules.

- **The factors**
  1. **One codebase, many deploys.** One repo per app; production, staging, PR previews and laptops are deploys of it at different commits (e.g. prod at 97, staging at 99, laptop at 100+). Config is what differs between them, never code. Multiple codebases = a distributed system, and each part follows the 12 factors. Copy-pasted shared code is a violation: make it a library. Monorepos are fine (five apps in one repo = five codebases); the anti-pattern is one app living in two places.
  2. **Declare and isolate dependencies.**
     - Declare: a manifest (package.json, go.mod, requirements.txt/pyproject) plus a **lockfile** (package-lock, go.sum) that pins exact transitive versions, so a rebuild years later gets the same packages.
     - Isolate: the app must see only declared deps (virtualenv, bundle exec), never system-installed ones. Example: system `requests` v1 shadowing the declared v2.
     - System binaries too: ImageMagick 7 (`magick`, Homebrew on a Mac) vs 6 (`convert`, apt on Ubuntu) behave differently.
     - Containers freeze OS, tools and runtime: the strongest form of this factor.
  3. **Config in the environment.**
     - Config = whatever differs per deploy (DB URL, email sandbox vs real, MinIO vs S3, Redis host).
     - Test: could you open-source the repo right now without leaking a credential?
     - Env vars change without editing files, can't be committed by accident, and work in every language.
     - Keep vars granular. Don't group them into named environments (`staging.rb`, `production.rb`): that doesn't scale as deploys multiply.
     - In the teacher's Go "Tasker" app, ~38 values load into one config struct with a `TASKER_` prefix. Config changed often; the loader code barely changed.
     - Caveat (Diogo Mónica, 2017): env vars leak into child processes, crash reporters, debug pages and `/proc/<pid>/environ`. For secrets beyond a small team, use a secret manager (Vault, AWS Parameter Store): fetch at boot, hold in memory, rotate, audit.
  4. **Backing services are attached resources.** DB, Redis, queue, SMTP, S3, payment APIs: anything reached over the network. Code treats local and third-party services identically, reaching them through a resource handle (URL + credentials) kept in config, e.g. `postgres://user:pass@host:5432/dbname`. Test: swap local Postgres for Neon/RDS by changing only config. This aged very well; app and DB on one box is now an anti-pattern for scaling.
  5. **Strictly separate build, release, run.**
     - Build = code at a commit + dependencies -> binary/bundle/image, with **no config inside**.
     - Release = build + a deploy's config, immutable, with an ID, in an append-only ledger.
     - Run = launch processes from a release.
     - Benefits: instant rollback (release 102 throws 4xx/5xx errors -> re-run 101; a config-only change like a password rotation is its own release), and no path for hot-patching production over SSH.
     - The rewrite makes immutability explicit.
  6. **Stateless, share-nothing processes.**
     - Persist everything in backing services. Local memory or disk is fine only as a per-transaction scratch space (e.g. a temp file while processing an upload).
     - **Sticky sessions are a violation**: killing an instance logs its users out, and new instances get no existing users. Put sessions in Redis or the DB.
     - Exception: WebSockets, where the connection itself is state. Fan out with pub/sub.
     - Rule: if you can't kill any process at any moment without a user losing data, state is in the wrong place.
  7. **Port binding.** The app embeds its own HTTP server as a declared dependency and listens on a port (rather than living inside Apache/Tomcat). Any app can then be another app's backing service. The rewrite generalises this to "export via a declared interface" to cover serverless functions.
  8. **Concurrency via the process model.**
     - Separate process types (web, worker, scheduler), each scaled independently: the **process formation** (Procfile, k8s `replicas`).
     - Teacher's food-delivery example at 500 MB per process:
       - lunch = 10 web + 2 workers (6 GB), versus 10 combined processes (10 GB, half of it idle workers);
       - night = 5 web + 20 workers.
     - Threads, goroutines and event loops are fine inside a process, but the process is the unit of scale.
     - Don't daemonise or write PID files; run in the foreground and let systemd/k8s manage the lifecycle.
  9. **Disposability: fast startup, graceful shutdown.**
     - Processes stop and start constantly: 5 deploys a week x 10 instances = 50 restarts.
     - Slow boots compound: 60 s boot x 10 instances = a 10-minute rolling deploy and autoscaling a minute late. Aim for a few seconds.
     - On SIGTERM: stop accepting new work, finish in-flight work, exit (typically ~30 s grace).
     - Workers return the current job to the queue, which requires **re-entrancy**: the job must leave no dirty state, so wrap it in a transaction or make it **idempotent** (safe to run twice, detects completed work and skips it).
     - Also survive sudden death (an OOM kill gives no warning): **crash-only design**, where the crash path is the shutdown path.
  10. **Dev/prod parity.**
      - Close three gaps: time (deploy hours after writing code, not weeks), personnel (whoever writes it deploys it and reads the logs), tools (same backing services everywhere).
      - Gotcha: SQLite in dev behind an ORM, Postgres in prod. `LIKE` is case-insensitive for ASCII in SQLite but case-sensitive in Postgres (`ILIKE` is the insensitive form), so search behaves differently in production with no error.
      - Containers make local parity cheap, but managed proprietary services (DynamoDB; LocalStack only emulates) reopen the gap. Accept that gap only when the managed-service benefit is large.
  11. **Logs are event streams.** Write unbuffered to stdout. The app never handles file paths or rotation; the platform captures, merges (by time across instances), tags and routes to Loki/Datadog etc. The destination becomes config, not code. Metrics and traces (modern observability) are additions on top of this.
  12. **Admin tasks as one-off processes.** Migrations and data fixes run from the same commit, config and dependency isolation as the app, shipped with the code and reviewable — never ad-hoc SQL over SSH. The teacher demoed a Procfile + Overmind formation: schema check -> job server -> port bind -> stdout logs, a one-off auto-archive task, then a clean Ctrl-C shutdown (pool closes, workers drain).

- **Rules of thumb / checklist**
  - Every environment is describable as "the codebase at commit X + config Y".
  - Commit the lockfile; run inside a venv or container.
  - Pass the open-source-the-repo test; move secrets to a secret manager as the team grows.
  - Every external dependency is a URL in config.
  - Images contain no config; keep releases immutable and roll back by re-running an old one.
  - Kill -9 any instance at any time with no data loss.
  - Boot in seconds; handle SIGTERM; make jobs idempotent.
  - Log to stdout.
  - Migrations and scripts ship in the repo.

- **Relevance for Sam**
  - Supplier API keys (Viator, RateHawk) and DB URLs go in env or a secret manager, never in the scripts.
  - Pin pandas/geo libraries with a lockfile; containerise the GeoNames pipelines for reproducibility.
  - Long matching or booking jobs must be idempotent and re-entrant so a crash or OOM on the 16GB laptop means "rerun", not "corrupt": checkpoint by chunk and use upserts.
  - Use the same Postgres locally as in production.
  - Run one-off data fixes as versioned scripts, not ad-hoc SQL.

## Sriniously — How Vercel works Part-2: CDNs and Edge Compute, the backbone network (https://www.youtube.com/watch?v=shh2sjzU-7U)
**You can't make light faster, so shorten the distance: CDNs cache content at internet exchange points near users, and edge compute runs small code there too.**

- **Mental models / first principles**
  - Latency floor = distance / ~200,000 km/s in fibre. You can't move users to San Francisco, so move servers (caches) to users.
  - A CDN is fundamentally a network of **cache servers**. Request -> nearest edge node -> if cached and fresh, serve. If missing or expired, fetch from origin, return, and cache.
  - "Edge location" / PoP (point of presence) comes from networking: the edge of a network, where ISPs interconnect at **internet exchange points (IXPs)**. Major routing decisions happen there, so caching or compute placed there sits on the user's path.
  - Edge compute changes the equation from "deliver static bytes fast" to "run logic fast".

- **Mechanisms**
  - Teacher's numbers: origin ~14–15,000 km away ≈ 120 ms per trip; edge ≤ ~500 km ≈ 2.5 ms in theory, ~10 ms generously with congestion — >10x lower. A page's HTML + JS + CSS + images can all load in under ~100 ms from the edge.
  - Example compute request:
    - From the origin: 120 ms there + 150 ms processing (DB/file work) + 120 ms back ≈ 400 ms.
    - At the edge: 10 + 150 + 10 ≈ 170 ms.
    - (Note: this assumes the 150 ms of work can itself run at the edge. If it needs the origin DB, you pay the long hop anyway.)
  - History:
    - Akamai (1998, MIT researchers; first commercial CDN, largest PoP footprint);
    - AWS CloudFront (2008);
    - Cloudflare (2009, CDN + DDoS/security);
    - Cloudflare Workers and Lambda@Edge (2017);
    - CloudFront Functions (2021).
  - Use cases: caching static assets; traffic routing, load distribution and DDoS absorption (attack traffic spreads across the huge edge fleet); video streaming (HLS chunks cached at the edge, so the first viewer's miss warms the cache for the rest).
  - What building your own CDN would take (why you shouldn't):
    - global PoPs with high-bandwidth uplinks, fast CPUs/RAM/NVMe, and 100s of Gbps of routing gear;
    - your own BGP and **anycast DNS** (many servers share one IP; BGP routes each user to the nearest or healthy one);
    - web-cache software (Nginx/Varnish/Apache-style reverse proxies) plus custom logic, and Linux;
    - an edge runtime, plus HTTP/1.1/2/3 (QUIC) and TLS 1.2/1.3 support, a WAF and pen testing;
    - ISP/IXP partnerships: years of work and billions of dollars.
  - **Cache-Control header** (response):
    - `private` = only the browser may cache (CDNs and proxies must not). `public` = any cache may store it.
    - `max-age=N` = seconds a cache may serve without revalidating. After that, revalidate with ETag / If-Modified-Since: unchanged -> keep serving the cached copy; changed -> new body.
    - `s-maxage=N` = applies only to shared caches (CDNs). E.g. `public, max-age=60, s-maxage=120`: the browser rechecks after 60 s, but the CDN still answers from cache until 120 s, so the origin isn't hit.
    - `no-cache` is commonly seen. Next.js ISR revalidation is built on these headers.
  - **Edge compute**: user code runs at the PoP on each request and can rewrite URL, headers and cookies, or answer directly.
    - Example: read `Accept-Language` and serve the German page immediately instead of English followed by a manual switch.
    - Runtimes: Lambda@Edge ran Node/Python on Firecracker, so it was not truly low-latency. CloudFront Functions and Cloudflare Workers use **V8 isolates** (~0 ms cold start; cheap, concurrent, secure).
    - Limits: no filesystem, no XMLHttpRequest-based libraries such as axios, no raw TCP to databases. Hence **serverless/HTTP databases** (Neon, PlanetScale, Supabase) in the Next.js/Vercel world.
  - Full request lifecycle:
    1. The DNS lookup returns the nearest PoP's anycast IP.
    2. The edge function (if any) runs and may modify the request or response.
    3. Cache lookup: if fresh, return it.
    4. On a miss, check a mid-tier cache (AWS regional edge caches / Cloudflare tiered caching) before the origin.
    5. Fetch from the origin, cache, return.

- **Decisions & trade-offs**
  - Rent a CDN; never build one.
  - Edge compute: lowest latency, but tight runtime limits; best for interception (auth, locale, rewrites, A/B tests), not heavy DB-bound work.
  - `private` vs `public`: personalised responses must never be cached by shared caches.
  - Split TTLs (`s-maxage` > `max-age`) keep browsers fresher while the CDN shields the origin.

- **Rules of thumb**
  - Set Cache-Control deliberately on every response, especially for API JSON that's safe to share.
  - Pair TTLs with ETag/Last-Modified so expiry means cheap revalidation, not full re-downloads.
  - Use tiered or regional caching to cut origin load further.
  - If code runs at the edge, its data must also be reachable over HTTP and from nearby, or the latency win disappears.

- **Failure modes**
  - Marking user-specific responses `public`, so one user's data is served to others from the CDN.
  - Expecting Lambda@Edge-style runtimes to behave like V8 isolates (cold starts).
  - Using TCP DB drivers at the edge.

- **Relevance for Sam**
  - Trameter content (Drive Button widget JS, location/destination pages, mostly-static supplier catalogue data) suits CDN caching with `s-maxage` and tag-based purges.
  - Prices and availability must be `private`/short-TTL or uncached.
  - Locale/currency selection by `Accept-Language` or geo is a natural edge function for a global travel audience.

## Sriniously — Don't be THAT backend engineer, EVER (https://www.youtube.com/watch?v=IrmPU7HRK0o)
**CRUD endpoints are the floor, not the job. A good backend engineer owns the API interface (the controller layer) so it is strictly validated, has sensible defaults, uses predictable errors, is self-documenting and is typed end to end for its consumers.**

- **Mental models / first principles**
  - Layering: route -> handler/controller (validate input, call service, shape the HTTP response) -> service (business logic, DB, email, cache). This video is about the controller: the interface others consume.
  - An API is a product for two audiences: frontend engineers who integrate it and backend colleagues who extend or debug it later. Ambiguity costs them hours (digging through validation -> controller -> service -> DB schema, then pinging you).
  - Design from the client's perspective: read the Figma files and PRD, and understand how the frontend fetches, caches and handles errors (JS `try` = success path, `catch` = unexpected path). The teacher argues backend engineers should study the designs even more closely than frontend engineers (dropdown values, which page calls what, what gets cached).

- **Mechanisms / practices**
  1. **Validate everything, as specifically as possible**: query params, path params and body. Use enums, not `string` (e.g. `type ∈ {active, deleted}`), UUID / Mongo-ID formats, integer ranges, string length bounds, arrays of UUIDs. Two payoffs: **security** (reject bad or malicious input at the entry point) and **documentation** (the validator says what's allowed).
  2. **Sane defaults**: list endpoints should work with zero query params. Default `page=1`, `limit=10/20`, `type=all` in the validation layer. Only genuinely essential params are required (e.g. the search string for a search endpoint). For POST bodies, require only what must be supplied (`name`) and default the rest (`status=pending`, an empty description).
  3. **Be conservative with errors**:
     - A list with no items -> `200` + `[]` (a new user with no books is an expected state, not an error).
     - A single entity by ID not found -> `404`. The client only gets IDs from your list endpoint, so a bad ID really is unexpected.
     - Rule: arrays return empty; single resources return 404. Map status codes to the client's success and catch paths.
  4. **Separate success and error schemas**, rather than one merged `{data, error, success}` envelope with nulls.
     - Success carries the data plus metadata.
     - Errors carry `status`, a human `message`, and a machine `code` the client maps to friendly text.
     - The HTTP status already routes the client to `try` or `catch`, so a `success` flag adds nothing.
  5. **Light future-proofing**: if the design asks for sort-by-name, also allow `sort_by=created_at` and `sort_order=asc|desc` (guessing from sibling endpoints such as `/authors`). Don't expose every column: that's overkill.
  6. **Pagination schema** (offset-based):
     - Always use a deterministic sort; without ORDER BY the DB order is arbitrary between calls.
     - Return `data` plus `total`, `page`, `limit` (the maximum per page, not the count returned) and `totalPages` (e.g. 100 items / limit 10 = 10). That's enough to render numbered page buttons and highlight the current page.
     - Offset paging allows jumping to any page. Cursor paging is more efficient but only offers prev/next.
  7. **Developer experience for colleagues**: comment non-obvious decisions, use architecture docs for big ones, and prefer enums over strings everywhere (frontend too).
  8. **OpenAPI (formerly Swagger) docs** as the single most important habit. A Postman/Insomnia collection is not a standard; OpenAPI is.
     - Render it with Scalar, Stoplight or Swagger UI to show required/optional fields, allowed values, response codes (201/400/403), example payloads and try-it.
     - Raw YAML is painful to maintain, so generate it from code, e.g. ts-rest contracts with Zod schemas (the teacher claims ~70% less YAML; other generators include zod-to-openapi).
     - Generate a typed client (fetch/axios/React Query) from the same contract: autocomplete for endpoints, params and bodies, typed responses, no hand-maintained frontend types drifting from the backend.
     - Claimed (rough) savings: an hour or two of integration time per complex endpoint, and many data-shape bugs avoided.

- **Decisions & trade-offs**
  - 404 vs empty: decide by whether the state is expected in the UI flow.
  - Merged vs split response envelopes: split, because the transport already signals failure.
  - Offset vs cursor pagination: page jumping and simplicity vs efficiency and stability on large or changing sets.
  - Future-proofing: cheap, likely fields only.

- **Rules of thumb / checklist**
  - Can a frontend engineer integrate the endpoint without asking you a single question? If not, it isn't done.
  - Every input: type + format + range + enum. Every optional param: a documented default.
  - Errors: a consistent shape, a stable machine-readable `code`, a helpful `message`.
  - The OpenAPI spec is generated from the same schemas the validator uses, so it can't drift.

- **Failure modes**
  - Accepting any string and letting invalid values reach the DB.
  - Throwing 404 on empty lists, which forces clients into special-case handling.
  - Unsorted pagination returning duplicates or missing rows across pages.
  - Hand-written frontend types drifting from backend responses.
  - Undocumented intent that leaves colleagues guessing.
  - (Note: the video's tone is a rant; the substance is the checklist above.)

- **Relevance for Sam**
  - Normalise third-party supplier payloads into strict internal schemas at the boundary (Pydantic/Zod with enums for booking status, currency and room type), so bad supplier data fails loudly at ingestion.
  - Expose Trameter's location and search APIs with stable pagination (sorted by ID or a cursor for million-row tables), explicit defaults and an OpenAPI spec generated from code. Frontend or Drive Button integrators can then use generated typed clients.
  - Use machine-readable error codes (e.g. `SUPPLIER_TIMEOUT`, `RATE_EXPIRED`) so the booking UI can react correctly.

## Sriniously — 25. Real-Time Backends (https://www.youtube.com/watch?v=wUQryt697cs)
**HTTP only lets the client speak first. Real-time means letting the server push and paying per event rather than per watcher: polling -> long polling -> SSE -> WebSockets, then pub/sub, catch-up IDs and fan-out control across instances.**

- **Mental models / first principles**
  - Request/response cannot carry news the client didn't ask for. Example: a task moved on board 412 must appear live for another viewer.
  - **Polling cost scales with watchers, not events.** An idle board costs as much as a busy one.
    - 10,000 open boards polling every 3 s ≈ 3,333 req/s, each passing through LB -> auth middleware -> logging -> DB -> JSON, and mostly answering "no change".
    - Average delay = half the interval (1.5 s). Polling every 1 s triples cost to ~10k req/s.
    - On mobile, each poll wakes the radio, draining battery and data.
  - Polling is perfectly fine for internal tools with few users.
  - **A persistent connection is state** that lives in one process. This contradicts the stateless-instance design, so you need a cross-instance messaging layer.

- **Mechanisms**
  - **Long polling**: the server holds the request until there's news, responds, and the client immediately re-requests. RFC 6202 documents its flaws: messages can be missed or delayed in the gap while the client processes and reconnects. Average latency is ~1 network transit; worst case is >3.
  - **Server-Sent Events (SSE)**: an ordinary GET; the response is `200` with `Content-Type: text/event-stream` and never closes. The server writes small text events with four fields: `id`, `event` (e.g. `task_moved`), `data`, `retry` (reconnect delay).
    - The browser's EventSource auto-reconnects and sends a **`Last-Event-ID`** header so the server can replay what was missed.
    - Pure HTTP, so it works everywhere. Used by Uber's driver push platform, LinkedIn messaging (typing indicators, read receipts) and **every LLM token-streaming API**.
    - Limitation: one-way. Client actions still go out as separate POSTs, which is often fine.
  - **WebSockets**: start as an HTTP GET with `Upgrade: websocket` and a `Sec-WebSocket-Key` (16 random bytes, base64).
    - The server appends a fixed spec GUID, SHA-1 hashes, base64-encodes and returns `Sec-WebSocket-Accept` with **101 Switching Protocols**. This proves a WebSocket-aware peer answered (not a cache replaying an old 200); it is not security.
    - After that, no headers, methods or status codes: a bidirectional stream of **frames**.
      - Frame header: FIN bit, 3 reserved bits, 4-bit opcode (0 continuation, 1 text, 2 binary, 8 close, 9 ping, 10 pong), MASK bit, 7-bit length (<126 inline; 126 -> 2-byte length; 127 -> 8-byte length). Small messages carry a 2-byte header, versus hundreds of bytes of HTTP headers per poll.
      - **Masking**: client->server payloads are XORed with a random 4-byte key sent in the clear. It is not encryption; it prevents cache-poisoning attacks in which attacker-chosen bytes look like an HTTP request to naive proxies. The key must be fresh and random per frame.
    - **Heartbeats (ping/pong)**: TCP gives no signal when a client disappears (lift, Wi-Fi->mobile switch), so idle and dead connections look the same. Ping, and close the connection if no pong arrives within ~30 s, to free resources.
  - **Capacity experiments** (16-core, 61 GB Linux box; minimal Go server):
    - **File descriptors**: every socket is a file. `ulimit -n` = 1,024 -> stalled at ~1,017 connections (stdin/out/err, the listener and epoll use the rest). Clients saw timeouts, not refusals; the server logged "too many open files" and stayed "healthy" for health checks.
    - Go's runtime raises its own soft limit at startup (to max-1, showing 1,048,575). Python and Node keep 1,024, so the "1k connection wall" depends on the runtime.
    - **Ephemeral ports**: the next stall was at ~28,190 ("cannot assign requested address"). The **client** ran out of source ports (range 32768–60999 ≈ 28,232, minus ports already in use). A TCP connection is identified by the 4-tuple (src IP, src port, dst IP, dst port), so the "65k connections per server" myth is false. Fix: dial from more client IPs (3 IPs x 25k = 75k, zero failures). Load tests therefore need multiple generator machines.
    - **Memory**: ~9.6–9.9 KB of process heap per idle connection (7.5 MB -> 722 MB at 75k), mostly the per-connection goroutine plus buffers; kernel socket memory comes on top. 1M connections ≈ <10 GB heap. At extreme scale, use one epoll loop instead of a goroutine per connection.
  - **Multiple instances**: A on instance 1 and B on instance 2 don't see each other's updates.
    - **Sticky sessions don't fix this**: they only pin a client to an instance; the instances still don't talk.
    - Fix: **pub/sub**. The receiving instance publishes to a topic (e.g. `board:412`) on Redis, NATS or Kafka; every subscribed instance writes frames to its own local sockets.
  - **Delivery semantics**: Redis Pub/Sub is fire-and-forget, **at-most-once**. Messages published while an instance is restarting during a deploy are lost. For **at-least-once**, use a stored, acknowledged log (Redis Streams, Kafka, a queue).
  - **Catch-up**: number every message with a sequence. On reconnect the client says "I have up to N" and the server replays from N+1 (Uber's approach, and the same idea as SSE `Last-Event-ID`). Disconnects (network switches, closed lids, LB idle timeouts) are normal events.
  - **Fan-out**: work = events x subscribers. One move on a board watched by 30k people = 30k writes. Discord measured 0.9–2.1 s to publish one message to a 30k-member guild; their fix was to spread fan-out across the machines that own the recipients.
  - **Reconnect storms**: an instance with 50k sockets dies -> 50k clients reconnect simultaneously and all request initial state. Slack's fix was to make connecting cheap by caching bootstrap state at the edge so the storm never reaches the core.
  - Not covered: presence and collaborative editing (CRDTs).

- **Decisions & trade-offs**
  - Polling: simplest; acceptable for small or internal use.
  - SSE: server->client streams (notifications, progress, LLM tokens) with built-in resume.
  - WebSockets: true bidirectional, low-overhead, chatty use.
  - Redis Pub/Sub (fast, lossy) vs Streams/Kafka (durable, replayable).

- **Rules of thumb**
  - Give every event an ID or sequence and support resume-from-ID.
  - Use heartbeats with timeouts.
  - Raise the FD limit (or use Go); budget ~10 KB of RAM per connection.
  - Load-test from multiple source IPs.
  - Plan for reconnect storms (jittered backoff, cached bootstrap state).
  - Keep delivery semantics explicit.

- **Failure modes**
  - Silent dead sockets.
  - A server that looks healthy but accepts no new connections (FD exhaustion).
  - Events lost during deploys with Redis Pub/Sub.
  - Sticky sessions mistaken for a cross-instance solution.
  - Fan-out latency on huge rooms.

- **Relevance for Sam**
  - For long data jobs and supplier-search aggregation, SSE is the simplest way to stream progress or results as each provider responds, with event IDs for resume. It also matches how LLM features stream.
  - Booking-status updates (pending -> confirmed from a supplier webhook) can go out over SSE or WebSocket through pub/sub. Use a durable stream (not plain Redis Pub/Sub) for anything a user must not miss, such as a booking confirmation.

## Sriniously — How Vercel works Part-4: Splitting apart and deploying Next.js (https://www.youtube.com/watch?v=RrEmS4f3xX0)
**A Next.js build is not one server. Vercel splits the output by rendering strategy and runtime across S3+CDN, AWS Lambda, edge workers and a gateway that routes between them.**

- **Mental models / first principles**
  - History: static HTML docs (1989–90; CDNs fit perfectly) -> CGI scripts -> PHP.
  - **MPA**: every route maps to server logic (`/about` -> `about.php`) returning fresh HTML. Hard navigation; data fetched server-side.
  - **SPA** (React/Angular/Vue/Svelte): one `index.html` plus route-level lazy-loaded JS chunks. A client-side router watches the URL (soft navigation); data comes via AJAX/JSON APIs and the client patches the DOM.
  - Next.js is a server-first hybrid of the two, so how each route renders decides where it runs.
  - Rendering strategies:
    - **SSG**: pages rendered to HTML/CSS/JS at `next build` and served as static files with no server work.
    - **ISR**: SSG without the build-time cost. Pages render on first request and are stored in the ISR cache, with time-based revalidation (e.g. invalidate after a week). This fixes huge build times for sites with millions of pages (a Medium-scale blog).
    - **SSR**: render on every request with DB calls. Freshest and most infrastructure-heavy; for dashboards and tickers.
  - Vercel's 2025 Ship conference brought no architectural change; "fluid compute" bills for active CPU time slices and runs concurrent invocations per instance, unlike per-invocation Lambda billing.

- **Mechanisms (the teacher's reconstruction; many internals are undocumented)**
  - Git push on the tracked branch -> new deployment -> `next build` -> a `.vercel` output directory -> artifacts fan out to different infrastructure:
    - **SSR pages / route handlers, Node runtime (default)** -> AWS Lambda in a single region (Vercel's default is Washington, D.C. / us-east, near most companies' data, to avoid DB round trips).
    - **SSR pages / route handlers with `export const runtime = 'edge'`** -> edge workers (the teacher says Cloudflare Workers): ~zero cold start but no fs, no full Node crypto, no TCP DB drivers.
    - **SSG pages** -> HTML/JS/CSS uploaded to S3 (durable object storage) and served directly.
    - **Function limit**: Vercel caps functions per app (the teacher recalls ~12, plan-dependent). Routes with matching config (runtime, timeouts, revalidation) are **bundled into one function** (e.g. `/about`, `/profile` and `/dashboard` SSR in one Lambda).
    - **Middleware** (`middleware.ts`): originally deployed as an edge worker worldwide, the first code every request hits (redirects, blocking, light auth). A Node runtime for middleware is now also supported.
    - **Static assets / `public/`** (images, fonts, bundled JS/CSS) -> S3 and cached on the CDN (provider undocumented).
    - **Image optimisation** (`sharp`: resize, quality, format per device/network) -> a separate function or server.
    - **Routing**: build output becomes a routing table that drives an API-gateway-like layer (the "gateway service"), which sends each request to the CDN/S3, the right Lambda or an edge worker.
  - **OpenNext**: a community project to self-host Next.js with Vercel parity on AWS, Cloudflare or Netlify (function splitting, warmer functions, external middleware as Lambda@Edge, ISR and on-demand revalidation). Alternatives: Docker or a plain VPS, which lose some features unless configured.

- **Decisions & trade-offs**
  - SSG (fastest, stale until rebuilt) -> ISR (fast, bounded staleness, no giant builds) -> SSR (fresh, costly).
  - Node vs edge runtime: full APIs and region-locked latency vs global, instant start and restricted APIs. Put compute near the data unless the logic needs no data.
  - Vercel vs self-hosting via OpenNext: convenience vs keeping everything inside your own AWS account.

- **Rules of thumb**
  - Choose rendering per route by how fresh the data must be.
  - Keep Node functions in the same region as the database.
  - Use edge only for lightweight, data-free logic (redirects, auth gates, locale).
  - Remember that everything on Vercel is serverless: no persistent connections or in-process state, so use serverless or HTTP DB drivers and poolers.
  - (Note: the provider mappings — Cloudflare Workers for edge, the exact function limit — are the teacher's best inference from docs and community sources, and may have changed.)

- **Failure modes**
  - Using SSG for millions of pages, causing endless builds.
  - Marking DB-heavy routes as edge (incompatible drivers, or far from the DB).
  - Expecting long-lived connections or in-memory caches to persist across invocations.
  - Unexpected costs from SSR on high-traffic pages that could be ISR.

- **Relevance for Sam**
  - For Trameter's location and destination pages (millions of GeoNames entities), ISR is the right fit: render on first hit, cache, revalidate periodically. SSG of all pages would be impractical.
  - Booking and pricing pages are SSR or client-fetched, with Node functions co-located with the DB region.
  - Long-running data pipelines do not belong in Vercel functions.

## Sriniously — 15. Full text search using Elasticsearch for blazingly fast search (https://www.youtube.com/watch?v=7_sovzAhRSM)
**`ILIKE '%term%'` is a full scan with no notion of relevance or typos. Search engines invert the problem: index terms -> documents, then rank with BM25.**

- **Mental models / first principles**
  - The problem: e-commerce search via `SELECT * FROM products WHERE name ILIKE '%laptop%' OR description ILIKE '%laptop%'` takes ~50 ms at 5k products and ~30 s at millions. The business also wants **relevance** (a MacBook before a laptop bag) and **typo tolerance** ("laptpo" -> laptop).
  - The DB-as-librarian analogy: it finds every match but must check every book (row) character by character, and returns results in arbitrary order with no idea which matter most.
  - The **inverted index** flips the lookup: at ingest time, tokenize each document and store term -> [documents + positions]. A query looks up its terms directly instead of scanning content. Information-retrieval research on this goes back to the 1960s.
  - Apache **Lucene** implements it; **Elasticsearch** (and others) build on Lucene. Postgres also offers built-in full-text search.

- **Mechanisms**
  - Elasticsearch stores JSON **documents** in an **index** (MongoDB-like). Field mappings matter:
    - `text` = analysed/tokenised for full-text matching;
    - `keyword` = exact match (e.g. `sentiment: positive|negative`, a filter or aggregation field).
  - **Relevance scoring with BM25**, built from:
    - term frequency (how often the term appears in the document);
    - document frequency (how common the term is across all documents; rarer = more informative);
    - document length (normalises long vs short documents).
    - **Field boosting**: a match in title > description > body, configurable per query in the JSON query DSL.
  - Typo tolerance and type-ahead/autocomplete (Google- or Amazon-style suggestions) come from fuzzy matching on the index.
  - Elasticsearch is also the "E" in the **ELK stack** (Elasticsearch, Logstash, Kibana) for log search and analytics.
  - Teacher's demo: 50,000 review rows (id, review, sentiment) in Neon Postgres and Elastic Cloud, both in US-West.
    - Loader: env-var config check; create the table if missing; filter invalid rows; insert into Postgres **in batches of 1,000** (Neon rejects one giant insert); recreate the ES index with mappings; one **bulk** insert of 50k docs.
    - The API runs both searches concurrently and **streams** each result to the UI as it finishes, so the slow one doesn't delay the fast one.
    - Results: "laptop": ES ~1 s vs Postgres ILIKE ~3–4 s; another term: ES ~0.5 s vs Postgres ~7.5 s, with the same ~8k hits.
    - (Note: only 50k rows, wildcard query strings on the ES side, and a serverless Postgres with no trigram/FTS index. That is the naive baseline, not Postgres's best. A `pg_trgm` GIN index or `tsvector` FTS would narrow the gap considerably.)

- **Decisions & trade-offs**
  - Postgres FTS vs Elasticsearch:
    - Start with Postgres FTS if Postgres is already your stack: one fewer system, and no sync problem.
    - Choose Elasticsearch when you need advanced relevance tuning, typo tolerance or autocomplete at scale, or the company already runs ELK.
  - Elasticsearch is a **secondary index**: you must keep it in sync with the source-of-truth DB (an implied cost; the teacher doesn't dwell on it).
  - Depth of learning: master databases (indexes, query tuning), since they are ~99% of backend work. For Elasticsearch, docs- or LLM-level snippets cover most needs; go deeper only when optimising.

- **Rules of thumb**
  - Never ship `ILIKE '%x%'` search over large tables.
  - Map exact-match fields as `keyword` and searchable prose as `text`.
  - Boost title/name fields.
  - Bulk-load in batches.
  - Run independent backends concurrently and stream partial results.

- **Failure modes**
  - Leading-wildcard LIKE can't use B-tree indexes: full scans that grow linearly with data.
  - Unranked results bury the relevant item.
  - Exact-match search fails on typos.
  - Single huge inserts hit provider limits.

- **Relevance for Sam**
  - Location search over ~6M GeoNames rows is the textbook case. Use an inverted index (Postgres `tsvector`/`pg_trgm` or Elasticsearch/OpenSearch) with typo tolerance, alternate-name fields and boosting for exact name, admin area and population.
  - Map country codes and feature classes as `keyword` filters.
  - Fuzzy matching across providers (Viator/RateHawk names vs GeoNames) can reuse the same machinery.
  - The demo's batched ingest and concurrent-query streaming patterns fit 16GB-laptop pipelines directly.

## Sriniously — 7. Serialization and Deserialization for backend engineers (https://www.youtube.com/watch?v=vzg90tY3uM0)
**Different languages have incompatible in-memory types, so machines agree on a shared wire format. Serialise into it on the way out and deserialise into native types on the way in.**

- **Mental models / first principles**
  - A JavaScript client (dynamic, untyped) and a Rust server (strict, compiled) cannot share in-memory objects. They need a **language-agnostic common standard** both sides agree on.
  - **Serialisation** = native data -> common format (for transmission or storage). **Deserialisation** = common format -> native types (a JS object, Rust struct, Go struct).
  - OSI mental model: you work at the application layer. The JSON you emit goes down the stack (frames, IP packets, bits over fibre) and back up on the other side. As a backend engineer, assume JSON in, JSON out; the intermediate encodings aren't your concern. A high-level OSI overview is enough.

- **Mechanisms**
  - Format families:
    - **text-based**: JSON (≈80% of client–server HTTP traffic in the teacher's estimate), XML, YAML;
    - **binary**: Protocol Buffers (most popular; used by gRPC), Avro and others.
  - JSON rules: an object in `{}`; **keys are always double-quoted strings**; values are string, number, boolean, array, nested object (or null). Arbitrary nesting (e.g. `address: {country, phone}`).
  - JSON is human-readable by design, and also used for config files and structured logs.
  - Demo (Burp Suite HTTP history): `POST /api/books` with a JSON body `{id, title, author}`; the server parses it, stores it and returns JSON with an array of book objects; the JS client parses and renders them.

- **Decisions & trade-offs** (partly beyond the video, flagged)
  - Text formats: readable, debuggable, universal, but larger and slower to parse. Binary formats (Protobuf): compact, fast, schema-enforced, but not human-readable and need shared schema files. (Note: the video only names these categories; the trade-off detail is standard knowledge.)
  - The series standardises on HTTP + JSON + Postgres as the most-used defaults; learn the alternatives (gRPC, WebSockets, Mongo/Dynamo) later.

- **Rules of thumb**
  - Treat the wire format as a contract. Validate after deserialising: parsing JSON successfully doesn't make the data valid (ties to the validation advice in "Don't be THAT backend engineer").
  - Remember serialisation has a cost: Part-1 of the scaling series lists JSON serialisation and large payloads as common hidden latency sources.

- **Failure modes / gotchas** (general knowledge, beyond the video)
  - JSON has no integer/float distinction, no dates and no decimals: big IDs and money can lose precision in JS (use strings or minor units). Dates should be ISO-8601 strings.
  - Single-quoted keys or trailing commas are invalid JSON.

- **Relevance for Sam**
  - Supplier APIs (Viator, RateHawk, Coras) each serialise differently (JSON, sometimes XML). Deserialise them into strict internal models at the boundary.
  - For multi-GB location datasets on a 16GB laptop, prefer streaming or line-delimited JSON, or columnar binary formats (Parquet/Arrow), over loading giant JSON documents into memory. (Inference; not from the video.)
  - Represent prices as integer minor units or strings with an explicit currency.

