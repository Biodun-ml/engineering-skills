# er-1 — cross-cutting principles

1. **Do the n² arithmetic first; blocking is make-or-break.** 1B records all-pairs ≈ 10^18 comparisons ≈ $1B even on a million-node cluster; 100M records likewise intractable. Good blocking buys budget for better matchers; bad blocking forces you to trade quality for speed. (Tamr/Bates-Haus; Jekal/Data Surge)
2. **Retrieval can be the blocker.** Index the big reference set once (BM25 via bm25s: precomputed sparse token scores, ~2 GB for 4M docs, 50–80 ms/query, memory-mappable) and query top-k per record; DuckDB FTS worked but used 20–30 GB RAM and was ~5× slower; pure-Python rank_bm25 took ~100 s/query. Implementation choice beats hardware. (Kulkarni/PyData)
3. **Monitor comparison counts and block-size skew.** Too few labels → weak learned blocking → huge comparison counts and spilling/failing jobs; giant blocks are the usual killer. (Zingg)
4. **Standardise before you compare.** Strip junk characters/spaces, unify schema, abbreviations and codes (M/male/1), stem and drop stopwords for token methods; string matchers aren't semantic — use embeddings or synonym tables for "doctor/physician". (Jekal; Zingg; Kulkarni)
5. **Choose few, distinctive, well-populated fields; model nulls explicitly.** Constant or mostly-null fields add no signal; two nulls default to "agree" in Zingg unless you add a null-aware feature (which costs more labels). More match types = more features = more labels and slower jobs. (Zingg)
6. **Pairwise Fellegi-Sunter assumes independent pairs and one-to-one linkage; real clusters of 3+ break it.** Separate A–B, A–C, B–C runs can't assert three-way matches and are non-transitive. Either model higher-order clusters (5 partitions for a triple; latent-class EM), or stage it: accept the largest confident clusters first, enforce one-to-one greedily, remove them, then run pairwise FS on the remainder. (Thibaudeau/Census)
7. **Three outcomes, not two: match, possible match, no match — plus "ambiguous" hold-outs.** When a record fits two entities equally, don't pick one (≈50% wrong); bias toward false negatives; persist derived/disclosed relationships separately from merges. (Jonas/Senzing)
8. **Match records to entities, not just to records.** An entity's accumulated names/addresses/phones can match a new record that no single member matches; this catches deliberate "channel separation" and messy data. (Jonas/Senzing)
9. **ER must self-correct as data arrives.** New records can merge prior possible matches, split earlier members out, and deletions (GDPR, ageing) must unlearn dependent decisions; otherwise accuracy drifts (e.g. 1%/month → 4% wrong in a quarter) and batch reloads give saw-tooth quality. (Jonas/Senzing; Bates-Haus/Tamr on deltas destabilising clusters)
10. **Plan for deltas and stable IDs from the start.** Judge systems on full-run cost, delta cost (Tamr: 500M records <3h/<$300; 1% delta <1h/<$100) and ID stability, not only pairwise accuracy. (Tamr)
11. **ER is extremely imbalanced; accuracy is meaningless.** "Always non-match" scores 99.99%; use precision/recall on positives and sample/label so rare matches appear. (Tamr; Zingg)
12. **Training-label accuracy is not production quality — keep a golden set.** Score every config/model change against a fixed truth set (e.g. 80% → 85%); synthetic corruption is "too friendly" and synthetic cluster sizes too small, so test on realistic messy data. (Zingg; Schnell & Christen; Tamr)
13. **Label consistently and cover field combinations.** Only label a match if the record data itself carries the evidence; ~40–50 labelled matches covering varied field-agreement combinations is Zingg's rule of thumb; matches teach blocking more than non-matches. (Zingg)
14. **Hunt for hidden labels and augment.** Near-certain match markers in the data (even if populated ~20%) plus synthetic variants (character swaps, dropped fields) give training data without manual labelling; identity pairs are free positives. (Jekal)
15. **Start pairwise + supervised, move to record-to-cluster later.** Pairwise is faster to train and ship; record-to-cluster must handle split/merge outcomes. Rules are good for EDA but rot in production. (Jekal)
16. **Route the uncertain band to humans.** The score range where your labellers disagreed is the clerical-review band; log reviews and feed them back. Clerical review alone doesn't scale to massive linkage. (Jekal; Thibaudeau; Schnell & Christen)
17. **Real-time means a defined latency budget; ER is soft real-time.** Pair it with eventually consistent search/serving stores (Elasticsearch), monitor consumer lag, and emit per-stage telemetry with a routing slip to find dropped records. (Jekal)
18. **Linkage error is uneven and propagates.** Non-English/minority names (and transliterated place names) link worse under one-size cleaning; check error rates per subgroup, tailor strategy per subgroup, and report linkage quality to downstream analysts. (Schnell & Christen)
19. **Use domain constraints as cheap QA.** One-to-one expectations (one GP per patient; observed max ~50) expose over-merging; check cluster-size histograms. (Schnell & Christen; Thibaudeau)
20. **PPRL caution:** hashed phonetic codes (MD5 of Soundex) are attackable yet "certified"; most PPRL assumes honest-but-curious parties and single-score thresholds, and lacks privacy metrics and missing-data handling. (Schnell & Christen)

**For Sam (6M GeoNames/OSM places, supplier listings, airports on 16GB):** block by country + geohash/grid + name tokens or retrieve top-k with a persisted bm25s index over names+alternate names; standardise per script/country; score with string + distance + category + numeric checks; hold out ambiguous same-name POIs; enforce one-to-one per source before connected components; match each supplier to the master dataset (not supplier-to-supplier); keep a stratified gold set; process deltas incrementally with stable IDs carried forward by cluster overlap.

---

## Yves Thibaudeau (US Census Bureau) — Fellegi-Sunter Decisions beyond Pairwise Record Linkage (https://www.youtube.com/watch?v=TNsOsYlhZLE)
**Core idea in one line: classic Fellegi-Sunter (FS) assumes independent pairs, which breaks as soon as a cluster has 3+ records; model triples (or higher-order clusters) directly with a latent-class EM, resolve the largest clusters first, enforce one-to-one, then run ordinary pairwise FS on what is left.**

- Mental models / first principles
  - Record linkage = assigning an unobserved matching status to clusters of records. For a pair there are only 2 statuses (match / non-match). For a triple there are 5 possible partitions: all three distinct; one of three pairs (AB, AC, BC) together with the third apart; all three the same entity. (note: the count follows the Bell numbers 2, 5, 15 ...; the speaker guessed 24 latent classes for quadruples, but a set of 4 has 15 partitions)
  - FS (1969) rests on two assumptions: (1) clusters (pairs) are independent of each other; (2) within a cluster, field agreements are conditionally independent given the true status. These give additive log-likelihood-ratio scores and cheap computation, which is why nearly every package (fastLink, Splink, RecordLinkage) is built on them, even though they are often violated.
  - Pair independence implies one-to-one linkage between two files. If one record links to two others, there are hidden duplicates, so you really have a triple — and a triple cannot be assembled from independent pairs, because agreement is transitive (A=B and A=C on a field forces B=C). So pairwise FS on overlapping pairs is mis-specified.
  - Running A-vs-B, A-vs-C, B-vs-C separately cannot assert a three-way match and uses fewer parameters (3×8=24 in his example vs 65 for the joint triple model), so it has less discriminating power.
- Mechanisms
  - Treat linkage as a missing-data problem: the latent status is missing, so fit a mixture of multinomials with EM (Dempster–Laird–Rubin 1977; used in linkage since the 1980s — Winkler, Jaro; first used by Pat Kelly at Census).
  - Comparison is still binary agree/disagree per field, but with three records each field has 5 configurations (none agree, only AB, only AC, only BC, all agree). With 3 fields that is 5^3 = 125 cells per latent class; 5 latent classes → 625 cells.
  - Implementation: the multinomial mixture is fitted as 5 independent Poisson log-linear regressions (one per latent class), each with conditional-independence main effects: 13 parameters per class, 65 total, intercept included so the Poisson reproduces the multinomial. E-step apportions observed cell counts across classes; M-step refits the Poisson regressions (Newton-Raphson, tolerance 1e-4 / 1e-8 / 1e-12 compared).
  - Demo data: three files of ~1,000 records drawn from the public RL10000 German-patient register (R RecordLinkage package): per file 500 standalone, 50 in each pairwise match type, 50 in three-way matches. ~8 million candidate triples; ~6.5M disagree on everything; only 38 agree fully on all three fields. Counts generated by piggybacking on fastLink's pattern counting.
  - Non-estimability: latent class models often are not identifiable. Using Fienberg & Rinaldo's extended MLE (exponential family allowing infinite parameters / collapsing dimensions), unestimable cells are set as structural zeros. Only 47 of 65 parameters and 288 of 625 cells were estimable; in the three-way class only 8 cells survived and only 5 of 13 degrees of freedom (checked via the rank of the Hessian restricted to surviving cells).
  - Classifier: Sadinle & Fienberg (JASA 2013) generalised FS to multiple files/partitions; with two classes it reduces exactly to FS. They apportion triples into all classes in one go (comprehensive optimality).
- Decisions & trade-offs
  - Staged alternative proposed: (1) score triples only for "three-way match vs everything else"; rank cells by posterior probability, pick a cutoff; (2) enforce one-to-one among accepted triples with a greedy algorithm so no record sits in two triples; (3) remove those records; (4) run standard pairwise FS/fastLink on remaining pairs. Step 1 keeps FS-type optimality (for fixed precision, maximise expected recall); later stages are only conditionally optimal. Generalises: start with the highest order cluster you care about.
  - Choose your starting point for EM deliberately so each latent class lands on the meaning you want (e.g. class 5 = three-way matches); this is legitimate and helps label-switching, though identifiability still isn't guaranteed.
  - Frequentist MLE route vs Bayesian (others at Census working on Bayesian higher-order clustering): both possible; MLE runs in a few lines of base R.
- Rules of thumb
  - Order candidate cells by posterior probability, eyeball the boundary rows, set the cutoff where the patterns start disagreeing on two fields (in the demo the fifth-ranked pattern, full first-name agreement but last-name and year problems, posterior ≈0.016, was cut).
  - After thresholding, always resolve conflicts (a record used twice) before accepting — that is what keeps the model's independence assumption true.
  - Pairs that are subsets of accepted triples are automatically matches; don't re-score them.
- Failure modes & gotchas
  - Pairwise matchers implicitly produce non-transitive or many-to-one output; production systems (incl. Census) paper over it with clerical review, which does not scale to massive linkage.
  - Agree/disagree binarisation discards field content and partial similarity; the method generalises but the demo did not.
  - Latent class EM can silently produce near-zero, unestimable cells; check identifiability rather than trusting every posterior.
- Relevance to Sam
  - For GeoNames + OSM + a third source (e.g. a supplier list), matching each pair separately then joining with connected components will create chains (A~B, B~C but A≠C). Either resolve multi-source clusters explicitly, or at minimum enforce one-to-one per source pair (greedy by score) before clustering.
  - The "resolve biggest confident groups first, remove, then match the remainder" staging is a cheap, practical pattern for a 16GB laptop: each stage shrinks the candidate set.
  - Splink/fastLink are pairwise FS; they will not model triples — clustering is a post-step.

## Zingg AI (Sonal Goyal, founder) — Training Entity Resolution Models: Zingg Community Event, Nov 2024 (https://www.youtube.com/watch?v=f_gsDvbWFek)
**Core idea in one line: in Zingg, both the blocking model and the match classifier are learned from your labels, so field choice, match types and label quality/coverage are the real levers for speed and accuracy.**

- Mental models / first principles
  - ER is compute-bound because there is no join key; naive all-pairs on even 10k records explodes. Every config decision trades runtime against precision (are declared matches right?) and recall (did we find all true matches?).
  - Zingg's two engines: (1) a learned blocking model that cheaply groups near-similar records; (2) a similarity predictor (logistic regression) that classifies pairs inside a block. Users are meant to give yes / no / not-sure labels and never hand-tune weights or thresholds.
- Mechanisms
  - Blocking is learned only from pairs you mark as matches. Zingg applies many fine-grained hash functions per field (e.g. first 2 characters, exact city) and builds a tree of them; a function is useful if true matches consistently share its output. Example: if every match you label has the same city, city becomes a blocking key (San Francisco vs Fremont never matched). All fields except those marked "dont use" participate in blocking.
  - Match types become classifier features; one match type yields one or several features (fuzzy ≈ 2–3 features: tolerant of typos/abbreviations; works on strings, ints, doubles, dates). Specialised types: email, pincode, numeric, numeric-with-units, only-alphabets exact/fuzzy (force extraction of a sub-pattern), null-or-blank. A column can carry several match types (e.g. fuzzy + numeric so house numbers are compared explicitly).
  - Classification uses the whole row, not one column: a single exact or mismatched field doesn't decide alone. Field weights are learned from labels.
  - Nulls: by default two nulls are treated as agreement. The null-or-blank type records nullness as its own signal so the model can learn how much to trust null comparisons — more powerful but needs more labels.
  - Semantics: matching is string-based (edit distance etc.), so "doctor"/"physician" or "M"/"male"/"1" don't match unless you standardise them upstream; for semantic needs, put embeddings in an array-type field.
  - Logs print the number of comparisons; it falls as blocking improves with more labels.
- Decisions & trade-offs
  - Pick the smallest set of distinctive, well-populated fields. A mostly-null column gives no signal; a ~70%-populated column can still help. A constant column (everything "California") is useless — mark it dont-use. IDs you want in output but not in matching are dont-use.
  - More match types → more features → more labels needed and slower jobs (modest but real hit). Add them deliberately.
  - Incremental: "link" maps a delta against an existing set and returns only links (a record matching three existing records links to all three); it is designed for two individually de-duplicated sets, and you must post-process to rebuild "historical + delta" output. Enterprise has a separate run-incremental flow.
- Rules of thumb / checklists
  - Profile first: know populated fields, variants, common words; standardise codes before matching.
  - Label consistently: only mark "same person, different address" as a match if the data itself carries enough signal and you'd label every analogous pair that way. Human-only knowledge that isn't in the record cannot be learned and makes results inconsistent.
  - Aim for ~40–50 labelled matches; check that labels cover field combinations (same first / different last name, near-same state, etc.). Feed extra training samples if coverage is thin.
  - Slow job? Most likely too few labels and a weak blocking model: look at the comparisons count in logs and label more.
- Failure modes & gotchas
  - Unbalanced manually added training samples can produce skewed blocks (some huge), heavy disk spill and failing jobs. Zingg is adding block-size diagnostics.
  - Early labelling rounds show mostly non-matches (10 matches + 10 non-matches requested), wasting cycles; they are experimenting with biasing toward probable matches since matches teach more (especially blocking).
  - Training-set accuracy on 40–50 labels is a poor proxy for production quality, because new data drifts. Build a golden/truth set and benchmark every model change against it (e.g. 80% → 85%).
  - Transitive-only cluster members (C joins via B but doesn't really match A) are hard to see in the open-source version; enterprise exposes them.
- Relevance to Sam
  - Learned blocking from labelled matches is an alternative to hand-written Splink blocking rules; either way, monitor comparison count and block-size skew — a giant block (e.g. all "Hotel X" names, or a dense city) is what kills a 16GB laptop.
  - For GeoNames/OSM and supplier listings: standardise category codes, country codes, abbreviations ("St"/"Saint") before matching; use numeric match on address numbers; consider embeddings for semantic name equivalence.
  - Keep a fixed gold set of known place/supplier matches and re-score it on every config change.

## Charles Jekal (Data Surge) — Real-Time Entity Resolution At Scale, Current 24 (https://www.youtube.com/watch?v=177DkF-K7ww)
**Core idea in one line: real-time ER is the same five-step pipeline (ingest/cleanse → standardise → block → match → canonicalise) run per event on a streaming backbone; start pairwise and supervised, make blocking strong, accept soft real-time with eventual consistency.**

- Mental models / first principles
  - Goal: one current, best-sourced view of each entity (people, products — e.g. not 30,000 listings of the same tape). Uses: case management/claims history, fraud, KYC, lifetime value.
  - Most literature (Fellegi-Sunter etc.) assumes batch. Real-time means a defined, guaranteed completion budget, not "instant". Hard real-time = missing the deadline is failure; firm = value drops sharply; soft = best effort. ER is soft real-time; chasing hard real-time wastes money. Soft real-time fits naturally with eventually consistent stores — the right entity appears eventually.
- Mechanisms (pipeline)
  1. Ingest + cleanse: strip illegal characters, leading/trailing spaces. Clean data is a prerequisite for string-distance comparisons.
  2. Standardise/normalise: consistent schema (first/middle/last), consistent abbreviations, status codes mapped to lookup keys.
  3. Blocking: 100M records all-vs-all is O(n²) and intractable; blocking is the "make or break" step. Good blocking yields high-quality candidates, freeing budget for expensive matchers; poor blocking forces you to trade quality for speed. Newer research even uses deep learning just to generate candidates. Verify that your blocking scheme actually saves time.
  4. Matching: record-to-record (pairwise) or record-to-cluster (compare to canonical entity). Methods: FS probabilistic, rules, decision trees/ensembles, deep learning; usually edit-distance/similarity features.
  5. Canonicalisation: choose the most accurate/current value per attribute (Charles over the alias Chuck, latest address); mostly organisation-specific rules, not ML.
  - Identity rule: a record must always match itself; if your system can't guarantee that, ER is near impossible.
  - Architecture: sources → Kafka → cleansing + matching (Databricks, or Lambdas) → canonicalise → publish to topic → Elasticsearch (serves both candidate search/blocking and API reads) → notification microservice/Flink for downstream "attribute changed" subscribers. Containerised, distributed.
  - Monitoring: alert on Kafka consumer lag (growing lag = slow stage or a bad deploy); emit per-record telemetry events on a separate topic at each stage and use a routing-slip pattern (declared expected stages) to find records that silently dropped out. Tiered storage keeps topic history cheaply on object storage → replay/reprocess (Kappa style).
- Decisions & trade-offs
  - Pairwise vs record-to-cluster: start pairwise. Training data is easy (identity pairs are guaranteed matches; augment by swapping characters or dropping fields to fabricate positives). Record-to-cluster must also handle a new record invalidating past merges → split or merge outcomes, much harder to train. Run pairwise for a while; the clusters it produces become training data for later record-to-cluster.
  - Statistical (FS): no labels needed but relies on conditional independence of fields — do EDA to check fields aren't dependent.
  - Rules (SSN equal → match; name+DOB equal → match): good for EDA, rigid in production; rules accumulate, and identifier schemes change (SSNs became randomised).
  - Preferred: supervised learning (even a simple decision tree is strong) with good MLOps so models can be upgraded.
  - Real-time over batch: easier to derive batch from real-time than the reverse, and not much harder if blocking is good.
  - Eventually consistent, highly available, parallel store (Elasticsearch) for both search and serving.
- Rules of thumb
  - Human-in-the-loop: the score band where your own labellers were unsure is the band to route to clerical review.
  - "Hunt for labels": find hidden near-certain match markers (a field that, when present, almost guarantees a match — even if populated only ~20%), use them as labels, augment with realistic synthetic variants, then train.
  - Before architecture: fix the latency requirement, then each source's latency and CDC availability, then the matching scheme.
- Failure modes & gotchas
  - Analytics on incomplete ER give misleading counts.
  - Rules-based systems rot as data practices change.
  - Records disappearing mid-pipeline without errors — need telemetry to catch.
- Relevance to Sam
  - For supplier listings arriving continuously (Viator, RateHawk updates), the pattern is: standardise → query a search index (Elasticsearch/OpenSearch, or locally DuckDB FTS / a BM25 index) for candidates → score pairwise → attach to existing entity. Batch backfill + incremental per-delta matching.
  - Hidden labels exist in Sam's data: shared Wikidata IDs, IATA codes, GeoNames IDs in OSM tags, supplier-provided lat/long+exact name — use as positives to train a classifier.
  - Synthetic augmentation (typos, dropped fields, abbreviations) is a cheap way to get positives for place-name matching.

## Aniket Abhay Kulkarni (Curlscape) — Scaling Fuzzy Product Matching with BM25, PyData Global 2025 (https://www.youtube.com/watch?v=njg9fEvp0BE)
**Core idea in one line: treat matching as search — build a persisted BM25 sparse index over the big reference set once and query it per record; implementation choice (bm25s with precomputed sparse scores) mattered more than hardware, taking 1M×4M fuzzy matching from ~11 days to under a day.**

- Mental models / first principles
  - Problem: enrich ~1M food products (≈2 GB) with ingredients/nutrition/allergens from Open Food Facts (~3.8–4M rows, ~11 GB TSV) and FDA data. No shared IDs, different naming conventions — only product names to match. M×N brute force is out; the budget was ~1 day, which means ≤~100 ms per query (1 s/query ≈ 11 days per reference set).
  - Product-name variation types: one side shorter / one longer with marketing attributes (bean type, aroma), word-order swaps ("reduced fat milk 2%" vs "2% reduced fat milk"), punctuation/spelling ("Coca-Cola" vs "Coca Cola"), units and pack sizes (units were not solved in this project).
  - BM25 fits because it is bag-of-words: order-insensitive, tolerant of extra tokens, boosts rare discriminative tokens (brand names) via IDF over common ones ("coffee", "milk"), saturates term frequency (k1) so repetition doesn't inflate scores, and normalises by document length (b). Default k1/b rarely need tuning.
- Mechanisms
  - Attempt 1 — rank_bm25 (pure Python): scores every document per query, then sorts → 90–120 s per query. Unusable.
  - Attempt 2 — DuckDB FTS extension (SQL, built-in stemmer, stopwords, language): 0.3–0.5 s/query, but 20–30 GB RAM for 3.8M docs. Better, still ~5 days; memory too high for a laptop. Nice for analytics/integration, not for this BM25 workload.
  - Attempt 3 — bm25s: at index time precompute per-token BM25 contributions and store them as SciPy sparse matrices; at query time, stem + remove stopwords, then only slice columns for the query's tokens (a 2-token query = 2 lookups out of, e.g., ~50k vocabulary), sum, take top-k. Token IDs in contiguous arrays, only non-zeros stored. ~2 GB RAM for 4M docs; memory-mapping lets it scale to 10–20M docs on disk. Latency 50–80 ms/query; ~1500× faster than rank_bm25.
  - Full pipeline: load with dask-cuDF (2× RTX 3090, 64 GB RAM, 24-core box) → build/persist index → batched search (50 workers, batch 500) → enrich via GPU UDFs → write Parquet. Total ~6 hours, ~5000× vs naive Python. CSV load 2.5–3 min in pandas vs 17 s in cuDF (~14×).
  - Quality: ~85% match quality measured as ranking accuracy on a monitored labelled subset.
- Decisions & trade-offs
  - Search-as-blocking: top-k retrieval replaces pairwise comparison; you still need a decision step (threshold or reranker) on top-k candidates.
  - Python library vs database FTS: DB is convenient but memory-hungry; specialised sparse-matrix library wins on both latency and RAM.
  - GPU dataframe helps the join/enrich side, not the retrieval itself; optional.
- Rules of thumb (speaker's lessons)
  - Persist the index; build once, reuse across runs.
  - Stemming (and stopword removal) is non-negotiable.
  - Batch queries.
  - Profile before optimising; algorithm/implementation choice beats hardware.
- Failure modes & gotchas
  - Pure-Python per-query scoring over millions of docs is O(N) per query plus sort; never do it.
  - Bag-of-words can't separate variants differing only in numbers/units (2% vs whole milk, 500 g vs 1 kg) — need explicit numeric/unit comparison after retrieval.
  - BM25 is lexical: no handling of typos within a token unless you add character n-grams; synonyms not handled.
  - "Match quality 85%" was ranking-based on a subset, not full precision/recall — evaluate your own thresholded decisions.
- Relevance to Sam
  - Directly applicable on a 16GB laptop: a bm25s index over ~6M GeoNames/OSM names (+ alternate names) should fit in a few GB, memory-mappable; query with supplier listing names (Viator/RateHawk) for top-k candidates, then filter by geo distance/country and score with Jaro-Winkler, token-set ratio, category, numeric checks.
  - Prefer bm25s over DuckDB FTS for big candidate retrieval if memory is tight; DuckDB remains good for the joins/enrichment and Parquet IO.
  - Consider combining with a geographic pre-block (country or geohash) so the index or the post-filter is per region.
  - Add a numeric/unit-aware check for hotel names with numbers ("Hotel 81 Chinatown") and room/pack attributes.

## Rainer Schnell & Peter Christen — PPRL Topics for Research (workshop panel, 2019) (https://www.youtube.com/watch?v=7VdL7kPX0SA)
**Core idea in one line: privacy-preserving record linkage (PPRL) has many encodings but lacks realistic test data, realistic attack models, richer classifiers, quality/privacy metrics, handling of missing data, multi-party/dynamic linkage and certified standards — and linkage error itself is an ethical and statistical problem.** (note: the auto-transcript is badly garbled; points below are only those clearly recoverable.)

- Mental models / first principles
  - Real data is far "weirder" than synthetic corruptors produce; generated test sets are too friendly. Access to real identifiers (even of deceased people, in Germany) can take over a year, so gold standards are scarce.
  - Census experience: some people intentionally vary their identifiers slightly to avoid being linked — a deliberate adversarial noise that ER research hasn't modelled.
  - Linkage quality is not uniform: migrant and minority subpopulations (Arabic, Sri Lankan names etc.) have more name variants that standard cleaning/standardisation (built for Smith/Williams) cannot fix; these are often exactly the groups a policy study targets. Errors therefore correlate with the variables you analyse, violating the independence assumption convenient for method development.
  - Linkage is the first step of analysis; linkage error propagates into downstream estimates and can create biased or harmful results (an audience point: errors concentrated in one subgroup produce side effects).
- Mechanisms / research gaps listed
  - Attack models: nearly all PPRL work assumes honest-but-curious (semi-honest) parties. Malicious and in-between models (covert adversaries; probability a party cheats) are barely studied; commercial settings (banks, insurers) have incentives to learn more than they should, unlike government agencies linking with each other. Human-insider attacks (e.g. a data breach by an owner/linkage unit) defeat any encoding.
  - Classification: PPRL usually reduces each pair to one similarity score with a threshold; mainstream ER uses per-attribute similarities, supervised/unsupervised classifiers and collective/relational ER (records' relationships, e.g. author–paper–venue graphs). Little of that exists in PPRL. Histograms of real similarity scores show matches and non-matches overlapping; a single threshold is crude.
  - Multi-database and dynamic linkage: real systems are many databases (e.g. a patient across subsets of ~20 hospitals), streams of query records, real-time identity checks. Pairwise linkage between every pair of databases doesn't scale or make sense; collusion among parties becomes a risk with more parties.
  - Evaluation without ground truth: in PPRL nobody sees plain records, so precision/recall are unknown. Ideas: privacy-preserving interactive clerical review with partially masked values; mining frequent mismatch patterns.
  - Privacy metrics: no agreed single number analogous to accuracy. Disclosure risk can be framed as probability an encoded record can be re-identified to one plaintext value (1 of 100 candidates → 1% risk), assuming the attacker holds a plaintext reference set. Privacy, linkage quality and computational cost must be traded off in the context of the purpose.
  - No systematic comparison of the many encoding and hardening methods (Bloom filters etc.). Missing data is the central practical problem; ad-hoc fixes may open new attack vectors and are unstudied.
  - Certification: German cancer registries (16 independent ones) use a certified method that is essentially MD5 hashes of Soundex codes — attackable, acknowledged by the certifier, still standard. An international recommendation/certification (ISO/IEC data-sharing standards work was mentioned) would beat nothing.
- Decisions & trade-offs
  - Consider different linkage strategies per subpopulation (tailor blocking/comparison to name origin) instead of one strategy for everyone.
  - Report linkage quality to analysts so they can adjust (Harvey Goldstein's work on propagating linkage error into analysis; econometrics literature on linkage error).
  - Full automation is unrealistic; keep human review of hard cases, log reviews, and use them (active-learning style) to improve the model over time (Australian Bureau of Statistics approach mentioned).
- Rules of thumb
  - Test methods on realistic, messy data; synthetic corruption underestimates difficulty.
  - Check whether linkage error rates differ by subgroup.
  - Don't treat a hashed phonetic code as privacy protection.
  - Sanity checks from domain constraints catch errors (UK example: each patient should be registered with exactly one GP; observed max was ~50).
- Failure modes & gotchas
  - Single-threshold classification; ignoring missing data; assuming semi-honest parties in commercial settings; no ground truth; homogeneous strategy over heterogeneous names.
- Relevance to Sam
  - Place names in non-Latin scripts and transliterations (Arabic, Yoruba, Chinese, etc.) behave like the "hard subpopulation": a single cleaning recipe designed for English names will underperform. Use per-script/per-country normalisation and check match rates per country.
  - Domain-constraint checks are cheap QA: an airport code should map to one place; one supplier listing should map to one destination; cluster size histograms reveal over-merging.
  - Linkage across many suppliers is multi-party; avoid N² pairwise source linking — link each source to a master (GeoNames-based) dataset instead.

## Jeff Jonas (Senzing) — Entity Resolution Explained Step by Step (https://www.youtube.com/watch?v=VFE3kGdoXzA)
**Core idea in one line: high-accuracy real-time ER is entity-centric and self-correcting — each new record can merge, split, re-resolve or hold out earlier decisions, and deletions must unlearn — otherwise accuracy silently drifts.**

- Mental models / first principles
  - ER makes thousands of decisions per second; most of the hard "moves" are expensive in real time. Jonas walks 10 records through 10 moves.
  - Bias toward false negatives: only merge when sure; otherwise record a possible match. Wrong merges are worse than missed ones.
  - Match records to entities, not records to records. An entity accumulates every name/alias, address, phone, email it has seen; a new record may match none of the individual records but match the entity's combined feature set ("entity-centric learning").
  - Accuracy is not a point-in-time number. If new data invalidates earlier decisions and the system can't revise them, error accumulates. Example: 1% drift/month (an aggregator's figure) → 4% wrong after four months → 40,000 errors per 1M records; batch systems "fix" this by periodic full reloads, so accuracy saw-tooths.
- Mechanisms — the 10 moves
  1. Fuzzy match: Rob/Bob/Robert in one name family, DOB with month/day transposed, phone with/without +1 → one entity on combined evidence.
  2. Derived relationship: same email and surname but different first name (Patricia vs Bob) → different people, but persist a "related" link (families share emails).
  3. Disclosed relationship: relationship given by the data (corporate hierarchy, spouse/co-signer) — no inference needed.
  4. Possible match: same name + same DOB only → don't merge; store as possible match (name+DOB is good evidence but not certain).
  5. Ambiguous: a record (e.g. shared email) that fits two existing entities equally → hold it out rather than assign randomly (random assignment is wrong ~50% of the time). George Foreman named all five sons George: home address + phone could be any of six people. If only one candidate entity existed, it would be merged confidently — ambiguity depends on context.
  6. Re-resolve: a new record adds a feature (work email) to entity E1, which now makes a prior possible-match record conclusively part of E1 → automatic merge. New data fixes old indecision.
  7. Un-resolve/split: a new record shows that an earlier member (record 3, name+ID, very different DOB) really belongs with the newcomer → pull it out of E1 into a new entity. Senzing spent ~a year making this real-time at billions of records.
  8. Entity-centric match (record 10, as above).
  9. Delete/unlearn: removing a record (data ageing, GDPR/CCPA) must cascade — undo matches that only happened because of it, demote confident merges back to possible matches.
  10. Cross-script and entity types: Arabic/Mandarin/English variants (100+ spellings of Muhammad, shortest "MHD"); organisations (names, addresses, fax, URLs); vessels (name + IDs).
- Decisions & trade-offs
  - Real-time self-correction vs periodic batch reload: reloads are simpler but leave you wrong between runs.
  - Record matching vs entity matching: entity matching catches "channel separation" — deliberate variation of name/SSN/DOB across channels (a casino case: 32 names, 8 SSNs, 5 DOBs) — crucial for fraud/KYC/counter-terrorism; also helps with ordinary messy data.
- Rules of thumb
  - Keep three outcomes, not two: match, possible match (review/relationship), no match; plus explicit "ambiguous" hold-out.
  - Persist relationships separately from merges.
  - When a record arrives, ask: does it change any previous decision? (re-resolve neighbours of affected entities).
  - Design deletes as first-class operations.
- Failure modes & gotchas
  - Random or first-come tie-breaking between candidate entities → silent false positives.
  - Merging on one strong-ish key (shared email, name+DOB) → over-merging families/namesakes.
  - Measuring accuracy only right after a reload hides drift.
- Relevance to Sam
  - Places: many distinct POIs share a name within a city ("Central Station", "Hilton" ×5) — the George Foreman problem. If a supplier listing fits two candidate places equally, hold it out for review instead of picking the nearest arbitrarily.
  - Entity-centric matching suits a master location dataset: match a new supplier record against the place entity's union of names (GeoNames alternate names, OSM name:* tags, supplier names already linked) and coordinates.
  - When incremental supplier updates arrive, re-check affected clusters (merge/split), or schedule periodic full re-clustering and measure drift between runs.
  - Possible-match tier + relationship links ("hotel within resort", "terminal within airport") rather than forcing merges.

## Nik Bates-Haus (Tamr) — Part 1: Why is Entity Resolution at Scale Hard (https://www.youtube.com/watch?v=K8iAQHfNDSQ)
**Core idea in one line: ER at scale is hard for five reasons — quadratic (and worse) algorithms, extreme class imbalance, no ground truth, many required skills, and data change that destabilises clusters — so success must be measured in end-to-end cost, delta-update time, stable IDs and quality metrics.**

- Mental models / first principles
  - Brute force on 1B records = 10^18 comparisons; at 10 µs each ≈ 100 million years single-threaded; even a million-node cluster ≈ 1 year and ~$1B. Blocking isn't optional.
  - Algorithmic hardness compounds: pairwise comparison is O(n²); clustering is O(n² log n) in their framing; tracking cluster survivorship (stable entity IDs) across successive versions is combinatorial (he calls it n!).
  - Class imbalance: true matches are so rare that "always non-match" is 99.99% accurate. Accuracy is meaningless; you need precision/recall on the rare positive class, and a classifier/labelling strategy that finds the rare positives.
  - No ground truth over any meaningful slice of the n² space; building it needs costly SME time.
  - Multi-disciplinary: data engineering, analysis, ML, domain SMEs, governance/compliance, parallel systems and cloud ops, plus product management (consumer feedback, bug triage, releases).
  - Data change is destabilising: systems run for years; small edits ripple across many clusters; distribution shifts degrade models; there's no cheap way to update clusters or to monitor model performance and know when to retune.
- Mechanisms / benchmark (vendor claims)
  - Synthetic test: FEBRL generator → 330M people + 50% duplicates = 500M records. FEBRL's default cluster sizes are too small versus real data; Tamr publishes a modified generator producing larger clusters (harder).
  - Full run, raw records → golden records: under 3 hours and under $300 on a 240-core compute cluster + 60-node storage cluster.
  - Incremental: a 1% delta (5M adds/deletes/modifications) against the 500M processed in under 1 hour and under $100, with stable IDs and accuracy metrics → "hourly updates for the US population" is affordable.
- Decisions & trade-offs
  - Judge an ER system on: end-to-end cost and time, incremental delta cost, ID stability across runs, and observable quality metrics — not on pairwise accuracy alone.
- Rules of thumb
  - Do the back-of-envelope: n²/2 × cost per comparison before designing anything.
  - Use precision/recall on matches, never overall accuracy.
  - Test with synthetic data whose cluster-size distribution resembles reality (large clusters stress clustering and blocking).
  - Plan for deltas (add/delete/modify) and stable entity IDs from day one.
- Failure modes & gotchas
  - Underestimating cluster sizes in test data → over-optimistic performance.
  - Re-running from scratch each time changes entity IDs, breaking downstream consumers.
  - Silent model decay with no monitoring.
- Relevance to Sam
  - 6M places all-pairs ≈ 1.8×10^13 comparisons — impossible on a laptop; must block (country + geohash/grid cell + name token) to bring it to ~10^7–10^8 pairs.
  - Supplier-to-destination matching is extremely imbalanced; evaluate with precision/recall on a labelled sample, and stratify the sample (by country, place type) so rare positives appear.
  - Trameter's master location IDs must stay stable across re-runs: carry forward IDs by maximum overlap between old and new clusters (survivorship), and process supplier deltas incrementally.
