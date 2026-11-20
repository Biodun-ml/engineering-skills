# zach-1 — cross-cutting principles

1. **Model for the consumer first.** Analysts get flat primitive columns; engineers can take arrays/structs/maps; ML gets features; execs and customers get charts. A fast, compact table nobody uses is a failure. Serve different consumers with different tables, e.g. a nested master table plus a short-retention exploded copy. (Data Lake Modeling; Funnel Analysis)
2. **Follow the continuum: prod snapshots -> master data -> OLAP cubes -> metrics.** Don't let analysts query raw snapshots, or five pipelines will give five numbers. The master layer, deduped and conformed, is where truth and trust live. (Data Lake Modeling)
3. **Raw logs are not facts.** A fact table is atomic, deduplicated, has non-null what/when/who, UTC timestamps, clean names (`dim_` for dimensions, `m_` for measures), parsed columns instead of JSON blobs, and no columns that are derivable or cheap to join. Keep the "when" and any dimension that grows without bound. (Meta Event Data)
4. **Fix problems upstream when you can.** Logging the app name at the source removed a 100 TB/hour join that no longer worked after IPv6. Shared schemas like Thrift, used by both producers and pipelines, stop the two from drifting apart. (Meta Event Data; Spark + Iceberg; Graph/Additive)
5. **Shuffle is the parallelism bottleneck.** SELECT/WHERE scale without limit. GROUP BY, JOIN and window PARTITION BY shuffle by key. A global ORDER BY or `.sort()` goes through one machine. Read `EXPLAIN` and count the Exchange steps. (Meta Event Data; Spark + Iceberg; Funnel)
6. **Pick join strategies on purpose.** Broadcast the small side (up to roughly 8–10 GB in Spark). Bucket both big sides on the join key in power-of-two counts, and only when the table is joined repeatedly. Handle skew with AQE, two-stage salting (only valid for additive measures), or by splitting outlier keys into their own path. (Spark + Iceberg; Meta Event Data)
7. **Reduce volume before tuning compute.** Options: sampling (for directional metrics, never for needle-in-haystack security work), retention tiers, daily aggregates or a metric repository, and reduced facts (a monthly array per entity and metric). Reduced facts took 10-year analyses at Facebook from weeks to hours. (Meta Event Data)
8. **Cumulative tables = FULL OUTER JOIN of yesterday's state with today's data.** This gives history in a single partition, analysis with no shuffle, and cheap transition labels (new, churned, resurrected, deleted). The costs: backfills must run in sequence, arrays need a cap (about 30), and deleted users or PII must be purged on purpose. (Data Lake Modeling; Meta Event Data date-list lab)
9. **Bit-pack activity history.** A 32-bit date-list integer per user, with `bit_count` and bitwise AND masks, gives daily, weekly and monthly active flags with almost no storage. Hide the bit maths from analysts by publishing ready-made flags. (Meta Event Data)
10. **SCD Type 2 is the idempotent gold standard.** Build it as a full rebuild with gaps-and-islands (LAG -> change flag -> running SUM -> GROUP BY streak) when dimensions are small. Use an incremental merge (closed history + unchanged + changed-as-two-rows + new) when scale forces it. Detect changes with null-safe comparisons. (SCD Lab)
11. **Sort before you write, and protect run-length encoding.** Sort within partitions from low to high cardinality: files were 10–15% smaller, up to about 80% on repeated strings. A downstream shuffle scatters the runs and undoes the compression, so keep temporal detail nested in arrays until after joins. (Spark + Iceberg; Data Lake Modeling)
12. **Deduplicate deliberately.** Define the grain and check it with GROUP BY ... HAVING count > 1. Use ROW_NUMBER with a deterministic ORDER BY. Choose a dedup time window. At huge scale, hourly micro-batch dedup merged as a binary tree beats both one giant daily query and stateful streaming. (Meta Event Data)
13. **Check additivity before rolling up.** A dimension is additive in a time window only if an entity can have just one value in that window, so device and platform are not additive for user counts. COUNT DISTINCT and ratio metrics can't be rebuilt from subtotals of non-additive dimensions. (Graph/Additive)
14. **Pre-aggregate dashboards with GROUPING SETS or ROLLUP plus an `aggregation_level` label.** Coalesce dimensions before grouping. Avoid CUBE except in small exploratory work. Dashboards then filter with WHERE instead of running GROUP BY. Analysts writing complex SQL (about 10 CTEs, piles of CASE WHEN) is a modeling smell: materialize the result, since storage is cheaper than compute and analyst time. (Funnel Analysis)
15. **Integrate many sources with an enum: the "little book of enums."** Each source gets a mapping function into one shared schema, per-source DQ thresholds stored in a small table, outputs partitioned by date and enum value, and a map column for source-specific extras. Use enums only for fewer than about 50 values. Maps and JSON compress badly. (Graph/Additive)
16. **Graphs need only two schemas.** Vertex = (id, type, properties); edge = (subject id/type, object id/type, edge_type verb, properties). Use graphs for multi-hop "how is X connected to Y" questions that would otherwise be chains of joins. (Graph/Additive)
17. **Default to daily scheduled batch** (about 90% of pipelines), with outputs partitioned by run date (`ds`) so they can be backfilled. Streaming (5–10%) needs a real latency reason, such as fraud. On-demand (1–2%) is for per-entity triggers. (Pipelines in 8 min; Spark + Iceberg)
18. **Size resources empirically and respect the driver.** Never `collect()` large data. Find the smallest executor memory that is stable for a few days instead of blanket 16 GB. Raise memory overhead for UDF-heavy jobs. For big API pulls, prefer bulk exports or reading the underlying DB over per-call REST from the driver. (Spark + Iceberg)
19. **Dimension definitions are one-way doors.** Changing Airbnb's "available" definition took two years, and Facebook's 5,000-friend cap can't be undone. Bucket derived dimensions from real percentiles (5–10 buckets), not invented ranges. (Meta Event Data)

**Laptop translation (16 GB, DuckDB/Polars/Parquet):** treat small lookups as broadcast joins. Hash-partition big-to-big joins into power-of-two Parquet buckets and process them bucket by bucket. Sort Parquet output by low-cardinality keys first. Run cumulative and SCD logic as daily FULL OUTER JOINs over date-partitioned files. Use per-supplier enum mapping functions with DQ thresholds for API integrations.

---

## Zach Wilson (DataExpert) — How Meta Models Big Volume Event Data, Full 4-Hour Course, Bootcamp Week 2 (https://www.youtube.com/watch?v=DQefW9sNmw0)
**Fact data is the biggest, most duplicated, least trustworthy data you own; turn raw logs into small, deduped, well-named, analyst-friendly facts, and shrink volume (aggregate, reduce, bit-pack) so long-horizon questions stay cheap.**

**Mental models / first principles**
- A fact is an atomic, immutable event ("something happened"). Atomicity matters: "ran a mile" is an aggregate of steps. Facts never change (unlike SCD dimensions), but they are 10–100x the volume of dimensions; size ~ (#entities x actions per entity). Facebook: 2B users x 25 notifications/day ~ 50B rows/day; Netflix network logs ~2 PB/day.
- Facts are worthless in isolation; value comes from context: funnels (sent -> clicked -> bought) and joined dimensions (click-through by country).
- Raw logs != fact data. Logs are owned by SWEs, have ugly schemas, JSON blobs, duplicates, no guarantees, short purpose. Facts have nice names, quality guarantees, longer retention, and should be smaller. The DE's job is converting one into the other.
- Describe a fact with who (IDs: user, device, car), where (location OR place in the app), how (device/method; who/how and where/how blur), what (atomic event name), when (timestamp, always UTC; client-side logs are higher fidelity than server logs but must log UTC, not local time).
- Facts vs dimensions blur. Dimensions = what you GROUP BY, come from snapshots of state (prod DB snapshots daily). Facts = what goes inside SUM/COUNT, come from logs/events. Aggregated facts become dimensions (dim_is_active derived from events vs dim_is_activated, a pure state flag; both can disagree, e.g. Messenger-only users). CDC sits in the middle: state changes modeled as events let you rebuild dimensions at any point. Airbnb nightly price "feels like a fact" but is a dimension: it is derived from host settings (state); the setting-change is the fact.

**Mechanisms**
- Normalized vs denormalized facts: normalized (IDs only) removes duplication and restatement problems and wins at small scale. Denormalize only when the join is unaffordable. Netflix case: joining every network request to an IP->app table worked only as a broadcast join (small side under ~5–6 GB). Moving to IPv6 broke that; a shuffle join cost >10x. The fix was upstream: a sidecar proxy so every app logs its own app name on each request (denormalized at the source). Lesson: biggest impact is often changing logging, not tuning the pipeline.
- Logging contracts: shared, language-agnostic schemas (Airbnb used Thrift between Ruby app code and Scala pipelines) so schema changes break a test in the producer's CI and force a conversation; without it the two calculations drift.
- High-volume levers: sampling (1% of requests was enough for an "infrastructure cost per A/B test" metric; never for security / needle-in-haystack cases); bucketing on the "who" ID to enable bucket joins, and sorted-merge-bucket (SMB) joins that need no shuffle (zipper merge); retention tiers (big tech: tables >100 TB kept ~1–2 weeks; <10 TB mostly governed by anonymization rules). Scale the thresholds to your budget; push back on "keep it just in case".
- Dedup is the hardest part. Duplicates come from logger bugs AND genuine repeats (clicking the same notification twice); decide a dedup time window (a click a year later doesn't matter). Facebook notification dedup took 9 hours as one daily Hive query, blocking downstream jobs. Streaming dedup works if the window is short (most dups arrive soon after the first event) but holding a full day of IDs in state exhausted memory at 50B/day. Solution: hourly micro-batch dedup — GROUP BY within each hour, then FULL OUTER JOIN hour pairs (0+1, 2+3...), then merge pairs of pairs as a binary tree until one deduped day; data was ready ~1 hour after midnight. Scales arbitrarily.
- Lab (fact_game_details): find the grain (game, team, player), check `GROUP BY grain HAVING count>1`, dedup with `ROW_NUMBER() OVER (PARTITION BY grain ORDER BY ...) = 1`. Drop columns joinable cheaply (team city/abbrev: 30 teams will always fit in Excel) but bring in the "when" (game date) because games grow without bound and the join gets expensive. Drop derivable columns (percentages). Parse a messy comment string into booleans (did-not-play / did-not-dress / not-with-team) without baking in business rules. Convert "MM:SS" strings to fractional minutes so rates are easy. Rename keyword columns (`to` -> turnovers). Prefix columns `dim_` (filter/group) and `m_` (aggregate). Primary key on the grain.
- Bucketizing dimensions derived from facts: avoid groups of one; aim ~5–10 buckets from actual percentiles (quartiles/deciles), not invented ranges. Trade-off: buckets lose flexibility but compress better.
- Dimension definitions are sticky and expensive to change: Facebook's 5,000-friend cap is a one-way door; changing Airbnb "is available" (rule-blocked night -> "can a trip containing this night be booked", ~3–4% different due to sandwich nights) took two years. Involve stakeholders before shipping a definition.
- Date list (bit-packed activity): naive MAU = scan 30 days of facts daily (3T rows at FB). Instead, cumulative table `users_cumulated(user_id, dates_active array, date)` built by FULL OUTER JOIN of yesterday's state with today's activity (prepend newest date). Then cross join with `generate_series` of 32 days and for each active day add `2^(32 - days_since)`; SUM -> one 32-bit integer per user. `bit_count(x) > 0` = monthly active; AND with a mask of the first 7 bits -> weekly active; first bit -> daily active. Integers compress extremely well. Facebook often used 28-day "months" (equal count of each weekday removes weekly seasonality).
- Shuffle as the parallelism bottleneck: SELECT/FROM/WHERE (no window) is embarrassingly parallel; GROUP BY, JOIN (both sides shuffled), HAVING (after GROUP BY) need all rows of a key on one machine (key mod `spark.sql.shuffle.partitions`, default 200); global ORDER BY funnels through one machine — only use it on small final results. Window ORDER BY is fine with PARTITION BY, global without it.
- Fix shuffle by (1) bucketing at write time, (2) reducing volume. Three fact flavours: raw facts (flexible, only good for days/weeks); daily aggregates / "metric repository" `(user, metric_name, date, value)` partitioned by date and metric_name (good for 1–2 years, still joins with daily SCDs; powers FB's Deltoid experimentation); reduced facts `(user, metric_name, month_start, value_array)` where array index = day offset. Reduced facts cut another 30x (or 365x yearly) and took 10-year backfills from ~a week to a few hours (50 metrics x 15 dimensions; enabled slow-burn decline RCA).
- Reduced-fact lab: daily aggregate FULL OUTER JOIN existing month row; if user is new mid-month, left-pad with `array_fill(0, date - month_start)`; append COALESCE(value,0) so all arrays have equal length; upsert (`ON CONFLICT DO UPDATE` in Postgres; `INSERT OVERWRITE` in lakehouses). Analysis: sum element-wise per index across users (no explode), then `UNNEST ... WITH ORDINALITY` only after aggregation to get back daily series. Validate with `cardinality(array)` all equal.

**Decisions & trade-offs**
- Normalize by default; denormalize when join cost explodes, ideally at the logging source.
- Streaming dedup for small windows/volumes; hourly micro-batch tree for huge volume + whole-day window.
- More aggregation = faster, fewer questions answerable; mid-month dimension changes in reduced facts are approximated (accept documented inconsistency).

**Rules of thumb**
- Fact must-haves: no duplicates, non-null what/when (and usually who), no raw JSON blobs (simple types; small arrays like experiment-group lists are OK).
- Log only what you need; logging "just in case" costs money.
- Build incrementally day by day rather than giant backfill queries.
- Prefer hiding bit tricks from analysts: publish ready-made daily/weekly/monthly active flags.

**Failure modes**
- Non-UTC client timestamps; CTR >100% from undeduped clicks; group-by on un-bucketed high-cardinality dims; global ORDER BY on millions of rows; NULL handling in array cumulation (arrays of nulls, NULL fill lengths, timestamp vs date from date_trunc); hardcoded dates in incremental SQL.

**Relevance to Sam**: hourly-tree dedup and FULL OUTER JOIN cumulation port directly to DuckDB/Polars over daily Parquet partitions; date-list ints are a cheap way to store per-entity activity (e.g. supplier/product availability per day) on a laptop; "drop cheap-to-join dimensions, keep growing ones" applies to location/GTFS facts; push suppliers or internal services to log IDs/fields at source instead of expensive joins.

## Zach Wilson (DataExpert) — Data Engineering Design Patterns at Meta: Funnel Analysis, Bootcamp Week 4 Day 1 (https://www.youtube.com/watch?v=7O7DH0ux0x4)
**Funnels are "event A then event B by the same actor" problems: dedup, self-join with time ordering, aggregate to the actor, then pre-aggregate dashboards with GROUPING SETS so analysts filter instead of group.**

**Mental models / first principles**
- A funnel = many actors at the top, fewer at the bottom; the business pushes actors down it (growth, revenue). Data-wise it is two (or more) fact events joined on the actor with an ordering constraint.
- The SQL interview vs the job: recursive CTEs, correlated subqueries and "rewrite without window functions" rarely appear in real work (hierarchies are shallow; two or three joins usually beat a recursive CTE). What interviews get right and matters on the job: count table scans, use COUNT/SUM(CASE WHEN ...) to pivot in one scan, cumulative design, query the minimum data, clean code (CTEs, aliases on every derived column).
- Learn `EXPLAIN` and the logical order of SQL (FROM/JOIN -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY); understanding the plan tree is what removes accidental slow queries.
- "Complex SQL by analysts is a data-modeling smell." Symptoms: slow dashboards (fix: pre-aggregate on the dimensions people actually cut by — billions of rows -> a handful per app), queries with ~10 CTEs re-run repeatedly (fix: materialize the first half as a staging table with short retention), piles of CASE WHEN (fix: conform values upstream). Storage is cheaper than compute, and analyst time is a third cost; even a 1:1 storage-for-compute trade is worth it.
- Learn your analysts' recurring query patterns early in a new job; making them faster is the job.

**Mechanisms**
- LAG/LEAD can be replaced by a self-join on `date = date + 1` (what interviewers want when banning window functions).
- Window function anatomy: function (row_number, dense_rank, rank, sum, avg, lag, lead, first/last_value) + OVER (PARTITION BY, ORDER BY, ROWS BETWEEN). Default frame = UNBOUNDED PRECEDING to CURRENT ROW, so SUM with ORDER BY and no frame gives a running total. Rolling 30-day average: `ROWS BETWEEN 29/30 PRECEDING AND CURRENT ROW` with correct ORDER BY (note: ROWS counts rows, not days — gaps in dates break it). Prefer ROW_NUMBER (unique) or DENSE_RANK (no gaps); avoid RANK (skips numbers).
- `CROSS JOIN UNNEST` (Presto/Postgres) == `LATERAL VIEW EXPLODE` (Spark/Hive).
- GROUPING SETS: several aggregation grains in one query, replacing UNION ALL of copy-pasted queries (UNION ALL is slow and hard to maintain). Excluded dims become NULL, so COALESCE grouping dims to 'unknown' BEFORE grouping, otherwise you can't tell "real null" from "rolled up". Use `GROUPING(col)` (0 = included) to build an `aggregation_level` label like `os_type__device_type__browser_type` (double underscore between dims, Facebook convention), then persist as a dashboard table; dashboards then use WHERE aggregation_level = ... instead of GROUP BY.
- CUBE: all 2^n combinations (3 dims -> 8 grains); explodes combinatorially, wastes compute on cuts nobody wants; only for EDA on small data, max ~3 dims. ROLLUP: hierarchical prefixes (country -> state -> city), grows linearly, includes grand total. Zach used GROUPING SETS and ROLLUP at Facebook/Netflix/Airbnb, never CUBE. BigQuery (at that time) only supported ROLLUP.
- Funnel lab (signup page -> POST /api/v1/user): (1) dedup events with GROUP BY user, url, event_time, date; drop null users; filter to the funnel URLs early (shrinks data). (2) Self-join d1 to d2 on user_id AND same event_date AND d2.event_time > d1.event_time (ordering prevents counting a later revisit after signing up). (3) Keep non-converters: put the step-2 URL condition inside the aggregate, not in WHERE (filtering in WHERE silently restricts to converters). (4) Collapse to one row per user with `MAX(CASE WHEN d2.url = signup_api THEN 1 ELSE 0 END)` (COUNT would double count). (5) Rate = SUM(converted)/COUNT(*) cast to real: ~34% of users who saw the page signed up. Per-hit/per-page conversion is very different (~2–3%) because repeat visits and bots inflate the denominator; add HAVING hits > N to drop junk paths.

**Decisions & trade-offs**
- Per-actor vs per-event conversion answer different questions; state which.
- Attribution window: same-day join is simpler; a longer window catches return visitors but needs a defined cutoff.
- GROUPING SETS (explicit, efficient) > ROLLUP (hierarchies) > CUBE (exploration only). Avoid "automagical" constructs that hide cost.
- Not every complex analyst query means bad modeling; one-off deep questions don't deserve a pipeline. Repeated ones do.

**Rules of thumb**
- One table scan where possible; conditional aggregation instead of multiple filtered subqueries.
- Coalesce dims before grouping sets; label aggregation levels; pre-aggregate dashboards.
- Distinct actor-level flags via MAX(CASE...) not COUNT DISTINCT hacks.

**Failure modes**
- WHERE clause on the right side of a funnel join turning it into converters-only.
- Matching the same URL to itself in self-joins; duplicate logger events; bots in denominators; endpoint renames (users -> user) splitting one event into two names over time.

**Relevance to Sam**: booking funnels (search -> product view -> checkout -> booked) across supplier integrations are exactly this pattern; in DuckDB, GROUPING SETS/ROLLUP produce small Parquet "dashboard cubes" from millions of rows cheaply; the analyst-smell checklist applies to anything Trameter teams query repeatedly.

## Zach Wilson (DataExpert) — Spark + Iceberg in 1 Hour: Memory Tuning, Joins, Partition, Bootcamp Week 3 Day 1 (https://www.youtube.com/watch?v=3R-SLYK-P_0)
**Spark is RAM-first distributed compute; performance is mostly about which join strategy the driver picks, how much you shuffle, how you handle skew, and how you sort data on write so columnar compression works.**

**Mental models / first principles**
- Lineage: Java MapReduce (2009) -> Hive -> Spark. MapReduce/Hive wrote to disk between every step (resilient, very slow). Spark keeps data in memory and only spills to disk when it lacks RAM; spilling effectively turns it back into Hive. Presto/Trino is all in-memory and simply fails when an operation does not fit, which is why Spark won for heavy jobs.
- Don't pick Spark if you're the only one on the team who knows it, or the company has standardized on BigQuery/Snowflake etc.: homogeneity of pipelines beats one clever outlier.
- Analogy: plan = the play (DataFrame/SQL/Dataset code, lazily evaluated), driver = coach (decides when to execute, which join type, parallelism per step), executors = players (do the work). Nothing runs until an action "takes the shot": write output or collect to the driver.

**Mechanisms & settings that matter**
- Driver: `spark.driver.memory` (default ~1–2 GB, up to ~16 GB) only needs raising for very complex plans or when collecting data to drive the plan (usually bad practice); `spark.driver.memoryOverhead` for JVM non-heap needs with complex plans. Zach found other driver settings had no meaningful impact.
- Executors: `spark.executor.memory` — don't blanket-set 16 GB after an OOM. Run at 2/4/6/8 GB for a few days and keep the smallest that's always stable, leaving modest padding (engineer time debugging OOMs costs more than a bit of extra RAM). `spark.executor.cores` default ~4, up to 5–6; beyond that disk/throughput becomes the bottleneck and more concurrent tasks raise OOM risk (more chances of a skewed task). `spark.executor.memoryOverhead`: raise for heavy UDFs (PySpark UDFs especially), many joins/unions; improves reliability without raising heap.
- Three joins (look for these names in `.explain()`): shuffle sort-merge join (always works, least performant); broadcast hash join (one side small — either side — shipped to every executor, no shuffle; workable up to ~8–10 GB in his experience); bucket join (both sides pre-bucketed on the join key; matching buckets zip together, no shuffle).
- Shuffle: rows routed by hash(key) mod N partitions. `spark.sql.shuffle.partitions` (default 200) and `spark.default.parallelism` are effectively the same knob (latter only for the RDD API; avoid RDDs). Initial parallelism of a read = number of files. In plans, "Exchange" == shuffle.
- Shuffle is fine for most volumes (under ~1–10 TB). It breaks at very large scale: Netflix 100 TB/hour network logs could broadcast an IP->app table until IPv6 made it too large; fix was upstream logging of app name (no join). Facebook notification features: 10 TB joined to 50 TB consumed ~30% of the notifications compute; fix was bucketing both sides on user_id into 1,024 buckets.
- Bucketing rules: always powers of two (bucket joins also work when counts are multiples, e.g. 4 vs 8); size buckets to data (~10 GB/bucket at Facebook); too many buckets on small data creates empty files (pigeonhole) that can break bucket joins. Bucketing costs one shuffle at write, so it only pays off when the table is joined/aggregated multiple times downstream. Presto can be slower on tables with few buckets because read parallelism is capped at bucket count.
- Skew: one hash partition gets far more data (celebrity users). Symptom: job reaches 99% then crawls and fails. Fixes: (1) Spark 3 adaptive query execution `spark.sql.adaptive.enabled=true` (+ skew join handling) — (note: AQE has been on by default since Spark 3.2, so it's rarely something you need to "turn on"); (2) for skewed GROUP BY on old Spark, salting: add a random column, aggregate by (key, salt), then aggregate again by key — only valid for additive measures (SUM/COUNT, not COUNT DISTINCT/medians); (3) for joins: filter out outlier keys or split the pipeline into "normal keys" and "outlier keys" paths.
- Notebooks vs spark-submit: notebooks for proof of concept; production code in git with tests, run via spark-submit. Notebooks lower the bar for non-engineers but discourage unit/integration tests.
- Reading: Spark reads lakes (Delta, Iceberg, Hive metastore), JDBC DBs, files, REST. REST calls run on the driver by default (memory risk). Option: parallelize the URL list so executors call the API — but that hammers the API. Usually better to read the source's underlying tables/DB than its API.
- Output: partition outputs by date = the pipeline's execution date (Airflow `{{ ds }}`) so runs are backfillable and tables don't bloat.

**Lab takeaways**
- `.collect()` on a large DataFrame pulls everything into driver memory -> Java heap OOM and dead kernel. Use `.show()`/`.take(n)`, or collect only aggregated/limited data.
- `.sort()` = global sort: plan shows an extra `Exchange rangepartitioning` (second shuffle through range partitioning) — painful above ~20 GB. `.sortWithinPartitions()` sorts locally with no extra shuffle; use it at scale.
- Read `.explain()` bottom-up (most indented first): FileScan -> Project (=select) -> Exchange hashpartitioning (repartition) -> Sort(global=false/true).
- Iceberg: partition transforms (`days(ts)`, `hours`, `years`) or plain columns; inspect `<table>.files` metadata to compare file counts and `file_size_in_bytes`.
- Sorting before write shrank files ~10–15% via run-length encoding; sorting only by date barely helped; sorting by low-to-high cardinality columns (date, browser, host) helped most, and which columns you choose matters (host beat OS family).

**Rules of thumb / checklist**
- Check `.explain()` for join type and Exchange count.
- Prefer broadcast for small dims; bucket (powers of two) only for repeatedly joined big tables.
- Fix the source (logging) before heroic Spark tuning at extreme scale.
- Size executor memory empirically; raise overhead before heap for UDF-heavy jobs.
- Never global-sort large data; sortWithinPartitions by ascending cardinality before writing Parquet/Iceberg.

**Failure modes**: blind 16 GB memory settings; driver-side collect/REST pulls; job dying at 99% from skew; empty buckets; extra range-partition shuffle from `.sort()`; querying the wrong table version after recreate (lab mistake).

**Relevance to Sam**: the same physics apply on a 16 GB laptop with DuckDB/Polars — joins against small lookups (GeoNames admin codes, country tables) are effectively broadcast joins; for big-to-big joins, pre-partition/hash both sides on the key into power-of-two Parquet buckets and join bucket-by-bucket to stay in RAM; sort Parquet output by low-cardinality columns (country, feature_class, date) first for much smaller files; pull supplier data from bulk exports/DB dumps where possible rather than per-item REST calls, and when calling APIs keep concurrency bounded.

## Zach Wilson (DataExpert) — Data Lake Modeling: 100 TBs into 5 TBs at Airbnb with Parquet + Run Length Encoding (Dimensional Data Modeling Day 1 lecture) (https://www.youtube.com/watch?v=7JbCVXmJ1bs)
**Model dimensions for your consumer: snapshot -> master data -> OLAP cubes -> metrics; use cumulative tables to carry history in one partition; keep temporal data in arrays/sorted so Parquet run-length encoding survives downstream shuffles.**

**Mental models / first principles**
- A dimension is an attribute of an entity. Fixed dimensions (birthday, account ID, signup date) are trivial; slowly changing dimensions (favourite food) are the hard part (covered in Day 2).
- Data modeling is an empathy exercise: a compact, fast table nobody uses (or that changes no decision) is a failure. Know your consumer:
  - Analysts / data scientists: want flat primitive columns; many will not learn arrays/structs — meet them where they are.
  - Other data engineers: can handle arrays/structs/maps; you can model "fancier" (Zach's Airbnb pricing/availability dataset had ~150 downstream pipelines).
  - ML models: may need vectors or a feature store; Zach owned the Smart Pricing feature-store dataset jointly designed with ML engineers.
  - Customers and executives: give charts, not tables (a simple multi-app growth line chart was praised by Zuckerberg).
- OLTP vs master data vs OLAP: OLTP (Postgres/MySQL/Mongo) is normalized for single-record reads/writes (WHERE user_id = 7), terrible for analytics (many joins). OLAP minimizes joins and optimizes GROUP BY over large volumes, slow for single-record access. Master data sits between: complete, deduped entity definitions that can serve OLAP. Mismatching model to consumer hurts either way.
- The continuum all dimensional work follows: daily production DB snapshots (typically taken ~1 hour after UTC midnight) -> master data (merge, dedup, conform; this is "where truth lives") -> OLAP cubes (pre-aggregated by dims) -> metrics (one number per day). Rule: analysts should be forbidden from querying raw prod snapshots directly, otherwise five people compute five slightly different metrics.

**Mechanisms**
- Cumulative table design: each day, FULL OUTER JOIN yesterday's cumulative table with today's snapshot; COALESCE IDs and unchanging dims; compute cumulative metrics (e.g. days since last active); append to history arrays. FULL OUTER JOIN captures new entities (only today) and disappeared ones (only yesterday). The first run's "yesterday" is empty. Facebook's `dim_all_users` used this and had ~15,000 downstream pipelines.
- Size win: storing (user, [7 booleans]) in one partition is less than half the size of 7 daily (user, bool) rows because the string key isn't repeated; "weekly active" then reads 1 partition instead of 7, and monthly reads ~97% less data. Cap arrays (Facebook kept 30 values) — unbounded arrays become unusable.
- Strengths: historical analysis with no shuffle (just inspect the array — Trino/Presto can scan billions of rows in seconds); trivial transition analysis between states: churned (active yesterday, not today), resurrected (inactive yesterday, active today), new (didn't exist yesterday), deleted (existed yesterday, gone today).
- Drawbacks: (1) backfills must be sequential, each day depends on yesterday (Zach backfilled 2010–2018 for a notifications metric; it took ~3 weeks). At small companies dimensional data is small so this is tolerable. (2) PII/retention: history drags forward past retention windows; you must actively filter deleted/inactive users or you keep them forever.
- Compactness vs usability: most usable = only string/int/decimal/boolean (maybe enum). Most compact = opaque byte blobs with custom codecs (Airbnb's listing calendar, shipped to clients to minimise IO/latency; only decodable with a custom Java decoder). Middle ground = arrays, maps, structs — best for master/staging data consumed by engineers. Struct = "table within a table" (Trino ROW type), fields can have different types; array = ordered list of one type; map = keys of one primitive type, values of one type. Employee time spent fighting complex types can wipe out cloud savings, so default to usability unless you're at extreme scale.
- Serve multiple consumers with multiple tables: Airbnb availability had a listing + array-of-nights table for engineers, and an exploded listing-night table with ~2-week retention for analysts.
- Temporal cardinality explosion: adding a time axis multiplies cardinality by >=10x (6–10M listings x 365 nights = ~2–3B rows). Listing-with-array vs exploded listing-night are similar size on disk IF the exploded table is sorted so each listing's rows sit together (Parquet compresses them well).
- Run-length encoding: Parquet stores repeated consecutive values once plus a run count, so sorted repeated strings (player name repeated per season) nearly vanish (~80% smaller in his example). Gotcha: any downstream join/shuffle scatters rows, destroying the runs; the output may compress only ~15%. Keeping nights inside an array means a join moves the whole array intact; consumers can explode after the join and rows stay grouped.

**Decisions & trade-offs**
- Cumulative vs daily-snapshot dims: cumulative = cheap queries and transition analysis, but sequential backfill and PII burden.
- Array vs exploded: arrays protect compression and row counts for engineers; exploded is friendlier but should have short retention.
- Middle-ground types when consumers are engineers; primitives for analysts; blobs only for online latency-critical systems.

**Rules of thumb**
- Decide the consumer before the schema.
- Never let analytics run directly on prod snapshots; build a master layer.
- Use FULL OUTER JOIN + COALESCE for any "yesterday + today" state merge.
- Bound history arrays (e.g. 30 days) and purge deleted users from cumulative tables.
- Sort before writing so RLE works; re-sort (or keep nested) after shuffles.

**Failure modes**: inconsistent metrics from many pipelines on raw snapshots; exploding temporal data then shuffling it (size blow-up); unbounded arrays; GDPR/retention violations via cumulative history; months-long sequential backfills.

**Relevance to Sam**: for Trameter master location data, build a deduped/conformed master layer before analytics; supplier availability/pricing per product per day is a textbook temporal cardinality explosion — store per-product arrays (or sorted product-date Parquet) and explode only on demand; daily supplier catalogue snapshots + FULL OUTER JOIN cumulation yield new/removed/changed products cheaply in DuckDB; sort GeoNames/GTFS Parquet by low-cardinality keys (country, admin1, feature code) to exploit RLE.

## Zach Wilson (DataExpert) — Data Modeling: Building Slowly Changing Dimensions (SCDs), Day 2 Lab (https://www.youtube.com/watch?v=nyu-8Si21ec)
**Build an SCD Type 2 two ways: a full-history rebuild with window functions (simple, backfill-friendly, rescans everything) or an incremental merge of last period's open records with this period's snapshot (cheap, but sequential and full of edge cases).**

**Mental models / first principles**
- SCD Type 2 (start/end validity per value) is the "gold standard" because it is the only SCD type that is idempotent: re-running for any date reproduces the same answer, and you can ask "what was the value as of date X".
- One SCD table can track several attributes together (here `scoring_class` and `is_active` for NBA players); any change in either opens a new record.
- Schema: `entity, tracked cols..., start_season, end_season, current_season` where `current_season` behaves like the date partition (the snapshot the table was built as of). Primary key = (entity, start_season) — not current_season (Zach's first attempt failed on that).
- Dimensions are small relative to facts, so "scan all history every day" is often perfectly acceptable — fine at Airbnb's millions of entities (he ran this in production for unit-economics line items); at Facebook's billions it would not fly. Scale (an extra zero or two) is the deciding line.

**Mechanism A — full rebuild (gaps-and-islands)**
1. `LAG(col, 1) OVER (PARTITION BY player ORDER BY season)` for each tracked column.
2. `change_indicator = CASE WHEN col <> prev_col OR other <> prev_other THEN 1 ELSE 0 END` (combine all tracked columns into one indicator).
3. `streak_identifier = SUM(change_indicator) OVER (PARTITION BY player ORDER BY season)` — a running count; each distinct value is one unchanged streak.
4. `GROUP BY player, streak_identifier, tracked cols` with `MIN(season) AS start_season, MAX(season) AS end_season`; drop the streak id from the output; stamp `current_season` with the run's parameter (e.g. Airflow-injected date), filter input `<= that date`.
- Cost: two stacked window passes over all history before any reduction; only the final GROUP BY shrinks data. Vulnerable to skew/OOM when a few entities change constantly (one player changed almost every season) because they produce many streaks. Upside: no dependency on yesterday, so it backfills in one shot.

**Mechanism B — incremental (yesterday's SCD + today's snapshot)**
Split last period's SCD:
- `historical_scd`: records with end_season < last period — closed, can never change; pass through untouched (select explicit columns; drop current_season so UNION ALL column counts match).
- `last_season_scd`: records with end_season = last period (the open records).
Join open records to this period's data:
- Unchanged: INNER JOIN where all tracked cols equal -> keep start_season, set end_season = current season (extend by one).
- Changed: LEFT JOIN where any tracked col differs -> emit TWO rows: the old record closed as-is, and a new record starting/ending at current season. Build `ARRAY[ROW(old...)::scd_type, ROW(new...)::scd_type]` then UNNEST and flatten (Postgres needs a composite TYPE; Spark handles structs natively).
- New: this period LEFT JOIN last_season_scd WHERE last.player IS NULL -> start = end = current season.
- Result = historical UNION ALL unchanged UNION ALL unnested changed UNION ALL new. Processes ~20x less data than the full rebuild in the lab.

**Decisions & trade-offs**
- Full rebuild: simpler, idempotent, parallel backfill, but O(history) every day and skew-prone.
- Incremental: far less data, but depends on yesterday (sequential backfill), more code paths and edge cases. Zach often prefers the rebuild for dimensions unless scale forces incremental.

**Rules of thumb / gotchas**
- NULLs break change detection: `NULL <> NULL` is not true, so changes to/from NULL are silently missed or rows filtered. Either guarantee non-null tracked columns (COALESCE to sentinel) or use `IS DISTINCT FROM`. Add data quality checks on these assumptions.
- Validate with known entities (e.g. a player who went active -> inactive should show exactly two records).
- Keep `current_season`/date as an injected parameter, never hardcoded.
- Remember the changed-record case yields two rows; forgetting the unchanged branch silently drops most entities (Zach nearly did).

**Relevance to Sam**: supplier product attributes (price band, availability flag, status) and location records (names, admin codes, coordinates) that change across catalogue refreshes are SCD2 candidates. With millions of rows on a laptop, the full-rebuild window approach in DuckDB is usually fine and simplest (DuckDB handles windowed gaps-and-islands well); switch to the incremental merge pattern only when history gets large, and use `IS DISTINCT FROM` for null-safe change detection.

## Zach Wilson (DataExpert) — Data Modeling: Graph Databases & Additive Dimensions, Day 3 Lecture (https://www.youtube.com/watch?v=XWaJFretFec)
**Know which dimensions are additive before you roll up counts; use enums for low-cardinality categories (quality, partitions, documentation); integrate many heterogeneous sources via a "little book of enums" + flexible map schema; and model relationships with the universal vertex/edge schema.**

**Additive vs non-additive dimensions**
- Definition: a dimension is additive over a time window iff an entity can hold only one value of it within that window. Then subtotals sum to the grand total.
- Example: Honda drivers != Civic drivers + Corolla drivers (one person can drive both); Honda cars = sum of models (a car is one model). Users on app != iPhone users + Android users. Dimensions that are a user's "tool" (device, car, platform) are the usual non-additive suspects.
- Time matters: at a 1-second grain almost anything is additive; over a day/month it may not be. Rule of thumb: "can an entity have two values of this in one day?" If yes, non-additive for daily counts.
- Only COUNT (distinct entities) and ratio metrics built on counts (miles per driver) are affected. SUM of a measure (miles driven) stays additive because a mile can't be driven in two cars.
- Consequence: for non-additive dims you can't roll up from partial aggregates; you must go back to row-level data and COUNT DISTINCT (expensive). Facebook built a framework ("Milky Way") to handle non-additive aggregations efficiently. Most dimensions (he estimates ~80–90%) are additive. Also matters for the two-stage salted aggregation trick (see Spark notes).

**Enums**
- Use when cardinality is small; rule of thumb < ~50 values (country with ~200 is too many).
- Benefits: (1) built-in data quality — a value outside the enum fails the cast and stops the pipeline; (2) static attributes travel with the enum in code (e.g. unit-economics line item -> revenue or cost), cheaper than even a broadcast join; (3) built-in documentation of all possible values.
- Enums make great sub-partitions: Facebook notifications partitioned by date AND channel (SMS, email, push, logged-out push — the last used where phones are shared in developing markets). Because the value set is exhaustive, the pipeline knows exactly which partitions to wait for, and dedup could run per channel in parallel.
- Share enums between logging and ETL via a schema IDL such as Thrift so producers and pipelines agree when values are added/removed.

**"Little book of enums" pipeline pattern (large-scale data integration)**
- Problem: one output table fed by dozens of heterogeneous upstream sources (Airbnb unit economics: fees, coupons, credits, insurance, infra cost, taxes, referral credits; Netflix infrastructure graph: apps, databases, servers, codebases, CI/CD; Facebook family of apps).
- Design: an enum (in Python/Scala) lists every source group. Each value has a source function that maps its raw data into one shared schema; a shared ETL calls them. The enum is also materialized as a tiny table (20–40 rows) holding per-value data quality thresholds (week-over-week / month-over-month bounds differ: what's anomalous for fees isn't for coupons). DQ checks join to it. Output partitioned by date + enum sub-partition.
- Adding a source = add enum value + source function; quality and documentation (query the book) come for free. Zach reports pipeline quality rose dramatically wherever he used it.

**Flexible schemas**
- Avoid a 500-column, mostly-null union table; use MAP columns (e.g. map<string,string>) for source-specific attributes, plus an `other_properties` map for rarely-queried fields you don't want to formally model.
- Pros: add keys without ALTER TABLE, no sea of nulls. Limits: ~65k keys per map in Spark-land (he never exceeded ~1,000).
- Big con: compression — maps (and JSON) are the worst-compressing types because the key names are stored as data in every row instead of once in the schema. Struct is NOT flexible (fixed keys/types); arrays are semi-flexible (one element type).

**Graph data modeling**
- Relationship-focused, not entity-focused; schemas for properties are deliberately loose.
- Universal vertex schema: `identifier, type, properties (map)` (e.g. player name, 'player', {height, weight, draft_year}).
- Universal edge schema: `subject_identifier, subject_type, object_identifier, object_type, edge_type, properties` where edge_type is a short verb ("plays_with", "plays_against", "plays_on", "is_a", "has_a") and properties describe the relationship (years on team, start date). Read it as a sentence: subject verb object. Vertex types are themselves an enum — the same idea as the little book.
- Why: multi-hop traversals ("how is X connected to Y") become cheap compared with chains of relational joins and repeated scans. Netflix built a graph of 70–80 entity types for infrastructure/security. The data layer can be built engine-agnostic in tables before loading into any graph DB.

**Rules of thumb**
- Ask "can an entity be two of these in the window?" before summing subtotals or building pre-aggregated cubes.
- Enum when < ~50 values; share enum definitions across logging and pipelines.
- Map columns for heterogeneity, but promote hot keys to real columns for compression and usability.
- Per-source DQ thresholds, shared schema.

**Failure modes**: double counting users across devices/products in rolled-up dashboards; ratio metrics silently wrong after roll-up; enum explosion (country as enum); giant mostly-null union tables; map-bloated storage.

**Relevance to Sam**: Trameter supplier integrations (Viator, RateHawk, Coras...) are a textbook little-book-of-enums case: an enum of suppliers, one mapping function per supplier into a shared product/location schema, per-supplier DQ thresholds (row counts, price ranges), output partitioned by date/supplier, with supplier-specific extras in a map column. Non-additivity bites when counting distinct travellers/products across suppliers or channels. The vertex/edge schema fits location hierarchies, transit stop-route-agency links, and travel-access agreements between countries.

## Zach Wilson (DataExpert) — Data Pipelines in 8 minutes: Streaming, Batch, and On-demand (https://www.youtube.com/watch?v=11lQbLQhIrM)
**A pipeline moves data from sources into one place so it can change decisions; ~90% should be scheduled daily batch, a small minority real-time, and a sliver on-demand.**

- Definition: a flow of information that ends as insight for a decision (clicks -> events table -> enrich -> "17 clicks in China"). Data has value only to the extent it changes or improves decisions; the more decisions, the more value.
- Sources: clickstream/app events, production databases (users, posts), third-party SaaS (Salesforce, Google Analytics, HubSpot). Goal: consolidate into one place so they can be analysed together. Historically a relational DB (still fine at small scale); for ~10 years the norm has been a cloud data lake of files on object storage (S3 / Azure / GCS equivalents).
- AI angle: consolidating structured plus unstructured data (video, text, audio) is "context engineering" — the same pipeline discipline feeds LLM context.
- Scheduling: cron is the primitive under every orchestrator (Airflow, Dagster, Prefect, Mage, Databricks Workflows). Computers count time as epoch seconds since 1970-01-01. (note: Zach says epochs are "16–17 billion" and "every ~80,000 seconds" is a day; actually current epoch seconds are ~1.7–1.8 billion and a day is 86,400 s. Also cron schedules by calendar fields — minute, hour, day-of-month, month, day-of-week — with 1-minute minimum granularity, not "seconds between tasks"; orchestrators add schedules, retries and dependencies on top.)
- Three pipeline types:
  1. Scheduled batch (~90%): run daily (or hourly) and process that period's data. Default choice.
  2. Real-time / streaming (~5–10%): always running, listening for events; needed when daily is too late (fraud detection on a stolen card). Costs: more expensive, harder to maintain, more skills required.
  3. On-demand / event-triggered (~1–2%): runs when something happens (user signs up -> fetch data to build profile).
- Rule of thumb: start with daily batch; justify streaming by a concrete latency-cost argument; use triggers for per-entity enrichment.

**Relevance to Sam**: most Trameter data work (GeoNames/OSM refreshes, GTFS feed updates, supplier catalogue pulls) belongs in daily or weekly scheduled batch with date-partitioned outputs; on-demand pipelines fit per-user/per-booking enrichment (e.g. fetch supplier details when a trip button is created); reserve streaming for genuinely time-critical signals like booking/payment fraud or live availability.

