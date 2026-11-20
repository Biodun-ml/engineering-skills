# kretz-reis — cross-cutting principles

1. **Choose storage and engine by access pattern, never by familiarity.** Ask "performant *for what*?": OLTP serving of many small writes, or scanning large volumes. Trickle-updating a columnar warehouse, or running heavy analytics on a transactional database, are the classic expensive mistakes. *(Reis & Housley, Fundamentals of DE; Kretz, DynamoDB schema design; Kretz, Distributed SQL)*
2. **Design keys and models around the queries you will run.** List the access patterns first. Denormalise to avoid joins where reads dominate. Keep every record bounded in size (DynamoDB's 400 KB item cap). Avoid hot partition keys. Each extra access pattern costs an index or a copy. *(Kretz, DynamoDB)*
3. **Model conceptual → logical → physical, and fix entity, grain, time and meaning explicitly.** The conceptual model (entities plus named relationships, validated with the business) is reusable by warehouses, ontologies and AI agents. Skipping it produces thousands of one-off tables and dashboards. Check AI-proposed cardinalities. *(Sami Hero; Reis/Truta, Mixed Model Arts)*
4. **Mix modeling styles to fit the problem.** Relational, analytical, application/NoSQL, ML and knowledge/graph camps are converging. Use each where its constraints fit. Agents now *produce* data too, so model their outputs deliberately. *(Reis/Truta, Mixed Model Arts; Reis, Great DE Reset)*
5. **Define metrics once, in versioned code, and serve them to every consumer.** This covers BI tools, apps and AI agents. Watch ratio-at-different-grain and multi-fact fan-out/chasm traps. Use aggregate-aware pre-aggregations for low-latency, high-concurrency serving. *(Bickell/Cube, Semantic Layer; Sami Hero on documenting metrics)*
6. **AI needs explicit semantics and deterministic guardrails.** LLMs can't infer your business definitions or data quirks from raw tables. Expose governed measures and dimensions. Pair probabilistic steps with deterministic validation. Expect agent-scale query volume (20–50x), so cache and deduplicate in front of stores. Route bulk work through cheap classical or statistical methods first. *(Bickell; Hero; Reis, Great DE Reset; Reis & Housley on classical NLP)*
7. **Keep extract/load dumb and land raw data first (ELT, raw → refined zones).** Transform in the parallel set-based engine. You can then re-run transforms without re-hitting sources. Store analytic copies as Parquet, not JSON. *(Kretz, ETL vs ELT; Kretz, Veritiv; Kretz, Stream vs Batch)*
8. **What makes something stream or batch is the trigger, not the tooling.** Event-triggered processing is streaming. Schedule- or human-triggered processing is batch, even if it reads from a queue. Don't drain queues in bursts. Let processing cadence match how fast the domain actually changes. *(Kretz, Stream vs Batch; Kretz, TCS Geospatial)*
9. **Put buffers in front of anything that can be overwhelmed, and rate-limit outbound fan-out.** Queues give back-pressure for your own DB (batch-drain status writes). Per-destination limits (Mb/s, connections, requests) protect partners' APIs. A buffer only helps if drain speed eventually exceeds inflow. *(Kretz, Nielsen 55TB; Kretz, Spark & Kafka)*
10. **Separate data, control/metadata and delivery planes.** Keep a file/job state table so work is scheduled explicitly, retried and skipped when done (idempotency). Use a lightweight trigger function plus heavy workers. Never do bulk processing inside a function that has a timeout. *(Kretz, Nielsen; Kretz, TCS Geospatial)*
11. **One consumer per concern.** When several sinks need the same event, use pub/sub or log semantics (Kinesis/Kafka, SNS→SQS) instead of one consumer doing two jobs. Single-consume queues couple teams and failure modes. *(Kretz, Veritiv; Kretz, What Is a Data Pipeline)*
12. **Don't move bytes you won't use; parallelise by independent units.** Read only the needed columns or bands (4 of 13 bands cut 60 TB to 20 TB). Partition by region or file and fan out. Avoid MapReduce-style materialisation after every step: keep transformation chains in memory or lazy and checkpoint only what you reuse. *(Kretz, TCS Geospatial; Kretz, Spark vs MapReduce; Kretz, Spark & Kafka)*
13. **Monitor in layers and alert by default.** Watch run, task, dataset (rows in/out, nulls, distincts, schema), custom metrics and alerts. At minimum, alert on failure, duration overrun and schema change, routed to a channel someone watches. Use % deviation from a baseline for drifting metrics. *(Kretz, 5 Ways to Monitor; Kretz, Airflow Problems)*
14. **Treat pipelines as software.** Use git, separate dev/QA/prod, run pytest on DAGs and transforms, deploy through merged PRs in CI/CD, and use infrastructure as code once a PoC proves itself. *(Kretz, Airflow Problems; Kretz, Veritiv; Kretz, Stonebranch)*
15. **Enterprise workflows need pre-flight checks and gates.** Check disk or memory before big transfers, scan or quarantine at the boundary, validate before load, check that targets and clusters are ready, add human approval for business-critical outputs, use business-calendar schedules, and prefer webhooks over polling. *(Kretz, Stonebranch pipelines)*
16. **Data quality starts upstream, and contracts must help the producer.** Most breaks are semantic or content changes, not schema changes. Put checks where data leaves the producing code, enforce them in CI, and invest in *detection* more than in elaborate specs. `information_schema` gets you far for databases. *(Sanderson & Freeman, Data Contracts)*
17. **Governance formalises accountability people already have.** Cover the defining, producing and using of data. Tie it to business value, keep it auditable on demand, and pair it with behaviour change and fluency. Govern AI outputs separately from data, and keep humans in the loop for consequential decisions. *(Seiner, Non-Invasive Data Governance)*
18. **Source atomic data and push logic left.** Pre-aggregated reporting extracts with unknown logic can't be reconciled. Fixing data in the BI layer traps logic in the front end. *(Sami Hero)*
19. **Fundamentals are gravity, and abstractions leak.** Higher abstraction, AI included, makes understanding the internals *more* important. Use AI as a red-team reviewer, not as an unchecked author. Avoid AI-suggested dependencies you haven't vetted. *(Reis, Do Fundamentals Matter; Reis & Housley; Reis, Great DE Reset)*
20. **Ask what the unit of work is before sizing a system.** "250B events/day" was about 17M files of roughly 15k events each. Size throughput, parallelism and cost on the real unit: files, rows or requests. *(Kretz, Nielsen 55TB)*

---

## Andreas Kretz — 5 Ways To Monitor Data Pipelines (https://www.youtube.com/watch?v=xWwG8gka1Eg)
**Monitoring has layers: pipeline runs, tasks, datasets, custom metrics, then alerts so you stop staring at dashboards.**
(Sponsored by Databand; the tool is incidental, the five-layer model is the reusable part.)

- **Mental model — observe from outside in, then inside the data.**
  1. *Pipeline runs*: did it run, when, did it fail, how long did it take. This is the minimum.
  2. *Tasks*: break the run into its stages (extract / transform / load) and time and status each one. A failed run tells you little until you know which stage broke. In his toy example extract took 1.1 s, transform 0.4 s and load 0.1 s. That breakdown is what tells you where to optimise. In real pipelines the API extract is usually the slow part.
  3. *Datasets*: look at the data passing through, not just the job. Log every read and write with its shape (rows and columns), schema, per-column null counts and distinct counts, plus a preview sample. Keep these over time so you can see trends.
  4. *Custom metrics*: business or aggregate values you compute yourself (averages, counts per category, match rates) and log on every run. Over many runs they form a history you can chart.
  5. *Alerting*: layers 1–4 all need a human to go and look. Alerts reverse that. Well-chosen alerts save the most time.
- **Mechanism (pattern, tool-agnostic):** wrap the job in a tracking context (job name, run name, project). Decorate each task function (Airflow-style `@task`). Call a dataset logger on each read/write dataframe. Call `log_metric(name, value)` for custom metrics. The library ships this metadata to a backend that stores the history and does the alerting.
- **What to check at the dataset layer (data quality):**
  - Column values staying inside their usual range.
  - Nulls suddenly appearing in a column.
  - Schema changes (added, dropped or retyped columns) that will break code downstream.
  - Row-count and column-count trends for reads compared with writes. A write that is much smaller than the read is a sign that rows were dropped silently.
- **Alert types worth having:**
  - Pipeline state (failed).
  - Pipeline duration (ran too long), which catches hangs and slow APIs.
  - Task state and task duration.
  - Schema-change alerts.
  - Missing data and table data-quality alerts.
  - Thresholds on custom metrics: >, <, =, or % deviation from a baseline.
  - Anomaly detection over the last N runs with a sensitivity setting. (note: he couldn't get the anomaly feature working. Anomaly detection needs enough run history before it is useful.)
- **Rules of thumb:**
  - Instrument even a tiny Python ETL script. If tracking works there it will work anywhere.
  - Alert on a % deviation from a known baseline rather than on a fixed absolute value whenever the metric drifts naturally.
  - Route each alert to a named owner or receiver. An alert with no recipient is noise.
- **Relevance for Sam:** for a GeoNames or GTFS batch job on a laptop you can get the same benefit without a SaaS tool. Each run, write a JSON or Parquet row to a small `runs` table in DuckDB holding run_id, task timings, row counts in and out, null counts on key columns, the schema hash and match rate. Then add a check script that fails loudly on a duration spike, a schema-hash change or a row-count drop above X%. For supplier API extracts (Viator, RateHawk), track per-call latency, HTTP error counts and records per page as custom metrics.

## Andreas Kretz — 55TB A Day: Reacting to Nielsen Marketing Cloud AWS Architecture (https://www.youtube.com/watch?v=WCQe1VP_q5A)
**Keep the data plane, the metadata/control plane and the delivery plane separate. Put queues in front of anything that can be overwhelmed, whether that is your own database or a partner's API.**

- **Architecture:**
  - *Data plane:* files land in S3, then Spark on EMR transforms and formats them, then the output goes to a second S3 bucket.
  - *Delivery plane:* Lambda functions take the processed files and upload them to more than 100 ad-network partners.
  - *Control/metadata plane:* a Postgres RDS database tracks every file and its state. A "work manager" Lambda reads that state, decides which files to process and how, and launches the EMR jobs. A second work-manager Lambda fans out many uploader Lambdas, one per file and network. When the uploaders finish they report status back through SQS, and that status ends up in RDS.
- **"250 billion events/day" is less impressive than it sounds.** Events arrive batched in files. The peak day was 55 TB in 17M files, roughly 15k events per file, with about 30M Lambda invocations and around 3,000 Lambdas running at once. Always ask what the unit of work is (files, rows or requests) before being impressed by an event count, or before sizing a system.
- **Why a metadata DB plus a work manager?** EMR/Spark batch clusters don't trigger themselves on new files. Something has to tell them what to consume. Keeping a file-state table lets you:
  - schedule work explicitly,
  - retry failures,
  - see what has been processed,
  - and it gives you idempotency, because a file that is already marked done is skipped.
- **Why Spark for this?** Each file makes sense on its own. The work is embarrassingly parallel, so files fan out across workers. **Rule:** when the units are independent, parallelise by file or partition. You don't need a shuffle.
- **Why many small Lambdas for delivery?** Each destination has its own API, auth, tokens and format. Per-destination functions isolate those differences and let each one be retried on its own.
- **Elastic load:** traffic runs from about 1 TB in the quietest hour to about 6 TB at peak. Serverless functions scale up and down on their own, while EMR still needs sizing.
- **Failure mode 1 — you DDoS your partners.** Unlimited fan-out pushed data faster than partner servers could take it, and partners complained. The fix was rate limiting in the work manager:
  - limits set per destination network on Mb/s, open HTTP connections and request count (for example 250 Mb/s for one network),
  - work over the limit stays unscheduled ("waits in the queue") and drains gradually,
  - decisions take file size and event count into account.

  **Lesson:** elastic compute moves the bottleneck downstream, so be a good neighbour.
- **Failure mode 2 — you DDoS your own database.** Thousands of Lambdas writing status straight to Postgres ran out of DB connections. The fix:
  - Lambdas write to an SQS queue as a buffer.
  - One consumer Lambda drains it in batches and writes to RDS.
  - Status updates don't need to be real time, and a delay of a few seconds is fine.

  This is the **back-pressure** pattern. The queue absorbs bursts and the consumer writes as fast as the sink can take. **Gotcha:** a buffer only helps if inflow eventually drops below drain speed. Otherwise the backlog grows without limit and you fall further and further behind. (Kretz also suggests a NoSQL/document store as an alternative sink for high-volume status writes.)
- **Three recurring concerns for any big system:** scale, rate limiting and cost.
- **Relevance for Sam:**
  - Supplier API extraction (Viator, RateHawk) is this same problem in reverse. You are the client hammering *their* API. Build a per-supplier rate limiter (token bucket or semaphore on concurrent requests plus requests/sec) into your async extractors, and respect 429 / Retry-After.
  - Keep a small file-state or job-state table (a DuckDB or SQLite table: file/chunk id, status, attempts, timestamps) so chunked jobs over millions of rows can resume and skip finished chunks.
  - Never have N parallel workers write straight into one SQLite/DuckDB file. Have the workers write Parquet parts, or push results into a queue that a single writer drains.

## Andreas Kretz — Apache Pinot Explained: Realtime Analytics at LinkedIn, Uber & More (https://www.youtube.com/watch?v=W-bwmvrcDVI)
**Pinot exists to give you fresh data and fast queries at the same time, at high concurrency. Most systems can only give you two of those three.**
(Sponsored by StarTree, a managed Pinot service.)

- **Three separate latency and scale axes. Name the one you actually need:**
  - *Data latency*: the time from an event happening to it being queryable.
  - *Query latency*: the time from asking to getting an answer.
  - *Concurrency*: how many queries run at once. User-facing analytics means thousands.

  Making queries fast on *stale* data is easy with materialized views, re-indexing or pre-aggregation, but you pay for it in data latency. The hard part is keeping both small.
- **Origin problem:** LinkedIn's "who viewed my profile" needed fresh counts broken down by location, company and role, at sub-second latency, under global traffic. Warehouses couldn't ingest fast enough or handle that concurrency. Key-value stores were fast but couldn't aggregate across many dimensions. Pinot sits between the two.
- **Architecture (each role scales independently):**
  - *Controller*: coordination. It handles table creation, segment assignment and config. Apache Helix handles segment assignment and balancing. ZooKeeper stores cluster metadata, elects the leader and tracks node health. The lesson he draws: any distributed system needs solid management of cluster state (which nodes are alive, where data lives, how to rebalance).
  - *Broker*: stateless query entry point (REST). It sends sub-queries only to the servers holding relevant data, then merges the partial results. Add brokers to handle more dashboards and users. It also does hybrid queries: a 30-day query reads recent rows from realtime servers and older rows from offline servers.
  - *Realtime servers*: consume from Kafka, Kinesis or Pub/Sub. Rows sit in memory and are queryable immediately, while segments are built and sealed in the background. They also handle offsets, back-pressure and recovery.
  - *Offline servers*: serve batch-loaded data (Parquet, Avro or CSV in S3, GCS or ADLS). External jobs (Spark, Airflow, scripts) convert that data into segments. "Offline" only means the data didn't arrive through a stream.
- **Segment:** a small, indexed, columnar file. It is the unit of both storage and query execution. Segments live on the local disk of their assigned server, with no central lake in the hot path. That is what keeps latency predictable. Tiered storage can move old segments to object storage.
- **Upserts:** Pinot keeps a primary-key index and keeps only the latest row per key during ingestion. Most OLAP stores assume immutable, append-only data. This matters when records change over time (order status, booking state).
- **Performance mechanisms:**
  - *Columnar storage*: reads only the columns a query needs, cuts I/O, compresses and encodes well, and speeds up filters and aggregates.
  - *Star-tree index*: pre-computes partial aggregates over common combinations of dimensions. Good for dashboards with many group-by/filter combinations, top-N queries and high-cardinality aggregations.
  - *Other indexes*: inverted (filters), sorted (ranges and top-K), range (numeric comparisons), bloom filters, text, **H3 geospatial**, JSON. Indexes can be added without downtime.
  - Typical query time is under 100 ms, at thousands of queries per second.
- **Decision rule:** reach for a realtime OLAP store like Pinot only when you need *user-facing*, high-concurrency, seconds-fresh aggregates. For internal analytics a warehouse or lakehouse is enough, and for a single user DuckDB is enough.
- **Relevance for Sam:**
  - The ideas carry over even without Pinot: columnar files, pruning, pre-aggregated rollups for common dimension combinations, and H3 cell ids as an index for geospatial lookups.
  - For location matching on a laptop, a precomputed H3 cell column in Parquet works as a cheap spatial index. Joining on the cell and its neighbours is far cheaper than computing all-pairs distances.
  - Upsert-by-key semantics are the pattern for supplier inventory snapshots, where only the latest state matters.

## Andreas Kretz — Data Engineer Reacts: TCS Geospatial Data Analysis on AWS (https://www.youtube.com/watch?v=MxVJAvFSTXg)
**For large geospatial work, the biggest savings come from reading only the bands, columns and regions you need, and from fanning out many small parallel jobs. Processing frequency should follow how fast the real world changes.**

- **Use case:** TCS's agriculture platform covers about 329M hectares and roughly 20 use cases: crop area, crop health, yield and soil moisture, at field and regional level.
- **Data sources:**
  - open satellite imagery (ISRO, NASA),
  - the AWS Registry of Open Data (the same imagery, already sitting in S3),
  - government GIS data such as field boundaries,
  - field-agent observations stored as GIS data in PostgreSQL/PostGIS.
- **Projection pushdown, in the satellite version:** a satellite captures 13 spectral bands but they only need 4. Pulling only those 4 from the open-data registry cut data from **60 TB to 20 TB** and also reduced transfer costs. The general principle: **don't move bytes you won't use.** Read only the needed columns or bands, at the source.
- **Mechanism:**
  - When new data is available, a Lambda **only triggers** the processing. It does no heavy lifting itself, because Lambda has a 15-minute timeout and limited memory.
  - SageMaker Processing runs about **3,400 parallel jobs**, each joining satellite tiles with field boundaries.
  - Statistics go to PostGIS and image outputs go to S3.
  - The UI runs on EKS/EC2 (Graviton for cost) and reads through CloudFront, which caches the data. One use case serves about 10 GB to a user.
- **Pattern: "trigger function + heavy workers."** Use a lightweight event handler to orchestrate and a properly sized batch compute service to process. Never do bulk processing inside a function that has a timeout.
- **Cadence follows the domain:** agriculture changes slowly, so they process every 5–10 days. They could tighten this to daily. Don't build streaming for data that changes daily or weekly.
- **Scaling story:**
  - *Spatial*: India today, Indonesia and Thailand next. Partitioning by region makes adding a region simply more parallel jobs.
  - *Temporal*: shorten the interval when needed.
- **Kretz's review points:**
  - Apps should never hit S3 or Postgres directly. Put an API in front.
  - Postgres with the GIS extension is a reasonable choice. A document store like MongoDB with geo indexes is an alternative, but don't switch unless you need what it offers.
  - Caching (CDN) in front of heavy, read-mostly query results makes things faster and also cuts transfer cost.
  - The system is fairly fault-tolerant because the inputs are immutable files that can be re-read and the jobs can simply be re-run.
- **Relevance for Sam (GeoNames, OSM, GTFS):**
  - Reading only the needed columns from Parquet in DuckDB/Polars is the laptop version of "4 of 13 bands."
  - Partition location work by country or region (or H3 cell) so each chunk fits in RAM and runs in parallel. This is the same structure as their per-region fan-out.
  - Precompute and store results in PostGIS or Parquet, and serve cached tiles or aggregates rather than recomputing per request.
  - Match refresh cadence to how often sources actually change. GeoNames dumps and GTFS feeds change daily to weekly, so a batch refresh is right.

## Andreas Kretz — Data Engineering Expert Reacts to AWS Architecture at Veritiv (https://www.youtube.com/watch?v=WgQ6lYSl4L4)
**A file-driven, serverless batch pipeline (CDC into the warehouse, an S3 drop bucket, event-triggered Lambdas, raw/refined zones) can be shipped in weeks. Watch out for single consumers doing two jobs and for components nobody can justify.**

- **Business driver:** COVID raised demand for timely order and demand analytics. The team rebuilt invoice-level sales and demand reporting in the cloud.
- **Flow:**
  1. On-prem sales database → VPN → AWS DMS with **CDC** → Redshift.
  2. The ERP also drops flat order and demand files into an **S3 drop bucket**. An S3 event goes to SQS, which starts an "initiator" Lambda.
  3. The initiator reads file metadata from DynamoDB and writes the data into the lake's **raw** and **refined** zones.
  4. An S3 trigger on refined starts a "complex" Lambda. It merges the order and inventory streams, applies business rules and exceptions, and runs Python statsmodels forecasting.
  5. Results are loaded into Redshift for BI.
  6. SageMaker work runs against the lake.
- **Mechanisms:**
  - *CDC (change data capture)*: stream only the row changes from the source database's log instead of re-dumping whole tables. This is the standard way to sync an operational DB into analytics without heavy load on the source.
  - *The queue message carries metadata, not the file.* The consumer reads the file from S3 using the pointer in the message (the claim-check pattern).
- **Kretz's critiques (the trade-offs):**
  - **An SQS message is consumed once.** One Lambda is therefore forced to do two jobs: write to the lake *and* write to DynamoDB. That couples both concerns to one deployable and one team. The fix is a fan-out-capable log or stream (Kinesis or Kafka, or SNS→multiple SQS) so each sink gets its own consumer. Then the two can evolve, fail and be owned separately.
  - **File-based integration is clunky.** He would rather have the ERP push through an API. Drop buckets are tolerated mainly because the ERP produces standard exports.
  - **Unexplained components are a red flag.** DynamoDB's role was never justified. Every store adds resources, cost and failure surface.
  - **Lake data lives in S3 on both sides.** Juggling raw and refined files gets tedious, and merges across file sets are where complexity accumulates.
  - Redshift acts mainly as the easy-to-query serving layer for BI. ML reads from the lake, not from the warehouse or DynamoDB.
- **Benefits they reported:** a small focused team, delivery in 4–5 weeks, and low cost from managed and serverless pieces.
- **Next step: infrastructure as code (CloudFormation) and DevOps.** Kretz's rule: a proof of concept built by hand must become automated, reproducible infrastructure before it matures. Monolithic Lambdas make that harder.
- **Rules of thumb:**
  - Separate one job per consumer when there are multiple sinks.
  - Use pub/sub or log semantics when more than one downstream needs the same event.
  - Use raw → refined zones so you can always reprocess from raw.
  - Keep the warehouse as the serving layer and do heavy merges in the lake or compute layer.
- **Relevance for Sam:**
  - Keep a raw zone for supplier API pulls (immutable JSON or Parquet as received, partitioned by extract date) and a refined zone for cleaned data. You can then rebuild refined whenever matching logic changes, without re-calling rate-limited APIs.
  - If one script both writes Parquet and updates a status DB, split it into separate steps.

## Andreas Kretz — ETL vs ELT Explained: Simple Example (https://www.youtube.com/watch?v=bv7tlrh32U4)
**ETL transforms inside the extracting process. ELT lands raw data in staging and transforms inside the warehouse, where the parallel compute already lives.**

- **ETL:** source (a relational database or an API) → extract → transform inside the job (a Lambda, cloud function or container) → load straight into warehouse fact and dimension tables. All the transformation logic and its compute load sit in that one job.
- **ELT:** extract → load as-is into a **staging table** in the warehouse → a SQL job, stored procedure or UDF (for example PL/pgSQL with DECLARE/BEGIN/SELECT, possibly Python UDFs) transforms staging into the destination facts and dimensions. A procedure can be triggered when new data arrives.
- **Why ELT won for warehouses:**
  - The extract-and-load step stays tiny and dumb. It needs little memory or CPU and no special transformation features in integration tools (Informatica and the like simply move the data).
  - Transformations run on the warehouse's distributed engine. A multi-node Redshift cluster processes a big staging table in parallel.
  - His example: a 1 GB (or 1 TB) file. In ETL, the single Lambda or container must hold and transform all of it, serially, and may run out of memory or time. In ELT, the loader only streams it into staging and the cluster does the heavy work.
- **Scope:** he presents ELT specifically for the analytics/warehouse side of a platform, not for operational NoSQL targets.
- **Trade-offs (my synthesis of his points):**
  - ETL keeps junk out of the warehouse but puts the scaling burden on the extraction job.
  - ELT keeps raw data available, so you can re-transform without re-extracting. It costs warehouse compute and storage and needs a staging discipline.
- **Rule of thumb:** keep extraction and loading as simple and reliable as possible. Push heavy, re-runnable transformations into the engine best at set-based parallel processing.
- **Relevance for Sam:** on a laptop, "the warehouse" can be DuckDB. Land raw API/CSV/GTFS data unchanged as Parquet (staging), then do the transforms as DuckDB SQL over those files. DuckDB is parallel, vectorised and can spill to disk. This beats row-by-row Python in the extractor, and you can re-run transforms after fixing logic without hitting the API again.

## Andreas Kretz — What Is A Data Pipeline Explained (https://www.youtube.com/watch?v=8-Hp31WHtI8)
**A pipeline is a chain of ingest → process → store that does one discrete job. Many pipelines can share ingestion and storage.**
(Short, informal 2019 whiteboard clip.)

- **Definition:** the word "pipeline" gets overused. A clean definition has three stages:
  - *ingest*: APIs receive data and push it into a Kafka topic,
  - *process*: Spark runs some analytics, possibly in several steps,
  - *store*: write to a sink such as HBase (key-value), HDFS or Hive.
- **Composability:** a second pipeline can either:
  - have its own API and topic, or
  - reuse the same ingested data with different analytics and a different sink (for example, the same topic goes to analytics-2 and lands in HDFS/Hive instead of HBase).

  A pipeline can also start from storage (read from HDFS, process, write back). Each one has its own discrete purpose.
- **Rules of thumb:**
  - Design ingestion once, as a shared buffer (a topic or log), then hang several purpose-specific pipelines off it.
  - Define each pipeline by its function and output, not by its tools.
- (note: the specific tools, HBase/HDFS/Hive and the Cloudera era, are dated. The ingest/process/store decomposition is still the right mental model. On a laptop: raw landing folder → DuckDB/Polars transform → Parquet/DuckDB store, with several derived pipelines reading the same raw zone.)

## Andreas Kretz — How to AWS DynamoDB Schema Design (https://www.youtube.com/watch?v=UXpQ5H939fk)
**In NoSQL you design the keys around the queries you will run. Relational design normalises and joins. Key-value and document design denormalises and picks partition and sort keys so each access is a single lookup or range read.**

- **Starting point (relational):** Customer(customer_id PK), Flight(flight_id PK, distance, origin…), Review(review_id PK, customer_id FK, flight_id FK, legroom, service…). The UI reads by customer: join the tables and filter `WHERE customer_id = ?`. This works, but you have to maintain the indexes and joins. At millions of customers with 10–20 reviews each, he asks whether something simpler is possible.
- **DynamoDB mechanics:**
  - Items are JSON-like documents.
  - The key is a **partition (primary) key** plus an optional **sort key**. Writing to an existing key with no sort key *overwrites* the item.
  - **Item size is capped at 400 KB.**
  - Attributes are schema-free and can be nested.
- **Option 1 — one item per customer, append each review as a nested attribute.** Rejected: the item grows without bound and eventually hits the 400 KB cap. (This "wide row" style *does* work in wide-column stores like Cassandra, which allow huge numbers of columns per row.)
- **Option 2 — PK = customer_id, SK = flight_id, with review JSON and flight JSON as attributes.** One query fetches all of a customer's rows. Costs:
  - flight data is duplicated across every reviewer, which is usually acceptable because storage is cheap. Alternatively store only flight_id and look up a separate flight table.
  - "all reviews for flight X" now needs a full table scan.
- **Option 3 — PK = customer_id, SK = timestamp.** Rows come back already sorted by time. You get cheap range queries ("2021 only") and cheap projections (just timestamps and flight ids for a list view, with details fetched lazily). Add a **secondary index on flight_id** for the airline's access pattern. That costs extra write work and storage.
- **Trade-offs:**
  - **Write vs read performance:** designs that make writes cheap often make reads expensive, and the reverse.
  - **Every extra access pattern needs another index or another copy of the data.** Enumerate the access patterns before you design.
  - **Hot partitions (hotspotting):** if many writes share one partition key (for example one huge airline receiving a flood of reviews), they all land on the same physical partition and get throttled. Choose high-cardinality, evenly spread keys.
- **Benefits:** no joins. One request returns everything a screen needs. Schema flexibility means the review payload can change without migrations.
- **Rules of thumb:**
  - Start from the screens and APIs. List the queries first, then design PK/SK.
  - Bound every item's growth: one item per event, not one ever-growing document.
  - Accept some duplication to avoid joins. Normalise only what changes often or is large.
  - Use a time-based sort key when you will want recent-first or date-range reads.
- **Relevance for Sam:** the same query-first thinking applies to booking-flow and supplier caches. For example, key a cache of supplier products by (destination_id, product_id) or (supplier, updated_at) depending on how the app reads it. Watch for hot keys such as one giant city that gets all the traffic.

## Andreas Kretz — Solving Engineers' 4 Biggest Airflow Problems (https://www.youtube.com/watch?v=b9bMNEh8bes)
**Running an orchestrator well means four things: a deployment you can maintain, a realistic local dev environment, git-based CI/CD with tests, and alerting that reaches a human.**
(Sponsored by Astronomer, a managed Airflow service. The tool demo is skipped and the generic problems kept.)

- **Problem 1 — deploying and maintaining Airflow.** Two problems hide in one:
  - *initial setup*: separate QA and prod environments, choosing instances, executor type (Celery vs Kubernetes), worker queues and worker types, min/max autoscaling, scheduler and runtime versions, plus an explosion of config options,
  - *ongoing ops*: watching resource use, scaling, fixing failures, upgrades.

  Self-hosting costs real time and expertise. A managed service trades money for that (it shows cost per hour up front).
- **Problem 2 — local development.** Airflow isn't one container. It needs a metadata DB, scheduler, webserver and workers, which is 5+ containers in a docker-compose setup. A good setup has:
  - a one-command local environment (UI on :8080),
  - a project template (DAGs, tests, config),
  - **pytest tests for DAGs and tasks**,
  - one command to deploy.
- **Problem 3 — source control and CI/CD.** Never deploy straight from a laptop to prod. Colleagues edit pipelines too. The pattern:
  - a git repo with `dev` and `main` branches,
  - a PR from dev to main triggers CI (GitHub Actions, GitLab, Jenkins, Bitbucket, cloud-native tools),
  - CI runs pytest, then builds containers and deploys once the PR is approved and merged.
- **Problem 4 — observability and alerting.** Kretz admits he has often not known that something was failing. Minimum alerts:
  - **DAG failure**,
  - **task duration exceeded** (catches hangs and slowdowns),
  - routed to email, PagerDuty, or **Slack via webhook**, where a phone push means you can't miss it.

  You also need an at-a-glance view of all DAG runs (green/red) with drill-down to the failing task and its logs.
- **Rules of thumb:**
  - Keep separate QA and prod environments.
  - Tests run before every deploy.
  - Deploy through a merged PR, not by hand.
  - Every pipeline needs a failure alert and a duration alert that goes to a channel someone actually watches.
- **Relevance for Sam:**
  - Full Airflow on a 16 GB laptop is heavy, at 5+ containers. For solo or small-team batch jobs, a lighter orchestrator or plain cron/Task Scheduler running Python entrypoints may be enough.
  - Keep the same discipline: pipelines in git, pytest on transforms (small fixture Parquet files), a CI check on PRs, and a Slack/Telegram/email webhook on failure or duration overrun.

## Andreas Kretz — Stream vs Batch Processing Explained with Examples (https://www.youtube.com/watch?v=1xgBQTF24mU)
**What makes something streaming or batch is what starts the processing: an event, or a schedule (or a manual run). The presence of a message queue doesn't decide it.**

- **Stream / event-driven:**
  - A client (another internal system producing transactions, a car or weather station in IoT, an app posting a tweet) sends an event to an API. The API puts it on a message queue, and processing and storage happen immediately as data flows in one direction.
  - **This doesn't require a queue.** A file dropped into storage with a trigger that starts processing straight away is also event-driven.
- **Batch:**
  - Processing starts **on a fixed schedule or manually**. Examples: every 10 minutes or hourly, read all new files and process them; or a scraper that runs on a schedule and writes results.
  - **Even with a message queue, if the real processing (validate, transform, store) runs on a schedule and reads a bunch of messages at once, it is batch.**
- **Gotcha — scheduled bulk reads from a queue:** queues are designed for load spread evenly over time. Hourly jobs that drain the queue in spikes put heavy pressure on it and use up the headroom you need for real spikes. It is not the queue's intended pattern.
- **Combining them (the lambda-ish hybrid):**
  - Live data sits in an OLTP or NoSQL store, and an hourly batch copies files into the lake or warehouse.
  - Better still: **two consumers off the stream**. One writes events into the operational store in real time. The other dumps them into object storage (S3, the raw/staging zone). A *batch* job then reads those files into the lake or warehouse.
  - Convert JSON to **Parquet** on the way into the final lake, because it is much easier and faster to read for analytics.
  - Hybrids always need **storage in between** (a file store or queue) to decouple the two speeds.
- **Rules of thumb:**
  - Ask what *starts* the processing. A trigger means stream; a schedule or a human means batch.
  - Don't use a queue as a batch staging table. Land events to files and batch-process the files.
  - Store analytic copies as Parquet, not JSON.
- **Relevance for Sam:**
  - Most of Sam's work (GeoNames dumps, GTFS feeds, supplier catalogue syncs) is batch by nature. Run it on a schedule against landed files.
  - Event-driven only makes sense for booking-flow events (booking created, cancelled), where an immediate reaction matters.
  - For supplier webhooks, land each payload as a file or row in a raw zone, then batch-compact it into Parquet.

## Andreas Kretz — Three Real-World Data Pipelines in Action (Workflow Automation with Stonebranch) (https://www.youtube.com/watch?v=u3sRYMSzmgY)
**Enterprise pipelines span many tools and platforms. The orchestrator's real job is connectivity (agents through firewalls), business-calendar scheduling, event triggers, staging checks, human approvals, fan-out transfers and step-level monitoring.**
(Sponsored by Stonebranch. The general patterns are kept.)

- **Orchestrator architecture: three layers.**
  - *Execute*: agent-based or agentless.
  - *Automate*: controllers. Separate dev, test and prod controllers let you promote bundled workflows between stages, a CI/CD-like flow.
  - *Operate*: UI, web services, CLI.
- **Agents vs agentless:**
  - Agentless means REST over HTTPS:443, JDBC or SSH. It works when the firewall allows inbound calls, which it usually does for HTTPS APIs.
  - Corporate firewalls typically **block inbound** connections. To run a job on an internal Linux box, you install an **agent that dials out** to the controller. The controller then sends commands back over that outbound connection.
  - **Lesson:** in locked-down networks, connectivity direction decides the architecture. Make the inside call out.
- **Core concepts:**
  - *Tasks* are units of execution: Linux/Windows, file transfer, SQL, email, Glue, Databricks, k8s, or another orchestrator such as triggering an Airflow DAG.
  - *Triggers* are schedules.
  - *Monitors* react to events.
  - *Workflows* are DAGs of tasks.
- **Scheduling beyond cron:** business calendars (for example 09:00 on the first business day of the month), with the ability to list the upcoming qualifying run times to check them. Business processes like commission runs happen on specific business dates that plain cron can't express.
- **Event-driven starts:**
  - file monitors (a new file in a folder), system monitors (disk, CPU), email monitors,
  - **polling vs webhooks**. Polling repeatedly asks "anything new?" and wastes requests and bandwidth when nothing has changed. Webhooks register once and get pushed on new data (for example an AWS SNS message to an HTTPS webhook). Prefer push when the source supports it.
- **Pipeline 1 ("DataOps"):**
  1. A monitor for new sellers starts it; SAP service orders and SQL Server data are extracted.
  2. **Check staging disk space** first, because SAP can only export files, so a staging area is mandatory. Then file transfers.
  3. **Human approval step** (a Microsoft Teams action) before upload.
  4. Upload to S3 and Azure Blob.
  5. **Informatica validation** (correctness, missing data).
  6. Fan out to two branches. *Snowflake branch*: check the tables are ready, load from S3, refresh Tableau. *Azure Data Factory / Databricks branch*: check whether the cluster is running and start it if not, run the notebook job, refresh Power BI. (He guesses one branch holds full history and the other current data.)
- **Pipeline 2 (SAP commissions):**
  1. A webhook on an SNS message signals new seller sales data; the data is downloaded and transferred to SAP.
  2. SAP bonus-model calculation, daily commission runs, a weekly Wednesday run, and a monthly run.
  3. Email the SAP team. Approval through Slack. Promote to the target through REST.
- **Pipeline 3 (mainframe):**
  1. A webhook on new sales data triggers it.
  2. **Check EC2 free space against the roughly 200 GB needed.** If short, attach EBS volumes using a Terraform template.
  3. Transfer the data from the mainframe and **virus scan** it. Infected files go to quarantine and open a ticket. Clean files go to the S3 lake.
  4. Glue ETL into Redshift and SQL databases, then refresh Power BI. SAP bonus/commission runs follow.
- **Data transfer feature:** cloud-to-cloud transfers stream in memory with no intermediate storage. Large files are **chunked and sent in parallel**. This makes fanning one source out to several destinations efficient.
- **Operations:** a live per-step status view (success, running, waiting, resource-wait). An operator can **force-finish or skip** a hung step so the rest continues.
- **Checklist from these pipelines:**
  - a pre-flight resource check (disk or memory) before a big transfer,
  - security scanning or quarantine at the boundary,
  - validation before load,
  - "is the target ready / is the cluster up" checks,
  - human approval gates for business-critical outputs,
  - dashboard refresh as the final step,
  - alert or ticket on any failure path.
- **Kretz's takeaway:** real workflows are multi-tool and multi-platform, each piece with its own quirks. Knowing one tool isn't enough.
- **Relevance for Sam:**
  - Add pre-flight checks to laptop jobs: free disk for DuckDB spill and temp Parquet, and available RAM before you pick the chunk size.
  - Prefer supplier webhooks over polling where they are offered.
  - Chunk big transfers and process the chunks in parallel.
  - Put a validation gate before you publish the master location dataset.

## Andreas Kretz — What Is Distributed SQL? A Better Way to Scale Your Database (https://www.youtube.com/watch?v=1cokFlB0aTw)
**Distributed SQL scales a relational, ACID database out across nodes (consensus-replicated key-value storage underneath, a stateless SQL layer on top). That removes manual sharding and read-replica workarounds while keeping transactions.**
(Sponsored by PingCAP/TiDB. Vendor claims are flagged.)

- **History and the problem:**
  - *Relational on one primary*: reliable and consistent, but it scales *up* (bigger box, more cost). Read replicas help reads, yet **every write still goes through the single primary**. His analogy: a bigger funnel on the same bottle. Next comes **manual sharding**: many databases and schemas, routing logic in the app, every schema change coordinated across shards.
  - *NoSQL*: fast and flexible, but you give up ACID, strong consistency and integrity.
  - The usual result is a "duct-taped" stack: an OLTP database plus an analytics database plus a queue plus thousands of lines of ETL plus sharding, caching, retries and failover scripts. It is fragile and expensive.
- **ACID refresher:** Atomic means all or nothing. Consistent means data always valid. Isolated means concurrent transactions don't interfere. Durable means a commit survives.
- **Mechanism:**
  - Data is split into ranges spread across nodes, each range replicated and kept in agreement by **Raft consensus**. A leader proposes a change, followers agree, and the write commits only after a quorum. If a node dies, the others still know the latest committed state. It is like blockchain consensus but inside a trusted cluster, so it is fast.
  - TiDB layout:
    - a stateless **SQL layer** that parses and plans queries and routes work,
    - a **Placement Driver**, the "brain" that tracks data location, balances load and scales,
    - **TiKV**, the row store (distributed key-value with Raft),
    - **TiFlash**, a columnar replica for live dashboards and HTAP queries (not a warehouse replacement),
    - an **object-storage tier** for cold and large data.
  - **Compute and storage are separated**, so each scales independently.
  - It speaks the MySQL protocol, so existing clients, connectors and SQLAlchemy keep working.
- **Case studies (vendor-reported):**
  - A fintech data-API company replaced Aurora plus dozens of manually sharded MySQL clusters. Reported results: about 25% less ops overhead, about 5x resource utilisation, zero-downtime upgrades.
  - Atlassian Forge runs more than 3M per-tenant tables in one cluster for multi-tenant isolation.
- **Decision guidance:**
  - Consider distributed SQL when write volume or data size outgrows one primary, when you are about to hand-shard, or when you need multi-region consistency.
  - If one Postgres or MySQL node with replicas comfortably handles the load, stay there. Distributed systems carry real operational and latency complexity.
  - (note: his line that a London write is visible "instantly" in New York is marketing. Strongly consistent cross-region writes pay consensus round-trip latency, so data placement and locality still matter.)
- **Gotchas:** sharding in the app pushes routing and schema-migration pain into application code, so avoid it if you can. Read replicas don't fix write bottlenecks. Replication-based multi-region setups bring lag and painful failovers.
- **Relevance for Sam:** this concerns Trameter's transactional and booking backends once write load grows. It doesn't apply to laptop analytics. The useful principles are separating compute from storage, adding columnar replicas for operational dashboards, and not hand-sharding prematurely.

## Andreas Kretz — Why Spark is Faster Than Hadoop MapReduce (https://www.youtube.com/watch?v=myVWLaUdJac)
**MapReduce writes intermediate results to disk between every step. Spark keeps them in RAM across the chain of transformations, and it parallelises the work, so multi-step analytics avoid most disk I/O.**
(Short 2019-era whiteboard clip.)

- **Batch vs stream framing:**
  - *Batch*: take a large chunk of data (possibly terabytes), run the analytics in one process, reach a conclusion.
  - *Stream*: many small records arrive fast and recurring analytics run as they arrive, with results going to storage or a visualisation.
  - Distributed engines handle both by splitting the work across parallel workers instead of one serial scan.
- **MapReduce mechanism:** read from HDFS, process, **write back to HDFS**, read again for the next step, and so on. Every stage round-trips through disk because there is no in-memory reuse.
- **Spark mechanism:** load from storage (HDFS, Cassandra…) into **RAM** once, chain the transformations in memory, and write only the final result back. Two advantages: parallel processing, and much less disk I/O.
- **Cost:** you need a lot of RAM. The "10x faster" claim depends heavily on the workload (Kretz thinks it's often an understatement).
- (note: modern Spark spills to disk when data exceeds memory, plans queries lazily, and uses columnar formats such as Parquet. "In memory" is the gist, not an absolute.)
- **Relevance for Sam:** the same principle drives single-node tools.
  - DuckDB and Polars (lazy mode) fuse a chain of operations and avoid writing intermediates.
  - On a 16 GB laptop, avoid the MapReduce anti-pattern of materialising a CSV after every step. Build one lazy query plan, or a DuckDB SQL chain using CTEs, and write Parquet only at checkpoints you will actually reuse.
  - When data exceeds RAM, rely on streaming/out-of-core execution and partition-wise processing rather than loading everything.

## Andreas Kretz — Why You Need To Learn Apache Spark and Kafka (Tutorial 1) (https://www.youtube.com/watch?v=hf5isv0gdUU)
**A scalable data platform is parallel at every stage (ingest, buffer, process, store) so that no single stage becomes the bottleneck. Kafka and Spark are popular because both are distributed by design.**
(2019 video. The Hadoop-era stack is dated, but the "parallel at every stage" principle still holds.)

- **Platform blueprint:** ingest (REST API) → buffer (Kafka) → process (Spark) → store (HDFS, HBase/NoSQL) → visualise (web UI or BI tools like Tableau/Power BI, with Hive giving SQL over HDFS).
- **Parallelism at each stage:**
  - *Ingest*: a load balancer in front of several API servers.
  - *Buffer*: Kafka topics are split into **partitions** spread across brokers, so producers write in parallel. The queue also absorbs **back-pressure** when processing falls behind ingestion.
  - *Process*: Spark workers read Kafka partitions in parallel (direct stream), one partition mapping to a worker task.
  - *Store*: HDFS writes in parallel across data nodes.
  - **Data locality:** run Spark workers on the HDFS data nodes so the compute goes to the data and less moves over the network.
  - Batch re-reads from HDFS happen in parallel too.
- **Composability:** Spark can write back to another Kafka topic, so stages can be chained to refine data step by step.
- **Rule of thumb:** throughput is limited by the least-parallel stage. Find it, partition it, or put a buffer in front of it.
- **Relevance for Sam:**
  - The "no serial bottleneck" principle maps onto laptop pipelines. Async or concurrent API extraction (bounded by rate limits) → landing files → parallel DuckDB/Polars processing over partitioned Parquet → partitioned outputs.
  - The single-threaded step (often a Python loop, or a single writer) is what to profile and fix first.

## Joe Reis & Matt Housley — Fundamentals of Data Engineering (GOTO Book Club 2025) (https://www.youtube.com/watch?v=VdGVmqiJkeg)
**Data engineering is a lifecycle plus its undercurrents, not a set of tools. The most common expensive mistake is picking a storage or processing system because it is familiar rather than because it fits the workload.**
(A three-year-retrospective conversation. Much of it is about AI and careers. The engineering substance is extracted here.)

- **Why the book exists:**
  - Search results defined data engineering by tools ("Hadoop, Pig, Spark") or by vendor SEO.
  - The cloud era made tools easy, which also made it easy to build "haphazard junk."
  - They define the job as the **data engineering lifecycle** (generation, ingestion, storage, transformation, serving) plus **undercurrents**: security, data management, DataOps, architecture, orchestration, software engineering. AI doesn't remove any of that. It only speeds it up, for better or worse.
- **"Curse of familiarity" (Reis's term):**
  - Modern tools look so easy that people misuse them without understanding the internals.
  - People treat new things like familiar old things, for example applying on-prem big-data habits to the cloud.
  - The GenAI version: assistants explain things but don't give you the underlying understanding.
- **The most common failure they saw in consulting: storage that doesn't fit the workload.**
  - Using *transactional (row/OLTP) databases* for large, batch-oriented analytical data creates bottlenecks.
  - Running *transactional or streaming updates against columnar warehouses* (for example trickle-updating Snowflake) brings them to their knees and costs a fortune.
  - **Always ask "performant for what?"** Serving 100k updates/sec from a million users is a different problem from scanning 10 PB efficiently. Choose by access pattern. The book has a whole chapter on choosing technologies.
- **Classical NLP and simple models over LLMs, where possible:** citing David Talby (John Snow Labs), LLMs hallucinate and are very expensive. The approach is to route as much work as possible through cheap statistical or classical NLP (tokenisation and statistics, often at Spark scale), using LLMs only for the remainder. A cited study found **logistic regression** was the best cost-adjusted model against deep learning and LLMs for a medical classification task. Housley predicts demand for refactoring costly LLM pipelines into classical methods.
- **Where DEs add value next to data science:** help data scientists move off single-node, in-memory tools (pandas) once data reaches real scale (large web apps, logs, millions of users).
- **Data quality is central and agile:** good DEs keep improving data quality iteratively. The same applies to AI training data: scaling laws are only as good as the data you scale on, so filtering scraped data is data engineering.
- **AI usage guidance:**
  - Learn the fundamentals and write SQL by hand, otherwise you can't spot when the AI is wrong.
  - Use AI as a **red team** (Terence Tao's framing): review code for security and privacy holes, critique drafts. Don't hand it the blue-team job of writing everything.
  - Beware AI-suggested dependencies. Reis found a vibe-coded app pushing an obscure package tied to a paid service, which created lock-in and supply-chain risk.
  - Build bottom-up so you understand each layer.
  - A careful engineer *should* remain the review bottleneck on generated code (Chip Huyen's point).
  - Generated PRs that change APIs are especially dangerous because public API changes are hard to undo.
- **Expertise:** you build it by grinding through repetitive problems. When you've seen a problem five times, you know the answer. LLMs give varied answers where an expert gives the known one.
- **Relevance for Sam:**
  - Choose the engine per workload: DuckDB/Parquet for scans over millions of location rows, Postgres for booking transactions, never trickle updates into a columnar store.
  - For entity matching (place names, supplier hotels), use cheap deterministic and statistical methods first (normalisation, tokenisation, TF-IDF/trigram similarity, geo blocking) and send only the ambiguous remainder to an LLM. This saves cost and avoids hallucinated matches.
  - Use Claude as a red-team reviewer of pipelines.

## Joe Reis w/ Chad Sanderson & Mark Freeman — Data Contracts Are For Software Engineers, Not Just Data Teams (https://www.youtube.com/watch?v=is1oeGAOhrk)
**Most data quality problems start upstream in application code. Contracts only stick when they give software engineers value in their own workflow (dependency discovery, lineage, breaking-change feedback). Quality then becomes a side effect rather than a favour you beg for.**
(Authors of O'Reilly's *Data Contracts*, founders of Gable. Book promotion and small talk are skipped.)

- **First principle — "culture problems" are really incentive or friction problems.** If doing the right thing were a one-click action, people would do it. Keep drilling into "why won't they behave that way?" until you reach something technology can make easy.
- **Why data teams suffer:** they must model, test and build semantic layers on whatever upstream systems hand them. Fix ownership and accountability upstream and everything downstream gets easier.
- **Why the naive contract fails software engineers:**
  - A contract on a Postgres output ("don't drop this column") helps the DE. But when a DB migration is blocked, the SWE has to map the blocked DB change back to the app code that caused it, which is hard.
  - Most incidents aren't schema changes at all. They are **semantic or content changes**: values change meaning, new categories appear, new data isn't accounted for. Schema-only contracts miss these.
  - So put contracts **in code, where data leaves a system**, so engineers get feedback at PR time rather than at deploy or DB time. That raises the next problem: nobody knows where in a million-line codebase data exits, especially when the code was written years ago by people who have left. **Discovery of contract points** is the real product.
- **Software engineers have data problems too.** Breaking changes, unknown dependencies, unknown lineage across microservices, and outages caused by an API payload change breaking a front end. Migrations and refactors are slow because dependents are traced by hand.
  - *Example:* a PE-acquired edtech company had to integrate 10–20 years of undocumented code. Knowing the producers and consumers and what happens in between could cut a 4-year migration to 1 year.
  - *Example:* M&A integrations reportedly fail about 75% of the time because input/output integration is opaque.
- **Speak their language:**
  - Say "tracing" or "OpenTelemetry," not "lineage."
  - Pitch it as making refactors and migrations safe, not as "care about our data."
  - SWEs *do* care about data, and getting the transactional data model right makes app development easier. They just aren't incentivised to, and feature-factory pressure squeezes it out. Leadership buy-in creates the bandwidth.
- **The data management stack, translated to code (three tiers):**
  - **contracts** (ownership, accountability, expectations),
  - **lineage** (how things connect),
  - **catalog** (which systems produce what data and what it means).
- **Mechanism (how code lineage works):**
  - *In-service:* find **ingress points** (where data enters code), **egress or usage points** (where it leaves or is used) and the **path** between them (enrichment, masking, writing a file…). They do this with a deterministic line-by-line static scan.
  - *Cross-service:* match one service's outputs to another's inputs, using AI/ML over repo context.
  - Meta reportedly spends around $11B on in-house equivalents, mainly for GDPR, where you must trace one user's data everywhere it goes.
- **AI coding risk:**
  - Tools generate huge amounts of undocumented code (sprawl).
  - Their context is limited to one repo, with no graph of cross-system dependencies. So an AI backend change can silently break the front end even when unit and integration tests pass.
  - Freeman's parallel: SWE is having its "data lake moment," trading craft for speed, much as ELT and data lakes traded away modeling discipline.
- **Expand the data team's scope:** about 95% of company data (API payloads, events that feed billing) is unowned yet drives revenue.
  - *Example:* Amtrak station screens showing wrong train times, so people missed trains. The business frames that as an ops issue, but it is a data quality and change-management issue between a producer service and a display team.
  - Owning problems like that ties the data team to $-sized outcomes instead of squishy reporting ROI.
- **Four components of a data contract (from their book):**
  - the data assets,
  - the contract spec,
  - **detection** (knowing what's worth protecting and when it changed),
  - **prevention** (blocking or flagging bad changes).

  **Detection is the hard, high-value part. Don't over-engineer the spec.** For databases, you can pull most metadata from `information_schema` (the book includes a Postgres SQL script to do this). Code and unstructured business context are harder. Metadata is exploding with AI (ontologies; Iceberg uses metadata to optimise lakes; Ole Olesen-Bagneux's *The Meta Grid*).
- **Rules of thumb:**
  - Put contract checks at the producer's egress, enforced in CI on PRs.
  - Cover semantics (allowed values, ranges, meaning), not just schema.
  - Start by inventorying producers and consumers.
  - Make the feedback useful to the producer, not only to the consumer.
- **Relevance for Sam:**
  - Supplier APIs (Viator, Coras, RateHawk) are upstream producers you don't control. Define *consumer-side* contracts as typed models (pydantic/JSON Schema) that validate every payload: required fields, enums, ranges. Log and quarantine violations, and alert on new enum values or shape changes.
  - For Trameter's own services, list which app endpoints feed the booking flow and the data pipelines, and add schema checks in CI for those payloads.

## Joe Reis — Do Data Fundamentals Even Matter Anymore? (https://www.youtube.com/watch?v=6f0K4QnrvXE)
**Fundamentals work like gravity. Building without them ("vibe architecture") works until it suddenly doesn't. Higher abstraction, AI included, increases the need to understand what's underneath.**
(Short solo monologue, conceptual rather than technical.)

- **Two camps (Bill Inmon's framing):**
  - *Technologists* tie their careers to specific tools and vendors.
  - *Fundamentalists* invest in the things that change slowly: architecture principles, data modeling, how systems work.
- **The pushback he answers:** "teams have no time for fundamentals, they're thrown in and pick technologies, so fundamentals are dead." Reis argues this actually shows why fundamentals matter: teams without them build from hearsay.
- **Analogy:** before building a house on a Thai hillside you get a geotechnical assessment, otherwise the monsoon brings a mudslide. Many data architectures are built on vibes and tribal knowledge ("we heard we should do X") with no site survey.
- **Data modeling example:**
  - Many practitioners know Kimball only from chapter 1: star schema, facts and dimensions, by word of mouth. That *may* or may not fit their situation.
  - Engineers who scaled warehouses at LinkedIn and Uber did outgrow classic Kimball at massive volume. But knowing its limits required knowing the body of knowledge well. You can only escape a method responsibly once you understand it.
- **Theory before practice:** like mechanical engineering (statics → dynamics → physics and maths underneath), you need the mental framework before you can apply tools sensibly.
- **Enduring references:** *Fundamentals of Data Engineering*, Kleppmann's *Designing Data-Intensive Applications* (2nd edition recommended), Kimball's *Data Warehouse Toolkit*.
- **AI and the role:**
  - He expects *more* demand for data engineering, because of what must be refactored and built for AI to work.
  - Engineers move up a level of abstraction toward orchestrating systems and designing systems agents can use productively.
  - Per Joel Spolsky's **Law of Leaky Abstractions**, higher abstractions mean you need to understand the internals *more*, because the abstractions leak.
- **Rules of thumb:**
  - Before adopting a pattern (star schema, lakehouse, streaming), state *why* it fits your workload and where its limits are.
  - When an AI or tool hides the complexity, still learn what it's doing. You'll need that knowledge when it fails.
- **Relevance for Sam:** this directly supports the skill's purpose. When Claude proposes architecture for Sam, it should reason from first principles (access patterns, data volumes, change rates, failure modes) and name the limits of each choice, rather than reaching for fashionable patterns.

## Joe Reis hosts Brian Bickell (Cube) — Semantic Layer Deep Dive (https://www.youtube.com/watch?v=kcctcQhlxOw)
**A semantic layer defines metrics and entities once, in code, and rewrites simple "select measure, dimension from entity" queries into the correct, ugly warehouse SQL for every consumer (BI tools, apps, Excel, AI agents), with caching and security in the middle.**
(The speaker is a Cube vendor but tried to stay agnostic. Vendor claims are flagged.)

- **What it is (his "platypus" metaphor, several technologies in one box):**
  - BI-style database connectivity and data modeling,
  - API endpoints,
  - caching and pre-aggregation,
  - security and governance.

  The defining capability is **query rewriting**: receive a query through some API, turn it into an internal representation, then generate dialect-specific SQL for the target database (Postgres, ClickHouse, Snowflake…).
- **The core problem it solves:** the same number everywhere. Without it, metric logic is copied into Tableau calc fields, Power BI models, spreadsheets and app code, and the copies drift. Companies commonly run 3–5 BI tools, sometimes about 10, plus Excel and Google Sheets. Push semantics *left* into one versioned place (YAML/JS models in git). Model once, deliver to many destinations.
- **Mechanism example:** the user writes `SELECT state, city, MEASURE(avg_order_value) FROM sales GROUP BY ...`. The layer expands it into joins, subqueries, cleanup (`UPPER`/`SPLIT` on state) and `quantity*price` logic, all defined ahead of time in the model. The `MEASURE()` function defers the choice of aggregation to the model.
- **Where plain views or SQL get painful (the real complexity):**
  - **ratios at different grains** (average revenue per user needs a subquery plus a distinct; per-member-per-month in insurance),
  - **time intelligence** (YTD, trailing 12 months, period-over-period, custom calendars),
  - dynamic row/column security and multi-tenant filtering,
  - **multi-fact modeling without fan-out, Cartesian products or chasm traps** (joining two fact tables through a shared dimension double-counts unless each is aggregated to a common grain first),
  - finance OLAP functions like XIRR.
- **Materialized views vs pre-aggregations:** the key difference is **aggregate awareness**. Define rollups at several grains (by day, by product…) and the engine automatically routes each query to the smallest rollup that can answer it, with no change to the query. Many warehouse materialized views lack this. Snowflake is adding it, other engines are limited.
- **Concurrency and latency:** embedding analytics in an app brings web expectations (about 200 ms, thousands of QPS). Warehouses are expensive and slow at that. A cache or pre-aggregation layer absorbs it. (Vendor claim: Cube customers handle more than 100k req/s. Cube Store is built on Apache DataFusion in Rust, hybrid memory plus Parquet on object storage, scaling horizontally.)
- **Embedded analytics:** iframing BI tools brings bad licensing, poor performance and poor UX. Teams want BI's data-access and modeling layer exposed as REST/GraphQL APIs, with their own React/Chart.js front end.
- **History:**
  - Business Objects "universes" and Cognos (1990s), business-friendly abstractions over SQL. The "should business users learn SQL?" debate is still going.
  - SSAS multidimensional and tabular cubes (MDX/DAX). The VertiPaq in-memory engine is single-node and memory-bound.
  - LookML/Looker revived the idea for cloud warehouses.
  - 2020s: dbt (MetricFlow via Transform), AtScale, Cube, Snowflake semantic views, Databricks Unity Catalog metric views, Salesforce, Power BI "semantic models."
  - Recent change: **universal** layers spanning many databases and many consumers.
- **The AI tailwind:** text-to-SQL straight against raw schemas is unreliable. An LLM can't know your company's quirks (8 M&As, 3 ERPs, data before date X is broken). Give agents a **semantic SQL** surface (pick governed measures and dimensions) and let the layer translate *deterministically*. This explains why Snowflake (Cortex) and Databricks (Genie) built semantic layers. If humans can't find the right metric definition, AI won't either.
- **Do you need one?**
  - You already have one: every BI tool's calculated fields and datasets are a semantic layer.
  - A *universal* one is worth it when you have complex metrics and ratios, embedded analytics needing caching, multi-tenancy and security, many BI tools, many databases, many user groups, legacy OLAP/finance needs, or AI analytics.
  - A 50-person company doing BI for the first time doesn't need one. A warehouse plus a modern BI tool is enough.
  - **Open problem:** expressiveness differs between layers, and porting models between them is lossy.
- **Relevance for Sam:**
  - For Trameter dashboards or an AI analytics agent, define core metrics once in versioned code: bookings, conversion, GMV, supplier match rate, coverage per country. This can be as simple as a Python/SQL metrics module or dbt/Cube models.
  - Have agents query those governed metrics rather than raw tables.
  - Watch for fan-out when joining two fact tables (bookings and searches) through a shared dimension. Aggregate each to the common grain before joining.

## Joe Reis hosts Robert S. Seiner — The Godfather of Data Governance Breaks It All Down (https://www.youtube.com/watch?v=TbEHGsbDmT4)
**Governance is already happening informally in every organisation. The non-invasive approach formalises the accountability people already have (definers, producers, users of data) instead of handing them new duties. It only works when it is tied to business value and paired with behaviour change and data fluency.**
(Mostly organisational. Biography and small talk are skipped.)

- **Definition:** data governance is *the execution and enforcement of authority over the management of data*. A friend's reframing: it is really "people governance," because you govern behaviour around data.
- **Three approaches:**
  - *Command and control*: top-down, new roles and duties assigned. Breeds resistance, and gives people responsibility without authority.
  - *"If you build it they will come"*: define a programme and roles and hope people gravitate to it. Fails.
  - **Non-invasive** (Seiner's trademarked approach): recognise people for the data relationships they already have and **formalise accountability** without adding work on top. Put it into existing workflows.
- **Premise:** an organisation that has survived for years is already governing data informally. People who use sensitive data are expected to protect it. People who produce data (for example entering diagnosis codes) are expected to enter it correctly. The job is to move from *informal to formal* and to identify the actual decision makers. Most organisations don't even know who those people are.
- **Three accountability roles around data:**
  - *Defining* it: new data brought in must come with definitions and modeling.
  - *Producing* it: enter it correctly.
  - *Using* it: use it only as allowed.
- **"Data Catalyst Cubed" (new book): governance × change management × data fluency.**
  - The three multiply, so if any one is zero the result is zero.
  - Successful programmes were the ones where people understood their steward roles.
  - Fluency is more than literacy. A 30-minute training video plus a checkbox is not fluency.
  - Even pristine, well-documented data gets misused by people who don't know how to use it.
- **Governance as a catalyst, not "enablement":** rebranding as "enablement" is too passive. Compliance-checkbox governance (SOC 2 badges, doing the legal minimum) is reactive. Programmes are reported as "failing" mainly because they aren't **linked to measurable business value** in BI, analytics or AI. You must be able to **demonstrate in an auditable way, on demand,** that the rules are being followed.
- **AI-era risks:**
  - Copilot-style tools used as a "BI tool" over messy SharePoint and Excel versions give answers that don't match the system of record.
  - Uploading sensitive data (HIPAA or financial) to LLM providers.
  - Agents generating synthetic data, which causes exponential data sprawl.
  - Autonomous agents with credentials, a security nightmare (Reis's OpenClaw example).
  - **Data governance and AI governance are distinct but both necessary.** Data governance covers behaviour around defining, producing and using data. AI governance covers model behaviour and outputs.
  - Keep **humans in the loop** for consequential decisions such as accepting or rejecting claims.
- **Where AI helps governance:** automated cataloguing and documentation, data-quality detection, and discovering who uses which data.
- **Specs and code:** an old pain point was poor business requirements. Now good specifications turn into code very quickly, which makes clear specs (and metadata) *more* valuable.
- **Rules of thumb:**
  - Name an owner/steward for each critical dataset, based on who already handles it.
  - Require a definition for any new data entering the platform.
  - Restrict and audit what gets sent to external LLMs.
  - Tie every governance action to a business outcome.
- **Relevance for Sam:**
  - For Trameter's master location dataset and supplier data, write down who defines, produces and consumes each table, plus allowed uses. Licensing matters here: GeoNames is CC-BY and OSM is ODbL with share-alike obligations.
  - Never paste PII or booking customer data into external LLM prompts without a policy.
  - Keep a human approval step before AI-matched records are published to production.

## Joe Reis hosts Sami Hero (Ellie.ai) — The Return of Conceptual Data Modeling (https://www.youtube.com/watch?v=vZIu4ZwEhck)
**Model conceptual → logical → physical. The conceptual layer (business entities and named relationships, validated with business people) is the durable foundation that warehouses, ontologies and AI agents all reuse. Skipping it produces table sprawl, conflicting metrics and confidently wrong AI answers.**

- **Unchanged problems at 100,000x scale:** volumes went from 1 TB being big to PBs, yet **time-to-insight has stayed roughly constant**. The bottleneck is human: asking the right question with the right context, and agreeing on definitions. Large banks spend weeks defining "customer" (account holder? policy holder? payer?), so an LLM can't infer it from raw data.
- **AI on raw warehouses:** executives point Claude or Copilot at the warehouse and get fast but often wrong answers.
  - Rough answers are fine for "leads last week," but unacceptable for AML or regulatory work.
  - Rather than 400-line prompts, *model the meaning* and expose it to agents through MCP or APIs: what gross margin means, what "active customer" means, what cleansing rules built this dataset, why the model exists.
  - Legacy documentation (Confluence) is often stale. Humans notice the mismatch, while agents brute-force and infer plausible nonsense.
  - (Reis notes Anthropic's own write-up on agentic analytics made a case for dimensional modeling.)
- **Conceptual model:**
  - Entities plus *named* relationships at business level ("customer buys product," "product is used by customer"), without deep detail.
  - A VP of marketing can read and validate it.
  - It is agnostic about the downstream method (dimensional, Data Vault, OBT) and the platform.
  - Next, add detail in the logical model, then make platform-specific optimisation choices in the physical model (Redshift, Snowflake, Databricks and BigQuery each favour different designs).
  - Analogies: John Giles's "data town plan" (zoning, then blueprint, then construction). A house's wiring and HVAC drawings are the ERD, and you shouldn't *start* there.
- **One model, many uses:** conceptual models can be exported to OWL for ontology and knowledge-graph tools, and also feed warehouse modeling. Ontologies and taxonomies came back because compute now makes them useful. The algorithms are about 20 years old. (Example: LexisNexis public-records graph linking a hunting licence → cabin → LLC.)
- **Anti-patterns:**
  - **One dbt model per ticket**: a global company with more than 15,000 Snowflake tables, each a one-off big table built for a request.
  - A fashion group with **28,000 dashboards**. More isn't better: people use vanilla and pistachio out of 42 flavours.
  - **Fixing data in the BI tool** (Tableau reclassifications and calcs). Hero admits this was "shifting right hard." Logic gets trapped in the front end and migrations become painful. It also overloaded the Tableau server, and 75-minute Redshift queries were "solved" with a bigger cluster, a costly fix.
  - **ERP "reporting" extracts instead of atomic data.** One feed was a pre-aggregated report from 1998 whose logic nobody knew, so it couldn't be reconciled with finance. Always source atomic data, or document the aggregation.
- **AI-assisted modeling needs a human in the loop:** AI produced an almost-perfect movie domain model but reversed a cardinality (many movies per trailer instead of many trailers per movie). Check cardinalities yourself.
- **Process:**
  - Model iteratively and embedded with the business (agile, collaborative tools, change notifications), not in a two-year ivory-tower effort.
  - A conceptual model that changes every five minutes signals a strategy problem. A model *built to tolerate* change is good.
  - Use proven patterns (Party/Role abstraction; Len Silverston's data model pattern books; John Giles).
- **Gaps and opportunities:**
  - Modeling tools are good at facts and dimensions but weak at **metrics**. Document each metric's calculation, source fields and rules (CAC, magic number) so AI answers "what is it and how is it calculated," not just "0.7".
  - Gartner's "universal semantic model" is essentially a well-documented conceptual model.
  - Tag entities and attributes with AI-consumption metadata. This doubles as a "poor man's" catalogue and glossary.
  - Goal: the **same answer to the same question**, as a report always gave. LLM non-determinism undermines that.
- **Selling the value:** quantify rework avoided and time-to-outcome. A law firm (Clyde & Co) measured this. French engineering schools still teach the Merise method (separating business from technology), which shows in practice.
- **Controversial view:** there is no shortcut. Buying a semantic-layer tool doesn't do the definitional work, and "no decision" to model is itself a decision that bites later.
- **Relevance for Sam:**
  - Before building more tables for Trameter's location and supplier data, sketch a conceptual model and validate it with the product team. Example entities: Place, AdminArea, TransitStop, Supplier, SupplierProduct, Booking. Example relationships: Place *is within* AdminArea; SupplierProduct *is offered at* Place; Booking *reserves* SupplierProduct.
  - Define key terms precisely ("destination," "city," "active product").
  - Pull atomic supplier records, not pre-aggregated exports.
  - Document metric formulas so AI agents answer consistently.

## Joe Reis (interviewed by Ramona Truta) — Why Data Modeling Matters MORE in the Age of AI: Mixed Model Arts (https://www.youtube.com/watch?v=zBJuD_EIJi4)
**Data modeling's building blocks (entities, attributes, relationships, time, grain, meaning) don't change. Modern practice mixes styles from five camps, choosing what fits the situation, and AI agents make explicit models *more* necessary, both for the data they consume and for the data they now produce.**
(Mostly a conversation about writing the book, careers and AI anxiety. The modeling substance is extracted here.)

- **Focus on what doesn't change:** with AI moving fast, the book (*Mixed Model Arts*, book 1 of 2) concentrates on durable building blocks:
  - **entities, attributes, relationships**,
  - **time**,
  - **grain**,
  - **meaning/semantics**,
  - extended to unstructured data as well as structured.

  "Get these right and everything else falls into place; get them wrong and you suffer." He sees William Kent's *Data and Reality* (1978) as the last attempt to define what data modeling fundamentally is. Most books teach a single approach (relational, Kimball, Data Vault). Data Vault was the last big new idea.
- **The five camps of data modeling:**
  1. relational,
  2. analytics (dimensional and similar),
  3. applications/software (including NoSQL and streaming),
  4. ML/AI,
  5. knowledge (graphs, ontologies, semantics).

  Historically they rarely talked to each other. The data world even misused graph databases while the knowledge world worked separately. AI moves across all five easily, which is forcing them together.
- **The MMA metaphor:** asking which martial art is best only makes sense with stated rules and constraints. A pure boxer gets taken to the ground. A pure grappler gets kept at distance and hit. Data modeling has its own tribal, dogmatic schools (he cites some Northern European ones). The working approach is to know several styles and pick what fits the workload. **Relational is still extremely important.**
- **Scale shaped the styles:** Google couldn't use relational at its scale and went protobuf-style. Truta: each technique reflects what compute allowed at the time. Context decides which style is right.
- **Expert confirmation:** Chris Taylor (Google Spanner) told Reis that data modeling matters more now, because agents need the context a good model provides.
- **Agents as producers (Truta):** agents don't just consume data. They now create databases and data, and many hyperscaler databases are created and used only by agents. So **model the outputs/"exhaust" too**: decide up front what will be needed, who consumes it and how it should be structured. Otherwise you create the next mess. (Reis's insight for book 2: machines are modeling data for humans and for machines.)
- **Unintentional modeling:** everyone models data. Most do it *unintentionally* and then live with the consequences. The book's reception (20k+ Substack subscribers) suggests practitioners feel that gap.
- **AI as an editor and sparring partner:** use AI to find inconsistencies (a number on page 7 contradicting page 300), to suggest literature you missed, and to argue with your own view. Ask models what they would want from a data model.
- **Truta on practice:** without tools you can build observability and lineage directly into SQL and the schema (adding audit and lineage columns). That is modeling thinking, which transfers across tools. Value skills over years spent with a particular tool.
- **Rules of thumb:**
  - For every table, state the entity, the grain (one row = ?), the time semantics (event time vs load time, validity ranges) and the meaning of each attribute.
  - Choose the modeling style per use case: normalised for OLTP and booking, dimensional or OBT for analytics, graph or ontology for relationship-heavy knowledge, document for flexible payloads.
  - Design agent outputs deliberately.
- **Relevance for Sam:**
  - Location data mixes the camps. Relational/dimensional for places and admin hierarchies. Graph-like relationships for transit networks (GTFS stops, routes, trips). Document style for raw supplier payloads. A knowledge or ontology layer for place types and aliases.
  - Fix grain explicitly. Is a "place" a GeoNames feature, a supplier destination, or a merged canonical entity? Keep each at its own grain with mapping tables between them.
  - Track time: snapshot date and source version for every load.

## Joe Reis — The Great Data Engineering Reset (Small Data SF 2025, MotherDuck) (https://www.youtube.com/watch?v=MucdAT69M08)
**AI pressed the reset button and everyone is back at the starting line. The fundamentals (lifecycle, modeling, quality, security, architecture) still hold, but architectures are moving toward hybrid deterministic and probabilistic systems, continuous feedback loops, knowledge orientation and agent-scale workloads.**

- **Before the reset, two worlds:**
  - *Enterprise land* (about 99% of companies): data as a cost centre, playing defence (control, compliance, reporting), dashboard-driven, always justifying value. CDO average tenure is about 18 months. In the zero-interest-rate era, cheap money hid the inefficiency of the expensive "modern data stack" until higher rates made it painful.
  - *Product land*: data as a revenue driver, playing offence, with data products, **shift-left** (data contracts, software-engineering practices) and product thinking. Many companies straddle both (finance in enterprise land, product teams in product land).
- **The old arguments are obsolete as the central question:** batch ELT vs ETL, Inmon vs Kimball vs OBT, lakehouse vs warehouse. These were optimisations *within* a deterministic, analytics-first, dashboard-centred system.
- **The reset:**
  - ChatGPT turned AI into the default interface. Vendors bolted LLM wrappers onto products that weren't AI-native.
  - Adoption signals conflict. The MIT study says 95% of GenAI projects fail. Another found agents used in only 2.5% of freelancer tasks. "Shadow AI" (people bringing their own tools) is common.
  - The biggest wins he sees come from *non-technical* teams redesigning their workflows.
  - Experts feel like beginners again. Red Queen effect: running faster to stay in place.
- **What doesn't change:**
  - the lifecycle (ingest, store, transform, serve),
  - data modeling (more important than before), data quality,
  - undercurrents: security and architecture never go away and matter more,
  - the four pillars of people, process, technology and data,
  - **Conway's law** (systems mirror the communication structure of the org that builds them).
- **What changes (traits of the "postmodern" or "live" data stack: connected, contextual, continuously evolving):**
  - **Hybrid deterministic plus probabilistic systems.** LLMs embedded in workflows next to rule-based logic.
  - **Agent-scale load.** Example from Berkeley's agentic-architectures paper: databases may face **20–50x more queries**, mostly *semantically similar*. Expect an "agentic query planner" or caching layer in front of the database to absorb and deduplicate them, so it doesn't fall over.
  - **Semantic models built for agents**, discoverable and updated in real time.
  - **Continuous compute and streaming.** Agents won't wait for a 24-hour batch, so event-driven feedback loops matter.
  - **AI-native governance and observability.** Self-healing systems, but regulators (especially in the EU) still require **audit trails**.
  - **Knowledge orientation.** Ontologies, taxonomies and knowledge graphs. Tim Berners-Lee's 2001 semantic web of RDF plus agents is becoming practical. "Knowledge lakehouse."
  - **Multimodal data** (images, text, audio, video) inside continuous workflows.
  - **From actionable insights to actions.** Autonomous actions partly replace dashboards.
- **Orgs and process:**
  - "Work charts vs org charts": big tech is cutting middle management.
  - CEOs and PMs vibe-code prototypes. "Show, don't tell" shortens learning cycles, so mistakes come faster. Make sure they aren't catastrophic ones.
- **The DE role:** becomes system designer and orchestrator of agents. Skills to build: master the fundamentals, **systems thinking**, a **product mindset** (what do users actually need), craftsmanship and expertise as differentiators. Advice: build new workflows yourself rather than waiting.
- **Rules of thumb:**
  - Expect machine consumers. Put caching and semantic deduplication in front of expensive stores.
  - Keep an audit trail for every automated action.
  - Pair LLM steps with deterministic validation.
  - Design pipelines to support feedback loops (outputs feed back as inputs), not only one-way A→B.
- **Relevance for Sam:**
  - Trameter's AI features (Drive Button injecting bookable trip buttons, AI research tools) are product-land, hybrid deterministic/probabilistic systems. Wrap LLM outputs with deterministic checks (does this place/product id exist, is it bookable, is the price fresh).
  - Cache repeated, similar lookups (place resolution, supplier availability) in front of supplier APIs and the master dataset.
  - Log every automated action with its inputs for audit.
  - Most of his heavy data work stays sensibly batch on DuckDB/Parquet ("small data"). Add streaming only where an agent or user truly needs freshness, such as availability and pricing.

