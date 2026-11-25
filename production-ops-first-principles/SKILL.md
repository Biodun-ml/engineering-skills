---
name: production-ops-first-principles
description: Production operations playbook from 17 talks (Charity Majors, Liz Fong-Jones, Richard Cook, Dave Farley, Nicole Forsgren, Google SRE authors, Edson Yanaga). Use when shipping, deploying, alerting, setting SLOs, logging, migrating schemas, backing up, or handling incidents and postmortems.
---

# Production operations from first principles

This skill turns 17 talks (17 note sections) into working judgment for running software in
production as a solo or small-team engineer: SLOs and alerts, observability and logging,
deployment pipelines, schema migrations, backups and disaster recovery, incidents and
postmortems, and platform choice. This file holds the procedures, the principles the teachers
keep coming back to, and checklists. `references/` (when present) holds detailed notes for every
video, each credited to its teacher.

Use it whenever code is about to meet real users or real data, even if the user doesn't say
"ops" or "SRE": deploying a FastAPI service, scheduling a pipeline, adding a column to a live
table, deciding what to alert on, choosing between Kubernetes and a PaaS, "the booking failed
for one customer, why?", or writing up an outage.

**How to use it**
1. Follow the matching procedure in section 1 (ship a service, define SLOs, migrate a schema).
2. Run the relevant checklists in section 4 against the design, PR or runbook.
3. If a `references/` folder sits next to this file, use it for depth. `references/INDEX.md`
   lists every video with its line number, or run `grep -n -i "keyword" references/*.md`. Each
   section ends with a "Relevance to Sam" or "For Sam" note: Sam runs supplier booking flows
   (Viator, RateHawk, Coras), GeoNames/GTFS pipelines over millions of rows on a 16 GB laptop,
   the Drive Button LLM pipeline, and small products (Tereka, Riselist) alone, on a small budget,
   with no on-call team. Use those parts only in that context. Lines marked "(note: …)" and those
   Sam notes are helper additions, not the speaker's words. If `references/` is missing, the full
   notes live in the user's private GitHub repo `Biodun-ml/engineering-skills` (folder
   `production-ops-first-principles/references/`); clone it when you need depth.
4. Explain the *why*, and credit the teacher when you use a specific pattern.
5. Vendor names, free tiers and prices date quickly. Check current ones before recommending a
   specific platform or tool.

---

## 1. Procedures

### 1a. Ship a new service to production
1. **Pick the highest-level managed platform that fits.** A web app, an API, a database and maybe
   a queue do not need Kubernetes. Use a PaaS, serverless or a managed container service, and
   move down the stack only when you hit a concrete limit. *(Will — DevOps For Developers)*
2. **Make the pipeline the only route to production.** Everything goes through version control
   and the same tests: code, config, infra, dependency bumps, hotfixes. Nobody edits the server
   or prod config by hand. *(Farley — 3 Reasons)*
3. **Make it deterministic.** Pin dependencies (lockfile), language version and container image,
   define the environment as code, isolate tests. A flaky test is a pipeline bug. *(Farley)*
4. **Commit stage under 5 minutes** giving ~80% confidence; acceptance stage (end-to-end, security,
   performance) under 1 hour; a definitive release verdict at least daily. Run slow independent
   suites in parallel, not as extra serial gates. *(Farley — 3 Reasons; Fintech pipeline)*
5. **Fake what's hard to test at a seam.** "Too hard to automate" is a design smell. Test supplier
   adapters against recorded responses or a sandbox, the way TransFICC fakes trading venues and
   the ATM example fakes card hardware. *(Farley)*
6. **Instrument while you write it.** Set `service.name` and `service.version`, auto-instrument
   HTTP and DB clients with OpenTelemetry, and emit one wide structured event per unit of work.
   Retrofitting loses intent. *(Majors; Better Stack — OTel; Fong-Jones)*
7. **Write the observability plan and deployment plan into the PR**: which query should change
   after deploy, and how you'll roll back. *(Fong-Jones — Honeycomb PR template)*
8. **Define one SLO and its alert before launch** (1b). Alert on absent data too. *(Stack to Scale)*
9. **Ship small and dark.** Feature flag, canary to ~10%, add the flag and build ID as event fields,
   ship within 15–20 minutes of writing while context is fresh. *(Majors)*
10. **Go and look at production after every deploy.** Did what you expected happen, and nothing
    else? If you never look when it's normal, you won't recognise abnormal. *(Majors — NDC 2018)*
11. **Set backups, RPO/RTO and a restore test before real data arrives** (1d). *(ByteMonk)*
12. **List the time bombs:** key, token and certificate expiries, free-tier limits, contract dates,
    disk-fill timelines. Put them in a calendar. *(Cook — infusion pumps)*

### 1b. Define SLOs for a booking flow
1. **Start from critical user journeys**, ranked by business value and frequency: "search returns
   results", "availability/price check succeeds", "booking confirmed". Order status can wait.
   *(Fong-Jones; Stack to Scale)*
2. **Write each SLI as good / eligible events, from the user's view,** with explicit scope,
   population and window. A correct business rejection (sold out, price changed, a 403 for bad
   credentials) is good or out of scope; a 500 or timeout is bad. Don't exclude hard requests to
   look good. *(Stack to Scale; Fong-Jones)*
   - Booking SLI: (confirmed + correct supplier rejections) / eligible booking attempts.
   - Fold latency in: a slow success is a failure. Use "X% of requests under T ms", not
     "P99 < T", so the bad tail is bounded and you get the actual failing requests.
     *(Fong-Jones)*
3. **Set the first target from history** (what users are already used to), then tighten.
   Never 100%. About 3–5 SLOs per service; start with one. *(Fong-Jones; Piyush)*
4. **Compute the error budget on a stated denominator.** Request-based for spiky traffic:
   10M eligible requests at 99.9% = 10,000 bad allowed. Time-based is different: 0.1% of 30 days
   = 43.2 minutes, and the two are not interchangeable. *(Stack to Scale)*
   Quick table (30-day, time-based): 99% = 7.2 h; 99.5% = 3.6 h; 99.9% = 43 min; 99.99% = 4.3 min.
5. **Check dependencies.** Your SLO can't beat a hard dependency's reliability. If a supplier's
   SLA is weaker than your target, add caching, retries with backoff, replicas or graceful
   degradation ("check availability" instead of an error). *(Fong-Jones; Piyush)*
6. **Alert on burn rate with multiple windows** (short window catches fast burns, long window
   confirms it's sustained). Page only on fast burn; send slow burn to a daily digest. Fit
   thresholds to your traffic, don't copy examples blindly. *(Stack to Scale; Fong-Jones)*
7. **Write a short error-budget policy in advance** (owner, review cadence, what happens on
   sustained burn, e.g. "next week is reliability work"). Treat it as a tool you can adapt, not
   dogma. *(Stack to Scale; SRE 2E authors)*
8. **Keep hard guardrails outside the SLO** for data corruption, wrong prices, payment errors and
   security incidents; a good percentage doesn't excuse them. *(Stack to Scale)*
9. **SLA, if any, looser than the SLO.** Ask "what happens if we miss?": nothing formal = SLO,
   credits owed = SLA. Don't promise what you don't measure. *(Piyush; Stack to Scale)*

### 1c. Run a schema migration safely (expand/contract)
Two app versions share the database during any rolling, blue-green or canary deploy, so every
step must work with both the previous and next version. *(Yanaga)*
1. **Versioned migration files only** (Alembic, Django, Flyway, Prisma), in the repo, through the
   pipeline. Backfills are migrations too, never typed into a console. *(Yanaga)*
2. **Baby steps, each its own release.** Ideally one statement per migration.
3. **Rename or change type (six releases):** add new column → code reads old, writes both →
   backfill new from old in batches → read new, write both → read and write only new → much
   later, after a backup, drop the old column. *(Yanaga)*
4. **Add a column:** add (nullable) → code writes it → batched backfill → code reads it.
5. **Delete a column:** stop reading, keep writing → stop writing → later drop.
6. **Relax constraints during the transition** (NOT NULL, FKs on the old column), recreate them
   once migration is done. *(Yanaga)*
7. **Batch every big UPDATE.** A 5-minute UPDATE is an outage. Rehearse on a copy, measure lock
   time per batch, pick a size you can tolerate (~1 s). For Sam's multi-million-row tables,
   key batches on primary-key ranges. *(Yanaga)*
8. **Verify between releases** with a SELECT that old and new columns are written correctly.
9. **Never ship a destructive statement in the same release that stops using the data.** Old
   data kept = rollback always possible. That makes this the safest way, not just the
   zero-downtime way. *(Yanaga)*
10. **Postgres specifics:** nullable columns and (PG 11+) constant defaults add without a table
    rewrite; set `lock_timeout`; build indexes with `CREATE INDEX CONCURRENTLY`. *(Yanaga notes)*

### 1d. Choose backups and disaster recovery
1. **Set RTO (how long you can be down) and RPO (how much data, in time, you can lose) per
   asset.** Tighter numbers cost more. *(ByteMonk)*
2. **Pick the cheapest tier that meets them:** backup & restore (hours) → pilot light → warm
   standby → hot/active-active (near-zero, ~double cost). Almost everything Sam runs is backup &
   restore. *(ByteMonk)*
3. **Isolate backups** (different account/provider, versioned or immutable) so a bad script or
   ransomware can't delete prod and backups together. *(ByteMonk notes)*
4. **Restore on a schedule** (quarterly and after big infra changes) to a fresh instance, and
   time it against RTO. A backup never restored is not a backup; untested failovers usually fail
   the first time they're needed. *(ByteMonk; Cook)*
5. **Plan failback** as carefully as failover. Keep a human in the loop for database promotion;
   automatic failover on false positives causes flapping and split-brain. *(ByteMonk notes)*

---

## 2. Principles that recur across the teachers

### Observability and monitoring
- **Observability = answering new questions about any state without shipping new code.**
  Monitoring covers known unknowns with predefined checks; production fails as a long tail of
  unknown unknowns. You need both. *(Majors — Iterate 2018; Fong-Jones)*
- **Emit one wide, structured event per unit of work** (request, booking attempt, job, chunk,
  LLM call) with every useful field: IDs, business nouns, downstream latencies, retries, build
  ID, feature flags. Hundreds of fields is fine. Emit once at exit or error. *(Majors — NDC 2018,
  Pragmatic Engineer)*
- **High-cardinality fields are the most valuable debugging data.** Booking ID, supplier, user
  ID identify the culprit. Metric stores price per tag combination (a `hostname` or IP tag can
  multiply a bill 10–100x), so never put IDs in metric tags; put them in events. *(Majors)*
- **System health is nearly irrelevant; request health is what matters.** Aggregates hide pockets
  of total failure: one supplier, one customer, one region 100% down inside a 99.9% average.
  Break down by any dimension and compare against baseline. *(Majors — Iterate 2018)*
- **Sample, don't aggregate.** Aggregation destroys data forever. Keep 100% of rare or high-stakes
  events (errors, deletes, bookings, cancellations), heavily sample common successes, store the
  sample rate on each event and reweight. Never build billing off operational logs. *(Majors)*
- **Use OpenTelemetry for vendor-neutral instrumentation:** API (safe no-op without an SDK),
  SDK, instrumentation libraries, semantic conventions, OTLP, and a Collector (receivers →
  processors for batch/sample/filter/redact → exporters). Switching backend becomes a config
  change. Initialise before importing instrumented modules. *(Better Stack — OTel; Fong-Jones)*
- **Storage you can't query interactively isn't observability.** Spans dumped to S3 and queried
  with Athena are too slow for live debugging. *(Fong-Jones)*
- **Every dashboard is an artifact of a past failure.** Use SLOs as the entry point, then slice
  events interactively ("how is the bad set different?"). The best debuggers become the most
  curious, not the longest-tenured. *(Majors)*
- **Budget heuristic:** observability ~15–20% of cloud spend; buy rather than build; never write
  your own database. *(Majors — Pragmatic Engineer)*

### Alerting
- **Page on symptoms, not causes:** SLO burn on a few signals (rate, errors, latency, maybe
  saturation) plus one or two end-to-end checks on the money path. Declare bankruptcy on
  CPU/disk threshold pages. Every page must be actionable. *(Majors; Fong-Jones)*
- **Missing telemetry looks like perfect availability.** Alert when expected events stop
  arriving. *(Stack to Scale)*
- **Exceptions are an incomplete signal.** Leaks, latency creep and requests that finish after
  the caller's timeout often throw nothing. *(Majors — NDC 2018)*

### Logging
- **Structure everything.** Unstructured logs are "very expensive text files"; structured vs
  string is "grep vs all of computer science". *(Better Stack — Logging; Majors)*
- **One canonical log line per request or chunk** with action, IDs, outcome, duration, time in DB.
  Traces are the stronger form once requests cross services. *(Better Stack — Logging)*
- **Levels mean something:** INFO business events, WARN degraded, ERROR real failure, FATAL dying.
  Prod defaults to INFO; build a way to raise verbosity without redeploying. *(Better Stack)*
- **Logs are a liability.** Never log secrets or PII (Twitter 2018 logged plaintext passwords).
  Make types redact by default, add a redaction stage in the Collector, encrypt, restrict access.
  *(Better Stack)*
- **Set retention before the first big bill:** e.g. debug ~7 days, errors ~90 days, audit ~1 year.
  *(Better Stack)*
- **Logging costs throughput:** in one Go benchmark Logrus cost ~20%, `slog` ~3%. Never log per
  row in hot loops. *(Better Stack)*

### Delivery
- **The pipeline is definitive for release.** Green means ship with no further manual work. If
  you wouldn't ship on green, add the missing check. Hotfixes go through the same pipeline, so
  keep it fast. *(Farley — 3 Reasons)*
- **Scope each pipeline to an independently deployable unit.** Coupled services tested only
  together are a distributed monolith, slower than a monolith. *(Farley)*
- **Speed and stability move together.** In 2017 State of DevOps data, high performers deployed
  46x more often, with 440x faster lead time, 96x faster restore and 1/5 the change failure
  rate. Small batches are the mechanism. *(Forsgren)*
- **Architecture beats technology.** Loose coupling (change, test and deploy on demand without
  coordinating, in business hours) predicted performance; mainframe vs cloud didn't. Fix the top
  constraint, re-evaluate, repeat; avoid "death by initiative". *(Forsgren)*
- **Deploys cause at least half of outages,** so harden deploys (flags, canaries, version-skew
  alerts such as Honeycomb's ">2 versions for >10 min") rather than polishing staging.
  *(Majors — NDC 2018)*

### How complex systems fail, and learning from it
- **Systems as found are not systems as imagined.** Diagrams and fault trees are static; real
  systems are dynamic and always being changed. Success comes from people monitoring,
  responding, adapting and learning. *(Cook)*
- **Design for the operator:** reveal the controls, show the "lift points", support mental
  simulation, open the black boxes, no "don't touch" zones. *(Cook)*
- **Time bombs are built from individually reasonable choices,** and same-day fleet-wide rollouts
  synchronise failure. Stagger rollouts; exercise recovery paths on purpose (Honeycomb kills its
  oldest Kafka node every Monday with spare error budget). *(Cook; Majors)*
- **Blameless postmortems fix systems, not people.** Ask "how did the system allow this?", record
  where you got lucky, turn each into an action. "Root cause" is really several contributing
  factors. *(Mihailescu; notes citing Cook/Dekker)*
- **Generative culture predicts performance:** failure leads to inquiry. *(Forsgren — Westrum)*
- **Reliability is a business function.** Argue for it in lost bookings and user trust, or it gets
  cut as overhead. Copy principles, not Google's implementations. *(SRE 2E authors)*
- **AI in production is non-deterministic.** 99.9% vs 99.5% accuracy matters; adopt step by step.
  If you can compute an answer, don't let AI guess it. *(SRE 2E authors; Majors)*

---

## 3. Where the teachers disagree, and how to resolve it

| Tension | Resolution |
|---|---|
| Majors: events over metrics, kill staging. Better Stack/OTel: three signals; alert on metrics. | Debug from wide events; keep cheap metrics for known system-level trends and long ranges (Majors herself targets ~80% structured / 20% metrics). Keep a thin staging or local full-stack run for smoke tests, but spend most effort on production guardrails. |
| Rigid error-budget freezes (Piyush, SRE 1E) vs "policy is a tool" (SRE 2E authors). | Agree consequences in advance, then adapt to your risk. Solo: "fast burn → stop features until fixed", not a calendar freeze. |
| P99 targets (Piyush) vs "X% under T ms" (Fong-Jones). | Use the threshold form; it bounds the tail and names the failing requests. |
| MTTR as a key metric (Forsgren 2017) vs "MTTR is less meaningful" (SRE 2E). | Track restore time per incident but don't trust averages over heavy-tailed data. DORA itself replaced MTTR with failed deployment recovery time in 2023. |
| Automatic failover "with no disruption" (ByteMonk) vs reality. | Async replication means RPO > 0 and DNS TTLs cause errors at cutover; keep a human in the loop for DB promotion. |
| Single "root cause" (Mihailescu) vs contributing factors (Cook). | Write contributing factors; the video's own example shows several. |
| Disk/CPU as SLIs (Piyush) vs user-facing SLIs only (Fong-Jones, Majors). | Keep SLIs to what users feel; resource metrics are for capacity, not paging. |

---

## 4. Checklists

**Production readiness (before first real user or real data)**
- [ ] Deploys only via pipeline; deps, runtime and image pinned; no hand edits to prod.
- [ ] Commit stage < 5 min; adapters tested against recorded supplier responses.
- [ ] `service.name`, `service.version`/git SHA on every event; one wide event per unit of work.
- [ ] Structured JSON logs, request/booking/chunk IDs bound once; secrets and PII redacted.
- [ ] Log retention set; telemetry sampled with rate recorded; errors and bookings kept 100%.
- [ ] One SLO on the critical journey, burn-rate alert, absent-data alert, one synthetic check.
- [ ] Timeouts on every outbound call; retries with backoff only for idempotent operations.
- [ ] Feature flags for risky paths; rollback is one command and has been tried.
- [ ] Migrations versioned, expand/contract, batched backfills.
- [ ] Backups automated, isolated, restore tested and timed; RPO/RTO written down.
- [ ] Time-bomb calendar: keys, tokens, certificates, domains, free-tier limits, contracts.
- [ ] Runbook with "lift points": pause a supplier, replay a booking, rerun a chunk, rotate a key.
- [ ] Managed platform chosen; no Kubernetes without a forcing requirement and a team.

**Incident response for one person** *(synthesised from Majors, Cook and Mihailescu; the status-update
and scratch-log items are general practice, not from the talks)*
- [ ] Confirm user impact from the SLO/events, not from CPU graphs. Who, which supplier, since when?
- [ ] Mitigate before diagnosing: roll back the last deploy, flip the flag off, pause the failing
      supplier, serve cached results. Deploys cause most outages, so check the last one first.
- [ ] Post a one-line status to affected users/stakeholders; update on a fixed interval.
- [ ] Bisect with events: select the bad set, compare every dimension to baseline.
- [ ] Keep a timestamped scratch log of what you saw and did (it becomes the timeline).
- [ ] Fix forward only through the pipeline, never by hand-editing prod.
- [ ] Verify recovery from the SLI, then check for data damage (duplicate or lost bookings).
- [ ] While it's fresh, write the postmortem (below).

**Blameless postmortem** *(Mihailescu, plus Cook's framing)*
- [ ] Impact (users, bookings, money, duration) and timeline.
- [ ] Contributing factors, including what made the actions sensible at the time.
- [ ] How could we have caught it? Why didn't we?
- [ ] Where did we get lucky? (a user complaint, someone happened to look) Each is a missing alert.
- [ ] 1–3 action items with owners and dates, tracked to done; stored in the repo.

**Backup and restore**
- [ ] RPO/RTO per asset: booking/payment records, master location dataset, LLM outputs, configs.
- [ ] Managed Postgres with point-in-time recovery for transactional data.
- [ ] Versioned Parquet snapshots in object storage for pipeline outputs; sources reproducible.
- [ ] Copies in a separate account/provider, versioned or immutable.
- [ ] Secrets and infra config recoverable from the repo/secret store, not only from a server.
- [ ] Quarterly restore drill to a fresh instance, timed, with a row-count/checksum check.
- [ ] Backup job itself monitored by a heartbeat; failed backups page.

**Logging and telemetry review**
- [ ] No string-concatenated log lines; consistent field names across services.
- [ ] One canonical line per request/chunk; no per-row INFO logs in million-row jobs.
- [ ] Flag names and build ID on events; sample rate on events.
- [ ] Billing and reconciliation data kept apart from operational logs.

---

## 5. Failure modes to hunt for in reviews

- Paging on CPU, memory or disk thresholds; dozens of alerts nobody acts on.
- An aggregate success rate with no per-supplier/per-customer breakdown.
- IDs as metric tags; unstructured log strings; logs scattered on machines.
- SLO of 99.99% or 100% for a service with one engineer and supplier dependencies at 99.9%.
- Counting "sold out" as an error, or excluding timeouts to look good.
- No alert on missing data, so a dead exporter reads as 100% uptime.
- A manual "emergency" deploy path, SSH config edits, flaky tests tolerated.
- `ALTER TABLE ... RENAME COLUMN` or DROP in the same release as the code change.
- One giant UPDATE backfill holding locks; backfills run by hand from a console.
- Backups in the same account as prod; never restored; backup job unmonitored.
- Fleet-wide same-day rollout; secrets expiring with nobody tracking the date.
- A postmortem that names a person, or action items that are never done.
- Kubernetes for a monolith plus a database run by one person.
- Microservices that can only be tested together (distributed monolith).

---

## 6. Applying this to Sam's work

- **Supplier booking flows:** one wide event per booking attempt (supplier, endpoint, product ID,
  search params, each call's latency/status, price before/after, currency, retry count, build
  SHA, flags, outcome). Booking SLI counts correct supplier rejections as good. Page only on fast
  burn of booking success and on a synthetic sandbox booking. Every supplier call gets a timeout,
  bounded retries with backoff, and a degraded path (cached prices, "check availability").
  Read each supplier's SLA: you can't promise more than they deliver without caching.
- **GeoNames/GTFS pipelines:** SLIs are freshness (% of runs done by deadline) and quality (rows
  in/out, match rate). One event per chunk, never per row. Backup tier is "re-run from source
  plus last good Parquet snapshot". Heartbeat-monitor scheduled runs so a silent skip alerts.
- **Drive Button LLM pipeline:** trace across gate → extraction → trip building with model,
  tokens, cost and validity per article. SLI: % of articles with valid output. Roll prompt and
  model changes out gradually behind a flag; treat output as non-deterministic.
- **Tereka, Riselist:** a PaaS or Cloud Run, managed Postgres with PITR, Sentry-style error
  tracking, one SLO, expand/contract migrations. Realistic target 99.5–99.9%, not 99.99%.
- **Track DORA-style numbers from git and deploy logs** (deploys/week, commit-to-prod time,
  change fail rate, recovery time): useful for Trameter reporting and job interviews.
- **Write a short postmortem for every customer-visible incident** and keep them in the repo.
- **Argue reliability in business terms** to Trameter: lost bookings, supplier trust, cost per
  booking.

---

## 7. Not covered by the sources, but know this

*Added by the reviewer from current practice (late 2026), not from the 17 talks. Verify free-tier
limits and prices before recommending.*

- **Burn-rate thresholds to start from (Google SRE Workbook, 30-day SLO):** page when 2% of budget
  burns in 1 h (burn rate 14.4, confirm on a 5 min window) or 5% in 6 h (burn 6, 30 min window);
  ticket on 10% in 3 days (burn 1, 6 h window). Below roughly 1 event/minute, burn alerts get
  noisy: use synthetic traffic, longer windows or plain "N failures in a row" alerts.
- **DORA has moved on.** Since 2024 it uses five metrics in two factors: throughput (lead time,
  deploy frequency, failed deployment recovery time) and instability (change fail rate,
  deployment rework rate). The 2017 "46x/440x/96x" figures are historical.
- **Cheap managed platforms for one person:** Cloud Run (scale to zero, generous free tier),
  Render, Railway, Fly.io (no longer a free allowance for new accounts), DigitalOcean App
  Platform, AWS Fargate/Lambda. Heroku entered "sustaining engineering" mode in Feb 2026 (no new
  features), so avoid it for new builds. A single VM with Docker Compose and systemd timers is fine
  for batch work.
- **Heartbeat monitoring for cron and pipelines:** the job pings a URL on success; you're alerted
  when a ping doesn't arrive (Healthchecks.io, Cronitor, Better Stack, Sentry Crons). This is
  the cheapest fix for "missing telemetry looks like success".
- **Error tracking and telemetry on free tiers:** Sentry (exceptions plus performance), Grafana
  Cloud free tier (metrics, logs, traces with short retention), Honeycomb free tier for wide
  events. Point the OTel Collector at one; switching later is config.
- **Uptime and status pages:** an external uptime check on the public endpoint plus a simple
  status page (Better Stack, UptimeRobot, Instatus) so users learn of incidents from you.
- **Postgres backups with PITR:** managed providers (Neon, Supabase on paid plans, RDS, Cloud SQL,
  Crunchy Bridge) give point-in-time restore; self-hosted, use pgBackRest or WAL-G shipping WAL
  to object storage. `pg_dump` alone gives RPO = time since last dump.
- **Secrets:** keep them in the platform's secret store or CI encrypted secrets, never in the
  repo or logs; prefer OIDC federation from GitHub Actions to cloud over long-lived keys; scope
  and rotate supplier API keys and put their expiry in the time-bomb calendar.
- **Cost alerts:** set billing budgets and alerts on every cloud and LLM provider account on day
  one, and hard spend caps where offered. A runaway loop or leaked key is a solo dev's most
  likely financial incident.
- **Runbooks:** one page per alert: what it means, how to confirm, the first three mitigations,
  how to roll back. Link it from the alert. Write it while calm.
- **Idempotent retries, timeouts, circuit breakers:** every outbound call has a timeout shorter
  than its caller's. Retry only idempotent operations, with exponential backoff, jitter and a cap.
  Send an idempotency key on booking and payment calls so a retry can't double-book. A circuit
  breaker stops hammering a failing supplier and fails fast to the degraded path.
- **Dependency and image updates:** enable Dependabot/Renovate with the pipeline as the gate;
  stale base images are a slow time bomb.

---

## 8. Where to look in `references/`

| Need | File (see INDEX.md) |
|---|---|
| Observability 2.0 and cardinality (Majors, Pragmatic Engineer), monitoring vs observability and sampling (Majors, Iterate 2018), OpenTelemetry components and demo (Better Stack), How Complex Systems Fail (Cook), pipeline failures and the TransFICC fintech pipeline (Farley ×2), DR tiers RTO/RPO (ByteMonk), SLI/SLO/SLA and error-budget arithmetic (Stack to Scale) | `ops-1-majors-stack-cook-farley-bytemonk-scale.md` |
| Observability engineering and SLO design (Fong-Jones), wide events and "not just ops" (Majors, NDC 2018), SRE 2nd edition authors, zero-downtime migrations (Yanaga), DORA and Westrum (Forsgren), SLO vs SLA plain English (Piyush), 12 logging practices (Better Stack), blameless postmortems (Mihailescu), when not to use Kubernetes (Will) | `ops-2-fongjones-majors-murphy-yanaga-forsgren-piyush.md` |
