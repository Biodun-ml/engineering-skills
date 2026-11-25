# ops-2 — cross-cutting principles

1. **Monitoring handles known unknowns; observability handles unknown unknowns.** Keep a few monitors for known-bad conditions, but invest in data that lets you answer new questions without shipping new code. (Liz Fong-Jones; Charity Majors)
2. **Alert on user-facing symptoms, not causes.** Page on error-budget burn or a handful of signals (request rate, errors, latency, maybe saturation) plus end-to-end checks of the money path. Declare bankruptcy on CPU/disk threshold alerts. (Fong-Jones; Majors)
3. **Define SLIs per request, from critical user journeys.** Each event resolves to good/bad/out-of-scope in business terms (a 403 for bad credentials isn't a failure). Count slow successes as failures, and prefer "X% of requests under T ms" to "P99 < T". (Fong-Jones; Piyush)
4. **Start with one SLO, not a catalogue.** Even "all HTTP requests, from server logs" beats nothing. Use history as the first, achievable target; about 3–5 SLOs per service; never 100% or six nines. (Fong-Jones; Piyush; SRE 2E authors)
5. **Measure → target → guarantee.** SLI before SLO before SLA; make the SLA looser than the internal SLO; error budget = 100% − SLO (99.9% ≈ 43 min/30 days). It decides when to ship vs stabilize. (Piyush)
6. **Error-budget policy is a tool, not dogma.** Agree consequences in advance, but adapt the policy to your own risk appetite. The lasting value of SLOs is making user experience visible. (SRE 2E authors)
7. **Dependencies cap your reliability unless you mitigate.** Never hard-depend on something less reliable than your own SLO. Add caching, retries, replicas or graceful degradation. Deliberately failing a too-reliable service (Chubby) stops dependents from assuming it never fails. (Fong-Jones; Piyush)
8. **Emit wide, structured, high-cardinality events.** One canonical event/log line per request (or batch) with every useful field: IDs, business nouns, downstream latencies, build ID, feature flags. High-cardinality fields (user/app/booking ID) are what locate culprits; metrics can't hold them. (Majors; Better Stack; Fong-Jones)
9. **Reduce data by smart sampling, not aggregation.** Keep every error/rare event, sample common successes, record the sample rate per event. Add retention tiers (e.g. debug 7 d, errors 90 d, audit ~1 y). (Majors; Better Stack)
10. **Logs are a liability too.** Never log secrets/PII (Twitter 2018 plaintext passwords). Make types redact by default, add a pipeline redaction stage, encrypt, restrict access. Use efficient structured loggers (slog ~3% vs Logrus ~20% overhead in one benchmark). (Better Stack)
11. **Use OpenTelemetry so instrumentation outlives vendors.** Auto-instrument HTTP/DB calls, export OTLP anywhere; tracing beats correlating timestamps across dashboards once requests cross services. (Fong-Jones; Better Stack)
12. **Observability is part of "done".** PRs should include an observability plan and deployment plan (a link to the query you expect to change). After each deploy, look at production; you must know what normal looks like. (Fong-Jones; Majors)
13. **Deploys cause most outages, so harden deploys rather than polishing staging.** Small batches, feature flags (and the flag as an event dimension), canaries, alerts when too many versions coexist. Test in production deliberately. (Majors; Forsgren)
14. **Speed and stability are not a trade-off.** The four DORA metrics (deploy frequency, lead time, time to restore, change failure rate) move together. High performers deploy 46x more often with 1/5 the change-failure rate. Small batches are the mechanism. (Forsgren)
15. **Architecture beats technology choice.** Loose coupling (deploy/test independently, no fine-grained coordination, deploy in business hours) predicted performance; mainframe vs cloud didn't. Fix the top constraint (often coupling, change approval, real CI), re-evaluate, repeat; avoid 50 parallel initiatives. (Forsgren)
16. **Zero-downtime schema change = expand/contract.** Two app versions share the DB during deploys, so every step must be backward and forward compatible. Add → dual-write → batched backfill → switch reads → stop old writes → drop much later after backup. Never run destructive DDL in the same release; batch big UPDATEs to bound lock time; keep backfills as versioned migrations. (Edson Yanaga)
17. **Blameless postmortems fix systems, not people.** Ask how the system allowed it; record where you got lucky; turn each into an action. Generative (Westrum) culture, where failure leads to inquiry, predicts high performance. (Clément Mihailescu; Forsgren)
18. **Reliability is a business function.** Its purpose is user trust and scaling sublinearly in cost. Argue for it in business terms or it gets cut as overhead. Copy principles, not Google's implementations. (SRE 2E authors; Fong-Jones)
19. **Don't run Kubernetes without a need and a team.** For a monolith/API + DB + queue, use PaaS, serverless or a managed container service. Prefer the highest-level managed platform that fits. (DevOps For Developers)
20. **For Sam specifically:** solo, no on-call → symptom alerts only (booking success, search latency, pipeline freshness) + one synthetic booking check; per-booking and per-chunk canonical events; supplier calls wrapped in timeouts/retries/fallbacks; expand/contract with chunked backfills; managed platforms over clusters; a short written postmortem per incident. (synthesis of the group)

---

## Liz Fong-Jones (The Geek Narrator) — Observability Engineering with Liz Fong-Jones (https://www.youtube.com/watch?v=_kLz18dbJZw)
**Monitoring tells you a known condition was breached; observability lets you answer new questions about a broken system without shipping new code, and SLOs tell you when to go looking.**

- **Mental models / first principles**
  - The word comes from control theory (observability is the dual of controllability: how well you can infer internal state from outputs). Twitter applied it to microservices around 2014–15. For practitioners the useful translation: internal state = what your code is actually doing; external output = telemetry.
  - Test of observability: can you explain a *new* problem (e.g. "why are these 500s happening?") using data you already emit, without pushing printf-debug code or attaching a debugger to production? Counting 200s vs 500s is monitoring; explaining *why* the 500s happen is observability.
  - Monitoring = known unknowns (alert when a known-bad condition fires). Observability = unknown unknowns (reconcile your mental model with what the system really does). You need both.
  - Observability is not a binary property and not "collecting all three pillars like Pokemon". It is an outcome and a socio-technical, continuous practice, like testability: new code and new people keep eroding it unless you keep investing.
  - In a well-tested system, anything that breaks in prod is by definition something you did not foresee; that is why pre-defined dashboards/alerts alone are insufficient.
  - Reliability is not the end goal; preserving user trust and letting developers ship confidently is. An SLO is the agreed level of reliability that keeps trust while still letting you move fast (not space-shuttle change control).

- **Mechanisms**
  - Why now: microservices broke single-binary APM, so distributed tracing came back; systems became business-critical (B2B/SaaS where each request belongs to a paying customer); complexity rose; and columnar stores (e.g. ClickHouse-style) made storing and querying wide structured events cheap enough.
  - OpenTelemetry (CNCF): vendor-neutral standard and libraries for producing traces, logs, metrics. Add the SDK or attach an agent; it auto-discovers inbound/outbound HTTP calls and DB calls and records attributes (prepared SQL statement, user agent, path) and exports in OTLP to any backend, even several at once. Point: instrument once, never be locked into a vendor's proprietary agent.
  - Retrofitting is possible in languages with monkey-patching/runtime injection (auto-instrumentation), but baking it in from day one is far easier.
  - SLI = a per-request (per-event) rule that resolves to in-scope/out-of-scope and good/bad. SLO = percentage of in-scope events that must be good over a window. Example: for an auth service a 403 for bad credentials is *not* a failure; you must define "good" in business terms.
  - Combine latency and availability into one SLI: a request slower than the threshold (e.g. 2000 ms) counts as failed. "Fast failures and slow successes are both failures."
  - Prefer "N% of requests finish under 300 ms" over "P99 < 300 ms". A P99 target says nothing about how bad the worst 1% is (could be 10–20 s); the per-request threshold form bounds it and gives you the specific failing requests to inspect.
  - Aggregate metrics cannot tell you *which* requests were the slow ones. Driving SLIs from structured events/traces lets you evaluate the rule per request, record the bad ones, and open the full trace.

- **Decisions & trade-offs**
  - SLO design starts from critical user journeys (CUJs), ranked by business criticality and frequency. E-commerce: homepage loads, search works, checkout works; "check order status" can wait.
  - Granularity: roughly 3–5 SLOs per service or major functional area. Bucket endpoints by expectation, e.g. a fast read-only bucket (lookups, < 250 ms) and a state-mutating bucket (checkout, update address). A DB platform team may need only 1–2 SLOs.
  - Group by user expectation vs by implementation/ownership: "it depends". If teams own sets of services, set SLOs per team-owned service so someone is accountable; with a central ops centre a whole-system view may fit better.
  - SLOs act as contracts between teams. Example: DB team's SLO allows up to 5 min downtime/month so they can do security patching using the error budget. If your SLO is stricter than a dependency's, you must mitigate: caching, retries, multiple replicas, or degrade gracefully (drop the recommendations panel rather than fail the homepage). Never take a hard dependency on something less reliable than your own target. Five hard dependencies each at 99.99% add up to more downtime than 99.99% for you.
  - First SLO target: use historical performance (the implicit SLO users are used to) as an achievable starting point; tighten later. Distinguish achievable vs aspirational SLOs.
  - Storage choice matters: dumping every span/log to S3 and querying with Athena is technically "having the data" but too slow for live debugging, so it isn't observability in practice.

- **Rules of thumb / checklist**
  - Start somewhere: even one SLO across 100% of incoming HTTP requests, computed from server logs, beats nothing; split later. You might otherwise never notice that 0.1% of users wait 30 s.
  - From day one: at least structured logging, ideally distributed tracing; metrics/time series alone are not enough.
  - Alert on error-budget burn (SLO about to be blown), not on every low-level signal; it is far less noisy. Then use traces/events to bisect from the top down to the cause.
  - Observability belongs in code review: reject PRs that add printf-style logs instead of structured attributes or that add code paths without spans. Honeycomb's PR template asks for an "observability plan" and a "deployment plan", with a link to a query expected to change after deploy.
  - Avoid single-wizard knowledge: if only one senior engineer can drive the tooling, you have a human single point of failure.
  - When people say "we're too busy firefighting to add instrumentation", add at least a little telemetry; it pays back in faster firefighting.

- **Failure modes & gotchas**
  - Misconceptions: (1) observability == monitoring renamed; (2) data types == outcomes (vendors selling "observability in a box" that is just expensive custom metrics + logs); (3) it is a static property of the system, independent of people.
  - Over-complicating SLOs (giant cataloguing project, 100 SLOs, measuring from every mobile client before measuring anything) and setting unachievable targets (six nines; consumer cell service is ~3.5–4 nines). An SLO you always violate and never act on (never freeze releases) is noise.
  - Using ML to sift a sea of low-level alerts (disk 90.1%, CPU 75%) is the wrong foundation; look at real failing traces instead.
  - Utilization/saturation targets get gamed (a Google team burned CPU to hit a 65% utilization target). If you set them, pair with "eliminate waste" and allow undershooting while experimenting. Cost per transaction matters more in lean times, but there's no mature science for utilization SLIs yet.

- **Relevance to Sam**
  - Define CUJs for Trameter: "search returns results", "supplier availability/price check succeeds within X s", "booking confirmed". One SLI per journey that counts slow as bad; compute it from structured logs at first.
  - Supplier APIs (Viator, RateHawk, Coras) are dependencies with weaker reliability than you want: plan caching, retries with backoff, and graceful degradation (show cached prices/"check availability" instead of erroring) rather than inheriting their downtime.
  - As a solo engineer, budget-burn alerts are the only sustainable alerting; low-level alerts will cause fatigue.
  - Use OpenTelemetry so instrumentation survives a change of backend (free tiers now, something else later). Keep the "observability plan" line in your own PR/commit checklist.

## Charity Majors (NDC Conferences, 2018) — Observability: it's not just an ops thing (https://www.youtube.com/watch?v=ACHr-qwHS8Y)
**Your code is your responsibility until it is decommissioned; instrument it with wide, structured, high-cardinality events so you can ask any question of production, and replace the pile of threshold alerts with a handful of symptom alerts.**

- **Mental models / first principles**
  - "Best code is no code; second best is code someone else writes and supports; third best is anything else." Code causes problems, so own fewer lines.
  - Monitoring's model (predict how it will fail, write checks) has barely changed in 20 years while architectures changed many times. It answers known unknowns quickly. Modern pain is unknown unknowns: vague, partly unreliable user reports where the hard part is figuring out what the question even is.
  - Monitoring and observability have partly opposed best practices, hence two words. Monitoring: don't stare at graphs, be alerted, every page must be actionable. Observability: explore to understand, validate hypotheses, decide what to build; not every look needs to be triggered by an alert.
  - "Observability is for software engineers as monitoring is for ops." A system is observable to the degree you can poke at anything that happens and work out why without strace/gdb or new deploys.
  - Production is the "fourth trimester": shipped code is a newborn that cannot walk yet; nurture it. Services need owners, not operators. Her stance: don't promote engineers to senior unless they show they can and want to own their systems in prod.
  - The health of the system is irrelevant; the health of each request/event (and each high-cardinality group of them) is what matters. If an AZ dies and users don't notice, sleep. If dashboards are green but users suffer, care a lot. Distributed systems are never fully "up", so a wall of green dashboards is lying.

- **Mechanisms**
  - Metrics: a single number plus tags; born when storage/RAM was scarce ("how much can we throw away and still understand anything"). They describe system health, not request health, and cannot do high cardinality (user ID, app ID, request ID, build ID) because each extra tag value multiplies series.
  - Cardinality: number of distinct values. In 100M users, user ID / SSN is highest, last name lower, gender low, species lowest. The *high*-cardinality fields are the ones that identify the culprit, so they are the most valuable for debugging.
  - Parse story: ~60,000 mobile apps on a shared platform (hundreds of MongoDB replica sets, user-uploaded queries and JavaScript). Dashboards were green while individual apps were broken; 70–80% of two teams' time went to chasing one-off issues taking hours to days. Facebook's Scuba (in-memory, high-cardinality group-by/slice) cut that to seconds/minutes. Being able to break down by one-in-a-million app ID "and any combination of anything else" was the fix.
  - Canonical wide event: when a request enters a service, start one event; during the request add everything potentially useful: business nouns (customer/app ID, cart ID), normalized query family, raw query, latencies of each downstream call, build ID, feature flags; hundreds of fields is fine. Emit it once when the request exits or errors. Low write rate, arbitrarily wide events. Avoids correlating timestamps across dashboards (and NTP skew).
  - Data reduction must happen somewhere. Metrics/TSDBs discard by aggregating over time (roll-ups you can't un-squish). Event systems discard by sampling, which preserves individual records. Sample smartly: keep all rare/valuable things (every DB delete, every HTTP 500), heavily sample common things (reads, 200s). Parse sampled common traffic heavily; Facebook rates were thousands to one. Store the sample rate on each event and reweight when computing graphs.
  - Schema evolves in three kinds of fields: predictable (HTTP attributes, DB query info, latencies), temporary fields added while developing, and fields added mid-outage that become permanent.
  - Deploys: build ID is a monotonically increasing high-cardinality value; you always have multiple versions in prod. Honeycomb alerts if >2 versions run for >10 minutes during a rolling deploy.

- **Decisions & trade-offs**
  - Measure before building ("observability-driven development"). Example: a customer asked Honeycomb to flatten nested JSON in flight. They first checked (30 s query) how many customers send nested JSON, then added fields to estimate depth/column explosion. Few customers, but a large share of traffic, and some would get hundreds of extra columns, so they made it opt-in behind a feature flag to limit blast radius.
  - Observability can let you write less code: client-vs-server latency percentiles showed median query time was over 50% of the JS poll interval; changing the client interval gave a large perceived speed-up with no new backend code.
  - Feature flags let you ship code dark or to one customer; add the flag name as an event dimension so you can slice by flag combinations.
  - Deploys cause at least half of outages and every deploy is a unique artifact+servers+environment combination you can only partly test. Invest in hardening deploys (guardrails, canaries that auto-promote with confidence) rather than in staging; she dislikes staging environments. Test in production deliberately.
  - Don't write a database unless you are a database company (Honeycomb's custom storage engine was a last resort and has no transactions/replication).
  - Query speed matters: target sub-second p95 for exploratory queries so investigation keeps up with thought.

- **Rules of thumb / checklist**
  - All data should be structured: a structured log vs a string is "grep vs all of computer science". Appoint a structured-logging zealot.
  - Pack every event with context; if the dimension that the broken requests share is not captured, the problem will be invisible.
  - Use consistent field names across services (pick dashes or underscores and stick to it).
  - After every ship, go look at production. Build the muscle memory; if you never look when things are normal you won't know what abnormal looks like. Ship small coherent changes so you can verify each one did what you expected and nothing else.
  - Paging: roughly four signals (request rate, latency, errors, maybe saturation) plus a couple of end-to-end checks that exercise the critical money path. Declare bankruptcy on the rest.
  - Debug like BI: start with a small question, follow the data, bisect from the edge toward the lowest point where the problem appears. You don't start with a list of every place your keys could be.

- **Failure modes & gotchas**
  - Alert fatigue: too many pages burn out people without improving systems; it's why nobody wants on-call. Alerting on the combinatorial explosion of thresholds will make the team quit.
  - Exceptions are an incomplete signal: memory leaks, latency creep and requests that finish in 3 s against a 1 s timeout often throw nothing.
  - Tests and benchmarks only cover what you could imagine; production with real users, networks and packet loss has no substitute.
  - Platform trap: spending engineer time debugging one tenant at a time instead of solving classes of problems.
  - "How to avoid high cardinality" advice comes from metrics vendors; with events you shouldn't avoid it.
  - (note: 2018 talk and she is a vendor, as she says; the "events over metrics" and "kill staging" views are strong opinions. Metrics remain cheap and fine for known system-level signals.)

- **Relevance to Sam**
  - For the booking flow, emit one wide JSON event per booking attempt: supplier, product/hotel ID, search params, each supplier call's latency and status, price before/after, currency, retry count, git SHA/build ID, feature flags, outcome. That alone answers "why did bookings to supplier X fail yesterday for Lagos searches?".
  - For pipelines, one wide event per chunk/batch (rows in/out, dropped, match rate, duration, memory, source file) makes data-quality drift visible.
  - Keep sampling cheap: log every failure and every booking; sample successful searches. Store sample rate in the event.
  - Solo, no on-call team: alert only on symptoms (booking success rate, search latency, pipeline freshness) plus one synthetic end-to-end booking check against a sandbox.

## Niall Murphy, Betsy Beyer, David (Google Cloud SRE), Kristoff (Google SRE Prodcast, Nov 2026) — Ship it! with the Authors of Site Reliability Engineering, 2nd Edition (Part 1) (https://www.youtube.com/watch?v=mZjCCOYz9Rk)
**The first SRE book described how Google runs production; the second edition (late 2026) keeps the principles but drops commoditized tech detail, adds platform engineering, culture and business value, and treats SLO/error-budget policy as a tool to adapt, not dogma.**

(Mostly a book-history podcast; the engineering substance is in the "what changed" parts.)

- **Mental models / first principles**
  - SRE took off because digital systems moved from "the thing in the basement" to business-critical, and companies needed a concrete, sustainable way to run complex systems at scale.
  - Copying Google wholesale failed for many organizations. Successful adopters made SRE their own, fitted to their culture and needs. The problems are "laws of physics" everyone hits; the specific solutions are context-dependent.
  - SRE is a business function, not just engineering: its purpose is to let an organization scale *sublinearly in cost*. If reliability work is seen as an operational cost centre rather than a competitive advantage, it gets cut at the next budget cycle. Make the value case explicitly ("value of reliability" chapter).
  - Culture (blamelessness, management support, people practices) is the foundation SRE needs; 1E assumed Google culture as invisible "water we swam in". You can fail spectacularly at SRE without touching technology.

- **What changed between editions (decisions & trade-offs)**
  - Platform engineering: 10 years ago teams built and ran their own infrastructure; much of that has moved "left and down" into shared platforms. The principles stay; who carries them changed.
  - Removed: detailed tech chapters (load balancing, distributed cron, etc.), which are now commoditized or cheap to produce (including with LLMs). Added: organizational and socio-technical material, case studies from Spotify, Microsoft, Etsy, Disney, Waymo.
  - 1E assumptions that no longer generalize: lots of idle N+1 compute to fail over to; a VM monoculture (which led to Borg/Kubernetes); MTTR as a clean metric. They say MTTR followed a nice bell curve then and is less relevant now (note: consistent with wider industry criticism that MTTR averages are misleading for heavy-tailed incident data).
  - SLOs/error budgets were the 1E's runaway success and its most dogmatic area. Budget-driven freezes worked when agreed in advance (plan when to freeze vs ship, so the decision is pre-made when things go badly). But many organizations felt the rigid policy removed their agency over risk trade-offs. Converged view: with transparency and a clear understanding of their own risk, teams can decide their own budget policy. The invariant value of SLOs is surfacing what users actually experience and care about. 2E's SLO chapter includes a "weaknesses of SLOs" section.
  - 2E is a companion, not a replacement: 1E chapters such as simplicity remain valid; aspiring SREs are told to start with 1E.
  - Other 2E highlights the authors recommend: cultural context of SRE; automation and tooling (progressive A→B→C path, long nuanced treatment of toil); a successor to the Dickerson hierarchy of reliability needs (the old pyramid's prioritization is less useful now); separate chapters on "AI for SRE" and "SRE for AI".

- **On AI in production**
  - Write about AI from first principles that will outlive the week-by-week churn.
  - LLMs hallucinate and get distracted: superhuman in some ways, inadequate in others. In production the difference between 99.9% and 99.5% accuracy matters a lot.
  - Adopt AI in production step by step, building confidence and limiting risk. Ops has always aimed for deterministic outcomes; LLMs don't favour determinism, which changes how you reason about reliability.

- **Rules of thumb**
  - Take principles, not implementations, from big-company playbooks.
  - Agree error-budget consequences before you need them, but keep the freedom to adjust the policy to your risk appetite.
  - Track and reduce toil; automate progressively.

- **Relevance to Sam**
  - As a one-person ops function, use SLOs mainly to make user experience visible (booking success, search latency), and write a simple pre-agreed rule for yourself, e.g. "if booking success drops below target this week, next sprint is reliability work". Adapt it rather than following a Google-style freeze.
  - Frame reliability work to Trameter leadership in business terms (lost bookings, supplier trust, cost per booking) so it isn't cut as overhead.
  - For the LLM article pipeline (Drive Button), treat model output as non-deterministic: measure accuracy as an SLI and roll changes out gradually.

## Edson Yanaga (Red Hat DevNation, 2017) — Update your Database Schema with Zero Downtime Migrations (https://www.youtube.com/watch?v=3mj6Ni7sRN4)
**Code is easy to run in two versions at once; state is hard. Zero-downtime schema change means every migration keeps the schema compatible with both the previous and next app version, ships as many small releases, and never runs a destructive statement until long after it's safe.**

- **Mental models / first principles**
  - The only hard requirement for zero-downtime deploys is a blue-green (or rolling/canary) setup, which means two app versions *will* run against the same database at the same time. So the schema must be backward and forward compatible across adjacent versions.
  - State comes in two kinds: ephemeral (HTTP sessions) and persistent (relational DB). The persistent kind is what makes zero downtime hard.
  - A migration = the ordered set of SQL (and possibly code) statements that moves a schema from version N to N+1. Manual migrations (emailing a SQL script to the DBA) are not allowed: automate them and keep them in version control with the code.
  - The "secret ingredient": never issue a destructive statement (DROP COLUMN, destructive rename) in the same step that stops using the data. The old data stays, so you can always roll back. That also makes this the *safest* way to change schemas, not just the zero-downtime way.

- **Mechanisms**
  - Tools: Flyway and Liquibase (Java world, but used widely); Rails, Django, Alembic etc. have equivalents. Flyway applies versioned SQL files from a migrations folder at app start (e.g. `V2__add_last_name.sql`) and records applied versions in a schema-version table.
  - Baby steps: ideally one SQL statement per migration, smallest possible batch.
  - Long locks = downtime. A five-minute UPDATE over a huge table is an outage. "Shard" the update (his word; really batching): update a slice of rows at a time (thousand or million rows per batch). Rehearse in a test environment to measure lock duration per batch and pick a batch size whose lock time you can tolerate (e.g. ~1 s).
  - Instead of `ALTER TABLE customers RENAME COLUMN a TO b`, do: add new column → backfill in batches → later drop old.
  - Constraints (NOT NULL, foreign keys) block dual-version operation: drop/relax them during the transition (demo removes NOT NULL on the old column before adding the new one) and recreate them in a later release once migration is done.

- **The four scenarios (each step = its own production release)**
  - **Add a column:** (1) ALTER TABLE add column; (2) code computes the value and writes the new column; (3) backfill existing rows in small batches; (4) code reads and writes the new column.
  - **Rename a column (hardest, six releases):** (1) add new column; (2) code reads old, writes both; (3) backfill new from old in small batches; (4) code reads new, writes both; (5) code reads and writes only new; (6) much later, drop old column (after a backup, maybe in a maintenance window, possibly by the DBA).
  - **Change type/format of a column:** identical steps to rename (new column in new type, dual-write, backfill, switch reads, stop writing old, drop later).
  - **Delete a column:** never drop straight away. (1) stop reading but keep writing it (previous version may need it); (2) if all is well, stop writing; (3) later, drop or mark for deletion.
  - Demo detail: dual-write was done in the ORM layer (JPA `@PrePersist`/`@PreUpdate` copying surname to last_name, and setters setting both). Each version ran on a separate port against the same MySQL DB to prove v(N) and v(N+1) both worked; rows written before step 2 had NULL in the new column until the backfill release. Repo tags each step (github.com/.../zero-downtime-migrations).
  - The backfill UPDATE went in as its own versioned Flyway migration, not typed into a DB CLI, so it is tracked and ships with the release that depends on it.

- **Decisions & trade-offs**
  - Cost: many releases for one logical change. Worth it when you deploy often (he saw it mainly in startups/small teams deploying several times a week or day, then increasingly enterprises).
  - Batch size is an empirical choice: measure lock time in rehearsal, don't guess.
  - Dual-write in app code is simple but couples the code to the migration; triggers are another option (not covered).

- **Rules of thumb / checklist**
  - Between every release, query the DB yourself (SELECT) to verify new and old columns are being written correctly and in the right format.
  - Commit every migration, including data backfills, to the repo and run it through the pipeline.
  - Keep the old version running briefly alongside the new; stop it only after you've seen the new one work.
  - Drop/destructive steps: delay until rollback to the previous version is no longer plausible, and take a backup first.

- **Failure modes & gotchas**
  - Big-bang migrations with hundreds of statements; long-running UPDATEs holding locks; renames that break the still-running old version; NOT NULL on a column the new version no longer writes; dropping data you need for rollback.
  - (note: 2017 talk with Java EE/WildFly Swarm tooling, now dated; the expand/contract pattern itself is current. Modern Postgres can add nullable columns, and columns with constant defaults (PG 11+), without table rewrites, but long UPDATEs and some ALTERs still take heavy locks. Use `lock_timeout` and `CREATE INDEX CONCURRENTLY` in Postgres.)

- **Relevance to Sam**
  - For booking/supplier tables in a live product (Tereka, Riselist, Trameter APIs), always expand → migrate → contract, even when solo, because a deploy briefly runs two versions and a rollback needs the old schema.
  - For the multi-million-row location tables, backfill in chunks (e.g. 10k–100k rows per transaction, keyed on primary-key ranges) with a pause between batches; measure on a copy first. Same chunking discipline he already uses for RAM limits.
  - Use Alembic/Django migrations/Prisma-style versioned files rather than ad-hoc SQL in a console; keep backfills as migrations too.

## Dr. Nicole Forsgren (IT Revolution / DevOps Enterprise Summit, 2017) — The Key to High Performance: What the Data Says (https://www.youtube.com/watch?v=RBuPlMTXuFc)
**Four years of State of DevOps data (23,000+ responses) show speed and stability move together, not against each other. Loosely coupled architecture, small batches, lightweight change approval and a generative culture predict both delivery performance and business outcomes.**

- **Mental models / first principles**
  - Software delivery performance = ability to develop and deliver software with both speed and stability. Measured by four metrics (now called DORA metrics):
    - Speed: **deployment frequency** (on demand, when the business wants: a business decision, not a technical constraint) and **lead time for changes** (code commit → running in production; the most predictable part of the pipeline).
    - Stability: **MTTR** (time to restore service) and **change failure rate**.
  - No trade-off: cluster analysis yields three clean groups (high, medium, low) where all four metrics move together. Going fast does not cost stability; stability does not require going slow.
  - "IT doesn't matter" (Nicholas Carr, HBR 2003) was true when tech was bought, put in a closet and ignored. Tech alone only gives parity. Distinctive advantage comes from "tech plus": technology + lean management/process + culture (CALMS: culture, automation, lean, measurement, sharing). Buying "DevOps in a box" fails.
  - Maturity models don't work: the landscape keeps moving (her World of Warcraft level-cap analogy), what counted as "good" in 2014 was out of date by 2015, and reaching "level 5" invites complacency when attention or budget goes away. Use a continuous-improvement/capabilities model instead.
  - Architecture matters, specific technology doesn't. Mainframe vs not, greenfield vs brownfield: no significant correlation with performance. Outsourced software was more common among low performers. What correlated was loose coupling, at team and technical levels (Conway's law).

- **Mechanisms / numbers (2017 report)**
  - High vs low performers: 46x more frequent deploys (multiple per day vs once a week or less); 440x faster lead time (< 1 hour vs > 1 week); 96x faster MTTR (< 1 hour vs days); 1/5 the change failure rate. Failures that do happen have a smaller blast radius and are easier to diagnose because the changes are small.
  - Business outcomes: high performers twice as likely to meet or exceed commercial goals (profitability, productivity, market share) and also non-commercial goals (effectiveness, efficiency, customer satisfaction). 2014 data: 50% higher market-cap growth over three years.
  - Performance is categorized per team; one organization contains teams at different levels. All industries and sizes appear in every group, so "our industry is different" is no excuse.
  - Loose-coupling test (answer yes to all): can your team make large design changes without permission from outside the team? Complete work without fine-grained coordination with others? Deploy and release on demand independently of other services? Do most testing on demand without an integrated test environment? Deploy during normal business hours with negligible downtime?
  - Capabilities that predict delivery performance:
    - Technical: version control for all production artifacts, deployment automation, continuous integration, trunk-based development, test automation, test data management, shift-left security, continuous delivery, loosely coupled architecture, architecture that empowers teams.
    - Process: gather and act on customer feedback, work in small batches, lightweight change approval, team experimentation.
    - Measurement/monitoring: visual management, monitoring to inform business decisions, proactive system-health checks, WIP limits.
    - Culture: Westrum generative culture, climate for learning (learning as investment), collaboration among teams, meaningful work, transformational leadership.
  - Capital One: focusing on just trunk-based development and a streamlined change approval process gave a 20x increase in releases in two months with no increase in incidents.
  - Westrum typology (sociologist studying high-risk domains like aviation and healthcare): pathological (power-oriented) correlates with low, bureaucratic (rule-oriented) with medium, generative (performance-oriented) with high performance. Generative traits map to DevOps practice: failure leads to inquiry (blameless postmortems), bridging between teams is encouraged, novelty is implemented.
  - Transformational leadership (vision, inspirational communication, intellectual stimulation, supportive leadership, personal recognition) is necessary but not sufficient: teams with the least transformational leaders (bottom third) were half as likely to be high performers, but the top third did no better than the median. Good practices also mean less deployment pain, less burnout, and employees 2.2x more likely to recommend their workplace (eNPS).

- **Decisions & trade-offs / where to start**
  - "It depends", but in 2017 data loosely coupled architecture was the biggest contributor to continuous delivery. Common top constraints: architecture + trunk-based development; heavyweight change approval (handoffs); continuous integration (check that you really mean CI: frequent merges to trunk with automated tests, not just "we have a build server").
  - Method: identify your constraints, pick a few (avoid "death by initiative", some companies run 50), eliminate them, re-evaluate because the bottleneck will move in a complex system, repeat.

- **Rules of thumb**
  - Measure to know where you are: start with about five things, including outcome measures (the four delivery metrics) and at least one process and one culture lever.
  - Small batches are the mechanism behind both speed and stability: easier review, easier diagnosis, smaller blast radius.
  - Deploy in business hours with negligible downtime as a design goal.

- **Failure modes & gotchas**
  - Treating tooling purchases as transformation; maturity-model box-ticking; heavyweight external change-approval boards; too many simultaneous initiatives; claiming industry exemption.
  - (note: later DORA work added reliability as an operational measure (2021), replaced MTTR with "failed deployment recovery time" (2023), and since 2024 uses five delivery metrics in two factors, throughput and instability, adding "deployment rework rate"; the "elite" cluster appeared after this talk. The 2026 SRE 2E podcast in this group also argues MTTR is less meaningful now.)

- **Relevance to Sam**
  - Track his own four metrics cheaply from git and deploy logs: deploys per week, commit-to-prod time, % deploys causing a fix/rollback, time to restore. Useful for Trameter reporting and for job-hunt stories.
  - Solo-engineer version of loose coupling: pipelines, the booking service and LLM pipeline should deploy independently and be testable without a full integrated environment (contract tests and recorded supplier responses rather than a shared staging).
  - Trunk-based development + small batches + automated tests + one-command deploy gives the biggest gain; skip heavyweight approval steps.

## Piyush (Tech Tutorials with Piyush, 2026) — SLO vs SLI vs SLA vs Error Budget | Google SRE in Plain English (https://www.youtube.com/watch?v=Akri1BlGp10)
**SLI is what you measure, SLO is the internal target for it, SLA is the contract with penalties if you miss, and the error budget (100% minus SLO) decides whether you ship or slow down.**

(Beginner explainer; several technical slips are corrected inline with "note".)

- **Mental models / first principles**
  - Layered "inside-out" reliability: you can't sign an SLA without an SLO, and you can't set an SLO without measuring an SLI. Measure → target → guarantee.
  - Car analogy: SLI = current speed (60 mph); SLO = the limit you hold yourself to (65); SLA = the speeding ticket you pay if you break the law.
  - Ownership: SLIs by engineering/SRE; SLOs by product + SRE; SLAs by business + legal, with engineering input.
  - One-line test: "What happens if we miss?" Nothing formal → it's an SLO. "We owe the customer money/credits" → it's an SLA.
  - Never target 100%: it's impossible and sets false expectations. Worse, it makes dependents assume your service never fails, so they skip fallbacks; when the real outage comes it cascades. Google's Chubby (distributed lock service) was deliberately taken down when it overshot its SLO each quarter, so dependents had to build resilience.

- **Mechanisms**
  - SLI categories: availability (share of successful requests, e.g. 99.9%), latency (e.g. 95% of requests under 200 ms), error rate (e.g. < 0.1% HTTP 5xx), throughput, durability.
  - Choosing SLIs by service type: user-facing/API/front end → availability, latency, throughput. Storage/data systems → availability, latency, durability.
  - Heuristic: "if your service broke today, what would users be tweeting about?" ("it's down" → availability; "it's slow/timing out" → latency).
  - Averages hide tail pain: if 1% of users see 30 s responses, the mean still looks healthy. Track percentiles (P50, P95, P99). (note: his explanation is muddled. P99 is the latency below which 99% of requests fall, so it describes the slowest 1%; it isn't "comparing the top 1% of users with each other". Liz Fong-Jones in this group recommends the stronger form "X% of requests under T ms" over a P99 target.)
  - An SLO needs the target, the SLI, and a window: e.g. 99% of requests over a rolling 30 days with latency < 200 ms.
  - Error budget = 100% − SLO. At 99.9%, 0.1% of a year ≈ 8.76 hours of allowed unavailability (≈ 43 min per 30 days). It's not downtime you must "use", but room to take risk.
  - Budget left → ship faster, experiment, do riskier changes. Budget spent → freeze releases, slow down, stabilize. Freezes often coincide with holidays/quarter-end.
  - Operating loop: read the SLI → compare with the SLO → check remaining budget → act (ship, freeze, or protect the SLA). Repeat continuously.

- **Decisions & trade-offs**
  - Set the SLA looser than the internal SLO to leave a buffer, so you see trouble internally before it costs money. (note: his EC2 example lists the same 99.99% for both SLO and SLA, which contradicts his own buffer advice; AWS's region-level EC2 SLA is 99.99% with service credits starting at 10%.)
  - (note: he says S3 guarantees "99.119%". In fact S3 Standard is designed for 99.99% availability with a 99.9% SLA, and 11 nines of durability. Also some units said "microseconds" where milliseconds were meant.)
  - (note: he gives disk utilization and CPU < 80% as SLI examples. Those are resource metrics, not user-facing SLIs; better to keep SLIs to things users feel.)

- **Rules of thumb**
  - Measure before you target; target before you guarantee.
  - Don't promise external SLAs you aren't already measuring.
  - Pick a rolling window (e.g. 30 days) and state it in every SLO.
  - Spend error budget deliberately on velocity; when it's gone, prioritize reliability.

- **Relevance to Sam**
  - Trameter's supplier contracts (Viator, RateHawk etc.) have their own SLAs. Read them: your SLO can't be tighter than what a supplier guarantees unless you add caching/fallbacks.
  - For Tereka/Riselist with no paying SLA, just run SLOs (no legal exposure) and make sure any future customer SLA is looser than the internal SLO.
  - Quick budget math to keep handy: 99% = 7.2 h/30 days; 99.5% = 3.6 h; 99.9% = 43 min; 99.99% = 4.3 min. A solo engineer without on-call can realistically defend 99.5–99.9%, not 99.99%.

## Better Stack — 12 Logging BEST Practices in 12 minutes (https://www.youtube.com/watch?v=I2mWnh66Bkg)
**Log with a purpose, in structured form, with full context, one canonical line per request. Centralize, sample, set retention, secure and redact. Use metrics (not log-grepping) to know when something is wrong.**

- **Mental models / first principles**
  - Logs are the system's black-box recorder: detailed enough to replay what happened and why, not just that something happened.
  - Logging is a strategy, not scattered print statements. Before writing log lines, ask: what are the app's main goals, which critical operations need watching, which KPIs matter? An error log's job is to give enough context to fix the problem.
  - "If your logs aren't structured, you're writing very expensive text files."
  - Division of labour: metrics tell you how often and that there is a problem (trends, alerting); logs tell you what happened (debugging). Don't answer "is my service healthy right now?" by grepping and counting log lines.

- **The 12 practices (mechanisms)**
  1. **Have objectives.** It's fine to over-log at first and trim; adding missing info once in prod is harder than removing noise. Review the strategy periodically for useful vs noise.
  2. **Log levels.** INFO = business-as-usual events worth tracking (user completed checkout, order #). WARN = something off but not broken (payment processing slower than usual). ERROR = real failure (DB connection failed, payment failed). FATAL = app/part of stack died (out of memory, shutting down). Production defaults to INFO.
  3. **Adjustable verbosity.** Build a way to raise verbosity temporarily (e.g. to DEBUG) while investigating, and lower it after, without redeploying.
  4. **Structured logging.** Each piece of information is its own field (JSON), so you can filter (all timeouts), aggregate (errors last Tuesday). Use a logging framework; for sources that can't emit structure, transform with a pipeline tool like Vector.
  5. **Rich context.** Bad: "something went wrong". Good: who/what/where/why, e.g. failed login with user ID, location, device, attempt count. Always include request ID (to trace across services), user ID where appropriate, system state (DB/cache status), full error context (stack trace).
  6. **Sampling.** At hundreds of GB–TB/day, store a representative sample. E.g. auth logs at a 20% rate keep 2 of 10 identical events. Be selective: keep all errors, sample successes; sample hot endpoints harder; keep full logs on critical paths. Claimed savings 80%+. Do it early. OpenTelemetry has built-in samplers.
  7. **Canonical log lines.** Instead of "clicked login… checking credentials… success", emit one line at the end of each request with everything: action, user, outcome/error, total duration, time spent in DB. A TL;DR per request. Distributed tracing (OpenTelemetry spans linked into one trace) is the stronger version across services.
  8. **Centralize/aggregate.** One searchable place for all services so you can correlate (payment service slow → cart times out → front end errors) without SSHing into 20 servers. Start early even with two services.
  9. **Retention policy.** Hot storage for recent logs, then cheaper cold storage, then delete. Vary by type: e.g. errors ~90 days, debug ~7 days, security audit logs ~1 year (or whatever regulation requires). Set it before the first big cloud bill.
  10. **Secure logs.** Logs hold user IDs, IPs, queries, auth attempts, internal error details. Encrypt in transit and at rest; role-based access (juniors see app logs, seniors system logs, security everything); audit who accessed what. (note: he calls this "security by obscurity", which is the wrong term; the point is least privilege and data minimization.)
  11. **Never log sensitive data.** Twitter (2018) had to ask all users to change passwords after plaintext passwords were logged; GitHub had a similar incident. Make types safe by default, e.g. Go `slog` `LogValuer` so logging a whole user object only emits the ID. Add a redaction stage in the pipeline (OpenTelemetry Collector) for card numbers, national ID numbers, API keys.
  12. **Mind performance.** Benchmark: Go server ~192k req/s with no logging; Logrus cost ~20% throughput, `slog` ~3%. Use efficient libraries, sample hot paths, write logs to a separate disk/partition from the app, load-test to find logging bottlenecks.
  - Plus: **don't use logs for everything**; alert on metrics, debug with logs.

- **Decisions & trade-offs**
  - Over-log early vs cost: start verbose, trim with review; enforce sampling and retention to control cost.
  - Canonical lines vs traces: canonical lines are simple and enough for one service; traces are better once requests cross services.
  - Fast logging libs (structured, zero/low-allocation) vs convenience libs: measurable throughput difference.

- **Failure modes & gotchas**
  - Logs without context at 3 a.m.; logs scattered across machines; uncontrolled volume and bills; secrets/PII in logs; logging overhead in hot loops; trying to derive health from log greps.

- **Relevance to Sam**
  - Python: use `structlog` or stdlib `logging` with a JSON formatter; bind `request_id`/`booking_id`/`supplier`/`batch_id` context once and have it on every line. Emit one canonical line per booking attempt and per pipeline chunk.
  - Redact supplier API keys, passenger names, passport numbers, emails and card data before logging (high stakes for travel bookings and for Tereka's financial data). Add a redaction filter as a safety net.
  - On a 16 GB laptop processing millions of rows, never log per row at INFO; log per chunk with counts and sample per-row debug, or performance and disk will suffer.
  - Cheap centralization for a small budget: a free-tier log service or a single Loki/Grafana or SQLite/DuckDB sink; set retention up front.

## Clément Mihailescu (2020) — Blameless Postmortem Culture In Software Engineering (https://www.youtube.com/watch?v=Fv0nwb1Qn6A)
**When one line of code takes the site down, blaming its author is inaccurate (reviewers, missing tests and the deploy process also let it through) and useless (it gives no action and makes everyone afraid). Run a written, blameless postmortem that fixes the system.**

- **Mental models / first principles**
  - Objectively, blame is spread: the author wrote the bug, but reviewers approved it (more senior reviewers arguably carry more), there were no integration tests that would catch a blank site, no manual check before shipping, a deploy process that allowed it. Config changes, a common cause of Google outages, are the same: why could one person push that config without tests or guardrails?
  - Subjectively, blame produces nothing actionable ("write less buggy code" is not a fix) and creates fear: the blamed person disengages or leaves, and everyone else worries they're next, so a toxic team hides problems.
  - Reframe: if an intern can take down production, the process is at fault, not the intern. Folklore examples: a Google intern who briefly took down search (said to have been promoted rather than fired); a Facebook intern whose search-bar placeholder change reached production for billions of users.

- **Mechanism: the postmortem**
  - Formality scales with severity and team: very formal for SRE-level incidents; informal team review for small but worrying issues. Same idea either way: gather the team, analyze in hindsight, write it down so others (and future you) can learn.
  - Questions the document should answer:
    1. What was the issue (impact, description)?
    2. What caused it (root cause, e.g. the specific buggy line)?
    3. How could we have caught it (code review, automated tests, manual testing)?
    4. Why didn't we (no tests, poor coverage, tests existed but the test service was down so it slipped through)?
    5. Where did we get lucky (was it caught only because someone happened to look at midnight, or a customer complained, or it hit the news)? Luck = an unreliable detection path to fix.
    6. What will we do to prevent it in the future (action items)?

- **Rules of thumb**
  - Ask "how did the system allow this?" not "who did this?".
  - Record where detection relied on luck; turn each into a monitoring or test action.
  - Keep postmortems written and shareable so learning spreads beyond the team.

- **Failure modes & gotchas**
  - Blame → fear → people hide mistakes → fewer signals and slower learning.
  - (note: "root cause" singular is a simplification. Modern resilience thinking (Dekker, Cook, Allspaw) prefers multiple contributing factors, and the video's own analysis shows several. Track action items to completion or postmortems become ritual. Short, introductory video, partly a sponsor plug.)

- **Relevance to Sam**
  - Even solo, write a short blameless postmortem for every customer-visible incident (failed bookings, bad pipeline output shipped, wrong prices): impact, timeline, contributing factors, where you got lucky, 1–3 action items. Keep them in the repo. It turns one-off firefights into fixes and gives evidence for job interviews.
  - "Where did we get lucky" is especially useful for supplier-API failures noticed only because a user complained: each one is a missing symptom alert or synthetic check.
  - Connects to Forsgren's finding (same group): generative culture where "failure leads to inquiry" predicts high delivery performance.

## Will (DevOps For Developers, 2021) — When to *not* use Kubernetes in DevOps (https://www.youtube.com/watch?v=6DQBnq8iRoA)
**Use Kubernetes only when your application genuinely needs what it offers (many intercommunicating services that must scale) and you have people to run it; otherwise a PaaS, serverless or managed container service gives you scaling without the operational burden.**

- **Mental models / first principles**
  - Kubernetes' easy scaling and deploys sit on a large, complex substrate you must run: nodes, cluster DNS, volume mounts, secrets, dynamically provisioned storage, RBAC. Neglect any of it and you go "from outage to outage as the cluster collapses in on itself".
  - The application must be "worthy" of Kubernetes' features: service discovery, networking and scaling across many microservices.

- **Decision criteria**
  1. Running on bare metal / your own data centre? You need sysadmins, network engineers and IT staff (usually already there if you own a data centre). Expect a learning curve.
  2. No ops staff but still want Kubernetes? Use managed Kubernetes (GKE, EKS, AKS) so the provider runs the control plane and infrastructure, and you only deploy apps.
  3. Typical web app (static site or SPA on a CDN + one backend API + a database + maybe a queue/cache like RabbitMQ or ElastiCache) → not a good fit; too much overhead. Use Heroku, DigitalOcean (App Platform), AWS Lambda, or AWS Fargate.
  - He has scaled significant apps on Fargate and sees it as the sweet spot between scalability/deployability and admin overhead. (note: Fargate is a serverless compute engine for ECS or EKS. It isn't itself "managed Kubernetes"; with ECS there is no Kubernetes at all. The point stands: you don't manage nodes.)

- **Rules of thumb**
  - Choose Kubernetes because its features meet an actual requirement, not because of hype.
  - Count the ongoing cost: managing, monitoring and maintaining a cluster is a big job; don't take it on alone without a forcing requirement.
  - Prefer the highest-level managed platform that meets your needs; move down the stack only when you hit a concrete limit.

- **Failure modes & gotchas**
  - Self-hosted clusters without dedicated staff; adopting Kubernetes for a monolith + DB; underestimating secrets/storage/RBAC/DNS work. (note: short video with a book plug; no numbers. Add from general practice: Kubernetes also adds upgrade churn (minor versions roughly every 4 months, with about a year of support) and a monitoring stack of its own.)

- **Relevance to Sam**
  - Solo, small budget, no on-call: Kubernetes is almost certainly wrong for Trameter's pipelines, booking API, Tereka or Riselist. Use a PaaS (Render, Fly.io, Railway, Heroku, DigitalOcean App Platform), Cloud Run/Fargate for containers, or Lambda/Cloud Functions plus scheduled jobs for pipelines. A single VM with Docker Compose and systemd timers is also fine for batch work.
  - Revisit only if Trameter grows to many independently deployed services with a team to run them, and even then start with managed Kubernetes.
