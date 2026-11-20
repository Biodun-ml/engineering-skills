# zach-2 — cross-cutting principles

1. **Conceptual modeling beats physical tuning.** Decide what data is worth having before deciding how to store it. Cut low-ROI columns/tables early; fix bad logging at the source. (One upstream conversation saved Facebook ~3 PB/month.) [Zach: Data Modeling in 8 min; OBT vs Kimball vs Relational]
2. **Default to Kimball (facts + dims + SCD2) for ~80–90% of needs.** Reach for OBT (ARRAY<STRUCT> at entity grain) when long-horizon analyses would otherwise scan/join huge facts. Use cumulative aggregates only for fixed metrics with no event context. Mixing paradigms is normal (Airbnb: vault-style raw inputs → OBT pricing table). [OBT vs Kimball; 8-min modeling]
3. **Keep raw inputs untouched and re-processable** (data-vault idea, dated snapshots partitioned by `ds`). Snapshots answer questions production can't (deleted rows, past state). [8-min modeling; OBT vs Kimball]
4. **Cumulative table design = yesterday's snapshot FULL OUTER JOIN today's data.** COALESCE stable attributes (order matters: earliest first_date, latest last_date); handle new / continuing / absent-today; carry state forward without appending nulls; derive state metrics (days since active, last class) in the same pass. Seed from an empty "yesterday" and roll forward period by period. [Cumulative Dimensions lab; Growth Accounting]
5. **"No data is data."** Absence of activity must be tracked; only a full outer join can see it. It powers growth accounting (new/retained/churned/resurrected/stale), deletion detection and survivorship curves. [Growth Accounting; OBT vs Kimball]
6. **Growth accounting is a general state-transition monitor**, not just for users. Use it for classifier outputs (fake accounts, risky hosts), catalogs, or supplier listings; sudden mass transitions flag model or feed bugs. [Growth Accounting]
7. **Avoid shuffle; prefer map-only work.** Arrays at entity grain allow `reduce`/index operations with no GROUP BY ("infinitely parallelisable"). Sorted, entity-contiguous rows compress well with run-length encoding. Always PARTITION BY in window functions on big data to avoid OOM. [Cumulative lab; OBT vs Kimball; Growth Accounting]
8. **Aggregate from a daily per-entity staging layer, not raw facts.** Joins become 1:1, slicing is cheap, and experiment metrics come for free. Watch cardinality explosions (dims × days) and always show raw counts beside percentages. [Growth Accounting]
9. **SCD Type 2 is the only SCD type you need.** It keeps full history, is the only type that backfills correctly, and collapses 365 daily snapshot rows into a handful. [OBT vs Kimball]
10. **Materialize reused intermediates.** CTEs/views store logic and re-run it each time. For debuggability prefer staging tables with short retention over temp tables. Full-refresh materialized views aren't incremental (Postgres concurrent refresh needs a unique index). [CTE vs Temp Tables]
11. **Make transformations pure functions and unit-test them** with tiny hand-built input/expected frames (pytest + chispa), including duplicates and null/no-match join sides. Keep I/O in `main()`. Tests let others safely change libraries you depend on, because CI blocks them. [Testing Spark Jobs]
12. **Fail loudly.** Silent failures are the enemy: raise on unparseable or duplicate records. Catch bugs in dev first, then with write-audit-publish before prod tables, and never rely on analysts as the quality system. [Testing Spark Jobs; MIDAS]
13. **Spec before SQL (MIDAS-lite).** Purpose, flow diagram, DDL with a comment on every column, quality checks, metric definitions (flag guardrails), example queries. Get technical and stakeholder review. Backfill **one month** first and have someone else validate it, because backfills are the most expensive mistake. Reserve heavy process for multi-year master data whose sources are stable. [MIDAS]
14. **Tiered quality checks by table type.** Basic checks everywhere (non-empty, not-null, unique key, enum validity). Row counts week-over-week, not day-over-day; seasonality-aware checks when infra allows. Dimensions: grow-but-not-too-fast plus business invariants. Facts: dedup, referential integrity, no "is growing" check. Don't repeat checks already guaranteed upstream. [MIDAS]
15. **Conventions are leverage:** `fact_`/`dim_`/`scd_`/`agg_` table prefixes, `dim_`/`m_` column prefixes, UTC timestamps stated in comments, partition column `ds` last. An `aggregation_level` column lets one agg table hold many grouping sets so the metric layer only selects. [MIDAS]
16. **"Real time" usually means "reliable by a known time".** Have the SLA discussion, and ask what decision would change with fresher data. Choose the cheapest tier: daily → hourly → micro-batch → streaming. Stream only when latency makes or breaks the product and ≥3 people can operate it. [Kafka & Flink]
17. **Memory-bound work needs hierarchical processing.** Global dedup over a whole day OOMed even at Facebook in streaming. Micro-batch + tree dedup fixed it. Sub-hour windows suit streaming; longer windows and dedup suit batch. On a 16GB laptop: chunk, dedup per partition, then merge, and let DuckDB spill. [Kafka & Flink]
18. **Stream processing essentials:** event-time watermarks (required for windows), checkpoints + explicit offset policy (earliest vs latest) for restarts, parallelism matched to Kafka partitions, avoid Python UDFs in hot paths. Iceberg's partition appends unify streaming writes with batch reads and make Kappa backfills feasible. [Kafka & Flink]
19. **Data engineering is engineering.** Build "information highways" (reusable models answering current *and future* questions), not one pipeline per question ("bespoke hell"). Readable code over micro-optimisations. Learn the patterns (aggregation, cumulation, window) so the SQL — or the LLM writing it — follows. [Testing Spark Jobs; MIDAS; Growth Accounting]

---

## Zach Wilson (DataExpert) — Learn Data Modeling in 8 minutes: Dimensional Data Modeling, Data Vault, and One Big Table (https://www.youtube.com/watch?v=WDwNow61JVE)
**Data modeling happens at three layers (conceptual → logical → physical), and the physical paradigms (dimensional, OBT, data vault) can be mixed to fit the business.**

- **Mental model — three layers.**
  - *Conceptual*: what data does the business need, what sources exist, is it feasible, what are the relationships at a high level. This is where the biggest wins are: pushing back on a low-value requirement (a column a data scientist will use once) removes work forever. Some data is not ROI-positive: months to acquire versus little decision value.
  - *Logical*: identify facts (events/verbs: login, purchase, message sent — immutable once they happen) and dimensions (nouns/actors: user, listing, device, browser) and how they relate. Dimensions enrich facts ("a user in China clicked an ad": China is user context, the click is the fact).
  - *Physical*: schema, column types, storage format, compression. People wrongly think this is all of data modeling; it is where the model "becomes real" but is arguably the least important layer.
- **Physical paradigms and trade-offs.**
  - *Dimensional (Kimball facts + dims)*: separate nouns and verbs, join at query time. Oldest, most foundational; the default most engineers should pick.
  - *One Big Table (OBT)*: denormalise all context into the fact so queries are pure aggregations, with no joins. Compute/storage trade-off: user attributes duplicated on every event (300 events → 300 copies), but group-bys are much cheaper than large joins at scale. Telltale sign of OBT: heavy use of complex types (STRUCT, ARRAY, MAP), i.e. "a table within a table".
  - *Data Vault*: preserve the rawest, untransformed form of inputs and stack them, so you can always go back if a filter/transform was wrong. Related to ELT. Strict methodology; Zach doesn't favour it as a whole, but its "keep raw inputs" idea is useful.
- **Worked example (Airbnb pricing/availability).** Raw host rules (daily price, discounts, min length of stay, advance-booking window) landed raw into an `inputs` table (vault-style) → processed into `listing_pricing`, an OBT holding all nights in one array column (the actual price per night given the rules). Dimensional modeling around it. Lesson: mix paradigms; it's an art matching business requirements to physical tools.
- **Rules of thumb.**
  - Spend the most thought on the conceptual layer; cut low-ROI requirements early.
  - Default to dimensional; reach for OBT when join cost dominates and consumers mostly aggregate; keep a raw-input layer so you can reprocess.
- **Relevance to Sam.** Supplier feeds (Viator/RateHawk rules, prices, availability) map naturally onto this pattern: land raw supplier payloads untouched (vault-like, re-processable), derive a computed pricing/availability table, and serve analytics from a denormalised table. In Parquet/DuckDB/Polars, nested STRUCT/LIST columns are supported natively, so the OBT-with-arrays pattern works on a laptop.

## Zach Wilson (DataExpert) — Common Table Expressions vs Subqueries vs Views vs Temp Tables for data engineers (https://www.youtube.com/watch?v=vstJyDo88kA)
**Split SQL abstractions into "store logic only" (CTE, subquery, view) and "store results" (temp table, materialized view, staging table), and pick based on reuse, sharing and refresh cost.**

- **Mental model.** Five ways to hold a transformation:
  - Non-materialized: CTE, subquery, view. They store *logic*, not results, so every reference re-executes the query.
  - Materialized: temporary table, materialized view (plus the "plain staging table" alternative). They store *results*, so reads are cheap.
- **CTE vs subquery vs view.**
  - In modern analytical engines (warehouses, lakes, Postgres recent versions), CTEs and subqueries have identical performance; pick the CTE for readability (a named step).
  - Exception: old MySQL/older Postgres, or engines where `WITH` breaks index optimisations (CTE as an optimisation fence); use subqueries there.
  - View = a named, persistent query. Use only when the logic genuinely must be shared across several pipelines/consumers, e.g. to hand analysts a simplified interface over complex logic.
  - Gotcha: views create dependencies. Dropping or moving the underlying tables is blocked, which makes data harder to reorganise.
  - Bias: CTE by default; view only when shared; subquery only on legacy tech.
- **When to materialize.** If one expensive transformation feeds several downstream transformations, a CTE recomputes it each time. Materialize it once. Demo: a self-join (players in the same match, different gamertag) took ~13 s; after `CREATE TEMPORARY TABLE` the downstream queries (teammate analysis where teams equal, enemy analysis where teams differ) ran in ~1–2 s each.
- **Temp table vs staging table vs materialized view.**
  - *Temp table*: lives for one session; auto-deleted, so no cleanup. But it disappears when the session ends, so you can't inspect the intermediate step when debugging.
  - *Staging table*: an ordinary table with short retention (e.g. keep ~5 days). Survives the session for troubleshooting; you own cleanup. Often the better choice for ETL helper steps.
  - *Materialized view*: persists; reads are instant; must be `REFRESH`ed. Plain refresh is **blocking** for readers. `REFRESH MATERIALIZED VIEW ... CONCURRENTLY` (Postgres) is non-blocking (readers see old data) but **requires a unique index**, so the grain (here analysis player, other player, match_id) must be in the view. Concurrent refresh is slower (6 s → ~45 s in the demo). Bonus: the unique index enforces a uniqueness/data-quality guarantee.
  - Big drawback of materialized views: always recompute the whole dataset, never incremental. Zach prefers building incremental tables with the plain table API (insert per partition) over materialized views.
- **Rules of thumb.**
  - Reused intermediate result → materialize (temp or staging table).
  - Need to debug intermediate steps → staging table with retention, not a temp table.
  - Need persistent precomputed results that are cheap to recompute fully → materialized view with a unique index and concurrent refresh.
  - Large or growing data → incremental table rather than a full-refresh materialized view.
- **Relevance to Sam.** In DuckDB the same logic applies: a CTE referenced several times may be recomputed, so `CREATE TEMP TABLE` or write an intermediate Parquet file for expensive joins/spatial joins (e.g. GeoNames matching) that feed several outputs. Writing intermediate Parquet "staging" files also gives restartability on a 16GB laptop: a crash doesn't lose hours of work. DuckDB has no materialized views; use tables/Parquet with explicit rebuilds.

## Zach Wilson (DataExpert) — Data Modeling: Cumulative Dimensions, Struct and Array, Day 1 Lab (https://www.youtube.com/watch?v=dWj8VeBEQCc)
**Build a cumulative dimension table: one row per entity per snapshot date, with history folded into an ARRAY of STRUCTs, built by repeatedly FULL OUTER JOINing "yesterday's cumulative table" with "today's new data".**

- **Problem.** `player_seasons` has one row per player per season. Player-level attributes (name, height, college, country, draft year/round/number) are duplicated on every row, and the temporal grain means a downstream join shuffles rows and breaks sorting, which ruins run-length-encoding compression in columnar files.
- **Fix: push the temporal component inside the row.**
  - Define a struct type `season_stats(season INT, gp INT, pts REAL, reb REAL, ast REAL)` (Postgres `CREATE TYPE`).
  - Table `players`: stable attributes + `season_stats season_stats[]` + `current_season INT`. Primary key = `(player_name, current_season)`. Each `current_season` partition is a full snapshot of all players ever seen.
  - Classify attributes carefully: age and weight change per season, so they belong to the season struct (or are derived), not the player. Draft year is TEXT because "Undrafted" exists.
- **Cumulative query mechanism ("yesterday + today").**
  - `yesterday` = `players WHERE current_season = N-1`; `today` = `player_seasons WHERE season = N`; `FULL OUTER JOIN` on player_name.
  - Stable columns: `COALESCE(t.col, y.col)`.
  - Array, three cases: yesterday null → new one-element array `ARRAY[ROW(...)::season_stats]`; today not null → `y.season_stats || ARRAY[ROW(...)]`; else (inactive/retired) → carry `y.season_stats` forward unchanged (don't append nulls).
  - `current_season = COALESCE(t.season, y.current_season + 1)`.
  - Seed query: the first run uses an empty "yesterday" (1995 when data starts in 1996), so it degenerates to today's data. Then roll forward one period at a time (`INSERT INTO players ...`, bump both years). Re-running the same period hits the primary key; that's the duplicate guard.
- **Derived cumulative metrics (computed in the same pass, no history scan).**
  - `years_since_last_season`: 0 if active this period, else `y.value + 1`; resets to 0 on comeback (Michael Jordan: 3 in 2000, 0 in 2001).
  - `scoring_class` ENUM (star >20 pts, good >15, average >10, bad): computed from today's points when active, otherwise carried forward from last value. Enums constrain allowed values (a light data-quality contract).
- **Going back to the original grain.** `UNNEST(season_stats)` then expand the struct (`(s::season_stats).*` in Postgres; `.*` works directly in Spark/Presto). So you can join at entity grain, then explode, and rows for each entity stay contiguous/sorted → good compression.
- **Analytics with no GROUP BY.** "Most improved" = `season_stats[cardinality(season_stats)].pts / season_stats[1].pts` (guard divide-by-zero with CASE). No shuffle, no reduce step: pure map-side work, "infinitely parallelisable", very fast in Spark/Presto. Filter on `scoring_class = 'star'` for meaningful lists.
- **Rules of thumb.**
  - Separate stable entity attributes from per-period attributes before designing.
  - Always handle three join cases: new entity, continuing entity, entity absent today.
  - Don't append nulls for inactive periods; carry state forward.
  - Keep state metrics (days since last active, last class) cumulatively so you never re-scan history.
- **Failure modes.** Forgetting a column in the INSERT gives confusing type errors; forgetting the "absent today" branch either drops retired entities or pollutes arrays with nulls; dividing by first-period values of 0.
- **Relevance to Sam.** Great for supplier/location entities that evolve daily: e.g. one row per hotel/product/place per snapshot with an array of daily price/availability structs and "days since last seen in feed" (detect delisting). Each day's run only reads yesterday's snapshot + today's feed, which is small, idempotent per partition and backfillable sequentially. DuckDB and Polars support LIST/STRUCT, `list_concat`, `unnest`, and FULL OUTER JOIN, so this works on a 16GB laptop, partitioned as one Parquet file per snapshot date. Caveat: arrays grow unbounded, so cap history length (e.g. last 30/365 entries) for long-lived entities.

## Zach Wilson (DataExpert) — Data Modeling: One Big Table vs Kimball vs Relational for data engineers (https://www.youtube.com/watch?v=ltQgbSs99WU)
**Relational modeling serves production apps, Kimball (facts + dimensions + SCD2) covers ~80–90% of analytics, and One Big Table (arrays of structs at entity grain) wins for long-horizon analyses where shuffle/joins dominate. Pick per question.**

- **Why so many techniques:** each has different strengths.
  - *Relational ("production") modeling*: great for filtering to a few records (one user, a dozen events) and for integrity (PKs, FKs, dedup enforced by the DB). Bad for analytics: answering "what affects price" at Airbnb was a ~25-table join. Holds only the latest state, no history.
  - *Kimball/dimensional*: the core skill (and the standard interview question: identify dims, facts, aggregates). Good for OLAP aggregates, large and historical data. Not for transactional updates; you load daily pulls.
  - *OBT*: Kimball denormalised further: a "table within a table" (ARRAY<STRUCT>) so aggregations need no join and no GROUP BY. Great for long-term analysis. Cons: complex types are hard for analysts to query; copies fact data into the dimension grain (trades cheap lake storage for expensive compute).
- **Canonical big-tech architecture.** App writes to a production DB (often sharded MySQL). Events are logged to Kafka → landed to the lake (e.g. hourly Spark job, hourly/daily partitions). Production tables are snapshotted daily (`SELECT *` at midnight UTC) into lake tables partitioned by snapshot date (`ds`). Facts (Kafka) + daily snapshotted dims (prod) → master data, OLAP cubes, metrics, features.
- **Upstream matters.** Normalise first, then denormalise (credited to Joe Reis). Bad prod modeling → "bad data on bad data". Check keys and ER diagrams, talk to source owners. Logging is where SWEs make the most mistakes because logs don't break their app ("thrown over the fence"). Story: a Facebook notifications log copied the user's full experiment array on every event (~50B rows/day). Dropping that column and joining to the user-experiment table saved ~3 PB in a month. One conversation upstream can beat any pipeline optimisation.
- **Snapshots enable questions prod can't answer.**
  - E.g. count deleted posts containing some text. Prod deleted them; logs have post_id but (correctly) not content. Daily snapshots retain it. Privacy policy decides how (Facebook anonymised user ids for deleted data).
  - Detect deletions via cumulative pattern: FULL OUTER JOIN today vs yesterday snapshot, `WHERE today.id IS NULL`.
  - Limitation: daily snapshots miss intra-day create+delete (logs see the events but not the content). Most companies only care about day-over-day change anyway.
- **SCD Type 2.** Daily snapshots of rarely-changing columns (name, email, country) are wasteful: 365 rows/year for one change. Collapse to SCD2 rows with `start_date`, `end_date`, `is_current`, so 365 rows → 2. Airbnb's view: Type 2 is the only correct type because it keeps all history and is the only one you can backfill correctly; other types can be derived from it. Posts SCD tracks `is_deleted` and `post_text` (edits).
- **Where Kimball strains.** "How many likes did those posts get before deletion?" means joining the posts SCD to a huge `post_actions` fact. Posts span years (2009–2018), so you must scan all of history or impose a creation window *and* a forward-looking cap (e.g. likes within 1 year). Hundreds of partitions scanned.
- **OBT solution.**
  - `post_obt(post_id, actions ARRAY<STRUCT<action_type, action_ts>>, ds)`, built cumulatively: each day concat new actions onto yesterday's array.
  - Like count via Trino `reduce(actions, 0, (s,x) -> s + IF(x.action_type='like',1,0), s -> s)`. No GROUP BY, no shuffle: effectively a simple select, "infinitely scalable".
  - Then join the small result to posts SCD.
- **Simpler alternative and when it fails.** A cumulative daily counter table (`post_likes_cumulative(post_id, like_count)`) is better when you need one or two metrics with no extra context. It breaks when a new question needs event-level context (e.g. likes only between 16:20–16:25 daily): the aggregate lost timestamps, so you need a new pipeline. With OBT you just add a condition inside the `reduce` lambda. Trade-off spectrum: raw facts (flexible, expensive scans) ↔ OBT (flexible, cheap reads, storage/complexity cost) ↔ cumulative aggregates (cheapest, inflexible).
- **Facebook 2017 example.** Facebook's first-ever growth dip: Zach built an OBT so analysts could map array functions over ~10 years of data without shuffle.
- **Rules of thumb.**
  - Kimball for most needs; OBT for long-history analyses needing event context; cumulative aggregates for fixed metrics.
  - Convert daily dimension snapshots of slow-changing columns to SCD2.
  - Fix data at the source when possible.
- **Relevance to Sam.** Supplier catalog/availability data is best kept as dated snapshots (Parquet partitioned by `ds`). Derive SCD2 for hotel/product attributes (name, address, coordinates, status) to see when things changed and to detect delistings via a today-vs-yesterday outer join. DuckDB supports `list_reduce`/`list_filter`/lambda functions on LIST columns for OBT-style no-GROUP-BY queries.

## Zach Wilson (DataExpert) — Testing Apache Spark Jobs in CI/CD, Week 3 Day 3 (https://www.youtube.com/watch?v=QH1339Fy40Y)
**Catch data bugs in development with unit/integration tests on pure transformation functions (PySpark + chispa + pytest), and treat data engineering as engineering, with software-grade quality standards.**

- **Where a quality bug can be caught (best → worst).**
  1. *In development*: fixed before deploy; effectively never existed, no postmortem.
  2. *In production but before production tables*: write-audit-publish (WAP). Write to a staging table with the prod schema, run audits, and publish only if they pass. Costs: data delays and on-call troubleshooting; false-positive checks are a real tax. The signal-table pattern is an alternative (Facebook prefers it; Zach thinks WAP is better).
  3. *In production tables*: an analyst notices a weird dashboard spike. Bugs can hide for weeks. Worst: people or algorithms act on bad data (a Facebook bug over-labelled accounts as fake). Analysts are not a scalable quality system.
- **Why software engineering has higher quality bars.**
  - Consequences: site down = 100% revenue loss and a 3 a.m. page; a broken pipeline can often wait till morning on stale data.
  - Maturity: ~50 years vs ~10–15; TDD/BDD are rare in DE.
  - Background: DEs come from analytics, economics, etc., so their SWE fundamentals vary.
- **Why DE is getting riskier.**
  - Data feeds revenue-critical ML. Airbnb Smart Pricing: they measured revenue lost per day of data staleness. Staleness degrades models slowly, then rapidly, rather than 100% at once.
  - Metric bugs corrupt A/B test conclusions.
  - Data-driven cultures rely on DE more. Seniority means owning riskier pipelines.
- **Why orgs miss the mark.** At Facebook Zach wrote zero quality checks in 18 months ("move fast and break things"). Bugs were found by analysts, fixed by query edit + backfill, and no check was added. Velocity vs sustainability trade-off: strong managers protect DEs from ad-hoc requests; weak ones let analytics "eat DE's lunch" → burnout.
  - Rule: cut corners only for genuine one-offs; ~99% of data should be built to last. DEs build "information highways", not dirt roads; benefits show in 6–12 months.
- **Where DE value grows over time.** Latency (batch → micro-batch → streaming; 24/7 pipelines have more failure exposure, like more miles driven = more crashes); quality (Great Expectations, Amazon Deequ, chispa); completeness (domain expertise plus stakeholder communication); ease of access (data via APIs, e.g. Netflix data product; good modeling = usability).
- **Software-engineering mindset for DE.**
  - Code is read by humans more than run by machines. Don't trade readability for a 5% speedup (three subqueries vs two CTEs).
  - Silent failures are the enemy. Prefer loud failures: `raise` (Python) / `throw` (Java). Fail the pipeline on unparseable records or duplicates, ideally with custom exceptions (e.g. a DuplicateRecordException). Loud failures also surface in CI.
  - DRY (encapsulate reusable logic) and YAGNI (build only what's needed). SQL is hard to encapsulate and test, which is a tension with DRY.
  - Design docs/specs catch errors early. Linters enforce shared style.
  - Care about efficiency: Big-O, shuffle, accidental cartesian joins.
  - Unit tests let you safely depend on other teams' libraries: if they break your test, CI blocks their merge (Airbnb pricing libs example).
  - LLMs will automate the SQL/analytics layer sooner than testing/optimisation, which depend on data shape. Learning these fundamentals keeps options open (SWE, MLE).
- **Lab mechanics (PySpark).**
  - Structure: `jobs/` holds each job with a `main()` (build SparkSession, read, write) and a separate pure function `do_x_transformation(spark, df)` that registers a temp view and returns `spark.sql(query)`. You test the transformation only; that's 99% of the risk.
  - `tests/conftest.py`: a session-scoped pytest fixture providing one `spark` for all tests.
  - Test: define input and expected rows as `namedtuple`s → `spark.createDataFrame` → run the transform → `chispa.assert_df_equality(actual, expected, ignore_nullable=True)`. Test functions must start with `test_`.
  - Demo: adding `+1` to a year makes chispa print every mismatching row. If the change is intended, update the test, which enforces the new logic.
- **Gotchas hit while porting Postgres SQL to Spark.**
  - `ROW_NUMBER()` requires `ORDER BY` in the window in Spark.
  - Reserved words like `type` need backticks.
  - `json_build_object` → use `MAP`/`STRUCT`. Struct vs map schema mismatch: map values must share one type, so cast year to string.
  - Nullable flags differ between created and computed frames → `ignore_nullable=True`.
  - Typos in map keys (`year_founded`) are caught by row comparison.
  - Include deliberate edge cases in fixtures: a duplicate row with the same id to verify dedup. For cumulative/join jobs, use two input frames and cover left-null, right-null and no-match cases.
  - Spark tests are slow-ish to start (~10 s).
- **Relevance to Sam.** The same structure works without Spark: keep DuckDB/Polars transformations as pure functions (`def transform(con_or_df) -> df`), test with tiny hand-built frames in pytest (`polars.testing.assert_frame_equal`). Encode edge cases from location matching (duplicates, nulls, missing joins, Unicode names) as fixtures. Raise loudly on parse failures from supplier APIs instead of silently dropping rows. Use a staging-then-publish (WAP) step for master location tables.

## Zach Wilson (DataExpert) — Build a Gold Pipeline like Airbnb MIDAS Process, Week 3 Day 1 Analytics (https://www.youtube.com/watch?v=JiedBnTFCeg)
**Data quality = trust + impact. Get both by writing and reviewing a spec before writing SQL, validating a small backfill with someone else's eyes, and attaching the right quality checks per table type.**

- **What "data quality" means.**
  - Discoverable (people can find the data exists; a catalog/infra concern).
  - Complete enough for the decision. Zillow's model missed variables; black-swan events like COVID can't be predicted.
  - Correct at row level: not null where required, no duplicates in dimensions.
  - Tied to business value: revenue (directly, or via leading indicators such as signups → retention → ad views) or cost savings (e.g. tracking AWS spend). Rarely, strategic one-off datasets.
  - Easy to use: obvious column names; Airbnb prefixes `dim_` for dimension columns and `m_` for metric columns.
  - Timely: agree an explicit refresh SLA. Zach's unit-economics pipeline had a 3-day SLA, so a 2-day delay meant "do nothing yet", which cut noise requests.
- **Building trust before production.**
  - Empathy session: ask downstream stakeholders what they'd want if cost/time were no object. It reveals the real problems and a 6–12-month roadmap (a staff-level skill: deciding what to work on next).
  - Design for current *and future* needs, not the narrow burning question (e.g. "availability in Brazil July 2023"). Otherwise you end up in "bespoke hell": one pipeline per question.
  - Clarify business impact in the spec; it also counts for performance reviews before delivery.
  - Stakeholder buy-in up front makes them co-owners and surfaces communication problems before 1,200 lines of SQL exist.
  - Backfills are expensive (I/O is a top cost), so front-load thinking to avoid backfilling 3–4 times.
- **Airbnb MIDAS: 9 steps.**
  1. Spec.
  2. Spec review: technical (another staff DE/architect, never yourself) plus stakeholder.
  3. Build + backfill of **one month only**.
  4. SQL validation by someone else (analyst/AE). Builders are blind to their own bugs; every one of Zach's ~8–9 MIDAS pipelines had bugs found here.
  5. Minerva (metrics layer) validation.
  6. Data review + code review by the architect: modeling oddities, and unit/integration tests present.
  7. Minerva migration.
  8. Final review by a (staff) data scientist.
  9. Launch announcement (PSA).
  - Zach thinks it's somewhat too heavy; most companies (e.g. Facebook) only do steps 3–4. Adopting even a few steps helps a lot.
- **When it's worth it.** Multi-year master data driving important decisions and not facing imminent upstream churn. Skip it for exploratory/prototype pipelines, or when the source is mid-refactor (e.g. payments rewrite) and would make you rebuild in three months. Heavy process ≠ value.
- **What a good spec contains.** Purpose/description and the questions it answers; flow diagram (raw + dims → fact (master data) → aggregate → metrics); schemas (DDL) with a comment on every column; quality checks; metric definitions; example queries (essential for array/struct tables).
- **Naming and schema conventions.**
  - Table names carry type: `fact_`, `dim_` (daily snapshot), `scd_`, `agg_`. Follow or create company naming conventions (a big impact opportunity where none exist).
  - Partition column `ds` last. Timestamps always UTC, and say so in the comment.
- **Quality-check tiers.**
  - *Basic*, nearly free, always: table not empty; not-null where required; no duplicates on the key; enum values valid (lakes, unlike Postgres, don't enforce enums).
  - *Intermediate*: row-count checks **week-over-week**, not day-over-day (weekday seasonality). Still false-positive on holidays.
  - *Advanced*: seasonality-adjusted or ML anomaly checks, comparing with last year too (a one-line YAML config at Airbnb once infra exists).
- **Dimension vs fact checks.**
  - *Dimensions*: should grow or stay flat (especially cumulative tables), but not too fast (+100% users in a week is implausible). Check business invariants, e.g. ≤5,000 friends; an active car-insurance policy must have a car. Simpler, fewer false positives.
  - *Facts*: seasonal, can shrink, so no "is growing" check. More prone to duplicates and nulls (logs are less critical to SWEs), so dedup checks matter most. Check referential integrity (user id exists; no notifications to deactivated users). Such checks have even caught security incidents at Facebook.
  - Tip: in Presto, use `approx_distinct` instead of `COUNT(DISTINCT)` for dup checks on large facts (≈99.9% accurate, far cheaper).
- **Lab: spec for a user-growth pipeline.**
  - Metrics table with name, definition, and *is-guardrail* flag. Guardrail = a drop blocks an A/B launch. Signup conversion (signups/hits) is a guardrail for the landing page; purchase conversion (purchases/signups) for checkout. Traffic breakdown by referrer is not a guardrail (experiments don't change where traffic comes from); better pivoted into a few percentage metrics.
  - Flow: raw `events` (JSON blob) → IP-geo enrichment + user-agent/device enrichment (API calls) → `core.fact_website_events` → `core.agg_website_events` → metrics.
  - Fact columns: `user_id` (nullable when logged out), `logged_out_user_id` (murmur3 hash of IP + device, never null), `dim_hostname`, `dim_country`, `dim_device_brand` (nullable for bots), `dim_action_type` (enum), `event_timestamp` (UTC), `other_properties MAP<STRING,STRING>`, `ds`. Unique key: (`logged_out_user_id`, `event_timestamp`). Synthesise an entity id when none exists.
  - Aggregate table holds several grouping sets in one table with an `aggregation_level` column (overall, by action type, by country, by device, by hour…) plus `m_total_events`. The metric layer then just selects and divides.
  - Checks: not-null on key columns, no dupes on PK, hostname format regex, enum check, WoW row counts per major hostname. Don't repeat upstream not-null checks on the aggregate (already guaranteed). Instead: overall rollup ≥ any other rollup, minimum events per hour, hourly seasonality pattern.
  - Writing the spec with a second person caught a missing column (hostname) and the logged-out id design.
- **Relevance to Sam.** For Trameter master-location or supplier datasets: write a short spec (purpose, flow, schema with comments, checks, example queries) before building. Validate a small slice (one country/one supplier) with a colleague before running all ~6M rows. Name tables `dim_`/`fact_`/`scd_`/`agg_`. Add cheap checks (non-empty, not-null coordinates, unique ids, enum validity for place types, WoW row counts per supplier/country). In DuckDB, `approx_count_distinct` is also available for cheap dup sanity checks.

## Zach Wilson (DataExpert) — Data Engineer Design Patterns at Meta: Growth Accounting, Week 4 Day 1 (https://www.youtube.com/watch?v=EIOZY3tMIVg)
**About 95% of analytical pipelines are one of three patterns: aggregation, cumulation (state-transition tracking, survivorship/J-curves) and window functions. Recognise the pattern and the SQL "writes itself".**

- **Mental model.**
  - Think one abstraction layer above SQL. SQL is itself an abstraction over engines.
  - Career scope grows from one pipeline → a family → the warehouse, and patterns are what let you scale.
  - LLMs will increasingly write the SQL; the engineer who knows the patterns directs them.
  - Low-/no-code is generally the wrong higher abstraction for technical people.
  - Enrichment (joins to add columns) is arguably a fourth pattern but belongs before the master-data layer.
- **Aggregation patterns (GROUP BY; SUM/AVG/COUNT, median/percentile).**
  - The most common: ≥ half of analyses. Uses: trends, composition, root-cause analysis.
  - Root cause: decompose a metric's week-over-week move by dimensions. +1M total might be +1.5M US and −0.5M India. A Facebook growth dip turned out to be Ethiopia shutting off the internet.
  - **Don't aggregate from raw facts.** Build a daily per-entity (usually user_id) aggregate staging layer (one row per user per day, metrics as columns). Joins to dimensions are then 1:1 and cheap to shuffle, and the same table powers A/B experiment metrics for free (join assignment group → slice).
  - Gotcha: too many dimensions at once re-identifies individuals and gives meaningless cells ("+50%" = 4→6). Always show raw counts alongside percentages; tiny countries swing 100%.
  - Gotcha: long time frames multiply cardinality (90 days × 200 countries = 18k rows before any other cut). Coarsen time to week/month/year for long-horizon analyses.
- **Cumulation patterns (built on cumulative table design).**
  - Care about today vs yesterday. FULL OUTER JOIN is mandatory because **"no data is data"**: absence of activity must be tracked.
  - *Growth accounting (state-transition tracking)*, with states:
    - new: absent yesterday, active today;
    - retained: active both days;
    - churned: active yesterday, inactive today;
    - resurrected: inactive yesterday, active today;
    - stale: inactive both days;
    - optional deleted/deactivated where retention policies purge users. Facebook anonymised instead, so users stay stale forever.
  - Net growth = new + resurrected − churned.
  - It generalises to any classifier or state. Zach's uses: Facebook notifications; Facebook fake accounts (new fake / reclassified / declassified); Netflix microservice risk classification; Airbnb host-cancellation risk. It's an excellent **MLOps health monitor**: a model change that suddenly reclassifies many entities shows up immediately. It also measures whether interventions (host education) change behaviour.
  - *Survivorship / retention / J-curves*: three parts. A **reference date** (cohort, e.g. signup or diagnosis), a **state check** (active? alive? smoke-free?), and the **curve** of % still passing over time since the reference date. Curves that flatten (or rise) mean a product with a sticky core; curves that decay to zero mean doom. Mind survivorship bias (WWII planes: armour where returning planes had *no* holes).
- **Window patterns.**
  - DoD/WoW/MoM/YoY = "derivatives": spikier, noise-amplifying; they depend on two points, so anomalies a year ago distort today. Airbnb in 2021 compared to 2019 ("year over two years") because 2020 was COVID-distorted.
  - Rolling sums/averages = "integrals": they smooth. Syntax: `SUM(x) OVER (PARTITION BY dim ORDER BY date ROWS BETWEEN n PRECEDING AND CURRENT ROW)`. Stock analogy: 50-day vs 200-day moving average.
  - Rank can often be done with ORDER BY.
  - Gotcha: always PARTITION BY something on big data; an unpartitioned window funnels everything to one task → OOM.
- **Lab mechanics (Postgres).**
  - Table `users_growth_accounting(user_id, first_active_date, last_active_date, daily_active_state, weekly_active_state, dates_active[], date)`.
  - `today` = events grouped by user for the day; `yesterday` = previous partition; FULL OUTER JOIN.
  - COALESCE order matters: `first_active_date = COALESCE(y.first, t.today)` (keep the earliest); `last_active_date = COALESCE(t.today, y.last)` (take the newest).
  - Daily state CASE: y null & t not null → new; `y.last = today-1` → retained; `y.last < today-1` and active → resurrected; t null and `y.last = y.date` → churned; else stale. Filter null user_ids (logged-out events).
  - Weekly: same idea with 7-day windows; churn is exactly when last_active hits 7 days ago (an equality, not `<`; the shipped code had `<`).
  - (note: Zach's weekly "retained" branch had an unresolved bug on camera: everyone stayed retained. It compared against `y.date - 7` (yesterday's partition) and was evaluated before the churn branch, which is an off-by-one window. When implementing, compute everything relative to today's partition date, order CASE branches so inactive-today cases are tested first, and unit-test each state with fixtures.)
  - Facebook does this with bit math on datelist integers (shift/compare bits), which is more efficient than date arithmetic.
  - Cold start caveat: the cumulation start date defines "first_active". Start late and everyone looks new on day 1; this washes out over time. Ideally backfill from the beginning of history.
  - Retention from the same table: `GROUP BY date - first_active_date` with `% in (new, retained, resurrected)`. Zach's site: 100% → ~5% → stabilises ~2%. Add cohort dimensions (e.g. day-of-week of first activity) to compare cohorts; ignore the tail days with tiny cohorts.
- **Relevance to Sam.** Growth accounting applies directly to supplier catalogs and location datasets: per product/hotel/route per day, track new / still-listed / delisted / relisted / gone, and per supplier or per country, to spot feed breakages (sudden mass "churn" = an integration bug, not real delistings). It's also an ML-classifier monitor (e.g. match/no-match labels from location matching across runs). Build a daily per-entity aggregate layer before slicing. In DuckDB, rolling windows with PARTITION BY are cheap; avoid unpartitioned windows on millions of rows.

## Zach Wilson (DataExpert) — Master Real-time Data Pipelines with Kafka and Flink, 3-hr Course (https://www.youtube.com/watch?v=dzB0HMtMIfI)
**Streaming is rarely what stakeholders mean by "real time". Choose the cheapest latency tier that changes a decision; when you do stream, understand watermarks, offsets/checkpoints, windows, parallelism and sinks.**

- **Vocabulary.**
  - *Streaming / continuous processing*: each event processed as it arrives (Flink).
  - *Near real-time = micro-batch*: collect minutes of data, then process (Spark Structured Streaming; Databricks is moving toward continuous, so the two may converge).
  - "Intraday" = runs more than once a day.
  - Four tiers on a latency↔complexity continuum: daily batch → hourly batch → micro-batch → streaming.
  - "Real time" is a myth even with Flink: interceptor→Kafka ~150 ms, Kafka→Flink 0.1 s to seconds, windows add minutes. A Netflix red-team compromise still took minutes end-to-end to alert.
- **What stakeholders mean by "real time".** Usually "predictable/fresh by a time" (e.g. a daily table ready by 09:00). Only ~2 of ~24 such requests in Zach's career truly needed streaming. Hold an **SLA discussion** ("when must it be ready?") instead of building "a Ferrari when a bicycle works".
  - Ask analysts complaining about latency: *what decision would change if it were fresher?* Usually none; business moves on day-scale. Minute-level dashboards can even harm decisions by focusing people on noise.
- **Should you stream? Checklist.**
  1. Does the team have the skill? Avoid a one-person "island". Netflix: 2 of 15 could fix Flink jobs → brutal on-call. Want ≥3 people.
  2. Incremental benefit of lower latency (security 15 min → 2 min: maybe; 3 a.m. → 1 a.m. refresh nobody reads: no).
  3. Homogeneity: a batch team should stay batch, and a streaming-first team should stay streaming (Uber, Kappa). Mixed stacks raise maintenance and on-call load.
  4. Where will data quality checks live? Batch has natural cut points between steps for checks or WAP; streaming is always on.
- **Use cases.**
  - *Streaming-only* (latency makes or breaks the product): fraud detection, high-frequency trading, live sports stats.
  - *Gray areas*: upstream master data that many downstream jobs wait on (earlier landing → whole DAG fires sooner and compute is spread across the day instead of the 00:00–02:00 UTC spike); customer-facing data (customers expect fresh, but an hour-late micro-batch is often fine and easier to quality-check).
  - Zach's failure story: streaming dedup of a 20 TB/day Facebook notification-events fact OOMed. A duplicate click can come at any time of day, so all ids for the day must sit in RAM. Switching to **micro-batch with tree-style (hierarchical) dedup** landed it at 01:00 instead of 09:00 UTC.
- **Streaming ≠ batch operationally.** It runs 24/7 like a web server, so more exposure to failure (more miles driven = more crashes). It needs more unit/integration tests than batch because there's no batch-style quality gate as a second safety net.
- **Architecture pieces.**
  - *Sources*: Kafka (high-throughput one-way firehose, partitions) vs RabbitMQ (lower throughput, richer routing/pub-sub). Dimensional enrichment via Flink **side inputs** from at-rest tables, refreshed on a cadence (e.g. every 3–6 h; tune to how fast dims change).
  - *Compute*: Flink or Spark Streaming.
  - *Sinks*: Kafka topic, data lake (Iceberg), Postgres.
  - Iceberg's key win: **append to existing partitions** (Hive required partition overwrite). Streaming jobs can write tables that batch jobs read.
  - Event generation layer: an HTTP interceptor middleware logs each page/API request (skips dev traffic and static files) to a Kafka producer, swallowing producer errors so logging never breaks the site. Future: Flink → Kafka → consumer → WebSocket to the client, a closed-loop "data product".
- **Streaming challenges and Flink mechanisms.**
  - *Out-of-order events*: a **watermark** on event time with a bound (e.g. `WATERMARK FOR ts AS ts - INTERVAL '15' SECOND`); Flink reorders within it. **Windowed aggregations fail without a watermark** ("group window expects a time attribute"), a bug found live in the lab.
  - *Late data*: `allowed lateness` re-fires or merges already-emitted windows (can produce duplicate or updated rows). Zach usually sets it to 0 and relies on the watermark, unless 100% capture is required.
  - *Failure recovery*: **checkpoints** (Flink-internal state snapshots; interval is in ms, so 10 means 10 ms, which is too frequent; use 10 000). **Savepoints** are portable state for operators and other systems. Kafka startup modes: earliest offset (reprocess everything), latest offset (new data only), or timestamp. `auto.offset.reset` governs restarts. The longer a stream is down, the bigger the backlog.
- **Windows.**
  - *Count*: close after N events per key, plus a timeout because N may never come. Good for fixed-step funnels (notification generated → sent → delivered → opened → clicked).
  - *Tumbling*: fixed, non-overlapping; closest to batch hourly/daily partitions. Common pattern: tumbling windows to hourly data, then batch.
  - *Sliding*: fixed width, overlapping, so double counting if you sum them all. Used to find peaks that straddle boundaries, or to study midnight edge effects: Facebook found DAU could swing ~7–8% just by choosing which 24-hour boundary/time zone, due to sessions spanning UTC midnight.
  - *Session*: per-user, closes after an inactivity gap (e.g. 5–20 min). Best for understanding user behaviour.
- **Performance and operations.**
  - Python UDFs (e.g. an IP-geolocation API call) are slower than JVM UDFs: a separate Python process plus Arrow ser/de round trips. The same applies in Spark.
  - Flink is lazy like Spark: each `INSERT` statement becomes a job. Call `.wait()` or the job ends after the current backlog.
  - Watch **busy** vs **backpressure** in the UI. Backpressure slows consumption from Kafka to avoid OOM during spikes (e.g. reading from earliest offset).
  - Parallelism: Flink default 1 (Spark default 200 shuffle partitions). Match it to Kafka partitions (3 partitions → parallelism 3 ≈ 3× faster). Task slots must cover parallelism × operators. Fixed parallelism is over-provisioned during quiet periods.
  - Postgres sinks are much slower than Kafka sinks; a Flink `CREATE TABLE` for a JDBC sink only declares the schema, so create the real table yourself. Include the JDBC driver jar.
- **Lambda vs Kappa.** Lambda = batch + streaming codebases writing the same data (batch is the correctness backup). Pain: double code. Used by Facebook, Netflix and Airbnb for master data. Kappa = streaming only (Uber), since Flink can also run batch mode. Pain: backfills must replay Kafka sequentially. Iceberg sinks (partitioned, appendable) make Kappa backfills feasible, so Netflix is reconsidering. A common staff-level interview question.
- **Where to aggregate.** Short windows (≤ ~an hour, e.g. 5-min tumbles) suit Flink: emitted then persisted, low RAM, and Spark startup would exceed the window. Longer windows/dedup suit batch: Flink would hold huge state in RAM. Pattern: sub-hour tumble in Flink, then hourly/daily rollups in Spark.
- **Relevance to Sam.** Trameter's needs (supplier catalogs, GTFS feeds, location data) are almost all batch/micro-batch. Agree SLAs, not "real time". If live supplier price/availability is ever needed, consider micro-batch polling into partitioned Parquet/Iceberg before Flink. The memory lesson transfers directly to a 16GB laptop: global dedup over a whole day/dataset in one process won't fit, so dedup hierarchically (per chunk/partition, then merge) or let DuckDB spill to disk. When consuming any queue/API stream, persist offsets/cursors (checkpoint equivalent) so restarts resume rather than re-pull.

