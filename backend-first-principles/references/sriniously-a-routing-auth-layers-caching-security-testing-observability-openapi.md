# sriniously-1 — cross-cutting principles

1. **Learn the layer, not the framework.** HTTP, routing, auth, caching and testing concepts carry over between languages and frameworks; framework knowledge doesn't survive a stack migration. (Roadmap #1)
2. **Every vulnerability is an assumption about data crossing a boundary.** Ask three things: where does data cross (user -> SQL, markdown -> HTML, user A -> user B's rows, process -> network)? What am I assuming? What if that's wrong? (Security #20)
3. **Keep code and data apart.** Use parameterized queries and argument-array exec. Never build SQL, shell commands or HTML by concatenating untrusted input (supplier/place-name data counts as untrusted). (Security #20)
4. **Layered request pipeline, with order mattering:** CORS -> logging/request ID -> auth -> permission -> bind/validate/transform -> handler -> service -> repository, and the global error handler last. Reject bad requests as early as possible. (Controllers #10, Roadmap #1)
5. **Strict responsibilities per layer.** Handlers own HTTP (bind, validate, set defaults, pick the status code). Services own business logic and orchestration with no HTTP awareness. Repositories run one query and return one shape. (Controllers #10)
6. **Request context carries identity, request/trace ID, and deadlines/cancellation** through every layer and into outbound calls. Take the acting user from auth context, never from the payload. (Controllers #10, Observability #18)
7. **Authorize at the point of access.** Scope every query by owner (`AND user_id=$ctx`), return 404 rather than 403 to avoid revealing that a record exists, check roles for admin functions, default-deny, centralize the logic, and test it in CI. (Security #20, Auth #8)
8. **Prefer stateful sessions** (random ≥128-bit ID in Redis; HttpOnly + Secure + SameSite cookie) for anything with users. Use JWTs only for real stateless scaling, with short expiries and refresh tokens. Use API keys for machine-to-machine calls and OAuth2/OIDC for delegation. Use a managed auth provider in production. (Auth #8, Security #20)
9. **Password storage = per-user salt + slow hash (Argon2id or bcrypt) with a tuned cost.** Auth error messages stay generic, and response timing is equalized so attackers can't enumerate users. (Auth #8, Security #20)
10. **Rate-limit in layers** (per IP, per account, global) using Redis counters. Apply the same idea to self-throttle outbound calls so you stay within third-party quotas. (Security #20, Caching #13)
11. **Cache read-heavy, slow-changing or expensive data.** Always use a TTL plus an eviction policy (LRU/LFU). Choose cache-aside + invalidation or write-through by how fresh the data must be. Cache third-party API responses to protect rate limits and per-call cost, but never cache data that must be live at decision time (prices/availability at booking). (Caching #13)
12. **Wrap anything you don't own behind your own interface.** Mock the wrapper, not the vendor SDK. Verify the wrapper against the vendor's sandbox, and where an OpenAPI spec exists, contract-test it so drift gets caught. (Testing #26, OpenAPI #28)
13. **Give network calls a signature.** One OpenAPI spec as the source of truth: generate clients and server interfaces from it (spec drift becomes a compile error), mock from it, and contract-test against it. Design-first lets frontend/backend (or agents) work in parallel. (OpenAPI #28)
14. **Functional core, imperative shell + dependency injection.** Pure functions hold the business and transform logic and need no mocks. I/O lives in a thin injected shell. Assert on resulting state, not call sequences, except where "exactly once" is itself the requirement (charges, emails). (Testing #26)
15. **Size tests by what they touch (process / machine / internet), not by "unit" vs "integration".** The cost comes from leaving the process. Use real databases cheaply (Testcontainers + template DBs or rollback-per-test). Never `sleep`; wait on a signal or inject a clock. Quarantine flaky tests instead of retrying them. (Testing #26)
16. **Coverage only shows what ran; the uncovered lines are where its value is.** Mutation testing checks whether your assertions actually constrain the code. (Testing #26)
17. **Observability = logs (what happened) + metrics (trends) + traces (where it failed).** Use structured JSON logs in production and pretty logs in dev, with request/user IDs on every line. Log business events, put level by environment (debug off in prod), and alert only on actionable thresholds. Instrument with OpenTelemetry to avoid vendor lock-in. (Observability #18)
18. **Routes:** identity goes in the path, modifiers (page/limit/sort/filter) in the query string, payloads in the body. Make query params optional with server-side defaults, paginate every list with metadata, version external APIs, and add a catch-all 404. (Routing #6, Controllers #10)
19. **Secrets and config hygiene.** Keep secrets in env vars or a secret manager, rotate any that ever touched git, and keep production logs at info level so traces and SQL don't leak. (Security #20)
20. **Offload heavy or non-critical work to queues, batch large datasets, and shut down gracefully.** Workers are producer -> broker -> consumer with retries and priorities. Process in batches and avoid N+1. On SIGTERM, stop intake, drain in-flight work, then close resources. (Roadmap #1)

---

## Sriniously — 1. Roadmap for backend from first principles (https://www.youtube.com/watch?v=0Rwb4Xmlcwc)
**Backend engineering = building reliable, scalable, fault-tolerant, maintainable systems; learn the language-agnostic foundations so knowledge transfers across stacks.**

- **Mental model / why first principles:** Most people learn backend through one framework (Express, Spring Boot, Rails) and see every problem through that ecosystem's lens. When the company migrates (e.g. Rails to Go for performance), only the underlying concepts transfer. The series is structured as a request's journey: browser -> network hops/firewalls -> server on a cloud box -> response, then each layer in turn.
- **Map of the curriculum (useful as a skill table of contents):**
  1. HTTP: raw messages, header categories (request, representation, general, security), method semantics, CORS simple vs pre-flight requests, status codes, HTTP caching (ETag, max-age), HTTP/1.1 vs 2 vs 3, content negotiation, persistent connections, compression (gzip, deflate, br), TLS.
  2. Routing: path vs query params, static/dynamic/nested/wildcard/regex routes, versioning and deprecation, route groups for shared middleware/permissions.
  3. Serialization: text (JSON/XML) vs binary (protobuf) — readability vs speed trade-off; missing/extra fields, nulls, date and timezone pitfalls; validate (JSON Schema) before deserializing.
  4. Auth: stateful vs stateless, sessions, JWT, cookies, OAuth/OIDC, API keys, MFA, hashing/salting, ABAC/RBAC/ReBAC; generic error messages, timing attacks (password-hash path is slower than unknown-user path, which leaks which usernames exist).
  5. Validation & transformation: syntactic (is it an email), semantic (DOB not in future, age 1–120), type checks; server-side validation is the real gate, client-side is only UX; fail fast; type-casting (path/query params arrive as strings), normalization (lowercase email, trim, add country code), sanitization, cross-field rules (password == confirm), conditional rules (partner name required only if married), aggregate all errors in one response.
  6. Middleware ordering (log -> authenticate -> validate -> handle -> error-handle), short-circuiting, security headers, CORS, CSRF, rate limiting, compression, body parsing.
  7. Request context: request-scoped state (user, request/trace IDs, deadlines, cancellation signals); keep it light and don't use it as a hidden coupling channel.
  8. Handlers/controllers/services/repositories (presentation -> business logic -> data access), CRUD-to-method mapping, pagination, sorting, filtering, REST conventions.
  9. Databases: relational vs not, ACID, CAP, indexing, connection pooling, transactions/concurrency, ORMs, migrations.
  10. Caching (cache-aside, write-through, write-behind, read-through; LRU/LFU/TTL/FIFO; L1 in-memory + L2 distributed), task queues & scheduling, Elasticsearch, error handling, config management, logging/monitoring/observability, graceful shutdown, security, scaling/performance, concurrency vs parallelism, object storage, realtime (WebSockets/SSE/pub-sub), testing, 12-factor, OpenAPI, webhooks, DevOps.
- **Task queue model:** producer -> broker/queue -> consumer, with a result backend. Use for emails, image processing, third-party calls (payments, webhooks), heavy batch work (e.g. "delete all my data" touching many tables: respond immediately, do the work in a background job). Support chains, parent/child dependencies, groups that run concurrently and join; retries; priorities (payment processing before notifications); rate limiting.
- **Graceful shutdown sequence:** catch SIGTERM/SIGINT -> stop accepting new requests -> finish in-flight work -> close DB connections/files -> exit. (SIGKILL cannot be caught.)
- **Performance rules of thumb:** avoid N+1 queries; index foreign keys and searched fields; process large data in batches to limit DB load; close file handles/connections to avoid leaks; compress and trim payloads; write clear code first, optimize after profiling; degrade gracefully when a dependency is down; offload non-critical work (emails, logging) to background workers. Concurrency helps I/O-bound work, parallelism helps CPU-bound work.
- **Webhooks vs APIs:** polling is client-initiated, webhooks are server push. Webhook hygiene: verify signatures, HTTPS only, acknowledge quickly then process, retries, logging; test locally with a tunnel (ngrok).
- **Code quality metrics:** cyclomatic complexity (number of independent paths through a function), maintainability index.
- **Relevance to Sam:** the queue/batch/graceful-shutdown/N+1/batching items map directly onto multi-million-row pipelines and supplier integrations; the webhook checklist applies to payment/booking callbacks from suppliers.

## Sriniously — 6. What is Routing in Backend? How Requests Find Their Way Home (https://www.youtube.com/watch?v=SubuU1iOC2s)
**The HTTP method expresses the "what" (intent); the route expresses the "where" (resource); method + path together form the key that maps to one handler.**

- **Mental model:** Routing is just a lookup from (method, path) to server-side logic. `GET /api/books` and `POST /api/books` share a path but are distinct keys, so they never clash and can hit different handlers. This is framework-independent: Java, Python, Node, Go and Rust routers all do the same matching.
- **Static routes:** fixed strings with no variable segments (`/api/books`). Always the same shape of response.
- **Dynamic routes / path parameters:** a placeholder segment, conventionally `:id` (`/api/users/:id`). The router extracts the segment into a named slot. Everything in a path is a string, even if it looks like a number, so the handler (or a validation/transform layer) must cast and validate it. Path params carry *semantic identity* ("the user whose id is 123") — the route should read like a sentence.
- **Query parameters:** key/value pairs after `?`. Used mainly on GET, which has no body in REST convention. They carry request metadata: pagination (`page`, `limit`), filters, sort field, sort order. Putting a free-text search term in the path (`/api/search/some value`) is technically possible but breaks the semantic structure and is hard to maintain; use `?query=`.
- **Pagination contract example:** `GET /api/books` returns `data: [...]` plus metadata like `total`, `currentPage`, `totalPages`, `limit` (e.g. default limit 20, 100 total -> 5 pages). The client then requests `?page=2`. Server picks defaults when params are absent.
- **Nested routes:** each extra level is its own route and handler: `/api/users` (all users) -> `/api/users/123` (one user) -> `/api/users/123/posts` (that user's posts) -> `/api/users/123/posts/456` (one post of that user). Use nesting to express ownership/containment; normal in any API of moderate complexity.
- **Versioning and deprecation:** prefix like `/api/v1/products` vs `/api/v2/products`. Example: v1 returned `{id, name, price}`, v2 returns `{id, title, price}` because a new client (mobile app) needed a different shape. Versioning lets you ship breaking changes without inventing new route names (`/api/new-products`), communicates intent, and gives client teams a migration window: announce deprecation of v1, let clients move, then remove v1.
- **Catch-all route:** register a final wildcard (`/*`) after all others so unknown paths get a clear "route not found" response instead of a framework default/empty response. Order matters: the catch-all must be last.
- **Rules of thumb:**
  - Identity in the path, modifiers (filter/sort/page) in the query string, payloads in the body.
  - Always validate and type-cast path/query values before use.
  - Version any external-facing API from day one; deprecate with a notice period.
  - Register a catch-all 404 handler.
- **Relevance to Sam:** when designing internal APIs over the location dataset or supplier data, design list endpoints with pagination metadata and filter/sort query params from the start (multi-million row tables cannot be returned unpaged); version the API because front-end/partner consumers will need shape changes; on the consuming side, supplier APIs (Viator, RateHawk) use the same patterns, so expect versioned paths and paginated list responses.

## Sriniously — 18. Logging, Monitoring and Observability (https://www.youtube.com/watch?v=5PEuwgLOQQM)
**Logs tell you what happened, metrics tell you the pattern/trend, traces tell you which component interaction failed; a system is observable when you can infer its internal state from those external outputs.**

- **Mental model:** Modern backends run on many servers/regions for users worldwide, so you need recorded evidence of what's going on. These are practices implemented on a spectrum, not a checklist you ever "complete"; nobody is 100% observable.
  - *Logging* = a journal of important events (logins, DB queries, failures, security-relevant events) with metadata: user ID, request ID, latency, which function/route, timestamp.
  - *Monitoring* = near-real-time state of the system: CPU, memory, requests/sec, open DB pool connections, error rate. "Real time" in practice means ~10–15 s aggregation delay so the telemetry pipeline isn't flooded.
  - *Observability* = three pillars: logs, metrics, traces. Traditional monitoring only says "something is wrong"; observability also tells you *where* and *why*.
  - *Trace* = one request's transaction across layers (middleware -> validation -> service -> repository -> DB), possibly starting at a frontend or load balancer.
- **Debugging workflow it enables:** alert fires (e.g. error rate above a threshold, posted to Slack via webhook) -> open metrics dashboard (throughput, error %, avg transaction time) -> jump to the logs correlated with the failing metric -> click a log line to open its trace -> see the exact function/segment where the request failed.
- **Log levels:**
  - DEBUG: maximum detail for local troubleshooting; disabled in production (too noisy).
  - INFO: normal operations and business events (e.g. "todo created").
  - WARN: not success, but not our fault/critical — e.g. a user typed the wrong password.
  - ERROR: validation failures, DB query failures, etc. — the main reason logging exists.
  - FATAL: unrecoverable; app shuts down and infra restarts it.
  - Choose the level from config/environment: debug in dev, info in prod.
- **Structured vs unstructured:** dev = coloured, human-readable console text; prod = JSON lines so log pipelines (ELK, Loki/Promtail/Grafana) can parse fields like user_id and request_id reliably. Parsing free text is fragile and error-prone. Make the format a config switch.
- **Instrumentation pattern shown (Go + New Relic):**
  1. A tracing middleware at the very start of the request creates a transaction and attaches attributes: service name, environment, client IP, user agent, request ID, user ID/email, tenant ID.
  2. The transaction is stored in the request context.
  3. Each service function pulls it from context, opens a segment, and defers ending it.
  4. Add business attributes (user ID, todo title, priority), log "starting create" at info, on DB error log at error *and* attach the error to the trace with the operation name, on success add the created ID, log debug details, and emit a business-event log with the entity metadata.
  - Result: one trace spans all layers and links to logs.
- **Terms:** *instrumentation* = measuring attributes of your code's execution. *OpenTelemetry* = open, vendor-neutral standard + SDKs for all major languages. You can run an OTel collector even when using a proprietary backend, so you keep control and portability.
- **Tooling trade-off:** open-source stack (Prometheus for metrics, Grafana dashboards, Loki/Promtail for logs, Jaeger for traces) gives control but takes real effort to run. All-in-one SaaS (New Relic, Datadog) is simpler for small teams without ops capacity. Dashboards also show runtime stats (GC time, memory, throughput).
- **Rules of thumb:**
  - Always attach a request ID and user/tenant ID to every log and trace.
  - Log business events, not just errors.
  - Treat auth failures as WARN, not ERROR, so error-rate alerts stay meaningful.
  - Use JSON in prod, pretty text in dev.
  - Alert on rates/thresholds and route to a chat channel. Only alert on actionable conditions.
  - Observability is shared: developers instrument code, ops runs the collection.
- **Failure modes:** text logs that tools can't parse; debug logging left on in prod (volume and cost); monitoring without traces (you know it's broken but not where); no correlation IDs, so logs can't be joined to a request.
- **Relevance to Sam:** for long-running pipelines over millions of rows, emit structured JSON progress/business-event logs (chunk number, rows in/out, rows rejected, duration) and per-chunk metrics so a failed run can be located without re-running. For supplier API calls (Viator, RateHawk, Coras), log supplier name, endpoint, latency, status and a correlation ID, and trace across your service -> supplier hop. For booking flows, the business-event logs double as an audit trail.

## Sriniously — 28. OpenAPI: The universal contract between clients and servers (https://www.youtube.com/watch?v=CwKsU84jIWs)
**Inside one codebase the compiler enforces function signatures; across a network hop nothing does. OpenAPI gives remote calls a machine-readable signature, so one file becomes the single source of truth for docs, clients, server stubs, mocks, validation and contract tests.**

- **The problem (first principles):** API knowledge normally lives in three places: the backend handler, the frontend fetch call, and a human-written doc. They drift.
  - Example: a colleague renames body field `to` -> `column` and updates backend and frontend but not the doc. Months later a new engineer follows the doc and gets validation errors.
  - Inside a Go codebase, changing a signature or a field type breaks the build at every call site at compile time. HTTP has no signature, so drift only shows up at runtime.
- **What OpenAPI is:** a JSON or YAML file describing every endpoint's inputs, outputs and behaviour. It is language-agnostic, readable by humans and machines, and lets you understand an API without reading source code or sniffing traffic.
  - Terminology: OAS = the spec/rules; OAD = your `openapi.json`/`.yaml` file; OAI = the Linux Foundation body that maintains the spec; Swagger = the original name, now just a tool brand (Swagger UI). OpenAPI 2.0 is the same content as Swagger 2.0.
  - History: Wordnik, 2010, ~6 engineers; customers wanted SDKs in many languages. The motivation was that teams code faster than they can document.
  - Versions: the latest spec is 3.2.0, but write 3.1.x because most tools only support up to 3.1. YAML is usually preferred over JSON because it allows comments.
- **File anatomy:**
  - **Required:** `openapi` (spec version), `info` (title + *your API's* version — don't confuse the two versions), and at least one of `paths`/`components`/`webhooks`.
  - **Optional root fields:** `servers` (list: local, staging, prod base URLs, so paths stay relative), `security`, `tags`.
  - **`paths`:** map of `/path` -> path item with one operation per HTTP method. Dynamic segments use `{id}`.
  - **Operation:** `summary`, `description` (markdown allowed), `tags`, `operationId`, `parameters`, `requestBody`, `responses`.
    - `operationId` must be unique and written like a function name (`getBoard`), because code generators turn it into a real function.
    - Parameters: `name`, `in` (path/query/header/cookie), `required`, `schema`. Path params are always required.
    - Request body: a `content` map from media type (`application/json`, `multipart/form-data` for uploads, `application/x-www-form-urlencoded`) to a schema. Document defaults (e.g. `actor` is optional and defaults to "guest").
    - Responses: keyed by quoted status codes, each with a description and content/schema. Document every outcome, not just 200. Example for `move`: 200 = event payload; 400 = validation error; 404 = task not found; 502 = the move succeeded but publishing to the message bus failed.
  - **Schema object:** type (string, number, integer, boolean, array, object, null). Strings take minLength/maxLength/pattern/enum, numbers take min/max, arrays take `items`. These compose to describe any shape.
- **Code-first vs design-first:**
  - *Code-first:* the spec is derived from handlers, through annotations/comments or from types (FastAPI generates it from type annotations; ts-rest + Zod schemas generate `openapi.json`). It suits existing servers. Generating from code/types beats generating from comments, which rot.
  - *Design-first / contract-first:* write the spec before the implementation. The teacher uses it for all new projects. Frontend and backend (or two AI agents) can then build in parallel against the agreed contract. Swagger's own creators moved this way too.
- **Design-first pipeline (concrete tools):**
  1. Lint the spec with Spectral (catches a missing contact, a missing description, an operation without an operationId, a response without a schema).
  2. Mock server with Prism: serves the documented examples so clients can build before the backend exists.
  3. Client codegen with openapi-typescript + openapi-fetch: typed paths/fields, so a misspelled field or wrong type is a compile error.
  4. Server codegen with oapi-codegen: produces Go interfaces the handlers must satisfy, so changing the spec without updating handlers breaks the build. This restores compile-time safety across the network.
  5. Docs: Swagger UI, Redoc or Scalar (the teacher's choice).
  6. Contract testing with Schemathesis: generates valid and invalid requests from the schemas, fires them at the real server, and flags responses that violate the contract. Also diff spec versions to catch breaking changes.
- **Other uses:** request validation in the server or API gateway straight from the spec; tool definitions for AI agents. GitHub (~344k lines) and Stripe (~171k lines) publish their specs.
- **Rules of thumb:**
  - One spec file is the source of truth; everything else is generated from it.
  - Document every status code and error body.
  - Use enums for status-like fields.
  - Keep operationIds function-like and unique.
  - Run lint and contract tests in CI.
- **Relevance to Sam:** for supplier integrations, check whether Viator/RateHawk/etc. publish an OpenAPI spec. If so, generate a typed client instead of hand-writing request/response models, and run Schemathesis-style checks or schema validation on responses to detect silent supplier changes (renamed fields, type changes) early. For Trameter's own internal APIs (location lookup, booking), going contract-first lets the frontend and backend work in parallel, and lets Claude/agents work on client and server sides independently.

## Sriniously — 10. What are controllers, services, repositories, middlewares and request context? (https://www.youtube.com/watch?v=hyc-7w3pee8)
**Inside the server, a request flows: entry point -> middlewares -> router -> handler/controller (HTTP in/out) -> service (business logic, orchestration) -> repository (one query, one result) -> back up; cross-cutting concerns live in ordered middleware and per-request state lives in a request context.**

- **Mental model:** The 3-layer split is not a requirement, it's a design pattern for scalability, maintainability, easier feature work and debugging. The dividing lines are responsibilities:
  - **Handler/controller:** owns HTTP concerns only. Receives request + response objects from the runtime.
  - **Service:** pure business logic. Looking at a service function, you shouldn't be able to tell it's used by an API — no status codes, no request parsing.
  - **Repository:** builds and runs one database query and returns its result.
- **Handler responsibilities, in order:**
  1. **Bind/deserialize:** extract query params (GET) or body (POST/PUT/PATCH/DELETE) and deserialize JSON into native types (Go struct, Python dict/class). Node often does this in a body-parser middleware upstream. Binding failure -> 400 and stop.
  2. **Validate** everything external (path, query, body), as specifically as possible.
  3. **Transform:** set defaults and normalize so downstream code has one predictable path. Example: `GET /books?sort=` with `sort` optional; if absent, default to `date`, so the service never branches on "was sort passed?". Rule: make query params optional and default them.
  4. **Call the service** with clean data plus auth info (user ID, permissions).
  5. **Map the result to a status code + body:** 200/201/204 on success (DELETE -> 204 no content); 4xx for client faults; 5xx for server faults. Structured error message.
- **Service responsibilities:** orchestrate — call multiple repository methods, merge/shape results, call external APIs, send emails/notifications. Some services never touch the DB (e.g. send an email, return `{success: true}`).
- **Repository rules:** single responsibility. Don't make one method return either a list or one item depending on an optional param; write `GetAllBooks` and `GetBookByID` separately. It takes filter/sort/insert data, constructs the query, returns the result.
- **Middleware mechanics:**
  - A middleware gets `request`, `response` and `next`. `next()` hands execution to the next context (another middleware, the router, or a handler).
  - It can read/modify the request, set response headers, or end the request early and send a response — saving server resources for rejected requests.
  - Why: the same reason functions exist. Thousands of endpoints all need the same security/logging/parsing/compression logic. Even calling a shared helper in every handler is duplication.
  - Good middleware candidates: code that must run for every (or a group of) request(s) and needs request/response access.
- **Common middlewares:**
  - CORS: check `Origin` against allowed frontends; if it matches, add `Access-Control-Allow-*` headers; otherwise add none and the browser blocks the response.
  - Security headers (CSP, etc.).
  - Authentication: extract token (JWT/session ID); invalid -> 401 immediately; valid -> put user ID/role/permissions into context, then `next()`.
  - Rate limiting: per IP, e.g. more than 30 calls in 2 s -> 429 Too Many Requests.
  - Logging/monitoring: method, path, query, body per request.
  - Compression (gzip) for large JSON.
  - Body parsing/binding, and optionally validation.
  - Global error handler.
- **Ordering matters:**
  - CORS first (reject foreign origins earliest), then logging, then auth, then route-specific permission checks, then the handler.
  - The global error handler goes **last**: execution flows one way, so an error handler placed mid-chain can't see errors raised after it.
  - The error handler classifies errors as client vs server (using fields on custom error types) and returns a consistent `{code/id, message, status}` that the frontend can map to friendly text.
- **Request context:**
  - Request-scoped key/value state readable and writable by every middleware and handler for that one request. It decouples layers: nothing has to be threaded through every function signature.
  - Uses:
    - Auth info (user ID, role, permissions).
    - A request ID (UUID) generated early, logged everywhere and forwarded to downstream services as a header like `X-Request-ID` for cross-service tracing.
    - Cancellation/abort signals and deadlines propagated to DB and external calls so the service never hangs forever.
- **Security rule:** take the acting user's ID from the auth context, never from the client payload. Otherwise a malicious client can send someone else's user_id (e.g. when inserting a book owned by user_id).
- **Relevance to Sam:**
  - Keep supplier-API calls in services (or a dedicated client layer) and DB access in repositories, so supplier logic can be unit-tested and swapped (Viator vs RateHawk).
  - Propagate a request/correlation ID into outbound supplier calls.
  - Always pass deadlines/cancellation to supplier HTTP calls so a slow supplier can't hang booking requests.
  - Default and normalize optional filters at the edge.
  - In pipelines, the same split applies: an I/O layer (read/write chunks), a transform layer (pure functions, easy to test), and an orchestration layer.

## Sriniously — 13. Caching, the secret behind it all (https://www.youtube.com/watch?v=estH64OkwxU)
**Caching = keeping a chosen subset of data in a place that is faster and cheaper to reach than the primary store. Use it whenever you would otherwise repeat heavy computation or repeatedly move heavy data, especially for read-heavy, rarely-changing data.**

- **First principles:** You pick the subset by access frequency and the likelihood of future use. You cannot cache everything: fast storage (RAM, edge servers) is expensive and limited, and caching everything defeats the purpose. Terms: *cache hit* (found) vs *cache miss* (not found, so go to the source, then usually store the result).
- **Motivating examples:**
  - *Google search:* a query like "weather today" is asked millions of times. Recomputing crawl/index/rank each time would crush the servers, so results go into a distributed in-memory cache spread worldwide. On a miss, run the full pipeline and cache the result.
  - *Netflix:* titles are encoded into multiple resolutions (1080p/720p/480p) for different devices and bandwidths, then served from CDN edge servers near users. ML/trend analysis decides *which* subset to place at each edge.
  - *X/Twitter trending:* computing trends over millions of tweets is very expensive and trends change over hours, not seconds. So recompute every few minutes, store the result in Redis, and serve every user from the cache. Recomputing on each request would take the service down.
- **Three levels a backend engineer meets:**
  1. **Network level — CDN:** request -> DNS resolves the CDN domain -> routed to the nearest PoP (point of presence = a cluster of edge servers), chosen by geography and network conditions -> edge checks cache -> hit: serve; miss: fetch from origin, cache, serve. Content carries a TTL so it is refreshed from origin after expiry. Also used for static web assets (Vercel edge network etc.).
  2. **Network level — DNS:** browser cache -> OS cache -> recursive resolver (ISP, or public resolvers like Google/Cloudflare) cache -> on a miss, root servers -> TLD (.com) server -> authoritative name server for the domain. Every layer caches so the recursive walk is rare.
  3. **Hardware:** CPU L1/L2 (per core), L3 (shared), RAM, then disk. Prefetching makes sequential array traversal fast. RAM is fast because any address is reached electrically in roughly constant time (vs a disk head seeking mechanically), but it is small and volatile — speed traded for capacity and persistence.
  4. **Software (in-memory key-value stores):** Redis, Memcached, managed AWS ElastiCache (open-source Redis alternative: Valkey). In-memory = reads/writes hit RAM; persistence to disk and reload on start is the tool's job. Key-value = no rigid schema; values can be strings, lists, JSON, etc.
- **Caching strategies:**
  - *Cache-aside / lazy:* check cache -> on a miss, read from DB, populate cache, return. Only caches what's actually requested.
  - *Write-through:* on every create/update, write to the DB and the cache in the same flow. The cache is always fresh, but every write is slower.
- **Eviction policies (memory is finite):**
  - *No eviction:* inserts error once full.
  - *LRU:* evict the key accessed longest ago.
  - *LFU:* evict the key with the lowest access count.
  - *TTL-based:* keys expire automatically; under pressure, evict the soonest-to-expire.
- **Backend use cases:**
  - *Expensive DB query cache:* a multi-join aggregation over millions of rows feeding a hot dashboard/landing endpoint. Cache the result with a TTL (e.g. 1 h); invalidate or update when the underlying data changes. Same idea: e-commerce product details during a sale; celebrity profile pages (read-heavy, written rarely).
  - *Session storage:* session tokens in Redis so every authenticated request avoids a DB round trip.
  - *External API response caching:* e.g. weather data with a 1 h TTL. Avoids hitting the provider's rate limits and per-call billing, and cuts latency. Only for data that is safe to be slightly stale.
  - *Rate limiting:* middleware reads the client IP (often from `X-Forwarded-For`, set by the reverse proxy) and increments a per-IP counter per window in Redis. Over the limit (e.g. 50 per minute) -> 429. It lives in Redis, not Postgres/MySQL: even 20–30 ms extra per request matters, and counter writes on every request would flood the primary DB.
- **Rules of thumb:**
  - Cache when the data is read-heavy and written rarely, or expensive to compute/fetch, and staleness within the TTL is acceptable.
  - Always set TTLs and an eviction policy.
  - Pick a freshness strategy: write-through for must-be-fresh data, cache-aside + TTL/invalidation otherwise.
  - Don't put data that must be real-time behind long TTLs.
- **Failure modes:**
  - Serving stale data after writes when nobody invalidates.
  - A full cache with no eviction policy, so writes error.
  - Treating the cache as the source of truth (it's volatile).
  - Caching everything, which raises cost and lowers the hit rate.
  - (Note: the teacher doesn't cover stampede/thundering-herd on popular-key expiry or cache key design; consider request coalescing or jittered TTLs in practice.)
- **Relevance to Sam:**
  - *Supplier APIs* (Viator, RateHawk, Coras): caching static/slow-changing data (product content, hotel static info, location metadata) with TTLs protects rate limits and per-call costs. Prices/availability need short TTLs or none, and must be re-checked at booking time.
  - *Matching pipelines:* memoize expensive lookups such as geocoding or GeoNames joins. On a 16 GB machine, keep hot lookup tables in RAM but bounded (LRU, e.g. `functools.lru_cache(maxsize=...)`), and spill the rest to disk-backed stores (SQLite/Parquet) rather than holding millions of rows in Python dicts.
  - *Rate limiting:* a Redis counter is also the pattern to *self*-throttle outbound calls to suppliers.

## Sriniously — 26. Testing for Backend Engineers: mocks, TDD and coverage (https://www.youtube.com/watch?v=HgVrfHX92y4)
**A test is a second program that runs your program and asserts on what happened. Its main job is catching regressions in CI. Classify tests by what they're *allowed to touch* (process / machine / internet), push logic into a pure core, wrap what you don't own, test against real databases cheaply, and never sleep in tests.**

- **First principles:** A bad, rushed or incomplete test is worse than none because it gives false confidence. The primary value is regression detection: CI (e.g. GitHub Actions) runs the suite on every PR, so already-fixed bugs can't silently return and you can ship fast. Scope here: the ~80% of testing a backend engineer writes and runs themselves, not manual QA, mobile or browser automation.
- **Mechanisms:**
  - *Test runner:* finds test files, generates the `main` entry point, compiles and runs, then reports. Shown with `go test -work` and `-x`: Go writes `_testmain.go`, links `board.test`, executes it.
  - *Assertion:* a check that fails the test. In Go it's just `if` + `t.Errorf`. Always write a failure message that says what was wanted vs what was got.
- **Test types:**
  - Unit = one unit in isolation. There is no universal definition of "unit"; with mocks or real collaborators, both count.
  - Integration = your code plus something you didn't write (DB).
  - End-to-end = the full user journey, including login and using the real session.
  - The pyramid is about cost (CPU, memory, time), not just money.
  - Demo: the same "move task" check run four ways (mock bus, real in-process bus/hub, real HTTP server on a port, real Postgres) took ~0 s, 0 s, 0 s and 10.5 s. **Cost comes from leaving the process, not from how much logic you cover.**
- **Better mental model — size by what a test may touch:**
  - *Small:* one process; no network, DB, filesystem or sleeps.
  - *Medium:* same machine only (local DB/Redis containers); no internet.
  - *Large:* anything.
  - Unlike "is this a unit test?", this is mechanically enforceable (did it open a socket?), and it removes most flakiness, which comes from crossing the machine boundary.
  - Regression, functional, performance and security tests are *purposes* layered on top of those scopes, not separate kinds.
- **Test doubles (the stunt-double analogy):**
  - *Dummy:* fills a required parameter, never used (e.g. a no-op logger).
  - *Stub:* canned answer regardless of input.
  - *Spy:* a stub that also records calls.
  - *Mock:* pre-set expectations (e.g. "save called exactly once with X"); fails otherwise.
  - *Fake:* working lightweight implementation (e.g. an in-memory map repo).
- **State vs interaction verification:**
  - Prefer asserting the resulting state (task is in Done, seq incremented).
  - Interaction checks ("publish called twice with task.moved") become *change-detector tests*: they break on harmless refactors such as merging two events into one.
  - Use interaction checks only when the interaction *is* the requirement, e.g. an email must be sent exactly once.
- **"Don't mock what you don't own":**
  - Mocking a payment provider's client with invented response shapes means the test only verifies your own assumptions. Production then fails on real field names, duplicate-charge behaviour and error formats.
  - Fix: wrap the third party in your own small interface with your own success/error types, and mock *that*.
  - Add a few small contract tests against the provider's **sandbox** to verify the wrapper's request/response mapping.
- **Testability design:**
  - Handlers that construct their own DB/Redis clients can't be tested. Use dependency injection: pass dependencies in.
  - *Functional core, imperative shell:* pure functions (values in, values out) hold the decisions/business rules; a thin outer shell does all I/O. The core needs no mocks; the shell is too thin to need much testing.
- **Databases in tests:**
  - In-memory fakes (Node in-memory, Go fake repos, Java H2) trade fidelity for speed.
  - The real bugs live in the DB layer: wrong column names, missing indexes, forgotten unique constraints, broken transactions, Postgres-only JSON operators.
  - Use a real DB, made cheap:
    1. *Transaction per test:* begin, run, roll back. Needs code that accepts a transaction handle (DI again).
    2. *Template database:* run migrations once on a template; each test clones it in milliseconds, so tests can run in parallel.
  - *Testcontainers* starts a real Postgres (v18 in the demo) from test code. Container ready in ~7 s once per package; 9 tests in ~3 s; 10.5 s total.
- **Network:** wrap external APIs. When you need to exercise real HTTP (headers, client code), spin up a local test HTTP server (Go `httptest`) or intercept at the network layer (MSW in Node).
- **Time:**
  - `sleep(100ms)` then assert passes locally but fails on a loaded CI box that needs 110 ms. Bumping the sleep only makes tests slower and still flaky.
  - Fix 1: wait on a condition or signal (a channel or callback).
  - Fix 2: inject a clock you control.
- **Hermetic tests:** a test brings up everything it needs and tears it down, so it runs anywhere (laptop, SSH box, CI). Testcontainers = hermetic DB; injected clock = hermetic time.
- **TDD:**
  - Red (write a failing test for missing behaviour) -> green (smallest code to pass, ugly allowed) -> refactor (mandatory, and the step most people skip).
  - Red forces API design from the caller's side (awkward test = awkward API) and proves the test can fail.
  - Example: rule "can't go To-do -> Done without Doing". Failing test (expected error, got nil) -> add an `if` -> extract a pure `canMove(from, to)`.
  - Ian Cooper's point: the "unit" is a *behaviour* at a public interface, not a class. One-test-class-per-class with everything mocked makes refactors break hundreds of tests.
  - Context: DHH's "TDD is dead" debate with Kent Beck and Martin Fowler.
  - Teacher's view: use TDD when the expected output is clear but the implementation isn't (parsers, pricing rules, state machines). Skip it for exploratory design work.
- **Flaky tests:**
  - Worse than missing tests: they train the team to hit "re-run", so real failures get ignored.
  - Luo et al. 2014 (201 flaky-test fixes): top causes are async-wait, concurrency races, and test-order dependency (B passes only after A leaves a row/global/file behind). Order dependency surfaces when tests are added or parallelism is turned on.
  - Options: delete it (often fine), quarantine it with owner, date and reason (teacher's preference), or retry (bad).
- **Coverage:**
  - The tool instruments basic blocks with counters, so coverage proves code *ran*, not that anything was checked. Assertion-free tests can hit 100%.
  - Line vs branch: an `if` with no `else`, tested only on the true path, gives 100% line and 50% branch coverage.
  - Inozemtseva & Holmes (ICSE 2014): controlling for suite size, coverage has low-to-moderate correlation with fault detection. (note: the transcript garbles the authors' names.)
  - Rough guide: 60% acceptable, 75% commendable, 90% exemplary; no universal target. The value is in the *uncovered* (red) lines.
  - Mandating 100% breeds low-value tests.
- **Mutation testing:**
  - Tests your tests: the tool introduces mutants (`>` to `>=`, `+` to `-`, return 0) and checks whether the suite kills them.
  - Demo: a pagination function at 100% line coverage scored only 52.63% with Stryker (9 of 19 mutants survived, including `<=` -> `<` on `offset+limit`).
- **Complexity metrics:**
  - *Cyclomatic complexity* (McCabe 1976) = E − N + 2, or 1 + number of decision points (if, loop, case, &&, ||). The "keep ≤10" guideline is reasonable, not magic. A flat 20-case switch scores 21 but reads easily.
  - *Cognitive complexity* (SonarSource) penalises nesting instead. Demo: hub `Run` method scored 8 cyclomatic vs 19 cognitive.
  - Treat both as flags for discussion, not rules.
- **Static tools:** linters (suspicious patterns like always-true conditions or unchecked errors; not syntax or logic), formatters, type checkers, static analysis (tracks values through code without running it). Code quality is ultimately subjective.
- **Relevance to Sam:**
  - *Supplier integrations:* wrap each supplier (Viator, RateHawk, Coras) behind your own interface and types, mock the wrapper in tests, and run a handful of sandbox contract tests to catch supplier response drift.
  - *Booking/payment flows:* interaction verification is legitimate for "charge exactly once" and "send confirmation once". Pair this with idempotency.
  - *Data pipelines:* make transform/matching logic pure functions and test them with small fixture DataFrames (small tests, no I/O). Keep file/DB I/O in a thin shell. Use a template DB or Testcontainers for SQL-level tests.
  - *Fuzzy matching, pricing rules:* mutation testing is a cheap way to check that the tests actually constrain the logic.
  - Never `sleep` in tests of async/concurrent pipeline code. Wait on completion signals.

## Sriniously — 8. Authentication and authorization for backend engineers (https://www.youtube.com/watch?v=A95rliroC8Q)
**Authentication answers "who are you?" in a context; authorization answers "what can you do there?". Pick stateful sessions for web apps, stateless JWTs for distributed APIs, API keys for machine-to-machine, OAuth2/OIDC for delegation and social login, and use RBAC for permissions.**

- **Mental model via history (why the mechanisms look the way they do):**
  - Implicit trust (a village elder vouches) doesn't scale.
  - Wax seals = "something you have"; forgery = the first bypass attack.
  - Telegraph passphrases = "something you know" (static shared secrets).
  - MIT CTSS 1961 introduced passwords for multi-user systems, stored them in plaintext, and someone printed the password file. That gave rise to hashing: one-way, fixed-length output, same input -> same hash.
  - Diffie–Hellman (1970s) -> asymmetric crypto/PKI. Kerberos -> ticket-based auth (precursor of tokens).
  - 1990s MFA = know + have + are (biometrics have false-positive/negative and template-security issues).
  - Today: OAuth2, JWT, zero-trust, passwordless/WebAuthn (hardware-held keys). Future: decentralized identity, behavioural biometrics, post-quantum crypto.
- **Sessions (stateful):**
  - HTTP is stateless by design, but carts and staying logged in need memory.
  - Flow: on login the server creates a random session ID, stores {session -> user data} in a store, sends the ID in a cookie; every later request carries it; the server looks it up. Sessions expire (e.g. 15 min).
  - Storage evolved: files on server (don't scale) -> DB (survives restarts) -> Redis/Memcached (in-memory, fast, shared across servers).
- **JWT (stateless, formalized 2015):**
  - Why: by the mid-2000s, holding session state for millions of users was costly, and replicating it across regions added latency and consistency problems.
  - Structure: base64url header (alg) . payload (claims: `sub` = user ID, `iat` = issued-at, optional name/role) . signature made with a server secret. Tampering breaks the signature.
  - Pros: no session store; any server holding the key can verify (microservices, horizontal scaling); portable (headers, URLs, cookies).
  - Cons: a stolen token can be used until it expires; you can't revoke a single token without rotating the secret, which logs everyone out.
  - The "hybrid" fix (check a blacklist in Redis on each request) brings back a state lookup. That raises the question of why not just use sessions.
  - Payload is only encoded, not encrypted. Don't put secrets in it.
- **Cookies:** a server-set value stored by the browser and sent automatically on every later request to that same server. Browsers isolate cookies per server. For session IDs use `HttpOnly` so JS can't read them.
- **Four auth types and when to use them:**
  - *Stateful (session ID + Redis + HttpOnly cookie):*
    - Pros: central control, real-time view of active sessions, easy revoke/logout.
    - Cons: harder to scale across regions, more operational complexity.
    - Teacher: most apps, especially web/SaaS, should default to this.
  - *Stateless (JWT in `Authorization` header):*
    - Pros: scales, no session-store dependency, good for distributed systems and mobile clients without cookies.
    - Cons: revocation is hard.
    - Hybrid: stateful for the browser web app, stateless for mobile/third-party/server clients.
  - *API keys:*
    - A cryptographically random string generated from a UI, with scoped permissions and expiry. Used for programmatic machine-to-machine access (e.g. calling OpenAI's API from your server rather than using ChatGPT's UI).
    - The server maps key -> account/plan/quota. The client keeps it in env/secure storage and sends it on every request.
    - No interactive login needed.
  - *OAuth 2.0 / OIDC:* third-party delegation and "Sign in with Google".
- **OAuth — the delegation problem:**
  - Apps needed other apps' resources (a travel app scanning Gmail for flight tickets; importing contacts).
  - Users first shared passwords: full access, no scoping, revocation means changing passwords everywhere.
  - OAuth (2007) shares *scoped tokens* instead.
  - Roles: resource owner (user), client (requesting app), resource server (holds the data), authorization server (issues tokens).
  - OAuth 1.0 flow: client redirects user to auth server -> user authenticates and consents to scopes -> auth server gives client a token -> client calls resource server with it. Downsides: complex, and its signature handling was error-prone.
  - OAuth 2.0 (~2010) brought bearer tokens (simpler, but whoever holds one can use it) and flows per client type:
    - Authorization code: server-side apps.
    - Implicit: browser apps; now discouraged.
    - Client credentials: machine-to-machine.
    - Device code: smart TVs and other limited-input devices.
  - (note: modern guidance is authorization code + PKCE for browser/mobile apps; the teacher doesn't mention PKCE.)
- **OIDC (~2014):**
  - OAuth handles authorization but not identity. OIDC adds an **ID token** (a JWT with user ID, iat, issuer, name, email).
  - Flow: "Sign in with Google" -> redirect -> login and consent at Google -> client receives an auth code (+ maybe an ID token) -> client exchanges the code for an access token + ID token -> uses the access token for scoped resources (e.g. Google Keep notes).
- **Authorization / RBAC:**
  - Example: a notes app where deleted notes move to a "dead zone" after 30 days and only admins may access it.
  - Anti-pattern: a shared "god-mode" string sent with API calls. If intercepted it's catastrophic, and it can't be managed per person.
  - RBAC: roles (user/admin/moderator/custom) map to permissions per resource (read/write/delete).
  - Flow: on signup assign a role; per request derive the role from session/JWT or a DB lookup early in the middleware chain, attach it to the request context, and enforce downstream. Insufficient permission -> **403 Forbidden** (vs 401 for not authenticated). Multi-tenant orgs let admins assign roles to members.
- **Security gotchas:**
  - *Generic auth errors:* "user not found" or "incorrect password" tells attackers which half to brute-force. Return one generic "authentication failed" for unknown user, wrong password and locked account. Be friendly elsewhere (validation), never in auth.
  - *Timing attacks:* steps are find user -> check lockout -> hash and compare. An unknown user returns faster than a wrong password because hashing takes time, which leaks valid usernames. Fixes: constant-time comparison functions; equalize timing (e.g. an artificial delay, or hash against a dummy hash for unknown users).
  - Passwords are stored only as hashes. Verify by hashing the input with the same algorithm (+ salt) and comparing.
- **Practical advice:** implement auth yourself to learn it, but in production for medium/large systems use a managed provider (Auth0, Clerk, etc.) unless you're very confident. Hashing, salting, token handling and revocation are their headache.
- **Relevance to Sam:**
  - *Inbound:* Trameter's own APIs consumed by partner servers or internal pipelines -> API keys with scopes/expiry stored hashed; web dashboard users -> session cookies; admin pipelines -> role checks (RBAC) and 403s.
  - *Outbound:* supplier APIs (Viator, RateHawk) are usually API-key or OAuth2 client-credentials. Keep keys in env/secret storage, never in code or logs. Cache client-credentials access tokens until near expiry instead of fetching one per call.
  - If Trameter's Drive Button/Creator Studio ever imports from Google (Gmail tickets, Drive), that is exactly the OAuth/OIDC delegation case.

## Sriniously — 20. Backend Security: Everything You Need to Know (https://www.youtube.com/watch?v=xB1C1xZZW4k)
**Every vulnerability comes from a developer's assumption (input is clean, the caller is who they claim, requests come from our frontend, nobody will open the network tab). Most of them happen where data crosses a boundary between languages, systems or privilege levels. Defend in layers, and be paranoid by habit.**

- **Scope and mindset:**
  - Security domains: browser (cookies, storage), network (TLS), server OS, and backend app code. This talk covers the backend.
  - Nothing is ever fully secure. The goal is a habit of asking "what could go wrong here?".
  - Attackers don't care about your framework. They hunt for your assumptions and poke every boundary. Deadline-driven, happy-path thinking is where assumptions creep in.
- **Injection — root cause:**
  - Your app "speaks" many languages: SQL to the DB, HTML/JS to browsers, shell to the OS. Each has special characters that separate code from data.
  - Injection happens when user data crosses into another language's context and gets interpreted as code.
- **SQL injection, worked example:**
  - Query built by string concatenation: `SELECT * FROM users WHERE email = '<input>'`.
  - Input `' OR '1'='1' --`: the first quote closes the string, `OR` adds an always-true clause, `--` comments out the trailing quote. Result: every row in users, including password hashes, addresses and phones in a real table.
  - Input `'; DROP TABLE users; --` stacks a destructive statement. Modern drivers often refuse stacked statements, but old drivers may not.
  - Attackers can also use UNION to read other tables (e.g. payments), DB functions to read server files, and in some configs run OS commands.
  - **Fix: parameterized queries / prepared statements.** Send the template with placeholders (`$1`) and the values separately; the DB treats the values purely as data. Every modern driver and ORM does this by default. You are only exposed if you build raw SQL strings yourself.
  - **Defense in depth:** the app's DB user should have DML rights only (SELECT/INSERT/UPDATE/DELETE), never DDL (DROP/ALTER). Validation should also reject non-email strings before they reach the query.
- **NoSQL injection:** Mongo queries are JSON-like objects with `$`-operators (`$ne`, `$gt`, `$exists`). If a raw client JSON body is passed in as the query, an attacker controls its *structure*, e.g. `{"email": {"$ne": null}}`.
- **Command injection:**
  - Example: an image upload processed by shelling out to ffmpeg with a user-chosen output name. Input `x.jpg; rm -rf /` runs a second command. Pipes and `&` (background spyware) are also possible.
  - Fix: use exec APIs that take the command and an **argument array** separately, so no shell interpreter sees the input (Node, Go and Python all have these).
- **Injection rule:** whenever you build a string that another system will interpret and it contains user input, stop and use the parameterized alternative. One almost always exists.
- **Authentication — use a provider:**
  - Managed auth (Clerk, Auth0) saves weeks of work on stateful sessions, device lists and revoke-all, social OAuth, linking email+password accounts with Google sign-in for the same email, RBAC, and new-attack response.
  - Consider bringing auth in-house only when the bill is around $10–20k/month and you have revenue and team capacity. You still need to understand how auth works to configure and integrate it safely.
- **Password storage progression:**
  1. *Plaintext:* breaches happen at companies of every size (insiders, compromised vendors). More than 70% of users reuse passwords, so one leak enables credential stuffing on other sites. Staff can also read passwords.
  2. *Hashing:* one-way function, fixed-length output, deterministic.
  3. *Rainbow tables:* precomputed hashes of common passwords reverse unsalted hashes. Fix: **per-user random salt** from a CSPRNG, stored alongside, and hash(password + salt). Precomputed tables then match nothing.
  4. *GPU brute force:* a GPU computes billions of SHA-256 hashes per second, so even salted fast hashes fall to offline cracking (all 8-character passwords within days). Fix: **slow password hashes** with a tunable cost/work factor — bcrypt, scrypt, or Argon2id (current standard). At ~400 ms per hash, a login barely notices but an attacker drops from billions to ~4–5 guesses per second (decades). Never use MD5/SHA-256 for passwords; they are for file integrity.
  - Reference: the Lucia auth guide.
- **Sessions:**
  - Session ID: random from a CSPRNG, ≥128 bits, so it can't be guessed.
  - Store in Redis or Postgres with metadata: user, created_at, expiry (e.g. 7 days), IP, user agent. This enables a device list and revocation.
  - Send it in a cookie. Cookie flags:
    - `HttpOnly`: JS (e.g. XSS payloads) can't read it. Never keep tokens in localStorage.
    - `Secure`: sent over HTTPS only, safe from Wi-Fi/ISP sniffing.
    - `SameSite`: `Strict` (best), `Lax` (top-level navigations only; browser default), `None` (requires Secure; avoid for session cookies).
- **JWT issues:**
  - Revocation is hard: you can't wipe a token from the client.
  - Workarounds: a server-side blacklist, or short-lived access tokens (5–10 min) plus refresh tokens (1–7 days). On a 401 the client trades the refresh token for a new pair, so a stolen access token dies quickly.
  - The payload is only base64, so never put sensitive data in it. Editing the payload breaks the signature.
  - Teacher's verdict: prefer stateful sessions unless you truly need stateless horizontal auth, and even then a shared Redis session store works. If you must use JWTs: short expiries (minutes to hours, not days), refresh tokens stored server-side, HttpOnly cookies.
- **Rate limiting auth endpoints (layered; stricter than for other endpoints):**
  - (a) Per IP, e.g. 10 login attempts per minute. Weak because NAT'd universities/offices share an IP and attackers rotate IPs via botnets, proxies or VPNs.
  - (b) Per account, e.g. 5 failures in 15 min locks the account for 24 h. Stops IP rotation, but not password spraying (one common password tried across many accounts).
  - (c) A global cap on failed logins system-wide (e.g. 100 per minute) that triggers alerts, blocking and CAPTCHA.
  - Rate limiting also protects availability: floods can crash the server.
- **Authorization pitfalls:**
  - Authentication + a role/permission check at the router gives a false sense of security.
  - **BOLA/IDOR (horizontal escalation):** `GET /books?id=5` passes "has read:books", but book 5 belongs to someone else. With a script, an attacker can iterate through every invoice. Fix: scope every query (select/update/delete/insert) by owner, e.g. `WHERE id=$1 AND user_id=$ctxUser`.
  - **Leaking existence:** fetch-then-check returning 403 confirms the object exists and enables enumeration for later social engineering. Return 404 when the scoped query finds no row.
  - **BFLA (vertical escalation):** `/admin/invoices` protected only by an unshared URL ("security through obscurity"). Anyone who logs in and finds the URL gets everything. Fix: role middleware (admin) in addition to permission checks.
  - **Sequential IDs** (101, 102…) invite enumeration. UUIDs help (note: random UUIDv4 primary keys have index-locality costs; the teacher flags "performance issues" without detail).
  - **Framework:**
    1. Centralize authorization logic in one layer.
    2. Default deny: anything not explicitly allowed is refused, so new endpoints/resources are protected automatically.
    3. Write automated authorization tests in CI: A can't read B's data, a member can't call admin functions, anonymous users can't reach authenticated routes.
    4. Audit log every admin/sensitive access and every failed authorization check (a probing signal).
- **XSS:**
  - Attacker JS runs in another user's browser in your site's context. It can read the page, call APIs as the user, steal non-HttpOnly cookies/localStorage, redirect to phishing pages, or rewrite content.
  - *Stored XSS:* e.g. a comment system rendering user markdown->HTML via innerHTML (React's `dangerouslySetInnerHTML` is named to warn you). A smuggled `<script>` runs for every viewer.
  - Other types: *reflected* (script in the URL), *DOM-based*.
  - Same root cause as injection: data treated as code.
  - Prevention: sanitize user-supplied markup server-side before storing it (strip scripts etc.).
  - **CSP** (Content-Security-Policy header): whitelist script/image sources and disallow inline scripts. It's a last line of defense, not the fix.
- **CSRF:** a malicious site makes the victim's browser send a request to bank.com, and the cookies ride along. Largely mitigated today by the SameSite default of Lax (unless you set None) plus CORS checks. Mostly a legacy concern.
- **Misconfiguration:**
  - *Secrets:* never in source control; use env vars or a secret manager (AWS Parameter Store, HashiCorp Vault). A secret deleted in a later commit is still in git history, so **rotate it immediately**.
  - *Debug logging in prod:* leaks stack traces (code structure), raw SQL, DB pool config and user data. Set the log level per environment (info in prod).
  - *Security headers:* e.g. `X-Frame-Options` against clickjacking (embedding your site in an attacker's iframe). Use the framework's security middleware (one line in Express/Echo/Gin/Django).
- **Unifying principle and layers:**
  - Every bug above is data crossing a boundary: input -> SQL, markdown -> HTML, user A -> user B's resources.
  - Ask three questions: where does data cross a boundary? What am I assuming? What if that's wrong?
  - Layers:
    1. Strict input validation at the entry point.
    2. Parameterized operations (DB, OS).
    3. Authorization at the point of access (repository), not just upstream.
    4. Headers/policies (CORS, CSP) to limit blast radius.
    5. Monitoring, logging and alerting on suspicious activity.
  - An attacker must beat all layers at once.
- **Resources:** PortSwigger Web Security Academy (free labs: SQLi, XSS, CSRF, clickjacking, SSRF, OS command injection, OAuth/JWT attacks), OWASP Top 10 (#1 broken access control; also misconfiguration, injection), OWASP Cheat Sheets (authentication, session management).
- **Relevance to Sam:**
  - *Pipelines:* when bulk-loading supplier/GeoNames data into a DB, use parameterized/bulk-copy APIs (`COPY`, `executemany` with params), never f-string SQL. Place names contain quotes (e.g. "Côte d'Ivoire", "L'Aquila") that break or inject into concatenated SQL.
  - *Shell-outs* (ogr2ogr, osmium, ffmpeg) must use argument lists (`subprocess.run([...])`, no `shell=True`) when filenames or params come from data.
  - *Supplier API keys* belong in env/secret manager; rotate any that touched git.
  - *Booking/invoice endpoints* need owner-scoped queries and 404-not-403 responses.
  - *Partner-facing APIs* need layered rate limits.
  - *Rendering supplier descriptions* (HTML from Viator etc.) in the Drive Button UI is a stored-XSS vector: sanitize it.

