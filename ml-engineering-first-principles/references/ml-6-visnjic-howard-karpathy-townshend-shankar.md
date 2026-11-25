# ml-6 — cross-cutting principles

1. **Baseline first, add complexity only when it earns it.** Mode imputation, a 1R single split and a linear model come before any deep net. On small tabular data a linear model, an NN and a deep NN tied, and 1R was among the best classifiers in a 1990s review. (Howard L5)
2. **For tabular data, feature engineering beats model depth.** Domain features (titles and surnames from a Name column) beat every generic model. Deep nets pay off on images and text; tabular rewards thought about features. (Howard L5)
3. **Own the two ends of the network: inputs and final activation/loss.** Most "won't train" bugs are a missing sigmoid on binary outputs, a ReLU on the last layer, or un-normalised inputs. (Howard L5)
4. **Prepare inputs so optimisation is easy.** Normalise column scales, `log1p` long-tailed money or population, add "was missing" flags, and group rare categories plus unseen ones into "other". Apply exactly the same preprocessing at inference. (Howard L5)
5. **Debugging discipline for neural training:** overfit one tiny batch first. Note that duplicate inputs with different labels set an irreducible loss floor. (Karpathy MLP)
6. **Training discipline:** use mini-batches (many noisy steps beat few exact ones), an exponential LR sweep picking the valley before instability, then a 10× LR decay at the plateau. (Karpathy MLP; Howard's `lr_find` is the same idea.)
7. **Use the train vs dev gap to decide what to do, and find the bottleneck.** Train ≈ dev means underfitting, so scale up. Dev worse means overfitting, so regularise. Scale the bottleneck component, not everything (the 2-d embedding, not the hidden layer). Use test only a handful of times. (Karpathy MLP)
8. **Embeddings let a model generalise to unseen combinations**, because similar tokens land near each other. An embedding lookup is the same as one-hot × matrix. Use them for high-cardinality categoricals and fuzzy names. (Karpathy MLP)
9. **Use library primitives for numerics.** `F.cross_entropy` (fused, simpler backward, max-subtraction stability) instead of hand-rolled softmax. Use `.view` (no copy) instead of `cat`. Check broadcasting. (Karpathy MLP; Howard L5 on broadcasting)
10. **Split criteria must be strictly concave** (entropy or Gini, not misclassification), or real gains look like zero. Grow then prune instead of stopping early on a minimum gain, which misses interactions. (CS229 L9)
11. **Ensemble variance = ρσ² + (1−ρ)σ²/M.** More models only shrink the second term; decorrelation (bootstrap, per-split feature subsampling) is what lowers the floor. More bagged trees never add overfitting. Even averaging 5 random-seed runs helps a lot. (CS229 L9; Howard L5)
12. **Match the ensemble to the error type.** Bagging/RF reduces variance (deep trees). Boosting reduces bias (stumps or shallow trees, reweighting mistakes). Trees handle interactions but struggle with additive/diagonal structure, so supply difference and ratio features. (CS229 L9)
13. **A monitor is only as good as its telemetry.** Profile inputs, outputs and delayed ground truth per comparable time window: schema at each pipeline stage, counts, null rates, distributions, cardinality, lineage metadata, resource usage. (Visnjic/WhyLabs)
14. **Use mergeable streaming sketches over 100% of the data, not samples.** They give constant memory, are map-reducible across chunks and workers, and avoid false alarms on long-tailed distributions that samples misrepresent. (Visnjic/WhyLabs)
15. **Alerting is a human-factors problem.** Tune thresholds with the builder and the on-call person to avoid alert fatigue. Use trailing-window and training baselines. Wire severe events to automatic actions (retrain trigger, kill switch when features go null). The monitor must cost less than what it protects. (Visnjic/WhyLabs)
16. **Count LLM calls like rows in a query plan.** AI joins over cross products explode (41M calls for a 5k×5k query). Push filters down, order operators by selectivity and cost, block candidates cheaply before calling an LLM, and use embeddings plus a tuned threshold as the cheap tier. (Shankar)
17. **Estimate the speed-of-light bound, then profile for the gap.** vLLM ran at 30× the bound because of host overhead and KV-cache recomputation. Put the shared long context first in prompts, and with known workloads, cache by reuse value rather than LRU. (Shankar)
18. **Run without a fixed seed to see variance.** How much results move from run to run is information about your data and model. (Howard L5)

## Alessya Visnjic (WhyLabs), guest in Stanford CS329S — ML Monitoring (https://www.youtube.com/watch?v=uVqfYkcodeE)
**Core idea: a monitoring system is only as good as its telemetry; profile 100% of the data with mergeable streaming sketches into comparable time-window "profiles", then compare profiles to baselines and alert.**

(note: the transcript contains the talk twice; content below covers the single talk.)

**Mental models / first principles**
- Monitoring has three steps: collect the pipeline's "vitals", detect changes in them, then bring in a human (or an automated action) to root-cause. Simple on paper; hard once you're embedded in production.
- Your ML app sits inside a larger software organism with upstream producers (feature pipelines, suppliers, ETL) and downstream consumers. The monitor must see inputs, outputs and their interactions, not just the model.
- Production constraints apply to the monitor too: latency (how fast do you detect?), scale (can it keep up with full volume?), cost (a monitor must never cost more to run than the thing it protects — ROI), and long-term ownership (maintenance, automation as infra evolves).
- Telemetry is the ceiling: whatever you fail to capture, or capture too slowly or rigidly, caps everything downstream (dashboards, alerts, root-causing).
- Monitoring vs observability: monitoring tracks predefined metrics, so it only covers known unknowns and stops at "something changed, page a human". Observability captures as much as possible (logs, traces, metadata, statistics), relates them, and helps explain *why* — reasoning about unknown unknowns. (Points to Charity Majors / Honeycomb for intuition.)
- Three outputs of an observability system (Chip Huyen's framing): logs/telemetry -> dashboards (fast root-causing) and alerts (notification or automatic triggers).

**What telemetry to capture for an ML system**
- Define a *profile* = unit of telemetry over a fixed time window (hour/day/week). Windows must be comparable so you can learn what a "healthy hour/day" looks like.
- Contents: (1) lineage metadata — model version, hyperparameters, code/data versions, so a drift alert can be tied to "what changed recently" (e.g., MLflow artifacts); (2) schema, ideally at multiple points along the feature pipeline because upstream bugs break the model; for embedding-only systems, track tensor shapes instead; (3) counts — volume, missing values, number of predictions/runs; (4) statistics and full distributions of inputs AND outputs (and ground truth when it arrives); (5) a persisted stratified sample of raw data for debugging; (6) resource stats (memory/CPU in train vs prod, dev vs prod) — OOMs and resource differences point at root causes.

**Mechanisms (whylogs design)**
- One line to profile a DataFrame (also images). Runs locally — data never leaves your environment; only small profiles are shipped. Profiles are near-anonymous summaries.
- Lightweight via stochastic streaming sketches (Apache DataSketches): single pass, near-constant memory regardless of rows, output of a few MB per profile. Scales with number of features, not rows.
- Profiles are mergeable/map-reducible: compute per worker or per 1–10 s micro-batch, merge into hourly/daily profiles. This needs density/sketch-based stats (non-parametric mergeable histograms, frequent-items for categoricals, HyperLogLog-style cardinality estimates), not plain pandas `describe()`.
- Process 100% of data rather than sampling: sampling rates get cut as volume grows, and long-tailed distributions get misrepresented by samples, producing false drift alarms.
- Uses: one profile = documentation/audit trail of what data went through; two profiles = baseline vs test comparison (train vs val, train vs today's prod); a series of profiles = time series to monitor.
- Demo: Lending Club data split into 7 "days"; monitor computed Hellinger distance between distributions, tracked median, and flagged a sudden jump in missing values on day 4 with zero configuration.
- Baselines: trailing window (past 2/3/7 days) or a fixed training-data baseline; thresholds via std-dev bands; change detection on the profile time series uses simple transparent models (ARIMA-like, non-parametric) rather than ML, for stability.

**Decisions & trade-offs**
- Trailing-window baseline catches sudden breaks but absorbs slow drift; training-data baseline catches drift from what the model learned but alerts persistently once the world moves. Use both where it matters.
- Auto-configured monitors give a warm start for models with ~150 features; nobody hand-tunes thresholds per feature. Then tune the noisy ones.
- Alert actions can be automatic: drift past a threshold -> kick off retraining; catastrophic data-quality event (e.g. 90% of features null) -> stop serving predictions (a kill switch).

**Rules of thumb**
- Profile inputs, outputs and (delayed) ground truth on the same time grain.
- Capture schema at every pipeline stage, not just at the model boundary.
- Keep the monitor's cost and memory small and constant; prefer sketches over storing raw data.
- Bundle drift alerts with lineage so "what changed?" is one lookup.

**Failure modes**
- Alert fatigue: a monitor that cries wolf gets ignored; one that never alerts gets blamed. Tune thresholds with both the system builder and the on-call person; ask whether an alert is one user on an odd browser or a mass event. Consider how alerts drive downstream automation.
- Sampled statistics on long-tail data -> false alarms.
- Monitoring only the model and missing upstream feature-pipeline breakage.

**Relevance to Sam**
- For the ~6M-row GeoNames/OSM pipelines and supplier feeds (Viator, RateHawk), build a profile per ingest batch/day: row counts, null rates per column, distinct-count (cardinality) of country/feature codes, frequent values, lat/lon histograms. Diff each new batch against the last good one and alert on jumps. That catches silent supplier schema changes and broken joins.
- Mergeable sketches fit a 16GB laptop: profile each chunk in a chunked/parallel job, then merge. whylogs (or DuckDB `approx_count_distinct`/`approx_quantile`) gives this in constant memory.
- For a matcher/classifier in production, profile the output score distribution and the match rate per day. A drop in match rate is often the first sign of upstream drift.

## Jeremy Howard — Lesson 5: Practical Deep Learning for Coders 2022 (https://www.youtube.com/watch?v=_rXzeWq4C6w)
**Core idea: a tabular linear model, a neural net and a deep net are the same few lines (matmul, activation, loss, gradient step). On small, simple tabular data they all tie, so feature engineering, simple baselines (1R) and cheap ensembles matter more than depth.**

**Mental models / first principles**
- Linear model = sum of coefficient times feature. Neural net = matmul, then ReLU, then matmul. Deep net = repeat that pattern. The only architecture details you must own are the **inputs** (they change with every dataset) and the **final activation/loss** (it changes with every target). The middle matters less.
- Normalise features so one random coefficient means roughly the same thing for each column. Otherwise a big-range column (Age, about 0–80) dominates and optimisation becomes hard. Dividing by max or z-scoring both work; pick the easy one.
- Squash binary outputs through a sigmoid. Then the optimiser only has to push logits large or small instead of hitting exactly 0/1. In his example this let the LR rise from 0.1 to 2, loss fell 0.295 -> 0.197, and accuracy went 0.79 -> ~0.83. If a binary model trains poorly, first check whether a sigmoid is (or isn't) applied, including inside libraries like HF Transformers.
- Loss and metric differ: accuracy has no useful gradient, so train on MAE/cross-entropy and report the competition metric separately.
- Run without a fixed seed when not teaching. Seeing run-to-run variance is part of understanding your data and model.

**Mechanisms**
- Missing values: the simplest fix that always works is filling with the column mode (works for categorical and continuous; take `.mode().iloc[0]` because ties return several rows). Don't drop rows or columns. Better still, add a boolean "was missing" column, since missingness itself can predict (e.g. empty Cabin).
- Long-tailed money/population columns: use `log1p` (log(x+1) avoids log 0). A "$" in the column is the hint. Linear models and NNs dislike long tails.
- Categoricals -> dummy columns (all n levels means no constant term is needed; n-1 levels plus an intercept is the alternative). A number that is really a code (Pclass 1/2/3) can be treated as categorical.
- Broadcasting (from APL): matrix x vector multiplies each row by the coefficient vector with no copy and no Python loop. It runs in optimised C/CUDA, which is why Python can be fast. Learn the rules: trailing axes must match. `x[:, None]` adds a unit axis (vector -> n×1 matrix) so targets line up with matmul outputs.
- Tensor vocabulary: shape; rank = len(shape) (scalar 0, vector 1, table 2).
- Autograd: `requires_grad_()` (a trailing underscore means in-place), `loss.backward()` fills `.grad`, then `coeffs.sub_(coeffs.grad * lr)` and zero the grad.
- Hand-built NN init is fiddly: scale the first-layer weights by 1/n_hidden so the next layer's sums don't blow up. Layer 2 needs a bias/constant. Getting init ranges and LR right by hand took a lot of trial and error, which is exactly what frameworks save you.
- Common architecture bug: wrong final activation (a ReLU on the last layer, or a missing sigmoid). Use ReLU on hidden layers only.
- LR finder: raise the LR exponentially from about 1e-7, one mini-batch per step, and plot loss. Choose a value between the "slide" and "valley" suggestions (e.g. 0.01 and 0.08 -> 0.03).
- Train/inference consistency: apply exactly the same preprocessing at inference. fastai's `test_dl` replays the learner's transforms. Hand-copying steps causes skew. The test set can also contain new problems, e.g. a NaN Fare absent from train.
- Unseen categories at test time: with hand-made dummies a new level is silently ignored. fastai maps unknown and rare levels to an "other"/#na# category. Group rare levels yourself.
- Ensembling: train the same model 5 times from different random inits and average the probabilities. It moved Titanic from about top-50% to top-25%. Options: average probabilities then threshold (usually best), average 0/1 votes, or take the mode (rarely better).

**Random forests / binary splits (start of next topic)**
- RFs are hard to misuse. In industry he has seen far more broken logistic regressions (wrong transforms, interactions, outlier handling) than broken RFs.
- Trees need no dummies. Pandas `category` dtype stores integer codes (`.cat.codes`), and ordinal codes are fine for splits.
- Scoring a binary split: for each side, std of the target × group size; sum both sides and average. Lower is better, and it works for classification and regression targets alike. Find the best threshold per column by trying every unique value and taking the argmin. Sex beat every other column on Titanic (~21.5% error alone).
- **1R** (best single split) was among the best classifiers across real datasets in a 1990s review. Always build it as a baseline. Kaggle's Titanic sample submission is just male/female.

**Decisions & trade-offs**
- On small tabular data with few simple columns, deep learning gave no gain over the linear model (same accuracy). Images and text reward throwing a deep net at raw data. Tabular rewards careful feature engineering, and richer models only pay off once there are many features or high-cardinality categoricals.
- Domain features can beat any generic model. A Titanic notebook built only on the Name column (titles, shared surnames as family) beat everything in the lesson.
- Do it from scratch to understand, then use a framework so that only your deliberate deviations are manual.

**Rules of thumb**
- Build the dumbest baseline first (mode-imputation, 1R, linear). Add complexity only when it is shown to help.
- Write code step by step in the notebook, then wrap the working cells into functions. Keep the exploration as documentation.
- Eyeball learned coefficients for sanity (older -> lower survival, male -> lower).

**Relevance to Sam**
- Place classification on tabular attributes (feature codes, population, name tokens): log-transform population and similar long-tailed counts, add "is missing" flags, group rare categories into "other", and start with a 1R/single-tree baseline before gradient boosting.
- Name strings hold strong signal (titles in Titanic; for places: "Saint", "Airport", "Station", admin suffixes, shared tokens). Engineer tokens before reaching for a bigger model.
- For any matcher/classifier, keep one preprocessing function used at both train and inference time, and make it handle unseen categories explicitly.
- Cheap seed-ensembles (5 runs, averaged probabilities) are an easy accuracy gain that fits on a laptop.

## Andrej Karpathy — Building makemore Part 2: MLP (https://www.youtube.com/watch?v=TCH_1BHY58I)
**Core idea: replace an exploding count table with learned embeddings plus an MLP (Bengio et al. 2003), and train it the disciplined way: overfit one batch, use mini-batches, sweep the LR, keep train/dev/test splits, and diagnose under- vs overfitting from the train–dev gap.**

**Mental models / first principles**
- Count-based n-gram models blow up exponentially with context length. With 27 characters: 1 char of context = 27 rows, 2 = 729, 3 ≈ 20k rows. Most contexts then have too few counts to estimate anything.
- Embeddings fix this. Each token gets a small learned vector (Bengio: 17k words in a 30-d space). Similar tokens drift close together during backprop, so knowledge transfers to contexts never seen ("a dog was running in a ___" generalises from "the dog…" because a≈the and dog≈cat). This is how you generalise out of distribution.
- An embedding lookup `C[idx]` is mathematically the same as one-hot(idx) @ C, i.e. a linear layer with no nonlinearity. Indexing is used because it's far faster.

**Mechanisms (architecture and code)**
- Dataset: a rolling context window of `block_size` (3) previous tokens, padded with a start token, predicting the next one. 32k names produce about 228k training examples.
- Forward pass: `emb = C[X]` (B×3×d) -> flatten to B×(3d) -> `tanh(emb @ W1 + b1)` -> `logits = h @ W2 + b2` (B×27) -> cross-entropy. The output layer is the expensive one when the vocabulary is large.
- PyTorch indexing accepts multi-dimensional integer tensors: `C[X]` embeds a whole B×3 batch in one call.
- Flatten with `.view(B, -1)`, not `torch.cat(torch.unbind(...))`. A view only changes shape/stride/offset metadata over the same 1-D storage (no copy). `cat` allocates new memory.
- Broadcasting check: (32×100) + (100,) aligns on the right, becomes 1×100 and adds the bias to every row. Always confirm the broadcast does what you intend.
- Use `F.cross_entropy`, not hand-rolled exp/normalise/log:
  1. Fused kernels with no intermediate tensors, so a faster forward pass.
  2. A simpler analytic backward pass, like tanh' = 1 − t².
  3. Numerical stability. exp(100) overflows to inf, giving NaN. Softmax is invariant to adding a constant, so subtracting max(logits) first keeps everything finite.
- Training loop: zero grads (`p.grad = None`), `loss.backward()`, then `p.data += -lr * p.grad`. Set `requires_grad=True` on all parameters.

**Training procedure / debugging discipline**
- **Overfit a single small batch first.** 3.4k params on 32 examples should drive the loss near 0. It can't reach exactly 0 when identical inputs have different labels (context "..." -> e, o, a, s). That irreducible ambiguity puts a floor on the loss.
- **Mini-batches**: random 32-example index batches via `torch.randint`. A noisy gradient with many cheap steps beats an exact gradient with few expensive steps. Evaluate the loss on the full train set separately, since the mini-batch loss is noisy.
- **LR range test**: find bounds by hand first (1e-3 barely moves, 10 diverges). Then sweep LR exponentially, `lrs = 10**torch.linspace(-3, 0, 1000)`, one step each, and plot loss vs exponent. Choose the valley before it gets unstable (here exponent ≈ −1, LR 0.1).
- **LR decay**: train at the found LR, and when the loss plateaus, cut it 10× for the last stretch (e.g. 100k steps at 0.1, then 100k at 0.01).
- Plot log10(loss) to remove the hockey-stick shape. A thick loss curve means mini-batch noise; a bigger batch smooths it.
- Splits: about 80/10/10 train/dev/test. Train fits parameters, dev picks hyperparameters (hidden size, embedding dim, context length, regularisation, LR schedule, batch size), test is used only a few times at the very end. Each look at test leaks it into your decisions.
- Diagnosis:
  - train ≈ dev loss means **underfitting**, so scale the model up.
  - train noticeably below dev means overfitting has started.
  - Here, a bigger hidden layer (100 -> 300) didn't help, so the **bottleneck was the 2-d embedding**. Embedding 10-d with 200 hidden units moved dev from 2.24 to about 2.17. Find which component is the bottleneck instead of growing everything.
- Inspect learned embeddings. In 2-d, vowels clustered together, while 'q' and the special '.' token sat apart: evidence of learned structure.
- Sampling: start from the pad context and loop: `softmax` -> `torch.multinomial` -> shift the window, until the end token appears.
- Results: bigram 2.45 -> MLP ~2.17 dev with ~11k params.

**Decisions & trade-offs**
- Knobs: hidden width, embedding dim, context length, batch size, LR schedule, number of steps. In production you'd turn these into hyperparameters and run many experiments, not hand-tweak inline.
- Bigger models can memorise. The training loss alone says nothing about quality; samples then just echo training data.

**Failure modes & gotchas**
- dtype mismatch: `one_hot` returns int64, so cast it to float before a matmul.
- Hard-coded shapes (the "6" that had to become "30" when the embedding dim changed). Derive shapes from the tensors, or use -1.
- Re-running an init cell wipes training progress. Keep init separate from the training loop.
- Accidentally training at the decayed LR. Track which LR is active.

**Relevance to Sam**
- Character/token embeddings are the principled answer to fuzzy place-name matching at scale. Train or borrow char n-gram embeddings so that "St." ≈ "Saint" and transliterations sit close together, then use vector similarity for candidate blocking across millions of GeoNames/OSM rows.
- Embedding tables for high-cardinality categoricals (country, admin1, feature code, supplier ID) generalise better than one-hot for place classification and travel ranking.
- The checklist applies to any model he trains on the laptop: overfit a tiny batch, LR sweep, mini-batches, train/dev gap to choose bigger vs regularise, and locate the bottleneck component.

## Raphael Townshend (Stanford CS229) — Lecture 9: Decision Trees and Ensemble Methods (https://www.youtube.com/watch?v=wr9gUr-eWdA)
**Core idea: a decision tree greedily partitions feature space using a strictly concave impurity loss. It is low-bias and high-variance, which makes it the ideal base learner for ensembles: bagging/random forests reduce variance by decorrelating trees, and boosting reduces bias by adding shallow learners that focus on past mistakes.**

**Mental models / first principles**
- Example: "can I ski?" from (month, latitude). Positives sit in two separated regions (northern winter, southern winter) and a linear boundary can't capture that. A tree plays "20 questions": greedy, top-down, recursive partitioning. Finding the globally optimal partition is intractable, so it takes the best split at each step.
- A split is S_p(j, t): feature j and threshold t send the parent region R_p to R1 = {x_j < t} and R2 = {x_j ≥ t}. Choose j, t to maximise L(R_p) − (L(R1) + L(R2)), the loss decrease or information gain.

**Mechanisms — split criteria**
- Misclassification loss, 1 − max_c p̂_c, is **too insensitive**. With a parent of 900+/100−, the split (700+/100−, 200+/0−) and the clearly better split (400+/100−, 500+/0−) both leave 100 errors, so the loss reports zero gain.
- Cross-entropy, −Σ p̂_c log p̂_c, and Gini, Σ p̂_c(1 − p̂_c), are **strictly concave** in p̂. Geometrically, the weighted child loss lies on the chord between the two children, which is strictly below the curve at the parent's p̂. Any split that changes class proportions therefore registers a gain. Misclassification is piecewise linear (a tent), so children on the same side of 0.5 show no gain. Good split criteria are all strictly concave. The log base doesn't matter.
- Regression trees: a leaf predicts the mean of its region, and the split criterion is squared error around the leaf means.
- Categorical features: a split can be any subset of the q categories, which means 2^q candidates and becomes intractable for high cardinality. Exception: for binary classification (and squared-error regression), sort categories by their positive rate (or mean target) and scan linearly; this finds the optimal split.

**Regularisation (trees are high variance; a fully grown tree gives one leaf per point)**
- Heuristics: minimum leaf size, maximum depth, maximum number of nodes.
- **Avoid a "minimum loss decrease" stopping rule.** With interacting features, a weak first split (latitude alone) can enable a very strong second one (latitude + month, i.e. XOR-like structure). Early stopping misses it. Instead, grow the full tree and **prune back** using validation-set misclassification.

**Runtime**
- Test: O(depth), and depth is usually about log n for a balanced tree.
- Train: each point appears in O(d) nodes, and each node costs O(f) per point (pre-sort continuous features and scan), so total O(n·f·d). That is a few passes over the n×f data matrix, i.e. fast.

**Pros / cons of single trees**
- Pros: easy to explain, interpretable, handle categoricals natively, fast.
- Cons: high variance (overfit), **poor at additive structure** (a diagonal boundary x1 = x2 needs a staircase of many splits, which logistic regression gets for free), and therefore low accuracy alone. They are worth learning because tree ensembles dominate Kaggle-style tabular work.

**Ensembling theory**
- For identically distributed estimators with variance σ² and pairwise correlation ρ, the variance of their mean is ρσ² + (1 − ρ)σ²/M. More models (M) shrink the second term. **Decorrelation (lower ρ)** shrinks the first term, and only that can push variance below the ρσ² floor.
- Ways to ensemble: different algorithms (NN + RF + SVM averaged, which works but is costly to build), different training sets (usually impossible), bagging, boosting.

**Bagging / random forests**
- Bootstrap: treat the training set S as the population and draw M resamples of size n *with replacement* (each holds about 2/3 of the unique points). Train a model on each and average.
- Adding more bagged models **does not increase overfitting**; it only lowers the (1 − ρ)σ²/M term. Keep adding models until validation error plateaus.
- Cost: slightly higher bias, because each model sees about 2/3 of the data. This is usually far outweighed by the variance reduction.
- Trees + bagging is a natural fit because tree error is mostly variance.
- Random forest adds **feature subsampling at each split**. Otherwise one dominant feature would be every tree's first split, keeping trees highly correlated. Restricting the candidate features lowers ρ further.

**Boosting**
- Reduces bias, and builds the model additively and sequentially. Use high-bias, low-variance base learners such as depth-1 stumps.
- AdaBoost: fit a stump, up-weight misclassified points, fit the next stump on the reweighted data, and so on. Each learner gets weight α_m ≈ log((1 − err_m)/err_m), and G(x) = Σ α_m G_m(x). Gradient boosting / XGBoost generalise the same additive, focus-on-errors idea.

**Decisions & trade-offs**
- Bagging/RF suits high-variance base learners (deep trees). Boosting suits high-bias ones (stumps or shallow trees).
- Very high-cardinality categoricals are a problem for exact subset splits. Use target-ordered encoding (the sorted-by-positive-rate trick), or group rare levels.

**Failure modes & gotchas**
- Using misclassification error as the split criterion stalls tree growth.
- Early stopping on minimum gain misses interactions. Prefer grow-then-prune.
- Bootstrap models stay correlated because they share S, so bagging alone has a variance floor. Feature subsampling is what breaks it.

**Relevance to Sam**
- Place-type classification and match/no-match classification on engineered pair features (name similarity, distance in km, same country/admin, feature-code agreement) are classic tree-ensemble territory. Interactions like "distance small AND name similar" are exactly what trees capture and a linear model misses.
- One drawback: distance and similarity often act additively or along diagonals. Supply ratio and difference features explicitly so trees don't need staircases.
- Feature codes and country codes are high-cardinality. Order them by target rate (or use native categorical handling in LightGBM/CatBoost) instead of one-hot encoding.
- On a 16GB laptop, training cost O(n·f·d) is cheap. Random forests parallelise across cores (`n_jobs=-1`), and histogram-based GBMs (LightGBM) bin features to save memory on millions of rows.

## Shreya Shankar — devreal.ai talk at SF Systems Club: query-aware LLM inference for AI-SQL (https://www.youtube.com/watch?v=V_2GnrKodqE)
**Core idea: LLM-powered SQL operators (AI filter/join/classify/extract) turn modest queries into millions of LLM calls. General-purpose chat-serving engines waste most of the GPU on them, so treat the workload as a database query. Plan it with cost models (filter pushdown, join ordering), batch it, and manage KV-cache reuse explicitly. Measure against a "speed of light" lower bound.**

**Mental models / first principles**
- Zero-shot LLMs now handle classic text operations (extraction, classification, summarisation) without collecting training data or carrying MLOps burden. Cost has dropped about 1000× in roughly 3 years, and open models are now good at it.
- About 80% of data is unstructured. "AI-SQL" adds LLM-backed relational operators: `SELECT * FROM reviews WHERE AI_FILTER('does the review discuss the ending?')`, plus AI_JOIN, AI_CLASSIFY, AI_EXTRACT and AI_COMPLETE (free-form map).
- Cost blow-up: an AI filter makes O(n) calls, an AI join makes O(n·m) because it runs an LLM over the cross product. Multi-class classification can be expressed as a join. One benchmark query on about 5k biomedical reports × about 5k reaction terms needed roughly **41 million LLM calls**.
- Standard levers to cut calls: reduce or sample calls, replace expensive models with cheaper ones where quality matches empirically, replace the LLM with code where possible.
- Filters and joins are **prefill-only** (one yes/no token, almost no decode). That should make the GPU very efficient, yet engines like vLLM are tuned for interactive chat and agents (many users, minimise per-request latency). AI-SQL is offline batch: all requests are known up front and only time-to-last-token matters.

**Mechanisms**
- **Speed-of-light estimate**: count the unavoidable FLOPs (matmuls, attention, MLPs) assuming 100% GPU utilisation and unlimited KV cache, giving an optimistic lower bound (894 s for the query). vLLM took about **30× that**. The PyTorch profiler showed GPU "bubbles" from (1) host/CPU overhead processing 40M requests while the GPU idles, and (2) **"KV regret"**: re-computing prefix KV that was computed earlier but evicted by the LRU cache.
- Hand-optimising on vLLM: one prompt per input, and in joins put the long shared document first and the short partner second, so the shared prefix's KV can be reused across requests.
- Quail (query-aware inference layer), built like a query engine (inspired by DataFusion, data in Arrow). Its goals are no KV regret, a GPU that is never idle, and high MFU (left to the inference experts).
  - Planner: start from dataset statistics (rows, column widths). Size batches to saturate compute while keeping the relevant KV resident (from GPU HBM size and model size).
  - Classic rewrites: **filter pushdown and optimal filter ordering** (Hellerstein), and **Selinger-style dynamic-programming join ordering** extended to account for KV reuse.
  - Inference-specific rewrites: e.g. with a single filter, don't write KV to HBM at all.
  - Executor: pull-based and vectorised, like a disk-based database with too little memory for every row's KV.
  - KV policy: after a filter passes, "rewind" to keep only the document's KV, and only if another operator will reuse it. Drop KV for documents that fail.
  - Eviction: **evict the shortest documents first, not LRU**. Everything cached will be reused and attention is quadratic, so keeping the long documents saves the most recompute.
  - Join attention: load the shared document KV once plus all "partner" suffixes, decompose attention into two calls and merge (cascade-style, cf. the Hydragen/shared-prefix attention line of work). Much faster when a batch heavily reuses one document. AI agents were used to write small fused kernels (e.g. quantisation).
- Results on their benchmark (29 queries): better than tuned vLLM on all but one. Up to **14×** faster on the biomedical join, where document lengths are very asymmetric. The exception is agent traces: they have perfect prefix sharing, so without automatic prefix caching Quail had about 500× more KV regret. Lesson: don't discard generic machinery like prefix caching.
- Demo: 10k IMDb reviews (about 500 tokens each) with two AI filters on an open Qwen3 model cost about $0.03, versus $0.17 on GPT-5 nano, which was also less accurate on their benchmark.

**Decisions & trade-offs**
- Embeddings vs LLM predicate: embeddings are far cheaper and fine for some filters, especially when only k rows will pass. But they need a tuned similarity threshold, which means labelled samples and upfront work, and they fail on hard natural-language predicates. LLMs handle arbitrary questions. (She mentions a report of a search vendor replacing its reranker with an LLM; the model name in the transcript is garbled.)
- Batch/OLAP vs online: this design assumes read-heavy analytical workloads. For write-heavy, frequently changing data, vLLM-style online serving may fit better. An OLTP-style AI-SQL engine is open work.
- Domain-specific engines win when the access pattern is known in advance (all requests up front), but reuse general components (vLLM model code, FlashAttention/FlashInfer kernels, prefix caching) where they already fit.

**Rules of thumb**
- Before optimising, compute a speed-of-light bound and profile for GPU idle gaps. A large gap means overhead or recompute, not model FLOPs.
- Count LLM calls in your query plan the way you'd count rows. Avoid cross-product joins; filter first (pushdown), order operators so the most selective and cheapest run first.
- Order prompt text so the reusable part is the prefix (long shared document first, short varying part last).
- With a known workload, cache by expected reuse value (length × future uses), not recency.

**Failure modes**
- Naively lowering an AI join to n×m prompts gives an explosion of calls.
- LRU KV eviction in batch workloads, and host-side request overhead at tens of millions of requests.
- Custom engines regress on workloads the generic one already handled well (agent traces).

**Relevance to Sam**
- Directly applicable to LLM-based place classification or matching over millions of GeoNames/OSM/supplier rows. Never do an LLM "join" over the cross product. First block candidates cheaply (geohash/country + name trigram or embedding similarity), then send only the surviving pairs to the LLM as a filter. That is filter pushdown in practice.
- Put the long shared context (instructions + category taxonomy, or the anchor place record) at the start of the prompt and the varying item at the end, to maximise prefix-cache hits in vLLM/Ollama or provider prompt caching.
- Use the $/1k-items comparison and a speed-of-light estimate to decide between a small local open model (Qwen-class) and an API nano model. Benchmark accuracy on a labelled sample first. Embeddings plus a tuned threshold are the cheap first tier, and the LLM is reserved for ambiguous cases.
- On a 16GB laptop without a big GPU, the same principles (batch everything, reuse prefixes, filter before you join) are what make LLM enrichment affordable.
