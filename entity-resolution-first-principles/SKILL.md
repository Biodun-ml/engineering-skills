---
name: entity-resolution-first-principles
description: First-principles entity resolution playbook from 14 talks (Linacre/Splink, Resnick & Thibaudeau/Census, Zingg, Jonas/Senzing, Tamr, BM25 matching). Use whenever deduping, fuzzy-matching or linking records (places, suppliers, products, people) or designing blocking, scoring, clustering or ER evals.
---

# Entity resolution from first principles

This skill turns 14 talks (14 note sections, two files) into working judgment for entity
resolution (ER): dedup, record linkage, fuzzy matching and master-data building at scale.
This file holds the procedure, the principles the teachers keep coming back to, checklists
and a worked design for Sam's places data. `references/` (when present) holds detailed
notes for every video, each credited to its teacher.

Use it whenever two records might describe the same thing and there is no clean shared key,
even if nobody says "entity resolution": merging GeoNames and OSM, mapping Viator/RateHawk
listings to master places, deduping a supplier feed, joining product catalogues, "why do we
have three Lagos rows?", or choosing between Splink, Zingg, dedupe and a search index.

**How to use it**
1. Follow the ER pipeline (section 1) for any new matching job; size it with the arithmetic first.
2. Run the blocking and evaluation checklists (section 3) against the design or code.
3. If a `references/` folder sits next to this file, use it for depth. `references/INDEX.md`
   lists every video with its line number, or run `grep -n -i "keyword" references/*.md`.
   Each section ends with a "Relevance to Sam" or "For Sam" note: Sam builds Trameter's master
   location dataset (~6M GeoNames/OSM rows), matches supplier listings (Viator, RateHawk) to it,
   and does all of it on a 16 GB laptop with DuckDB/Parquet. Use those parts only in that
   context. Lines marked "(note: …)" and those Sam notes are helper additions, not the
   speaker's words. If `references/` is missing, the full notes live in the user's private
   GitHub repo `Biodun-ml/engineering-skills` (folder
   `entity-resolution-first-principles/references/`); clone it when you need depth.
4. Explain the *why*, and credit the speaker when you use a specific pattern.
5. Library APIs move fast (Splink 5 shipped in Sep 2026 with breaking changes). Check the
   current docs before writing library calls; section 7 lists what changed.

---

## 1. The ER pipeline (procedure)

ER = **estimate the missing unique ID column**. Dedup and linkage are the same problem:
linking is stacking the datasets and deduping. *(Linacre; Resnick; Goyal)*

0. **Do the arithmetic.** All-pairs is n(n−1)/2: 1M records ≈ 5×10^11 pairs; 6M ≈ 1.8×10^13.
   1B records at 10 µs per comparison ≈ 100 million CPU-years. Blocking is not optional, and
   good blocking buys budget for better matchers. *(Bates-Haus — Tamr; Juvenal; Jekal)*
1. **Define the entity and the cost of error.** What counts as "same place"? (Terminal within
   airport, hotel within resort: relationship, not merge.) What does a wrong merge cost vs a
   missed one? Mailers tolerate errors; justice does not; a wrong booking destination costs
   more than a wrong blog-button suggestion. *(Barney; Jonas — Senzing)*
2. **Profile.** Nulls, skew, junk values ("Sir" as a first name), placeholder values
   (0,0 coordinates, 000-000 phones), typos in the rare tail, constant columns.
   *(Linacre; Zingg 2024; Melnick)*
3. **Standardise.** Case, punctuation, hyphens, spaces, accents (unidecode), abbreviations
   ("St"/"Saint"), codes (M/male/1), schema. Set invalid placeholders to real NULL so they
   contribute nothing instead of "agreeing". Cleaning beats cleverness: a plain edit ratio
   scored 0.16 on reordered tokens; token-sorted, accent-free comparison scored 1.0.
   *(Resnick; Juvenal; Jekal)*
4. **Block (candidate search).** Union of several cheap rules or fingerprints, tuned for
   *recall*. Count comparisons per rule before running; inspect the biggest blocks.
   Alternative: retrieve top-k from a search index (BM25). *(Linacre; Resnick; Kulkarni)*
5. **Compare.** Per field, graded comparison levels (exact → fuzzy bands → else), plus
   distance, numeric and category checks. Model the error types your data really has.
   *(Linacre; Resnick; Barney)*
6. **Score.** Fellegi-Sunter (m/u weights, EM, no labels needed) or a supervised classifier
   (dedupe, Zingg, a decision tree) trained with active learning. *(Resnick; Linacre;
   Juvenal; Goyal; Jekal)*
7. **Decide.** Three outcomes: match / possible match (clerical review) / non-match, plus an
   explicit "ambiguous" hold-out when a record fits two entities equally. *(Barney; Jonas;
   Security & Privacy Academy)*
8. **Cluster.** Pairwise output is intransitive; always cluster. Enforce one-to-one where the
   domain demands it *before* transitive closure; flag giant or weakly attached clusters.
   *(Juvenal; Melnick; Thibaudeau; Goyal)*
9. **Evaluate** on a fixed, stratified gold set: blocking pair completeness, pairwise
   precision/recall, cluster-level checks, per-subgroup error rates. Never accuracy.
   *(Resnick; Bates-Haus; Schnell & Christen; Zingg 2024)*
10. **Fuse and operate.** Build the golden record per cluster, keep lineage (source record →
   entity id), carry IDs forward across runs, process deltas incrementally, monitor drift.
   *(Melnick; Jekal; Jonas; Bates-Haus)*

---

## 2. Principles that recur across the teachers

### The core maths (Fellegi-Sunter)
- **m = P(agree | match), u = P(agree | non-match).** Agreement weight log(m/u), disagreement
  log((1−m)/(1−u)); sum the field weights and add the prior to get a match probability.
  Missing on either side contributes 0. Summing is optimal *if field agreements are
  independent given match status*. *(Resnick)*
- **u is chance agreement,** estimated from random pairs (nearly all non-matches; exclude
  known matches if match prevalence is high). m comes from labels or from EM on unlabelled
  pairs. *(Resnick; Linacre)*
- **Read weights as log2 Bayes factors.** Linacre's waterfall: prior 1/9000 → −13.1; exact
  full name +11.4; fuzzy DOB +3.6; exact postcode +12.3; "West Devon" vs "Devon" −2.6 (a
  missing comparison level, not a real disagreement); total +14.8 → near-certain. Additive
  weights make every decision explainable. *(Linacre)*
- **Three evidence sources per field:** fuzzy similarity of the pair, term frequency (rare
  values agreeing is stronger evidence than common ones), and data quality (how often true
  matches *disagree*: a month typo is expected in hand-keyed data, damning in a reliable
  field). Most systems drop the third. *(Linacre)*

### Respect the independence assumption
- **Nested fields double-count.** Postcode within province, locality within admin1 within
  country: compare the child only if the parent agrees (set the child to missing otherwise);
  keep the parent for when the child is null. Treating them as independent gives "pretty
  poor" linkage. *(Resnick)*
- **Low-information correlated fields partition; they don't score.** Sex barely informs and
  shifts other fields' m; split the linkage into passes by it, duplicating missing values
  into both passes. For places, feature class plays this role. *(Resnick)*
- **Pairwise FS assumes independent pairs, which means one-to-one linkage.** Clusters of 3+
  break it: A–B, A–C, B–C runs can't assert a three-way match and are non-transitive (a triple
  has 5 possible partitions). Practical fix: accept the largest confident clusters first,
  enforce one-to-one greedily, remove them, then run pairwise FS on the rest. *(Thibaudeau)*
- **Check independence before trusting model-based error estimates** (Σ(1−p) over accepted
  links). Do EDA for dependent fields first. *(Resnick; Jekal)*

### Standardise and compare
- **Graded levels, not binary.** Barney / Barny / Barnie / Barnabas / Bernie are different
  levels, each with its own learned weight. Resnick nests them: JW ≥ 0.85, then ≥ 0.90
  conditional on passing 0.85; JW 0.80 can already signal similarity. Custom levels (name
  inversion, token subset) are SQL CASE WHENs. *(Barney; Resnick; Linacre)*
- **Token-order-insensitive metrics for names** (Monge-Elkan, token sort/set); q-grams are cheap
  and robust. Weight token agreement by rarity: "Hotel", "Airport", "Inc" carry little.
  *(Juvenal; Resnick; Security & Privacy Academy)*
- **Nickname/synonym tables only for what metrics miss** (Bill/William, doctor/physician), not
  Rob/Robert. String matchers aren't semantic; use tables or embeddings. *(Resnick; Zingg 2024)*
- **Geocode and use distance** as its own comparison; numbers and units need explicit numeric
  checks because bag-of-words can't separate "2%" from "whole" or "Hotel 81" from "Hotel 18".
  *(Juvenal; Kulkarni)*
- **Few, distinctive, well-populated fields.** Constant or mostly-null columns add nothing; a
  ~70% populated one can still help. Every extra match type costs labels and runtime. Zingg
  treats two nulls as agreement by default; add a null-aware feature if nulls matter.
  *(Zingg 2024)*

### Block for recall
- **Blocking needs recall, not precision;** scoring decides later. Any single key loses matches
  whose key is corrupted, so union several passes and keep the best link. Quality metric =
  *pair completeness* (true matches kept); *reduction ratio* is only compute.
  *(Resnick; Linacre)*
- **Count before you run.** first_name equality alone → ~16M pairs on 50k records; Splink
  counts a rule's comparisons up front to avoid "50 trillion" surprises. *(Linacre)*
- **Big blocks are data-quality bugs.** "Sir" + "Baronet" made ~300k of 625k comparisons. Generic
  tokens, placeholder phones, shared default coordinates become supernodes that kill graph
  queries and laptops. Fix the data or drop the key. *(Linacre; Melnick; Zingg 2024)*
- **Fingerprint families:** sorted tokens, first N tokens, prefix, acronym, phonetic code,
  q-grams, geohash for coordinates, magnitude for numbers; records can emit several.
  Approximate blocking via ANN/LSH. *(Juvenal; Melnick)* Choose passes with a sequential
  coverage algorithm rather than guessing. *(Resnick)*
- **Learned blocking** (dedupe, Zingg) picks predicates that cover labelled matches with the
  fewest comparisons; Zingg reports ~0.05% of the Cartesian product. It is only as good as the
  labels: too few → weak blocks → exploding comparisons and failing jobs. *(Juvenal; Goyal;
  Zingg 2024)*
- **Search is a blocker.** Index the reference set once, query top-k per record, then decide.
  bm25s (precomputed sparse scores) took 1M × 4M product matching from ~11 days to under a
  day: ~2 GB RAM for 4M docs, 50–80 ms/query, memory-mappable. DuckDB FTS worked but used
  20–30 GB and 0.3–0.5 s/query; rank_bm25 took 90–120 s/query. Stem, drop stopwords, persist
  the index, batch queries. *(Kulkarni)*

### Labels and learning
- **Probabilistic beats rule piles.** Deterministic SQL got ~60% of matches; hand fuzzy rules
  ~80–90% after years of arguing rule priority; learned per-level weights match an exhaustive
  rule set without the bikeshedding. *(Barney)* Rules are fine for EDA and rot in production.
  *(Jekal)*
- **Active learning beats random labelling;** random pairs contain almost no matches. Start
  with ~50–60 labelled pairs (2022 talk) or ~40–50 labelled *matches* covering varied
  field-agreement combinations (2024 talk); include hard negatives. Matches teach blocking
  more than non-matches. *(Goyal; Zingg 2024; Juvenal)*
- **Label only what the record shows.** Knowledge that isn't in the data can't be learned and
  makes labels inconsistent. *(Zingg 2024)*
- **Hunt for hidden labels.** A field that nearly guarantees a match when present (even if
  only ~20% populated), plus synthetic variants (character swaps, dropped fields) and identity
  pairs, gives training data without manual labelling. *(Jekal)*

### Decide and cluster
- **Thresholds follow downstream cost;** there is no theoretical best. Too conservative biases
  results (deaths undercounted); forcing exact ID agreement biases the linked population toward
  clean records. Options: minimise estimated type I + II, or weight analyses by match
  probability instead of cutting. *(Resnick)*
- **The clerical band is where labellers disagreed.** Probabilistic scoring shrank a 100k
  manual job to ~100. Log reviews and feed them back. Clerical review alone doesn't scale.
  *(Barney; Jekal; Schnell & Christen; Thibaudeau)*
- **Hold out ambiguity; bias toward false negatives.** A record that fits two entities equally
  (George Foreman's five sons named George) is wrong ~50% of the time if you pick one. Persist
  relationships ("related", "possible match") separately from merges. *(Jonas)*
- **Always cluster.** Clustering adds missed links; hierarchical clustering (`dedupe`) can also
  drop weak ones — clustered pairs had more TPs and one fewer FP than raw pairs. (note: pure
  connected components only adds edges.) *(Juvenal)* Connected components on the edge list +
  group-by to merge is the standard closure step. *(Melnick)*
- **ER is micro-clustering:** millions of tiny clusters (5–6 records per entity in 20M), so
  k-means doesn't fit, and a giant cluster is a bug signal. *(Goyal)*
- **Keep cluster-quality signals:** min/max pair score per cluster (Zingg's z_minScore /
  z_maxScore) flags weakly attached members and chains. *(Goyal)*
- **Use relational context:** resolve strong entities first; "shares an already-resolved
  neighbour" becomes a feature for weak records. *(Melnick)*

### Evaluate honestly
- **Accuracy is meaningless:** "always non-match" scores 99.99%. Use precision/recall/F on
  matches and sample so rare positives appear. *(Bates-Haus; Security & Privacy Academy)*
- **Three error-estimation routes, each flawed:** gold-standard subset (may be unrepresentative,
  e.g. only records with an SSN), model-based Σ(1−p) (needs independence), clerical review
  (humans can't tell common names apart). *(Resnick)*
- **Training accuracy is not production quality.** Keep a fixed golden set and score every
  config change against it (e.g. 80% → 85%). *(Zingg 2024)*
- **Synthetic data is too friendly** and its clusters too small; test on realistic, messy data
  with realistic cluster sizes. *(Schnell & Christen; Bates-Haus)*
- **Error is uneven and propagates.** Minority and transliterated names link worse under
  one-size cleaning; check error per subgroup, tailor strategy per subgroup, and tell
  downstream users the linkage quality. *(Schnell & Christen)*
- **Domain constraints are cheap QA:** one GP per patient (observed max ~50) exposed
  over-merging. Check cluster-size histograms. *(Schnell & Christen)*

### Operate
- **Match records to entities.** An entity's accumulated names/addresses can match a record
  no single member matches. *(Jonas)*
- **ER must self-correct.** New records can re-resolve possible matches, split earlier
  members out, and deletions (GDPR, ageing) must unlearn dependent merges. Without it,
  accuracy drifts (1%/month → 4% wrong after four months) and batch reloads saw-tooth.
  *(Jonas)*
- **Plan deltas and stable IDs from day one.** Judge a system on full-run cost, delta cost
  (Tamr: 500M records under 3 h/$300; a 1% delta under 1 h/$100, vendor claim) and ID
  stability, not pairwise accuracy alone. Re-running from scratch changes IDs and breaks
  consumers. *(Bates-Haus)*
- **Real-time ER is soft real-time:** define a latency budget, pair with eventually consistent
  search/serving stores, alert on consumer lag, emit per-stage telemetry with a routing slip
  to find silently dropped records. Start pairwise + supervised; record-to-cluster later.
  *(Jekal)*
- **Keep lineage:** source record → entity id, so over/under-merges can be debugged and
  explained. *(Melnick)*
- **Start where quality and overlap are high,** or with single-system dedup, for fast,
  checkable wins. *(Barney)*
- **Privacy:** combined quasi-identifiers re-identify people; DP doesn't fit microdata linkage;
  hashed phonetic codes (MD5 of Soundex) are attackable even when "certified"; most PPRL assumes
  honest-but-curious parties. Don't link people who don't want to be linked. *(Security &
  Privacy Academy; Schnell & Christen; Juvenal)*

---

## 3. Checklists

**Blocking design**
- [ ] n²/2 computed; target comparison budget set (laptop: ~10^7–10^8 pairs).
- [ ] Two or more independent passes (geo key; name key; ID key) unioned; each pass alone
      survives a corrupted value in the other's key.
- [ ] Comparisons counted per rule and cumulatively *before* predict.
- [ ] Top block-generating values inspected; generic tokens, placeholders and default
      coordinates fixed or excluded; block-size cap or alert.
- [ ] Grid/cell keys cover cell borders (neighbour cells, or a second offset grid).
- [ ] Pair completeness measured on the gold set (target: very high, e.g. ≥ 0.98, since
      blocking losses are unrecoverable).
- [ ] Learned blocking: enough labelled matches covering field combinations; comparison
      count in logs falling as labels grow.
- [ ] Search-as-blocking: index persisted, stemming/stopwords on, top-k recall measured, a
      decision step after retrieval.

**Comparison and scoring**
- [ ] Placeholders → NULL; nulls contribute 0 (or modelled explicitly).
- [ ] Graded levels per field reflect the real error types (typos, reordering, abbreviations,
      token subsets like "Lagos" vs "Lagos State", transliteration).
- [ ] Nested fields compared conditionally; correlated low-information fields used to
      partition.
- [ ] Term-frequency adjustment on names; generic tokens down-weighted.
- [ ] Distance and numeric comparisons explicit.
- [ ] Match-weights chart read for sanity (signs and sizes make sense); waterfalls checked on
      a handful of borderline pairs.

**Decide and cluster**
- [ ] Thresholds chosen from downstream cost; clerical band defined and sized.
- [ ] Ambiguous candidates (two near-equal best entities) held out, not tie-broken.
- [ ] One-to-one enforced per source where the domain says so, before closure.
- [ ] Cluster-size histogram, max cluster size, min edge score per cluster reported; giant
      clusters alert.

**Evaluation**
- [ ] Gold set fixed, versioned, stratified (country, feature type, script, source) and
      including hard negatives (same name, different place).
- [ ] Blocking pair completeness + pairwise precision/recall/F at chosen thresholds; PR
      curve across thresholds.
- [ ] Pairs derived from clusters evaluated, not just raw scored pairs; a cluster metric too
      (see box in section 7).
- [ ] Error rates per subgroup (country, script, type).
- [ ] Domain constraints asserted (one place per IATA code, one master place per supplier
      listing).
- [ ] Every config/model change re-scored against the same gold set; results logged.

**Operate**
- [ ] Lineage table: source, source_id, entity_id, run_id, rule/score that linked it.
- [ ] Entity IDs carried forward across runs (maximum overlap with previous clusters).
- [ ] Delta path: new record → block against existing entities → score → attach / review.
- [ ] Merges, splits and deletes handled; drift tracked between runs.

---

## 4. Failure modes to hunt for in reviews

- A single blocking key; no comparison count before predict; one generic token creating a
  block of millions.
- Placeholder values ("Unknown", 0,0, "000-000") left as values, so they "agree".
- Summing country + admin1 + postcode as independent evidence.
- Binary fuzzy thresholds instead of graded levels; plain edit ratio on reordered tokens.
- Reporting accuracy; evaluating only raw pairs, never clusters; tuning on the training labels.
- Connected components on all edges with no one-to-one constraint or cluster-size alert
  (chaining A~B~C into one mega-place).
- Picking the nearest candidate when two are equally good.
- Re-running from scratch and issuing new entity IDs each time.
- Synthetic test data with tiny clusters treated as proof of quality.
- One normalisation recipe for every script and country.
- Pure-Python all-pairs or per-query loops over millions of rows.

---

## 5. Where teachers disagree, and how to resolve it

- **Unsupervised FS (Linacre, Resnick) vs supervised/active learning (Goyal, Jekal, Juvenal).**
  FS needs no labels and explains itself; supervised handles dependent fields and needs labels.
  Resolve: start with Splink FS plus a small gold set; switch to (or add) a classifier when
  independence is clearly violated or hidden labels are plentiful.
- **Connected components (Melnick, Linacre) vs hierarchical clustering (Juvenal/dedupe) vs
  staged one-to-one (Thibaudeau).** Components are cheap and transitive but chain;
  hierarchical can drop weak links. Resolve: enforce one-to-one and threshold high, then
  components, then inspect min-edge-score and giant clusters.
- **Hand-written blocking rules (Linacre, Resnick) vs learned blocking (Zingg, dedupe) vs
  search (Kulkarni, Jekal).** Rules are transparent and countable; learned rules need labels;
  search is best when one side is a big reference set and you match many small queries into it.
- **Clerical review: necessary (Barney, Jekal) vs unreliable (Resnick).** Both true: humans
  fail on common names. Use review for the narrow band, give reviewers full context, and
  prefer model-based estimates where independence holds.
- **Batch vs real-time.** Jekal says derive batch from real-time; Jonas says only real-time
  self-correction avoids drift. For a laptop pipeline, run batch plus a delta path, and measure
  drift between runs.
- **Graph or not.** Melnick: graphs help with heterogeneous attributes and relational context,
  but use SQL for static joins and a search engine for fuzzy lookup.

---

## 6. Applying this to Sam's work: 6M GeoNames/OSM places on 16 GB

**Goal:** one master place ID per real place, with lineage, that supplier listings attach to.

1. **Scope and partition.** Partition by feature class rather than scoring it *(Resnick)*:
   populated places/admin areas, transport (airports, stations, ferry terminals), POIs/hotels.
   Each gets its own distance bands and thresholds. Start with the highest-quality,
   highest-overlap slice: cities with population and airports via IATA *(Barney)*.
2. **Free labels first** *(Jekal)*: OSM `wikidata` tags joined to Wikidata's GeoNames ID
   property, OSM `iata`/`ref` tags, GeoNames IDs already in OSM tags. These are positives
   for the gold set and for training. Add hard negatives: same name, same city, different place.
3. **Standardise in DuckDB.** Lowercase, `strip_accents`, punctuation → space, collapse
   spaces, expand abbreviations, sorted-token fingerprint, alternate-name arrays (GeoNames
   alternatenames, OSM `name:*`). Placeholder coordinates → NULL. Per-script normalisation,
   keeping the original *(Schnell & Christen)*. Write Parquet.
4. **Block, at least three passes, counted first:**
   - country + geohash-6 (~1.2 × 0.6 km) *with neighbour cells* (or H3 res 8 with
     `grid_disk(k=1)`) for POIs; geohash-4/5 for cities.
   - country + name fingerprint (sorted tokens, or first token + phonetic code), with generic
     tokens ("hotel", "church", "school", "bus stop") excluded from the key.
   - exact shared ID (wikidata, IATA) where present.
   Inspect the largest blocks; cap them *(Linacre; Melnick)*. Aim for ~10^7–10^8 pairs.
   For supplier → master matching, use a persisted bm25s index over master names + alternate
   names, query top-k (k≈10–20) per listing, then filter by country/distance *(Kulkarni)*.
5. **Comparisons:**
   - name: exact → token-sort/set exact → JW ≥ 0.92 → ≥ 0.88 → token subset ("Lagos" vs
     "Lagos State") → q-gram/Jaccard → else; plus alternate-name array overlap; TF
     adjustment on exact name *(Linacre; Resnick)*.
   - distance: `DistanceInKMAtThresholds` with type-specific bands: POIs 0.1 / 1 / 10 km
     *(Linacre's For-Sam note)*; cities wider, e.g. 5 / 25 / 50 km (reviewer suggestion).
   - admin hierarchy: compare admin1 only when country agrees; admin2 only when admin1 agrees
     *(Resnick)*.
   - category compatibility and explicit numeric checks ("Hotel 81") *(Kulkarni)*.
6. **Train** (Splink, DuckDB): u by random sampling, m by EM, then the prior *(Linacre)*.
   In Splink, estimate the prior from deterministic rules (shared wikidata/IATA) with a
   stated recall, and run EM on two or more different blocking rules so every field gets
   estimated (Splink docs practice). Read the match-weights chart.
7. **Predict with a threshold** so DuckDB never materialises every scored pair; set a DuckDB
   `memory_limit` (~10 GB) and a `temp_directory` on SSD; process country by country if needed.
8. **Decide and cluster:** auto-merge above a high threshold; clerical band below it;
   ambiguous same-name candidates held out *(Jonas)*. Enforce one-to-one per source pair
   before closure (Splink's `cluster_using_single_best_links` with duplicate-free datasets),
   then components; alert on any cluster > ~10 members or containing two different IATA
   codes *(Thibaudeau; Schnell & Christen)*.
9. **Evaluate** on the stratified gold set (pair completeness, P/R, per-country error).
10. **Operate:** lineage table; carry IDs forward by maximum overlap; supplier deltas go
   through the delta path; relationships ("terminal within airport") stored, not merged
   *(Jonas; Bates-Haus; Melnick)*.

---

## 7. Reviewer's notes: corrections and current state (late 2026)

**Corrections applied to the source notes**
- Thibaudeau guessed 24 latent classes for quadruples; a set of 4 has 15 partitions
  (Bell numbers 2, 5, 15).
- `dedupe` clusters hierarchically; connected components never removes an edge.
- In Splink, term-frequency adjustments apply only to exact-match levels (`.configure(
  term_frequency_adjustments=True)`); Linacre's point that rarity matters for fuzzy matches
  too is conceptual. For fuzzy rarity, add a custom level or token-rarity weighting.
- Barney's "~1M records needs nothing beyond DuckDB" is a floor, not a ceiling. Several
  million rows run on DuckDB when blocking is tight and predict is thresholded.
- Tamr's benchmark figures are vendor claims on synthetic (FEBRL-derived) data.
- Zingg's label guidance differs between talks: 50–60 labelled pairs (2022) vs ~40–50
  labelled matches with field-combination coverage (2024). Use the stricter 2024 version.

**Tool status**
- **Splink 5.0** (Sep 2026) breaks Splink 4 code: register inputs with `db_api.register(...)`
  to get a SplinkDataFrame, then `Linker(sdf, settings)`; blocking diagnostics are
  `splink.blocking_analysis.count_comparisons_from_blocking_rules`,
  `chart_comparisons_from_blocking_rules` and `n_largest_blocks`; salting was removed
  (chunking added); `find_matches_to_new_records` was replaced by `predict_between` /
  `score_pairs`; precomputed blocked pairs can be registered with
  `register_blocked_pairs_for_predict` (useful for geohash-neighbour or BM25 candidates).
  `linker.training.*` and `linker.clustering.*` names are unchanged. Check the docs.
- **Zingg** is AGPL-3.0 (other licence on request), runs on Spark (Docker for local use),
  latest 0.7.x. The AGPL matters if Trameter ships it inside a product.
- **bm25s** is the right local BM25; DuckDB FTS is fine for analytics, not 4M+ retrieval on
  16 GB.

**Not covered by the sources, but know this**
- **Embeddings / ANN blocking.** Multilingual sentence embeddings (e.g. LaBSE-class models)
  plus an ANN index (FAISS, hnswlib) catch transliterations and synonyms that BM25 misses.
  Union ANN top-k with lexical blocks; never rely on cosine alone, since nearest neighbours
  always return something. Splink has `CosineSimilarityAtThresholds` for an embedding column.
- **LLM-as-judge for the review zone.** Send only the clerical band, with both records'
  full context (names, coordinates, category, distance), ask for a binary match/no-match plus
  evidence, validate the judge against human labels with a confusion matrix, pin the model
  version, and keep ambiguous cases for humans.
- **Unicode and transliteration.** Apply NFKC + casefold before anything else; `unidecode`
  is lossy for CJK and Arabic. Keep the original name, a romanised form (ICU `Any-Latin;
  Latin-ASCII`) and alternate names, and compare across them.
- **Multilingual place names.** Compare the *sets* of names (array overlap, best-pair
  similarity), not one display name. "Lagos" vs "Èkó" will only match through alternate names
  or coordinates.
- **Golden-record survivorship.** Per attribute, choose by explicit rules: source priority
  (e.g. official gazetteer for admin codes, OSM for POI geometry), recency, completeness,
  majority. Store which source won each attribute.
- **Cluster metrics.** Report pairwise P/R from cluster-derived pairs *and* B-cubed
  precision/recall (per-record, less dominated by big clusters); for ID stability, track the
  share of entities whose ID changed between runs. Splink's `compute_graph_metrics` gives
  per-cluster density and bridge edges for spotting chains.
- **Geospatial keys.** Geohash cells are rectangles with discontinuities at borders and get
  narrower toward the poles; H3 hexagons have uniform neighbours. Either way, include
  neighbour cells.

---

## 8. Where to look in `references/`

| Need | File (see INDEX.md) |
|---|---|
| FS beyond pairs (triples, one-to-one, staged resolution), Zingg training and labels, real-time ER with Kafka/Elasticsearch, BM25 search-as-blocking with bm25s, PPRL research gaps and subgroup error, Senzing's 10 moves (re-resolve, split, ambiguity, delete), Tamr on why ER at scale is hard | `er-1-thibaudeau-zingg-jekal-kulkarni-christen-senzing.md` |
| Fellegi-Sunter maths and extensions (Resnick), Spark + graph ER (Melnick), Splink deep dive (Linacre), Splink in practice (Barney), Python dedup with `dedupe` (Juvenal), open-source Zingg (Goyal), record linkage overview and privacy | `er-2-resnick-melnick-linacre-barney-junior-goyal.md` |
