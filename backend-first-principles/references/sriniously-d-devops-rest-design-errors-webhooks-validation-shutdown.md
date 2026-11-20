# sriniously-4 — cross-cutting principles

1. **Validate at the boundary, fail fast with precise errors.** Validate and transform all input (body, query, path, headers, and third-party responses) before business logic; cast query strings; normalise identifiers. Validate required config at boot and refuse to start rather than 500 at runtime. [Sriniously — 9. Validations; 16. Error Handling]
2. **Assume at-least-once and make side effects idempotent.** Retries, lost acks and replays are normal. Dedupe on a stable ID (webhook delivery ID, idempotency key) with a unique constraint, inserted in the *same transaction* as the side effect: work-then-record double-processes, record-then-work skips. POST is not idempotent by nature, so creates (bookings, payments) need explicit keys. [29. Webhooks; 11. REST API Design]
3. **Never trust event order or event payloads as current state.** Treat notifications as signals; re-fetch current state from the source of truth, or apply only newer versions (`updated_at`/version check) in one transaction. For syncing another system's data, pull from its ordered event log with a stored cursor instead of mirroring pushes. [29. Webhooks]
4. **Record intent durably first, act asynchronously (outbox / write-ahead pattern).** Commit the business change and the outbound event in one DB transaction, then let a worker deliver with exponential backoff + jitter, honoring Retry-After, with per-destination queues. Keeps third-party latency and failures out of your request path. [29. Webhooks; 23. Object Storage (pending-row two-phase upload)]
5. **Keep request handlers thin; push slow work to background workers.** Ack webhooks within the provider deadline (GitHub 10 s, Shopify 5 s, Slack 3 s) and enqueue. Do not make your server a slow file relay or a CDN. [29. Webhooks; 23. Object Storage]
6. **Stream, never buffer, large data.** Reading whole uploads/files into memory OOM-kills containers (20 × 100 MB uploads ≈ 2 GB). Pipe chunks; wrap bodies in size limits; enforce sizes at the storage layer with pre-signed POST policies. cgroup `memory.max` is what kills you. [23. Object Storage; 30. DevOps]
7. **Servers must be stateless and disposable; state lives in purpose-built stores.** Local disk is ephemeral, unshared across instances, fixed-size and single-copy. Files go to object storage, data to the DB; servers are replaced, never patched (immutable infra, no SSH fixes). [23. Object Storage; 30. DevOps]
8. **Your database is the index; object storage holds bytes.** Generate keys yourself (never user filenames), put high-entropy parts early, encode ownership in prefixes, never LIST in the request path, reconcile DB and bucket with lifecycle rules plus sweeper jobs. Use conditional writes (`If-None-Match: *`, `If-Match` ETag) for exactly-one-writer and compare-and-swap. [23. Object Storage]
9. **Durability is not backup.** Replication/erasure coding protects against hardware loss, not against your own bad deletes or overwrites. Enable versioning and replicate to a separately credentialed account. [23. Object Storage]
10. **Centralise error mapping; never leak internals.** One global error middleware maps typed errors to HTTP (validation/unique → 400, no-rows/FK → 404, unknown → generic 500). No raw DB messages to clients, no user enumeration on login, no PII/secrets in logs (log IDs and correlation IDs). [16. Error Handling]
11. **Classify failures and respond by class.** Transient → retry with exponential backoff without amplifying load; non-recoverable → contain and degrade (cache, disable features, fallbacks). Treat every external dependency as a failure domain with timeouts. Monitor latency and business metrics, not only error rates. [16. Error Handling]
12. **Health checks must reflect real capability, and probes must not cascade.** Readiness = can I serve now (gates traffic); liveness = am I stuck (restarts). A liveness probe that hits a slow DB restarts healthy pods in waves and makes the DB slower. [30. DevOps; 16. Error Handling]
13. **Shut down gracefully.** Handle SIGTERM/SIGINT: fail readiness, stop accepting, drain with a bounded timeout (~30 s, under the orchestrator's grace period), commit or roll back transactions, release resources in reverse acquisition order. SIGKILL cannot be caught. [19. Graceful Shutdown]
14. **Small, frequent, reproducible deploys are both faster and safer.** Trunk-based development + feature flags; pinned toolchains and lock files; commit hash stamped on artifacts, images and a /health endpoint; one change at a time so failures are traceable. Rolling deploys run two versions at once, so schemas and messages must be backward compatible. [30. DevOps]
15. **Database connections are scarce, stateful resources.** Always pool; total pool size across all instances/workers must stay under `max_connections`. Serverless/edge compute needs a pooler/proxy or serverless DB (HTTP/WebSocket drivers). Never connect to a DB from the browser. [How Vercel works Part 3; 3. What is a Backend]
16. **Design interfaces before code, and be consistent.** Plural lowercase resources, camelCase JSON, identical field names across resources, sane defaults (page 1, limit 10–20, sort createdAt desc), deterministic sorting, empty list = 200 not 404, side-effecting non-CRUD operations as `POST /resource/{id}/action`, OpenAPI docs from day one. [11. REST API Design]
17. **Authenticate machine-to-machine traffic cryptographically and defend outbound calls.** Verify HMAC over raw bytes before parsing, constant-time compare, timestamp-bound replay window. When calling user-supplied URLs, resolve and block private ranges on every send and never follow redirects (SSRF). Prefer short-lived OIDC credentials over long-lived CI secrets. [29. Webhooks; 30. DevOps]
18. **Keep secrets and privileged logic server-side.** Browser code is shipped to and inspectable by users, runs sandboxed, is CORS-limited and may run on 256 MB devices. Third-party API keys, DB access and heavy compute belong on the backend. [3. What is a Backend; How Vercel works Part 3]
19. **Measure reliability in user terms and budget for failure.** SLI → SLO → SLA; 99.9%/month ≈ 43 min downtime; spend the error budget on deploys and experiments, freeze when exhausted; blameless postmortems; stop the bleeding (rollback, flag off) before root-causing. [30. DevOps]
20. **Pin and scan your supply chain; keep infra in code with protected state.** Pin base images and dependencies, scan for CVEs (xz backdoor lived in a release tarball, not the repo). Terraform state remote, locked, encrypted, never committed; console hotfixes create drift that the next apply reverts. [30. DevOps]

---

## Sriniously — 30. DevOps for backend engineers (https://www.youtube.com/watch?v=_E8X5RIJfQ4)
**Make deployment small, cheap and boring so you can ship many tiny changes a day; speed and stability come together, not as a trade-off.**

- **Mental models / first principles**
  - DevOps exists because "dev" (rewarded for shipping) and "ops" (rewarded for uptime) had opposing incentives, which pushed releases to rare, huge, risky events. The fix was cultural plus tooling: infra in code, one-command build and deploy, feature flags, shared observability.
  - DORA's four metrics: change lead time (commit to live), deployment frequency, change failure rate (% of deploys causing incidents), failed-deployment recovery time. 2024 top performers: deploy on demand, commit-to-prod under a day, ~5% failure, recovery under an hour. Bottom group: deploy every 1–6 months, ~40% failure, a week to recover.
  - Why frequent = stable: a monthly deploy bundles ~200 commits from 10–20 people, so finding the culprit takes a day and the author has forgotten the code. A one-commit deploy points straight at the cause while it is fresh.
  - Immutable infrastructure: never SSH in and patch a running server; change code/config and replace instances. Hand-maintained "snowflake" servers concentrate knowledge in one or two people. Containers made replacement cheap.
- **Mechanisms**
  - CI is a *practice* (everyone merges to a shared branch at least daily, every merge auto-tested); the CI server only enforces the testing half. Long-lived branches drift and produce hours of merge conflicts.
  - Branching: git-flow (develop/release/hotfix/main) was designed for shipped, multi-version software; its own author recommends something simpler for continuously deployed web apps. Use trunk-based development: short-lived (1–2 day) feature branches merged into main; long features hide behind feature flags in production.
  - Reproducible build: pin the compiler/runtime version and commit the lock file (go.sum, package-lock + `npm ci`) so the same source gives a byte-identical artifact. Stamp the commit hash into the artifact, image tag and a /health or /info endpoint so you can verify what is running. Deployments use commit hash (and build ID), not semver; semver is for libraries consumed by others.
  - Pipeline stages ordered cheap-and-fast first (compile, unit tests, fail fast and notify), then slower (integration tests against real DBs, staging), then production.
  - GitHub Actions vocabulary: workflow (YAML in .github/workflows), event (push/PR/schedule/manual), job (steps on one machine; jobs can run in parallel or depend on each other), runner (fresh VM each time, hence every workflow starts with checkout and must reinstall or restore cache), step.
  - Cloud credentials from CI: prefer OIDC over long-lived repo secrets. GitHub issues a short-lived signed token naming repo/branch/workflow; the cloud is configured to trust only tokens for that repo+branch and returns credentials that expire when the job ends.
  - Containers are not small VMs. A VM has its own kernel (strong isolation, heavy: GBs of RAM, slow boot). A container is a normal host process that the kernel gives a private view via:
    - **Namespaces** (what you can see): PID, mount, network, UTS (hostname), IPC, user, cgroup, time. Created via syscalls `clone`, `unshare`, `setns`. Liz Rice's ~76-line Go "containers from scratch" shows the whole thing: clone with new UTS/PID/mount namespaces, set hostname, chroot into an Ubuntu folder, mount /proc, exec a shell.
    - **cgroups** (how much you can use): a folder under /sys/fs/cgroup with files like memory.max, cpu.max, pids.max; write a PID into cgroup.procs to join. A Kubernetes 512 MB limit is literally the kubelet writing that into memory.max; exceeding it triggers the kernel OOM killer ("OOMKilled").
    - **Layered filesystem** (OverlayFS): read-only lower layers plus a thin writable upper layer; writes copy-up. Ten containers from one image share all lower layers. Writes vanish when the container stops, so persist state elsewhere.
  - Image = layer blobs (content-addressed by SHA-256) + config JSON (layer order, cmd, env, workdir, ports) + manifest JSON; image identity is the manifest's hash. Registry stores each layer once; push uploads only missing blobs, pull fetches only missing layers, so small code changes pull in seconds. Tags (e.g. myapp:v2.3.1) point to digests.
  - `docker run` → dockerd → containerd (image/lifecycle management) → runc (reads JSON spec, makes the syscalls, writes cgroups, execs, exits leaving a shim). The OCI standard separates image format, runtime and registry, which is why Kubernetes can run Docker-built images via containerd/CRI with no Docker daemon.
  - Single-machine production: systemd unit (run as user X, restart on failure, logs to journal via journalctl) + reverse proxy (Nginx/Caddy on 80/443, TLS termination) + DNS A record + firewall. TLS via ACME/Let's Encrypt: client proves domain control via HTTP token on port 80 or DNS record; certs last 90 days and auto-renew at ~60 days so a stolen key is short-lived. Caddy does all of this automatically.
  - Multi-machine without Kubernetes: Kamal over SSH builds, pushes, pulls, starts new container beside old, waits for /health, flips proxy, stops old.
  - Kubernetes is declarative: you state desired end state; controllers loop comparing desired vs actual. Parts: API server (only thing touching etcd), scheduler, controller manager, kubelet per node, kube-proxy. Pod = one or more containers sharing network namespace and volumes (sidecars like log shippers); each pod has its own IP. ReplicaSet keeps N pods; Deployment owns ReplicaSets and rolls templates; Service gives a stable virtual IP/DNS name over ephemeral pods via label selectors; Ingress/Gateway API maps host+path to services. ConfigMap for non-sensitive config, Secret for credentials (only given to pods that request them). Resources: request (reserved) vs limit (hard cap via cgroups).
  - Rolling update governed by maxSurge/maxUnavailable (default 25% each); with 3 replicas: add one new, wait ready, remove one old, repeat. Old ReplicaSet kept at 0 replicas for instant rollback. Readiness gates traffic so a broken image only stalls rollout.
  - Probes: readiness (remove from endpoints, no restart), liveness (restart after repeated failures), startup (hold the other two until a slow process boots).
  - HPA scales pod count between a min and max based on average CPU against a target.
- **Decisions & trade-offs**
  - Start small: one machine with systemd/Docker Compose or a PaaS (Render, Railway, with buildpacks/Nixpacks that build images without a Dockerfile); then several machines with Kamal; adopt Kubernetes only once you have earned it.
  - Continuous delivery (always deployable, one button) is a technical property; continuous deployment (no button) is a business choice. You can have delivery without deployment, never the reverse.
  - Release strategies: recreate (simple, causes downtime); rolling (zero downtime but old and new versions serve simultaneously, so APIs/schemas must tolerate both); blue-green (one version at a time, instant rollback, roughly double infra cost while both run); canary (route ~10% to new version, compare error rate/latency, widen 20→50→75→100%); progressive delivery automates canary analysis and rollback.
  - GitOps (ArgoCD, Flux): desired state as declarative files in Git; an in-cluster agent pulls and reconciles, reverting manual drift. Each deploy is a PR changing an image digest; the CI pipeline never holds cluster credentials.
  - Environments are the same image with different config (12-factor). Preview environments per PR; promotion moves the same artifact forward through approval gates.
  - IaC tool kinds: provisioning (Terraform / OpenTofu fork: plan, review, apply), configuration management (Ansible), image baking (Packer + cloud-init). Test: could you rebuild prod identically in a fresh account in an hour?
- **Rules of thumb / checklists**
  - Deploy as often and as small as possible; one change at a time.
  - Dockerfile ordering: copy only the dependency manifest, install deps, then copy source, so frequent code edits do not invalidate the dependency layer. Use multi-stage builds for compiled languages (compiler stage → tiny runtime stage with only the binary).
  - Good image: small, one process (no init/cron inside), non-root, pinned base (never `latest`), vulnerability-scanned.
  - Terraform state: remote store with locking, never committed (it contains secrets), protected/encrypted.
  - SLI = user-facing measurement (e.g. % requests succeeding, % under 300 ms); SLO = target over a window (99% over 30 days); SLA = contract with penalties. 99.9% monthly ≈ 43 min of allowed failure; 99.99% ≈ 52 min/year (~4.5 min/month). Each extra nine is 10x the promise.
  - Error budget: spend it on deploys, migrations and experiments; once exhausted, freeze deploys and work on reliability until it recovers.
  - On-call rotation with a primary and a backup. Incident roles: commander (coordinates, does not fix), fixer(s), communicator (status page, management). First goal is to stop the damage (roll back, flip a feature flag), root cause second. Write a blameless postmortem: timeline, impact, root cause, actions. Blame systems, not people, or you never learn why the wrong action seemed right.
  - Design for failure; chaos engineering (kill pods, remove nodes, inject latency) to prove safety mechanisms work before a real outage.
- **Failure modes & gotchas**
  - "Works on my laptop" failures: older system libs, missing CA certs or timezone DB on server, Node 18 vs 22. Ship the environment (container), not just code.
  - Liveness probe that hits the database: a slow DB makes probes time out, kubelet restarts healthy pods in waves, adding more load to the DB. Liveness checks only "is the process stuck"; readiness checks "can I serve now".
  - Lost Terraform state makes Terraform plan to recreate everything; two people with separate state copies overwrite each other. Console hotfixes create drift that the next `apply` silently reverts (the demo: port 22 opened by CLI gets closed again).
  - Long-lived repo secrets are readable by every job; a leak grants access until rotated.
  - Supply chain: dependencies vanish/change and break builds, or get poisoned (2024 xz backdoor was inserted into the release tarball, not the source repo, so code reviewers missed it). Pin everything and scan.
  - Cloudflare's Nov 2025 outage: a DB permission change caused duplicate rows, a config file doubled in size, and a consumer with a hard size limit failed. Generated config needs validation and size limits handled gracefully.
- **Relevance to Sam's work**
  - Pipelines on a 16 GB laptop: cgroup memory limits explain OOMKilled; set container memory limits deliberately when running chunked jobs in Docker, and remember container writes are lost on exit, so write outputs to mounted volumes or object storage.
  - Supplier/booking services: rolling deploys mean two versions run at once, so payloads, queue messages and DB migrations must be backward compatible. Blue-green suits booking flows needing a single version and instant rollback.
  - Stamp commit hash on every pipeline artifact/output dataset so you can trace which code produced which file.
  - Feature flags let you merge half-built supplier integrations to main daily without exposing them.

## Sriniously — 11. Complete REST API Design (https://www.youtube.com/watch?v=RG6q57DwV8Y)
**Design the API interface first, from the product's nouns, and follow one consistent REST convention so consumers never have to guess.**

- **Mental models / first principles**
  - REST came from a scaling crisis: the early web (Berners-Lee's URI, HTTP, HTML, server, browser) grew faster than it was designed for. Roy Fielding (Apache co-founder) proposed constraints, helped write HTTP/1.1, and named the style REST in his 2000 dissertation.
  - Fielding's constraints: client–server separation (independent evolution); uniform interface (resource identification, manipulation through representations, self-descriptive messages, hypermedia as engine of state); layered system (each layer only sees the next, enabling proxies and load balancers); cacheability (responses must say whether they are cacheable); statelessness (every request carries everything needed, so any server behind a load balancer can handle it); code on demand (optional).
  - "Representational state transfer": a resource (e.g. a user, a cart with items, quantities and total) has state; client and server exchange *representations* of it (JSON, XML, HTML) over standard HTTP methods.
  - Standards exist to remove guesswork. If you violate them, an integrator's only options are reading your code or trial and error.
- **Mechanisms**
  - URL anatomy: scheme, authority/subdomain (convention: `api.example.com`), version (`/v1`), path (resource), query params, fragment. A `/` expresses hierarchy: collection, then member, then action.
  - Resource paths are always plural and lowercase: `/v1/books`, `/v1/books/{id}` (still plural for a single item). No spaces or underscores; slugs are lowercased with hyphens (`harry-potter`).
  - Idempotency = repeating the same request produces the same *side effect* on the server as doing it once (not necessarily the same response). GET: idempotent (others creating data in between does not count). PUT/PATCH with the same payload: name A→B, then B→B, so idempotent. DELETE: first call deletes, later calls return 404 but change nothing, so idempotent. POST: not idempotent; replaying a create makes a new row with a new DB-generated ID each time. (note: by the HTTP spec PUT is idempotent but PATCH is not guaranteed to be; a PATCH like "increment count" or "append item" is not idempotent. Treat the teacher's claim as true only for set-field-to-value patches.)
  - PATCH = partial update (send only changed fields); PUT = full replacement of the representation. In JSON SPAs, PATCH is the usual choice. Mixing them is tolerable internally but misleading in a public API.
  - POST is the open-ended method for non-CRUD custom actions (send email, archive, clone).
  - Design workflow: read wireframes / requirements → extract nouns as resources (projects, users, organizations, tasks, tags) → DB schema → list actions per resource (CRUD + custom) → design the interface in an API client (Insomnia/Postman) or OpenAPI before writing code.
  - Standard route set per resource:
    - `GET /organizations` list → 200
    - `POST /organizations` create → 201 + created entity (server-generated id, createdAt)
    - `GET /organizations/{id}` → 200, or 404 if missing
    - `PATCH /organizations/{id}` → 200 + updated entity
    - `DELETE /organizations/{id}` → 204 No Content, empty body
    - `POST /organizations/{id}/archive` custom action → 200 (or 201 if it creates something, e.g. `POST /projects/{id}/clone` returns 201 + new project).
  - Collection and member routes share paths and are distinguished by method.
  - List endpoints support pagination, sorting and filtering via query params. Paginated response envelope: `data`, `total`, `page`, `totalPages`. Params: `page`, `limit`, `sortBy`, `sortOrder`, plus field filters like `status=archived`, `name=...`. Example: 5 rows, limit 2 → 3 pages (2,2,1).
  - Why paginate: serialising thousands of rows is CPU-heavy, slows the response, and the UI shows only 10–20 at a time anyway. The client stops infinite scroll when page == totalPages.
- **Decisions & trade-offs**
  - Custom action vs PATCH: "archive" looks like setting status=archived, but it may cascade (remove projects, notify members, delete tasks). If an operation has server-side consequences beyond the field, model it as a POST action, not a field update. Same for clone: client could fake it with create, but the server may need to copy child tasks or send emails.
  - Do not assume POST means 201: status follows what actually happened.
- **Rules of thumb / checklist**
  - Plural, lowercase, hyphenated resource paths; hierarchy via `/`.
  - JSON fields in camelCase; identical field names for identical concepts across all resources (`description` everywhere, never `desc` in one place). Consistency beats any particular style: once integrators learn one endpoint they will assume the rest work the same.
  - Never accept server-managed fields (id, createdAt, updatedAt) from the client.
  - Sane defaults: page=1, limit 10–20, sortBy=createdAt, sortOrder=desc; on create, default fields like status=active and require only what is truly needed. Always sort explicitly, since DBs return rows in no guaranteed order and unsorted pages shuffle between calls.
  - A list with no matches returns 200 with `data: []`, `total: 0`, never 404. 404 is only for a specific requested resource that does not exist.
  - No abbreviations in fields or params; integrators lack your context.
  - Ship interactive docs (OpenAPI/Swagger) from day one and keep them updated; this doubles as a test playground.
  - Hold a separate, language-agnostic design session before coding.
- **Failure modes & gotchas**
  - Inconsistent naming or singular/plural mix forces integrators back to docs and wastes their time.
  - Returning 404 on empty lists breaks clients that treat errors as failures.
  - Missing default sort makes pagination non-deterministic (duplicates or skipped rows across pages).
  - (note: offset/page pagination gets slow on very large tables and can skip or duplicate rows if data changes between page requests; cursor/keyset pagination is the usual fix at scale. Not covered in the video.)
- **Relevance to Sam's work**
  - POST is not idempotent: for booking or payment creation, a retried request creates a duplicate booking. Add an idempotency key (client-supplied, stored with the result) so retries return the original outcome. The video establishes why this is needed; it does not cover the key pattern itself.
  - Model supplier-side actions (cancel booking, confirm, refund) as `POST /bookings/{id}/cancel` style actions, because they cascade into supplier calls, emails and payments.
  - For internal data APIs over millions of location rows, always paginate with a deterministic sort (ideally keyset on an indexed id) and never return a whole table in one response.

## Sriniously — 23. Object Storage: Everything You Need to Know, Part 1 (https://www.youtube.com/watch?v=Ie0TjKI9cDI)
**Files are not your server's data: store them in object storage, let clients upload directly with pre-signed (policy) URLs, and keep your database as the index of what exists.**

- **Mental models / first principles**
  - Naive approach (save upload to local disk, store path in DB) fails for six reasons:
    1. Container disks are ephemeral; every restart or deploy wipes them. A persistent volume fixes only this one.
    2. Horizontal scaling: with N instances behind a round-robin load balancer, a file lives on one box, so roughly (N−1)/N of reads 404. Local files break statelessness.
    3. Disk is fixed-size and only grows vertically (resize, often downtime, provider ceiling), while user files are the fastest-growing data (1,000 users × 100 MB = 100 GB/month and it never shrinks).
    4. Durability: one disk, one copy. Availability = can I reach it now (fixable by waiting); durability = does it still exist anywhere (not fixable by waiting). The only cure for durability is replication plus checksums plus repair processes, which is a whole engineering field.
    5. Downloads tie up workers: 50 MB to a 2 Mbps phone is ~3.5 minutes of a goroutine/event-loop slot doing slow I/O. Your server becomes an expensive, low-concurrency CDN.
    6. No transaction spans file system and DB. File first then DB fails → orphan file (wasted money). DB first then file fails → row pointing at nothing (500/404). Object storage makes this easier but does not remove it.
  - Three storage kinds:
    - **Block** (SSD, AWS EBS): numbered 512 B/4 KB blocks, microsecond latency, what Postgres/MySQL sit on; too primitive for apps and attachable to one machine at a time.
    - **File** (ext4, NTFS, NFS): hierarchy, permissions, POSIX semantics (seek, overwrite bytes in place, atomic rename, append, locks). Those guarantees need coordination that does not scale across networks or continents.
    - **Object** (S3, R2, B2): asks for the *minimum* interface so it can scale without limit: PUT, GET, DELETE by key, LIST keys. No in-place edit, append, seek-write, rename (= copy + delete, paying full size), locks, or real folders; latency is milliseconds to tens of ms because every op is an HTTP request.
  - Payoff: with no mutation there is nothing to coordinate; no tree to lock or rebalance; every request is self-contained so any of thousands of front-end nodes can serve it; capacity effectively infinite; S3 durability "11 nines"; any HTTP client (browser, video player, app) can fetch directly. General distributed-systems lesson: to reach unlimited scale you give up mutability (same as caches, stateless servers, immutable deploys).
- **Mechanisms**
  - Object = key (its full identity) + value (bytes) + system metadata (size, last-modified, content-type, ETag) + user metadata (your own key/values, e.g. uploaded-by user id). Objects live in buckets; bucket names are globally unique per provider.
  - Folders are an illusion. `uploads/2026/cat.png` is one key; consoles fake folders by listing with prefix + delimiter and returning `CommonPrefixes` (a group-by on strings). Hence "moving a folder" of 40,000 objects is 40,000 copies + deletes, and empty folders do not exist (tools fake them with a zero-byte key ending in `/`).
  - PUT path in S3: a stateless front-end node authenticates (recomputes the signature), then the work splits into a **data plane** (store bytes durably; huge volume, simple ops) and a **metadata plane** (strongly consistent, globally distributed sorted index, conceptually a distributed B-tree, mapping bucket+key to shard locations, size, ETag, content-type; small data, enormous op count; the hard problem).
  - Durability: 3× replication costs 200% overhead. Erasure coding (Reed–Solomon) splits an object into e.g. 14 data shards + 6 parity shards = 20, spread across drives, racks, power domains and buildings; any 14 of the 20 rebuild the object. That is ~1.43× overhead, tolerates 6 simultaneous losses instead of 2, at the price of CPU and extra reads on recovery. 11 nines ≈ store 10 million objects, expect to lose one every 10,000 years.
  - Index is partitioned by key range, so LIST is a range scan over a distributed index (slow, paginated at 1,000), size lookups are instant (metadata only), and rename is hard (key decides partition).
  - Consistency: S3 was eventually consistent (read-after-write could 404) until December 2020; now strong read-after-write. Old advice to add sleeps/retries after upload is obsolete.
  - Conditional writes (S3, 2024): `If-None-Match: *` → create only if key absent, else 412 Precondition Failed. `If-Match: <etag>` → replace only if the object is unchanged since you read it (compare-and-swap; the slower writer gets 412 instead of silently clobbering). This gives object storage a coordination primitive: exactly-one-writer, leader election, write-ahead logs on S3.
  - Pre-signed URL: normal URL plus query params (algorithm, access key id, timestamp, expiry seconds, signed headers, HMAC signature over method + bucket + key + expiry etc. using your secret). S3 recomputes the HMAC; any change to key/method, or use after expiry, gives 403. Credentials never leave the server; the browser gets one operation on one key for a few minutes.
  - Pre-signed **POST with policy**: sign a JSON policy and return URL + form fields. Conditions enforced by the storage service: `content-length-range` (e.g. 1 KB–5 MB), content-type starts with `image/`, key starts with `tenant/user/` prefix. Violations rejected (EntityTooLarge / 403) and nothing is stored. Client: append the file field last in FormData, and do not set Content-Type on fetch (let the browser set the multipart boundary).
- **Decisions & trade-offs**
  - Proxy-through-server upload: you see every byte (validate, scan, hash, reject) and auth is ordinary middleware with no CORS. Fine for small files (avatars, small CSVs). Costs: memory if buffered, bandwidth, load balancer idle timeout (~60 s default), reverse proxy body limits (Nginx 413), and hard serverless/API-gateway payload caps of a few MB.
  - Direct-to-bucket with pre-signed POST + policy is the production pattern.
  - S3-compatible ≠ feature-complete: R2 works with AWS SDK; Backblaze B2 is S3-compatible but (per the video) lacks conditional writes. Check docs before relying on them.
- **Rules of thumb / checklist**
  - Never use the user's filename as the key: collisions (PUT silently overwrites, no error), path traversal via dots/special chars, and hostile characters (emoji, RTL override, null bytes). Generate the key yourself; keep the original name in the DB (and optionally user metadata) for downloads.
  - Put high-entropy parts early in the key (`tenantId/userId/uuid`, not `date/user/file`) to avoid hot index partitions that throw 503 SlowDown. Costs nothing on day one.
  - Encode ownership in the key prefix so authorization is a string-prefix check and IAM credentials can be scoped to a prefix, skipping a DB round-trip.
  - Never call LIST in the request path; your database is the index of files, the bucket stores bytes.
  - If proxying, stream: read body in chunks and pipe to the SDK uploader (Go: multipart reader part → uploader) so memory is one small buffer regardless of file size. Always wrap the body in a size limit before reading. Demo: 500 MB upload buffered grew RSS by ~500 MB; streamed stayed flat, same 6.5 s.
  - Two-phase upload: (1) client requests upload; server authorizes, generates key, inserts DB row `status=pending` with expected constraints, returns pre-signed POST. (2) Client uploads, then calls a completion endpoint with the upload id (not the key); server does HEAD object (cheap metadata call) to confirm existence, size and content-type, then marks `ready`.
  - Cleanup: bucket lifecycle rule deleting objects under a pending prefix after 24 h, plus a background job marking stale pending rows as expired, so DB and bucket do not drift.
  - CORS on the bucket: allow your frontend origin, PUT/POST, and expose the `ETag` header (needed for multipart). If uploads work in curl/Postman but fail in the browser, suspect bucket CORS.
  - Durability is not backup: S3 will durably execute your mistaken prefix delete or buggy overwrite. Enable versioning and replicate to a second bucket in a separate account (so one compromised credential cannot destroy both).
- **Failure modes & gotchas**
  - Frameworks that parse multipart into memory (Go `ParseMultipartForm`, Node default body parsers): 20 concurrent 100 MB uploads ≈ 2 GB RAM → OOM kill. Invisible in local dev (small files, fast network, 16 GB+ machine).
  - Plain pre-signed PUT URL does not limit size: the demo stored a 300 MB file through a URL meant for an avatar. Use a policy.
  - Upload succeeds but completion call never arrives (tab closed, mobile drop) → orphaned objects and stuck pending rows unless you have lifecycle + sweeper.
- **Relevance to Sam's work**
  - For large data on a 16 GB laptop: object storage is the right home for multi-GB pipeline inputs/outputs (GeoNames dumps, GTFS zips, parquet chunks). Stream to and from it in chunks; never `read()` the whole object.
  - Key design for datasets: `dataset/version-or-commit/part-uuid.parquet`, and keep a manifest table in the DB rather than listing the bucket.
  - `If-None-Match: *` gives a cheap idempotency / exactly-once guard for pipeline outputs and booking documents (vouchers, tickets): a retried job cannot overwrite a finished artifact. `If-Match` ETag gives optimistic concurrency for shared state files.
  - Supplier files (images, PDFs) fetched from third parties should be stored under generated keys, never supplier-provided filenames.

## Sriniously — 16. Error Handling and Building Fault Tolerant Systems (https://www.youtube.com/watch?v=8NaM_9aKS24)
**Errors are normal: classify them, detect them before users do, recover or degrade deliberately, and funnel everything through one global error handler that never leaks internals.**

- **Mental models / first principles**
  - The question is not whether errors happen but how you respond. Fault tolerance is a mindset: assume DB queries fail, external APIs time out, users send garbage, and business logic hits unconsidered edge cases.
  - The best error handling starts before the error: catch problems before they spread or reach users.
  - Data is the only tangible asset; code and services can be redeployed, corrupted data cannot. Data integrity is priority one.
- **Error taxonomy (with causes)**
  - **Logic errors** (most dangerous): code runs, results are wrong, nothing crashes. Example: discount applied twice → negative shipping, losing money on every order for weeks. Causes: misunderstood requirements, wrong algorithm, unconsidered edge cases. Especially costly in payments.
  - **Database errors**: connection failures (network down, DB overloaded, pool exhausted; pools exist to skip TCP handshake per request); constraint violations (unique email, foreign key to a missing customer); query errors (typos, missing tables, timeouts on overly complex queries); deadlocks (operations waiting on each other in a cycle).
  - **External service errors**: payments, email, object storage, Redis, auth providers (Clerk, Auth0). Network issues (timeouts, DNS failures, partitions); auth errors (bad/expired credentials, insufficient permissions); rate limiting (HTTP 429); full outages or maintenance. You cannot drop dependencies, so plan for each failing.
  - **Input validation errors**: format, range (numbers, string length, array size), required fields → 400. Easiest class because you define the rules; validation is the first line of defense.
  - **Configuration errors**: e.g. a new API key added to local .env but not to the production parameter store. If you validate config at startup, the new deploy fails to boot and (with blue-green style rollout) the old version keeps serving. If you do not, it boots and users hit 500s at runtime when the code path runs. Crash at startup is strictly better.
- **Mechanisms**
  - Proactive detection:
    - Liveness-style `/health` or `/status` returning 200 (status code is what matters).
    - Deeper checks: DB connectivity plus a representative query timed against its usual baseline (500 ms creeping to 4–5 s is a warning).
    - External dependencies: periodic test transactions with the payment processor, test emails to internal inboxes, test tokens validated against the auth provider.
    - Core functionality: config loaded, required caches warmed, internal structures consistent.
  - Monitoring: track performance (latency, resources, throughput), not only error rates; degradation usually precedes failure. Track business metrics too: a drop in successful transactions can reveal a technical fault while error rates look normal. Structured JSON logs into aggregators (Grafana/Loki).
  - Immediate response by class:
    - Recoverable (email send failed, pool exhausted, transient network) → retry with exponential backoff (e.g. wait 1–2 min after a 429, then double each attempt). Retries must not pile more load on an already stressed system.
    - Non-recoverable → contain and degrade gracefully: serve cached data, disable non-essential features, use fallbacks (e.g. Redis down → in-memory cache or a second node) to protect core flows such as payments and order processing.
  - Recovery: automatic (restart hung processes, clear corrupted caches, fail over to backups), designed carefully since naive automation can worsen incidents; manual runbooks documented and rehearsed for cases needing human judgment. Data recovery: backups at key moments, restores, transaction log replay.
  - Propagation control: catch low-level errors, wrap them with business context, bubble them up deliberately. Error boundaries between services: separate processes, timeouts at service boundaries, message queues (e.g. RabbitMQ) for async decoupling so one service's bug does not take another down.
  - **Global error handler middleware** (route → handler → service → repository layering). Repositories do one DB operation; services orchestrate repositories; handlers validate and bind. Every layer returns or throws typed/custom errors upward; one middleware maps them to responses:
    - validation error → 400 with field-level errors
    - unique-constraint violation → 400 (or 409) "book already exists"
    - "no rows" from a single-resource select → 404 "book 123 not found"
    - foreign-key violation (e.g. unknown authorId) → 404 for the referenced resource
    - anything unrecognized → 500 "internal server error" with a generic message.
  - Benefits: no forgotten conditions (otherwise a missed unique-violation check becomes a confusing 500), and no duplicated error-mapping code in every repository method.
- **Decisions & trade-offs**
  - Retry vs degrade: retry when the operation can tolerate delay and the failure is transient; degrade when it cannot recover soon.
  - Handle at the point of failure vs propagate: propagate to where enough context exists to decide, but keep the bubbling path under control.
  - Validation cannot prevent all DB errors (only the DB knows uniqueness), so you still need clean mapping of constraint errors.
- **Rules of thumb / checklist**
  - Validate all required config at boot and refuse to start if any is missing or malformed.
  - Health checks should prove the service does its job (DB query, dependency checks), not merely that the process is up.
  - Back off exponentially on 429s and transient errors.
  - One central error-to-HTTP mapping; default branch returns a generic 500 message.
  - Never echo raw DB errors (table, index, constraint names) to clients; they help attackers craft SQL injection.
  - Login errors: always "invalid email or password", never "no user with this email" vs "wrong password", which enables account enumeration then password spraying. Consult the OWASP cheat sheets.
  - Logs: never log passwords, emails, API keys, card numbers. Log user IDs and correlation IDs. Logs are shipped to third-party services and are a common breach source.
- **Failure modes & gotchas**
  - Logic bugs that silently lose money for months when business metrics are not monitored.
  - Retry storms amplifying an overload.
  - Missing env var discovered only when a rarely used endpoint is hit in production.
  - Unhandled errors defaulting to 500 when the real cause was a user mistake (bad UX, wasted support time).
- **Relevance to Sam's work**
  - Supplier integrations (Viator, RateHawk, etc.): treat each supplier as a failure domain with timeouts, 429-aware exponential backoff (ideally honoring Retry-After), and a fallback (cached availability, hide that supplier) instead of failing the whole search.
  - Booking/payment flows: the double-discount example generalizes; monitor business metrics like bookings confirmed vs attempted, price mismatches, refunds.
  - Synthetic checks against supplier sandboxes (test search, test hold) every few minutes catch credential expiry or API changes before customers do.
  - Long pipelines: validate config and input schema at start; classify row-level errors (skip and log to a reject file) vs fatal errors (stop) so a 6M-row run does not die at row 5.9M on one bad record.

## Sriniously — 29. Webhooks: how the server calls you (https://www.youtube.com/watch?v=eWM0CVReP04)
**A webhook is a provider's plain HTTP POST to your URL: verify raw bytes with HMAC, dedupe by delivery ID in the same transaction as the work, ack fast and process async, never trust order; as a sender, use an outbox.**

- **Mental models / first principles**
  - Problem: something happens in another system (Stripe payment succeeds) and your backend must learn it. The browser cannot be the receiver: tabs close, networks drop, and client-reported "payment succeeded" can be forged. Your server must be the source of truth.
  - Polling is the brute-force fix: poll every 1 s → 0.5 s average delay, mostly wasted requests, and 10,000 merchants polling = 10,000 req/s on the provider, so providers rate-limit hard; at a 60 s interval you average 30 s delay. Webhooks flip the direction: the provider calls you when the event occurs.
  - History: Jeff Lindsay (2007) proposed webhooks as a Unix-pipe-like primitive for the web (user-defined callbacks via HTTP POST); Google engineers built PubSubHubbub on the idea about two years later.
  - Vocabulary: provider/sender, consumer/receiver, endpoint (your URL), event (e.g. payment.succeeded, task.moved), delivery (one attempt; one event can have many deliveries), subscription (URL + event types).
  - Delivery is **at-least-once**, never exactly-once: your 200 can be lost after you did the work, so the provider retries.
- **Mechanisms (receiving)**
  - Request shape (GitHub example): POST, `Content-Type: application/json`, event-type header (`X-GitHub-Event`), unique delivery ID header (`X-GitHub-Delivery`), signature header (`X-Hub-Signature-256`), JSON body. GitHub also keeps a "recent deliveries" log with the response you returned.
  - Success = any 2xx within the provider's deadline: GitHub 10 s, Shopify 5 s, Slack 3 s, Microsoft Graph similar. Anything else (including 3xx) or a timeout counts as failure.
  - Handshakes on registration prevent attackers pointing a provider at a victim as a DDoS: Microsoft Graph wants a validation token echoed as plain text, Discord sends ping expecting pong, AWS SNS sends a subscription-confirmation URL to visit.
  - Local dev: inbound requests cannot reach a laptop behind a home router, and "localhost" means nothing to the provider; use a tunnel (ngrok, Cloudflare Tunnel) for a temporary public HTTPS URL.
  - Authenticity options, weakest to strongest/commonest: secret token in the URL (leaks into logs everywhere); IP allowlist from the provider's published ranges (good, incomplete alone); mutual TLS (provider presents a client cert); **HMAC-SHA256 over the body with a shared secret** (~65–80% of providers); asymmetric signature verified with the provider's public key (Discord). HTTPS only protects bytes in transit, it does not prove who sent them.
  - What gets signed varies: GitHub signs body only; Stripe signs `timestamp.body` (header `t=...,v1=...`, possibly multiple v1 values, compared with `hmac.Equal`); Standard Webhooks signs `id.timestamp.body`; Twilio signs the full URL plus sorted form fields with SHA-1. Read each provider's docs.
  - Replay protection: include a timestamp in the signed string and reject old deliveries (Stripe's library rejects anything older than 5 minutes).
  - Constant-time compare (Go `hmac.Equal`, Node `crypto.timingSafeEqual`, Python `hmac.compare_digest`); ordinary `==` exits at the first mismatched byte and leaks timing that can be used to derive a signature byte by byte.
  - **Idempotency via delivery ID** (GitHub header, Stripe `event.id` in body, Standard Webhooks `webhook-id`; stable across retries): processed-events table with a unique constraint on delivery ID, and insert it **in the same DB transaction as the side effects**. Work-then-record risks double processing after a crash; record-then-work risks skipping work that failed.
  - Ordering is not guaranteed (Stripe says so explicitly; timestamps have 1-second resolution so near-simultaneous events tie). Example: `created` delivery fails and retries 5 min later, `deleted` succeeds immediately → you recreate a deleted object. Fixes: (1) treat the event as a signal only and fetch current state from the provider API (costs one call); (2) store version/updated_at and apply only newer events, in one transaction; (3) buffer per object until earlier events arrive (unpredictable). Some providers sell strict ordering.
  - Retry schedules differ: Stripe exponential backoff for up to 3 days then disables the endpoint and emails you; Svix immediately, 5 s, 5 min, 30 min, 2 h, 5 h, 10 h...; GitHub does **not** retry at all (redeliver manually or script it via their API).
  - Thin receiver: read raw body → verify signature → persist delivery (dedupe) → return 2xx immediately → background worker/queue does the real work. Doing the work inline misses deadlines under bursts (billing renewals on the 1st of the month, receipts for a 500k-recipient email blast), and each miss becomes a retry adding more load.
- **Mechanisms (sending, when you are the provider)**
  - Naive `http.Post` inside the request handler fails three ways: customer endpoint latency becomes your latency; crash after DB commit loses the event; post-then-commit announces something that then fails to persist.
  - **Transactional outbox**: in one DB transaction update the business row and insert an outbox row with the full event; a separate dispatcher reads the outbox and delivers. Same idea as a write-ahead log: durably record intent first, act later.
  - Demo schema: `endpoints` (URL, secret), `outbox`, `deliveries` (one row per attempt with status code, first 2 KB of response, timing).
  - Dispatcher per delivery: sign id + timestamp + body (Standard Webhooks format) with the endpoint secret; POST with 10 s timeout; record attempt; 2xx → delivered, else schedule next retry with exponential backoff + jitter (random offset so thousands of retries do not land on the same instant), honoring the receiver's `Retry-After`; after the last attempt mark exhausted; after 10 consecutive failures disable the endpoint and email the owner. Use one queue per endpoint so one dead customer URL does not stall others.
  - SSRF defense: attackers register URLs like the cloud metadata IP or internal admin services, which your dispatcher would call from inside your network. Resolve DNS yourself and reject private/loopback ranges **at every send**, not only at registration (DNS can change later); never follow redirects (treat 3xx as failure); ideally send through an egress proxy on a separate network (Stripe's open-source Smokescreen).
- **Decisions & trade-offs**
  - One endpoint per provider (`/hooks/github`, `/hooks/stripe`), never a generic `/hooks/callback`: headers, signature schemes and retry rules differ.
  - Use webhooks as **notifications** (post to chat, start CI, unlock a plan). Using them as a **data sync** mechanism (e.g. mirroring Clerk users/orgs/roles to cut 500–600 ms API latency down to 50–60 ms local reads) forces you to handle timeouts, replays and ordering for a copy to stay correct.
  - Better sync: providers keep an ordered event log (`/events` API). Either poll it every N minutes from a stored last-event-ID cursor (fine if minutes of staleness are OK), or subscribe to a webhook that only says "new events exist" and then pull from the cursor. Near-real-time, ordered, and replay-safe.
  - Webhooks-as-a-service (Svix) if you need to send webhooks and do not want to build all of the above.
- **Rules of thumb / checklist**
  - Read body as raw bytes, verify, then parse. Parsing and re-serialising (e.g. Express `express.json` then `JSON.stringify`) changes whitespace and breaks the HMAC. Demo: correct secret → 202; changed secret → 401; correct secret but parse-then-verify → fails.
  - Constant-time signature comparison; reject stale timestamps.
  - Unique delivery ID + same-transaction side effects.
  - Never trust event order or event payload as current state for anything that matters.
  - Ack within the deadline; enqueue work.
  - Know each provider's retry schedule; for GitHub-style no-retry providers, build reconciliation.
- **Failure modes & gotchas**
  - Guessable `/webhooks` URL + no signature check = anyone can POST "payment succeeded for user X".
  - Lost acks cause duplicates; slow handlers cause retry storms.
  - Out-of-order create/delete resurrects deleted objects.
  - SSRF through customer-supplied URLs, including DNS rebinding after registration and redirects.
- **Relevance to Sam's work**
  - Payment and supplier booking callbacks (payment gateway, supplier booking-status notifications): this is the exact pattern for "booking confirmed / cancelled / refunded". Dedupe on event ID, re-fetch booking status from the supplier API before acting (ordering-safe), and never mark a booking paid based on a browser redirect alone.
  - If Trameter ever notifies partners or creators (e.g. Drive Button bookings attributed to a blog), use an outbox table plus dispatcher with backoff + jitter and per-partner queues.
  - For mirroring supplier catalogues, prefer cursor-based pulls of change feeds over relying on push events for correctness.

## Sriniously — How Vercel works Part 3: Serverless databases to the rescue (https://www.youtube.com/watch?v=nvQd1B4mijo)
**Relational DB connections are expensive stateful TCP sessions; pools protect the DB, and serverless/edge compute needs serverless-style databases or proxies that pool globally and speak HTTP/WebSocket.**

- **Mental models / first principles**
  - A traditional DB (Postgres/MySQL on a VM, port 5432) needs a persistent TCP connection, unlike stateless HTTP. Postgres's postmaster forks one OS process per connection; MySQL uses one process with a thread per connection.
  - Opening a connection costs a TCP 3-way handshake, a TLS handshake, user authentication, and buffer allocation. Each connection also carries session state: session variables, temp tables, prepared statements.
  - Each Postgres connection costs roughly 1 MB+ of RAM (note: this memory is consumed on the database server, which the teacher blurs with the API server; real per-backend usage is often several MB). Example: a 512 MB box with 500 connections is in trouble (note: the teacher calls this a t2.micro; t2.micro is actually 1 GiB, t2.nano is 512 MB, but the point stands). Every DB also has a `max_connections` ceiling.
  - Serverless compute (Lambda, Cloudflare Workers) scales API instances to thousands in seconds; each instance opening its own DB connections exhausts `max_connections` (errors). With a fixed pool, requests queue instead (latency). Either way, bad UX. That mismatch is why serverless databases exist.
- **Mechanisms**
  - **Connection pooling**: pre-open N connections; a request borrows an idle one, runs its query, returns it. Active + idle never exceed the DB's max; excess requests wait in a queue rather than crashing the DB. Example: pool of 500 with 1,000 concurrent requests → 500 run, 500 wait.
  - Where pools live:
    1. Client-side, inside your app's driver (node-postgres/postgres.js, Go `database/sql`, pgx).
    2. Standalone proxy in front of the DB (PgBouncer for Postgres, fast and widely used).
    3. Managed-service drivers/proxies (Neon, PlanetScale) that pool and multiplex for serverless clients.
  - Why browsers cannot query the DB: browsers (and V8-isolate edge runtimes) cannot open raw TCP sockets, only HTTP/WebSocket; and any credentials in browser JS can be extracted, giving every user full DB access. Never do it.
  - Serverless DBs replace per-client TCP with HTTP/WebSocket drivers talking to a global proxy/pool. PlanetScale (MySQL + Vitess) uses a global proxy layer and published a test at 1 million concurrent connections. Neon's serverless driver multiplexes many queries over fewer TCP connections.
  - **Separation of compute and storage** (Neon, open source, Rust): data lives in object storage (S3/R2-like, cheap and durable); lightweight Postgres compute instances spin up and read from it, so compute scales independently and can scale to zero.
  - Edge runtimes (Cloudflare Workers, CloudFront Functions, Next.js edge routes) are server-side (credentials safe) but lack TCP, so they need edge-compatible drivers that use only fetch/WebSocket. Cloudflare D1 = SQLite-based edge database. Cloudflare Hyperdrive = global proxy/pool that holds TCP connections to your existing traditional Postgres and exposes it to Workers, so you can move compute to the edge without migrating the DB.
  - Next.js on Vercel: Node runtime runs on AWS Lambda; edge runtime runs on Cloudflare Workers. Neither suits a plain VM database without pooling/proxying.
- **Decisions & trade-offs**
  - Managed vs serverless DB: managed = someone else runs the server (RDS, Neon, PlanetScale, Supabase); serverless = scales to zero and to demand. Often used interchangeably but not the same thing (note: the teacher at one point says managed DBs "internally use serverless databases", which is not generally true; RDS is managed but not serverless).
  - Serverless selling points: scale to zero (pay nothing when idle) and scale to demand (viral spike, no DevOps team needed), with cold-start latency as the cost (kept small by Firecracker microVMs and V8 isolates).
  - Long-running server + traditional Postgres + in-driver pool is fine; serverless/edge compute + traditional Postgres needs PgBouncer, Hyperdrive or a serverless DB.
- **Rules of thumb / checklist**
  - Always use a pool; size it below the DB's `max_connections`, accounting for every app instance (instances × pool size ≤ max).
  - Borrow a connection per unit of work and return it promptly; long holds starve the pool.
  - If compute is serverless, put a pooler/proxy between it and the DB.
  - Never ship DB credentials to the client.
- **Failure modes & gotchas**
  - Connection storms during traffic spikes from autoscaled serverless functions → "too many connections" errors.
  - Pool exhaustion shows up as latency spikes rather than errors; monitor pool wait time.
  - TCP-based drivers throw at runtime in edge environments.
  - (note: transaction-mode poolers like PgBouncer break session-level features, such as session variables, temp tables and some prepared-statement usage, because consecutive queries may hit different server connections. That follows from the per-connection state described above, but the video does not say it.)
- **Relevance to Sam's work**
  - Parallel pipelines on a 16 GB laptop: if you spawn many worker processes (multiprocessing, joblib) each with its own connection pool, you can blow past Postgres `max_connections` and also burn RAM on the DB side. Use a small pool per worker (1–2) or a shared PgBouncer, and batch writes (COPY / executemany) instead of many connections.
  - For bulk loads, one connection doing `COPY` beats hundreds of concurrent inserts.
  - If Trameter services move to serverless (Vercel/Lambda) for supplier search APIs, use a pooled/serverless Postgres (Neon, Supabase pooler, RDS Proxy, Hyperdrive) from the start.

## Sriniously — 9. Validations and transformations for backend engineers (https://www.youtube.com/watch?v=qedj_JjjL-U)
**Validate and transform every piece of client input (body, query, path, headers) at the controller entry point, before any business logic, so bad data fails fast as a 400 instead of exploding deep inside as a 500.**

- **Mental models / first principles**
  - Layering: controller (HTTP concerns: parse input, choose status codes, shape output) → service (business logic, orchestration, emails, notifications, webhooks) → repository (DB queries). Validation/transformation sits right after route matching, before the controller calls any service.
  - Without it, bad input travels down the stack: e.g. `{"name": 0}` reaches an `INSERT` into a `text NOT NULL` column, Postgres rejects the type, and the client sees a vague 500 for what was their mistake. Validating up front converts it into a precise 400 Bad Request.
  - Client-side validation is for **UX** (instant feedback). Server-side validation is for **security and data integrity**. Clients vary (your web app, Postman/Insomnia, scripts, other services) and some have no validation at all, so the server must be strict regardless of what the frontend does.
- **Mechanisms**
  - A reusable validation + transformation pipeline (middleware/utility) that takes a schema and checks: field presence → type → constraints, returning all field-level errors at once (demo returns an array of errors for missing email, phone, date).
  - Validation types:
    - **Type**: string/number/boolean/array/object, including element types inside arrays (demo: array of numbers rejected when strings expected) and nested objects.
    - **Syntactic**: structure matches a format: email, phone (country code + digits), date (e.g. YYYY-MM-DD).
    - **Semantic**: value makes sense: date of birth not in the future; age within 1–120 (430 rejected).
    - **Complex / cross-field**: password confirmation must equal password (and password ≥ 8 chars); `partner` required only when `married` is true.
  - **Transformation**: changing data into the shape the service needs, before or after validation. Key example: query params always arrive as strings (`page=2&limit=20` → `"2"`, `"20"`), so cast to numbers before range checks (page > 0 and < 500; limit > 0 and ≤ some max). Normalisation examples from the demo: lowercase emails, prefix phone numbers with `+`, reformat dates.
  - Keep validation and transformation together in one pipeline so all input rules for an endpoint live in one place.
  - Bonus: descriptive validation errors act as fallback documentation for integrators who call an endpoint with an empty payload.
- **Decisions & trade-offs**
  - How strict to be is a design choice per API, but default to as strict and specific as the business logic requires.
  - Transform-then-validate (casting query strings) vs validate-then-transform (normalising casing for storage) depends on the field; both belong in the same pipeline.
- **Rules of thumb / checklist**
  - Validate body, query, path params and headers for every endpoint.
  - Check required → type → format → semantics → cross-field rules.
  - Cast query/path params explicitly; bound pagination params (max limit).
  - Normalise identifiers (lowercase email, E.164-style phone, canonical dates) before they hit the DB, so uniqueness checks and lookups behave.
  - Return 400 with per-field messages; never let type errors reach the DB.
  - Never rely on frontend validation for correctness.
- **Failure modes & gotchas**
  - Treating `"2"` from a query string as a number without casting → spurious validation failures or string comparison bugs.
  - Mixed-case emails stored as-is → duplicate accounts that bypass unique constraints.
  - Missing validation surfaces as 500s and DB errors, hurting UX and leaking internals.
- **Relevance to Sam's work**
  - Treat supplier API responses as untrusted input too: validate them with schemas (e.g. Pydantic/Zod) at the integration boundary, because suppliers change fields, send nulls, or switch number-as-string. Fail or quarantine at the edge rather than corrupting the master dataset.
  - In data pipelines, the same split applies: type/syntactic checks (lat/lon numeric, ISO country codes), semantic checks (lat within −90..90, lon −180..180, dates not in the future), cross-field checks (end date after start date; GTFS stop times monotonic), plus normalisation (Unicode NFC, trimmed whitespace, lowercased slugs) before matching/deduplication.
  - Booking requests: semantic validation of travel dates (check-out after check-in, not in the past), passenger counts, and currency codes before calling paid supplier APIs saves money and avoids supplier-side errors.

## Sriniously — 19. Graceful Shutdown (https://www.youtube.com/watch?v=6rfBgphiCWM)
**On SIGTERM/SIGINT, stop taking new work, let in-flight work finish within a bounded timeout, release resources in reverse order of acquisition, then exit; otherwise SIGKILL ends you mid-transaction.**

- **Mental models / first principles**
  - Even with zero-downtime deploys, the old instance must stop at some point. If it dies mid-request (say mid-payment), you risk lost transactions, double charges, refunds and corrupted data.
  - Your backend is an OS process with a lifecycle: start → run → terminate. The OS and process managers talk to it through Unix **signals** (a form of inter-process communication). The app registers handlers for the signals it wants to catch.
  - Analogy: a restaurant closing: stop seating new guests, tell seated guests they have ~15–20 minutes, then clean up and lock the door.
- **Mechanisms**
  - **SIGTERM**: polite request to terminate, sent by process managers/orchestrators (Kubernetes, systemd, PM2). Catchable; you get a window to finish.
  - **SIGINT**: interrupt, typically Ctrl+C in a terminal (user-initiated). Handle exactly like SIGTERM; what matters is the intent to stop, not who sent it.
  - **SIGKILL**: cannot be caught or ignored; immediate death, no cleanup. It is what you get if you ignore the polite signals or overrun the grace period.
  - Graceful shutdown = (1) stop accepting new connections/requests, (2) finish in-flight work (**connection draining**), (3) clean up resources, (4) exit.
  - Draining differs by server type: HTTP → stop accepting, let in-flight requests complete; database → finish running queries/transactions, refuse new ones, then close; WebSocket → notify clients you are closing, then close sockets (never just drop them).
  - Drain timeout: a hard cap, commonly 30 s (sometimes 60 s). Too short interrupts legitimate work; too long makes deploys sluggish. Base it on your real request durations; long-lived connections (WebSockets) need more thought.
  - Draining must coordinate with load balancers, health checks and service discovery (deregister/fail readiness so no new traffic is routed to you).
  - Resource cleanup: file handles, network sockets, DB connections, temp files, caches. The OS caps per-process file descriptors and connections; leaks exhaust them and memory. Open DB transactions must be explicitly committed or rolled back, or you risk inconsistent state, deadlocks and corruption.
  - Release in **reverse order of acquisition** (acquired Redis then DB → close DB then Redis) so nothing is torn down while something that depends on it is still running.
  - Go demo: register a signal-aware context for interrupts; on signal call the shutdown function: `server.Shutdown` (framework stops accepting and waits for active requests), close the DB pool (stops new queries, finishes existing, closes TCP connections), then stop the Redis-backed job server (Asynq: "waiting for all workers to finish... exiting"), then log a clean exit. Locally with no in-flight requests it still took ~1 s.
- **Decisions & trade-offs**
  - Timeout length: speed of deployments vs risk of cutting off long operations.
  - Most frameworks ship a shutdown method; use it rather than hand-rolling, but understand what it does.
- **Rules of thumb / checklist**
  - Catch SIGTERM and SIGINT; route both to one shutdown function.
  - First fail readiness / deregister from the load balancer, then stop accepting, then drain with a timeout, then close resources in reverse order, then exit with a clear log line.
  - Commit or roll back open transactions; never leave them dangling.
  - Background workers: stop pulling new jobs and let current jobs finish (or hand them back to the queue) within the grace period.
  - Keep the app's drain timeout shorter than the orchestrator's grace period so you finish before SIGKILL arrives (note: Kubernetes' default `terminationGracePeriodSeconds` is 30 s; this is standard practice rather than something the video states).
- **Failure modes & gotchas**
  - Ignoring SIGTERM → SIGKILL after the grace period → requests cut mid-flight, half-written data.
  - Wrong teardown order: closing the DB pool while job workers are still running makes those jobs fail. (note: in the demo the DB is closed before the background-job server is stopped; if those jobs use the DB, stop workers first, per the teacher's own reverse-order rule.)
  - Leaked file descriptors/connections over a long-running process lifetime.
- **Relevance to Sam's work**
  - Long pipeline runs on the laptop: handle Ctrl+C (SIGINT) so a chunked job finishes or checkpoints the current chunk, flushes output files, closes DB connections and writes a resume marker, instead of leaving a half-written parquet/CSV that a rerun might trust.
  - Combine with idempotent, chunk-level outputs (write to temp name then atomic rename) so a kill at any moment leaves either the old or the complete new file.
  - Booking/payment services: a payment call to a supplier interrupted by shutdown is the "charged twice / lost" scenario. Pair graceful shutdown with idempotency keys and a reconciliation job, because SIGKILL and crashes can still happen.

## Sriniously — 3. What is a Backend, how do they work and why do we need them? (https://www.youtube.com/watch?v=6Ss4dJD9Kzg)
**A backend is a centrally hosted process listening on a port that owns data and privileged actions; frontend code runs sandboxed on untrusted, underpowered user devices, so data, secrets and heavy logic belong on the server.**

- **Mental models / first principles**
  - Definition: a computer listening for HTTP/WebSocket/gRPC (or other) requests on an internet-reachable port (80/443) that serves content (static files, JSON) and accepts data.
  - Reduced to one word, the backend exists for **data**: fetching, receiving, persisting it and acting on it. Example: you like a friend's Instagram post → app sends request → server identifies you, persists the like, finds the post owner, triggers a notification. That needs a centralized system with state about all users; each client only sees its own slice.
  - Frontend vs backend runtime: for a frontend, the server ships code (HTML, then CSS/JS/fonts in separate requests) and the **browser executes it** on the user's machine (paints after CSS, hydrates event listeners after JS). For a backend, the request is processed **on the server** and only results are returned.
- **Mechanisms (request path traced on a real AWS deployment)**
  1. Browser resolves the subdomain via DNS: an A record maps a name to an IP (CNAME maps a name to another name).
  2. Traffic reaches the EC2 instance's public IP but first passes the cloud firewall (AWS security group). Only allowed ports get through: 22 for SSH, 80 HTTP, 443 HTTPS. Block 80/443 and the request never reaches the machine.
  3. Nginx reverse proxy on the instance (a server in front of other servers, centralizing redirects/config): listens on 80, redirects to 443 (TLS certificates auto-managed by Certbot), matches `server_name` (the subdomain) and forwards to `localhost:3001` (the Node API). The frontend (Next.js) on the same box is a separate `server_name` forwarded to `localhost:3000`.
  4. Processes are kept alive by PM2 (one for frontend, one for backend). `curl localhost:3001/users` on the box returns the same JSON as the public URL: from the instance's view, the app is just a localhost service exposed via DNS + firewall + proxy.
- **Why not put backend logic in the frontend**
  - Security sandbox: browser code can access only the DOM, browser APIs (localStorage, cookies) and external APIs. Backends need the file system (logs) and environment variables (secrets). The sandbox exists because browsers run untrusted remote code that would otherwise read users' files.
  - CORS: browsers block cross-origin calls unless the target sends the right headers; backends must call arbitrary third-party APIs you do not control.
  - Databases: native drivers need raw sockets, binary protocols and persistent connections, which browsers lack. Backends keep a connection pool so thousands of requests per second do not open and close connections each time; per-user DB connections from browsers would overwhelm the DB and could not be pooled.
  - Compute: clients range down to 256 MB RAM single-core devices; heavy business logic would lag or crash. A central server can be scaled up whenever needed.
  - (Implied, and stated in the Vercel Part 3 video:) any credentials in frontend code are extractable.
- **Decisions & trade-offs**
  - Reverse proxy in front of app processes gives one place to handle TLS, redirects and routing many domains to many local ports, at the cost of one more hop and config to maintain.
  - A single VM hosting frontend and backend behind Nginx is fine to start (see DevOps video for when to grow out of it).
- **Rules of thumb / checklist**
  - When "it works on localhost but not on the domain", trace the hops in order: DNS record → IP → firewall/security group ports → reverse proxy `server_name`/upstream port → process manager status → app.
  - Expose only 80/443 publicly; keep app ports bound to localhost behind the proxy; restrict 22.
  - Keep secrets, DB access and third-party API calls server-side.
- **Failure modes & gotchas**
  - Missing security-group rule silently drops traffic before it reaches the server.
  - Calling third-party APIs directly from the browser fails on CORS and leaks API keys.
- **Relevance to Sam's work**
  - Supplier API keys (Viator, RateHawk, etc.) must only live server-side; the Drive Button or any blog-embedded widget should call Trameter's backend, which calls suppliers. This also lets you cache, rate-limit and normalise supplier responses centrally.
  - The hop-tracing checklist is the fastest way to debug a newly deployed internal tool or API.

