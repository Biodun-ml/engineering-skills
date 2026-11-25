# search-3 — cross-cutting principles

1. **Retrieve, then rank.** A cheap, recall-focused retrieval stage (inverted index / BM25, or two-tower + ANN) cuts millions of items down to hundreds. Costlier rankers and re-rankers then order them. Keep each stage replaceable. [DataTrek two-tower; Parmar/Shipt; Mohan]
2. **Ranking is a grouped problem, not classification or regression.** Scores matter only relative to the other candidates for the *same* query. Carry a query/group ID, split by query, and evaluate per query with NDCG/MRR/AP@K. [ritvikmath LTR]
3. **Relevance is semantic as well as lexical.** Relevant documents can share zero words with the query, so dense or hybrid retrieval is needed for vocabulary gaps. Lexical matching is still the faster, cheaper workhorse for exact text matches, so don't throw ML at what the engine already does well. [Potts; Parmar]
4. **Query understanding lives outside the engine and starts from your own catalogue.** Tag query tokens as entity, brand, size, modifier, negative intent, compound phrase or ignorable, using dictionaries mined from the catalogue. Then build must / should / must_not / phrase clauses, auto-filters and minimum_should_match from that intent. ML classifiers come later. [Parmar/Shipt]
5. **Know the traffic shape and optimise the head first.** At Shipt, 50% of searches were the top 1,000 queries and 30% of purchases were repeats. Hand-check head queries, use history as a first-class signal, then work on torso and tail. [Parmar]
6. **Blend text score with behaviour signals, and let context set the weights.** final = w·BM25 + Σ w_i·f(signal_i), with logs for heavy-tailed signals.
   - Weight text more for new stores/items and long queries, popularity more for mature inventory and short head queries.
   - Per-(query, item) conversion scores make ranking self-improving, but watch for feedback loops.
   - BM25 can't be normalised in-engine, so calibrate against its observed range.
   
   [Parmar]
7. **The top slots are what count.** Users rarely go past the first row or page. Personalised post-processing (repeat buys, sponsored placements, diversity) belongs in a final re-rank service over the top-K. Precision@K and AP@K should reward getting the first few right. [Parmar; Mohan]
8. **Two towers scale because the towers are independent.** Precompute item vectors offline, encode only the query/user online, and L2-normalise so dot product = cosine. Small output dimensions (16–64) keep ANN fast and memory-light. Start from pretrained text embeddings and fine-tune on conversions. [DataTrek; Mohan]
9. **Cold start needs content.** CF/matrix factorisation (user·item dot product in a shared latent space) can't score unseen items. Content embeddings (and hybrids trained on behaviour + content) can. Most real inventories (new activities, new hotels) start cold. [Mohan; DataTrek]
10. **Choose dimensions and models by quality vs latency/compute, measured with evals.** Typical embedding sizes are 128–1,024. In PLUM, 900M beat 3B for its compute budget. Training loss and eval loss can disagree, so trust evals. [Mohan; Yan]
11. **Relevance isn't the only objective.** Diversity (embeddings can find *dissimilar* items too), freshness, fairness across user and item groups, coverage of the long tail, and latency budgets all shape the design. Separate models or re-rank passes per objective are fine. [Mohan]
12. **Bridge the offline–online gap deliberately.** Offline proxies (P@K, recall, NDCG), large held-out tests that include new items, dogfooding, then randomised A/B tests on business metrics. Useful online diagnostics:
    - add-to-cart / conversion per search
    - **searches per session** (rising means failure)
    - median position of conversions
    - comparable keyword mixes across experiment arms.
    
    [Mohan; Parmar]
13. **Make stacked rankers debuggable.** Return each signal's score contribution per result. A/B "base query only" against "base + ranker" to isolate changes. Eyeballing nearest neighbours and top queries is a cheap first check, but it doesn't replace a judgment set. [Parmar; DataTrek]
14. **Semantic IDs: items as short hierarchical tokens.** An RQ-VAE turns content embeddings into a few discrete codes (coarse → fine). An LLM can then learn behaviour over items, take natural-language requests, generate real catalogue items and explain them.
    - Use stop-gradients to avoid codebook collapse, and monitor codebook usage and ID uniqueness.
    - Clean the item data first.
    - Add a co-occurrence loss so IDs reflect behaviour, not just content.
    
    [Yan]
15. **Always validate generated outputs against ground truth.** Map generated semantic IDs, URLs and citations back to the catalogue and drop misses (over-generate to compensate). Fabricated evidence is worse than none. Keep provenance (a ranked, inspectable evidence list) even when an LLM writes the answer. [Yan; Potts]
16. **Search, recommendations and chat are converging.** Users now state constraints no keyword query can express. Retrieval (classical, dense or generative) remains the grounding layer, and retrieval can also supply few-shot demonstrations to prompts. [Yan; Potts]
17. **Search-engine engineering gotchas.**
    - Apply fuzzy matching only when the query isn't already correctly spelled.
    - Raise refresh_interval on write-heavy indices to protect read p99.
    - Prefer denormalised documents over parent-child joins.
    - Split query understanding into its own service.
    
    [Parmar]
18. **The reward/label definition is half the system.** A handcrafted per-click reward (watch fraction, likes, …) at YouTube, and conversion labels for two-towers: what you count as a positive defines what the model learns and how you evaluate it. [Yan; DataTrek]

**For Sam (Trameter):**
- Fuzzy search over ~6M GeoNames places:
  - Skip fuzziness for known-correct tokens.
  - Phrase-match multi-word place names.
  - Rank by BM25 + log(population) + per-(query, place) click conversion.
  - Make it LTR (LightGBM lambdarank) once click logs exist.
- Viator/RateHawk discovery and Drive Button article→trip matching: a two-tower or sentence-embedding retriever with FAISS (32–64-d fits in RAM), measured by recall@K, then a ranker with business signals and a diversity re-rank.
- "Where can I go": structured passport/visa filters plus dense retrieval, with every fact grounded in the travel-access DB. Semantic IDs + a small LLM are the longer-term conversational path.

---

## Karthik Mohan (Data Science Dojo) — Recommender Systems: Basics, Types, and Design Consideration (https://www.youtube.com/watch?v=Ams4sEn50cw)
**Recommenders proactively surface relevant items. Collaborative filtering learns from behaviour, content-based filtering handles cold start, hybrids (two-tower) combine both, and you judge them on ranking-aware relevance metrics plus diversity, freshness, fairness, latency and business metrics.**

- **Why recommenders exist.** Users find items four ways: search, browse, popular categories, and recommendations. Recommendations are the proactive, personalised one. Businesses fund them for four reasons: they save user time, surface relevant items over merely popular ones, personalise (every Netflix home page differs) and give **coverage**, so the long tail of a huge catalogue gets seen.
- **Carousel / widget framing.** Amazon and Netflix pages are stacks of carousels such as "Buy it again", "Pick up where you left off", "Recommended for you" and "Trending". Each one is a separate recommendation task. "Buy it again" is a cheap, very strong baseline built from the user's own history. A new team should start with one carousel. A mature team does **whole-page optimisation**, reordering carousels as well as items inside them and measuring the gain.
- **Taxonomy.** Non-personalised (sponsored, popular) vs personalised. They can be mixed: "popular in a category this user buys a lot" pairs popularity with a personal category signal. That is a cheap and effective pattern.
- **Collaborative filtering = matrix completion.** You have a users x items matrix with mostly missing ratings. Intuition: fill a gap from similar users (Karthik likes Avatar and dislikes the rom-com, like Rishi does, so copy Rishi's "Arrival" rating). The baseline model factorises the matrix into user factors and item factors in a **shared latent space**. predicted score = dot(user_vec, item_vec), passed through a sigmoid to get a probability (e.g. 0.9 means recommend). The factors are learned only from observed ratings. Most deep recommenders still end in a dot product.
- **Netflix Prize.** It filled missing ratings and was scored on RMSE. The winning team blended several models, CF among them. (note: RMSE on ratings is a poor proxy for top-k ranking quality; modern work uses ranking metrics.)
- **Cold start.** A new item (or user) has no learned factor, so pure CF fails. **Content-based filtering** fixes this: embed items from metadata (description, genre, tags) so a new item lands near similar known items (Men in Black next to Arrival). Embeddings can be trained on **behaviour + content together** ("people who watched X watched Y" plus text), so the space reflects both.
- **Embedding mechanics.** Typical dimensions are 128 / 256 / 1024, sized to roughly reflect the number of underlying concepts (genres number in the hundreds). The trade-off is quality vs latency and training time: 128 may be too small, 1024 costs latency, so pick the sweet spot (e.g. 512) by testing. Dimensions are latent. To interpret dimension d after training, see which items activate it and map those to known labels (genres). Cosine = dot product of normalised vectors (an angle). Orthogonal items are unrelated.
- **Hybrid / two-tower.** One tower builds a deep user representation, the other builds an item representation from content and IDs, and they are scored by dot product. It is popular in industry because it gives **fast inference** compared with an MLP over concatenated features (item vectors can be precomputed). Train it first as binary classification with BCE on like/not-like. Move to ranking losses (pairwise) later.
- **Model choice follows input type.** Text content means a transformer encoder for item embeddings. Tabular data means feed-forward layers.
- **Metrics (relevance first).** Example: 5 held-out purchases and 6 recommendations, 4 of them hits. Precision = 4/6 = 0.67, recall = 4/5 = 0.8. Precision alone can be gamed by repeating one safe item, and recall checks variety. **Precision@K** matters because users only look at the top ~5. Average Precision@K rewards putting hits early. In his example, moving the non-relevant items to the front dropped AP@4 from 0.375 to about 0.21. (note: he divides by K. The standard AP divides by the number of relevant items. Either way, order now matters.) Also named: NDCG, MRR, MAP, and F1 of precision/recall.
- **Beyond relevance (design considerations).**
  - **Diversity:** use embeddings in reverse. Once cereal is in the cart, pick a dissimilar item (coffee) rather than another cereal. Or run a separate diversity model next to the relevance model.
  - **Freshness / recency:** critical for fashion and TikTok.
  - **Fairness:** check whether all user groups and all item categories get comparable quality.
  - **Latency:** cached recommendations are free. Content-based on-the-fly generation for new items costs latency, and the latency budget shapes the architecture.
- **Business metrics vs modelling metrics.** Revenue, items purchased, subscriptions and growth can't be optimised directly by a model, so offline proxies are used. Expect an **offline–online gap**. Bridge it with (1) large held-out offline tests that include new items, (2) internal dogfooding before launch, and (3) randomised A/B tests against the status quo, which make the launch call.
- **Relevance to Sam / Trameter.**
  - Carousels map directly to trip pages: "trending in [city]", "because you viewed X", "complete your trip" (hotel + activity).
  - Viator/RateHawk inventory is mostly cold start (few interactions), so content-based embeddings from descriptions, categories and location are the base, and CF gets added as behaviour accumulates.
  - "Popular in your favourite category/region" is a strong cheap hybrid for early-stage traffic.
  - Diversity re-ranking (not 6 walking tours of one city) matters.
  - Measure P@K / AP@K offline, then A/B test on bookings.

## Dipak Parmar (Shipt, Haystack LIVE / OpenSource Connections) — Query understanding and ranking at Shipt Inc. (https://www.youtube.com/watch?v=VOrpRdWqe7A)
**Production e-commerce search is mostly query understanding (outside the engine) plus a multi-phase ranking stack: BM25 → boosts → signal blend → top-K re-rank → personalised post-process. It is tuned by A/B tests, with focus on head queries.**

- **Know your query distribution first.** Shipt is grocery delivery: 100+ retailers, thousands of store locations each with its own inventory and prices.
  - About 50% of searches come from the top 1,000 unique queries, and 30% of purchases are repeat buys.
  - Query length: 20% are one token, 50% two tokens, 30% three or more.
  - So optimise **head, then torso, then tail**, and treat "what you bought before" as a first-class signal. Validating ~1,000 head queries by eye covers most traffic.
- **Text analysis is the foundation.**
  - Lowercasing, synonyms (quick→fast), stemming (jumped→jump), stop words, special characters, and n-grams for autosuggest.
  - **Shingles plus concatenated shingles** so "ice cream" and "icecream" match the same thing.
- **Query understanding = inferring intent before retrieval, outside the search engine.** Segment the query into typed parts:
  - entity (bread, pizza, milk), brand (Kraft), size (1 gallon), modifier (2%), dietary (veggie), colour/pattern, flavour or shape (spicy)
  - **negative intent** ("egg free bread" becomes a must_not on eggs)
  - **compound words** that must be phrase-matched (peanut butter, sour cream, coffee filters, hot dog), versus modifier + entity (salted butter)
  - **ignorable tokens** ("hot coffee" can drop "hot", but "hot dog" can't)
  - query expansion (produce → its category subtree), replacements and spell fixes (common misspellings of broccoli), and hyphenation variants.
- **How to build query understanding cheaply.** Start by building a **knowledge base from your own catalogue**: its brands, sizes and categories become dictionaries for tagging query tokens. Later add clustering or classifiers over the query log, then embeddings. Shipt admitted its ML intent work was still early, which is a useful reality check: dictionaries from the catalogue carry you far.
- **Query builder is driven by intent.**
  - The main entity is a must-match, modifiers are should-match, and compounds are phrase-match.
  - **minimum_should_match / word-drop** is set from intent and token count. Partial matching boosts recall but can flood results, so Shipt ran several A/B tests on word-drop alone.
  - When a brand or type is detected, apply an **automatic filter** to cut junk recall.
  - A **rule engine for boosts**: seasonal boosts by keyword and time of year, plus category and brand boosts.
- **Five ranking phases.**
  1. Base query BM25 shaped by intent.
  2. Boosts.
  3. Custom ranking with external signals via Elasticsearch script_score / Painless or a plugin. Signals include product popularity, regional/neighbourhood trends, and **product→converted-keyword scores** (this string cheese converts on "string cheese" but not on "cheese", so keep a per-(query, product) conversion score).
  4. Re-rank the top-X inside the engine (LTR or vector similarity, still being explored).
  5. **Post-process re-rank in a separate microservice**: boost products the user bought before into the **first two slots**, because most conversion happens in the first row. Sponsored or featured placement and personalisation also belong in this last phase, outside the engine.
- **Scoring formula.** final = w0·BM25 + w1·m1·signal1 + w2·m2·signal2 …, applying a log to heavy-tailed signals.
  - **BM25 can't be normalised** inside script_score, because the max isn't known at that point. The workaround is to look at the typical score range (e.g. 1,000–5,000) and scale the other signals relative to it.
  - The weights are found by **A/B testing**, and can depend on context:
    - New store or product with few signals: weight BM25 higher.
    - Mature store: weight popularity higher.
    - Short head queries ("milk"): weight popularity more.
    - Long specific queries: weight text relevance more.
  - Because the signals refresh continuously, the system **"learns by itself"**: a relevant product that starts on row 5 climbs as add-to-carts accumulate. (note: this is a feedback loop with popularity/position-bias risk; new items need exploration.)
- **Platform.**
  - Go services, Elasticsearch (Elastic Cloud), Redis and Postgres for signals, Kafka, Docker and Kubernetes.
  - Real-time indexing: product events join the latest signal scores at index time, with no batch jobs.
  - A separate purchase-history cluster fed by cart events.
  - Clusters are grouped by store traffic, index traffic and document size.
  - One query returns results plus aggregations at **p99 ≈ 70 ms**.
  - An in-house experimentation platform: every relevance change goes through an A/B test.
- **Metrics they use (online).**
  - Add-to-cart rate and search conversion (add-to-carts / search events).
  - **Searches per session**: if it rises, results are failing and users are reformulating.
  - **Median grid position of add-to-carts** between control and variant.
  - Control and variant need similar keyword mixes for a fair comparison.
  - No offline NDCG harness yet: they eyeball screenshots of the top queries and ramp 1–2% traffic. (note: an offline judgment set would catch regressions earlier.)
- **Debugging a stacked ranker.** Return score metadata per document (raw text score, each signal's contribution). Then BM25 = total − signals. To test a base-query change in isolation, A/B "base only" against "base + ranking formula".
- **Lessons and gotchas.**
  - **Single responsibility**: pull query understanding into its own service and share modules rather than duplicating logic.
  - **Don't solve everything with the search engine**: it only scores text similarity.
  - **Don't solve everything with ML**: the engine often does text similarity faster and cheaper, and ML takes longer to iterate.
  - Raising the **Elasticsearch refresh_interval** from 1s to 30–60s improved read p99 on a write-heavy index, at the cost of up to 60s staleness.
  - **Parent-child** (product → per-store price/inventory) degraded performance in load tests.
  - **Denormalising and duplicating data is usually fine** in a search index.
  - **Blanket fuzzy search is expensive**: only apply fuzziness when the query isn't already correctly spelled (check spelling first).
  - Act fast, evaluate fast, fail fast.
- **Relevance to Sam / Trameter.**
  - The ~6M-location fuzzy search should skip fuzziness for exact or known-correct tokens, and handle compound place names as phrases ("New York", "Cape Town", "Port Harcourt").
  - Build a gazetteer knowledge base (countries, admin regions, POI types) from GeoNames to tag query parts ("boat tour Lagos" = activity type + place).
  - Ranking can be BM25 + log(population / popularity) + per-(query, place) click conversion.
  - Personal history ("searched before") goes in a post-process re-rank.
  - Track searches per session and the position of bookings.
  - Ask whether per-supplier offers (Viator/RateHawk prices that vary by date) belong in nested or parent-child documents or denormalised ones. Shipt's experience favours denormalising.

## Eugene Yan (Latent Space Paper Club) — Replacing Embeddings with Tokens: The Semantic ID Revolution in RecSys (https://www.youtube.com/watch?v=uU4WznsSDcU)
**Turn each catalogue item into a short sequence of meaningful discrete tokens (a semantic ID) via residual quantisation. Then an LLM can be fine-tuned on behavioural sequences, so it recommends real items from natural-language requests and explains why. Google's PLUM paper adds multi-resolution codebooks and a co-occurrence loss so the IDs reflect behaviour as well as content.**

- **The problem.**
  - LLMs have world knowledge and can converse, but they don't know your catalogue and they lean towards popular items.
  - Classic recommenders know proprietary behaviour well but can only be steered by item IDs in and out.
  - Customers increasingly ask things no keyword query can express ("on the couch with my kid, child-friendly, preferably unicorns"; "educational documentary to fall asleep to, no gore").
  - Eugene's thesis: **search, recommendations and chat are converging**, and semantic IDs are the bridge. YouTube (PLUM / Semantic IDs v2) and Spotify are investing in this.
- **Why not random IDs or raw embeddings?**
  - A hashed item ID tokenises into roughly one token per character, and its structure is meaningless, so a model can only memorise it.
  - A model can't consume a raw float embedding as language.
  - A semantic ID is something like 4 tokens (e.g. <sid_start> 7 1 4 … <sid_end>). It is compressible and **hierarchical**: in a YouTube example, a shared prefix 886 = food vloggers, 886-1360 = Asian cuisine, 886-514 = snacks and unboxing.
  - Payoff at scale: in YouTube's recommenders, about **90% of model parameters sat in item embedding tables**. Semantic IDs free that capacity for computation and learning patterns.
- **RQ-VAE mechanism (the hard part, about 20–30 experiments vs 3–4 for fine-tuning).**
  1. Encode the item's content embedding.
  2. Find the nearest vector in codebook level 1 and record its index.
  3. Subtract that vector to get the **residual**.
  4. Find the nearest vector to the residual in codebook level 2.
  5. Repeat for L levels. The code sequence is the semantic ID.
  6. Decode it back to reconstruct the embedding.
  
  Losses:
  - **Reconstruction** (MSE of input vs output).
  - **Codebook loss**, with stop-gradient on the encoder residual, which pulls the code vectors towards the residuals.
  - **Commitment loss**, with stop-gradient on the codebook, which forces the encoder to commit to existing codes.
  - A β weight balances the codebook and commitment losses.
  
  **Without stop-gradients the codebook collapses** to a single code (Eugene verified this himself), and it degenerates to a plain VAE. Health check: plot **codebook usage per level** (you want a flat-ish spread over e.g. 256 codes) and the **proportion of unique IDs**. Cleaning the item data noticeably lowered the loss and raised ID uniqueness.
- **Eugene's own setup.**
  - Codebooks of 4 levels × 256 codes = 1,024 new tokens, plus sid-start, sid-end and a recommendation token (vocabulary extension).
  - Qwen3-8B, fine-tuned on (a) item-to-metadata pairs (SID ↔ title, description, platform, genre) and (b) behavioural sequences (bought Halo 3, Lost Planet → next SID).
  - Results: "Xbox games similar to this Zelda game" returns Xbox titles, multi-turn refinement works, it explains its recommendations, and it can even name and describe a bundle.
  - The two halves are **decoupled**: first a tokeniser for items, then training data that teaches behaviour. Good item data is a precondition.
- **PLUM changes (semantic IDs v2).**
  1. **Fused multimodal content**: concatenate text, image (maybe audio) embeddings and project, instead of averaging frame embeddings.
  2. **Multi-resolution codebooks**: level 1 has many codes (e.g. 4,096), then 2,048, then 1,024 and so on, halving each level, because residuals shrink and later levels need fewer codes. This is for efficiency.
  3. **Co-occurrence contrastive loss** added to the RQ-VAE: items users consume together get similar SIDs. Content-only IDs don't capture similarity as users perceive it, and this loss reportedly adds a lot.
  4. Progressive masking (Eugene didn't understand it).
- **PLUM training.**
  - Continued pre-training on sequences that mix SIDs with metadata (channel, watch ratio, watch time) and SID ↔ title, description and topics.
  - Then task fine-tuning for **generative retrieval**: watch history, region and demographics go in, the decoder generates the next SID, decoded with **beam search**.
  - The labels use a **handcrafted per-click reward** (clicks, watch fraction, likes…), which is secret sauce. Eugene: defining a good reward is half the battle, and it's your labels and evals.
  - **Invalid or hallucinated SIDs** (<5%): look each one up in the SID→item table, drop misses, and over-generate (ask for 10, show the 8 valid).
- **Results and scaling.**
  - Gemini 1.5 MoE sizes of 110M / 370M / 900M / 3B. **900M was the sweet spot.**
  - LLM-initialised beats random init by a small, persistent margin (e.g. 0.28 vs 0.27), but random init is "not too bad". You don't strictly need a frontier checkpoint.
  - Training loss and eval loss told different stories (larger models overtook smaller ones at different compute points), so **judge by evals, not training loss**.
  - Eugene used 8B but thinks ~1B may suffice.
  - Serving: no off-the-shelf inference stack exists yet.
  - A live demo of YouTube's free-text search (apparently SID-based) showed about a 2% defect rate on adversarial queries. Recency handling and personalisation are open issues.
- **Gotchas.** Codebook collapse, dirty item text, invalid IDs, reward design, and evaluating on non-IID adversarial queries. A deliberate share of "off" recommendations can be a feature for exploration.
- **Relevance to Sam / Trameter.**
  - This is the long-term shape of Trameter's "where can I go" and conversational trip search: user text plus passport plus history, generating real bookable items from Viator/RateHawk.
  - A cheap path: content embeddings of activities and destinations → RQ-VAE (3–4 levels × 256) → fine-tune a small open model (~1B) on (SID ↔ metadata) plus session sequences. Always validate the output against the catalogue table.
  - The hierarchical-prefix idea is also usable without an LLM, as a coarse-to-fine bucket key for fast candidate retrieval or dedup across 6M locations.
  - The RQ-VAE trains fine on a 16GB laptop for catalogue sizes in the millions if embeddings are precomputed and streamed.

## DataTrek — Building Scalable Retrieval System with Two-Tower Models (https://www.youtube.com/watch?v=o-pZk5R0TZg)
**A two-tower model fine-tunes pretrained text embeddings on conversion labels so query and item vectors live in one small, normalised space. Item vectors are precomputed offline, the query vector is computed online, and ANN returns candidates for a ranker.**

- **Why two towers.**
  - The towers are generic pairs of entities. For search: query↔product, query↔document, query↔person. For recommendations: user↔product, user↔video, user↔user (whom to follow).
  - They scale because the towers are **independent until the final dot product**. Precompute every item embedding offline into a KV store or ANN index (FAISS/Annoy), then compute only the query/user embedding at request time and run approximate nearest neighbour search.
  - A content-based item tower can still embed **unseen items** from their features, which helps with cold start.
- **The two-stage pipeline.** **Retrieval** (millions → hundreds or thousands of candidates, cheap and recall-focused), then **ranking** (an expensive model over the candidates). Two towers sit in retrieval.
- **Why not just keyword match or raw pretrained embeddings?**
  - Both work as baselines, but neither knows *what converts*. For "Nike foam running shoes women black", should brand dominate, or the other attributes? Plain word match gives no control over that.
  - Past conversion data lets the model learn the weighting of fields (brand vs colour vs title vs description).
  - You can add numeric features (price, rating, review count) and, for personalisation, user signals such as price sensitivity and brand history, so the same query ranks differently for different user segments.
  - Recipe: **start from pretrained embeddings and fine-tune them on behaviour**.
- **Reference architecture.**
  - Item tower: separate embeddings for title, description, bullet points, brand and colour, plus numeric features, through dense layers to a condensed vector, then L2 normalisation.
  - Query tower: query-text embedding plus query features (length, historical popularity), through dense layers, then L2 normalisation.
  - Score = dot product, which equals **cosine because both are normalised**.
  - Loss = binary cross-entropy against "did this (query, product) convert/match". The outputs can be small (16–32 dims) for fast real-time retrieval, even if the inputs are 1,000–2,000 dims.
- **His implementation.**
  - Data: the Amazon Shopping Queries dataset (labels Exact / Substitute / Complement / Irrelevant). He kept only Exact = 1 and Irrelevant = 0, plus **random products as generated negatives**.
  - 150k-row sample on a free Kaggle GPU.
  - Embeddings from a sentence-transformers DistilBERT model (768-d, subword tokenisation copes with out-of-vocabulary words).
  - Compression to 32-d per field by averaging adjacent dimensions. The product input is 5 fields × 32 = 160-d, and the query input is 32-d.
  - Each tower is a Dense layer to 16-d, then normalise, then dot. Trained with BCE and model checkpointing on best validation accuracy.
  - Afterwards, **chop the network** into separate query and product encoders, embed every product to 16-d, and run ANN.
  - Sanity checks: ANN on queries→queries and products→products before and after training ("goodness preserved"), then free-text test queries ("Nike running shoes women", "fitness tracker" → Fitbit) return sensible products.
- **Rules of thumb.**
  - Inspect nearest neighbours by eye at every stage. It's the cheapest quality check.
  - Normalise vectors so dot product equals cosine.
  - Keep the final dimension small for latency and memory.
  - More data and richer numeric features help in production.
- **Gotchas and caveats.**
  - (note: averaging adjacent dimensions to shrink 768→32 is a crude hack that can destroy information. Prefer PCA, Matryoshka-trained models, or a learned projection.)
  - (note: random negatives are easy. Production two-towers usually train with in-batch softmax / sampled softmax and **hard negatives**, plus a correction for popular items in-batch. BCE on pointwise labels is a weaker retrieval objective.)
  - Only eyeballed results were shown, no recall@k evaluation. (note: always measure recall@K of the retrieval stage against held-out positives.)
  - "Substitute" and "complement" labels were thrown away. They could become graded relevance or separate tasks (complements suit "complete your trip" recommendations).
- **Relevance to Sam / Trameter.** A practical laptop-scale recipe.
  1. Embed Viator activities and RateHawk hotels (title, description, category, city) with a small sentence-transformer.
  2. Use clicks and bookings, or Drive Button article→trip matches, as positives.
  3. Train small towers to 32–64-d.
  4. Store the item vectors in FAISS (a 6M × 32-d float32 index is about 770MB, which fits in 16GB).
  5. Encode queries online.
  
  Blog article→bookable trip matching is the same pattern, with an article tower and an activity tower. Evaluate with recall@K on held-out matches and add hard negatives (same city, wrong activity type).

## Christopher Potts (Stanford XCS224U) — Information Retrieval, Part 1: Guiding Ideas (https://www.youtube.com/watch?v=enRb6fp5_hw)
**Retrieval is a hard language-understanding problem, so better NLU means better IR. Equally, IR improves NLP by grounding open, knowledge-heavy tasks in evidence with provenance, as opposed to "LLMs for everything", which fabricate.**

- **Why NLP is changing IR.** After BERT, Google and Bing folded transformers into core search. Then LLMs went directly into search (You.com, Bing + OpenAI) and into enterprise document search (Morgan Stanley + GPT-4). Internal search inside organisations is a large use case in its own right.
- **Why relevance is semantic.** Query "what compounds protect the digestive system against viruses?" vs document "gastric acid and proteases … defenses against ingested pathogens": **zero string overlap**, yet highly relevant. Lexical matching alone can't bridge the vocabulary gap. Deeper understanding of query and document is what bridges it.
- **Why IR is changing NLP: closed → open tasks.**
  - SQuAD-style QA is closed: you get title + passage + question, and the answer is guaranteed to be a substring. That is now easy, and it doesn't match real life.
  - **OpenQA** gives only the question. The evidence must be retrieved from a large corpus, with no substring guarantee. It is harder, but it is the real problem.
  - Other knowledge-intensive tasks: claim verification, commonsense reasoning, long-form reading comprehension, information-seeking dialogue.
  - Even closed tasks (summarisation, NLI) might improve if retrieved world knowledge is added to the input.
- **Three paradigms.**
  1. **Classical IR**: an offline **inverted index** (term → scored documents), then term lookup and document scoring at query time, returning a ranked list. The user inspects the documents.
  2. **"LLMs for everything"**: the query goes into an opaque model and an answer comes out directly. It is convenient, but there is **no provenance**, the model can fabricate, and you can't tell whether your information need was really met.
  3. **Neural IR**: encode every document offline with a language model into dense vectors (this replaces the inverted index), encode the query with the same model, then score and return a ranked list. It keeps the classical experience and its provenance but works in a richer semantic space. Potts prefers this direction.
- **Retrieval-augmented in-context learning (retrieve-then-read).** For a prompt question, retrieve (a) context passages as evidence, (b) **similar training questions as demonstrations**, and (c) evidence passages for those demonstrations too. Weave them into the prompt. This is the starting point of DSP (Demonstrate-Search-Predict). Retrieval feeds the prompt in several places, not just one.
- **Failure modes he warns about.**
  - LLM search giving **fabricated citation URLs** (404s). This is worse than no evidence, because users treat URLs as ground truth.
  - Bing citing Potts's own paper's illustrative example as evidence for a factual question. Having *a* search step doesn't guarantee *good* search, and source quality matters.
  - Fabricated biographies. Hallucination about real people has real downstream harm.
  - The conclusion: high-quality retrieval and provenance are the remedy. See also the Khattab, Zaharia and Potts posts on scalable, explainable retrieval-based NLP and on radically better AI web search.
- **Rules of thumb.**
  - Keep a ranked, inspectable evidence list even when an LLM writes the final answer.
  - Validate every cited link or item against your own store.
  - Treat retrieval quality as the bottleneck for any knowledge-intensive LLM feature.
- **Relevance to Sam / Trameter.**
  - For a "where can I go" assistant or the Drive Button, never let the LLM produce visa rules, prices or booking links from memory. Retrieve them from the travel-access database or supplier APIs and show provenance.
  - Validate every generated product or URL against the catalogue (the same lesson as invalid semantic IDs).
  - Users' trip queries often share no words with the listings ("somewhere warm, cheap, no visa for a Nigerian passport"), so use dense or hybrid retrieval plus structured filters.
  - Few-shot prompts can retrieve similar past queries as demonstrations.

## ritvikmath — Learning to Rank: The ML Problem You've Probably Never Heard Of (https://www.youtube.com/watch?v=YroewVVp7SM)
**Learning to rank is its own problem type: for a query and its candidate documents, learn a scoring function f(q, d) whose job is to *order* documents within that query's group. It is not to predict independent labels or values.**

- **The setup.** Given query q and candidate documents d1…dk, learn f(q, d) that outputs scores s1…sk, and sort by score. Example: query "cats" → animal shelters (0.9), cat care (0.7), Cats the musical (0.3). Only the **relative order within the query** matters. The absolute values don't.
- **Why it isn't classification.** You could label each document relevant / not relevant and predict a probability, but that treats documents independently. Here documents are **grouped by query and related to each other**: a different query ("dogs") has a completely different candidate set, and the question is which one should beat which.
- **Why it isn't regression.** Regression also treats rows as independent and fits values. Ranking explicitly cares about **interactions between items in the same list**.
- **But classification and regression ideas still get reused.** Coming techniques borrow from them. This maps to the standard taxonomy:
  - **pointwise**: score each (q, d) with a classification/regression loss
  - **pairwise**: learn which of two documents should rank higher (RankNet / LambdaRank)
  - **listwise**: optimise a list metric such as NDCG directly (LambdaMART / ListNet).
  
  (note: the video only motivates the problem. These names come from the field, not from this video.)
- **Applications.** Web search, **email search** (find last year's email about dogs), file-system search. Anywhere a query returns several candidates and the right one should be at the top.
- **Practical implications.**
  - Training data must carry a **query/group ID**. Split train/test by query, not by row, so a query's documents don't leak across the split.
  - Evaluate per query with ranking metrics (NDCG@k, MRR) and average over queries, not with accuracy or RMSE.
  - Scores aren't comparable across queries, so don't threshold them globally without calibration.
- **Relevance to Sam / Trameter.**
  - Ranking the Viator activities for "Lagos" (or the place candidates for a fuzzy query among 6M GeoNames rows) is exactly an LTR problem. Group by query/search session.
  - Features: BM25 or fuzzy score, population, distance, popularity, bookings, price fit.
  - Use pairwise/listwise objectives, e.g. LightGBM's lambdarank (it fits easily in 16GB RAM).
  - Labels: clicks and bookings as graded relevance, with position-bias correction.
