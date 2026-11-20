# zach-3 — cross-cutting principles

1. **Idempotency is the master property:** same inputs give the same output, regardless of how many times, which day, or what hour you run it; production run = backfill. Use INSERT OVERWRITE/MERGE (never bare INSERT INTO), bounded `[start, end)` windows, full input sensors, depends_on_past for cumulative jobs, and never read the "latest" partition of an upstream table. (Zach — SCDs & Idempotency, Day 2)
2. **Non-idempotency fails silently and transitively:** jobs keep succeeding while the numbers stop reproducing; unit tests can't catch it; every downstream table inherits it; cumulative tables carry it forward forever. (Zach — Day 2)
3. **Writing to production is a contract:** schema + quality checks + publish mechanism. Write-audit-publish (staging with an identical schema, then audits, then partition exchange) prevents about 80–90% of bad-data pain. Classify checks as blocking vs warning. (Zach — Data Contracts; Data Quality 2022)
4. **WAP vs signal table is a real trade-off:** WAP guarantees ad hoc users never see unaudited data but pays write IO twice and adds latency; signal tables are cheaper and faster but let analysts query bad data. (Zach — Data Contracts; Maintenance)
5. **Distrust third-party sources the most:** put more checks on external APIs than on internal data, run pre-checks before the pipeline starts, and plan for the API changing or disappearing. (Zach — Data Quality 2022; Data Contracts; Maintenance)
6. **Validate small, then backfill big, and have someone else validate:** backfill about one month, chart every assumption (nulls, dupes, enums, row counts), then backfill history. Authors are blind to their own data, and buggy multi-year backfills are expensive. (Zach — Data Contracts; Maintenance)
7. **Blast radius scales with fan-out:** a master table with 1k–20k downstreams demands the strictest contracts. Fixing propagated bad data meant re-running secondary and tertiary consumers (about $1M at Facebook). (Zach — Data Contracts)
8. **Model for the consumer along the OLTP → master data → OLAP cube → metrics continuum:** nested types for data-engineer consumers, flat columns for analysts and ML, charts for executives. (Zach — Complex Types & Cumulation, Day 1)
9. **Cumulative tables (yesterday FULL OUTER JOIN today, COALESCE, append history arrays) turn history questions into single-partition SELECTs:** they enable state-transition metrics and cut IO (1 partition instead of 30 for MAU). Costs: sequential backfills, explicit pruning and deletions. (Zach — Day 1; Maintenance)
10. **For changing dimensions use only SCD type 0 or type 2** (start/end dates + is_current), or Max Beauchemin-style daily snapshots. Type 1 and "latest snapshot" break backfills; type 3 loses history. (Zach — Day 2)
11. **Sort once upstream and protect the sort:** Parquet RLE/dictionary compression depends on order. Joins and shuffles destroy order, so either model as id + array<struct> (join, then explode stays sorted) or `sortWithinPartitions` before writing. Never use a global sort for this. (Zach — Day 1; High-Performance Spark)
12. **Shuffle is the enemy at scale:** broadcast the small side (explicit `broadcast()`, up to single-digit GB), bucket both sides of huge joins on the key (only above about 1TB), handle skew with AQE, then memory, then salting. (Zach — High-Performance Spark; Maintenance)
13. **Materialise reused intermediates:** temp views and CTEs recompute on every use. Cache in memory only when reused at least twice; if you'd cache to disk, write a real staging table instead (inspectable, enables partial reruns). (Zach — High-Performance Spark)
14. **IO dominates cloud cost** (about 85% in his Airbnb savings): shrink heavily-read master tables, use low-cardinality sub-partitions (at most about 30 values) for pruning, sample by entity for directional metrics (never for audits), and kill duplicate data models. (Zach — Maintenance)
15. **Every pipeline is a permanent liability:** failure incidents scale with pipeline count, so "the only thing better than optimized is deprecated." Consolidate multiple sources of truth through stakeholder conversations and a signed-off spec. (Zach — Maintenance)
16. **Operational hygiene:** primary and secondary owner; runbooks listing upstream TEAMS, common issues, critical downstreams and SLAs; SLAs negotiated from measured delay impact; document every failure; dedicated tech-debt time; aggressive migration deadlines (no permanent `_v2` tables). (Zach — Maintenance)
17. **Data engineers supply simple aggregates and counts at entity grain:** leave ratios and percentiles to analysts. Prefer counting people over actions for skewed metrics; ratio metrics are non-additive across overlapping dimensions. (Zach — PM: KPIs & Experiments)
18. **Every metric needs a counter-metric or guardrail and a place in the funnel:** examples are reachability vs notification blasts and AWS cost vs refresh rate. Guardrail metrics owned by data engineering must be WAP-grade quality and fresh. Don't put quality thresholds on over-narrow metric cuts. (Zach — PM: KPIs & Experiments; Data Contracts)
19. **Experiment hygiene:** all logging in place before launch, stable identifiers, enough power (few cells), run long enough to outlast novelty effects, 95% confidence, winsorize outliers, and judge practical rather than just statistical significance. (Zach — PM: KPIs & Experiments)
20. **Pragmatism over perfection:** a pipeline needn't be a Ferrari (opportunity cost), infrastructure is a marathon not a sprint, and ownership problems are solved organisationally, not by heroics. (Zach — Day 2; Maintenance)

## Zach Wilson (DataExpert) — Why data engineers should care about data quality (and how to do it right) (https://www.youtube.com/watch?v=GKQTp2TvWGY)
**Bad data produces bad decisions at scale; catch it before it reaches production with write-audit-publish.**
- Mental model: data exists to drive decisions. Low-quality inputs degrade three kinds of decision-makers: executives (few, expensive decisions), data scientists running experiments (a wrong call can flip), and ML models (millions of automated low-quality decisions per day; the worst amplifier).
- Three classes of quality problem:
  - Incorrectness: duplicates, unexpected nulls, wrong values, schema non-conformance, abnormal row counts.
  - Incompleteness: data can be correct yet missing a causal factor (his example: Zillow's home-pricing models). However much you gather, the data is never complete; stay humble when "data-driven".
  - Design problems: the question is answerable but very expensive to answer. He treats efficient access patterns (cumulative tables, arrays of compacted history) as part of quality, since a dataset's usability is its UX.
- Root causes:
  - Bad software pushes: a logger starts double-logging, or a schema change silently drops a field so the column goes all-null.
  - Race conditions in the production OLTP database (two concurrent writes) create duplicates there, which the lake snapshot inherits.
  - Third-party APIs that break their contract. Rule: put MORE checks on third-party sources than on internal ones, because you control neither their schema nor their timing.
- Mechanism (write-audit-publish, used at Netflix/Airbnb): write the day's output to a staging table, run audit queries (nulls, duplicates, week-over-week row-count growth within an expected band, neither too slow nor too fast), and only on pass move staging into production. Downstream jobs read only production.
- Tools named (2022): Amazon Deequ for Spark, Apache Griffin for streaming, Great Expectations for warehouse batch (Snowflake/BigQuery); all can implement WAP. (note: in 2026, similar checks are easily written as plain SQL asserts in DuckDB/dbt tests, or with Soda; the pattern matters more than the tool.)
- Relevance for Sam: supplier feeds (Viator, RateHawk, etc.) and open datasets (GeoNames/OSM/GTFS) are all "third-party", so they need the heaviest checks: schema, null-rate per key column, duplicate primary keys, and row count vs the previous load.

## Zach Wilson (DataExpert) — Master Data Contracts in 25 minutes! (Boot Camp Week 3 Day 2 Analytics) (https://www.youtube.com/watch?v=ja7VOqDVkuo)
**Writing to production is a contract (schema + quality checks + how data gets published); write-audit-publish enforces it and prevents most quality pain.**
- Claim: just doing write-audit-publish (WAP) prevents roughly 80–90% of the headaches from publishing bad data (lost analyst trust, wasted debugging queries).
- Causes of bad data:
  - Logging errors (mostly affect fact data). Examples: a log schema change no longer matches the ingest table, so nulls appear; a submit button not disabled during submission lets double-clicks create two "form submitted" events.
  - Snapshotting errors (dimensional data). Rare (he hit it less than once a year): users/dimensions missing or extra.
  - Bad data already in the production app DB: you have to filter it, and usually work with the software engineers to fix it.
  - Schema evolution on either the logs or the production tables.
  - Pipeline logic mistakes (bad joins, CASE WHEN).
  - Non-idempotent pipelines that corrupt data when backfilled.
  - Most common of all: insufficient validation before release.
  - Third-party APIs (Salesforce, Stripe, Slack...) changing their contract with no recourse; be especially careful there.
- Validation best practice (Airbnb):
  - For a new pipeline, backfill a small slice (about one month) first.
  - Check every assumption (nulls, duplicates, enum values, time-series shape, row counts) and record it as a doc of charts.
  - Only then backfill the full history.
  - Have SOMEONE ELSE validate. He failed to validate his own Airbnb pricing pipeline even as a staff engineer: authors are blind to their own data.
- A contract has three parts:
  - Schema (columns/types).
  - Quality checks. In a data lake you lose RDBMS constraints (UNIQUE, NOT NULL), so they must be re-implemented as explicit checks: row count, duplicates, not-null, etc.
  - How data appears in production. This is where the two patterns below differ.
- WAP mechanism:
  - The pipeline writes to a staging table with the IDENTICAL schema to production.
  - Run the checks.
  - On pass, exchange partitions into production. Production landing is itself the signal for downstream pipelines.
  - Checks are blocking or non-blocking:
    - Non-blocking (call it "data weirdness"): alert, but publish anyway.
    - Blocking: halt and troubleshoot manually.
  - A blocking failure can still be real: when Ethiopia's government cut internet access, Facebook users there dropped sharply. They confirmed it against world events, then exchanged the partition anyway.
- Signal-table pattern (Facebook):
  - Write directly to production, run the checks on production, and on pass write a row to a "signal table" that downstream jobs wait on.
  - Pros: simpler, less compute/IO, lands sooner, more likely to hit the SLA.
  - Cons: ad hoc users (analysts, data scientists) query production, never the signal table, so they can read unaudited data. Engineers may also wrongly wait on production instead of the signal.
- WAP trade-off: guarantees nothing reaches production unaudited, but the staging-to-prod exchange adds delay (usually minutes). It can be near-instant if the exchange is a metadata or folder rename (e.g., S3 partition swap). His preference: WAP.
- Bad-data propagation:
  - Pipelines form a tree, so blast radius scales with downstream fan-out.
  - Dashboard-only output: low stakes.
  - Airbnb pricing/availability tables: over 1,000 downstream pipelines.
  - Facebook dim_all_users: 10–20k downstreams. A compute-engine migration bypassed the contract, and the cleanup (fix the source, find and re-run secondary and tertiary consumers, roughly doubling warehouse compute for that day) cost about $1M.
  - Publishing bad data twice on a critical table is a firing offence. The more critical the dataset, the stricter the contract.
- Metric/quality-check gotcha: do not put quality thresholds on over-narrow metric cuts. Each added dimension shrinks the population until normal variance looks like a 30% drop (e.g., 7 users become 5). Keep checked metrics broad, with at most about 1–2 dimensional cuts.
- Relevance for Sam:
  - Implement WAP locally with Parquet: write to `staging/date=X/`, run DuckDB assertions (count(*) vs previous, count(distinct id)=count(*), null rates), then an atomic `os.replace`/directory rename into `prod/date=X/`. That is a cheap partition exchange on a laptop.
  - Classify checks as blocking (duplicate primary keys, schema drift on supplier feeds) or warning (row-count wobble).

## Zach Wilson (DataExpert) — Data Modeling: Complex Data Types and Cumulation (Day 1) (https://www.youtube.com/watch?v=5U-BbZ9G_xU)
**Model for your consumer: hold complete entity history in cumulative master tables, and use arrays/structs to keep data compact and sorted.**
- Dimensions are attributes of an entity.
  - Identifier dimensions uniquely identify it (user_id, device_id).
  - Attributes are either fixed (birthday, phone manufacturer: trivial to model) or slowly changing (favourite food: time-dependent, and the hardest part of dimensional modeling).
- Know your data customer (the empathy step LLMs are worst at):
  - Analysts/data scientists: flat, easy columns (numbers/strings) that are trivially summed and grouped (an "OLAP cube").
  - Other data engineers consuming "master data": nested structs/arrays are fine, they can handle them.
  - ML models: identifier plus flat feature columns; naming matters less.
  - Customers/executives: annotated charts, never raw tables.
- The modeling continuum: OLTP, then master data, then OLAP cube, then metrics.
  - OLTP (software engineers): 3NF, minimise duplication, constraints, many joins, optimised for fetching ONE entity fast.
  - OLAP: optimised for scanning populations; tolerate duplication so queries avoid joins and can just GROUP BY.
  - Master data: deduplicated, complete, still fairly normalised; the trusted middle layer. His Airbnb pricing/availability master table merged about 40 transactional tables into one.
  - OLAP cube: flattened, possibly many rows per entity, for slice and dice.
  - Metrics: aggregate down to single numbers (40 tables become 1 number).
  - Mismatch symptoms: OLTP modeled like OLAP gives a slow app (fetching unneeded wide rows). Analytics run straight on a dumped OLTP schema means join and shuffle hell on every query.
- Cumulative table design (Facebook dim_all_users, about 10k downstream pipelines):
  - Inputs: yesterday's cumulated table (null on day 1) and today's daily snapshot, with the same columns.
  - FULL OUTER JOIN them, COALESCE the ids and values, compute cumulative metrics (days since last active, increment if still inactive), and append today's values to history arrays.
  - The output becomes tomorrow's "yesterday".
  - Enables growth accounting / state transitions from yesterday vs today: new, churned, resurrected, retained.
  - Prune stale entities with an explicit rule (Facebook: inactive more than 180 days), or the table grows forever.
  - Strengths: historical questions ("active in the last 90 days?") become a single-partition SELECT, with no GROUP BY over 90 daily partitions and no shuffle.
  - Weaknesses: backfill must be SEQUENTIAL (each day depends on the previous one), unlike daily tables, which backfill in parallel. Deleted users / PII must be actively filtered, usually by joining a deletions table.
- Compactness vs usability:
  - Most usable: flat id plus primitives (OLAP, analysts).
  - Most compact: id plus an encoded/compressed blob. Example: Airbnb ships the availability calendar to the app as a compressed blob to minimise network IO; analysts can't use it, so he built decoded tables.
  - Middle: arrays/maps/structs, the master-data layer.
- Complex types:
  - Struct: fixed keys, values may differ in type ("table within a table").
  - Map: arbitrary number of keys but ONE value type, which causes casting pain. Key-count limits are around 32k–65k depending on engine.
  - Array: ordered, single element type, but elements can be structs/maps, so nesting is allowed (array<struct> is the workhorse).
- Temporal cardinality explosion: Airbnb listings (about 6M) × next 365 nights is about 2B listing-night rows. The choice is one row per listing with an array of nights vs one row per listing-night.
  - If sorted, Parquet run-length encoding makes both roughly the same size on disk, because the repeated listing_id collapses into one value plus a count.
  - Gotcha: a JOIN (shuffle) destroys the sort order, so RLE stops working and the output silently balloons. Many engineers never notice.
  - Fix options: tell every consumer to re-sort after joins (fragile), or model as id + array<struct> so consumers join on the id and then EXPLODE. The exploded rows stay contiguous and sorted.
  - Principle: sort once upstream and make downstream misuse impossible.
- Relevance for Sam:
  - For location/transport data (e.g., stop × service-day, or place × name variants), consider one row per entity with an array of child structs, or explicitly ORDER BY the entity key before writing Parquet. DuckDB/Polars hash joins also reorder rows, so re-sort before `COPY ... TO parquet` to keep RLE/dictionary compression effective.
  - Cumulative tables suit things like "supplier product last seen / first seen / days missing" tracking across daily API pulls.

## Zach Wilson (DataExpert) — Data Modeling: Slowly Changing Dimensions and Idempotency (Day 2) (https://www.youtube.com/watch?v=emQM9gYh0Io)
**An idempotent pipeline is a pure function of its inputs: production run = backfill = rerun; model changing dimensions as SCD type 2 (or daily snapshots) to keep it that way.**
- Definition (data-engineering sense): the same output regardless of how many times you run it, on which day you run it, or at what hour, provided all inputs are present. Production and backfill must produce identical data.
- Why non-idempotency is insidious:
  - Nothing fails; the output is simply non-reproducible. It surfaces weeks later as "why don't these two numbers match?" from analysts.
  - It is transitive: any table built on a non-idempotent table inherits the problem.
  - Unit tests still pass, because they can't reproduce the time-dependent behaviour.
  - Cumulative pipelines built on a non-idempotent input carry the bug forward every day, so you must restart the cumulation from scratch.
- Causes and fixes:
  1. INSERT INTO without TRUNCATE: each rerun duplicates rows (N runs = N× rows). Rule: never use bare INSERT INTO; use MERGE (rerun matches and changes nothing) or, better, INSERT OVERWRITE of the partition. The Hive-era "everything is insert overwrite" convention exists for exactly this reason.
  2. `date > X` without an upper bound: each later run picks up one more day. Always process a bounded window (`start <= date < end`). An unbounded filter can also OOM during backfills.
  3. Missing partition sensors: production fires before all inputs land, while the backfill sees complete inputs, so the results differ. Wait on the FULL set of inputs.
  4. Cumulative pipelines without depends_on_past / sequential processing: a parallel backfill reads a not-yet-built "yesterday", silently restarts the cumulation, and you get only today's data.
  5. Relying on the "latest" partition of an upstream table instead of the run date's partition. His Facebook story: dim_all_fake_accounts read the latest dim_all_users to land sooner, so some days it used yesterday's users. He spent over a month chasing irreproducible inflow/outflow numbers and quit about 6–7 weeks later. Never trade correctness for landing time this way.
     - Only exception: when backfilling, reading the latest snapshot of a properly modeled SCD table is OK.
- Dimension types by change rate: fixed (birthday), slowly changing (country, phone OS, favourite food), rapidly changing (heart rate, effectively a metric). The slower the change, the bigger the SCD compression win. If something changes about weekly, just use daily snapshots.
- Three ways to model changing dimensions:
  - Latest snapshot only: NEVER for analytics. A backfill stamps today's value on history (e.g., age 29 attached to posts written at 18).
  - Daily partitioned snapshots (Maxime Beauchemin's "functional data engineering" view): the full dimension copied each day. Storage is cheap; quality bugs are expensive; simple and idempotent.
  - SCD: collapse consecutive identical daily values into one row with a validity range (365 rows become 1).
- SCD types:
  - Type 0: truly immutable attribute; just id plus value. Idempotent.
  - Type 1: overwrite with the latest value. Fine for OLTP apps, which only need current state; NOT idempotent for analytics. His most hated.
  - Type 2 ("gold standard" at Airbnb): one row per value with start_date and end_date.
    - The current row's end_date is a far-future sentinel (9999-12-31) or NULL (debated), plus an is_current boolean.
    - Point-in-time lookup: `start_date <= d AND d < end_date` (or `<=`, by convention). Idempotent if the dates are handled carefully.
  - Type 3: original + current value only (optionally with change dates). One row per entity, but history is lost after a second change, so it's not idempotent.
  - Types 4–6 are rare. Practical rule: only use type 0 and type 2.
- Loading SCD2:
  - (a) Full recompute: one query that collapses all daily snapshots each run.
  - (b) Incremental: yesterday's SCD plus today's snapshot (cumulative style), processing one new day.
  - Prefer incremental for production, but for small data both are similar. His Airbnb unit-economics SCD (payments later refunded) reprocessed everything since 2016 daily and was never optimised. Rule: a pipeline needn't be a Ferrari; weigh optimisation against opportunity cost.
- Relevance for Sam:
  - Supplier pulls and location refreshes should write `date=YYYY-MM-DD` partitions via overwrite (e.g., DuckDB `COPY ... (PARTITION_BY date, OVERWRITE_OR_IGNORE)` or writing to a temp dir then atomic rename), never append.
  - Every job takes an explicit `[start, end)` window parameter.
  - Hotel/attraction attributes from suppliers (name, category, coordinates, price band) are SCD2 candidates (`valid_from`, `valid_to`, `is_current`), so a backfill or audit can reconstruct what was shown on a given date.
  - A GeoNames snapshot per release is the "daily snapshot" approach and is acceptable given cheap disk.

## Zach Wilson (DataExpert) — High Performance Spark in 1 hour: DataFrame, Dataset, UDFs, Caching (Week 3 Day 2) (https://www.youtube.com/watch?v=JriG4d9o2Mk)
**Spark performance comes from avoiding recompute and shuffle (cache what you reuse, broadcast small sides, bucket big sides, sort within partitions), and from choosing the API by who maintains the code.**
- spark-submit ("Spark server") vs notebooks:
  - spark-submit starts a session per job and kills it at the end, so dev behaves like prod. Slower to submit because fat jars with many dependencies can take minutes to upload. Airbnb's way and his preference.
  - Notebooks keep a live session, so cached state makes reruns look faster than production. Gotcha: call `unpersist()` before timing.
  - Danger: scheduled notebooks (Netflix pioneered, Databricks enables) mean an edit goes straight to production with no review. Insist on CI/CD for any code that writes production data; everyone ships bad changes eventually, so build the checks.
- Temp views ≈ CTEs: they are RECOMPUTED on every downstream use unless cached. His Airbnb unit-economics job reused a temp view 7 times; adding `.cache()` took it from about 3h to about 10min.
- Caching rules:
  - `.cache()` = `persist(MEMORY_ONLY)`. It only pays off if the dataset is used at least twice; the first (cold) use is actually slightly more work.
  - Disk caching is effectively a throwaway materialized view. If you need it, write a real staging table instead (`write.mode("overwrite").saveAsTable(...)`): you can inspect intermediate results, a failed job leaves evidence, and backfills can rerun just the second step. He has never seen disk-only persist be the right answer.
- Caching vs broadcast:
  - A cached dataset stays partitioned. Example: 100GB over 200 partitions is about 500MB per task and about 2GB per executor, so huge sets can be cached.
  - A broadcast ships the ENTIRE small dataset to every executor (effectively one partition), so it must be small.
- Broadcast join (the most important optimisation):
  - Avoids shuffling the small side. Auto-triggered under `spark.sql.autoBroadcastJoinThreshold` (default 10MB). Can be raised (he has gone about 200× higher) up to single-digit GB with enough executor memory; never tens of GB or TB.
  - Prefer explicit `broadcast(df)`: it documents intent and doesn't break silently when the table grows past a threshold.
  - Setting the threshold to -1 disables it, which is useful for testing other join strategies.
- UDFs:
  - A Python UDF serialises rows JVM to Python and back (4 serde hops). Apache Arrow (pandas/vectorised UDFs) made Python UDFs roughly on par with Scala for row-level UDFs. Python UDAFs (aggregate functions) still lag.
  - Most pipelines (he says about 1 in 100 use UDFs) never need them; prefer built-in functions.
- PySpark vs Scala Spark: Scala gives (1) faster UDF/UDAF and (2) the Dataset API. These matter only at extreme scale. Learn PySpark unless you already know Java (Scala compiles to the same JVM bytecode).
- API choice (continuum):
  - Spark SQL: many cooks / data scientists, fast iteration, lowest barrier.
  - DataFrame API: long-lived PySpark pipelines; modular and unit-testable functions.
  - Dataset API (Scala, Airbnb's default): typed case classes. Non-`Option` fields are guaranteed non-null and the job fails loudly with an NPE otherwise, so it doubles as an in-pipeline quality check. The same case classes generate fake test data, so no separate mock schemas are needed. Lets you write pure-function `map`/`filter` without registering UDFs.
- Parquet/Iceberg and sorting:
  - Run-length encoding rewards sorted data (smaller storage, cheaper reads).
  - NEVER use global `.sort()`/`orderBy` for this: it adds an extra shuffle that funnels data toward a single ordered output.
  - Use `.sortWithinPartitions()`: a local sort, nearly free, and captures most of the RLE benefit.
- Tuning:
  - Don't default executors to the 16GB max "to be safe"; that's wasteful, so think first.
  - Driver memory only matters for `.collect()` or very complex plans.
  - Shuffle partitions (default 200): target about 100–200MB per partition/output file. Example: 20GB output / 100MB is about 200 partitions. (note: the transcript says "2,000", but the arithmetic gives 200, and 2,000 would be about 10MB files.)
  - Empirically try the estimate at -50%, 1×, and +50% and keep the clear winner; IO-, network- or memory-heavy jobs shift the optimum.
  - AQE (adaptive query execution) handles skew automatically; he says enable it when you know the data is skewed. (note: AQE is on by default since Spark 3.2.)
- Bucket joins (lab, Iceberg):
  - Write both tables bucketed on the join key with the same bucket count (e.g., `bucket(16, match_id)`, plus a date partition on one).
  - The join then shows no Exchange (shuffle) in the physical plan, just a bucketed batch scan feeding a sort-merge join within matching buckets.
  - Partitioning by date × 16 buckets produced about 3,000 files, so watch the small-file explosion.
  - Use `explain()` and diff the plans to verify that a cache/bucket/broadcast actually took effect.
- Relevance for Sam (laptop scale):
  - The same ideas transfer to DuckDB/Polars: materialise reused intermediates to Parquet staging files instead of recomputing CTEs.
  - Join small dimension tables (country codes, admin codes, supplier mappings) as in-memory hash builds.
  - Write Parquet sorted by the main filter/join key (e.g., country_code, geonameid) with about 100–250MB row-group-friendly files.
  - Pre-partition/"bucket" huge inputs by key hash (e.g., `hash(id) % 16`) into separate files so very large joins can run bucket-by-bucket within 16GB RAM.
  - Avoid Python row-wise UDFs; use vectorised expressions.

## Zach Wilson (DataExpert) — Data Engineering like a Product Manager: KPIs & Experiments (Week 5) (https://www.youtube.com/watch?v=ZRmBIktFyDI)
**A data engineer's impact comes from metrics that change decisions: supply simple, trustworthy aggregates, protect them with counter/guardrail metrics, and place every metric somewhere in the user funnel.**
- Why metrics matter: they provide visibility (fewer "icebergs"). How much weight they carry depends on founder culture: Facebook was purely number-driven, while at Airbnb you also needed a narrative arc.
- Metrics drive the data model. The aggregations in the spec determine the table grain. Ratios and anything beyond count/sum complicate modeling.
- Division of labour (key rule):
  - Data engineers supply aggregates and counts at the entity grain, e.g., a daily metrics table of (entity_id, metric_name, value, date).
  - Push back when data scientists ask you to bake rolling-average/percentile/ratio logic into pipelines; that's their layer.
  - Reasons: you won't be rewarded for doing data science in a data-engineering review; you'll field statistical questions you'd rather not own; and "if you define a metric you must answer every question about it", so keep the ones you own simple. Statistics needs roughly 2–3/10 for a data engineer.
- Metric types:
  - Counts/aggregates (the Swiss-army knife): counts of actions vs counts of PEOPLE. Use people-based binary metrics (did_receive_notification) to neutralise skew; a viral user can have 100× the actions.
  - Ratios measure quality: CTR, purchase rate, CAC (e.g., $1,000 of ads for 10 users = $100 per user). They are non-additive across dimensions where entities overlap: a user on both Android and iOS makes per-OS conversion rates fail to roll up to the total. Leave numerator/denominator inclusion rules to data science.
  - Percentiles find tails, where incremental impact lives:
    - p99 latency (Netflix aimed for 99.99% uptime, about 52 minutes of downtime a year; the transcript says "25 minutes").
    - Facebook's low-end-device latency data led to Facebook Lite (about 90% smaller install), which gained hundreds of millions of users.
    - Low-percentile engagement shows the lurkers.
    - Supply daily aggregates; let data scientists compute percentiles.
- Metric gaming and counter-metrics:
  - Notification "blasts" (10× volume) raised short-term growth, but users turned notifications off. He created a "reachability" metric (% of users you can still notify); after that, experiments shipped only if growth rose AND reachability was neutral or up.
  - Netflix background feed refresh: more frequent refresh meant more retention, but higher AWS cost. He added AWS cost per A/B cell as a counter-metric so an ROI decision could be made, given diminishing (logistic) returns.
  - Denominator fiddling (engagement per DAU vs per MAU) and p-hacking can make experiments say anything. Fix the hypothesis and metrics upfront.
  - Novelty effect: an emoji in notifications gave a 15–20% lift in week 1, then decayed. Run experiments long enough (big tech: at least about a month).
- Experiment mechanics:
  1. Hypothesis: null = no difference. You reject or fail to reject the null; you never "prove" the alternative.
  2. Group assignment:
     - Exclude long-term holdouts (e.g., users who never get notifications, used to measure a whole product's value), out-of-scope geographies, and ineligible platforms.
     - Choose allocation: big tech starts at 0.5–1% to cap downside; small companies need about 100% for power.
     - Identifiers: logged-in user_id is richest. Logged-out options: hashed IP (one user on mobile↔Wi-Fi becomes two; two users on one Wi-Fi become one) or a stable device ID.
  3. Collect data: smaller effects need longer runs. Too many test cells underpower each one; Google's 41 shades of blue (about 2.5% each) only works at Google scale. Put ALL logging in place before launch; adding a metric mid-experiment skews results.
  4. Analyse:
     - A CI overlapping 0 is not significant. Standard alpha is 0.05 (95%); 80% CIs are weak.
     - Statistical significance does not mean practical value: long-running tests make tiny deltas "significant".
     - Results can differ by segment (significant in the US, not in India/UK; prices in INR would help India and hurt the US).
     - Outliers (celebrities in control) skew means. Winsorize, i.e., clip at about the 99.9th percentile, or count users instead of events.
- Server-side vs client-side logging: server-side is easy and trusted, so start there. Client-side gives higher fidelity (actual click time, scroll/dwell, "time spent") but needs auth so random clients can't log arbitrary events.
- Feature gates vs experiments: gates are yes/no splits that can be edited anytime (rollouts, long-term holdouts); experiments have fixed multi-cell configuration and decision workflows.
- Data-engineering role in experimentation:
  - Pipe ETL'd user-level metrics/dimensions (daily metrics, SCDs) into the experimentation platform; he did this at every big-tech job. Airbnb's metric repository is "Minerva".
  - Own guardrail metrics (e.g., Airbnb reservation profitability). An experiment that significantly hurts a guardrail does not ship, so guardrails must be very high quality (WAP) and very fresh. State-change metrics (churn, fake accounts) are typical guardrails.
- Leading vs lagging: revenue/cost is lagging. Inputs (impressions, visits, signups) are leading and faster to move. Every metric should ultimately tie to making or saving money (non-profits: still cost efficiency). Full-funnel conversion rate = bottom / top; analyse every step, since the problem may be top-of-funnel volume rather than conversion.
- The product funnel: impressions/ads → clicks/visits → signup (capture a contact point such as email, so users can re-enter deeper in the funnel) → purchase (pricing, elasticity, decoy pricing) → engaged/retained (onboarding, re-engagement notifications) → referral/testimonial/LTV.
  - Each step has a metric and is experimentable.
  - Attribution: track the referrer/UTM down to the content-piece level, not just the platform.
  - Optimise page speed and responsiveness for low-end devices.
- PM lens for data engineers: at each funnel step ask "what causes pain, what sparks joy?". Any data work you can't place in the funnel is likely waste; infrastructure-cost work maps to CAC.
- (Skipped: career-funnel and boot-camp self-promotion segments, beyond noting that the same leading/lagging thinking applies to a job search: effort → brand/network → interviews → success → giving back.)
- Relevance for Sam (Trameter):
  - Model a travel booking funnel: search impressions → product views → Drive-Button clicks → supplier availability check → booking → cancellation/refund. Supply daily per-user/per-product aggregates rather than ratio logic.
  - Add counter-metrics, e.g., supplier API cost/latency per experiment cell, plus booking profitability as a guardrail.
  - Count distinct users, not events, for skewed metrics.

## Zach Wilson (DataExpert) — Maintain Data Pipelines Like Netflix and Airbnb (Week 6) (https://www.youtube.com/watch?v=nt38LZhk7jQ)
**Every pipeline is a permanent maintenance liability. Minimise the count (deprecate, consolidate, sample), make failures cheap (runbooks, sane SLAs, clear ownership), and treat infrastructure as a marathon.**
- First principle: maintenance cost compounds. With a 10% daily failure chance per pipeline, 1 pipeline means about 3 incidents a month, 10 pipelines about 1 a day, and 100 pipelines about 10 a day. Data engineering is inherently unsustainable unless you actively prune. "The only thing better than optimized is deprecated."
- Why the job is hard: high expectations leading to burnout, data quality issues, and unclear priorities / ad hoc requests. Data engineers get little visibility and carry on-call; analysts present the results.
- Marathon, not sprint:
  - Pipelines are infrastructure (like the US highway build-out). Every time he rushed a pipeline he "ate pain" later in quality, completeness or maintainability. Analytics, by contrast, IS a sprint.
  - His Facebook 2017 growth-drop investigation was treated as a sprint for 7 months, ending in burnout and quitting.
  - Mantra: "we're not saving lives". A dataset two days late rarely matters. (One exception he hit: Netflix external-data requests from law enforcement.)
- Ad hoc requests:
  - Budget about 5–10% of time (roughly half a day a week).
  - If it's a genuine 10–15 minute pull, just do it. That's under half of requests.
  - Complex asks go on the quarterly roadmap; plan with all analytics partners at the start of each quarter. "If everything is urgent, nothing is."
  - Saying "next quarter" creates healthy pain that changes partner behaviour; always saying yes doesn't. This needs a strong engineering manager.
- Ownership models (five ownable things: datasets, pipelines, documentation, metrics, experimentation):
  - Common model: software engineers own logging/exports; data engineers own pipelines and master data; analysts own metrics, dashboards and experiments. Edges (logging specs, aggregate master data vs metrics) are blurry.
  - "Analytics engineer / spanner" model (Netflix): one person from logs to dashboards. Faster metric changes, but the data engineer lacks time for business context and becomes a tribal-knowledge island.
  - Boundary failures:
    - Data engineers wandering upstream into servers/logging.
    - Data scientists writing poor pipelines because the data engineers didn't deliver.
    - A data engineer owning pipeline + master data + metrics + business questions. His Airbnb unit economics: about 20 downstream and about 30 upstream teams, and everything melted when he went on vacation.
  - Consequences: burnout, blame games, perceived credit-stealing.
  - Fixes: solve organisationally (escalate to managers), not by heroics. Every important pipeline needs a secondary owner.
- Team models: centralized data engineering team (his preference: shared on-call, knowledge sharing; cost is cross-team prioritisation) vs embedded (one data engineer per business team; isolated, with messy mixed-role on-call).
- On-call:
  - Joined about 6 months into a role; mostly business hours, with weekend checks, unless latency-critical (Netflix personalisation, which pages at 3am).
  - Negotiate SLAs from measured impact. He deliberately let an Airbnb pipeline be late and measured about a 0.5% ML-effectiveness loss per day of delay, then changed the 4-hour fix SLA to 24 hours. Compare the cost of the delay with the cost of burnout and turnover.
  - Document every failure (from the orchestrator alerts) to find "problem children". Hold a 20–30 minute handoff each rotation.
- The four common on-call problems:
  1. OOM, mostly skew (e.g., a viral post producing 100M notifications to one key). Fixes in order:
     - Spark 3 AQE (`spark.sql.adaptive.enabled`).
     - Bump executor memory (a quick test).
     - Salt the join/group key with a random number, pre-aggregate, then aggregate again.
     - Non-skew OOM usually means bad code (e.g., `.collect()` to the driver).
  2. Missing data / schema change: run pre-checks on untrusted inputs (especially third-party APIs) so the pipeline never starts on bad input. Get the upstream owner to fix it. For third parties, adapt your code. They may simply shut the API off: his 2020 startup died when the Call of Duty stats API was turned off.
  3. Backfills:
     - Small: backfill in parallel into `table_backfill`, validate, rename prod to `table_old` and backfill to prod, notify downstreams, drop old.
     - Large (over about 30 downstreams): build `table_v2` plus a parallel pipeline, migrate consumers, then drop prod and rename v2 to prod. Never leave permanent `_v2/_v3` names.
     - Validate one month before backfilling years; he has wasted thousands of dollars backfilling 7 years with a bug.
  4. Business questions: set a response SLA (e.g., 2h or 1 day) so you keep a life on call. Maintain an FAQ doc. Ideally analysts own the business-question on-call.
- Remedies for painful pipelines:
  - Delete: at Netflix, about 50 hourly security-detection batch pipelines had no queries in 60 days. It took about 6 months to convince stakeholders; on-call went to near zero.
  - Consolidate via better modeling (100 pipelines at 1% failure is about 1 incident a day; 10 pipelines is a few a month).
  - Migrate technology. Hive→Spark gave 50–90% gains only for high-cardinality GROUP BY/JOIN, and 5–10% elsewhere; don't over-promise. Spark is faster but less reliable (in memory, so OOM/skew), while disk-based Hive is slow but finishes.
  - Streaming (e.g., Flink) only for a genuine new use case. It can smooth memory spikes.
  - Sampling:
    - His Netflix A/B-test AWS-cost metric used under 1% of hundreds of TB/hour of request logs, because direction was all it needed.
    - Choose the sampling unit with a data scientist: all requests for a sample of USERS preserves per-user flows; random requests do not.
    - Never sample for audits (needle in a haystack).
  - Bucketing:
    - Pre-hash both join sides on the key into N files (e.g., 1,024) so the join matches files with no shuffle. Shuffle breaks down above about 10TB.
    - Facebook notification features × events: bucketing on user cut the job from 25% to 4% of the notifications compute.
    - Don't bucket tables under about 1TB: opening many small files costs more than the shuffle.
- Cloud cost = IO + compute + storage; IO is usually the biggest.
  - His Airbnb savings split: about 85% IO, 10% compute, 5% storage. Shrinking a heavily-read master table multiplies savings by the number of downstream readers.
  - Leaf/dashboard tables are compute-dominated.
  - This is the real argument for signal tables over WAP: WAP pays the write IO twice.
  - IO wasters:
    - Duplicate data models (Airbnb had 7 definitions of "available").
    - Recomputing MAU by scanning 30 days of facts instead of a cumulative table (1 partition vs 30).
    - Hasty full backfills.
    - Not sampling.
    - Not sub-partitioning.
  - Sub-partitions: a low-cardinality, frequently filtered column (aim for 5, at most about 30 values, NOT country) enables partition pruning. Example: notifications partitioned by date/channel, where push is about 100× SMS, so an SMS query skips 99% of the IO.
  - Avoid O(n²) nested loops in UDFs.
  - Good Parquet compression cuts IO because data decompresses only on the executor.
- Multiple sources of truth (the highest-leverage clean-up):
  - Find all variants: talk to stakeholders, since table names are arbitrary; code-search as a start; use lineage back to shared raw sources if available.
  - Document the discrepancies (his spreadsheet consolidated 30–40 tables down to about 4).
  - Write a new spec and get sign-off BEFORE coding. The Airbnb availability result was a new definition none of the existing ones matched.
  - Diagnose root causes: lack of trust between teams, the owning team lacking bandwidth or skills. Fixing those stops recurrence. Agree future ownership and the change process.
- Tech-debt models:
  - Fix-as-you-go: rarely works; Facebook was the messiest.
  - Dedicated time each quarter (Airbnb's "tech excellence week"): his favourite. Shared accountability, but debt saw-tooths and the week is costly.
  - On-call person fixes debt (Netflix): they know the worst pain, but about half don't do it, since it's invisible work.
- Migration models:
  - Cautious: run parallel for months, rename to `_do_not_use`, then drop. Doubles cost; zombie tables.
  - "Bull in a china shop" (Airbnb from about 2021): migrate the critical/expensive consumers (ML) first, give others about a month, then turn off the old pipeline so breakage forces migration.
- Runbook contents (for complex or high-fan-out pipelines; can live in the spec):
  - Primary and secondary owner.
  - Upstream owners listed as TEAMS, not individuals (reorgs are rarer than attrition).
  - Common issues with known fixes, built up from on-call logs.
  - Critical downstream owners only (e.g., Smart Pricing ML).
  - SLAs, e.g., "lands by 4h after UTC midnight", so "late" is defined.
  - Also hold recurring monthly/quarterly 1:1s with upstream and downstream teams. Relationships make incidents gentler and surface duplicate models early.
  - Noisy non-blocking checks should be tuned for signal-to-noise.
- Relevance for Sam:
  - As a one-person data function at a startup, keep the pipeline count low.
  - Treat each supplier API (Viator, RateHawk, Coras) as untrusted: pre-checks before the run, plus a runbook entry with known failure modes (rate limits, schema drift, API shutdown risk).
  - Sample by entity (e.g., whole countries or whole suppliers) for exploratory work on the 6M-row GeoNames set, never for audits.
  - Partition Parquet by a low-cardinality filter column (e.g., continent or feature_class), not by country if there are many tiny files.
  - Keep one canonical "master location" definition rather than per-project variants.
