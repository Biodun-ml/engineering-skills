# seattle-data-guy — cross-cutting principles

1. **Layer every pipeline raw -> staging -> production, and keep raw dumb and immutable.** Extract to timestamped, entity-named files with file IDs. Load them with zero business logic, so a QA failure points either at the source or at your own logic and never at both. *(SQL Best Practices: Designing An ETL; What Are ETLs; DBs vs DWs vs Lakes)*
2. **Put a QA gate after every step that adds logic. Production must be "pristine".** Even one day of bad data can drive decisions you can't trace. Decide per check whether to fail the run or only warn. *(What Are ETLs; Designing An ETL)*
3. **Reconcile counts across layers.** Distinct keys and aggregates in raw must match staging/prod after transforms. A lost customer usually means a bad join or filter, and a wrong LEFT JOIN quietly produces null columns. *(What Are ETLs; Designing An ETL)*
4. **A small set of check types catches most problems:** range, accepted-values/category, data type (positional CSVs reorder), freshness, volume (3–10x spikes), null-rate per field, uniqueness. Store expected bands in a config table so adding a check means adding a row, not writing code. *(Data Quality video; What Are ETLs)*
5. **Ask every source how it handles corrections before you model it.** Some append reversing rows that you net (claims 150 then -50). Others replace records by ID, so latest wins and you dedupe. The tables look the same, but the loaders differ. *(What Are ETLs; Data Modeling Where Theory Meets Reality)*
6. **Declare the grain of every table before you build it.** Keep different grains in separate tables. Merging company-level with company x competitor rows creates duplicates that people then hide with AVG in the BI tool. *(Dimensional Modeling 101; End-to-End Project)*
7. **Kimball process: business process -> grain -> dimensions -> facts true to the grain, repeated as you learn.** The facts are usually easy to find. The dimensions are hard because they live in other systems. *(Dimensional Modeling 101)*
8. **Prefer one line-item-grain fact with header attributes denormalised onto it over joining two big fact tables.** Bridge tables risk fan-out double counting. Use weighting factors, arrays or capped repeated columns, and be clear about what each one costs. *(Dimensional Modeling 101)*
9. **History has three main shapes. Choose by query pattern and storage budget:**
   - SCD2 rows with effective dates and a consistent open-end sentinel or is_active flag. Compact, and good for "when did it change".
   - Daily full-snapshot partitions (Facebook ds). Trivial to write and backfill, easy as-of joins, heavy on storage, bad for change analysis.
   - Cumulative/date-list rows per entity (Roblox). Built for billions of events, at the cost of complexity and lost granularity.
   
   Type 1 overwrites silently misattribute history. *(SCDs; Where Theory Meets Reality; Modeling Challenges)*
10. **Multi-source integration is the real modeling problem.** The same entity appears under many IDs. Either pick a master system or build a central ID crosswalk. Make the central ID deterministic/reproducible, because losing a non-reproducible ID forces a full backfill. *(Data Modeling Challenges)*
11. **Normalise operational schemas (no multi-valued cells, no col_1..col_35). Denormalise deliberately for read-heavy analytics.** You can only denormalise what you first normalised. Model conceptual -> logical -> physical, with the business in the loop. *(Why Data Engineers Need Data Modeling)*
12. **Never average averages.** Weight by the underlying counts: sum(n x avg)/sum(n). Join on IDs, not names, and normalise case if you must use names. *(End-to-End Project)*
13. **Every API is different. Design the extractor for auth variety, pagination and failure:**
    - Pagination preference: next-URL/cursor first, offset last, because offset skips or duplicates rows when inserts land mid-crawl.
    - Watch for silent row caps, call counts that explode with nested fan-out, and ~6-monthly schema/endpoint churn.
    - Validate completeness after each pull.
    
    *(Extracting Data From APIs)*
14. **Ask before building: which fields, how often/SLA, how often the schema changes, who consumes it and for what decision.** Then plan for schema drift, volume growth, credential rotation (config, not code) and the maintenance tax of small requests. A control/config table lets one generic pipeline serve many sources. *(What To Consider When Building Pipelines; What Is A Data Pipeline)*
15. **Silent and intermittent failures are worse than hard ones.** Log every file and step (start/end, duration, rows in/out/rejected, status, error with row and field). Trend the logs to find steps that keep slowing down or keep failing: "do analytics on your own systems". *(Designing An ETL; What Is A Data Pipeline; What Are ETLs)*
16. **Orchestrate with a DAG, not cron-time guesses.** Know the moving parts (scheduler, metadata DB, executor, sensors, hooks, XComs for small values only). In production:
    - Deploy DAGs separately from the platform image.
    - Use connections/variables for secrets.
    - Plan for worker and pool contention. At very large scale, self-hosting costs you a platform team.
    
    *(Airflow vs Luigi; Realities of Airflow)*
17. **Match the system to the workload.** Row-store OLTP is for CRUD on current state. Warehouses are integrated, historical and structured. Lakes are cheap, schema-on-read, partitioned files (Parquet over CSV). Never run heavy analytics on production OLTP. Columnar storage helps but doesn't define a warehouse, so use the simplest thing that is fast enough. *(DBs vs DWs vs Lakes)*
18. **Use CDC (WAL/binlog tailing or triggers) when you need low latency or every intra-day change.** A daily batch snapshot loses intermediate states. Buy managed CDC if you can't staff Kafka/Debezium. *(Change Data Capture)*
19. **Grow the stack with maturity, because every tool adds maintenance debt.** Start with ingestion -> storage -> BI, plus naming standards (`_ts`, `is_`), a core-vs-research split and automated tests. Add a catalog/lineage/observability layer when tables and users outgrow tribal knowledge. Reverse ETL without lineage creates circular dependencies. Treat the core data model as versioned infrastructure that changes deliberately. *(Modern Data Stack; Data Catalog; Data Platform)*
20. **Read the query plan.** SQL goes parse -> validate -> relational algebra -> cost-based optimizer -> physical plan (hash/merge/nested-loop joins). Use EXPLAIN, filter and project early, and remember plan caches. *(Understanding SQL Query Execution)*

---

## Seattle Data Guy (Ben Rogojan) — Basic Data Engineering Project: End-To-End From Web Scraping to Tableau (https://www.youtube.com/watch?v=vSgJ3bOyE0w)
**A minimal end-to-end pipeline: scrape -> object storage (S3) -> warehouse stage -> flatten nested JSON in SQL -> a few purpose-built tables at clean grains -> dashboard.**

- **Mental model / first principles**
  - Most "project" tutorials pile on Airflow, cloud services, dbt, etc. Start with the smallest chain that delivers value (raw data -> tables -> answer), then add orchestration once the manual path works.
  - Scraping is hard for two separate reasons: (1) parsing messy HTML/JSON, and (2) infrastructure — sites rotate element IDs, throttle you, detect bots; you need proxies and retries. Buying the infrastructure layer (a managed scraper/proxy platform) lets you spend effort on the analytics, which is where the value is.
  - The standard landing pattern: extractor drops files to object storage (S3/GCS/Azure/SFTP/webhook) -> a trigger/loader moves them into the warehouse. Pick output format (JSON/CSV) and batch vs real-time at the extractor.
- **Mechanisms**
  - Hard-coded input lists (a fixed list of product URLs) are a smell; an automated scraper should crawl from a category page and discover targets itself.
  - Snowflake "stage" = a named pointer to a storage location; you can query raw files in it directly (`$1` = first column of the staged file). In practice point the stage at a partitioned lake path like `lake/<source system>/<entity>/<date>/`, not a single file.
  - Raw payload arrived as one giant JSON array. `LATERAL FLATTEN` unrolls arrays into rows; nested arrays need repeated flattens (product -> top-10 competitors -> categories per competitor = three flattens). Then extract fields with JSON path syntax.
  - Built three tables, each at a single grain: vendor rating (one row per company), vendor-competitor rating (one row per company x competitor), and category comparison (company x category with category average).
- **Decisions & trade-offs**
  - Separate tables per grain instead of one merged table: more tables, but merging company-level rows with company x competitor rows duplicates the company-level values and forces de-dup hacks in the BI tool. He prefers doing shaping in SQL, not in the BI tool's prep layer.
  - The source was already aggregated (avg stars, review counts), so a true fact/dimension model wasn't possible — reviews would be the fact and companies/users the dimensions if raw reviews existed. Know the grain of what you receive before promising a model.
- **Rules of thumb**
  - Never average averages. Compute a weighted mean: sum(reviews x stars) / sum(reviews). With tightly clustered values the error looks tiny, but a vendor with 2 reviews at 3.2 counts as much as one with thousands at 4.8 in a naive average.
  - Join on IDs, not names; he joined on company name only because the data was tiny, and flagged it as bad practice. Normalise case (`LOWER`) before name joins — he hit mismatches from casing.
  - Dashboard layout: headline metrics on top, then progressively more granular detail (comparisons, then raw rows/reviews).
- **Failure modes & gotchas**
  - Duplicates leaked into the final join and he masked them with `AVG` in the BI layer — a classic sign the grain was wrong upstream; fix it in the data source.
  - Live "data QA": sanity-check a few values against the source site to confirm parsing is right.
- **Relevance to Sam**: the stage-and-flatten pattern maps directly to DuckDB (`read_json_auto`, `unnest`) over Parquet/JSON landed on disk; land raw supplier/scrape responses untouched in date-partitioned folders, then build one table per grain. Weighted averages matter for any rating/review aggregation in travel products.

## Seattle Data Guy — Building Data Pipelines Part 1: Airbnb's Airflow vs Spotify's Luigi (https://www.youtube.com/watch?v=xBGmJDD-dbk)
**Orchestrators exist because cron-plus-scripts can't express dependencies, failure, or visibility; Airflow is configuration-style (operators), Luigi is code-style (task classes with requires/output/run).**

- **Mental model / first principles**
  - Use cases: ETL/ELT pipelines, batch ML scoring (hourly/daily model runs when live isn't needed), and plain task automation.
  - Why not hand-roll? The DIY result is a contraption of cron + bash/PowerShell + Python + config files. With cron you *guess* durations ("job A at 12:00 takes ~2h, so schedule B at 15:00"). When an upstream changes and A fails, B still runs on stale/missing data, nobody is alerted, and nothing stops downstream. At hundreds/thousands of pipelines this is unmanageable, and if you build a framework yourself you just reinvent Airflow.
  - Target shape: a DAG (directed acyclic graph) — each task runs only after its upstream tasks finish successfully.
- **Mechanisms — Airflow**
  - Scheduler: cron-like intervals (daily/hourly) plus dependency resolution.
  - Metadata DB (SQLite/Postgres/MySQL): records DAG runs, task instances, start/end times, state (~30 tables). The UI and retries are driven from it.
  - Executor: how tasks physically run — sequential, local (parallel on one box), or distributed across machines. This is where scaling decisions live.
  - Concepts: *operator* (reusable action template: PythonOperator, BashOperator, BigQuery operator); *task* (an instantiated operator); *DAG* (tasks + dependencies); *sensor* (waits for an external event — a file landing, another DAG's task finishing); *hook* (interface to an external system holding stored connections/secrets); *XCom* (small values passed between otherwise-isolated tasks — not for datasets).
  - DAG file anatomy: `default_args` (owner, depends_on_past, start_date, email_on_failure, retries), DAG id, schedule_interval, tasks bound to the DAG, then dependency arrows.
  - Cross-DAG chaining: a file sensor kicks DAG A; a task sensor in DAG B waits for A's task to finish. Easy to build accidental circular/complex webs.
- **Mechanisms — Luigi**
  - A task is a class with `requires()` (upstream tasks), `output()` (a Target, e.g. local file or S3 object), `run()` (the work). A task is "complete" if its output target exists — dependencies are resolved through outputs.
  - Central scheduler mainly prevents the same task running twice concurrently and visualises progress; it does *not* schedule by time — you still need cron to trigger. Weaker logging/history than Airflow.
- **Decisions & trade-offs**
  - Airflow: richer scheduling, logging, UI, ecosystem of operators; more moving parts (scheduler, DB, executor).
  - Luigi: simpler, more Pythonic, output-target-based idempotence; needs external triggering and has thinner observability. (note: Luigi is little used for new projects today; Airflow, Dagster and Prefect dominate. The concepts still transfer.)
- **Rules of thumb**
  - Always set owner, failure alerts and retries at the DAG level.
  - Keep data out of XComs; pass paths/IDs, store data in storage.
- **Relevance to Sam**: even on a laptop, express multi-step location/GTFS jobs as a DAG of steps whose completion is marked by an output file (Luigi's output-target idea is a cheap, idempotent "make"-style pattern: skip a step if its Parquet output already exists).

## Seattle Data Guy — Building Data Pipelines Part 2: How To Create A Task In Luigi (https://www.youtube.com/watch?v=vtZba4pnGuQ)
**A first Luigi task: hit an API, dump raw JSON to a LocalTarget; transformation and loading come as later tasks.**

- **Mechanisms**
  - Task class subclasses `luigi.Task`; `output()` returns `luigi.LocalTarget('raw.json')`; `run()` does `requests.get(url)`, parses the body, and writes it via `self.output().open('w')` with `json.dump`. `requires()` unused here — it will chain transform -> load in later steps.
  - Run with `--local-scheduler` and `module TaskName` (no central scheduler daemon), e.g. on an EC2 box.
  - Example source: StockTwits symbol stream API (messages about a ticker).
- **Pattern worth keeping**
  - Step 1 of an ETL should only *extract and persist raw* — no transforms. Then a transform task (aggregate/count) and a load task (to BigQuery) depend on it. This separation lets you re-run transforms without re-hitting the API.
  - (note: added context, not from the video) Writing through the target's `open()` is atomic in Luigi (writes to a temp file then renames), so a half-written file isn't mistaken for a completed output — a useful idempotence trick to copy in any home-made pipeline (write to `.tmp`, then rename).
- **Gotchas**: messy API bodies may need a custom JSON decode step; hard-coded URL/symbol should become a task parameter.

## Seattle Data Guy — Data Modeling Challenges: The Issues Data Engineers & Architects Face When Implementing Data Models (https://www.youtube.com/watch?v=G3m94JeuAtc)
**Textbook star schemas are the "Titanic dataset" of modeling; real modeling is shaped by storage constraints, multi-source integration and, above all, requirements gathering.**

- **Mental model / first principles**
  - Star/snowflake diagrams (fact in the middle, 4–5 dims around it) teach the vocabulary but hide the hard parts. Kimball vs Inmon debates conflate tactics with the practice; the core question is always *why* each modeling choice serves a business question.
  - Modeling layers: conceptual -> logical -> physical. Most people start and end at physical (tables/columns) and skip the higher-level thinking, which matters more.
- **Mechanisms / real-world examples**
  - **Storage engine shapes the model.** At Facebook the warehouse was HDFS-like: no cheap UPDATEs, so instead of classic SCD rows they wrote a *full snapshot of each dimension every day into a new date partition* (one table, partitioned by `ds`). Consequences: queries must join on ID **and** date; a macro was used to auto-select the latest partition instead of writing `MAX(ds)` subqueries. They still said "dim" and "fact", but these weren't Kimball dims/facts.
  - Why track change at all: if you only store current state (e.g., employee title) you can't answer "how often are people promoted" — the business question dictates whether you need history.
  - **Integration is the real pain.** The diagram implies "customer" comes from one source; in reality regional arms (US/EU/Asia) run different ERPs, the same customer exists multiple times, and reporting must unify them. Choices: UNION them, or merge into one row per real customer — and decide which ID is the master.
    - Option A: designate a master system (e.g., Salesforce) that pushes the canonical record to others. Needs process, cross-team coordination, often an expensive iPaaS tool.
    - Option B: build a central ID/mapping service in the data layer. Risk: if the central ID isn't reproducible/reverse-engineerable from source keys, losing it forces a full backfill.
  - Facebook had the opposite problem: apps were built to share IDs, so they *removed* redundant IDs to avoid confusion.
- **Decisions & trade-offs**
  - Snapshot-per-day vs SCD2: snapshots are trivially simple and backfillable on append-only storage but cost storage and require date-aware joins; SCD2 is compact but needs update/merge logic.
  - Don't add an SCD "because the book says so" — add it when a question requires history.
- **Rules of thumb**
  - Spend most of the effort on requirements: which questions now, which in future, and design so the model adapts quickly. Enterprise models historically took months to a year.
  - In interviews (and real work), ask clarifying questions before drawing boxes.
  - Learn by working on, or reading, real company models; alternative patterns now in play include activity schema and one-big-table.
- **Failure modes**: assuming one source per entity; non-deterministic surrogate IDs; modeling to a template rather than to questions.
- **Relevance to Sam**: location data from GeoNames, OSM and suppliers (Viator, RateHawk) is exactly the multi-source "same entity, many IDs" problem — keep a crosswalk table (source, source_id -> master_id) where the master ID is derived deterministically (e.g., hash of source+source_id or a reproducible matching rule) so it can be rebuilt; daily snapshot partitions in Parquet are a cheap SCD alternative on a laptop.

## Seattle Data Guy — Data Modeling: Walking Through How To Data Model As A Data Engineer (Dimensional Modeling 101) (https://www.youtube.com/watch?v=gG7upg6QaBI)
**Dimensional modeling = business process -> declare grain -> choose dimensions -> identify facts that are true to the grain; the hard design calls are header/line-item, many-to-many bridges, and snapshots.**

- **Core table types**
  - *Fact*: numeric measurements of business events/transactions (sales, grades per assignment, inventory moved, invoice line amounts). Mostly foreign keys to dims plus a few additive quantities. This is what you SUM/COUNT/divide.
  - *Dimension*: the "by" words in a request ("sales by week by brand by region by rep") — the attributes used to filter, group and label. Change slowly relative to facts.
  - *Snapshot*: a periodic pre-aggregated copy of a fact (e.g., monthly sales). Reasons: save compute; save storage when old detail is deleted/archived; or freeze reported numbers so retroactive source corrections don't change past reports (a management preference, not a technical need).
  - *Bridge*: resolves many-to-many between dimensions (patient <-> diagnoses, account <-> customers). Generally discouraged.
- **Decision: header vs line item (invoices, orders)**
  - Header holds buyer, dates, PO number; line items hold product, price, quantity, per-line delivery/sign-off dates.
  - Two fact tables (header + line) means joining two large tables — expensive, slow, and in his early job it sometimes simply died on an on-prem system.
  - Kimball's recommendation: one fact table at line-item grain with header attributes denormalised onto each line. Cost: header attributes repeated (update in many places) and invoice number is no longer unique — users must `COUNT(DISTINCT invoice_no)`.
  - The key discipline: *state the grain explicitly* (here: invoice number + line number). Building a table without a declared grain is asking for wrong counts.
- **Decision: many-to-many**
  - Bridge table: risk of row explosion — one fact row fans out to N rows and gets double-counted; Kimball adds a weighting factor (allocations summing to 1) to fix sums.
  - Alternatives: fixed repeated columns (diag_code_1..5) — works only if the max count is small and known, ugly to query (which are null?), not scalable; arrays/nested columns — compact but consumers must know how to UNNEST.
  - Every option is a trade-off between query safety and usability; pick knowingly and document it.
- **Kimball process (retail sales case study)**
  1. Select the business process (point-of-sale transactions).
  2. Declare the grain — lowest useful level of detail; "how low is too low" depends on the questions (course grade vs individual assignment grade).
  3. Identify dimensions (date — possibly several roles like purchase/delivery date; store/channel; product with category/department/brand hierarchy; promotion).
  4. Identify facts — must be true to the grain.
  - It is iterative: fact discovery may force a change of grain or dims.
- **Rules of thumb**
  - Finding the fact is usually easy; dimensions are hard because they live in other systems (e.g., promotion info in ad platforms or link-shortener logs) and may not join cleanly to the fact.
  - Start from business questions (conceptual model) before physical tables.
  - Beware over-dimensionalising / snowflaking dims into many normalised sub-tables — more joins, harder for users.
- **Relevance to Sam**: for bookings, a booking (header) with multiple travellers/products/legs (lines) is exactly this case — prefer one fact at booking-line grain with booking attributes denormalised; places with multiple categories/tags or a POI served by many transit routes is a many-to-many — in DuckDB/Parquet an array column (`LIST`) plus explicit `UNNEST` is often cleaner than a bridge, but document the fan-out risk.

## Seattle Data Guy — Data Modeling Where Theory Meets Reality: How Different Companies I Worked At Modeled Their Data (https://www.youtube.com/watch?v=rqLbn1PQPKA)
**Three practical ways to keep history: SCD2 rows with effective dates, daily full-snapshot partitions (Facebook), and cumulative/date-list tables (Roblox-style) — chosen by query patterns, storage budget and scale.**

- **Facts already carry history — but how they arrive matters**
  - An event fact (date_id, event_ts, customer_id, product_id, measures) is append-only, so history is preserved by construction.
  - Load semantics vary by source and change the pipeline design: (a) pure append — corrections arrive as new reversing rows (a return, a disputed bill as -$50); (b) replacement — a record with the same event ID arrives and *replaces* the earlier one (delete old, insert new = upsert). The table looks the same, but the loader must be written differently. Find out which semantics your source uses.
- **Dimensions lose history unless you design for it**
  - `dim_customer(id, name, city)`: John moves NYC -> Seattle. Overwriting (SCD type 1) means past sales now report as Seattle — information is lost.
  - **SCD2**: add `start_date`/`end_date` (effective dates); on change, close the old row's end_date and insert a new row with end_date NULL. Current view: `WHERE end_date IS NULL`. Historical reporting: join fact to dim on `customer_id AND fact.date BETWEEN start_date AND end_date` so each sale picks the attribute valid at the time. He has mainly seen type 2 and partial type 6 in practice.
- **Facebook approach: daily snapshot partitions**
  - Every day, write the full dimension (e.g., all employees) into a new `ds` partition of the same Hive/HDFS table. Possible because storage was cheap and updates weren't.
  - Every table carried `ds`; a macro filled in the latest date; joins included `a.ds = b.ds`, and a dashboard date filter propagated the ds so all joined tables used the same day — avoids duplicates and gives "as-of" reporting for any day.
  - Weakness: change-oriented questions ("average days between role changes") require scanning hundreds of partitions to locate the change points. Retention was ~90 days by default, so long history disappears anyway. So they *still* built SCD2-style tables when needed: two rows for John beat 90 partitions.
- **Roblox-style cumulative / date-list table for huge event facts**
  - Page-visit events at billions/day are expensive to scan. If queries filter by user, build one wide row per user holding a compact history: a date list as a bitmap (0/1 per day active) or a dictionary/map {date: visit_count}, plus first/last seen dates.
  - Effect: scanning ~2B user rows instead of trillions of events; filter to a subset (e.g., US users) cheaply, then compute activity over any window from the array.
  - Costs: extra technical complexity for analysts (map/array functions), and loss of granularity (intraday detail gone).
- **Decisions & trade-offs**
  - SCD2: compact, good for "when did it change", harder current-state queries (need end_date filter) and BETWEEN joins.
  - Snapshots: simplest to write and to backfill, easy as-of queries, storage-heavy, bad for change analysis.
  - Cumulative/date-list: best at extreme scale for user-activity questions; more complex, lossy.
  - SCDs partly exist because storage used to be scarce; on-prem or budget-limited setups can't snapshot everything daily — fit the model to business needs *and* technical limits.
- **Gotchas**: forgetting the `end_date IS NULL` filter yields duplicated customers; joining snapshots without matching ds duplicates rows; unknown retention silently deletes history.
- **Relevance to Sam**: for location master data (names, coordinates, admin codes changing across GeoNames dumps) daily/monthly snapshot Parquet partitions are cheap on disk and trivially backfillable; derive an SCD2 "changes" table from consecutive snapshots when you need to know when a place changed. For supplier availability/price logs, a cumulative per-product row with a date->price map keeps queries fast on 16GB.

## Seattle Data Guy — Data Modeling: Why Data Engineers Need To Understand It (An Introduction) (https://www.youtube.com/watch?v=nof9jYIhv54)
**A data model is a derivative of the business; build it conceptual -> logical -> physical, normalise for write-heavy systems, and denormalise deliberately (you can only denormalise what you first normalised) for read-heavy analytics.**

- **Motivation**: two client horror stories — an OLTP table with `course_id_1 ... course_id_35` columns, and an analytics "customer table" where every transaction was crammed into a single column (one-big-table taken to absurdity). Both came from skipping modeling to ship MVPs fast.
- **Definitions**: modeling = organising and standardising data so it yields believable, useful information for humans and machines (Joe Reis's framing). The model captures entities, relationships and events of the business (who transacted, what product, where); the data just flows through that shell.
- **Three layers**
  - *Conceptual*: entities and relationships in business language (customer buys product; product sits on shelf in store; distributor supplies store). No keys/attributes — a business person must understand it; if they can't, simplify it.
  - *Logical*: the "how" — entities with attributes, primary/foreign keys, self-references, cardinalities — still independent of the database engine and data types.
  - *Physical*: data types, engine, partitioning, actual DDL. Where most engineers wrongly start.
  - Modeling starts in the business and operational systems, not at the warehouse.
- **Abstraction level decision**: separate `employee` and `customer` tables vs a generic `person` — and how far is too far (a universal `things` table is unworkable). Legit generalisation example: **custom fields** — instead of ALTERing the table for each customer-defined field, use `custom_field(id, name, type)` plus a junction/value table `(entity_id, custom_field_id, value)` (EAV pattern) in relational DBs; document DBs handle this differently.
- **Normalisation (mechanics)**
  - Purpose: reduce duplication, protect integrity, avoid update anomalies (one fact stored in one place); also saves storage.
  - 1NF: one value per cell (his pain: multiple project IDs in a single cell), a primary key, no repeating rows/column groups.
  - 2NF: in 1NF and no non-key attribute depends on only part of a composite key (employee_id + location_id table holding employee age -> move age to employee).
  - 3NF: in 2NF and no transitive dependencies (employee row with zip, city, state — city/state are determined by zip -> address/zip table).
- **Denormalisation**
  - Warehouses are write-once/read-many, so trading redundancy for fewer joins speeds reads. Normalised chains (employee -> zip -> city -> state) can be 3–4 joins deep for one attribute.
  - Costs: redundancy, potential anomalies, more storage, and corrections must touch many rows (customer moves -> update 30 transaction rows in OBT vs one row normalised).
  - Star schema and OBT are points on the denormalisation spectrum.
- **Rules of thumb**: no multi-valued cells or numbered repeating columns in operational schemas; normalise first, then denormalise for specific read patterns; keep the business in the loop at the conceptual stage.
- **Relevance to Sam**: in a location master table, country/admin names depend on codes (transitive) — keep normalised reference tables (countries, admin1, admin2) as the source of truth and generate a denormalised wide Parquet for reads/matching. For supplier attributes that vary per provider, an EAV-style or JSON/MAP column beats endlessly adding columns.

## Seattle Data Guy — Databases vs Data Warehouses vs Data Lakes: What Is The Difference And Why Should You Care? (https://www.youtube.com/watch?v=FxpRL0m9BcA)
**OLTP databases are built for row-level CRUD on current state; warehouses for integrated, historical, structured analytics; lakes for cheap, flexible, schema-on-read file storage — no single system does all well.**

- **Databases (OLTP)**
  - Classic RDBMS (MySQL, Postgres, SQL Server) are row stores: a whole record is stored together, so inserting/updating/deleting one row is cheap. NoSQL (MongoDB, Cassandra) is also transaction/app oriented. Facebook's core backend still runs on MySQL.
  - Problems with doing analytics on them: heavy queries (or accidental writes/temp tables) can slow production; they usually keep only current state — overwrites and hard deletes erase history (logs exist but aren't a history model).
- **Data warehouses**
  - Inmon's definition: subject-oriented, non-volatile, integrated, time-variant collection in support of management decisions.
  - *Integrated*: many sources (apps, Salesforce, HR, finance) land in one place and are joined so cross-domain questions are answerable. Most companies spend lots of time on this (unlike Facebook, integrated at app level).
  - *Time-variant*: keep history via SCDs (or, in lake-ish setups, daily snapshots and diffs). Example: "customers per state per year over 5 years" is impossible if only current addresses exist.
  - *Structured/rigid*: star/snowflake (also Data Vault); schema changes need ALTERs and effort.
  - Columnar storage: aggregates read only the needed columns; metadata/indexes/cluster keys locate data. He argues columnar isn't a *definitional* requirement — he has seen warehouses on Postgres/SQL Server perform fine, and dedicated columnar warehouses (Redshift, Snowflake, Teradata, Vertica) are costly. Use the simplest thing that's fast enough.
  - Layering: raw (as extracted, light checks) -> staging (fix types, integrate, de-duplicate, apply SCD logic, more checks) -> production/"core" layer everyone builds on -> analytical layers (pre-joined wide tables for analysts).
  - Purpose: reporting/dashboards for decision-makers; make data easy for non-engineers (star schema is simpler than 3NF; wide analytic tables simpler still).
- **Data lakes**
  - Born in the ~2012 big-data/Hadoop era: constantly changing app schemas (e.g., Mongo) made rigid warehouses painful, so store files with schema-on-read and process on clusters of cheap hardware.
  - Layout: folder per table, sub-folders per partition (e.g., `orders/date=.../`), files in Parquet/JSON/CSV (CSV causes many problems) or even unstructured docs.
  - Uses: ML/data-science exploration; trying data out before productionising it into the warehouse; fast operational reporting that skips batch ETL. Users need to be technical (code to access).
  - Common flow: databases -> lake -> warehouse.
- **Decision rule**: match system to workload — transactions to OLTP, analytics to OLAP. Systems trying to do both (Snowflake/Databricks app ambitions) are hard to make excellent at both.
- **Relevance to Sam**: a laptop "lakehouse" — date-partitioned Parquet folders (`raw/`, `staging/`, `core/`) queried with DuckDB/Polars — gives warehouse-style layering and columnar reads without a paid warehouse; never run heavy analytical scans against a supplier or production Postgres used by the booking flow; prefer Parquet over CSV for anything beyond hand-off files.

## Seattle Data Guy — Extracting Data From APIs As Data Engineers: The Basics And Challenges (https://www.youtube.com/watch?v=YST1sWFPDh4)
**Every pipeline starts with extract, and API extraction is hard to generalise because auth, pagination, response shapes, limits and stability differ per vendor.**

- **Mental model**: an API is the waiter between client and the "back of house" (server/DB). The docs are the menu; the endpoint + params are your order (e.g., customers between start and end date, or one customer by ID). You don't see the kitchen — you only judge whether what came back is correct. Whole companies exist just for connectors, and even those with ~1,500 connectors miss some, so you must be able to write one.
- **Mechanisms**
  - Endpoint = URL bound to a resource (`/customers`). REST is the common style; SOAP and others exist.
  - Methods: GET to read; POST to send data — but some report-style APIs require a POST (submit a "form" describing the report) to get data back.
  - Status codes: 2xx ok; 4xx = your fault (bad syntax, wrong path, auth); 5xx = server's fault (retry later).
  - Parsing: usually JSON (`response.json()`), but can be CSV files, ZIPs, etc. For many endpoints, move hard-coded endpoint/field lists into a config file and drive one generic extractor from it.
- **Authentication variants**
  - API keys (simplest, treated as weaker); Basic auth (username:password encoded — a leak exposes everything that user can see); IP whitelisting (only listed IPs may call — he lost time because a client's vendor silently required whitelisting nobody knew about); OAuth (authorize -> access token carried on calls; painful outside an interactive app, tokens expire/refresh); also client certificates, SAML, JWTs.
- **Pagination variants**
  - Offset/limit (`offset=0&limit=100`, then 100, 200...): assumes stable ordering and no inserts during the crawl; new rows shift pages so you skip/duplicate records — crawl oldest-first. His least favourite.
  - Page number (`page=1,2,3` with fixed/selected page size).
  - Cursor-based: server-issued cursor marks position; robust to inserts.
  - Hypermedia ("next" URL in the response): his favourite — loop while `next` is non-empty; no position bookkeeping. Often combined with cursor/page schemes.
- **Challenges / failure modes**
  - Silent caps: API returns at most N (e.g., 10,000) rows with no error or documented pagination — you get truncated data and must contact the vendor. Always compare counts against an expected total.
  - Nested/fan-out calls: APIs designed for app integration, not bulk export. Example: projects -> sections per project -> tasks per section; call count explodes multiplicatively.
  - Schema/endpoint churn: one client's vendor API changed roughly every 6 months — fields removed/added, whole endpoints deleted. Plan for drift (schema checks, alerts), or pay for managed connectors.
- **Rules of thumb**
  - Don't assume any two APIs behave alike; read docs for auth, pagination, limits, rate limits.
  - Generalise via config, but expect per-vendor code.
  - Validate completeness (row counts, date coverage) after every pull.
- **Relevance to Sam**: directly applies to Viator/RateHawk/Coras integrations. Implement pagination with a "next/cursor first, offset last" preference; persist the last cursor/page so a crashed crawl resumes; land raw JSON per page to disk (partitioned by run date) before parsing; for fan-out APIs (destinations -> products -> availability), run bounded concurrency with retries on 5xx/429 and respect rate limits; snapshot response schemas so supplier changes are detected rather than silently dropping fields.

## Seattle Data Guy — How And Why Data Engineers Need To Care About Data Quality Now, And How To Implement It (https://www.youtube.com/watch?v=wvUiRHd47M0)
**A small set of check types (range, category, type, freshness, volume, null, uniqueness) catches most problems; the implementation should scale with need — from Slack alerts to a config-driven check engine to DQ operators wired into the orchestrator/catalog.**

- **Why**: bad data quality is claimed to cost companies a large share of revenue (~20% figure quoted); AI/automated decisions raise the stakes. DQ gets skipped because it's less exciting than building pipelines.
- **Check types (with his war stories)**
  - *Range*: numeric values should sit within expected bounds (static limits or anomaly models). A sudden $200k transaction where ~$1k was normal turned out to be a *process* failure (someone allowed to overspend) — DQ checks surface business-process bugs, not just bad data.
  - *Category / accepted values*: closed sets like US state codes — despite a dropdown upstream, 4–5 non-existent state abbreviations arrived. PTO types: ops teams kept creating new categories without telling data, so a check alerts on any unmapped value.
  - *Data type*: especially for headless/positional CSVs — a partner sends the same layout for 6 months, then a staff change or script edit reorders columns and a date column now holds something else. Catch it at the raw layer, before staging.
  - *Freshness / timeliness*: alert when latest load exceeds an age threshold; show "updated as of" on dashboards so execs don't assume the system missed a transaction from 30 seconds ago.
  - *Volume*: daily row counts should be roughly stable (weekend dips aside); 3–10x spikes or sharp drops signal upstream bugs — e.g., a feature release that fired 10x the events. Also catches "table updated but only 10 rows instead of 100k".
  - *Null*: either zero nulls allowed or a tolerated percentage; know how nulls behave in joins/aggregates; fill defaults only where appropriate.
  - *Uniqueness*: primary key unique (mentioned with dbt tests).
- **Implementation tiers (choose by budget and decision criticality)**
  1. *Slack alerts*: a handful of hand-written SQL checks UNION'ed together, run at the end of key jobs; post failures to the on-call engineer. Not generalised, but enough for ~10 checks.
  2. *DQ dashboard*: freshness/volume/null status of key tables (red when >24h stale). Needs someone to look — pair with alerts (push) + dashboard (pull).
  3. *Generalised in-house engine*: a table/folder of check definitions (SQL, check type, allowed failure threshold, severity); a wrapper script (Python/PowerShell) loops through them, runs each, and stores results. Adding a check = adding a row, not code. Optionally persist results over time to track table health trends (what paid vendors sell).
  4. *DQ operators in the orchestrator* (Facebook): pre-built check tasks in the DAG — you supply a query and expectation and whether failure should block; results flow automatically into the data catalog with table health.
  5. *dbt tests*: built-ins `unique`, `not_null`, `accepted_values`, `relationships`, plus custom generic tests — only covers models built in dbt.
  6. Buy an observability tool if you lack engineers to build.
- **Rules of thumb**
  - Check at raw before loading further; decide per check whether to fail the pipeline or just warn.
  - Thresholds should be expectations agreed with the business, not guesses.
  - Make adding a check cheap (config-driven) or nobody will add them.
- **Relevance to Sam**: for multi-million-row GeoNames/OSM/GTFS runs, a config-driven check list executed in DuckDB is ideal: lat in [-90,90], lon in [-180,180] (range); country codes in ISO list, feature codes in allowed set (category); row counts per country vs previous run within ±X% (volume); file date of each GTFS feed (freshness); null rate on name/coords; uniqueness of (source, source_id). Write results to a Parquet "dq_results" log per run to see drift over time.

## Seattle Data Guy — SQL Best Practices: Designing An ETL, Part 1 (https://www.youtube.com/watch?v=sLhInuwdwcc)
**Never go straight from the operational DB to the warehouse: extract to named flat files -> load raw with zero logic -> QA -> staging (map + dedupe) -> QA (reconcile) -> warehouse core -> numbered aggregate layers, with logging at every step.**

- **Why staged, not direct**
  - Direct live queries load the operational (CRUD) DB with analytics; operational DBs keep only current state; warehouses need history, dedup and business logic.
- **Step 1 — extract to flat files** (CSV/TSV/fixed-width positional/XML/JSON)
  - Keep them: if the source changes or a later step breaks, you can reprocess from that point-in-time snapshot without hitting the source again.
  - Name files with metadata: extraction timestamp + entity/table (+ source). Give each file a file ID. Known names make automation and "which files are loaded?" bookkeeping possible.
- **Step 2 — load into a raw DB with no business logic**
  - Temptation: dedupe/map during load. Don't — when QA fails you can't tell if the bug is in your logic or in the data.
  - Raw-load failures are often trivial and nasty: `\n` vs `\r\n` line endings; commas inside unquoted fields shifting every column right by one; wrong field mapping. Keep ingestion dumb so these are easy to spot.
  - QA raw: zip codes look like zips, names like names, amounts numeric.
- **Step 3 — staging (`stg_` prefix)**
  - *Mapping*: providers encode the same thing differently (provider 1 code `11`, provider 2 `A7` -> both mean `US`); map to canonical values.
  - *Dedup*. Optionally split into two staging steps (dedupe table, then mapping table).
  - Staging is a temporary holding area, not long-term history; appends/updates (upsert logic) usually belong in the warehouse fact load.
- **Step 4 — QA after transformation**: reconciliation — distinct counts of unique keys (customers, claims, transaction IDs) and aggregates must match between raw and staging after the logic; any drop must be explained and tracked.
- **Logging (he stresses heavily)**
  - File log: which file IDs loaded, when.
  - ETL step log: each script/stored procedure — start/end, duration, success/failure, error reason, rows loaded, rows rejected.
  - Uses: trends in duration show which step to optimise (a proc that keeps getting slower), recurring errors show which part to harden. "We do analytics on our own systems."
- **Step 5 — warehouse**
  - First insert is clean, QA'd data with minimal transformation (core tables, e.g., `f_transactions`).
  - Further derived tables get level markers (T1, T2...) showing how many transformations away from source-of-truth they are; aggregates `agg_`, snapshots, etc.
  - Naming conventions: `f_` fact, `d_` dimension, `agg_` aggregate, snapshot prefixes, `usp_` stored procs, `vw_` views. Consistency helps everyone.
  - Fact = an instance of something happening with measures: store_id, customer_id, item_id, quantity, price, date_created, date_id.
- **Rules of thumb**: separate extract/load/transform; keep raw immutable; put QA after every step that adds logic ("where something can go wrong, it will"); clean layering also makes it easy to spin off other systems from the warehouse.
- **Relevance to Sam**: mirrors a laptop pipeline perfectly — `raw/<source>/<entity>/dt=YYYY-MM-DD/<file_id>.json|parquet` kept immutable; `stg` step maps supplier codes (country, category, currency) to canonical; reconciliation counts between raw and staged after each matching/dedupe pass on millions of location rows; a simple run-log table (step, start, end, rows_in, rows_out, rejected, status) shows which step is slowing as data grows.

## Seattle Data Guy — Slowly Changing Dimensions For Data Engineers (https://www.youtube.com/watch?v=1FZ7et0pN4c)
**Of the SCD types, 1, 2 and 6 are the ones seen in practice; type 2 (new row + validity dates) is the default because type 1 silently misattributes history.**

- **Type 1 (overwrite)**: store 1's district manager John Smith is overwritten by Jordan Doe. You lose *when* it changed and *who* it was before; "total sales under each manager" now credits Jordan with John's sales. Only supports current-state snapshot reporting.
- **Type 2 (add row)**: insert a new row for the new value and close the old one. Columns: `insert_date` (effective from), `update_date` (effective to), optionally `is_active` flag. Open rows need a sentinel: either NULL or a far-future max date (e.g., 9999-12-31) — pick one and be consistent. The load is an insert + update in one operation (insert new version, set end date on prior version). Now each month's sales attribute to the manager active then.
- **Type 6 (2 + previous-value column)**: also carry `previous_manager` on the row. Convenient for reporting, but he dislikes it for core dims: do you add a "previous_" column for every attribute that can change (store name, etc.)? He prefers growing tall (rows) over wide (columns); "previous X" fits better in a reporting table.
- (Types 3, 4 exist; he skips them as rarely seen.)
- **Rules of thumb**
  - Use a stable ID (not names) as the natural key in the dim.
  - Ask "what reports must this support?" — if any report needs as-of attribution, type 1 is insufficient.
  - SCDs are for dimensions; facts have their own question (append vs replace).
- **Relevance to Sam**: supplier product/hotel attributes (rating, name, status, coordinates) and manager-like mappings (place -> canonical master_id) change over time — keep SCD2 with a consistent open-end sentinel (`9999-12-31` makes BETWEEN joins simpler than NULL in DuckDB/SQL).

## Seattle Data Guy — The Realities Of Airflow: The Mistakes New Data Engineers Make Using Apache Airflow (https://www.youtube.com/watch?v=gkKY6Q3GApw)
**Airflow is deceptively easy to start and hard to run at scale; deploy DAGs separately from the platform, use built-in hooks/variables, and plan for resource contention.**

- **Learn `airflow.cfg`**: sets metadata DB, DAG folder, log location, executor, parallelism, etc. Tutorials (docker-compose, default executor) hide decisions that block scaling.
- **Mistake 1 — DAG folder bundled with the Airflow image/repo**
  - With CI/CD, every DAG edit rebuilds and redeploys the whole container, killing running tasks — a 3-hour job dies and restarts from scratch.
  - Fix: separate DAG repo -> CI pushes DAG files to object storage (S3/GCS) -> synced into a network/mounted folder that Airflow instance(s) read (Scribd and Shopify describe this pattern). Platform and DAG deploys become independent.
  - Workaround if small: deploy only in a window when no DAGs run — doesn't scale.
- **Mistake 2 — not using built-in features**
  - *Hooks/connections*: named connection objects (Postgres, BigQuery, Slack, custom NetSuite, HTTP). Credentials not repeated in code (security), connection proven to work once and reused (faster dev).
  - *Variables*: key-value config, e.g., prod vs dev target locations; secrets stored masked/encrypted by naming with `_password`/`_secret` (PGP passphrases, etc.). Avoid custom secret-handling code.
- **Mistake 3 — not planning for scale**
  - Past ~10–100 jobs, tasks contend: limited slots per worker, number of workers/schedulers, queue setup, executor choice. Shopify: many contention points; easy to chase bottlenecks through trial-and-error config changes; some need infrastructure changes (DevOps).
  - Knobs: pools, priority weights, parallelism/concurrency settings — mitigations, not a full solution.
  - Managed options (MWAA, Cloud Composer) scale decently for many basic tasks, with their own problems.
  - At 1k–100k tasks (Facebook's Dataswarm: ~10k jobs on his team alone) you need a dedicated platform team of several people — self-hosting isn't free; you pay in headcount.
- **Relevance to Sam**: for a solo engineer on a laptop, full Airflow is likely overkill; a lightweight orchestrator (or simple Python DAG runner with output-file checkpoints) avoids the ops burden. If Trameter later runs Airflow, keep DAG code in its own repo and never hard-code supplier API keys — use connections/secrets.

## Seattle Data Guy — Understanding SQL Query Execution: What Happens When You Run A SQL Command (https://www.youtube.com/watch?v=uvACOp4WFR4)
**A SQL query passes through parse -> (syntax + semantic) validation -> relational-algebra/logical plan -> cost-based optimizer -> physical execution plan -> execution, with security checks and plan caching along the way.**

- **Pipeline inside a SQL engine** (names vary across MySQL/Postgres/SQL Server/Snowflake/Presto; he uses "SQL engine" generically)
  1. *Lexical analysis*: split text into tokens — keywords (SELECT, FROM, WHERE), identifiers (tables, columns), operators (=), literals. Using a reserved word as a name fails here.
  2. *Syntax check*: tokens must follow SQL grammar; produces a parse tree. MySQL's parser = lexical scanner + grammar-rule module.
  3. *Semantic check*: do the referenced tables/columns exist?
  4. *Translation ("algebrizer"/query transformer)*: convert to relational algebra (selection, projection, joins); may generate several equivalent logical plans.
  5. *Optimizer*: chooses the cheapest physical plan using table statistics, indexes, filter placement (push filters early) and physical join algorithms.
  6. *Execution plan* -> executed -> results.
- **Logical vs physical joins**: INNER/LEFT describe *what*; the engine decides *how* — hash join, merge join (benefits from sorted inputs/keys), nested loop (compares row combinations; quadratic-ish worst case). This is where Big-O intuition matters. You often can't fully force the choice.
- **Tools**: `EXPLAIN` / estimated vs actual execution plans to see what the optimizer chose.
- **Also in the path**
  - Security/authorization on every object (tables, columns, procedures, functions). Presto had none built in, so Facebook added a separate access-check layer before queries hit the engine.
  - Query/plan cache: a repeated query may reuse an old plan — if you add an index and nothing changes, a cached plan may be why.
- **Rules of thumb**: read the plan before "optimising" blindly; keep statistics fresh; make join keys sortable/typed consistently so cheaper join algorithms are available.
- **Relevance to Sam**: DuckDB and Polars (lazy mode) follow the same pipeline — `EXPLAIN`/`EXPLAIN ANALYZE` in DuckDB and `.explain()` in Polars show whether filters and column projections were pushed down into the Parquet scan, and which join algorithm is used. Hash joins on huge tables are the main memory risk on a 16GB machine; filtering/projecting before joining shrinks the hash table.

## Seattle Data Guy — What Are ETLs And Why We Use Them (For Data Scientists And Data Engineers) (https://www.youtube.com/watch?v=Jw4Or_FiT7k)
**ETL = extract from apps/APIs/scrapes/files, transform (remodel, dedupe, SCD), load to a trusted production layer; pick tooling on the no-code -> framework -> custom spectrum, and gate production with QA.**

- **E / T / L**
  - Extract: application DB tables (MySQL), SaaS APIs (Salesforce, Workday), scraping, downloaded government XML/Excel/CSV drops.
  - Transform: remodeling, de-duplication, denormalisation/renormalisation, SCDs for history.
  - Load: into the production store (lake, warehouse) analysts/data scientists use.
- **Tool spectrum and trade-offs**
  - *No-code/low-code* (SSIS, Informatica drag-and-drop; Fivetran/Stitch GUI parameter-driven, Fivetran can schedule SQL/dbt transforms): fast for supported sources/destinations; hard wall when a source/API/transform isn't supported.
  - *Code-based frameworks* (Airflow, Luigi): parameterised operators; the framework handles the infrastructure-heavy parts — error handling, logging, dependency tracking, scheduling. Most data engineers land here.
  - *Pure custom code* (Python/PowerShell/Scala/Java + cron + home-grown metadata DB for dependencies): maximum freedom, lots of undifferentiated infrastructure work for little payoff.
- **Three-layer process**
  - *Raw*: extract to raw files, load raw tables with at most trivial cleanup (e.g., date formats). Sanity QA only (data arrived, looks plausible). If raw has an error and you did no transforms, the problem is upstream in the application — clean separation speeds debugging.
  - *Staging*: business logic, standardisation, dedup, denormalisation, SCD. Keep most logic here (not split across stage and prod) so failures are localised.
  - *Production*: mostly a QA/certification gate — unit-like checks and end-to-end raw->prod integration checks. Production must be "pristine": even a day of bad data may drive decisions you can't trace. Failed check -> stop the ETL, or allow through if within tolerance (e.g., 3 nulls) — decide per column.
- **Staging mechanics — source semantics for repeated IDs** (ask every provider which they use; it's one of his first client questions)
  - *Merge/net*: corrections arrive as additional rows with the same ID (e.g., medical claims: 150 then -50 -> net 100 after insurer adjudication) — sum them.
  - *Latest-wins dedup*: a newer row with the same ID supersedes the old (keep the 100 row, drop the 150) — `ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at DESC) = 1`.
  - Some systems log every click/update as a new row; without dedup analysts overcount.
- **SCD in staging**: store manager with `start_effective_date`/`end_effective_date` (NULL = current). Add an explicit `is_active` flag for analysts — NULL-as-current is obvious to engineers, confusing to analysts. Whatever convention, use it consistently across all tables.
- **Supporting infrastructure checklist**: QA framework (start with simple tests, evolve to abstracted reusable checks with history for comparison); error handling + logging (ETLs fail constantly — sometimes just a flaky distributed node that needs a restart; logs must pinpoint e.g. "field X expected number, got string at row 32"); monitoring dashboards (not grepping logs in vim); lineage + catalog (field-level "depends on" maps instead of reading code); metadata DB for pipeline dependencies.
- **QA check types**
  - Categorical (valid state/territory codes; free-text fields create typos).
  - Null thresholds per field: primary keys 0%; some fields 5%, 20%, even 95% null can be legitimate.
  - Aggregate reconciliation raw vs prod: distinct customers 5 in raw but 4 in prod -> something (often a join/filter) dropped data.
  - Anomaly checks against expected ranges stored in a table: row count expected 980–1,020 got 1,000 (ok); null % outside its band (flag); avg price expected 60–120 (ok).
  - Wrong LEFT JOIN or mis-ordered dependencies produce silent null columns — a classic thing prod QA catches.
- **Relevance to Sam**: supplier feeds differ in correction semantics (booking amendments/cancellations as new rows vs overwrites) — determine it per supplier before modeling. Store expected-range bands per table in a small config table and compare each run (row counts per country, null % for coordinates, price ranges per product type) — cheap anomaly detection that runs fine in DuckDB on a laptop.

## Seattle Data Guy — What Is A Data Catalog And Why Do People Use Them? (https://www.youtube.com/watch?v=syN4rGwE0rg)
**A catalog turns metadata (what exists, where it came from, who owns it, is it trusted/fresh) into a searchable product; you need one when table count, users and breakages outgrow tribal knowledge — usually earlier than teams adopt one.**

- **Definition**: uses metadata about data assets plus management and search features so people gain context on data without reading code or querying it.
- **Why teams get one (guest from Braintree's central data team)**: tables grew from ~100 to thousands as SQL-savvy users multiplied; no visibility into how data was generated, replicated, transformed, used in reporting/compliance; upstream breaking changes caused unpredictable downstream ripple effects; productivity ground to a halt. dbt docs/lineage graph were too overwhelming for consumers; *column-level lineage* ("where did this attribute come from?") was the most-asked question.
- **Expected features**: search & discovery; business glossary; lineage (table and column level — not all catalogs include it); ownership and usage (who owns it, who queries it, how they join it); collaboration/sharing; governance (certified vs test/unverified assets, access); integrations that push metadata into the tools people already use ("active metadata": status, owner, freshness visible inside the BI tool or Slack; automated workflows like deprecating unused tables).
- **Benefits**: less duplicate work (no rebuilding a table that already exists), faster owner lookup, fewer errors from using uncertified/test tables, clear quality and freshness signals.
- **Build/buy**: open source (e.g., DataHub; one team spent a month with a DevOps engineer failing to deploy an earlier OSS option before switching) needs platform capacity; SaaS needs a contract. Question: what are your end goals and who will support the tool?
- **Facebook's iData**: he rarely had to ask people about datasets — could see how others used and joined a table, when its pipeline last changed, and ping the owner when data looked stale.
- **When to adopt**: growing team, more data users/dashboards/tables, constant "what is this table?" questions. A static Notion page works at first; an auto-scanning catalog of the warehouse/lake is the eventual need.
- **Relevance to Sam**: at solo/small-team scale, a lightweight catalog is enough — a `catalog.yml`/table per dataset recording owner, source, grain, refresh cadence, row count, last-run date and upstream inputs (can be auto-filled from the pipeline run log). It answers the same "where did this column come from?" question for Trameter colleagues consuming the location master.

## Seattle Data Guy — What Is A Data Pipeline? Data Engineering 101 (ft. Alexey from DataTalksClub) (https://www.youtube.com/watch?v=5P2luRwaKek)
**A pipeline is any chain of steps moving data from A to B with something done in between — but the real job is the lifecycle: requirements/SLAs, choice of type, quality, failure handling and maintenance.**

- **Definitions**: data goes in, something happens, data comes out; steps can be chained (one-in/many-out, many-in/one-out). Unix pipes, shell scripts and Python scripts are pipelines too; in production people mean orchestrated ones (Luigi, Airflow, Prefect, ADF, Matillion, dbt). Joke definition: whatever outsiders message you about when it breaks.
- **Types of pipelines**
  - Integration (CRM <-> ERP, operational; iPaaS).
  - Batch ETL/ELT (analytics).
  - Streaming: CDC or direct event streams.
  - Transformation-only (the "T" became its own complex pipelines, e.g., dbt).
  - Reverse ETL: warehouse -> operational tools (essentially iPaaS sourced from the warehouse).
- **Questions to ask before building** (from Colin and Alexey)
  - Which tables/sources — now and likely next? (Avoid rebuilding per table: use a *control table* listing sources/tables/configs so one generic pipeline handles all.)
  - Straight replication or CDC?
  - Frequency and SLA for availability.
  - How many steps; what will consumers do with it? If few steps, a simple script on a scheduled cloud trigger (e.g., CloudWatch/EventBridge rule) may suffice.
  - Who consumes it, how often (monthly report vs live decisions)? This drives tooling, style and required quality — operational systems where errors can harm people need far higher rigor than a weekly dashboard. Integration cost and maintenance scale with that rigor.
- **Failure modes**
  - Upstream producer changes a value or type -> pipelines break (motivation for data contracts, Chad Sanderson). Sometimes you don't want a break but a conversation, so downstream logic is re-validated.
  - Hard failures are the easy ones (error, log, respond). Scarier: *silent* failures (wrong data, no error) and *intermittent* ones (fails once a week out of every-2-hour runs, cause unknown).
  - Value drift: e.g., typical $30 charges then a $500,000 charge — should be flagged.
- **Rules of thumb**: cron scripts worked when needs were basic; as data becomes business-critical, quality and reliability checks are part of the pipeline, not extras. Plan who supports it long-term and how often it will need changing.
- **Relevance to Sam**: a control/config table of sources (supplier, endpoint, cadence, last cursor, target path) lets one extractor serve Viator/RateHawk/etc.; define SLAs per consumer (booking flow = strict, research dashboards = loose) and size checks accordingly; log run outcomes to spot intermittent failures.

## Seattle Data Guy — What Is A Data Platform And Why You Should Build One (https://www.youtube.com/watch?v=_BoM2ahSJV0)
**A data platform is technology + process + people whose purpose is to make data easy and safe to use — storage/compute, orchestration, a stable core data model, governance/catalog, and a serving layer for BI/ML — sized to your stage.**

- **Definitions**: vendor definitions differ (Snowflake: real-time insight, cost-efficient, scalable, secure; IBM: collection, storage, cleaning, transformation, analysis, governance, including hardware). His view (drawing on Dylan Anderson): goal is making data *useful* — empower decisions, break silos/integrate, consistent definitions (governance), compliance, and make each persona productive (DEs ship pipelines, ML engineers deploy models, analysts query without confusion).
- **Why build one**: Facebook example — shipping a pipeline was just writing the script and pushing it; dependency management, compute and infra were handled by a platform team. Friction kills use cases; make the "service layer" easy.
- **Components**
  1. *Storage + compute* (the heart): Snowflake (2012, popularised separated storage/compute in the cloud — no more fighting over memory/disk on one SQL Server box, but "infinite compute, infinite bills"); earlier Redshift still coupled storage/compute; Databricks (Spark plus workflows etc.); BigQuery; open-source Trino/Presto over S3 (pay only cloud infra); purpose-built unstructured/AI stores (sponsor segment — skip specifics).
  2. *Orchestration/pipelining*: piecemeal tools per E/T/L step or a general orchestrator (Airflow; Riot Games even runs ML workflows on it). Must make pushing workflows easy for DEs, analytics engineers, ML engineers.
  3. *Core data layer*: a granular, business-faithful model (enterprise data model) that others build on. Treat it like infrastructure with a software lifecycle — change deliberately, not per use case, because many depend on it. Keep asking "does this represent our business?"
  4. *Governance & management*: catalog (find assets, lineage back to sources, dashboards using it, owner, quality scores), shared terminology, access policies (salaries, customer PII) so data is easy to use but not easy to leak or get fined for.
  5. *Serving layer*: make dashboards and ML deployment easy with clear standards (e.g., "we deploy models in Python; here's the R path if needed").
- **Reference architecture (Netflix)**: sources -> Kafka & internal data-movement tooling -> S3 with Hive/Iceberg table formats -> metadata/catalog -> multiple engines (Trino, Spark, Druid, Elasticsearch) -> notebooks/Tableau.
- **Trade-offs / failure modes**: you don't need the giant vendor diagram — a SQL Server that works is fine; over-tooling means you spend more time managing tools than shipping. Platforms may need extension for unstructured data/LLM workloads.
- **Relevance to Sam**: a laptop-scale "platform" = Parquet (optionally Iceberg/Delta later) on disk/object storage + DuckDB/Polars compute + a simple orchestrator + a stable core location/supplier model + a catalog file + clear serving outputs (API/Parquet extracts for the Drive Button and booking flow). The principle "treat the core model as infrastructure" fits Trameter's master location dataset: version it, change it deliberately.

## Seattle Data Guy — What Is Change Data Capture? Understanding Data Engineering 101 (https://www.youtube.com/watch?v=hNJCxF3IWC4)
**CDC streams row-level inserts/updates/deletes out of a database (usually by reading its write-ahead log) to get near-real-time data and preserve intra-day history that daily batch snapshots lose.**

- **Mechanism**
  - *Write-ahead log (WAL)*: before a DB modifies data pages/indexes on disk, it appends the intended change to a sequential log. If power fails mid-write, the DB replays/rolls back from the log — guaranteeing a consistent, recoverable state (see *Designing Data-Intensive Applications*). Because every change lands in the log in order, a reader can tail it as a live change stream (Postgres logical replication, MySQL binlog; Debezium does this into Kafka).
  - *Triggers*: per-table procedures firing on INSERT/UPDATE/DELETE that write changes elsewhere; Postgres can push notifications to a listening consumer (near producer/consumer). Simpler, but adds load to every write.
  - "Near real-time": latency is never zero; don't over-promise "real-time".
- **Why use it**
  - Near-real-time analytics/operations (e.g., factory operations monitored live).
  - *History preservation*: a once-a-day batch pull sees only the end-of-day state; anything that changed several times during the day is lost. CDC records every change — matters for fast-changing dimensions.
  - Supplement batch pipelines and onboard sources quickly: point-and-click CDC tools can dump changes into the lake now and model later, so nothing is lost while the team lacks time to build a proper pipeline.
- **Build vs buy (his projects)**
  - Custom Kafka + Debezium for an open-source-minded CI/CD-platform client streaming live run metadata — works but many components to operate.
  - StreamSets for a manufacturer needing live multi-factory data — got expensive at their capacity.
  - A cheaper managed tool (Estuary) for a telecom with TB-scale call records — simple UI; sources included Postgres/MySQL CDC but also S3 and SFTP drops.
  - Clients often hire him precisely because they lack a team to run the components — favour managed tools then.
- **Rules of thumb**: choose CDC when you need low latency or full change history; otherwise batch is simpler. Ensure the DB is configured for log-based replication (e.g., Postgres `wal_level=logical`) — (note: config detail added, not from video).
- **Relevance to Sam**: if Trameter's booking/inventory lives in Postgres, CDC into Parquet partitions captures every status/price change (booking amended, cancelled) that a nightly dump would flatten. For third-party supplier APIs (no DB access), emulate CDC with frequent incremental pulls using `updated_since` and keep every version (append-only), then derive current state.

## Seattle Data Guy — What Is dbt and Why Is It So Popular? Intro To Data Infrastructure Part 3 (https://www.youtube.com/watch?v=8FZZivIfJVo)
**dbt wraps SQL transformations in software-engineering practices (version control, tests, modularity, environments), so one person maintains the framework and everyone else only needs SQL.**

- **History/why**: a decade ago transforms lived in SSIS (plus C# scripts, injected SQL), scheduled by SQL Server Agent; or stored procs/SQL files run by PowerShell/bash via cron/Task Scheduler; SAP/Oracle tools similar. Even at Facebook, pipelines were parameterised Python whose core was SQL creating a table. The SQL was constant; the surrounding framework varied and each needed a specialist — a high barrier.
- **Unbundling**: Airflow used to do everything (connect to sources, run SQL via BigQuery operator, DQ checks). Connector tools (Fivetran, Airbyte, Meltano) took the E/L; dbt took the T.
- **Mechanics**: `pip install dbt-<adapter>`; `dbt init <project>`; SQL models in `models/` (one file = one table/view in the target); Jinja templating + macros (reusable SQL snippets, like UDFs); compile renders SQL into `target/`, then runs it against the warehouse; tests declared in config (schema tests, referential integrity, unique/not_null) run against built tables; profiles for dev/prod environments.
- **Pros**: open source; version control from day one; only SQL required; testing built in; docs and a big community (~30k Slack).
- **Cons**: SQL isn't ideal for all logic (complex logic hidden in joins, hard to read); weak debugging of custom macros; it's only the T, so you still need ingestion and orchestration (ELT is common in SMB: buy Fivetran/Airbyte + dbt; enterprise more ETL). Still needs good modeling skills and process — he's seen dbt projects go wrong without them.
- (note: VC valuation commentary — skip; "pip install dbt" is now per-adapter packages like `dbt-duckdb`, `dbt-postgres`.)
- **Relevance to Sam**: `dbt-duckdb` gives the full dbt workflow (models, tests, lineage docs) on a laptop over Parquet files — a cheap way to add version-controlled, tested transforms to the location pipelines, and shareable with colleagues who only know SQL. Keep heavy Python matching logic outside dbt; use dbt for the SQL layers and tests.

## Seattle Data Guy — What Is The Modern Data Stack? Intro To Data Infrastructure Part 1 (https://www.youtube.com/watch?v=-ClWgwC0Sbw)
**Grow the stack with analytics maturity: start with ingestion -> storage -> visualization, and add lineage, observability, catalogs, reverse ETL only when team size and complexity justify their maintenance cost.**

- **Context**: huge VC funding (dbt, Fivetran, Airbyte, Starburst...) produced a crowded landscape (Matt Turck's chart), making tool choice hard. (note: funding figures are 2021-era context, not engineering guidance.)
- **Maturity model (after Taylor Brownlow / Count)**
  - ~4 people: a dev pulls data manually, someone analyses in Excel. A warehouse isn't worth the ROI yet — it needs at least one person to build and run it.
  - ~10–12 people, hundreds/thousands of transactions a day, 6–10 sources: hire a data engineer; build the minimal stack — ingestion, storage (warehouse), visualization/reporting.
  - Larger: add lineage, observability, discoverability, automated testing — needed once one DE can't manually verify data or track what feeds what.
- **Principles**
  - Every tool adds technical debt, maintenance and upkeep; with 1 DE + 1 analyst, document flow simply (even a spreadsheet) instead of buying lineage tools.
  - Human side matters early: make it easy to tell *core/certified* datasets from research/test data; publish naming standards — timestamps end in `_ts`, booleans start with `is_` (e.g., `is_active`).
  - Automate data tests so DEs aren't stuck doing manual spot checks and can keep building.
  - Baseline stack layout: raw files -> staging -> transformed (dbt) -> BI; notebooks often attach directly to raw data so data scientists can explore sources not yet productionised.
- **Failure mode — reverse ETL without lineage**: metrics pushed back into Salesforce then re-ingested into the warehouse create circular "spaghetti" dependencies; he's seen it at large companies.
- **Observability tools** are a late-maturity purchase in practice, usually adopted once pipelines are already a mess.
- **Relevance to Sam**: as a small team, keep the stack minimal (extract scripts -> Parquet lake -> DuckDB -> simple outputs) and invest in naming standards (`_ts`, `is_`, `_id`, `_code`) and a core/research split in folder names rather than extra tools. Avoid writing derived values back into a supplier/source table you later re-ingest.

## Seattle Data Guy — What To Consider When Building Data Pipelines: Intro To Data Infrastructure Part 2 (https://www.youtube.com/watch?v=lSoEI8jia3Q)
**Before extracting, ask what fields, how often, and how often the schema changes; expect schema drift, volume growth, credential rotation and endless small requests; load raw early so you can QA source data fast.**

- **Why pipelines**: centralise data in a warehouse. The old way — SSIS or a midnight bash script with dependencies guessed via time gaps (+30 min grace) — is what the EL tool boom replaced; EL is now largely commoditised. ELT is popular because it gets raw data to you faster, but the T still has to be built.
- **Extract sources**: APIs, databases, SFTP drops (extraction really happened at the partner), web scraping.
- **Questions to ask first**
  1. *What data is required?* Source tables can have hundreds of columns — many unused, defaulted or erroneous, mixed in with valid ones. Pull what's needed (with awareness of future needs).
  2. *How often?* Monthly report vs hourly vs weekly. Most tools are comfortable in the hourly-to-daily range; Fivetran's UI floor was ~5 minutes; Airflow DAGs can use any cron schedule. Near-live needs may force a different tool/pattern.
  3. *How often does the schema change?* Yearly in some systems; ~every 3 months at Facebook, where app engineers constantly changed tables.
- **Challenges**
  - Schema changes break pipelines tightly coupled to exact column names/types.
  - Volume/velocity growth: a 1-hour job on 3x data may miss the morning report SLA.
  - Credential changes: keep credentials in configuration/secrets, not code, so rotation needs no code change.
  - Maintenance tax: a stream of small requests (add a column, change a type) eats time for new work.
- **Load raw early (his EL rationale)**: land extracts as CSV/JSON/Parquet in S3/GCS, then load into raw tables and immediately run DQ checks (dates are dates, ints are ints, categories unchanged). At a job where partners sent files via SFTP and his team had an SLA to return reports, fast raw checks let them reject bad files and ask for re-extracts quickly. ELT doesn't save as much time as claimed, but it shortens time-to-detect source problems.
- **Patterns to know**: ETL, ELT, EtLT (small transform before load), CDC.
- **Relevance to Sam**: list required fields per supplier endpoint before building; make parsers tolerant of extra/missing fields but alert on them (schema drift); benchmark job runtime vs input size so a 3x GeoNames/OSM growth doesn't blow past memory or time — chunk/stream by partition (country, date) so runtime scales linearly; run type/category checks right after landing raw files.

