---
name: data-engineering-first-principles
description: First-principles data engineering playbook from 66 lectures and talks (Zach Wilson/DataExpert bootcamp, Seattle Data Guy, Andreas Kretz, Joe Reis). Use when designing, building, reviewing or debugging pipelines, ETL/ELT, data models (dimensional, SCD, cumulative, OBT), data quality checks, backfills, orchestration, batch vs streaming, Spark/DuckDB/Parquet performance, API extraction, or multi-source entity integration.
---

# Data engineering from first principles

This skill turns 66 lectures and talks into working judgment. This file holds the procedure,
the principles the teachers keep coming back to, and checklists. `references/` (when present)
holds detailed notes for every video, each credited to its teacher, with numbers, worked
patterns and corrections.

Use it whenever someone is moving, modelling, cleaning, integrating or serving data, even if
they don't say "data engineering": a CSV-to-database job, a supplier API sync, a dashboard
table, a dedupe of millions of rows, a nightly script.

**How to use it**
1. Follow the design procedure for any new pipeline or table.
2. Run the relevant checklists against the design or code.
3. If a `references/` folder sits next to this file, use it for depth. `references/INDEX.md`
   lists every video with its file and line number, or run
   `grep -n -i "keyword" references/*.md`. Each file starts with a cross-cutting principles
   block, then one section per video. Those sections end in "Relevance to Sam": Sam works in
   travel tech with GeoNames/OSM/GTFS location data, supplier APIs such as Viator and
   RateHawk, and millions of rows on a 16 GB laptop. Use those parts only when the task is in
   that context. Lines marked "(note: …)" and "Relevance to Sam" are helper additions, not
   the teacher's words. If `references/` is missing, work from this file and ask the user for the
   notes pack (`data-engineering-first-principles-full.zip`) when you need a teacher's detailed
   treatment.
4. Explain the *why*: name the underlying force (shuffle, IO, grain, idempotency, trust).
   Credit the teacher when you use a specific pattern so the user can watch the video.

---

## 1. Design procedure

1. **Start from the consumer and the decision.** Who reads this table (analyst, ML, app, exec,
   another pipeline)? What decision changes with it? How fresh must it be, and by when? Ask
   which fields, how often, how often the schema changes, and who owns the source.
   *(Seattle Data Guy "What to consider", Zach "model for the consumer")*
2. **Write a short spec before SQL:** purpose, flow diagram, DDL with a comment on every
   column, grain, quality checks, metric definitions with guardrails, example queries. Get it
   reviewed. *(Zach — Airbnb MIDAS)*
3. **Declare the grain of every table** and keep different grains in separate tables.
   *(Seattle Data Guy — dimensional modeling)*
4. **Pick the cheapest freshness tier that meets the SLA:** daily batch (~90% of pipelines)
   → hourly → micro-batch → streaming. "Real time" usually means "reliable by a known time".
   Stream only when latency makes or breaks the product and people exist to run it.
   *(Zach — Kafka/Flink, Pipelines in 8 min; Kretz — stream vs batch)*
5. **Lay out the layers:** raw (immutable, timestamped, no logic) → staging (cleaned,
   conformed) → production/master (audited, trusted) → aggregates/cubes → metrics.
   *(Seattle Data Guy — designing an ETL; Zach — data lake modeling continuum)*
6. **Make it idempotent and backfillable from day one** (section 2). A production run should
   just be a one-day backfill.
7. **Plan quality gates and the publish step** (write-audit-publish or a signal table).
8. **Backfill one month, have someone else validate it, then backfill history.** A buggy
   multi-year backfill is the most expensive mistake. *(Zach — MIDAS, Data Contracts)*
9. **Assign ownership and write the runbook:** primary and secondary owner, upstream teams,
   common failures, critical downstreams, SLA. *(Zach — maintaining pipelines)*

---

## 2. Principles that recur across the teachers

### Idempotency and reproducibility
- **Idempotency is the master property.** The same inputs give the same output however many
  times, on whichever day, at whatever hour you run it. Use INSERT OVERWRITE / MERGE on a
  partition, never bare INSERT INTO. Use bounded `[start, end)` windows, never `now()`. Wait
  for complete inputs (sensors). Set depends-on-past for cumulative jobs. Never read the
  "latest" partition of an upstream table. *(Zach — SCDs & idempotency)*
- **Non-idempotency fails silently and spreads.** Jobs keep "succeeding" while numbers stop
  reproducing, and every downstream table inherits the error. *(Zach)*
- **Keep raw inputs untouched and re-processable** (dated snapshots partitioned by `ds`).
  Raw lets you answer questions production can't (deleted rows, past state) and recover from
  your own bugs. *(Zach; Seattle Data Guy)*
- **Ask every source how it handles corrections.** Some append reversing rows that you net
  out. Others replace records by ID, so latest wins and you dedupe. Same-looking tables need
  different loaders. *(Seattle Data Guy)*

### Modelling
- **Conceptual before physical.** Decide what data is worth having before deciding how to
  store it. Cut low-value columns early, and fix bad logging at the source instead of
  patching downstream. *(Zach; Sami Hero with Joe Reis — conceptual modeling;
  Reis/Truta — Mixed Model Arts)*
- **Default to Kimball (facts + dimensions + SCD2) for most needs.** Use Zach's One Big Table
  (one row per entity holding an array<struct> of history, not the usual wide flat table) when long-horizon analysis would otherwise scan and join
  huge facts. Mixing paradigms is normal. *(Zach — OBT vs Kimball)*
- **Raw logs aren't facts.** A fact is atomic and deduplicated, has non-null who/what/when,
  UTC timestamps, parsed columns rather than JSON blobs, and no columns that are cheap to
  join. Prefer one line-item-grain fact over joining two big facts. Bridge tables risk fan-out
  double counting. *(Zach — Meta fact data; Seattle Data Guy)*
- **History:**
  - Use SCD Type 0 for attributes that never change and Type 2 (start/end dates + is_current)
    for change analysis. Daily full snapshots suit cheap storage, trivial backfills and as-of
    joins. Type 1 overwrites silently misattribute history and break backfills; Type 3 loses
    history.
  - Detect changes with null-safe comparisons.
  - When dimensions are small, a full rebuild with gaps-and-islands is simplest. Use an
    incremental merge when scale forces it.

  *(Zach — SCD labs; Seattle Data Guy — SCDs)*
- **Cumulative tables:** yesterday's state FULL OUTER JOIN today's data, COALESCE stable
  attributes, append capped history arrays.
  - History questions become single-partition reads with no shuffle.
  - "No data is data": only a full outer join sees absence, which gives
    new/retained/churned/resurrected labels.
  - Costs: backfills must run in sequence, arrays need a cap (~30), deletes/PII must be pruned
    deliberately.

  *(Zach — cumulative dimensions, growth accounting)*
- **Growth accounting is a general state-transition monitor.** Point it at supplier listings,
  catalog items or classifier outputs: a sudden mass transition flags a feed or model bug.
  *(Zach)*
- **Check additivity before rolling up.** A dimension is additive in a window only if an
  entity can have just one value in that window. COUNT DISTINCT and ratios can't be rebuilt
  from subtotals of non-additive dimensions. Never average averages: weight by counts.
  *(Zach; Seattle Data Guy)*
- **Multi-source integration is the real modelling problem.** The same entity appears under
  many IDs. Pick a master system or build a central ID crosswalk with deterministic,
  reproducible IDs. For many feeds into one schema, give each source an enum (fine below ~50
  values) with its own
  mapping function, per-source quality thresholds and a map column for extras.
  *(Seattle Data Guy — modeling challenges; Zach — "little book of enums")*
- **Dimension definitions are one-way doors.** Changing a core definition later costs years.
  Bucket derived dimensions from real percentiles, not invented ranges. *(Zach)*
- **Conventions are leverage:**
  - Table prefixes `fact_` / `dim_` / `scd_` / `agg_` (e.g. `dim_users`).
  - Column prefixes inside facts: `dim_country`, `m_revenue`.
  - `_ts`, `is_`, UTC stated in column comments, partition column `ds` last.
  - An `aggregation_level` column lets one aggregate table hold many grouping sets.

  *(Zach; Seattle Data Guy)*

### Quality and trust
- **Writing to production is a contract:** schema + quality checks + publish mechanism.
  - Write-audit-publish (stage with an identical schema → audit → swap/exchange partition)
    prevents most bad-data pain, at the cost of a double write and some latency.
  - A signal table is cheaper and faster but lets ad hoc users read unaudited data.
  - Classify each check as blocking or warning.

  *(Zach — Data Contracts; Sanderson & Freeman on Joe Reis's show — contracts are for software engineers)*
- **A small set of checks catches most problems:** not-empty, not-null on keys, uniqueness of
  the grain, accepted values, ranges, types, freshness, volume (week-over-week, not
  day-over-day; seasonality-aware), null-rate per field, and referential integrity for facts.
  Keep expected bands in a config table so a new check is a new row. *(Seattle Data Guy;
  Zach — tiered checks)*
- **Reconcile across layers.** Distinct keys and totals in raw should match staging/prod. A
  lost entity usually means a bad join or filter. *(Seattle Data Guy)*
- **Distrust third-party sources most.** External APIs get the most checks, pre-checks before
  the run, and a plan for the API changing or disappearing. *(Zach; Seattle Data Guy — APIs)*
- **Fail loudly.** Raise on unparseable or duplicate records, and never rely on analysts as
  the quality system. *(Zach — testing Spark jobs)*
- **Quality starts upstream.** Most breaks are semantic changes, not schema changes, so put
  checks where data leaves the producing code and invest in detecting breaks more than in
  elaborate specs. Source atomic data and push logic left; logic fixed in the BI layer gets
  trapped there. *(Sanderson & Freeman with Joe Reis — data contracts; Sami Hero with Joe Reis)*
- **Define metrics once, in versioned code, and serve them to every consumer** (BI, apps, AI
  agents) through a semantic layer. Watch for fan-out/chasm traps and ratios computed at the
  wrong grain. LLMs can't infer business definitions from raw tables, so give them governed
  measures and add deterministic checks after anything they produce.
  *(Bickell/Cube and Sami Hero with Joe Reis; Joe Reis — Great DE Reset)*
- **Metrics:** data engineers supply counts and sums at entity grain; leave ratios and
  percentiles to analysts. Every metric needs a counter-metric or guardrail. Don't put quality
  thresholds on very narrow metric cuts. Pre-aggregate dashboards with GROUPING SETS/ROLLUP
  plus an `aggregation_level` column (avoid CUBE); an analyst writing ~10 CTEs is a modelling
  smell, so materialise. *(Zach — PM-style metrics, funnel analysis)*
- **Governance formalises accountability people already have** for defining, producing and
  using data; tie it to business value. *(Bob Seiner with Joe Reis)*
- **AI raises the bar on fundamentals.** Expect agents to query 20–50x more, so cache in front
  of stores. Use AI as a reviewer, check what it writes, and don't add dependencies it
  suggests without vetting them. *(Joe Reis — Great DE Reset, Do Fundamentals Matter; Bickell)*
- **Blast radius scales with fan-out.** Heavily-used master tables need the strictest
  contracts. *(Zach)*

### Performance
- **Shuffle is the bottleneck.** SELECT/WHERE parallelise freely. GROUP BY, JOIN and window
  PARTITION BY shuffle by key. A global sort adds an extra
  range-partition shuffle (a single reducer in Hive): keep it for small final results and use
  `sortWithinPartitions` before writes. Count the
  Exchange/shuffle steps in EXPLAIN. *(Zach — Meta fact data, Spark)*
- **Choose joins deliberately.** Broadcast the small side. Bucket or hash-partition both big
  sides on the key when they're joined repeatedly. Handle skew with AQE (on by default since
  Spark 3.2; it splits skewed joins only), then memory, then two-stage salting (valid only for
  SUM/COUNT, not COUNT DISTINCT or medians) or splitting hot keys into their own path. Spark's
  broadcast auto-threshold is 10 MB; raise it or use explicit `broadcast()` up to low
  single-digit GB. *(Zach — Spark + Iceberg, High Performance Spark)*
- **Reduce volume before tuning compute.** Prune columns and partitions, keep
  low-cardinality sub-partitions, sample by entity for directional metrics (never for audits),
  build daily per-entity aggregates or reduced facts. IO dominates cost.
  *(Zach — Meta fact data, maintenance)*
- **Sort before writing; protect the sort.** Sort within partitions from low to high
  cardinality so dictionary indices run-length-encode well and min/max stats allow
  row-group skipping (if one column is filtered far more than others, sort by it first). A later shuffle scatters the
  runs. *(Zach)*
- **Materialise reused intermediates.** CTEs and views may recompute on every use
  (Spark and most warehouses inline them). Prefer
  short-retention staging tables you can inspect and partially rerun. *(Zach — CTE vs temp
  tables; High Performance Spark)*
- **Memory-bound work must be hierarchical.** A global dedup over a day OOMed even at Facebook;
  Zach's fix was hourly micro-batch dedup merged pairwise as a binary tree. On a laptop: chunk →
  dedup → merge. *(Zach — Kafka/Flink, Meta fact data)*
- **Dedup deliberately.** Check the grain with `GROUP BY … HAVING count(*) > 1`, use
  `ROW_NUMBER` with a deterministic ORDER BY, choose the dedup time window, and join on IDs,
  not names. *(Zach — Meta fact data; Seattle Data Guy)*
- **Read the query plan.** SQL goes parse → optimise → physical plan. Filter and project
  early, keep predicates sargable. *(Seattle Data Guy — query execution)*

### Pipelines, orchestration and operations
- **Choose storage and engine by access pattern, never by familiarity.** OLTP rows serve many
  small reads/writes; columnar warehouses and Parquet scan. Trickle-updating a warehouse row by
  row and running heavy analytics on the production database are the classic expensive
  mistakes. For key-value stores, list the access patterns first and design keys around them.
  *(Reis & Housley; Kretz — DynamoDB, distributed SQL; Seattle Data Guy — DBs vs warehouses vs lakes)*
- **Keep extract/load dumb and land raw first (ELT).** Transform in the set-based engine so
  transforms can be re-run without hitting sources again. Analytic copies go in Parquet, not
  JSON. *(Kretz — ETL vs ELT; Seattle Data Guy)*
- **The trigger decides stream vs batch, not the tool.** Event-triggered means streaming.
  Schedule-triggered is batch even if it reads from a queue. Match cadence to how fast the
  domain really changes. *(Kretz — stream vs batch, geospatial)*
- **Separate the control plane from the data plane.** Keep a job/file state table, so work is
  scheduled explicitly, retried, and skipped when already done. A light trigger hands work to
  heavy workers. Never do bulk work inside something with a short timeout. Put queues in front
  of anything that can be overwhelmed. Rate-limit outbound fan-out per destination. Use one
  consumer per concern (pub/sub), not one consumer doing two jobs.
  *(Kretz — Nielsen 55 TB/day, Veritiv)*
- **Size on the real unit of work.** "250B events/day" turned out to be 17M files, so
  parallelism, cost and retries were planned per file. *(Kretz — Nielsen)*
- **Pipelines are software:** git, dev/QA/prod, pytest on transforms (pure functions with tiny
  hand-built inputs, covering duplicates and nulls), CI/CD, infrastructure as code once proven.
  *(Kretz — Airflow problems; Zach — testing Spark jobs)*
- **Orchestrate dependencies as a DAG, not cron-time guesses.** Know the scheduler,
  executor, sensors, retries and pools. Keep secrets in connections or variables. Deploy
  pipeline code separately from the platform. Pass only small values between tasks.
  *(Seattle Data Guy — Airflow; Kretz — Airflow problems)*
- **Log everything and analyse your own pipelines.** Record start/end, duration, rows
  in/out/rejected and status for each step. Trend the logs to find steps that keep slowing
  down or flaking. Silent and intermittent failures are worse than loud ones.
  *(Seattle Data Guy; Kretz — monitoring pipelines)*
- **Every pipeline is a permanent liability.** Incidents scale with pipeline count, and the
  only thing better than optimised is deprecated. Consolidate duplicate sources of truth and
  set migration deadlines with no permanent `_v2`. *(Zach — maintaining pipelines)*
- **Grow the stack with maturity.** Start with ingestion → storage → transformation → BI plus
  naming standards and tests. Add a catalog, lineage, observability and a semantic layer when
  people and tables outgrow tribal knowledge. Every tool adds maintenance.
  *(Seattle Data Guy — modern data stack/catalog/platform; Bickell/Cube with Joe Reis — semantic layer; Joe Reis —
  fundamentals of data engineering)*
- **Think in the data engineering lifecycle:** generation → ingestion → storage →
  transformation → serving, with security, data management, DataOps, architecture,
  orchestration and software engineering running underneath. Tools change; these stages and
  undercurrents don't. *(Joe Reis & Matt Housley — Fundamentals of Data Engineering)*
- **CDC when you need low latency or every intra-day change.** A daily snapshot loses
  intermediate states. Use log-based CDC (WAL/binlog) or a managed tool. *(Seattle Data Guy)*

---

## 3. Checklists

**New table / model**
- [ ] Consumer, decision, SLA and owner named; spec reviewed.
- [ ] Grain declared; one grain per table; key uniqueness test.
- [ ] Facts atomic and deduped with UTC timestamps; dimensions SCD2 or snapshot, never
      silent overwrite.
- [ ] Additivity of every metric/dimension checked; no averaged averages.
- [ ] Naming conventions; every column commented; partition column `ds`.
- [ ] Multi-source IDs resolved through a deterministic crosswalk.

**Pipeline**
- [ ] Idempotent: OVERWRITE/MERGE by partition, bounded windows, no `now()`, no "latest"
      partition reads, sensors on complete inputs.
- [ ] Raw layer immutable and timestamped; reruns start from raw.
- [ ] QA gate after each logic step; blocking vs warning decided; write-audit-publish for
      important tables.
- [ ] Layer-to-layer reconciliation (counts, distinct keys, totals).
- [ ] Per-step logging of rows in/out/rejected and duration. At minimum, alert on failure,
      duration overrun and schema change, sent to a channel someone watches *(Kretz —
      monitoring)*; add volume-deviation alerts *(Seattle Data Guy)*.
- [ ] Job/file state table so reruns skip finished units.
- [ ] Backfill tested on one month, validated by someone else, before full history.
- [ ] Retries are safe; credentials in config/secret store; runbook written.

**API extraction** *(Seattle Data Guy — APIs; Zach — data quality, Spark + Iceberg)*
- [ ] Auth type handled, with rotation via config.
- [ ] Pagination by cursor/next-link (offset skips or duplicates when rows are inserted
      mid-crawl).
- [ ] Rate limits and silent row caps known; fan-out of nested calls estimated.
- [ ] Raw responses landed as dated files before parsing; schema drift detected.
- [ ] Completeness validated after each pull (expected vs received counts).
- [ ] Prefer bulk exports or a direct DB feed over per-record REST at scale.
- [ ] One generic extractor driven by a config/control table; cursors persisted so restarts
      resume. *(Seattle Data Guy; Zach)*

**Performance (Spark, warehouse or laptop)**
- [ ] Shuffles counted in EXPLAIN; small sides broadcast; big-big joins partitioned by key.
- [ ] Skewed keys found and handled.
- [ ] Only needed columns read; partition pruning works; Parquet sorted low → high cardinality.
- [ ] Reused intermediates materialised; no `collect()` of big data to the driver.
- [ ] Memory: chunked/hierarchical processing, spilling allowed.

**Streaming (only if the SLA really needs it)**
- [ ] Event-time watermarks; checkpoints; explicit offset reset policy.
- [ ] Parallelism matched to partitions; no slow UDFs in the hot path.
- [ ] Late and duplicate events handled; consumers idempotent.
- [ ] A batch path for backfills (e.g. Iceberg appends readable in batch).

---

## 4. Laptop-scale translation (16 GB RAM, DuckDB / Polars / Parquet)

The teachers mostly speak in Spark/warehouse terms. The helpers' "laptop translation" notes
map them to single-machine work:
- Store as Parquet, partitioned by date and/or a low-cardinality key, sorted low → high
  cardinality. Read only needed columns.
- Small lookups = broadcast joins. For big-to-big joins, first let DuckDB run them out of core
  (`SET memory_limit`, `temp_directory`, `preserve_insertion_order=false`); only if that still
  fails, hash-partition both sides into the same number of buckets and join bucket by bucket.
- Don't load everything into pandas. In Polars use lazy frames with
  `collect(engine="streaming")`.
- Pre-flight: check free RAM and disk (for spill) before choosing chunk sizes. *(Kretz —
  pre-flight checks)*
- Dedup in chunks then merge; run cumulative and SCD logic as daily FULL OUTER JOINs over
  date-partitioned files.
- Process in checkpointed chunks with atomic file writes (write temp, then rename) so a crash
  resumes instead of restarting.

---

## 5. Failure modes to hunt for in reviews

- INSERT INTO without overwrite: reruns duplicate data.
- Pipelines that read `max(ds)` or call `now()`: not reproducible.
- Type 1 overwrite of an attribute later used for history.
- Mixed grains in one table hidden by AVG/DISTINCT in the BI tool.
- LEFT JOIN that silently nulls columns, or a join that fans out and double counts.
- Offset pagination on a live API; unvalidated row counts from a capped endpoint.
- Averaging averages; ratio metrics summed across non-additive dimensions.
- Global sort, `collect()` or a big shuffle where a broadcast or pre-aggregation would do.
- A full multi-year backfill run before a one-month validation.
- Streaming built for a decision that's made daily.
- A second "source of truth" table built instead of fixing the first.

---

## 6. Where to look in `references/`

| Need | File (see INDEX.md for line numbers) |
|---|---|
| Fact data at Meta scale, funnels, Spark + Iceberg, data lake modelling, SCD lab, additive dims/graphs/enums, pipeline types | `zach-a-*` |
| Modelling basics, CTE vs temp tables, cumulative lab, OBT vs Kimball, testing Spark, MIDAS, growth accounting, Kafka + Flink | `zach-b-*` |
| Cumulative tables & complex types, SCDs & idempotency, data contracts/WAP, data quality, high-performance Spark, PM-style metrics & experiments, maintaining pipelines | `zach-c-*` |
| ETL layering, QA, dimensional modelling, SCDs, integration, APIs, Airflow, CDC, dbt, query execution, catalogs, platforms | `seattle-data-guy-*` |
| Stream vs batch, real-world architectures, Pinot, distributed SQL, monitoring, Airflow problems, DynamoDB design, ETL vs ELT, geospatial; Joe Reis on fundamentals, contracts, conceptual modelling, governance, semantic layer | `kretz-and-reis-*` |

Lines marked "(note: …)" are corrections or caveats to what a teacher said. Prefer the
corrected version.
