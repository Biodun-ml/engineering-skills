# ops-1 — cross-cutting principles

1. **Observability is the ability to answer new questions about any system state without shipping new code.** Monitoring (predefined checks and dashboards) only covers known failure modes; modern systems fail as a long tail of unknown-unknowns. [Majors, Iterate 2018; Majors, Pragmatic Engineer]
2. **Emit one wide, structured event per unit of work** (request, booking, job, chunk, LLM call), with as much context as you can, and store it once in a columnar store. Logs, traces, metrics and SLOs can all be derived from it. Copying every request into 5-15 separate tools multiplies cost and leaves a human to correlate them. [Majors, Pragmatic Engineer]
3. **High-cardinality fields (user ID, booking ID, supplier, app ID) are the most valuable debugging data.** Metric/time-series tools price per tag combination and punish them, so don't put IDs into metric tags. Use structured events instead. [Majors, both talks]
4. **The health of "the system" matters less than the health of each user's requests.** Aggregates hide pockets of total failure (one customer, one supplier, one region), so you have to be able to break down by any dimension and compare against the baseline. [Majors, Iterate 2018]
5. **Aggregation throws data away for good, while sampling keeps it representative.** Keep 100% of rare or high-stakes events (errors, deletes, bookings) and sample common successes heavily. Curating sample rates replaces curating alerts. Never build billing off operational logs. [Majors, Iterate 2018]
6. **Instrument while you write the code, the same way you write tests.** Retrofitting loses intent. Use OpenTelemetry so instrumentation is vendor-neutral: API/SDK, auto-instrumentation libraries, semantic conventions (`service.name`, `service.version`), and a Collector with receivers, processors (batch/sample/filter) and exporters. Switching backend is then a config change. [Better Stack; Majors, Pragmatic Engineer]
7. **Page on symptoms, not causes.** Page on user-facing SLO burn (golden signals: rate, errors, latency) plus a few end-to-end checks on the money path. Every page must be actionable, and every dashboard is an artifact of a past failure. [Majors, Iterate 2018; Stack to Scale]
8. **An SLI is good/eligible events, defined from the user's view with explicit scope and window.** Correct business rejections (sold out, price changed) are not failures, and a 500 is. Keep SLI (observed), SLO (internal target) and SLA (contract, often looser) separate. [Stack to Scale]
9. **The error budget is 1 minus the SLO, on a stated denominator.** Request-based and time-based budgets are not interchangeable: 43.2 min/30 days is not the same as 10k bad requests. Alert on burn rate using multiple windows, and write a documented error-budget policy that links burn to deploy freezes or reliability work. Spare budget can pay for chaos drills. [Stack to Scale; Majors, Pragmatic Engineer]
10. **Missing telemetry looks like perfect availability.** Alert on absent data. SLOs don't cover data corruption or security incidents, which need their own stricter guardrails. [Stack to Scale]
11. **The deployment pipeline is the only route to production and is definitive for release.** If it's green you ship with no further manual work. Emergency fixes go through the same pipeline (so keep it fast), and nobody hand-edits prod config. [Farley, 3 Reasons]
12. **Pipeline results must be deterministic.** Pin everything (deps, environment through IaC, language version), isolate tests, and treat flakiness as a bug. Aim for a commit stage under 5 min with about 80% confidence, acceptance under 1 h, and a release verdict at least daily. The commit stage should catch most defects, with acceptance as the backstop. [Farley, 3 Reasons; Farley, Fintech pipeline]
13. **"Hard to test" means the design needs fixing,** e.g. fake the hardware or supplier at a seam. Scope each pipeline to an independently deployable unit. Coupled microservices that are only tested together form a distributed monolith. [Farley, 3 Reasons]
14. **Production is where code meets reality, so ship small and watch it.** Use feature flags, canaries and around 10% rollouts, look at the result within minutes while the context is fresh, and spend less effort perfecting staging and more on production guardrails. [Majors, both talks]
15. **Systems as imagined (diagrams, fault trees, postmortem reconstructions) are not systems as found,** which are dynamic, stochastic and under constant maintenance. Success comes from people monitoring, responding, adapting and learning, so give operators exposed controls, "lift points", visibility into internals, and support for mentally simulating an action before taking it. [Cook]
16. **Latent "time bombs" are made of individually reasonable choices.** Examples are expiry dates, certificates and keys, and same-day fleet-wide rollouts. Untested features and failovers usually fail the first time they are really needed. Stagger rollouts and exercise recovery paths on purpose. [Cook; ByteMonk]
17. **Choose disaster recovery by RTO (downtime tolerance) and RPO (data-loss tolerance, in time).** Tiers run backup and restore, then pilot light, then warm standby, then hot/active-active, with cost rising at each step. Failback matters as much as failover. A DR plan is only real if it is tested on a schedule (quarterly and after big infra changes) with restore drills, synthetic transactions and chaos tests. [ByteMonk]
18. **Engineers own their code in production,** with platform/ops people as consultants and tool-builders, not as a separate team that operates it for them. [Majors, both talks; Cook]
19. **Budget heuristic:** observability spend is roughly 15-20% of cloud spend. Consolidate vendors, and "care less about the customer" is the only free cost-cut. LLM features are a trace-shaped, high-cardinality software problem, and if you can compute an answer, don't let AI guess it. [Majors, Pragmatic Engineer]

---

## Charity Majors (with Gergely Orosz) — Observability: the present and future (https://www.youtube.com/watch?v=SvEjS4-2WJQ)
**Observability 2.0 = one store of wide, structured, high-cardinality events per unit of work, from which logs, traces, metrics and SLOs are all derived — instead of copying every request into 5-15 disconnected tools.**

- **Definition.** Observability is understanding the intersection of your code, your systems and your users. It is not only about errors/outages/"nines"; it underpins the development feedback loop and lets engineers explain their work in business terms (money, impact). Origin: at Parse (60k -> 1M+ mobile apps, Rails fixed worker pool) any one hot app or slow backend took everything down; logs could not say which app caused it. Facebook's Scuba (in-memory columnar store) let them break down by app ID, user ID, raw/normalised query — time to find the cause fell from hours (or never) to seconds, so it stopped being an engineering problem.

- **Three pillars (metrics, logs, traces) = Observability 1.0.** Coined ~2017 and amplified by vendors who each had a product per pillar. Underneath, the same request is stored in metrics, structured logs, unstructured logs, tracing, profiling, analytics... The only "join" is a human eyeballing similar graph shapes or copy-pasting IDs. Cost scales with the number of copies. Vendor "unified observability" often means a unified bill or a unified UI, not unified storage.

- **Observability 2.0 mechanisms.**
  - Emit *fewer but wider* logs: one wide structured event per unit of work (request, job, batch) with lots of context fields. "The bridge from 1.0 to 2.0 is logs."
  - Store them in a columnar store (ClickHouse-style) so you can add a field any time without predefining indexes/schemas, then slice, group-by and filter in real time.
  - No dead ends: from an event go to a trace, plot it over time, derive metrics and SLOs from the same data, click an SLO burn and see exactly which events are failing and what is different about them.
  - "BubbleUp"-style analysis: select an anomalous region of a graph, compare every dimension inside vs baseline (e.g. Android + region X + secondary DB + batch size N). Most debugging is "here is what I care about — how is it different?"
  - Newer startups are converging on this: ClickHouse/columnar, OTel-native, wide structured events organised around units of work.

- **Metric vs metrics.** "metrics" loosely = telemetry; "the metric" = a number with tags. Very cheap and fast, but carries no context or relationships between fields.

- **Cardinality (key concept).** Number of unique values in a field: user ID/request ID/IP = highest; `species=human` = lowest. Time-series metric stores keep a separate series for every unique tag combination ("custom metrics" means unique combinations, not lines of code). Adding `hostname` past ~100 hosts or adding IP address can blow up cost 10-100x overnight — e.g. a $200k/month Datadog bill becoming $2M with no code change. Mature teams on 1.0 tools spend most of their time *governing cardinality*, which is perverse: high-cardinality fields are the most valuable for debugging because they identify things.

- **Why observability is expensive.** (1) Your bar: firms that must understand every request (banks, delivery) pay more than spray-and-pray ad firms — the cheapest way to cut the bill is to care less about customers. (2) The multiplier of storing each request many times. (3) Cardinality pricing. Rule of thumb: observability spend ~15-20% of cloud spend; five vendors each aiming for that is nonsense — hence industry consolidation.

- **Trade-off note.** Metrics were designed for an era of expensive storage/compute ("can't afford to keep the nginx log, so derive counters"). Falling costs now allow raw structured events. Metrics still fit: cheap long-range trends, counters, and very high scale where structured data becomes too expensive (higher than most think). Target ratio: invert today's 80% metrics / 20% structured to 80% structured / 20% metrics. Prometheus/Datadog are mature "last great metrics products".

- **OpenTelemetry.** Successor to OpenCensus/OpenTracing; now the top CNCF project by commits/committers (overtook Kubernetes). Instrument once with OTel and point the firehose at any backend — ~90-95% true. Big value even if you ignore its internals: consistent naming, structure and semantic conventions mean backends can compute/derive more. Reduces lock-in and gives negotiation leverage. Critique: big and bloated, but does the job. (note: CNCF velocity reports have generally ranked OpenTelemetry second to Kubernetes by contributor activity; treat "top project" as her loose claim. It is among the most active CNCF projects.)

- **Common mistakes.**
  - Waiting until production breaks to add any instrumentation. Debug in dev the way you will in prod.
  - Static dashboards as the main view — fine for show, poor for building a mental model; you only see what you thought to graph. You need to iterate "and then what?" interactively. (Interviewer's admission: their team dashboards were mostly kept green for visitors.)
  - Use **SLOs as the entry point** instead of dashboards. SLOs are the API between an engineering team and the business. Anti-pattern: SLOs computed from separate data than you debug with.
  - Instrument when you write the code, like tests — retrofitting loses original intent. Start when code is "real" (meant for users); skip for throwaway prototypes.

- **Error budgets in practice.** Leftover budget can fund chaos experiments: Honeycomb kills the oldest Kafka node every Monday, forcing the bootstrap/recovery path to stay tested — pages for Kafka stopped. SLOs also protect against micromanagement (if obligations are met, how the team spends time is its own business) and give a negotiating tool ("we are missing SLO, feature work pauses for reliability work").

- **Deploys.** Observability + progressive delivery (feature flags, canaries, 10% rollouts, test requests in prod) is greater than the sum: ship within 15-20 minutes while context is fresh, watch precisely, roll forward/back with a scalpel. Also trace CI/CD runs to see where test time goes; keep build-to-prod time short.

- **Ownership.** Engineers own their code in production; platform teams treat internal engineers as customers and build tools but don't take your on-call. The split "devs build, ops operate" never produced excellence.

- **AI.** Three intersections: training models, building with LLMs, and the flood of code nobody fully understands. "If you can compute the answer, compute it" — AI guesses are worse than computed answers when you had the context. LLM observability can't live in isolation; it is a trace-shaped, high-cardinality problem from inputs across services, through the model, to human feedback — a software problem with non-deterministic parts. Good observability is a prerequisite to letting AI-written code or agents near production.

- **Build/buy.** Most should buy; you don't want a broken observability tool at 2am on top of broken software. Innovation tokens: startups get ~3; Honeycomb spent two on a custom columnar store (Retriever), later moved query execution to Lambda with data aged from local SSD to S3 and fanned out at query time, because ~99% of written data is never queried. Normally: never write your own database.

- **For Sam.** One wide JSON event per booking attempt / supplier call / pipeline chunk / LLM call (booking_id, supplier, endpoint, latency_ms, status, retry_count, chunk_id, rows_in/out, model, tokens, cost) is the cheapest high-value move. Store in DuckDB/ClickHouse/Parquet or a free-tier OTel backend; avoid per-tag-priced metrics tools for high-cardinality IDs. Define SLOs (e.g. supplier booking success) from the same events. Instrument the LLM pipeline as a trace across retrieval, prompt, model, post-processing, not just model I/O.

## Charity Majors — Observability and the Glorious Future, Iterate 2018 (https://www.youtube.com/watch?v=rS0tH9DnKUg)
**Modern distributed systems fail as an endless long tail of unknown-unknowns; monitoring (predefined checks + dashboards on aggregated metrics) can't explain them, so you need rich, high-cardinality events you can query in ways you didn't predict.**

- **Monitoring vs observability.**
  - Monitoring: black-box, built for known failure modes, biased to outages and actionable alerts, best practice is "don't stare at graphs, let the system tell you". Every postmortem ends with "add a dashboard so we find it faster next time" — a model unchanged in ~20 years.
  - Observability (from control theory): you can explain *any* state of the system, including ones never seen before, **without shipping new code** and without predicting the question in advance. It's a superset of monitoring; monitoring is how you operate a service, instrumentation is how you develop it.
  - Test of an observable system: the team can track down a never-seen-before problem without dread or pain.

- **Why the old model broke (complexity).** LAMP era: few components, known-unknowns, intuition-friendly dashboards, and health evenly distributed (99.5% uptime = everyone sees ~0.5% errors). Modern systems (Parse: hundreds of MongoDB replica sets running user-uploaded queries, user JavaScript, million apps on shared hardware) look more like a national power grid: some problems only visible hyper-local, some only zoomed-out, some systemic (a batch of aging parts). Polyglot persistence is now normal. Problems: heisenbugs, a single node snapshotting a live volume infecting everything, causality impossible to read off dashboards because everything spikes together.
- **Health of "the system" is nearly irrelevant**; the health of each event/request/user is what matters. Lose one AZ and dashboards are red while users notice nothing; meanwhile one small customer (Disney at 8 req/s vs platform 100k req/s) can be 50% down and never appear in a top-10 graph. Distributed systems create pockets: you're 99.999% up for most but 100% down for users whose surname starts with a certain letter.
- **War story.** Push notifications "down" for some users: more capacity + round-robin DNS exceeded the UDP packet size; TCP fallback should have worked everywhere except one router in Eastern Europe. Lesson: without per-user slicing, chasing user reports is so painful teams stop respecting users' complaints.

- **High cardinality is the secret.** Breaking down by any one of a million app IDs and then by any combination of other fields = effectively auto-generated dashboards per customer. Example insight: a customer's login query went from 29.9s to 30.1s because their dataset grew, tipping it past a 30s timeout.
- **Events not metrics.** A metric is a number plus a handful of tags (hundreds), designed when disk was precious; high-cardinality tags blow up the key space (write amplification). An event = a structured record where all fields were true for the same request at once, so correlation is built in. Events = structured logs, ideally not written to local disk. Unstructured strings vs structured data is "grep vs all of computer science" — fix your logs.
- **Aggregation destroys data.** Time-series DBs collapse each interval into one number; once smushed you cannot un-smush, so you can never ask a new question about the past. You need the raw event traceable to a real user's experience.

- **Sampling (instead of dropping fields).** No one pays for observability 100x the size of prod, so trade *rows* not *width*:
  - Keep 100% of rare/important things, sample heavily the common things that only matter in aggregate. Web: sample 200s heavily except rarely hit endpoints; keep most 500s (during a full outage it's fine to sample those too). Parse DB: kept 100% of deletes (rare, high stakes), ~100% of inserts, sampled selects.
  - Reframe from "I must find that exact event" to "I must find a representative example of it".
  - Curating sample rates is "the new, much better curating alerts". Start by paying for 2-3x what you think you need and prune.
  - **Never build billing off operational logs** — billing data and operational telemetry have different correctness needs; mixing causes bad outcomes either way.
  - Keep events wide (hundreds to thousands of dimensions); don't discourage adding fields.

- **Debugging style.** Dashboards = pattern-matching with eyeballs, relying on whoever has been there longest (hero culture, burnout, doesn't scale). Event-query debugging = BI-style: ask, look, ask a follow-up, follow breadcrumbs. On her teams, within 3-6 months the best debuggers became the most *curious* people, not the longest-tenured. "Every dashboard is an artifact of a past failure."

- **Alerting reward.** With trustworthy exploratory tools you can cut paging to roughly the golden signals — request rate, latency, errors (maybe saturation) — plus a few end-to-end checks on the money-making critical path. Drop the embarrassing "50 unrelated alerts fire, which we know means X".

- **Testing in production.** You test in prod whether you admit it or not; staging only catches the easy stuff, and most teams overspend making staging work and underspend on production guardrails. "Fourth trimester" metaphor: new code is helpless — wrap it in a feature flag, canary to ~10%, watch, then expand. Ship, then actually go look whether what you expected happened. Q&A: with strict B2B SLAs, isolate and loosen the critical path so most things fail gracefully, and create tiers of users you can experiment on (internal, free tier, bots).
- **Ownership.** Services need owners, not operators. Ops skills are table stakes for software engineers; on-call shouldn't be a life sentence. "Suck" is a legitimate, cheaper choice if your users tolerate it; otherwise invest in tooling or throw bodies at it.

- **For Sam.** For supplier-API booking flows, slice by supplier + endpoint + destination + client — a single supplier silently failing is invisible in an aggregate success rate. Log 100% of bookings/cancellations/payment events and errors; sample successful search/availability calls. Keep billing/financial reconciliation records separate from operational logs. Page only on booking success rate, latency and an end-to-end synthetic booking check.

## Better Stack — What is OpenTelemetry? Explanation and Demo (https://www.youtube.com/watch?v=LzLULxhyIpU)
**OpenTelemetry is a vendor-neutral standard (API + SDKs + instrumentation libraries + Collector + OTLP wire protocol) for producing high-quality, correlated telemetry; it is deliberately not a backend — you bring your own UI/storage.**

- **Problem it solves.** In a system of microservices + SaaS + cloud providers, a symptom ("checkout keeps spinning") could live anywhere. Historically logs, metrics and traces were produced by separate, often proprietary tools, each holding part of the picture; correlation meant hopping between services. No standard meant vendor lock-in and every company instrumenting differently. Goal: "high-quality, portable telemetry everywhere" — the point is better data, not more data.
- **The three signals as presented (classic framing):**
  - Logs: text records of specific events; tell you *that* a process died of OOM, not that memory was creeping up.
  - Metrics: numeric state/resource use over time; good for trends and early warning (e.g. warn or auto-restart at 90% memory).
  - Traces: follow one request across services; best for localising where an error happens.
  (note: compare with Charity Majors' critique above — OTel helps unify these via shared context, but storing them separately still multiplies cost.)
- **Three fundamentals of OTel.**
  1. *Signals* — logs, metrics, traces (data points).
  2. *Context* — when, where, what triggered it; trace/span IDs propagated so signals correlate.
  3. *Semantic conventions* — standard attribute names (`service.name`, `service.version`, `http.method`, `url`, `user_agent`...) so your data, your cloud provider's and your SaaS vendor's mean the same thing and can be joined.
- **Components and how they fit.**
  - **API**: language-agnostic interface you code against. Safe to call when no SDK is installed (no-op, no overhead, no errors) — so library authors can embed instrumentation that lights up only if the user enables OTel.
  - **SDK**: per-language implementation that captures, processes and exports.
  - **Instrumentation libraries**: prebuilt for popular frameworks (Express, HTTP, AWS SDK, fs, GraphQL, loggers like Bunyan/Pino/Winston). Search the OTel registry before hand-writing spans. OTel doesn't want to replace your logger; it bridges logger output into OTel format.
  - **Zero-code / auto-instrumentation**: install a couple of packages + env vars and get useful traces out of the box. In Node, `getNodeAutoInstrumentations()` detects libraries in use.
  - **Resource**: attributes describing the emitting service (service name, version, host, process, runtime) attached to every signal.
  - **Exporters**: start with console exporters to inspect the format, then switch to **OTLP** exporters (gRPC or HTTP, default collector port 4317/4318 — demo used localhost:4318 paths `/v1/traces`, `/v1/logs`, `/v1/metrics`).
  - **Collector** (run e.g. as a Docker container): *receivers* (OTLP, hostmetrics, Prometheus scrape — including the collector's own metrics) -> *processors* (batching, filtering, sampling, attribute modification) -> *exporters* (to Prometheus, Jaeger, a SaaS backend), wired together in `service.pipelines`. Swapping vendors = changing the exporter config, not app code.
- **Demo mechanics (Node/Express).** `instrumentation.ts` builds a `NodeSDK` with resource, trace/log/metric exporters, and auto-instrumentations; it must start *before* the app code loads so everything is patched. A single "listening" log line came out enriched with SDK language, PID, host, runtime version, timestamp, severity. Express auto-instrumentation produced a trace with URL, method, host, user agent, and an `http.server.duration` histogram metric — no manual instrumentation code.
- **Rules of thumb.**
  - Use semantic-convention constants for keys, never invent your own names for standard things.
  - Always set `service.name` and `service.version` (version lets you correlate behaviour with deploys).
  - Route through a Collector so sampling/filtering/batching and backend choice stay out of app code.
  - Check the language's maturity status per signal (traces/metrics/logs are at different stability levels by language).
- **Gotchas.** Console output is unreadable at volume — it's for verifying setup only. OTel provides no analysis UI. Initialise before importing instrumented modules or you'll miss spans.
- **For Sam.** Python has equivalent auto-instrumentation (`opentelemetry-instrument`, instrumentations for requests/httpx/FastAPI/SQLAlchemy). Instrument supplier HTTP clients automatically to get per-call spans with latency and status; add custom attributes (supplier, booking_id, chunk_id). Run a local Collector to fan out to a free-tier backend or to files/ClickHouse, and to sample high-volume successful calls — switching vendor later is a config change.

## Richard Cook — How Complex Systems Fail (Velocity 2012) (https://www.youtube.com/watch?v=2S0k12uZR14)
**The surprise is not that complex systems fail but that they fail so rarely; success comes mostly from people continuously monitoring, responding, adapting and learning in the "system as found" — so design must give operators the controls and visibility to do that, rather than locking them out.**

- **Framing.** Cook (anaesthesiologist, safety researcher, 25 years studying ORs, ER/ICU, cockpits, military software, semiconductor fabs, power) flips the question: given how systems are built and run, we "deserve" far more accidents. Why so few? Ops people all know near-misses (everything unplugged except one forgotten power tap) — the system ran on luck and human intervention.

- **Infusion-pump story (latent failure / time bomb).** A hospital standardised on one vendor's new infusion pumps, installed all on the same day. A configuration field "mandatory software upgrade date" was set to one year out during setup without anyone thinking about it. Exactly a year later, just after midnight, pumps started refusing new commands; within ~45 minutes ~20% were down, eventually all would have failed within ~7-8 hours as staff tried to change settings. One biomed tech on duty swapped pumps and later every pump had to be touched in 24 hours. Lessons:
  - Individually reasonable features + reasonable setup steps combine into a hidden time bomb.
  - Same-day fleet-wide rollout = synchronised failure (no staggering).
  - Recovery depended on one human being present; nobody treated it as a big event afterwards.
  - Software analogues: certificate/token/API-key expiries, licence dates, hardcoded TTLs, cron dates, disk-fill timelines.

- **What the real operating world looks like (research findings).**
  - Surprises are common and some are existential (not "latency a bit high" but "it stops working").
  - Demand/op-tempo varies a lot and unpredictably.
  - "Stuff don't work as advertised": many features/failovers are first exercised years after install and fail then (note: this is why restore tests and failover drills matter).
  - Novel conditions are frequent; systems need constant tailoring (parameters, turning services on/off).
  - Users use systems in unanticipated ways (email for huge files).
  - Technology and staff change continuously.
  - Constant shallow conflicts (squeeze a bit more) and deep conflicts (take it down to fix vs fix live).
  - "The normal world is not well-behaved", yet success is high — the question is whether success is *because of* or *despite* the design.

- **System as imagined vs system as found.**
  - As imagined: state diagrams, rack layouts, fault trees, probabilistic risk assessments — static and deterministic. Used in design, development, and (importantly) in post-incident reviews.
  - As found: dynamic, stochastic, only statistically predictable, held together by people at screens and "do not touch these wires" signs. Mean time between maintenance is effectively zero — something is always being changed.
  - Gotcha: postmortems are done in the as-imagined world, so they misread what actually happened (ties to blameless postmortem practice: reconstruct what operators saw and why their actions made sense).

- **Resilience = four activities** carried out by often-invisible operator communities: **monitoring** (seeing what's happening), **responding** (anticipating where it's heading and acting), **adapting** (changing how the system behaves), **learning** (about the system).
- **Reliability vs resilience.**
  - Reliability is built at design time: rigid boundaries, layers, defence in depth, redundancy, abstraction/hiding details, formal reviews, accountability.
  - Resilience is wanted at run time: absorb transients, recover smoothly and fast, prioritise high-level goals and choose what to sacrifice under pressure, recognise and respond to situations never considered in design, adapt to change (systems designed with 4-7-year-old assumptions must work 3-5 more years).

- **Design-for-resilience checklist.**
  1. **Support continuous maintenance** as a first-class design concern, not periodic add-on.
  2. **Reveal the controls** to operators: at 3am only they can act; trust them rather than designing them out.
  3. **Show the lift points**: make it obvious how to pick up a component and move/replace it (like markings on heavy machinery).
  4. **Support mental simulation**: give operators deep knowledge of configuration and current state so they can predict what an action will do.
  5. **Open the objects/methods**: excessive black-box abstraction prevents reasoning during failure; you must be able to see inside layers.
  6. **No "don't touch" zones**: get rid of parts that cannot be operated.
  7. **Empower operator learning.**
  - Agenda: hand ops the keys (and build the competence if you don't trust them yet), make resilience the top design priority, fund understanding of resilience across the lifecycle. Further reading: Hollnagel, "resilience engineering"; FRAM (Functional Resonance Analysis Method).
- (note: this talk covers the "how complex systems *don't* fail" angle; Cook's well-known 18-point paper "How Complex Systems Fail" — e.g. catastrophe needs multiple failures, root cause is a fallacy, hindsight bias — is referenced by title but not walked through here.)

- **For Sam (solo operator).** You are the operator community. Build "lift points" and exposed controls: admin scripts/runbooks to pause a supplier, replay a failed booking, re-run a pipeline chunk, flip a feature flag, rotate a key. Keep a calendar of time bombs (API key/token/cert expiries, supplier contract dates, free-tier limits). Stagger rollouts. Exercise failover/restore paths on purpose since untested ones won't work when needed. Prefer libraries/services whose internals you can inspect over opaque black boxes on the critical booking path.

## Dave Farley — 3 Reasons Your CI/CD Pipeline Isn't Working As It Should (https://www.youtube.com/watch?v=m1oMj29P--Y)
**A deployment pipeline is not a build script or a Jenkins install: it is the single, deterministic, definitive route to production — if it passes, you release with no further work.**

- **Definition (from the person who coined the term).** Semantic diffusion has watered it down. A build script and a CI server are parts, not the thing. Nor is it just "a workflow of automation steps". It is part of the release process whose job is clear, fast, trustworthy feedback: either "good, move on" or "stop, you broke something". Defining property: **definitive for release**.

- **Failure 1 — results are not repeatable/reliable.**
  - Same versions of *everything* (code, dependencies, config, host, environment, language version) must give the same result every run. "Sometimes it passes" is not acceptable.
  - Requires controlling all variables: infrastructure as code so environments are version-controlled and reproducible, pinned dependency versions, controlled system state within each test, test isolation so tests don't affect each other.
  - Side effect: forces more determinism into the design itself — a good thing.
  - Reliable also means *testing what matters*: enough meaningful tests that passing implies production will very likely work.
  - **Three phases:**
    - *Commit*: target ~80% confidence that if these pass, everything later passes; feedback in **under 5 minutes**; catches most common mistakes.
    - *Acceptance*: everything else that determines releasability — functional behaviour, security, performance, regulatory compliance. Ideal **under 1 hour**; to count as continuous delivery, a definitive release verdict **at least once a day**.
    - *Production*: last resort for learning — what users like, what you missed; not primarily bug-finding. Teams doing this well see production bugs drop roughly two orders of magnitude (hundreds -> single digits).

- **Failure 2 — the pipeline isn't the only route to production.**
  - Multiple paths explode complexity and destroy control. Every change — code, infra, OS update, config — goes through version control and the same tests. One failing test rejects the release.
  - Anti-pattern: an untested "emergency release" fast path (a firm's "class 7 release in under an hour" that skipped tests). Emergencies are exactly when you most need tests; make the normal pipeline fast enough to use in emergencies too. A one-line typo fix and a critical hotfix go the same way.
  - Anti-pattern: ops logging into servers to change config manually, silently undoing infrastructure-as-code changes and making releases flaky (configuration drift).

- **Failure 3 — the pipeline isn't definitive for release.** Symptom: all tests pass but there's still manual work (sign-offs, manual QA, "test with subsystem X first").
  - Making it definitive removes excuses and steers design choices.
  - "Too hard to automate" is a design smell, not a reason for manual testing (slower, less repeatable, lower quality). Example: an ATM test rig with toy robot arms inserting cards — lateral, but better: modularise so the card mechanism is tested separately (occasional smoke test) and transactions are driven by faking hardware inputs in software, enabling many more, faster tests. Testability should drive design (TDD helps).
  - **Scope**: the commonest real problem — premature decomposition into microservices that are so coupled they're only trusted when tested together = a **distributed monolith**, slower and more complex than a monolith, and it delays integration until "release time", killing CI's benefits. CI means integrating at least daily. Scope each repo + pipeline to an **independently deployable unit** — something you can release alone without worrying about its dependencies (dependencies exist, but the design means you don't have to coordinate).

- **Rules of thumb.**
  - If the pipeline is green, you ship. If you wouldn't, add the missing check to the pipeline.
  - Commit stage < 5 min, acceptance < 1 h, verdict at least daily.
  - Zero manual changes to prod; everything via VCS.
  - Hotfixes use the same pipeline.
  - Flaky tests are pipeline bugs; fix determinism.

- **For Sam.** For a solo dev: one GitHub Actions workflow per deployable unit; fast unit tests + lint on every push; slower acceptance stage running pipeline jobs on a fixed sample dataset and supplier adapters against recorded responses/sandbox (fake the supplier like the ATM hardware). Pin Python deps (lockfile) and container images for reproducibility. Never hot-edit the server or production DB by hand; even urgent fixes go through the pipeline — so keep it fast. Don't split Tereka/booking code into microservices prematurely.

## Dave Farley — Real Example of a Deployment Pipeline in the Fintech Industry (https://www.youtube.com/watch?v=bHKHdp4H-8w)
**A walkthrough of TransFICC's live pipeline: boring tools (Java, Gradle, Jenkins, Ansible, SSH-based deploy) organised around trunk-based development, TDD, a ~7-minute commit stage, a big shared red/green display, and parallel acceptance/perf/adapter stages.**

- **Context.** TransFICC is a London fintech that acts as a translator/router between trading organisations: instead of N-to-N bespoke integrations, each client integrates once with TransFICC, which maintains adapters to many trading venues ("a device driver for trading"). Highly regulated, security-critical, high performance. Built on CD principles from day one. Stack choice doesn't matter — CD works with ordinary tech. (Directly analogous to Trameter as an aggregator over many supplier APIs.)

- **"Big Feedback" display.** Plain HTML/CSS. Green = releasable as far as this stage knows; red = not releasable; grey = stage disabled (being added or retired). Commit stage box shows current change, commit message, author, time since last run, and a progress bar (so people waiting for a change know how long). Keep the message simple. A big red block on a shared screen is far harder to ignore than an email or Slack ping.

- **Pipeline shape.**
  - *Commit stage*: unit tests, the gate.
  - *Integration tests*: a tactical addition for things too slow for commit but which catch common errors earlier than full acceptance. Farley's advice: run them **in parallel** with acceptance after commit passes, for faster early failure, rather than as another serial gate.
  - *Acceptance stage*: whole-system trading-scenario tests (thousands of test cases total, many automated environments).
  - Then three groups: multiple **performance tests**; **common stable libraries** with their own independent pipelines (they change rarely, so they're treated like third-party deps — when one changes, its built binary is committed into the main pipeline and re-evaluated); and **venue adapters**, each tested independently and in parallel.

- **Happy path demo.** Developer writes a focused unit test first (true TDD), makes it pass, then uses a single build target that builds, tests, packages and *deploys the whole distributed system on his laptop using the same deployment mechanism as production*, writes and passes an acceptance test, commits. Requirement-to-commit: ~11 minutes. Commit stage: ~7 minutes (Farley: due an optimisation pass to get back under 5). Once committed, the developer moves on; the pipeline will reject anything non-releasable; green means releasable with no extra work.

- **Failure demo.** Another dev runs ~500 unit tests locally in ~15 seconds (TDD-produced tests are fast), commits straight to master (trunk-based development, true CI), watches his commit, sees the commit stage reject it (8 failing tests), clicks through to Jenkins details. Uses the team's **embargo** feature to block the pipeline while he fixes — a blunt instrument that blocks everyone's path to prod, use sparingly; CI is a publication process and the pipeline is a shared resource. Reproduces locally by deploying and running the most relevant acceptance tests, finds the failure, fixes, reruns locally, recommits, green.
  - Healthy signal: commit stage catches most errors; acceptance acts as **backstop** and confidence-builder. Optimise so defects are found at the cheapest stage.

- **Rules of thumb.**
  - Developers watch their commit through the commit stage and fix immediately if red.
  - Local deploy of the full system via the production deploy mechanism = same path tested every day.
  - Parallelise independent suites (adapters, perf) to keep wall-clock time low.
  - Treat slow-changing shared libraries as versioned external deps with their own pipelines.
  - Watch commit-stage time creep and periodically optimise.

- **For Sam.** Model each supplier adapter (Viator, RateHawk, Coras...) as an independently tested unit with its own contract tests against recorded fixtures, run in parallel. Have one command that brings up the whole booking stack locally (docker compose) using the same scripts as prod deploy. Even solo, a visible status (GitHub Actions badge/notification) and "don't leave main red" discipline matters.

## ByteMonk — The Ultimate Guide to Disaster Recovery: RTO, RPO & Failover (https://www.youtube.com/watch?v=OmASCUJEVy8)
**Pick a DR strategy by working backwards from two business numbers — RTO (how long you can be down) and RPO (how much data, in time, you can lose) — then prove it works with monitoring, synthetic checks and regular failure drills.**

- **Definitions.**
  - *Disaster recovery*: plan + actions to bring systems back after anything from a server crash to a regional/natural disaster; goal is minimising downtime and data loss.
  - **RTO (Recovery Time Objective)**: max tolerable downtime before the business is hurt (lost revenue, unhappy customers, chaos). RTO 1h = must be restored within 60 min. About *speed*.
  - **RPO (Recovery Point Objective)**: max tolerable data loss measured in time. RPO 15 min = can lose up to the last 15 minutes of writes. Drives *how often you back up/replicate*.
  - Example: e-commerce RTO 1h, RPO 5 min (losing >5 min of orders = lost sales).
  - Tighter RTO/RPO => more robust and more expensive DR; the job is finding the right balance per system.
- **Cloud advantage.** Multiple regions make geographic distribution of infra and backups cheaper/faster than owning multiple physical data centres.

- **Failover / failback mechanics (example: primary region A, backup region B).**
  - Continuous health checks on response time, error rate, uptime (e.g. failover if response time > 500 ms or unreachable).
  - On detection, a traffic manager / load balancer / DNS health-check routing (e.g. Route 53) redirects traffic to region B automatically; DNS records updated.
  - Region B must have current data via real-time or near-real-time replication (cross-region object replication, DB replication) — orders replicated within seconds.
  - **Failback** is as important: once A is healthy, re-sync writes made during failover back to A, reroute traffic, and restore B to standby readiness.
  - (note: the video says failover is "without any disruption" and "no data loss"; in practice async replication means RPO > 0 and DNS TTL/caching means some users see errors during cutover. Also automatic failover risks flapping/split-brain on false positives — many teams keep a human in the loop for DB promotion.)

- **The four AWS DR strategies (cheap/slow -> expensive/fast).**
  1. **Backup & restore**: back up data and app definitions, nothing running on standby. Cheapest (storage only), simplest; RTO/RPO hours, often manual restore. For non-critical systems/archives.
  2. **Pilot light**: minimal core always on (e.g. replicated DB, core app), scale up to full capacity on disaster. Cheaper than a duplicate; scaling takes time. Moderate RTO, tolerates some data loss.
  3. **Warm standby**: scaled-down but fully functional copy always running; switch fast with less scaling. Costs more than pilot light. Low-moderate RTO/RPO.
  4. **Hot standby / multi-site active-active**: full duplicate in parallel; near-zero RTO and RPO, seamless failover; roughly double cost. For mission-critical (banking, major e-commerce).
  - Decision rule: tight budget and can tolerate downtime -> backup & restore; mission critical -> hot; in between -> pilot light/warm standby.

- **Proving it works.**
  - *Monitoring*: server health, network latency, DB performance; threshold alerts or real-time dashboards to catch small problems early.
  - *Synthetic monitoring*: scripted fake users (log in, place order, view page) continuously; you learn of breakage before real users report it.
  - *Chaos testing*: deliberately kill servers / block network to confirm failover and backup procedures actually work under pressure.
  - Test on a schedule — quarterly or after any major infra change — and keep the DR plan a living document updated with lessons.
- **Missing but important (note):** the video doesn't stress **restore tests** explicitly — a backup you have never restored is not a backup; measure actual restore time against RTO. Also keep backups isolated (different account/provider, immutable/versioned) so ransomware or a bad script can't delete both prod and backups.

- **For Sam.** Backup & restore is the right tier for nearly all of Sam's systems (data pipelines can be re-run from source; Tereka/Riselist are small). Set explicit RTO/RPO per asset: e.g. booking records RPO minutes (managed Postgres with PITR), master location dataset RPO = last good Parquet snapshot (versioned object storage), LLM pipeline outputs reproducible. Schedule a quarterly restore drill to a fresh instance and time it. Add one synthetic end-to-end booking check (sandbox) as the main external probe.

## Stack to Scale — SLI, SLO, SLA & Error Budgets: Google SRE Approach (https://www.youtube.com/watch?v=yjMQ4BrfPSA)
**"99.9%" means nothing until you define which events are eligible, which count as good, over what window and on what denominator; then the error budget and its burn rate turn reliability into a measurable product requirement that governs release risk.**

- **Problem.** An advertised 99.9% is marketing unless you specify: all users? API requests? partial outages? slow responses? maintenance? Reliability is not an unlimited goal; it's a product requirement to be measured and managed, traded against feature velocity.
- **SLI (indicator) — what you observe.** A ratio of good events / eligible events for one aspect of behaviour, from the user's view.
  - Availability example: fraction of eligible API requests returning a correct non-error result within a bound.
  - Business-semantic example: fraction of checkout attempts that either confirm a ticket *or* correctly reject (sold out). A correct "sold out" is not a reliability failure; a 500 is.
  - Latency as its own SLI: fraction of successful searches completed within 300 ms — better than "average latency is fast".
  - Be explicit about scope, event definition, population and window, or two teams can both report 99.9% while measuring different experiences.
  - Different user-visible stages differ: rendering the homepage vs completing checkout; notification *accepted into queue* vs *delivered to device*.
- **SLO (objective) — what engineering aims for.** Target for an SLI over a window, e.g. 99.9% good eligible checkout requests over a rolling 30 days. It's a chosen target, not an observed value or universal best practice. Multiple SLIs can make sense (chat: acceptance and eventual delivery; video: startup time and buffering) but too many conflicting SLOs paralyse decisions — pick those that track **critical user journeys**.
- **SLA (agreement) — what's contractually promised.** External commitment with measurement rules, exclusions, reporting, remedies/service credits. Internal SLOs are not automatically SLAs, and SLAs don't dictate implementation. Commonly set internal SLO *stricter* than the SLA to leave margin. Distinguish by *role*, not as a ladder of percentages.

- **Error budget arithmetic (request-based).** 10M eligible requests in 30 days, SLO 99.9% -> allowed bad fraction 0.001 -> 10,000 bad requests. 6,000 used -> 4,000 left. Valid only once events are defined honestly: don't exclude hard requests to look good; don't count contractually correct business rejections as failures.
- **Time-based vs request-based — never conflate.** 30 days = 43,200 minutes; 0.1% = 43.2 minutes. This is NOT interchangeable with 10,000 requests: 5 minutes down at peak hour hits far more requests than 5 minutes at 3am, and partial degradation fails some requests rather than whole minutes. "Three nines = ~43 min/month" is only approximately true for a 30-day time-based window with a stated definition of "unavailable". Say which basis you use and how partial failures count.
- **Burn rate.** Speed of consuming budget relative to plan. Burn rate 1 = would use exactly the whole budget over the window; 10 = ten times too fast. Need to tell short spikes from sustained degradation: Google SRE Workbook's **multi-window, multi-burn-rate alerting** (check short and long windows together). Don't copy example thresholds blindly — fit them to your SLO, traffic and response process. Principle: alert on meaningful user impact and urgency, not metric wobble; a page must mean an actionable reliability problem.
- **Why error budgets.** A shared decision framework: without it product pushes features at any cost while ops demands zero risk. Healthy budget -> take managed launch risks; fast burn -> evidence to slow risky deploys, improve tests, fix a weak dependency. Make it a **documented error-budget policy**, not a weekly argument.
- **Gotchas.**
  - A good percentage doesn't excuse security incidents, data corruption or user harm — these need separate, stricter guardrails.
  - Monitoring data loss can silently produce fake 100% availability (no events = no bad events) — guard against missing data.
  - A dashboard alone isn't an operational policy: write down SLO owner, review cadence, and actions on sustained burn.
- **Dashboard design exercise.** Feed good/bad counts, compute good fraction vs target and budget remaining; add short-window error-rate chart and long-window trend; a simulated incident shows the short window reacting well before the 30-day number moves. Build on trustworthy request counters/event streams.
- **Interview/design recipe.** Don't start with "N replicas". Start with the user journey -> define success -> pick measurable SLI -> agree realistic SLO + window -> compute budget on the right denominator -> monitor long-term attainment and burn rate -> actionable alerts backed by logs/metrics/traces -> tie reliability data to deploy policy and investment.
- (note: small channel, very recent upload; content is consistent with the Google SRE book/workbook.)

- **For Sam.** Natural SLIs: booking success = (confirmed + correct supplier rejections e.g. "sold out"/"price changed") / eligible booking attempts — supplier unavailability counted honestly; search latency SLI (e.g. % availability searches < 2-3 s, supplier-dependent); pipeline freshness SLI (% of daily runs completing by a deadline); LLM pipeline SLI (% of articles processed with valid output). Use request-based budgets since traffic is spiky. As a solo engineer, page only on fast burn of the booking SLO (multi-window), and email/digest slow burns. Treat payment/booking data-integrity issues as separate hard guardrails. Alert on "no events received" so silent telemetry loss doesn't look like perfection.
