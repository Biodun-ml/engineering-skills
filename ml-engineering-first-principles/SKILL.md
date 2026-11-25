---
name: ml-engineering-first-principles
description: First-principles ML engineering playbook from 27 lectures (Karpathy, Andrew Ng, Jeremy Howard, Zinkevich, FSDL, Shreya Shankar, WhyLabs, StatQuest). Use when framing, training, debugging, evaluating, shipping or monitoring any model, tabular or neural, even if "ML" isn't said.
---

# ML engineering from first principles

This skill turns 27 lectures and talks into working judgment for the whole ML loop, from
framing to monitoring. This file holds the procedure, the recurring principles and checklists.
`references/` (when present) holds detailed notes for every video, credited to its teacher.

Use it whenever a learned model is involved, even unnamed: a match/no-match or place-type
classifier, ranking, a loss that won't go down, "is this model good enough?", validation worse
than training, a retraining job, a drift alert, or heuristic vs GBM vs neural net.

**How to use it**
1. Follow the working procedure for any new model or model change.
2. Run the relevant checklists (section 3) against the design, code or training run.
3. If a `references/` folder sits next to this file, use it for depth. `references/INDEX.md`
   lists every video with its line number, or run `grep -n -i "keyword" references/*.md`. Each section ends with a "Relevance to Sam" or "For Sam" note: Sam matches and
   dedups millions of location rows (GeoNames ~6M, OSM, supplier feeds like Viator and
   RateHawk), classifies places, and runs everything on a 16 GB laptop. Use those parts
   only in that context. Lines marked "(note: …)", "reviewer addition" (all of §2.9) and
   the Sam notes are helper or practitioner additions, not the speakers' words. If `references/` is missing, the full notes live in the user's
   private GitHub repo `Biodun-ml/engineering-skills` (folder
   `ml-engineering-first-principles/references/`); clone it when you need depth.
4. Explain the *why*, and credit the teacher when you use a specific pattern.
5. Library defaults, tool names and hosting products in the lectures date quickly (several are
   flagged in the notes). Check current versions before recommending a specific API or setting.

---

## 1. Working procedure

1. **Frame it as task T, metric P, experience E.** Write down what is predicted, the single
   number that says "better", and where labelled data comes from. If you can't name the metric
   and the labels, you aren't ready to train. *(Ng — CS229 L1, Mitchell's definition)*
2. **Ship without ML first if you can.** A heuristic (popularity, recency, rating, distance)
   gets you data from the real problem. Move to ML when the heuristic becomes harder to
   understand than a model. Instrument many metrics before any model exists. *(Zinkevich)*
3. **Pick a simple, observable, attributable first objective** (clicks, bookings, confirmed
   matches) rather than "user happiness". Train on a loss with a gradient, report the business
   metric separately. *(Zinkevich; Howard L5 — loss vs metric)*
4. **Build the splits from the deploy distribution**, by group/time where rows correlate, and
   confirm every feature exists at prediction time (§2.2, §2.9) before training anything.
   The dev set is the problem spec; never change the test set midway. *(Ng — Nuts and Bolts)*
5. **Train a trivial baseline on day one**: the mean or base rate, ln(K) for uniform guessing,
   1R (best single split), logistic regression. Most projects wait far too long to train their
   first model. *(Howard L1/L3/L5; Ng L12; Starmer; Karpathy)*
6. **Use that first model to clean the data**: sort examples by loss and review the top. Then
   retrain. *(Howard L2; Frye)*
7. **Compute the five error numbers and diagnose** (human, train, train-dev, dev, test) before
   choosing what to try next. Brainstorm every option, then pick using evidence, not the last
   paper you read or the loudest voice. *(Ng — Nuts and Bolts, CS229 L12)*
8. **Add one component at a time and make each earn its place with a number.** Karpathy's GPT
   went bigram 2.5 → attention 2.4 → multi-head 2.28 → FFN 2.24 → residual 2.08 → LayerNorm
   2.06 → scaled 1.48 val loss. Know your seed-to-seed noise first; you can't claim a gain
   smaller than it. *(Karpathy — GPT; Zinkevich)*
9. **Do error, ceiling and ablation analysis** on the dev set to decide where effort goes.
   *(Ng — CS229 L12)*
10. **Ship early behind a gate**: validate against production on a fixed check, shadow mode,
    gradual rollout, fallback. Deploying is part of finding out whether the model is good.
    *(Zinkevich; Tobin)*
11. **Monitor inputs, outputs and delayed labels per time window; retrain on a schedule; gate
    every retrained model** against other time windows, not the batch it trained on.
    *(Visnjic; Shankar)*

---

## 2. Principles that recur across the teachers

### 2.1 Framing and baselines
- **A model is (representation, loss, optimiser).** Write all three down. The loss follows
  from a noise assumption: least squares is MLE under Gaussian IID noise, cross-entropy is MLE
  for Bernoulli labels; switch to a GLM (e.g. Poisson for counts) when that doesn't hold.
  *(Ng — CS229 L2, L3; Karpathy — micrograd)*
- **Don't classify with linear regression plus a 0.5 threshold.** One far-away point moves the
  boundary. Logistic regression has a concave log-likelihood (no local optima) and is one of
  Ng's two most-used algorithms. *(Ng — CS229 L3)*
- **Start interpretable and calibrated**, not "a seven-layer neural factor model". Zinkevich's
  test is mean prediction = mean label; check per-bin calibration too (§2.9). *(Zinkevich)*
- **Prefer a pipeline to end-to-end unless you have lots of paired (x, y) data.** Pipelines
  let you inject known structure (bone detection → age table). End-to-end projects chosen
  because x and y exist make cherry-picked demos that fail at scale. *(Ng — Nuts and Bolts)*
  (note: pretrained embedding/LLM models now make end-to-end viable with few labels for
  text-heavy subproblems such as name normalisation; keep the pipeline so ceiling analysis
  still works.)
- **Transfer learning for images and text.** For tabular data, Howard (2022) said no pretrained
  model fits arbitrary tables. *(Howard L1)* (note: as of 2025–26, tabular foundation models
  such as TabPFN v2/2.5 and TabICL are competitive on small tables, roughly ≤10–50k rows, with
  no tuning; try one as a baseline on small labelled sets. For millions of rows, GBMs remain
  the default.)
- **Heuristic patches on top of a model rot.** Hard constraints and utility weights are fine;
  "prediction × fudge factor" hurts once the model improves. Document and revisit them.
  *(Zinkevich)*

### 2.2 Data first, splits and leakage
- **Most of the work is data and train/serve consistency, not algorithms.** Spend early effort
  on data with a small, fast model; fancy architectures come last ("the number-one beginner
  mistake"). *(Zinkevich; Howard L3)*
- **Dev and test come from the distribution you will deploy on.** Put large off-distribution
  data in train; split the small in-distribution data into dev and test. Carve a train-dev
  slice from the training distribution to measure mismatch. *(Ng — Nuts and Bolts)*
- **IID is usually false.** Errors for nearby or same-street items are correlated, so split by
  time, region or group, not at random, or near-duplicates leak across splits and inflate
  scores. Validate retrained models across time windows. *(Ng — CS229 L3; Karpathy — held-out
  tail split; Shankar)*
- **Leakage also comes from features**: anything computed after the label, or by the process
  that produced it, and preprocessing fitted on all data. See §2.9.
- **Use test a handful of times only.** Train fits parameters, dev picks hyperparameters, and
  each look at test leaks it into your decisions. ~80/10/10 is Karpathy's default for small
  data. *(Karpathy — makemore MLP)* (note: at millions of rows, size dev/test by the confidence
  you need, a few thousand to tens of thousands of examples per slice (e.g. 98/1/1), not by
  percentage; Ng makes this point in his deep learning courses.)
- **Log features at serving time.** Logging only IDs and re-joining features offline causes
  skew; fixing it gave some Google teams dramatic gains. *(Zinkevich)*
- **Audit filters when copying pipelines.** Dropped data is invisible (Google+ kept a "drop old
  posts" filter; a team logged only shown items and lost the negatives it needed). *(Zinkevich)*
- **You only observe what you showed.** Choose negatives from the true candidate set and handle
  position effects, or you build feedback loops. *(Zinkevich Q&A)*
- **Data is curated, not given**: transform, filter, dedupe, mix. Filtering is an efficiency
  lever. Keep private eval sets to avoid contamination. *(Liang — CS336 L1)*
- **Synthetic data must match the real distribution**, not just look rich (20 GTA cars). *(Ng)*
- **Stay close to the data**: model developers label fresh production data on a rotation; look
  at the system by hand weekly. *(Frye via Shankar; Zinkevich; Howard — look at every step)*

### 2.3 Diagnosis: error numbers, bias and variance
- **The five numbers** *(Ng — Nuts and Bolts)*:
  human → train = avoidable bias; train → train-dev = variance; train-dev → dev = distribution
  mismatch; dev → test = overfitting the dev set (get more dev data). Examples: 1/5/6% is bias,
  1/2/6% is variance, 1/5/10% is both. 8% train / 10% dev is variance if human is 7.5%, bias if
  human is 1%.
- **Use the best human level** (a team of experts) as the Bayes-error proxy. Once past it, find
  slices where you are still worse. *(Ng)*
- **Fix the problem you have.** Bias → bigger model, better features, train longer, less
  regularisation; more data never fixes bias. Variance → more data, regularisation, fewer
  features. Mismatch → more target-like data, synthesis. New architecture is the hard last
  resort. In deep learning the trade-off is weak *when you can keep growing both model and
  data*: bigger model cuts bias, more data cuts variance (see §5). *(Ng — CS229 L12, Nuts and Bolts; Frye — "scale fixes most things")*
- **Learning curves** (error vs training-set size, with a target line) are Ng's most-used tool
  and are cheap to run on subsamples before you pay for more labels. *(Ng — CS229 L12)*
- **Optimiser or objective?** Evaluate your surrogate loss J on a competitor's parameters. If
  the competitor scores better on *your* J, your optimiser failed (train longer, fix code). If
  you win on J but lose on the business metric, J is wrong. *(Ng — CS229 L12)*
- **Ceiling analysis for pipelines**: on the dev set, substitute ground truth for each stage in
  turn and measure the end-to-end gain (face pipeline: perfect face detection +~6 points,
  perfect background removal +0.1). Someone wrote a PhD on the +0.1 stage. *(Ng — CS229 L12)*
- **Ablation** explains a gain over a simple baseline and shows which expensive parts can go
  (spam 94% → 99%; removing the text parser cost ~4%). *(Ng — CS229 L12)*
- **Use the model to find dirty data and named failure slices.** Confidently wrong and
  unconfidently right items expose mislabels; the confusion matrix separates real ambiguity
  from label noise. Group high-loss cases into named suites and track metrics per slice.
  *(Howard L2; Frye)*
- **Find the bottleneck component; don't grow everything.** A bigger hidden layer didn't help
  because the 2-d embedding was the limit; BatchNorm gave nothing because context length was.
  *(Karpathy — makemore MLP, Part 3)*

### 2.4 Training and debugging neural nets
- **Understand one level down; backprop is a leaky abstraction.** Local derivative × upstream
  gradient, in reverse topological order; gradients accumulate (`+=`). Saturation, dead units
  and odd bugs (clipping the *loss* instead of the gradient silently zeroes outliers) only make
  sense if you can picture the backward pass. *(Karpathy — micrograd, Backprop Ninja)*
- **A model can train despite bugs on easy problems.** Karpathy forgot `zero_grad` on camera and
  the toy run still converged. Gradient-check numerically against a reference (`allclose`, max
  diff; ~1e-9 is float noise) and init with small random values, not zeros, which can mask a
  wrong gradient. *(Karpathy)*
- **Check the loss at initialisation**: ≈ ln(K) for K classes (ln 27 ≈ 3.29; ln 65 ≈ 4.17). Much
  higher means confidently wrong; set output bias to 0 and scale output weights down (×0.01)
  to remove the hockey stick. *(Karpathy — makemore 3, GPT)*
- **Keep activations roughly unit-Gaussian at every layer**: weight std = gain/√fan_in (gain 1
  linear, √2 ReLU, 5/3 tanh). Aim for ~5% saturation (|t| > 0.97) and similar stats at every
  depth. Careful init matters less with residuals, normalisation and Adam. *(Karpathy — Part 3)*
- **Watch the update:data ratio** (lr × grad.std / param.std) per weight matrix: log10 ≈ −3.
  Well below means LR too low; well above means too high or a mis-scaled layer. *(Karpathy)*
- **The learning rate is the first knob.** A rising loss means LR too high. Sweep it
  exponentially (`10**linspace(-3, 0)`, one step each; or `lr_find` from ~1e-7) and take the
  valley before instability; cut it 10× at the plateau. AdamW ~3e-4 is a sane default; bigger
  and attention models want lower. *(Karpathy — MLP, GPT; Howard L3/L5; Ng L2)*
- **Own the two ends of the network.** Most "won't train" bugs are inputs (unscaled, long-tailed)
  or the final activation/loss (ReLU on the last layer, missing sigmoid). Match head to labels:
  sigmoid + BCE per class for multi-label, softmax + CE for exclusive classes (add "none"),
  linear/ReLU + L1/L2 for regression. *(Howard L5; Katanforoosh — CS229 L10)*
- **Use fused library primitives** (`F.cross_entropy`) for speed and stability; hand-rolled
  softmax overflows. *(Karpathy — MLP)*
- **Check shapes and broadcasting.** `P / P.sum(1)` without `keepdim=True` silently normalises
  columns; assert row sums. *(Karpathy — makemore; Katanforoosh)*
- **Deep stacks need residuals and normalisation.** If adding depth hurts, add these first.
  Prefer LayerNorm/GroupNorm to BatchNorm when batches are small or inference is one row;
  BatchNorm couples examples and breeds train/eval bugs. *(Karpathy — GPT, Part 3)*
- **Evaluate properly**: average loss over many batches for both splits, `model.eval()` and
  `torch.no_grad()`, and sample/look at outputs; a good loss isn't good output. *(Karpathy)*
- **Make it run → make it fast (profile) → make it right**, in that order. NaN/Inf: log gradient
  norms, try float64 (gone = precision; still there = bad numerics). OOM: lower precision,
  smaller batch plus gradient accumulation. *(Frye)*
- **Confidence is not calibration, and not OOD awareness**: a "dog-cat" went 50/50 → 100% cat
  after longer training. *(Howard L2)* Over-trained nets are overconfident; temperature-scale
  and check reliability (§2.9, reviewer addition).

### 2.5 Tabular data, trees and gradient boosting
- **Feature engineering beats depth on tabular data.** A linear model, NN and deep NN tied on
  Titanic; a notebook using only the Name column beat them all. *(Howard L5)*
- **Feature hygiene**: fill missing with the mode plus a "was missing" flag; `log1p` long-tailed
  money and counts; scale numerics; dummies for codes that are levels; group rare and unseen
  categories into "other". Apply identical preprocessing at inference. *(Howard L3/L5)*
- **Build 1R first.** The best single split was among the best classifiers in a 1990s review.
  Random forests are hard to misuse; Howard has seen far more broken logistic regressions.
  *(Howard L5)*
- **Trees**: use strictly concave split criteria (Gini, entropy), never misclassification. Grow
  then prune; minimum-gain early stopping misses interactions (XOR-like). Trees are bad at
  additive/diagonal structure, so add difference and ratio features. For high-cardinality
  categoricals, sort levels by target rate. *(Townshend — CS229 L9)* (note: compute that
  ordering/encoding inside each training fold or out-of-fold; on rare levels it otherwise
  leaks the label. LightGBM/CatBoost native categorical handling is safer; CatBoost uses
  ordered target statistics.) (note: XGBoost's gamma is a bottom-up prune after growing to
  max_depth, consistent with Townshend; LightGBM's `min_gain_to_split` is a pre-stop, so keep
  it near 0. For single sklearn trees, prune with `ccp_alpha`.)
- **Ensembles**: variance of the mean = ρσ² + (1−ρ)σ²/M. More models only shrink the second
  term; decorrelation (bootstrap, per-split feature subsampling) lowers the floor. Adding bagged
  trees never increases overfitting (it only shrinks the second term); deep trees can still
  overfit, so tune leaf size and max_features. In boosting the opposite holds: more trees
  eventually overfit, so use early stopping. Bagging for high-variance learners, boosting for
  high-bias ones.
  Averaging 5 seed runs helps a lot. *(Townshend; Howard L5)*
- **Gradient boosting** = constant start (mean, or log-odds of base rate), then small trees on
  pseudo-residuals, each scaled by the learning rate. Many small steps beat a few large ones;
  tune learning rate × trees × tree size with early stopping on validation. *(Starmer — GBM)*
- **XGBoost knobs**: eta (step, default 0.3), max_depth (default 6) and gamma (prune if
  gain − gamma < 0), lambda (shrinks leaves built on few rows). With lambda > 0, gamma = 0 still
  prunes negative-gain splits. If leaves hinge on one or two rows, raise lambda first. (note:
  the video uses lambda 0; current xgboost defaults `reg_lambda` to 1. Set values explicitly.
  `tree_method="hist"` is the default since XGBoost 2.0; use `device="cuda"`, not the old
  `gpu_hist`. Pass `early_stopping_rounds` to the constructor; recent versions removed it from
  `fit()`.)
  *(Starmer — XGBoost)*

### 2.6 Testing ML code
- **A test suite is a classifier of commits**: weigh real bugs caught against false alarms
  before adding a test. No coverage gates, minimal opt-in linting, push checks under ~10
  minutes, slow jobs on a schedule. *(Frye)*
- **Minimum ML test set (a day's work)**: loose data expectation tests (nulls, ranges, ordering),
  a memorisation test (overfit a tiny batch, time-boxed by steps or wall time), and loss-based
  regression suites of named hard slices. *(Frye)*

### 2.7 Deployment
- **Deploy early, keep it simple**: prototype UI at a URL → model separated from the app →
  scaling tricks → edge only if forced. *(Tobin; Howard L2)*
- **Batch prediction when inputs are finite and some staleness is fine**; a model service when
  inputs are open-ended. A failed batch job silently serves stale predictions, so add freshness
  and row-count checks. *(Tobin)*
- **Latency ≠ staleness.** Decide freshness per feature: daily batch, cadence, or event-triggered;
  pay for streaming only where staleness hurts. Enforce schemas at ingestion. *(Chip Huyen)*
- **CPU first.** Training needs a GPU, inference usually doesn't (Howard: 54 ms on a laptop;
  Roblox: ~1B BERT requests/day on CPU). Concurrency with tuned thread counts and caching help
  a lot; distillation plus quantisation buy roughly 2–10× at most. If you're 100× off, change
  architecture.
  *(Tobin; Howard)*
- **A prediction = weights + preprocessing code + exact library versions + label vocab.** Ship
  them together; custom functions referenced by a pickle must ship too. *(Tobin; Howard L2/L3)*
- **Safe rollout**: validate before export and fail closed to the previous model, shadow mode,
  1% → 10% → 50% → 100%, instant rollback, fallback to previous model or rules. *(Zinkevich;
  Tobin)*

### 2.8 Monitoring, drift, corruption and tech debt
- **Corrupted data beats slow drift as the main threat.** Upstream jobs fail and features go
  all-null; clients swap CSV/Parquet or reorder columns. Retraining fixes ~80% of complaints,
  so automate it, but gate promotion against historical windows: a fully corrupted hour passes
  a holdout drawn from the same hour. A model is a materialised view: refresh it. *(Shankar)*
- **Telemetry is the ceiling.** Profile per comparable time window: lineage (model/code/data
  versions), schema at every pipeline stage, counts and null rates, input *and* output
  distributions, delayed ground truth, a stored sample, resource use. *(Visnjic/WhyLabs)*
- **Profile 100% of data with mergeable sketches**, not samples; samples misrepresent long
  tails and cause false alarms. Merge per-chunk profiles into daily ones. *(Visnjic)*
- **Baselines**: trailing window catches sudden breaks but absorbs slow drift; training baseline
  catches drift but alerts forever once the world moves. Use both where it matters. Wire severe
  events to actions (retrain; kill switch when most features are null). *(Visnjic)*
- **Tie alerts to business impact.** With thousands of features something is always slightly
  broken; alert fatigue kills monitors. Tune thresholds with the on-call person. The monitor
  must cost less than what it protects. Define an ML SLA (minimum accuracy before users
  complain) and alert against it. *(Shankar; Visnjic)*
- **ML failures are silent**: a bad feature makes the model a bit worse, untraceably. Monitor
  training data and serving features explicitly. *(Zinkevich)*
- **Tech debt is permanent; manage it.** Features only get added, authors leave, names turn
  cryptic. Record code version, data snapshot and config per run, aiming to reproduce metrics,
  not weights. Write team knowledge somewhere durable. Adopt run-layer tools that give ~10× on
  velocity, early validation or versioning. *(Shankar & Biewald)*
- **Offline gains not showing up online? Fix the objective, not the features.** *(Zinkevich)*

### 2.9 Not covered by the sources, but know this *(reviewer addition, not from the lectures)*
- **Target leakage / future features.** Every feature must be computable at prediction time
  from data available before the label existed. Write down each feature's as-of timestamp and
  build training rows with point-in-time joins (DuckDB `ASOF JOIN`, pandas `merge_asof`), never
  by joining current tables. Red flags: one feature dominates importance; validation AUC above
  ~0.98 on a hard problem; a feature filled in by the process that creates the label (a
  "status=merged" column when predicting duplicates, `booking_id IS NOT NULL` when predicting
  bookings). Fit every preprocessing step inside the CV fold (scalers, imputers, target or
  sort-by-target-rate encodings, feature selection, dedup thresholds) with an
  `sklearn.pipeline.Pipeline` or out-of-fold encoding. Remove exact and near duplicates across
  splits, by entity or cluster, before splitting.
- **Class imbalance.** Keep the true base rate in dev and test; never resample them. Report
  PR-AUC (average precision) and precision/recall at the operating threshold, not accuracy or
  ROC-AUC alone (at 1% positives, 99% accuracy is the trivial baseline). Fix in this order:
  (1) choose the threshold on dev from the cost of a false positive vs a false negative;
  (2) class weights (`scale_pos_weight`, `class_weight`) or focal loss; (3) under-sample
  negatives only for speed, then correct the probabilities. SMOTE rarely helps strong GBMs.
  For neural nets, init the output bias to log(p/(1−p)). Reweighting and resampling distort
  probabilities, so recalibrate afterwards.
- **Calibration.** Calibrated means a "0.8" is right 80% of the time in every bin, not just on
  average. Check with a reliability diagram (`CalibrationDisplay`) and Brier score or log loss
  on data the model never saw. Fix with Platt/sigmoid scaling (small data) or isotonic
  regression (more than ~1k positives), fitted on a separate calibration split
  (`CalibratedClassifierCV` with held-out data or `FrozenEstimator`); temperature scaling for
  neural nets. Recalibrate after every retrain and any base-rate shift. Thresholds, cluster
  merges and "auto-accept above p" rules all assume calibrated scores.
- **Time-series and temporal splits.** If rows are time-ordered or the model predicts the
  future: train on the past and validate on the next window; use rolling or expanding-origin CV
  (`TimeSeriesSplit`) with a gap/embargo at least as long as the label horizon plus the longest
  feature lookback; never shuffle; compute lag and rolling features only from data before the
  cut-off; when entities repeat, combine the time split with a group split. Beat a naive "last
  value" or "same period last year" baseline, and report the metric per window to show
  stability.

---

## 3. Checklists

**Debugging a model that won't learn** (cheapest first)
- [ ] Data: look at a batch as the model sees it; labels aligned with inputs; no broken files.
- [ ] Inputs scaled; long tails logged; categoricals encoded; train and inference preprocessing identical.
- [ ] Head and loss match labels (sigmoid/BCE, softmax/CE, linear/L2); no ReLU on the output.
- [ ] Initial loss ≈ ln(K) for balanced classes, or base-rate entropy −[p ln p + (1−p) ln(1−p)]
      for imbalanced binary (init output bias = log(p/(1−p))); fix output bias/scale if far higher.
- [ ] Overfit one tiny batch to near-zero loss (floor = duplicate inputs with different labels).
- [ ] Training loop: `zero_grad` each step, eval/no_grad in eval; update sign `p -= lr * grad`
      (hand-written loops only).
- [ ] LR: rising loss = too high; run an exponential sweep; check update:data ≈ 1e-3.
- [ ] Activations/gradients per layer: saturation ~5%, no dead columns, no vanishing/exploding norms.
- [ ] Shapes and broadcasting asserted (`keepdim=True`, row sums = 1).
- [ ] Custom ops gradient-checked against autograd or numerical gradients.
- [ ] NaN/Inf: log grad norms, retry in float64; clip gradients, never the loss.
- [ ] Depth hurts: add residuals and LayerNorm before anything else.
- [ ] Still stuck: five error numbers; is the bottleneck data, a component, or the objective?

**Before shipping a model**
- [ ] Metric and objective written down; trivial baseline and heuristic beaten by more than seed noise.
- [ ] Dev/test drawn from the deploy distribution with group/time/region splits; test used rarely.
- [ ] Every feature available at prediction time (as-of joins); preprocessing fitted inside folds. *(§2.9)*
- [ ] Metric suits the base rate (PR-AUC, precision@threshold); threshold set from error costs on dev. *(§2.9)*
- [ ] Calibration checked (reliability plot, Brier) and redone after each retrain. *(§2.9)*
- [ ] Per-slice metrics on named hard suites, not just the aggregate.
- [ ] Error, ceiling and ablation analysis done; expensive features that don't pay removed.
- [ ] Data expectation tests, memorisation test and regression suites run in CI.
- [ ] Weights + preprocessing + library versions + label vocab packaged and versioned together.
- [ ] Accuracy and latency benchmarked on the target runtime; size/latency tests in CI.
- [ ] Features logged at serving time; unseen categories and new nulls handled.
- [ ] Export gate vs production on a fixed check; fail closed to the previous model.
- [ ] Shadow mode, gradual rollout, instant rollback, rule/previous-model fallback.
- [ ] Monitoring: per-window profiles of inputs, outputs, labels; freshness/row-count checks for batch jobs.
- [ ] Retrain schedule plus promotion gate against other time windows.
- [ ] Run lineage recorded (code commit, data snapshot, config).

---

## 4. Failure modes to hunt for in reviews

- Random splits on correlated rows (same region, same entity, time series); dev/test not from the deploy distribution.
- Tuning on the test set, or changing it midway.
- Claiming wins smaller than seed-to-seed variance.
- Cleaning data by hand before any model has ranked it by loss.
- Jumping to a fancy architecture before data work and a baseline.
- Collecting more data for a bias problem; growing the model for a mismatch problem.
- Missing `zero_grad`, wrong update sign, `grad =` instead of `+=`, softmax over multi-hot labels.
- Clipping the loss instead of the gradient; BatchNorm left in train mode at eval.
- Preprocessing copied by hand between notebook and server; unseen categories silently dropped.
- Logging IDs and re-joining features offline; copied pipelines carrying stale filters.
- Batch jobs with no freshness check; retrained models promoted without a gate.
- Monitors on sampled data or on the model only; alerts on every tiny change.
- Misclassification error as a tree split criterion; minimum-gain early stopping.
- *(§2.9, reviewer additions)* Features computed after the label time; target encoding or
  scaling fitted on the full dataset.
- Accuracy or ROC-AUC reported on a 1%-positive problem; dev/test resampled to balance.
- Raw scores used as probabilities (thresholds, auto-merge rules) without a calibration check.
- Shuffled CV on time-ordered data; no gap between train and validation windows.

---

## 5. Where the teachers disagree, and how to resolve it

- **Clean data then train, or train then clean?** Howard trains a rough model first and uses its
  loss ranking to find bad labels. Resolve: do cheap expectation checks (Frye) first, then let
  a quick model rank the rest.
- **First model: logistic regression, 1R, or GBM?** Zinkevich and Ng want interpretable
  logistic regression; Howard wants 1R and finds random forests hardest to misuse; Starmer's
  notes position GBMs as the strong tabular default. Resolve: 1R and logistic regression as
  baselines in an hour, then a GBM with early stopping, and keep the baseline in every report.
- **Random split or structured split?** fastai defaults to a random 20%. Ng (IID is often false)
  and Shankar (validate across time windows) say otherwise. Resolve: random only when rows are
  truly independent; otherwise split by group, region or time.
- **Careful init or not?** Karpathy shows init matters, then says residuals, normalisation and
  Adam make it matter less and he just scales by 1/√fan_in. Resolve: use framework defaults,
  but always check the initial loss and activation stats when training is odd.
- **BatchNorm?** Widely used (ResNet), but Karpathy prefers LayerNorm/GroupNorm after repeated
  bugs. Resolve: LayerNorm for small batches, one-row inference and tabular MLPs.
- **Fixed seeds?** Karpathy seeds generators for reproducibility; Howard runs unseeded to see
  variance; Zinkevich measures the distribution across seeds. Resolve: fix seeds while
  debugging, vary them to measure the noise floor before claiming improvements.
- **Bias/variance trade-off?** CS229 L12 frames classic fixes (fewer features vs more features);
  Ng's deep learning talk says bigger models and more data break the trade-off. Resolve: with
  neural nets and enough data, scale; with small tabular data, the classic trade-off holds.
- **Drift or corruption?** WhyLabs builds drift detection (Hellinger distance, baselines);
  Shankar says corruption dominates and retraining fixes most drift. Resolve: monitor schema,
  nulls, counts and row-level breaks first; add distribution drift with both baselines next.
- **Batch or real time?** Tobin calls batch prediction simple and proven; Huyen pushes streaming
  features where freshness matters. Resolve per feature: how stale can it be before the
  business notices (Shankar's question)?

---

## 6. Applying this to Sam's work

- **Place matching/dedup is pairwise classification.** Pipeline: normalise → block candidates
  (geohash/country, name n-grams or embeddings) → score pairs → threshold/cluster. Baseline
  logistic regression, then LightGBM/XGBoost on pair features (name similarity, haversine
  distance, admin-code and feature-class agreement, population ratio, plus difference and ratio
  features). Run ceiling analysis: blocking recall is often the real ceiling, not the scorer.
- **Labels and splits.** Hand-label a few hundred pairs *from the target supplier feed* for
  dev/test; keep weak/auto labels as train with a train-dev slice; have two people label ~200
  pairs for a human-level number. Split by country or region, and dedupe exact/near-duplicate
  entities across splits before splitting (border towns and multi-country suppliers cross
  regions). *Reviewer additions:* split by connected component/cluster of the match graph, not
  just by pair, or A–B in train and B–C in test leak through transitivity. Positives are rare
  after blocking (often <1%): report precision/recall/F1 at the chosen threshold plus PR-AUC,
  and set the threshold from the cost of a false merge (usually worse) vs a missed duplicate.
  Calibrate the scorer before clustering on thresholds.
- **Find dirty rows with the model.** Sort pairs and places by loss; confidently wrong ones are
  usually bad feature codes, wrong supplier mappings or duplicates. Review top-k, not 6M rows.
  Track named slices (transliterations, same name in different admin regions, airport vs city).
  Character n-gram NLL per name is a cheap junk/garbled-name detector and a matching feature.
  *(Karpathy — makemore)*
- **16 GB constraints**: mini-batch/`partial_fit`, histogram GBMs (XGBoost's default `hist`,
  LightGBM), float32, `max_bin` lowered if memory is tight, batch prediction written to
  Parquet/DB with row-count and freshness checks, CPU inference, chunk-level profiles merged
  into a daily profile (DuckDB `approx_*`, Apache DataSketches, or whylogs/Evidently; WhyLabs
  wound down in 2025, so check a library is still maintained before adopting it).
- **Supplier feeds are "a CSV that keeps changing"**: schema, column order, null-rate and
  lat/lon range checks at ingestion; compare each refresh's match rate and class distribution
  to the last good snapshot before promoting.
- **LLM enrichment**: never run an LLM over a cross product; block and filter first. *(Shankar)*

---

## 7. Where to look in `references/`

| Need | File (see INDEX.md) |
|---|---|
| Backprop from scratch (micrograd), CS229 intro + linear regression/GD, Zinkevich Rules of ML, XGBoost regression | `ml-1-karpathy-ng-zinkevich-starmer.md` |
| Init, activations, BatchNorm, update:data ratio; FSDL deployment; model-assisted cleaning + export; FSDL troubleshooting & testing; gradient boosting | `ml-2-karpathy-tobin-howard-frye-starmer.md` |
| GPT from scratch, Howard L3 (ReLUs, tabular from scratch), logistic regression/LWR/Newton, Chip Huyen real-time ML | `ml-3-karpathy-howard-ng-huyen.md` |
| Backprop Ninja, Ng debugging/error/ceiling/ablation analysis, CS336 efficiency + tokenization, Shankar operationalising ML | `ml-4-karpathy-ng-liang-shankar.md` |
| makemore bigram/NLL, Ng Nuts and Bolts (five error numbers, mismatch), Howard L1 (transfer learning, DataBlock), CS229 neural nets | `ml-5-karpathy-ng-howard-cs229.md` |
| WhyLabs monitoring, Howard L5 (tabular, 1R, ensembles), makemore MLP (LR sweep, splits), decision trees + ensembles, Shankar AI-SQL inference | `ml-6-visnjic-howard-karpathy-townshend-shankar.md` |
