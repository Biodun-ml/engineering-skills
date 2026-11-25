---
name: search-ranking-recsys-first-principles
description: Search, ranking and recommender playbook from 17 talks (Eugene Yan, Doug Turnbull, Jo Bergum, Booking.com, Andrew Ng, Shipt, Elastic, ritvikmath). Use whenever building or tuning search, BM25, embeddings/ANN, hybrid search, query parsing, learning to rank, recommendations or relevance evals.
---

# Search, ranking and recommendations from first principles

This skill turns 17 talks and lectures into working judgment for product search, ranking and
recommendations: architecture, two build procedures, recurring principles, checklists and a
Trameter design. `references/` (when present) holds credited notes for every video.

Use it whenever results get ordered for a user or an agent, even if nobody says "search":
a search box, autocomplete, a fuzzy place lookup, "similar activities", a carousel, matching
articles to products, a tool an agent calls, or "why is this result on top?".

**How to use it**
1. Locate the problem in the retrieve → filter → score → order architecture (section 1).
2. Follow procedure 2 (v1 search) or 3 (ranking/recommendations); don't skip to models.
3. Run the checklists in section 5 before trusting a metric or shipping a change.
4. If a `references/` folder sits next to this file, use it for depth. `references/INDEX.md`
   lists every video with its line number, or run `grep -n -i "keyword" references/*.md`.
   Each section ends with a "Relevance to Sam" or "For Sam" note: Sam builds Trameter's
   location data (~6M GeoNames rows), Viator/RateHawk supplier integrations, the Drive Button
   pipeline that turns travel blog articles into bookable trips, and "where can I go"
   passport-based discovery, all on a 16 GB laptop. Use those parts only in that context.
   Lines marked "(note: …)" and those Sam notes are helper additions, not the speaker's
   words. If `references/` is missing, the full notes live in the user's private GitHub repo
   `Biodun-ml/engineering-skills` (folder `search-ranking-recsys-first-principles/references/`);
   clone it when you need depth.
5. Explain the *why*, and credit the teacher when you use a specific pattern.
6. Library defaults, engine features and model names date quickly; the 2018–2019 talks
   especially. Check current docs before recommending a specific tool or setting.

---

## 1. The architecture almost every system shares

- **Two stages named by their goal: recall, then precision.** Retrieval cuts millions of items
  to hundreds–1,000 cheaply and coarsely; ranking scores those with rich user, item and context
  features and shows the top 20–50. DoorDash literally labels the stages "recall" and
  "precision". *(Eugene Yan — system design; DataTrek; Parmar)*
- **Four stages in practice: retrieve → filter → score → order.** Filter drops invalid
  candidates (unavailable, ineligible, already bought) *before* expensive scoring; order holds
  business logic such as diversity (no Harry Potter 2–7 in a row). *(Yan, citing NVIDIA)*
- **Offline builds the artifacts, online serves them:** embedding model, ANN index, feature
  store, ranker; logs flow back into training. Search adds query processing in front
  (gentle spell-check, synonyms, expansion — DoorDash). *(Yan)*
- **Batch first.** A daily job writing results to a key-value store is an MVP in weeks; if
  today's job fails you serve yesterday's results, with almost no on-call. Go real-time only
  where intent shifts within a session (news, short video, shopping missions, brand-new
  users), and dogfood it to measure the gain first. Yan names travel as session-driven: the
  next trip is often not predicted by past ones. *(Yan)*
- **Route before you rank.** Which vertical or collection should answer (destination vs
  activity vs hotel) is a classification problem upstream of ranking, because ranking scores
  are not comparable across queries. *(LTR panel)*

---

## 2. Procedure: build a v1 search for a catalogue

1. **Check the surface matters.** Log impressions and clicks per slot before anything else.
   Booking spent ~6 months tuning hotel ranking in a homepage module almost nobody clicked;
   hiding it changed nothing. *(Lukas Vermeer)*
2. **Know the traffic shape.** Pull the query log: head/torso/tail share, query length,
   repeat behaviour. At Shipt ~50% of searches were the top 1,000 queries and 30% of
   purchases were repeats, so they fixed the head first and eyeballed ~1,000 head queries.
   *(Dipak Parmar)*
3. **Model the documents for search, denormalised.** One flat document per searchable thing
   with the fields users query (name, alternate names, category, place, description) and the
   numeric signals you will boost on. Parent-child joins hurt Shipt's performance; duplicating
   data in an index is usually fine. *(Parmar)*
4. **Text analysis first:** lowercasing, synonyms, stemming, stop words, special characters,
   shingles plus concatenated shingles ("ice cream" = "icecream"), n-grams for autosuggest.
   Phrase-match compound names. Fuzz only misspelled tokens; blanket fuzzy is expensive.
   *(Parmar)*
5. **BM25 per field, weighted sum, then tune k1 and b on your data.** Short fields (names,
   URLs) want different settings from body text. Compare several rank profiles side by side
   on real queries, then on a judged set. Tuning k1/b alone moved BrowseComp-Plus retrieval
   ~30 points over defaults tuned for short uniform passages. *(Abhishek Thakur; Turnbull &
   Bergum)*
6. **Query understanding from your own catalogue, outside the engine.** Build dictionaries of
   brands, categories, places and sizes from the catalogue; tag query tokens as entity,
   modifier, negative intent ("egg free" → must_not), compound phrase, ignorable ("hot
   coffee" may drop "hot", "hot dog" may not). Main entity = must, modifiers = should,
   compounds = phrase; auto-filter when a type or brand is detected. *(Parmar)*
7. **Add numeric signals as saturated boosts.** score = BM25 + Σ w·f(signal), f = saturation,
   log or sigmoid with a pivot that means something (score halves at 1 km, at 7 days, at
   median popularity). Text match gates; signals only add to docs already matched.
   *(Mayya Sharipova; Parmar)*
8. **Add dense retrieval only for the vocabulary gap.** Relevant docs can share zero words with
   the query; that is what embeddings are for. Keep BM25 as the lexical half. *(Christopher
   Potts; Thakur)*
9. **Always keep an escape hatch.** Structured filters only when confident; fall back to
   plain lexical/vector search; allow zero results. *(Doug Turnbull)*
10. **Make it debuggable and measured.** Return each signal's score contribution per result;
    A/B "base only" vs "base + ranking"; judged set offline (section 5), searches per session
    and median conversion position online. *(Parmar; Mohan)*

---

## 3. Procedure: add ranking and recommendations

1. **Write the hypothesis and the action ladder.** Observed problem → cause → change →
   expected behaviour → metric, fixed before building. Write the value ladder (impression <
   click < add-to-cart < booking); it picks your labels and metric. *(Vermeer; ritvikmath)*
2. **Start with label-free, cheap baselines.** "Rule 1 of ML: don't start with ML." Popular
   items, "popular in the category this user favours", "buy it again", item-item
   co-occurrence counts, word2vec over session sequences, logistic regression. Production
   systems lag academic SOTA on purpose. *(Yan; Karthik Mohan)*
3. **One carousel at a time.** Each widget ("because you viewed X", "trending in [city]") is
   its own task; whole-page optimisation comes later. *(Mohan)*
4. **Handle cold start from day one.** New users fall back to item averages or popularity
   (mean normalisation); new items need content features or semantic IDs. *(Andrew Ng; Mohan;
   Yan)*
5. **Build point-in-time features.** Each feature takes the value it had when the event
   happened (bookings *so far* at search time, not today's count). Store features as
   time-stamped change logs. *(Ottens & Falsina, Booking)*
6. **Start with user-interest features,** add seller/business features later and carefully;
   adding all at once made one client's results worse. More features need deeper judgments
   per query. *(LTR panel)*
7. **Train as a grouped problem.** Carry a query/session ID, split train/test by query, use a
   pairwise/listwise objective (e.g. LambdaMART-style lambdarank), evaluate per query.
   *(ritvikmath — LTR; names of methods are field standard)*
8. **Ship model + transforms + feature list as one unit** so online scoring is identical to
   offline. *(Ottens & Falsina)*
9. **Order stage last:** diversity, repeat-purchase slots, sponsored items in a separate
   post-processing service over the top-K. *(Parmar; Yan; Mohan)*
10. **Hold out a control forever and A/B on business metrics,** with a power-calculated run
    length you stick to. Plan retraining (every 3–6 months, recency-weighted) and refreshed
    popularity features against drift. *(Vermeer; LTR panel)*

---

## 4. Principles that recur across the teachers

### Lexical retrieval and BM25
- **BM25 = IDF × saturating TF × length normalisation, per field.** A term appearing 50 times
  is not worth 5× one appearing 10 times. k1 sets how fast TF saturates (common default ~1.2,
  range ~1.2–2.0); b sets length normalisation (0 = ignore, 1 = full; Lucene, Elasticsearch
  and Vespa default 0.75; Thakur cites 0.7–0.9). *(Thakur)*
- **An inverted index maps term → scored documents,** built offline and looked up at query
  time. Indexing vs scanning is only a performance trade-off: BM25 can be computed by scan
  with a top-k heap; an index pays off for repeated queries under a time budget. *(Potts;
  Bergum)*
- **Speed comes from skipping.** Lucene stores per-block "impacts" so block-max WAND skips
  blocks that can't reach the top-k (up to ~10× on some queries). rank_feature values get this
  for free; scripts and brute-force vectors don't, so filter first. *(Sharipova)*
- **Don't throw ML at what the engine does well.** The engine does text similarity faster and
  cheaper; ML takes longer to iterate. Equally, don't solve everything in the engine.
  *(Parmar)*

### Dense retrieval, two towers and ANN
- **Two towers scale because they are independent until the dot product.** Precompute item
  vectors offline into FAISS/ANN; encode only the query/user online; L2-normalise so dot =
  cosine; keep output small (16–64 dims). *(DataTrek; Yan)*
- **Start from pretrained text embeddings and fine-tune on conversions.** Raw embeddings don't
  know what converts or which field (brand vs colour) should dominate. *(DataTrek)*
- **Budget memory:** rows × dims × 4 bytes float32 (6M × 384 ≈ 9.2 GB; 6M × 32 ≈ 770 MB).
  ANN trades recall for speed: 0.5 is too low, 0.99 is hard; benchmark libraries on your
  data (ScaNN won Yan's test). *(Yan; Sharipova; arithmetic from the Sam notes)*
- **Pure vector search has structural weaknesses:** generic models collapse in-domain items
  together (blue couch vs brown table ~0.45 cosine), don't say *which* attribute made things
  similar, and no single score threshold works for broad and specific queries. kNN always
  returns something. *(Turnbull; Sharipova's "searching directories" hit for a zip query)*
- **Inspect nearest neighbours by eye at every stage,** but measure recall@K of retrieval on
  held-out positives; eyeballing is not evaluation. *(DataTrek, with note)*

### Blending signals (hybrid scoring)
- **Saturate every unbounded signal** (raw reviews, popularity swamp text); for geo/time use
  decay, e.g. distance_feature = pivot / (pivot + distance). *(Sharipova)*
- **BM25 has no fixed scale.** Calibrate other signals against its observed range (Shipt saw
  ~1,000–5,000). *(Parmar)*
- **Let context set the weights:** text weighted higher for new stores/items and long
  queries; popularity higher for mature inventory and short head queries ("milk"). Found by
  A/B tests. *(Parmar)*
- **Per-(query, item) conversion scores** make ranking self-improving ("string cheese"
  converts on "string cheese", not on "cheese") — but this is a feedback loop: new items need
  exploration. *(Parmar, with note)*
- **Bake quality into ANN** by concatenating a quality vector (rating, freshness, conversion)
  to item vectors and padding query vectors with a constant (Etsy: +2.6% conversion).
  *(Yan — keynote)*

### Query understanding
- **QU = map free text to a controlled vocabulary,** then filter, boost or route. Tools in
  rising cost: hand rules → click group-by (promote the most-clicked category per query; among
  Turnbull's best Shopify experiments) → supervised classifier → tiny embedding index over the
  label set (also snaps hallucinated brands to real ones) → small LLM with an enforced schema.
  *(Turnbull)*
- **Schema is prompt.** Readable field names and enum values, never opaque IDs; always an
  "unknown" option (else "leather couch" → "brown"). Prefer the narrowest explicit task and
  written guidelines over many few-shot examples, which can overfit. *(Turnbull)*
- **QU needn't run at query time.** Precompute for seen queries nightly with batch APIs, cache
  as a dictionary, fall back on a miss; an inline ~300 ms LLM call is usually unacceptable in
  site search (fine in RAG). Smallest model, no reasoning; needing a big model means the task
  is decomposed wrongly. Refresh the cache as vocabulary drifts. *(Turnbull)*
- **Content enrichment mirrors QU:** if queries mention entities, extract them onto
  documents too. An agent can also explore structured queries offline; save the winner per
  query and replay it in site search. *(Turnbull)*

### Learning to rank
- **Ranking is a grouped problem.** Only order within a query matters; scores aren't
  comparable across queries, so don't threshold them globally. Pointwise, pairwise
  (RankNet/LambdaRank), listwise (LambdaMART/ListNet). *(ritvikmath; method names from the
  field)*
- **LTR learns labels, not features.** Its value is demoting things that *look* relevant; you
  need features separating apparently-relevant from truly-relevant. *(LTR panel)*
- **A query is keywords in a context** (segment, platform, time, passport). Personalise with
  intersection features (user's top brands ∩ item brand), not raw high-cardinality IDs.
  Over-segmenting kills training data: Wikimedia drops ~60% of click logs for lack of
  repetition, and NDCG kept improving until ~30–40M observations. *(LTR panel)*
- **Click labels need repeated queries and bias handling;** humans find the great result
  buried on page 3 that clicks never reveal. Blend human and click judgments. *(LTR panel)*
- **Drift is the fatal flaw:** intent shifts (Blade Runner 1982 → 2049). Refreshed
  popularity/recency features, time as context, retrain every 3–6 months, filter bots (the
  same query 500×/minute). *(LTR panel)*
- **Pool models first, split by segment** only when feature semantics differ (Snag split
  desktop/mobile/app). Splitting needs a router that can itself err. *(LTR panel)*

### Recommenders and cold start
- **Collaborative filtering = low-rank matrix factorisation:** learn item features and user
  preferences jointly from sparse interactions; learned item vectors give "similar items" by
  distance. Initialise randomly to break symmetry. *(Andrew Ng)* (note: his 2010 lecture uses
  explicit ratings; modern systems use implicit feedback with ALS/BPR or two towers.)
- **Mean-normalise or use biases** so a new user gets item averages instead of all zeros.
  *(Ng)*
- **Content-based handles cold items;** hybrids/two towers train on behaviour + content so the
  space reflects both. Most real inventories start cold. *(Mohan; DataTrek)*
- **Item-item counts are a real-time recommender without a neural net:** weight users by
  1/(items they touched); counts live in a KV store and update incrementally. Alibaba's Swing
  trusts pairs co-touched by dissimilar users more. *(Yan)*
- **Semantic IDs fix cold start and sparsity.** Cluster frozen content embeddings (Kuaishou:
  k-means over ~100M videos → ~1,000 IDs with trainable embeddings; +3.6% cold-start
  coverage), or RQ-VAE into hierarchical codes (e.g. 4 levels × 256) that an LLM can generate.
  Use stop-gradients (else the codebook collapses), monitor codebook usage and ID uniqueness,
  clean item text first, add a co-occurrence loss so IDs reflect behaviour (Google PLUM).
  Validate every generated ID against the catalogue; over-generate and drop misses.
  *(Yan — keynote and paper club)*
- **Beyond relevance:** diversity (use embeddings in reverse to pick dissimilar items),
  freshness, fairness across user and item groups, long-tail coverage, latency. Separate
  re-rank passes per objective are fine. *(Mohan)*
- **LLM labelling cascade, quality over quantity:** expert seed labels → strong LLM →
  fine-tuned mid model → distilled tiny classifier for real time; pick the metric that
  matches the harm. Indeed filtered ~20% of bad recs: applications +4%, unsubscribes −5%. Use
  conventional data first, LLMs to augment (Spotify LLM query recs: +9% exploratory queries).
  *(Yan)*
- **The reward is half the system.** What counts as a positive defines what the model learns.
  *(Yan; DataTrek)*

### Agentic search
- **The agent is the new searcher.** Agents send long (~23 terms), keyword- and phrase-heavy,
  web-syntax queries (site:, year ranges) and reformulate over ~25 calls. A simple
  interpretable retriever plus an agent that reformulates beats one-shot retrieval. Dense
  models trained on natural questions underperform on agent keyword queries unless rewritten.
  *(Turnbull & Bergum)*
- **Match the tools models were trained on:** one well-named search tool or a grep/file-like
  interface translated to a real ranked engine. Renaming a tool changed accuracy. Scored
  top-k beats unscored grep for context. Use search sub-agents to keep context clean.
  *(Turnbull & Bergum)*
- **Precision/recall *is* context engineering;** one or two bad results can derail a long
  trajectory. Build your own index for the questions you care about rather than federating
  silo lookups. *(Turnbull & Bergum)*
- **Auto-research on ranking code needs a holdout,** or the agent writes `if query == X`
  hacks. *(Turnbull & Bergum)*

### Evaluation and experiments
- **Pick the metric from the behaviour you value:** MRR for one right answer (known-item,
  navigational), MAP for binary multi-answer lists, NDCG for graded labels; recall@K for the
  retrieval stage. Report more than one and study disagreements. *(ritvikmath; Mohan)*
- **NDCG normalises per list against that list's own ideal,** so [click, none, watch] (0.69)
  scores below [click, none, click] (0.92). Compare across segments with care; report DCG too.
  *(ritvikmath)* (note: textbook AP divides by all relevant items, not only those retrieved.)
- **Classifier gains may not move NDCG:** you may be improving queries that already worked.
  Target segments that hurt the downstream metric. Track precision + coverage for QU.
  *(Turnbull)*
- **Offline–online gap is real:** large held-out tests including new items → dogfooding →
  randomised A/B tests on business metrics, which make the launch call. *(Mohan)*
- **Online diagnostics:** conversion per search, searches per session (rising = failure),
  median position of conversions, comparable keyword mix across arms. *(Parmar)*
- **Never switch off the control.** Pre-register the analysis and power-calculated duration;
  no peeking. Cross-check metrics with an independent pipeline (Booking runs two per
  experiment). Big assumptions deserve tests, not just big code changes. *(Vermeer)*
- **Latency is relevance:** a model 4% better but 2 s slower may lose; ship fast behind
  feature flags. *(Turnbull)*

### Production
- **Point-in-time correctness and identical transforms** (serialised mean/std, buckets,
  encodings) prevent leakage and train/serve skew. Online feature construction is often the
  slowest step. Monitor input distributions, not only outputs. Keep a feature registry (name,
  owner, offline and online source). *(Ottens & Falsina)*
- **Out-of-core on small hardware:** sample to disk, stream 1–3 passes (Vowpal Wabbit style).
  *(Ottens & Falsina)*
- **Engine hygiene:** raise refresh_interval (1 s → 30–60 s) on write-heavy indices to protect
  read p99; split QU into its own service. *(Parmar)*
- **Unified models** (Netflix UniCoRn: shared schema, task as input, item title imputed as
  the query) cut duplicate pipelines but carry an alignment tax: expect 2–3 models. *(Yan)*

---

## 5. Checklists

**Relevance evaluation**
- [ ] Surface proven to matter: impressions and clicks per slot logged; "hide it" tested.
- [ ] Value ladder written (view < click < book) and metric chosen from it: MRR / MAP /
      NDCG@k for ranking, recall@K for retrieval, precision + coverage for QU.
- [ ] Judged set of real queries (head + torso + tail, broad + specific, misspelled),
      labels from humans/LLM judges *and* clicks, split by query.
- [ ] Tuned BM25 baseline in every comparison.
- [ ] Each stage measured separately (QU → retrieval recall → ranking → order).
- [ ] Per-segment results, not just the average; NDCG compared only within comparable lists.
- [ ] Zero-result rate tracked; nearest neighbours eyeballed; holdout untouched by anyone
      (or any agent) tuning the ranker.

**Before shipping a ranking change**
- [ ] Hypothesis, expected behaviour and metric written before launch.
- [ ] Offline judged set shows no regression on head queries.
- [ ] Features are point-in-time correct; transforms shipped with the model; feature list
      versioned.
- [ ] Latency p99 measured against the budget.
- [ ] Per-signal score contributions visible for debugging.
- [ ] Filter stage removes unavailable/ineligible items; order stage adds diversity.
- [ ] A/B with a control kept on, power-calculated duration, no peeking, comparable query
      mix across arms; searches/session and median conversion position watched.
- [ ] Feature flag, rollback, input-distribution monitoring; exploration for new items.

---

## 6. Failure modes to hunt for in reviews

- Tuning ranking on a surface nobody uses; comparing against an untuned BM25.
- Pure vector search with a global similarity threshold and no lexical fallback.
- Inline LLM query understanding that could be precomputed and cached.
- LLM-generated IDs, URLs, prices or visa rules not validated against your own store.
- Features computed with today's values (leakage); serving transforms rewritten by hand.
- Raw popularity or review counts added linearly; global thresholds on LTR scores.
- Train/test split by row instead of by query; over-segmented personalisation.
- No control group "so everyone gets the best"; real-time infra before batch proved value.
- Blanket fuzzy matching; parent-child joins for per-supplier offers.

---

## 7. Where the teachers disagree, and how to resolve it

- **Cross-encoder rerankers.** Bergum calls standalone rerankers a dead end below eBay/Yahoo
  scale (let the LLM or agent judge), and labels it opinion; most RAG practice outside these
  talks treats a reranker as standard. Resolve with your eval: a cross-encoder over the top
  50–100 is cheap to try.
- **Latency.** Search veterans default to single-query, millisecond thinking (Shipt p99 ≈
  70 ms; Turnbull calls a ~300 ms inline LLM too slow); Turnbull & Bergum argue many domains
  can afford ~5 s of agentic multi-query search. Resolve per surface: typeahead and listings
  are fast paths; "plan my trip" assistants can be slow and agentic.
- **Vectors vs structure.** Potts favours neural IR to bridge vocabulary gaps; Turnbull
  favours explicit structured attributes and simple tools. Resolve: hybrid, with structured
  filters as the precision path and dense retrieval for the vocabulary gap.
- **Unified vs separate models.** Yan reports unified models matching specialists (UniCoRn);
  the LTR panel split models when segment semantics differed (Snag). Start pooled, split only
  when a segment measurably suffers.
- **Self-improving signals vs bias.** Parmar's conversion scores "learn by themselves"; that
  same loop entrenches popularity and position bias. Keep the signal, add exploration and
  bias correction.

---

## 8. Not covered by the sources, but know this

*Reviewer additions from current practice (late 2026), not from the 17 talks.*
- **Hybrid fusion.** Reciprocal rank fusion, score = Σ 1/(k + rank) with k ≈ 60, fuses BM25 and
  vector lists without calibrating score scales. OpenSearch, Vespa, Typesense, Meilisearch
  and Weaviate ship hybrid modes; Elasticsearch's RRF retriever has sat in a paid licence
  tier, so check before relying on it. Tune the blend (or a weighted linear blend of
  normalised scores) on your judged set; RRF is a sane default, not optimal.
- **Cross-encoder reranking.** Score (query, doc) jointly over the top 50–200 candidates;
  far more precise than a bi-encoder, too slow for the whole corpus. Open multilingual
  rerankers (e.g. BGE-reranker family) run on CPU at small k; hosted ones exist too.
- **Embedding choice.** Check the MTEB leaderboard on retrieval tasks in your languages, not
  the overall score. For multilingual queries ("Èkó", French/Arabic place names) use a
  multilingual model (multilingual-E5 / BGE-M3 class). Matryoshka-trained models let you
  truncate dimensions cleanly; int8 or binary quantisation cuts memory 4–32× with modest
  recall loss (rescore the top candidates in full precision). Train two towers with
  in-batch/sampled softmax, hard negatives (same city, wrong activity) and a popularity
  correction, not BCE on random negatives.
- **Typo tolerance.** Lucene/ES `fuzziness: AUTO` allows 0 edits for 1–2 character tokens,
  1 for 3–5 and 2 for 6+, a good default; character n-gram or trigram fields catch partial
  matches; skip fuzz on the first character or two for speed. Add phonetic or
  transliteration keys for names. Prefix/edge n-grams power autocomplete.
- **Geo-aware ranking.** Filter by bounding box or admin area first, then decay by distance
  (Gaussian/exponential/linear decay or distance_feature); scale the pivot to the query
  (a city query tolerates kilometres, "near me" tolerates hundreds of metres). Disambiguate
  homonyms ("Paris") with a prior on population/importance plus user location.
- **Position bias and click models.** Clicks favour top slots. Correct with inverse propensity
  weighting (estimate propensities by randomising top results briefly), or click models
  (cascade, DBN), or train with position as a feature and set it to a constant at serving.
  Never train directly on raw CTR by position.
- **Interleaving.** Merge results from rankers A and B into one list (team-draft) and credit
  clicks to the ranker that supplied them; published results (Chapelle et al., Netflix) show
  it detects preferences with roughly 10–100× less traffic than an A/B test. Ideal for
  low-traffic teams; confirm big wins with an A/B on bookings.
- **Popularity bias and feedback loops.** Ranking by past clicks shows popular items more,
  which earns them more clicks. Reserve exploration slots (epsilon-greedy or bandits) for new
  and long-tail items and track catalogue coverage.
- **Engines for a small team.** DuckDB FTS: BM25 in SQL over Parquet, ideal for offline
  evals and batch matching; the index must be rebuilt when the table changes. Tantivy (Rust,
  Python bindings): fast embedded BM25, no server. Typesense / Meilisearch: single binary
  with typo tolerance, facets, geo and hybrid built in; fastest to a good typeahead, less
  scoring control. OpenSearch / Elasticsearch: full control (rank_feature, decay, kNN, LTR
  plugin) at higher ops cost. Vespa: most ranking expressivity, steepest curve. Postgres
  pg_trgm + pgvector covers modest scale. Check current versions before choosing.

---

## 9. Applying this to Sam's work (Trameter design)

**Destination and activity search (Viator/RateHawk + ~6M places)**
- Route first: classify the query as place, activity, hotel or visa/discovery; a place
  gazetteer tagger ("boat tour Lagos" = activity type + place) built from GeoNames names,
  alternate names and admin hierarchy. *(Parmar; LTR panel)*
- Place index: fields `name` (phrase + edge n-grams), `alt_names`, `admin_path`, `country`;
  BM25 with low b on short name fields (helper suggestion ~0.2–0.4, tune it), plus saturated
  boosts on log(population) or the existing place importance score, feature class, and a
  distance decay from the user's or article's location. Fuzziness only on misspelled tokens.
  Metric: MRR and recall@10 on query → correct place ID. *(Thakur; Sharipova; ritvikmath)*
- Activity/hotel index: denormalised docs per product (title, category, city, place ID,
  price, rating, review count, duration); filter on availability/date/geography before
  scoring; boost saturated rating and bookings; diversity in the order stage (not six walking
  tours of one city). Metric: NDCG@10 with view/click/book grades. *(Parmar; Mohan)*
- QU: small model with a readable enum schema {destination, activity type, dates, budget,
  travellers}, precomputed nightly for seen queries, cached, BM25 fallback on a miss.
  *(Turnbull)*
- On 16 GB: Tantivy or DuckDB FTS for lexical; FAISS with 32–64-d or int8 vectors for dense;
  filter by country/admin area before vector scoring. *(Sharipova; DataTrek)*
- Expose one well-named, forgiving, API-first `search` tool for agents: travel is the typical
  complex agent task, and agents return to providers whose search satisfies them ("inverted
  SEO"). Treat retrieved text as untrusted. *(Turnbull & Bergum)*

**"Where can I go" (passport-based discovery)**
- Query = intent + passport + home city. Passport/visa eligibility from the travel-access
  database is a hard FILTER stage; visa ease and refusal odds become query-dependent features,
  not a separate model. *(Yan — filter stage; LTR panel)*
- Parse "beach in December, no visa, Nigerian passport" into structured filters with an
  "unknown" option and a fallback; never let an LLM state visa rules from memory — retrieve
  them and show the source. *(Turnbull; Potts)*
- Score with saturated features (visa ease, cost/distance from home, season fit, budget
  fit, popularity); order for diversity across regions and budgets. Log impressions/clicks
  per suggestion and keep a holdout. *(Sharipova; Mohan; Vermeer)*

**Drive Button (blog article → bookable trips)**
- Treat it as retrieve → filter → score → order: extract places and activities from the
  article as structured attributes (content enrichment), snap places to canonical IDs via an
  alias/embedding index, retrieve products near those places (BM25 + dense hybrid), filter
  on availability and geography, rank, then diversify. *(Turnbull; Yan)*
- Prefer attribute filters + ranking over raw article-to-product embedding similarity with a
  fragile threshold; allow "no good match, show nothing". *(Turnbull)*
- Labels: LLM judgments of (article, trip) fit validated against Sam's own labels, then
  distil to a small reranker for bulk scoring. Optimise precision: bad buttons erode reader
  trust like bad recs drove Indeed unsubscribes. *(Yan)*
- Measure button impressions → clicks → bookings separately, keep articles without buttons
  as a control *(Vermeer)*, and give new listings content or semantic-ID clusters *(Yan)*.

---

## 10. Where to look in `references/`

| Need | File (see INDEX.md) |
|---|---|
| System design for recsys/search, matrix factorisation and cold start, LLM query understanding, MRR/MAP/NDCG, numeric features and decay in Elasticsearch | `search-1-yan-ng-turnbull-ritvikmath-sharipova.md` |
| Booking.com experiments, agentic search, LTR panel (features, drift, segmentation), Booking ML platform (point-in-time features), Yan's LLM-era recsys keynote, BM25 k1/b | `search-2-vermeer-turnbull-ottens-yan-thakur.md` |
| Recommender basics and design, Shipt query understanding and ranking stack, semantic IDs (RQ-VAE, PLUM), two-tower retrieval, Potts on IR, learning-to-rank framing | `search-3-mohan-parmar-yan-datatrek-potts-ritvikmath.md` |
