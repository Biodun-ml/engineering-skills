# search-2 — cross-cutting principles

1. **Test the deepest assumption first.** Before improving a ranker, check that the surface matters at all: log per-item impressions and clicks and run a "hide it" holdout. Booking spent 6 months tuning hotel ranking on a homepage module nobody clicked, and hiding it changed nothing. (Vermeer, Booking experiments)
2. **Product improvement is learning, not optimisation.** Write a hypothesis (observed problem → cause → change → expected behaviour → metric) before building. It lets you reject bad variants and come up with better ones without testing. (Vermeer)
3. **Never switch off the control group.** A recommender or LTR model without a holdout can't prove its value. Pre-register the analysis and the power-calculated run length, and stick to it. (Vermeer)
4. **Analysis is subjective, so cross-check.** Independent pipelines (Booking runs two for every experiment), independent analysts and independent judgment sources (clicks vs humans) catch bugs and forking-path bias. (Vermeer; LTR panel)
5. **Always compare against a tuned BM25.** k1 (TF saturation) and b (length normalisation) are corpus-specific. Tuning just these moved BrowseComp-Plus retrieval about 30 points. Weak baselines make new methods look falsely good. (Turnbull & Bergum; Thakur)
6. **BM25 basics:** TF with saturation × IDF × length normalisation, per field, combined as a weighted sum. Short fields (names, URLs) want different k1/b than body text. (Thakur)
7. **The agent is the new searcher.** Agents send long (about 23 terms), keyword- and phrase-heavy, web-syntax queries and reformulate over about 25 calls. A simple, interpretable retriever plus an agent that reformulates beats one-shot retrieval. Dense models trained on question-style data underperform on agent keyword queries unless those queries are rewritten. (Turnbull & Bergum)
8. **Match the tools models were trained on.** Use one well-named search tool or a grep/file-system-like interface (virtual, translated to a real ranked engine). Renaming a tool changed accuracy. Use sub-agents for search and code execution to keep context clean. Precision/recall *is* context engineering. (Turnbull & Bergum)
9. **Build your own index for the questions you care about.** Federated silo search is rarely relevance-optimised. Scored top-k also manages context better than unscored grep. (Turnbull & Bergum)
10. **LTR learns labels, not features.** Value comes from features that separate apparently-relevant from truly-relevant docs. Start with user-interest features before seller- or business-interest ones. More features need deeper judgments per query. (LTR panel)
11. **A query is keywords in a context** (user segment, platform, time, passport). Personalise by enriching the query or adding intersection-style features, but over-segmenting destroys click-model training data. Wikimedia already drops about 60% of logs for lack of repetition and needed about 30–40M observations before NDCG plateaued. (LTR panel)
12. **Route first, then rank.** Choosing the vertical or collection is a classification problem upstream of LTR, because LTR scores aren't comparable across queries. Split models by segment when feature semantics differ (platforms, departments); pool when signal transfers. (LTR panel; Yan's "alignment tax")
13. **Drift is the fatal flaw of LTR.** Intent shifts (Blade Runner 1982 → 2049). Inject refreshed popularity or recency features, treat time as context, retrain every 3–6 months with recency weighting, and filter bot traffic. (LTR panel)
14. **Point-in-time-correct features and identical online transforms.** Store features as time-stamped change logs. Ship model + stateful transforms + feature list as one unit. Use a feature store with offline and online bindings and owners. Monitor input distributions, not just outputs. (Ottens & Falsina, Booking ML)
15. **Out-of-core on limited hardware.** Sample to disk, not RAM, and stream over it a few passes (Vowpal Wabbit style). Put long-running streaming jobs on non-preemptible infrastructure. (Ottens & Falsina)
16. **Semantic IDs beat hash IDs for cold start.** Cluster frozen content embeddings (k-means → about 1k IDs) and learn trainable embeddings per cluster, so new items inherit behaviour (Kuaishou: +3.6% cold-start coverage). (Yan)
17. **LLM labelling cascade, then distil.** Expert seed labels → strong LLM → fine-tuned mid model → distilled tiny classifier for real-time use. Optimise the metric that matches the harm. Filtering bad recs *raised* applications 4% and cut unsubscribes 5% (Indeed): quality beats quantity. Use conventional data first and LLMs to augment (Spotify query recs +9% exploratory queries). (Yan)
18. **Unified models and embeddings** with a shared input schema and smart imputation (Netflix UniCoRn), and quality baked into ANN retrieval by concatenating a quality vector and padding the query vector with a constant (Etsy: +2.6% conversion). Expect an alignment tax that can force 2–3 models. (Yan)
19. **Cross-encoder rerankers may be optional below big-tech scale.** LLM judges and agentic loops can replace them (Bergum's opinion). Auto-research agents can write ranking code, but only with a holdout set to stop `if query == X` overfitting. (Turnbull & Bergum)

## Lukas Vermeer (Booking.com), DataFramed podcast with Hugo Bowne-Anderson — #23 Online Experiments at Booking.com (https://www.youtube.com/watch?v=4e_rBd96iGA)
**Improving a product (including ranking and recommendations) is a learning problem, not an optimisation problem: write down the hypothesis and the stack of assumptions under it, and test the deepest, cheapest assumption first.**

- Mental models / first principles
  - Booking (2018: ~1.7M properties, 1,500–2,000 people running experiments) uses three pillars: exploratory analytics (where to look), qualitative user research (why users struggle, e.g. eye tracking in an in-house lab) and quantitative experiments (did the fix actually work). Experiments come at the *end* to validate a hypothesis that came from the other two.
  - Vermeer got into this through recommender work at a consultancy: clients kept asking to "switch off the control group" so everyone gets the best recommendations. That misses the point. Without a holdout you can't know whether the recommender adds any value.
  - "Website optimisation" (41 shades of blue, change the button colour and watch the magic number) assumes a closed, measurable space. Product work is not like that. Long-term value and satisfaction are hard to measure, and the real gains usually come from serving an unmet need (e.g. "can I bring my pet?"), which no button colour fixes.
  - Hypothesis framing changes what humans can reason about. "Users miss this button (eye-tracking evidence) because white text on yellow has low contrast, so a darker background should raise hovers, clicks and finding what they need." Once the intent is written down, you can reject bad variants without testing them (light green doesn't raise contrast) and come up with alternatives (bigger button, different font colour).
  - For pure ranking and recommendations, prediction without understanding is often fine: you don't need to know *why* someone likes a hotel type. You still need theory to frame the question, read the data and interpret the answer.
  - "Philosophers and telescope builders": data science needs people who reason about causality and blind spots, and people who build telemetry (instrumentation). The heliocentric theory waited about 2,000 years for a strong enough telescope. Missing data and missing telemetry are hugely undervalued.

- Mechanisms
  - Shared experimentation infrastructure is exposed as an API, not point-and-click, because every cross-functional team has developers. The effort goes into making the *decision process* self-service: guidance on method, power calculations for duration, and enforcing that teams actually run for the planned duration (no early peeking or stopping).
  - Two fully independent data pipelines, built by different people who rarely read each other's code, measure the same experiment metrics. Results are cross-checked for every experiment. The premise is that bugs and edge cases *will* exist, and you can't find them without something independent to compare against.
  - Continuous delivery lets a change go from decided to live in an experiment within about an hour.

- Decisions & trade-offs
  - Pre-registration: write down the analysis before you see data. It avoids the "garden of forking paths". This works for confirmatory tests but is harder for exploratory analysis.
  - "Bootstrapping the analysis": have several independent analysts (ideally with diverse backgrounds) answer the same important question and compare. It is expensive, so reserve it for big business decisions.
  - A/B tests are not only for small changes. What matters is how important the assumption being tested is, not how big the code change is.

- Rules of thumb
  - Before tuning a ranker, list the stack of assumptions: ranking quality matters → the module matters → anything on this page matters → users come here to browse rather than to type a search. Test the bottom layers first.
  - Instrument intermediate steps (impressions, clicks per item), not only the final conversion. Without clicks you can't tell whether a widget is used at all.
  - Fundamental assumptions tend to produce larger effects, so they are also easier to detect statistically.
  - "Fail fast" is a misnomer. The goal is learning; failures are a by-product.

- Failure modes & gotchas
  - **Six months wasted on ranking:** Vermeer's team spent about 6 months trying academic ranking variants (diversity, user origin, logged-in history) for the 5 hotels shown under each recommended destination on the homepage. The success metric was booked-and-stayed. When a paper needed per-hotel click counts, they started logging clicks and found almost nobody clicked those hotels. Hiding them entirely (a CSS change, live within an hour) had zero measurable impact. The whole ranking effort had been optimising an unused surface.
  - "Many analysts, one data set" (about 30 teams, same soccer referee/red-card data, same yes/no bias question) gave roughly a 60/40–70/30 split in conclusions. After discussing, few teams changed their answer. Analysis is subjective, and confirmation bias is strong.
  - Teams of like-minded analysts give a biased "bootstrap". Diversity improves exploratory and innovative work. It only hurts in factory-like repetitive work.

- Relevance to Sam / Trameter
  - Before building fancy ranking for Trameter destination or activity carousels (Viator/RateHawk), or for "where can I go" suggestions, log impressions and clicks per slot and run a "hide it" holdout. That shows whether the surface matters at all.
  - For Drive Button, measure button impressions → clicks → bookings separately. Keep a control (articles without injected buttons) instead of switching it off for "everyone gets the best".
  - On a small team, a cheap version of the dual pipeline is to reconcile key metrics from two sources (e.g. supplier booking reports vs your own event logs).
  - Write each experiment as hypothesis → expected behaviour change → metric, fixed before launch, and compute run length with a power calculation up front.

## Doug Turnbull & Jo Kristian Bergum (Hornet, ex-Vespa) — Why 2026 is The Year of Agentic Search (https://www.youtube.com/watch?v=h370222tnAQ)
**Agentic search means the agent writes the queries. That raises the ceiling of simple retrievers (BM25, grep), makes old-style query understanding largely solved, and turns precision/recall into the core of context engineering. Harnesses win by matching the tools models were trained on.**

- Mental models / first principles
  - Definition (Bergum): agentic search = the *agent* formulates the query; "the agent is the new user". It is not tied to grep or vectors. "RAG is dead" came from people equating RAG with "embed everything + vector similarity". Claude Code's grep-over-files is still retrieval.
  - Two camps: people building rich harnesses that gather context from existing tools, and people building retrieval for cases where the agent lacks knowledge (deep research, multi-vector, etc.).
  - Search relevance used to be funded only by a few firms (Google, Yahoo, e-commerce, Shopify). Now every team building agents needs it, because retrieval is central to agent success.
  - Precision/recall = context engineering. Recall is getting all the needed information. Precision is getting only that. Stuffing everything in gives perfect recall but performs badly, so precision matters again.
  - Indexing vs scanning is only a performance trade-off. You can compute BM25 by scanning with a top-k heap. An index is a data structure for repeated queries under a time budget at scale.

- Mechanisms / evidence
  - BrowseComp-Plus benchmark: about 830 hard pub-quiz questions, a fixed corpus of about 105k web pages, and a minimal harness with one `search(string)` tool returning 5 hits truncated to 512 tokens. GPT-5 trajectories average about 25 search calls per question. Queries average about 23 terms (vs roughly 2 in old AOL logs). There are many phrase queries, and about 5% of about 20k queries use `site:`, plus year ranges, all without being told the syntax exists. Agents query like web-search power users because of their training data. They reformulate based on results: broad first, then narrow, backtrack, retry.
  - Glasgow follow-up ("revisiting text ranking for deep research"): agents send keyword-style queries, so dense retrievers trained on question-style data (MS MARCO, Natural Questions) underperform under a plain search tool. Rewriting keyword queries into natural questions helped the embedding retriever.
  - BM25 baseline in BrowseComp-Plus used Anserini defaults, which are tuned for uniform short passages. On long, variable web documents, tuning the two parameters (k1, b) gave about **30 points** better retrieval. It is easy to build a weak BM25 baseline that makes your new method look good.
  - Paper: put the documents in a flat file tree and let a coding agent use grep/head/tail. It scored very well, because it could read whole documents instead of 5 truncated chunks.
  - Multi-shot beats single-shot. A "dumb" retriever plus an agent that reformulates beats one-shot retrieval. "The semantic layer becomes the LLM, not the embedding model."
  - Search as a sub-agent: the main agent calls a search tool that is itself an agent with its own context. The mess stays encapsulated, and you can use a smaller, specialised model (examples mentioned: Chroma's SID-1, Glean's model).
  - Virtual file systems: make data *look* like a file system or grep target (an abstraction over a DB or search engine). Intercept grep calls (often regex or `a|b|c` alternation) and translate them into a real relevance-ranked query. No need to dump the DB to disk.
  - Code execution to save context: let the agent write code that chains tool outputs instead of passing big payloads through the context window. Cloudflare's "two tools: search + execute" over an OpenAPI spec runs in V8 isolates. Related: recursive language models, where context lives in a REPL variable.

- Decisions & trade-offs
  - One web-search-like tool vs unbundled primitives (lexical tool, embedding tool with "ask a natural question" instructions, reranker or LLM judge). Models are post-trained on specific tool names and layouts (e.g. gpt-oss), so lots of novel tools confuse them. **Renaming a tool alone measurably changed task accuracy** (Hornet). Follow what frontier labs train for (grep, bash, web search), and your harness improves as models improve.
  - Native index vs federated MCP calls to each silo: both hosts favour building your own index aimed at the questions you need answered. Silo search is usually plain DB lookup, not relevance-optimised. Agents cope with dumb tools, but a real index gives incremental gains.
  - Keyword search is more interpretable *for the agent*: it can see why "dress shoes" returned dresses and rewrite. Predictable tools beat opaque ones for agent reasoning.
  - Grep has no scoring, so it can flood the context. Scored top-k is valuable for context management.
  - Bergum's hot take: standalone cross-encoder rerankers are a "dead end" except at eBay/Yahoo scale where latency and cost dominate. At enterprise or small scale, let the LLM or agent judge. (note: this is opinion. Measure on your own eval.)
  - Latency bias: search veterans default to single-query, sub-millisecond thinking. Many domains can afford 5 seconds of multiple agent queries for a much better answer. LLM-based query understanding (spellcheck etc.), once a multi-month project, can now run as a nightly batch with a small self-hosted model. The hard part is the remaining 10–20% domain-specific knowledge.

- Rules of thumb
  - Evals are still king. Measure tool names, prompts and feedback styles. Turnbull's untested hunch: "made the user happy/unhappy" feedback may work better than "3/5 relevant".
  - Auto-research on ranking: give a coding agent search primitives, a metric, a train set and a **holdout**, and let it write ranking code. Watch for overfitting: it will produce `if query == X: return [...]` hacks unless the harness checks against holdout data. The output is code, not a model needing inference hardware. LLM-judged relevance makes this usable where user signals are sparse.
  - Personalisation is easier via in-context learning (pass preferences into the prompt or API call).

- Failure modes & gotchas
  - One or two bad results can derail long agent trajectories, and exhaustive multi-query search pushes filtering onto the agent. Mitigate with search sub-agents and keep precision behind the tool.
  - Prompt injection from retrieved content.
  - "Inverted SEO": consumer agents will keep returning to providers whose search satisfies them (fast, good recall and relaxation, API-first). Brittle or slow providers lose agent traffic. Travel was named as the typical complex personal-agent task. Pricing and payment for agent traffic is still unresolved.

- Relevance to Sam / Trameter
  - Trameter should expose an agent-friendly, API-first search over destinations and Viator/RateHawk inventory: a single well-named `search` tool with forgiving syntax (phrases, filters), fast and with scored top-k. Agent-friendliness may become a distribution channel ("inverted SEO").
  - For the ~6M-location fuzzy search on 16GB, tune BM25 k1/b on Trameter's own field lengths (place names are short, descriptions long) before reaching for embeddings. Always compare against a *tuned* BM25.
  - For Drive Button and "where can I go", a search sub-agent that queries a lexical index and judges results with an LLM is a cheap path. Hold out labelled article→trip pairs if you let an agent optimise the ranking code.

## Panel: Doug Turnbull, Erik Bernhardson (Wikimedia), René Kriegler, Xun Wang (Snag) and others — Learning to Rank Panel, Haystack 2019 (https://www.youtube.com/watch?v=0nE4fLtJjzg)
**Practitioners on LTR in production: the model only learns as well as its features and judgments allow. The hard problems are training data (click models, sparsity), feature design, segmentation, drift and freshness, not the algorithm.** (note: speaker names are partly inferred from the transcript; Wikimedia/Wikipedia and Snag were explicitly mentioned.)

- Mental models / first principles
  - LTR doesn't create features for you. It creates *labels* that tie past user opinions to features you already have. No features, no learning.
  - A random document is ~99.999% irrelevant to a query. Much of LTR's job is demoting things that *look* relevant (false positives), so you need features that tell "apparently relevant" apart from "actually relevant".
  - A "query" is keywords *in a context* (user, segment, time, platform). Personalisation can be framed as making the query ID richer (e.g. "star trek" for user cluster 3). Segment attributes then act as query-only features, like query length or term document frequency. The more you subdivide, the less training data each query-context has.
  - LTR scores are only for ordering. They are not calibrated or comparable across queries. "Which collection or vertical should answer this query" (case law vs practical guidance at LexisNexis) is a *classification* problem upstream of ranking, not an LTR problem.
  - Much of "ML in search" isn't LTR at all. It's classifiers and enrichment feeding the index or query: intent classification (blender: kitchen vs audio), so you filter or boost a category, plus quality scores and topic labels. These can be used directly in the query DSL or as LTR features.

- Mechanisms
  - Embeddings in LTR: (a) random-projection trees or forests (an LSH family). Each random hyperplane splits points left/right. A doc's path becomes a bit string (e.g. 101). Query-to-doc closeness is then a fast prefix match on those codes in Lucene, and agreement across many trees in a forest approximates vector closeness. (b) Inject query–doc similarity as one feature. (c) Offline at one company, all 300 query dims plus 300 doc dims of word2vec were used as raw features alongside the others and worked well, but feeding them efficiently through the ES LTR plugin was unsolved. (d) Convert to the TF-Ranking plugin's expected format. Prefer dimensionality reduction (LSH or similarity) to fewer features.
  - Training data from click models needs repeated queries with the same intent. Wikimedia discards about 60% of click logs for lack of repetition. Personalisation splits the data further and kills the long tail.
  - Wikimedia's data curve: NDCG kept improving meaningfully with more data until about 30–40 million observations on English Wikipedia.
  - Personalisation sketch: build an anonymous user profile (favourite brands or categories) and use a feature like *intersection of the user's top brands with the doc's brand*, or a vector of preference probabilities. Don't use raw high-cardinality categoricals.
  - Freshness: Wikipedia injects regularly refreshed page popularity as an indexed feature, so the model learns "popularity matters" and new hits (a movie of a book) rise as clicks accrue. Ideas: per-doc match or publication counts in the last 24h, and query-frequency spikes over time as features.

- Decisions & trade-offs
  - One model vs many ("model pooling"): separate models per department, platform or content type when segments have different feature semantics. Price sensitivity differs for socks vs cars. Snag's desktop, mobile web and app UIs differ so much that pooling introduced bias, so they split them. Pool when segments share signal or you're unsure, and start pooled. A single uber-LambdaMART needs features that describe each item's segment so its trees can branch, plus training data across all of them. Separate models are smaller and more specific but bring more ops overhead, and you need a router that classifies the query (fashion vs furniture), which can itself make errors.
  - Where possible, unify heterogeneous content into one data model for a single search experience.
  - Human judgments vs click-model judgments: academic results say blends are strongest (rebalance by up- or down-sampling). Keep independent judgment sources to cross-check analytics bugs. Humans catch *missing or buried* content that clicks can't reveal: users are "happy" with mediocre results because they never see the great buried one.
  - Gains vs baseline depend on the incumbent: a hand-tuned ranker refined over 8 years took a long time for LTR even to match.

- Rules of thumb
  - Start with user-interest features. Add seller- or business-interest features later and carefully. One client added all features at once, including some good for the seller but bad for the user, and results got worse.
  - More features need deeper judgments per query (curse of dimensionality) or you overfit.
  - Retrain periodically (every 3–6 months, depending on how dynamic the inventory is) and weight recent observations more.
  - Bot defence: one obnoxious user barely moves things, but the same query 500 times a minute is a bot. Filter such traffic out of training logs. Bias in training data mostly needs human intervention and balancing.

- Failure modes & gotchas
  - LTR can only optimise toward its training data. When intent shifts (Blade Runner 1982 → Blade Runner 2049, "Robert Mueller" news), historical labels are wrong. This "garbage in, garbage out" drift is called the fatal flaw of most LTR systems. Mitigate with time-aware or popularity features, and treat time as part of the query.
  - Over-segmenting personalisation throws away long-tail data.

- Relevance to Sam / Trameter
  - Trameter has little click data, so: start with BM25 plus hand features, collect clicks, use LLM or human judgments, and only then LTR. Expect needing large data volumes before NDCG plateaus.
  - Use a query→vertical classifier (destination vs activity vs hotel vs visa info) before ranking, rather than one LTR model scoring mixed types.
  - Popularity and seasonality features (refreshed bookings or clicks per activity or destination) handle freshness, e.g. events and seasonal destinations.
  - "Where can I go" personalisation: the passport is a context that makes the query richer (query = intent + passport). Visa-ease becomes a query-dependent feature, not a separate model.

## Brammert Ottens & Luca Falsina (Booking.com) — Scaling Machine Learning at Booking.com, Spark+AI Summit 2018 (https://www.youtube.com/watch?v=X8D36DY22T8)
**ML at scale is a platform problem: point-in-time-correct offline features, the exact same transformations online, a feature store with offline and online bindings, a self-service model server, and monitoring of input and output distributions.**

- Context: 2018 Booking had about 15k employees (about 2k in tech), 130k destinations, about 1.5M room nights per day, and about 200 data scientists using varied tools. The platform team's goal is to identify common problems, build tooling, then train and consult users, without forcing one way of working.

- Mental models / first principles
  - The pipeline: problem formulation (e.g. predict booking cancellation) → instances + labels → feature enrichment → transformations → model (H2O, TensorFlow, Vowpal Wabbit) → **online feature construction (often the slowest step to production)** → publish model + transforms + feature list → real-time or batch prediction → monitor outputs *and* inputs.
  - Historical correctness ("time-coordinate matching"): a feature must have the value it had *when the instance was created* (bookings so far at that moment, not today's count). Otherwise you leak the future into training.
  - Transformations carry state (mean/std for standardisation, bucket edges, target-encoding maps). That state must be serialised with the model so online scoring applies identical transforms. This avoids training/serving skew.

- Mechanisms
  - Offline: MySQL production DBs dumped nightly to Hadoop, plus an event stream into Hadoop for behavioural features. "Feature Vader" (Dutch *vader* = father) is a custom Spark ML Transformer: give it an instance DataFrame, the time column and the feature names, and it returns the enriched DataFrame. It handles billions of rows without days of waiting, with a low learning curve. Features are stored as change logs over time in Hive, so point-in-time joins are possible.
  - Transform portability: a "pipeline writer" exports a Spark ML pipeline into a minimal-dependency object servable online.
  - Modelling: H2O via Sparkling Water. Spark does munging, and the data is shipped to an H2O cluster for distributed training on billions of rows (random forests popular). Problem: YARN fair-scheduler preemption on a shared cluster kills executors, and H2O nodes in the same JVM aren't highly available, so one lost node kills the whole training run. Fix built with H2O: an *external backend* mode with H2O nodes on a separate non-preemptible YARN queue, while Spark keeps dynamic allocation.
  - Vowpal Wabbit: out-of-core online learning. Sample the big data to something that fits on *disk* (not RAM) and stream over it 1–3 passes.
  - Online features: "feature mappers" are Spark Streaming apps over Kafka streams (hotel impressions, searches). Stateless transforms (filter to searches for Italy) and stateful ones (visits per hotel page in the last 10 minutes). They run as long-lived containers, not on the preemptible batch cluster. Output goes back to Kafka (for monitoring rates and chaining mappers) and to **Cassandra** for durable millisecond lookups. Data modelling of the Cassandra tables is key to latency.
  - Feature store: a registry binding each feature name to offline and online implementations, with description, owner, and badges showing which exist. A feature with both is reusable "for free" by any data scientist.
  - Model service: an in-house Scala service with a common interface for H2O MOJO/POJO (compiled Java, no Spark/H2O core dependency, fast; loaded via reflection), VW and TF. Single and batch predictions. The caller passes the model name and entity keys, and the service fetches registered features from the store, applies the stored transforms and predicts. Self-service upload from an HDFS path goes live on all boxes in about 20–40 minutes.
  - Model portal: sampled request logging shows distributions of the output probability and every input feature over time, to spot feature degradation or the need to retrain. It is searchable by owner, status (testing/deployed) and the experiments that use the model.

- Decisions & trade-offs
  - Separate offline (Spark/Hive batch) and online (streaming → Cassandra) implementations of the same feature. Flexible, but it doubles the work and risks skew. The stated future direction is to auto-generate online feature mappers from offline Feature Vader configs.
  - Developer time is the bottleneck (few devs per data scientist), so invest in automating developer tasks.
  - Cassandra: relaxed consistency guarantees, but horizontally scalable with fast keyed reads. Good for feature serving.

- Rules of thumb
  - Store features as time-stamped change logs so any training set can be rebuilt as of any instant.
  - Ship model + transformation pipeline + feature list as one deployable unit.
  - Monitor input feature distributions, not only predictions.
  - Give every model and feature an owner and status in a searchable catalogue to encourage reuse.
  - Long-running streaming jobs need non-preemptible infra.

- Failure modes
  - Preemption killing non-highly-available distributed training on shared clusters.
  - Rewriting transforms by hand online leads to training/serving skew.
  - Online feature construction is the real bottleneck to production.

- Relevance to Sam / Trameter
  - On a 16GB laptop, the VW pattern applies: sample to disk and stream over the data (out-of-core). The same idea works with DuckDB/Parquet chunked scans.
  - For any ranking or cancellation model on Viator/RateHawk bookings, build point-in-time features (e.g. an activity's bookings *as of* the search time) to avoid leakage. Keep a small feature registry (name, owner, offline SQL, online source).
  - Serialise preprocessing with the model (sklearn Pipeline / ONNX) so the online booking flow scores exactly as offline. Log sampled inputs and outputs to watch for drift.

## Eugene Yan (Amazon) — Recsys Keynote: Improving Recommendation Systems & Search in the Age of LLMs, AI Engineer World's Fair 2025 (https://www.youtube.com/watch?v=2vlCqD6igVA)
**Three LLM-era levers for recsys and search: semantic IDs (content-aware item IDs that fix cold start), LLM-driven data augmentation (labels, queries, metadata, distilled into cheap models) and unified models (one ranker or embedding for many surfaces).**

- History: language-modelling ideas in recsys aren't new. 2013 item2vec-style embeddings from co-occurrence in interaction sequences, then GRU4Rec for short-session next-item prediction, then transformers over long histories (hundreds to about 2,000 item IDs).

- 1) Semantic IDs (cold start, sparsity)
  - Problem: hash-based item IDs encode nothing about content. Every new item starts from zero (cold start), the long tail has 1–10 interactions (sparsity), and systems drift toward popularity bias.
  - Kuaishou (number 2 short-video app in China, hundreds of millions of uploads per day): a standard two-tower model (user tower = user ID + sequence of item IDs; item tower = item ID) plus content inputs. Visual via ResNet, description via BERT, audio via VGGish. Back-propagating into those encoders is impractical, so: freeze them, **concatenate** the content embeddings, run k-means over about 100M videos into about 1,000 cluster IDs, and give each cluster ID a *trainable* embedding table. Training learns to map content space (via clusters) into behaviour space.
  - Results: beat hash IDs on clicks and likes, **cold-start coverage +3.6%** (share of new videos among those shown) and higher cold-start velocity (new videos hitting a view threshold).
  - Benefit: new items inherit learned behaviour from their content cluster. YouTube's work blends LLMs with semantic IDs, so the model can explain recommendations in human-readable text.

- 2) LLM data augmentation (labels, queries, metadata)
  - Search depends even more than recsys on metadata: query expansion, synonyms, spell correction, tags. Human annotation is costly, and LLMs are now very good at this.
  - **Indeed (email job recs):** bad recs lead to unsubscribes, which are nearly irreversible. Explicit thumbs-down is sparse. Implicit non-action is ambiguous (bad fit? wrong time? spouse works there?).
    - Process: experts labelled (user resume + activity, job) pairs. Open models (Mistral, Llama 2) did poorly because they ignored resume and job details even with enough context. GPT-4 reached about 90% precision and recall but was costly and took about 32 seconds. GPT-3.5 had only 63% precision (it would have thrown away 37% good recs). Fine-tuned GPT-3.5 reached the target precision at about 1/4 of GPT-4's cost and latency but still took about 6.7 seconds. Final step: **distil into a lightweight classifier** trained on the fine-tuned model's labels, AUC-ROC about 0.86, fast enough for real-time filtering (roughly under 200ms, as Yan recalled).
    - Outcome: about 20% of bad recs filtered. Contrary to the fear that fewer recs mean fewer applications, **applications rose 4% and unsubscribes fell 5%**. Quality beats quantity.
  - **Spotify (podcasts and audiobooks):** cold start on a whole *category*, with users habituated to song and artist search. Solution: query recommendations. Candidate queries come from conventional sources (catalogue titles, playlist titles, search-log mining, templated "artist + cover"), plus an LLM to generate natural-language exploratory queries. Rank them alongside the instant results, so the UI surfaces new categories without banners. Result: **+9% exploratory queries**.
  - Lesson: use conventional techniques first and the LLM to augment where needed. "Don't use the LLM for everything at the start." LLM augmentation gives richer data on tail queries and items at far lower cost than human labelling.

- 3) Unified models
  - Problem: separate systems for ads, search and recs, plus per-surface rec models (home, item page, cart, thank-you page). Duplicate pipelines, maintenance cost, and improvements don't transfer. (Stripe even built a transformer foundation model over payment sequences for fraud.)
  - **Netflix UniCoRn (unified contextual ranker):** a user foundation model (watch history) plus a context/relevance model. Unified input schema: user ID, item ID, search query (if any), country, task (search, pre-query, more-like-this). Missing fields are imputed cleverly: for item-to-item there's no query, so use the current item's title as the query. It matched or beat the specialised models across tasks. The value is less tech debt and faster future iteration.
  - **Etsy unified embeddings:** problem: very specific or very broad queries ("mother's day gift" matches few titles) over constantly changing handmade inventory, and lexical retrieval ignores user preference. Two-tower: product encoder (T5 text embeddings + query→product engagement logs) and query encoder, with *shared* token, category and user-location encoders (so location matches product location), plus user preference features (past queries, purchases). **Quality vector trick:** concatenate a quality vector (ratings, freshness, conversion rate) to the product embedding, and pad the query embedding with a constant vector of equal dimension so the dot product still works and quality is baked into ANN retrieval. Results: **+2.6% site-wide conversion, more than 5% more search purchases**.
  - Trade-off ("alignment tax"): cramming many tasks into one model can make some worse. You may end up with 2–3 unified models rather than 1.

- Rules of thumb
  - Cascade for LLM labelling: expert seed labels → try cheap or open models → strong model for quality → fine-tune mid model → distil into a tiny online classifier. Pick the metric that reflects the business harm (here precision, because false "bad" verdicts discard good recs).
  - Measure cold-start-specific metrics (new-item coverage and velocity), not just CTR.
  - Design a shared input schema with imputations so multiple surfaces can share a model.

- Relevance to Sam / Trameter
  - New Viator/RateHawk products and new destinations have no interactions. Semantic IDs (cluster IDs from frozen text or image embeddings of activity descriptions) give them behaviour-sharing from day one, and k-means on a laptop is feasible.
  - Drive Button: use GPT-class LLM judgments of (article, trip) fit as labels, then distil to a small classifier or reranker for cheap bulk scoring across thousands of articles. Measure precision, because bad injected buttons erode reader trust (the Indeed unsubscribe lesson).
  - Spotify-style query recommendations can introduce new Trameter categories (e.g. "visa-free from Nigeria", "weekend trips from Lagos") in search without banners.
  - The Etsy quality-vector trick lets you bake rating, freshness and conversion into a single vector index for destinations or activities. Location-sharing encoders are useful for travel.
  - A single unified ranker over search, "similar activities" and "where can I go" with task as an input is plausible once the data schema is unified.

## Abhishek Thakur — A no nonsense intro to BM25 (https://www.youtube.com/watch?v=TW9vHU1GpU4)
**BM25 scores a document for a query from three signals: term frequency (with saturation), term rarity (IDF) and document length relative to average. Two knobs, k1 and b, control it, and they should be tuned per field.**

- Mental models / first principles
  - The question BM25 answers: how relevant is document D to query Q? It is still at the core of most search engines, and of RAG and hybrid search alongside vectors.
  - Three intuitions: (1) more occurrences of a query term suggest more relevance, (2) rare terms are worth more than common ones, (3) long documents match many terms by accident, so they get a mild penalty. Together these stop huge documents from dominating and reward focused matches.
  - **Saturation** is the key property: a term appearing 50 times does not score 5x one appearing 10 times. There are diminishing returns, unlike raw TF-IDF.

- Mechanism (formula)
  - score(D,Q) = Σ over query terms qᵢ of IDF(qᵢ) · f(qᵢ,D)·(k1+1) / ( f(qᵢ,D) + k1·(1 − b + b·|D|/avgdl) ).
    - f(qᵢ,D) = term frequency in D. IDF = rarity across the corpus. |D|/avgdl = field length over average field length.
  - **k1** controls how much repeated terms are rewarded. Low k1 means one occurrence is nearly enough and repeats add little. High k1 means repeats keep boosting. (note: common defaults are k1 ≈ 1.2, range about 1.2–2.0.)
  - **b** controls length normalisation. b = 0 ignores length. b = 1 fully normalises, so long docs are penalised most. Thakur cites typical b of 0.7–0.9. (note: Lucene, Elasticsearch and Vespa default to b = 0.75.)
  - BM25 is computed per field. Multi-field ranking is a weighted sum.

- Demo (Vespa)
  - Schema: string fields `text` and `url` with `indexing: index | summary` and `index: enable-bm25`. A rank profile's first-phase expression sets the scoring, e.g. `bm25(text) + 0.1 * bm25(url)`.
  - Field-level parameters set via rank-properties, e.g. `bm25(text).k1 = 1.8`, `bm25(text).b = 0.4`, `bm25(url).k1 = 0.9`, `bm25(url).b = 0.3` (arbitrary demo values). Redeploying the application package with new rank profiles doesn't lose data.
  - Corpus about 4 million web documents. Query "Chicago deep dish pizza" returned in about 5 ms or less. The four profiles (text-only, url-only, text + 0.1·url default, tuned) gave visibly different top-4s. The URL-only profile promotes pages whose URL spells out the query. Changing k1/b reorders results, so tune per use case.

- Decisions & trade-offs / rules of thumb
  - Pick per-field weights and per-field k1/b. Short fields (titles, URLs, place names) usually want lower b (length varies little and is meaningful), and long body text wants standard b.
  - Expose several rank profiles and compare them side by side on real queries before choosing. Better still, compare on a judged query set (see the Turnbull/Bergum note: tuning k1/b alone moved BrowseComp-Plus retrieval about 30 points).
  - BM25 is the lexical half of hybrid search. Keep it even when adding embeddings.

- Failure modes
  - Defaults tuned for one corpus shape (short uniform passages) can be badly wrong for another (long variable web pages).
  - Vanilla BM25 has no notion of synonyms or semantics, which is why hybrid with dense vectors exists.

- Relevance to Sam / Trameter
  - For about 6M GeoNames place names, the name fields are short. Consider low b (about 0.2–0.4) and a separate field for alternate names, combining `bm25(name) + w·bm25(alt_names) + popularity prior`. Fuzzy and typo handling needs n-gram fields or edit-distance on top, since BM25 is exact-token.
  - Vespa, or a single-node BM25 library (e.g. bm25s, Tantivy), handles millions of docs on a 16GB laptop. Test several k1/b profiles against a small labelled set of Trameter queries.
