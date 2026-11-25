# search-1 — cross-cutting principles

1. **Two-stage funnel is universal**: fast, coarse, high-recall retrieval (millions -> hundreds/1,000) then slow, precise ranking with rich user/item/context features. Name the stages by their goal: "recall" then "precision". (Eugene Yan)
2. **Four stages in practice: retrieve -> filter -> score -> order.** Drop invalid candidates (unavailable, ineligible, already bought) before expensive scoring; put business logic such as diversity in a separate ordering layer. (Eugene Yan, citing NVIDIA)
3. **Offline builds the artifacts, online serves them**: embedding model, ANN index, feature store, ranker. Offline jobs can fail softly; online paths need 24/7 on-call. (Eugene Yan)
4. **Start with batch, add real time only where intent shifts within a session** (news, short video, shopping missions, cold-start users). Measure the gain before paying the cost of running it. Travel "next destination" is named as session-driven. (Eugene Yan)
5. **Simple, label-free baselines win first**: item-item co-occurrence counts (which can be updated incrementally), word2vec over interaction sequences, logistic-regression rankers. Production systems are usually years behind academic SOTA, and on purpose. (Eugene Yan)
6. **Collaborative filtering = low-rank matrix factorization**: learn item features and user preferences together from sparse interactions. Learned item vectors also give you "similar items" by distance. (Andrew Ng)
7. **Cold start needs a fallback prior**: mean-normalize so new users get item averages or popularity; use content features for new items. (Andrew Ng; Eugene Yan on real-time profiling of new users)
8. **Query understanding = map free text to a controlled vocabulary of attributes**, then filter, boost or route. Tools in rising order of cost: rules, click group-by, a classifier, a tiny embedding index over the label set, then a small LLM with an enforced schema. (Doug Turnbull)
9. **Pure vector search has structural weaknesses**: hubness/collapse in generic models, no say over *which* attribute drives similarity, and no score threshold that works across broad and specific queries. Structured attributes fix this, and agents can read them too. (Doug Turnbull; Sharipova's irrelevant semantic hit)
10. **Precision path + escape hatch**: only take the structured path when you're confident, and always keep a lexical/vector fallback. Allow "unknown". (Doug Turnbull)
11. **"Query understanding" doesn't have to run at query time**: precompute it for seen queries in nightly batch jobs (batch APIs), cache it as a dictionary, and fall back on a miss. Use the smallest model and no reasoning. If the task needs a big model, the task is probably decomposed wrongly. (Doug Turnbull)
12. **Evals define the task**: hand-label examples before prompting. Track precision + coverage for the classifier, and check the downstream ranking metric (NDCG), because classifier gains may land on queries that already worked. Prefer written guidelines to many few-shot examples, which can overfit. (Doug Turnbull)
13. **Choose the metric from the behaviour you value**: MRR for one right answer, MAP for binary multi-answer lists, NDCG for graded labels (view < click < book). (ritvikmath)
14. **NDCG is normalized per list against that list's own ideal**, so a list with a more valuable item placed low can score below a list of weaker items. Be careful comparing NDCG across lists or segments. (ritvikmath)
15. **Combine text relevance and numeric signals linearly through saturating transforms**: score = BM25 + Σ w·f(feature), with f = saturation/log/sigmoid and a pivot that has a human meaning (e.g. halves at 1 km or 7 days). Text match gates (must) and features boost (should). (Mayya Sharipova)
16. **Speed comes from skipping, not from faster scoring**: block-max impacts let Lucene skip blocks that can't make the top k (up to ~10x faster). Static features indexed as term frequencies get that for free; scripts and brute-force vectors don't, so filter first. (Mayya Sharipova)
17. **Budget memory explicitly**: vectors cost rows × dims × 4 bytes in float32. Small embeddings (32–64 dims) and quantized ANN indexes are standard in industry. (Eugene Yan; Mayya Sharipova)
18. **Ship to production fast, keep people on call, and use feature flags**. Latency is part of relevance: a model that is a bit better but much slower can lose. (Doug Turnbull; Eugene Yan on starting simple)

## Eugene Yan — System Design for Recommendations and Search (MLOps Meetup #78) (https://www.youtube.com/watch?v=lh9CNRDqKBk)
**Almost every production recsys/search system is the same 2x2: offline vs online, crossed with fast-coarse retrieval then slow-precise ranking; start with batch and simple models, add real-time only where users' intent shifts fast.**

**Mental models / first principles**
- The interesting part of recsys/search is the system around the model, not the model architecture. Papers obsess over neural CF and transformers; ~90–95% of real systems share one simple skeleton.
- Batch vs real time is a product decision. Batch: a big Spark job runs daily, results go into a key-value store, the app just looks them up. Real time: a REST/gRPC service takes user id + recent behaviour + context (time) and computes recommendations per request.
- Batch strengths: everything precomputed; compute decoupled from serving (if today's job fails, serve yesterday's slightly stale results); almost no on-call load.
- Real-time strengths: (1) reacts to short-lived "missions" (user who normally buys books suddenly shops for winter gear; home page should change within the session); (2) cold-start users: a new install with a few clicks can already be profiled (browsing phones => electronics shopper) — this matters a lot for retention when users are new to the product; (3) saves compute on users who never return — batch computes for 100% of users though only 5–10% come back on a given day.
- Real-time cost: a 24/7, low-latency, high-throughput API, on-call, and above all developer time.
- Travel is an explicitly named case: the next place someone travels to is often not predicted by where they went before, so recent session context matters more than long history.

**Mechanisms — the canonical architecture**
- Offline: batch jobs train models, embed items, build indexes/graphs, load feature stores. Online: uses those artifacts to serve requests.
- Retrieval (a.k.a. recall / candidate generation / nearest neighbours): from millions of items get hundreds–1,000 candidates fast but coarsely. Typical: train an embedding model producing small dense vectors (32–64 dims), embed the whole catalogue, build an approximate nearest neighbour (ANN) index.
- Ranking (precision): score those hundreds–1,000 candidates with rich features — item metadata (category, brand, seller, sales over 1/7/14 days), user features (usual price point, usual categories), context (time of day, day/month of year). Output is a probability of click/purchase (sigmoid). Show the top 20–50.
- Four artifacts flow offline -> online: embedding model, ANN index, feature store, ranking model.
- Request path: item/user -> vector -> top-1,000 neighbours -> join features from the feature store -> rank -> top-k. Interaction logs feed back into offline training of both retriever and ranker.
- Search adds a query-processing step before retrieval (spell-check, query understanding, expansion).

**Industry examples**
- Alibaba (Taobao) retrieval: build a bidirectional item graph from users' browse sequences, random-walk it (edge weights = transition strength) to make item "sentences", train word2vec (skip-gram), precompute top-100/1,000 similar items per item into an item-to-item similarity map. Online: take the user's recent items (e.g. 3 browsed, 2 bought), pull ~100 neighbours each, send to ranking service.
- Facebook search: two-tower query/document model puts queries and documents (profiles, groups) in one vector space. Offline: embed all documents with Spark, quantize, publish index. Online: embed query with query tower, ANN retrieve, then rank using a "forward index" (= feature store) plus filters.
- DoorDash search: spell-check (deliberately not aggressive) -> query understanding/synonym mapping ("kentucky fried chicken" -> KFC) -> query expansion by traversing a Neo4j knowledge graph (KFC -> fried chicken, other fast food) -> Elasticsearch retrieval -> ranking by location, user price point, rating, food similarity. They label retrieval "recall" and ranking "precision", which states the goal of each stage.
- Retrieval approaches compared: (a) plain item-item collaborative filtering via counts — user weight = 1/(number of items user interacted with), so heavy, noisy users count less and selective users count more; item-item similarity aggregates co-interacting users. Alibaba's "Swing" variant scores by user pairs: two users with very different tastes who both touched the same item pair is stronger evidence than two similar users. Key benefit: it is just counts in a KV store, so it can be updated incrementally/streaming — real-time retrieval without a neural net. (b) YouTube deep candidate generation: average watched-video embeddings + search-token embeddings + demographics -> ReLU layers -> huge softmax over videos (trained with sampled softmax); at serving, use the last layer as the user vector and ANN against video vectors. (c) Instagram: sequences of accounts a user engaged with treated as sentences -> word2vec -> FAISS. Assumes accounts post homogeneous content.
- Ranking examples: Wide & Deep (logistic regression + DNN; he never got it to work in production), YouTube ranking DNN with weighted logistic output, Alibaba Behaviour Sequence Transformer (session sequence + target item -> sigmoid). Pattern: retrieval = "given item/user, find many similar items"; ranking = "given user features + ONE target item, predict engagement probability", run once per candidate.

**MVP recipe (his recommendation)**
1. Embeddings: self-supervised word2vec on interaction sequences (items in a session as a sentence). No labels, no negative sampling design — both are hard in real life. Uber did query2vec (queries + eventually-ordered restaurants in one sentence).
2. ANN: he benchmarked ScaNN vs FAISS vs HNSW on queries/sec vs recall; ScaNN won in his test (note: library results shift over time; benchmark on your own data). Recall 0.5 is too low; 0.99 is hard — pick the trade-off.
3. Ranker: plain logistic regression on user + item features. (The hidden hard part is the feature store that joins them.)
4. Serving: retriever + ranker on the same instance behind a load balancer (or SageMaker); scale horizontally — e.g. 50 QPS per box and 1,000 QPS target => ~20 boxes.
- NVIDIA's extended pattern (4 stages): retrieve -> FILTER (remove invalid candidates before expensive ranking: age-inappropriate, unavailable in region, already purchased) -> SCORE (model) -> ORDER (business logic: diversity, e.g. don't put Harry Potter 2–7 consecutively; intersperse similar franchises).

**Decisions & trade-offs / rules of thumb**
- If you have nothing in batch, start with batch: weekly/daily precompute into a KV store gives an MVP in weeks.
- Go real time only where preference changes fast (news, Twitter, short video, e-commerce sessions). Groceries, appliances, job listings: batch is fine. Dogfood a little real-time and measure the delta; if none, skip it.
- "Rule 1 of ML: don't start with ML." "Rule 2: most production systems don't use academic SOTA" — they use n-grams, word2vec, LR, trees because they are simple, robust, need no labels. Start with the cheapest thing that works and iterate.
- Filter before rank to avoid wasting compute.
- Learn by applying to a concrete problem; read only papers you can apply (his applied-ml repo: ~63 recsys and ~51 search papers).

**Failure modes / gotchas**
- Real-time systems bring 24/7 on-call severity; offline jobs can fail softly.
- Pure relevance ranking produces repetitive lists; need an ordering/business layer.
- Instagram-style account embeddings break if accounts are heterogeneous.

**Relevance to Sam**
- Trameter recommendations: start with batch "similar destinations/activities" (item-item co-view counts or word2vec over session sequences of viewed Viator/RateHawk products) stored in a KV table; add session-based real-time only for the home page if users show trip "missions".
- 'Where can I go': a classic FILTER stage — passport/visa eligibility removes invalid destinations before ranking; ordering stage adds diversity (regions, budgets).
- Drive Button: article -> retrieval (BM25/embedding over bookable products near the places mentioned) -> filter (availability, geography) -> rank -> order.
- Fuzzy place search over ~6M rows: DoorDash-style pipeline (light spell-correct, synonym/alias map from GeoNames alternate names, expansion via admin hierarchy) then ranking by population/popularity. Small 32–64-dim embeddings + quantized ANN fit easily in 16GB.

## Andrew Ng — Recommender Systems (Stanford ML Lecture 16) (https://www.youtube.com/watch?v=GIcuSNAAa4g)
**Recommendation as filling in a sparse user-by-item ratings matrix: with item features it is per-user linear regression (content-based); without them, collaborative filtering learns item features and user preferences jointly (low-rank matrix factorization).**

**Mental models / first principles**
- Recsys drives a large share of revenue at Amazon/Netflix-type companies yet gets relatively little academic attention — high business leverage.
- It is the textbook case of a "big idea": the algorithm can learn its own features instead of you hand-designing them.
- Problem setup: n_u users, n_m items, r(i,j)=1 if user j rated item i, y(i,j) the rating. Real matrices are extremely sparse (each user touches a tiny fraction of items). Goal: predict the missing entries, then recommend highest-predicted unseen items.

**Mechanisms**
- Content-based: each item i has a feature vector x_i (e.g. x1=romance degree, x2=action degree, plus intercept x0=1). Each user j has a parameter vector theta_j. Predicted rating = theta_j · x_i. Learn theta_j by regularized least squares over only the items that user rated; sum the objectives across users to learn all at once (gradient descent or L-BFGS/conjugate gradient). Worked example: Alice theta=[0,5,0], movie features [1,0.99,0] -> predicted 4.95.
- The m_j (count of user's ratings) normaliser can be dropped; it doesn't change the argmin.
- Inverse problem: if users' preference vectors were known, you could infer item features — choose x_i so theta_j·x_i matches the ratings of all users who rated item i (a movie loved by romance fans and hated by action fans must be romantic).
- Chicken-and-egg resolved two ways: alternate (guess theta -> solve x -> solve theta ...) which converges; or better, one joint objective minimizing squared error over all observed (i,j) pairs + L2 on all theta and all x, optimized simultaneously with gradient descent. Initialize both to small random values (to break symmetry, like neural nets). Drop the x0=1 convention: the model can learn a constant feature if it wants. No special unregularized bias term.
- Works only because each user rates several items and each item is rated by several users — "collaborative": every rating helps learn features that improve predictions for everyone.
- Vectorized: stack item features as rows of X (n_m x n) and user params as rows of Theta (n_u x n); all predictions = X · Theta^T. Hence the name low-rank matrix factorization.
- Related items: learned x_i are often not human-interpretable but capture what drives liking. Items with small ||x_i - x_j|| are similar => "people who viewed this also..." recommendations (top-5 nearest by distance).
- Mean normalization for cold-start users: a user with zero ratings gets theta=0 from regularization alone, so every prediction is 0 and nothing can be ranked. Fix: subtract each item's mean rating mu_i from its row before training, predict theta_j·x_i + mu_i. A new user then gets each item's average rating — sensible default. Column normalisation (for items with no ratings) is less important: arguably don't recommend an item nobody rated.

**Decisions & trade-offs**
- Content-based needs good item features, which are expensive/slow to hand-label; CF needs enough overlapping interactions but learns features for free.
- Regularization lambda controls overfitting for users/items with few ratings; also determines what happens to cold-start entities (shrinks to zero/mean).

**Rules of thumb**
- Always mean-normalize (or include biases) so new users fall back to item popularity/average.
- Use learned item embeddings for item-to-item similarity, not just for rating prediction.

**Failure modes / gotchas**
- Pure CF gives no signal for a brand-new user or item (cold start); without mean normalization new users get degenerate all-zero predictions.
- Learned factors are hard to interpret — don't promise stakeholders "the romance dimension".
- (note: 2010-era lecture: explicit 0–5 ratings and squared-error loss. Modern systems mostly use implicit feedback — clicks, bookings — with ALS/BPR or two-tower models and ranking losses; the factorization intuition carries over directly. Item-to-item distance at scale uses ANN indexes, not brute force.)

**Relevance to Sam**
- Trameter activities/hotels: implicit-feedback matrix factorization (user x product views/bookings) gives product embeddings -> "similar activities" via nearest neighbours; content features (category, city, price, duration from Viator/RateHawk) cover cold-start products that CF cannot.
- New Trameter users: fall back to popularity per destination (the "mean" baseline), then personalize as interactions arrive.
- Hybrid for Drive Button: content-based (article text/places as features) since there is little per-reader interaction history.
- Memory: a sparse factorization (e.g. implicit ALS) for even millions of interactions fits in 16GB; store only the factor matrices.

## Doug Turnbull (with Jason Liu) — Cheating at Query Understanding with LLMs (https://www.youtube.com/watch?v=UfHoLV674do)
**Turn free-text queries into structured attributes (category, color, room, brand) with small LLMs + enforced schemas, mostly precomputed offline and cached, always with a fallback search path; agents work better given explicit, interpretable search tools than opaque vector search.**

**Mental models / first principles**
- Query understanding (QU) = decomposing a query into properties from a (usually controlled) vocabulary — "crimson suede couch" -> color=red, material=suede, item_type=couch — then filtering or boosting on them, or routing to a specialised backend (e.g. Reddit product-review backend vs news backend).
- Google's handling of "the the" (a band) shows the power: recognise entities and change both ranking and UX.
- For a stable domain (e-commerce) the query->attributes mapping is largely stable, so it is basically a dictionary/cache. It can drift (a brand called "Crimson" appears) so it must be refreshed.
- Why not just embed the whole query? Several failure modes of pure vector search:
  - Embedding collapse / hubness: off-the-shelf models (e.g. generic CLIP) put all in-domain items close together — a "blue couch" query vs a brown table can score ~0.45 cosine just because both are furniture. Re-centering (subtract mean embedding) helps a bit but is a band-aid; the model never learned the domain's distinctions (worse for specialised domains like electronic components).
  - Embeddings don't say WHICH property makes things similar: is red couch closer to "blue couch" than blue chair? You get a vague similarity spectrum with grey zones.
  - Thresholding vector results is very hard: a cutoff like 0.73 might suit one query and wipe out relevant results for another. Broad queries ("chair") sit near the centre of the space and need a lower cutoff; specific ones separate cleanly.
  - Search data is multi-modal: numeric popularity, hierarchical categories (couch vs loveseat), titles vs descriptions behave differently.
  - Long conversational/RAG queries rarely appear in logs, so there's not enough data to fine-tune rankers/embeddings for them.
- Alternative (not mutually exclusive with fine-tuning): decompose into a structured query and expose attributes as tools for an LLM/agent. Structured, human-interpretable attributes are also agent-interpretable. Full circle: agents do better with explicit language and simple tools (like Claude Code with grep).

**Mechanisms — the QU toolbox, from oldest to newest**
1. Hand rules / human-curated spreadsheets ("query contains crimson => color red") — long the secret sauce of e-commerce; risk: not data-informed.
2. Click aggregation: for each historical query, group-by the attribute of clicked documents and promote the dominant value. At Shopify, "promote the most-clicked category for this query" was among his most successful experiments. Only works for seen queries.
3. Supervised classifier: BERT query embedding -> label learned from click-derived labels; generalizes to unseen phrasings ("scarlet sofa" -> red).
4. Tiny in-memory embedding index of the controlled vocabulary (e.g. ~8 colours, or a brand list): embed query, take nearest label. Also used to snap LLM-hallucinated brand names to real brands.
5. LLM prompt with structured output (Pydantic schema / enum) — "GAR: generative-AI-augmented retrieval".

**LLM-specific practice**
- Enforce schemas (Pydantic, provider structured outputs, Instructor, OpenRouter to smooth API differences). Everything in the schema — field names, docstrings, enum values — becomes prompt text, so name things meaningfully. Never use opaque enum IDs (color_id 3); LLMs are language machines.
- Allow "unknown"/empty list. Without it, "leather couch" -> "brown" (a guess). Decide explicitly: extract only explicit mentions, or infer? He prefers the narrowest explicit task.
- Define the task via a hand-written eval set before prompting ("blue couch"->blue, "azure pillows"->blue, "leather couch"->unknown). Writing evals surfaces the rule you actually want. Vague tasks like "expand synonyms" are ill-defined (castle -> palace, fortress, hacienda?) vs strict variants (USA = United States of America).
- Metrics: precision (of non-unknown predictions, how many correct) and coverage (share of queries that get any label — proxy for recall and for how many queries the change will affect). For multi-label, use Jaccard overlap.
- Classification gains may not move search metrics (e.g. NDCG): you may be improving queries that already worked. Target the query segments that hurt retrieval metrics most.
- Bias to precision + escape hatch: high-confidence structured path when the attribute is surely right; otherwise fall back to a solid lexical/vector search. Designing precision paths with fallbacks is a core IR pattern.
- Business-specific defaults need guidance: rugs belong in many rooms; "bistro table" in US home furniture is usually outdoor/patio, not restaurant. He now prefers written guidelines over many few-shot examples: few-shots can overfit to test cases and confuse small models (a paper shows rules vs few-shot produce different internal representations). Guidelines + a few examples also works.
- Latency: search is latency-sensitive; a ~300 ms LLM call inline is usually unacceptable. Precompute QU as a nightly batch job over seen queries (use provider batch APIs), store in a dictionary/cache, fall back on cache miss. In RAG, inline is fine since you're already calling an LLM.
- Use the smallest model, short prompt, little context; if a QU task needs a big model it's a smell that the task decomposition is wrong. Reasoning is usually unnecessary (and costly) for extraction.
- Agentic QU: a reasoning agent with a simple search tool can try parameter combinations, judge results and iterate (7–8 calls; "couch fit for a vampire" worked well). Save the winning structured query per query in a DB and replay it for traditional site search — offline agentic exploration as a QU-learning loop. Hypothesis: simple backends + agents instead of thick complex backends. Agents are persistent: even mediocre recall can be compensated by many calls.
- Content enrichment is the mirror of QU: if query logs often contain company names or fiscal years, extract those onto documents too. Domain quirks (fiscal year ending March) can be encoded in the prompt.
- Multi-topic queries: tag the content with entities so one structured query ("mentions A and B") replaces many searches.

**Team / process lessons**
- Biggest mistake: staying theoretical too long. Get everyone shipping and on call; people should hold the whole system (model + operational constraints) in their heads. Ship to prod fast behind feature flags; avoid elaborate lab-only experiment tooling. A model 4% better but 2 s slower may lose in production (~300 ms latency can cost ~1% revenue — anecdotal figure).

**Relevance to Sam**
- Trameter search: extract {destination, activity type, date, budget, travellers, duration} from queries with a small model + enum schema whose values are human-readable (Viator category names, not IDs). Precompute for frequent queries nightly; cache; fall back to BM25/fuzzy search on miss.
- 'Where can I go': parse "beach in Dec without visa for Nigerian passport" into structured filters (passport=NG, visa=free/on-arrival, theme=beach, month=12) — perfect controlled-vocabulary QU; precision path with fallback.
- Place search (~6M rows): snapping LLM output to canonical GeoNames IDs via a tiny embedding/alias index mirrors his brand-resolution trick; avoid letting the LLM invent IDs.
- Drive Button: content enrichment — extract places/activities from blog articles as structured attributes, then match to bookable inventory via attribute filters + ranking rather than raw embedding similarity with fragile thresholds.
- Evals first: build a hand-labelled set of real Trameter queries; track precision + coverage, then NDCG on the downstream search.

## ritvikmath — Every Ranking Metric: MRR, MAP, NDCG (https://www.youtube.com/watch?v=2XegvMul_mE)
**Pick the ranking metric only after agreeing what user behaviour you actually care about: MRR for "first success", MAP for "binary holistic list quality", NDCG for graded relevance — and know each one's blind spots.**

**Mental models / first principles**
- Ranking is its own problem family, distinct from classification/regression; evaluation must look at positions in a list.
- Before choosing a metric, answer "what do we care about?" with collaborators: position of first click? first watch (stronger action)? or that all engaged items sit generally high? Different answers -> different metrics.
- Running example: a stats streaming app shows a ranked list of 3 shows; users can click (interest) or watch (commitment).

**Mechanisms**
- Reciprocal rank (RR) = 1 / position of the first engaged item; MRR = mean of RR over all lists served. Click at position 1 => RR = 1 regardless of the rest of the list.
- Average precision (AP): at every position where the user engaged, compute precision@that position (engaged so far / items so far), then average those. Example: engage at 1 and 3 => (1/1 + 2/3)/2 = 5/6 ≈ 0.83. Engagements at 1 and 2 => 1.0. Interpretation: at each success, how much junk did the user wade through to get there. MAP = mean of AP across lists. (note: textbook AP divides by the total number of relevant items, including ones not retrieved in the top-k; the video's version divides by relevant items retrieved, which ignores missed recall.)
- NDCG = DCG / IDCG, where DCG sums graded gains discounted by log of position, and IDCG is DCG of the ideal ordering. Allows grades, e.g. watch=2, click=1, nothing=0. Examples given: [click, none, watch] -> 0.69, [click, none, click] -> 0.92, [click, watch, none] -> 0.80 (as stated in the video).

**Decisions & trade-offs**
- MRR: + dead simple, most interpretable. − ignores everything after the first hit; binary labels only. Use when there's essentially one right answer (navigational/known-item search) and you want it at the top.
- MAP: + considers the whole list. − binary relevance only: a click and a watch count the same.
- NDCG: + handles graded relevance (the main reason to use it), the most common industry metric. − harder to explain, more compute.
- Don't default to NDCG blindly; MRR and MAP are often the right tool.

**Failure modes / gotchas**
- Key NDCG pitfall: normalisation is per-list against that list's OWN ideal ordering. [click, none, watch] (0.69) scores lower than [click, none, click] (0.92) even though the first list contains a strictly more valuable item, because NDCG measures deviation from the best possible ordering of what's in the list, not absolute value. Comparing NDCG across lists/systems with different label content can mislead. Consider variants (e.g. a fixed/global ideal, or reporting raw DCG alongside) when you need cross-list comparability.
- MRR and MAP force you to binarize labels, discarding the click/watch/booking distinction.

**Rules of thumb**
- Write down the business-valued action ladder first (e.g. impression < click < add-to-cart < booking), then choose metric.
- Report more than one metric; examine cases where they disagree.

**Relevance to Sam**
- Place/fuzzy search over ~6M locations: users usually want one specific place => MRR (and recall@k for the retriever) is the natural offline metric on a labelled set of query -> correct GeoNames ID.
- Trameter activity/hotel listings: graded labels from behaviour (view=1, add-to-cart=2, booking=3) => NDCG@k; beware the per-list normalisation when comparing segments where some queries never produce bookings.
- 'Where can I go' and Drive Button suggestions: multiple acceptable answers, binary "relevant/not" judgments from human or LLM raters => MAP or precision@k.

## Mayya Sharipova (Elastic) — Improving Search Relevance with Numeric Features in Elasticsearch (https://www.youtube.com/watch?v=Ih8JD6tkp6Q)
**Blend the query-dependent BM25 text score with query-independent numeric signals (quality, topic affinity, distance, recency, vector similarity) by adding transformed features linearly in a bool "should" clause — and use index structures that let the engine skip non-competitive blocks.**

**Mental models / first principles**
- Text queries are scored with BM25 ("organic" text score), but users want results tailored by numeric signals. Three recurring patterns:
  1. Recommendation-style boosting: prefer favourite topics, reputable sources (PageRank-like quality).
  2. Proximity/recency: prefer restaurants near me, newer news.
  3. Similarity: find semantically related docs that share no keywords ("zip up files" -> "compressing folders").
- Query-independent features (PageRank, URL length, popularity) need to be combined with query-dependent scores. Elastic followed research (Craswell et al.-style "query-independent evidence" work) showing the best combination is linear: text_score + w · f(feature), where f is a saturating transform so a huge feature value can't swamp text relevance.

**Mechanisms**
- `rank_feature` (single value, e.g. pagerank) and `rank_features` (sparse map, e.g. topics: {business: 20, canada: 50}) field types. Sparse map avoids creating one field per topic.
- Transforms: saturation S/(S+pivot) (default), log(scaling + S), sigmoid S^a/(S^a + pivot^a). Optional boost weight.
- Query pattern: bool { must: match(text) ; should: [rank_feature(topics.business), rank_feature(topics.canada), rank_feature(pagerank)] }. Should-clauses add to the score only for documents already matched by must.
- Efficiency: Lucene stores postings in blocks of 128 doc IDs + 128 frequencies + skip data; since ES 7.0 it also stores per-block "impacts" (max score the block can contribute). With a current top-k threshold, blocks whose max can't beat it are skipped (block-max WAND) — up to ~10x speed-ups on some queries. rank_feature values are indexed as term frequencies of a hidden `_feature` field so they get impacts too, making feature boosting nearly free.
- `distance_feature` query on existing date or geo_point fields: score = pivot / (pivot + |origin − value|), range 0–1; equals 0.5 at distance = pivot. Example: Japanese restaurants (must) + should distance_feature(location, origin=me, pivot=1000 m) => score halves at 1 km. Same for recency with origin=now and pivot e.g. 7d. Also skips non-competitive hits, but less efficiently than rank_feature because it computes dynamically. Replaces slower, expert function_score scripts.
- Vector fields (2019): `dense_vector` (≤1024 dims) and `sparse_vector`, stored as binary doc values; scored by `script_score` with cosineSimilarity/dotProduct. Add +1 to cosine because ES 7 forbids negative scores. Storage ≈ 4 bytes × dims per doc (1,000 dims => ~4 KB/doc); sparse ≈ 6 bytes per non-zero dim. Demo: Universal Sentence Encoder over 2,000 Stack Overflow questions, <50 ms — but only because the set was tiny.

**Decisions & trade-offs**
- rank_feature: fastest; needs new field type at index time; static per-document values.
- distance_feature: works on existing fields; dynamic; good for geo/time decay.
- Brute-force vectors: exact similarity, but a linear scan over every matching doc — dangerous on millions of docs. Restrict with a filter query first so only a subset is scored.
- Semantic matches can be irrelevant (a result about "searching directories" for a zip query) — vectors find related-but-wrong items.
- (note: outdated — since ES 8.x, dense_vector supports HNSW approximate kNN with quantization, and hybrid lexical+kNN with RRF exists; the 2019 brute-force-only limitation no longer holds. The rank_feature/distance_feature advice remains current.)

**Rules of thumb**
- Keep text relevance as the gate (must) and numeric signals as additive boosts (should), each through a saturating transform with a tuned pivot.
- Pick the pivot to encode a human-meaningful halving point (1 km, 7 days, median popularity).
- Filter before any expensive vector or script scoring.
- Budget index size for vectors: rows × dims × 4 bytes.

**Failure modes / gotchas**
- Unbounded features (raw review counts, raw popularity) dominate linear combos; always saturate/log.
- Function-score scripts are flexible but slow and can't use block skipping.
- Two-sided matching (publisher ↔ writer both must agree) is not solved by boosting; it needs a reciprocal matching model (raised in Q&A, left open).

**Relevance to Sam**
- Place search over ~6M GeoNames rows: BM25/fuzzy name match as the gate, then rank_feature boosts on log(population), feature-class importance, and a distance_feature boost from the user's location or the article's location — exactly how geocoders order "Paris" (France before Texas).
- Trameter activities: boost by rating, review count (saturated), bookings; recency for new Viator products; distance_feature from the hotel/destination centre.
- 'Where can I go': rank_features map per destination {visa_free: x, beach: y, budget_fit: z} for sparse affinity boosting.
- Memory math on a 16GB laptop: 6M × 384 dims × 4 B ≈ 9.2 GB float32 — too much alongside everything else; use int8/binary quantization or smaller dims, and filter (country/admin area) before vector scoring.

