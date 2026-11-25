# ml-4 — cross-cutting principles

1. **Abstractions leak; understand one level down.** Autograd, prompting and MLOps tools all hide mechanisms that you need when they fail. Build or trace the layer below once so you can debug it. (Karpathy, Backprop Ninja; Liang, CS336 L1)
2. **Debug ML like software: hypothesise, then run a cheap diagnostic before spending on a fix.** Brainstorm all options, then pick using evidence rather than gut feel or the loudest voice. (Ng, CS229 L12)
3. **Bias vs variance via learning curves is the first diagnostic.** Big train/dev gap means variance (more data, regularise, fewer features); training error above target with a small gap means bias (better features, bigger model). More data cannot fix bias. (Ng, CS229 L12)
4. **Separate "optimiser failed" from "wrong objective".** Evaluate your surrogate loss J on a competitor's parameters: if theirs scores higher on J, your optimiser is the problem; if yours wins on J but loses on the business metric, J is wrong. (Ng, CS229 L12)
5. **Ceiling analysis for pipelines.** Substitute ground truth for each stage on a dev set and measure the end-to-end gain; spend effort where the ceiling is highest. Ablation (remove components) explains what your gains came from and which costly parts can go. (Ng, CS229 L12)
6. **Start with a quick-and-dirty baseline and always compare new methods against it.** The biggest research-to-production bottleneck is knowing whether something new beats a simple off-the-shelf baseline by enough. (Ng, CS229 L12; Shankar)
7. **Verify gradients and implementations against a reference.** Compare exact equality, allclose and max diff to autograd or numerical gradients; initialise with small random values rather than zeros so simple values don't hide bugs. Unit-test each component locally before using expensive compute. (Karpathy; CS336 L1)
8. **Shapes are a correctness tool.** A gradient has its tensor's shape; matmul backward is a matmul whose transposes are fixed by the shapes; a sum in the forward pass becomes a broadcast in the backward pass, and the other way round. (Karpathy)
9. **Cross-entropy gradient = softmax minus one-hot.** It pushes in proportion to how wrong the prediction is, so confidently wrong (often mislabelled) examples dominate training. Clipping the loss zeroes outliers' gradients, which is a bug; clip gradients instead. (Karpathy)
10. **Avoid train/test mismatches.** Examples are batchnorm's biased variance in training vs unbiased variance at inference, and validating a retrained model on the same corrupted batch it trained on. Use the same computation in both phases and validate across time windows. (Karpathy; Shankar)
11. **Efficiency x resources = results; profile before optimising.** Know where FLOPs and memory go. Most work is memory-bandwidth-bound, so minimise data movement: fuse passes and read each chunk once. Efficiency matters more as scale grows. (Liang, CS336 L1)
12. **Predictability is as important as optimality.** Fit small-scale curves (scaling laws / learning curves) to predict large-run outcomes before committing budget, and choose setups whose hyperparameters transfer across scale. (Liang, CS336 L1; Ng's learning curves)
13. **Data is curated, not given.** Transform, filter, dedupe and mix. Filtering is an efficiency lever because compute spent on bad or redundant data is compute not spent on good data. Keep private eval sets to avoid contamination, and keep internal dev metrics separate from external business metrics. (Liang, CS336 L1)
14. **Training stability is a first-class concern.** Keep activation, parameter and gradient norms in a goldilocks zone. Saturation, dead units and exploding gradients come from not watching what flows backwards. (Karpathy; CS336 L1)
15. **In production, corrupted data beats drift as the main threat.** Nulls from failed upstream jobs, changed file formats and reordered columns are common. Frequent automated retraining handles staleness (models are views to refresh) and fixes ~80% of complaints, but you must gate promotion of retrained models. (Shankar)
16. **Monitor tied to business impact, or drown in alert fatigue.** With thousands of features, something is always slightly broken. Detect when it actually matters. (Shankar)
17. **Velocity, validate early, version.** Judge every tool and process by whether it improves one of these by ~10x. Run-layer tools that one engineer can adopt win over pipeline or infra migrations. (Shankar)
18. **Reproducibility is a spectrum.** Record code version, data snapshot and config per run, and aim to reproduce metrics rather than exact weights. (Shankar & Biewald)
19. **ML pipelines are data pipelines.** Data-engineering skills (SLAs, scheduling, backfills, provenance) matter as much as modelling. Single-node DBMS engines (DuckDB) may beat MapReduce-style tools for many workloads. (Shankar)
20. **For applications: prompt, then fine-tune, then pre-train only as a last resort.** (Liang, CS336 L1)

## Andrej Karpathy — Building makemore Part 4: Becoming a Backprop Ninja (https://www.youtube.com/watch?v=q8SA3rM6ckI)
**Backprop is a leaky abstraction: if you cannot write the backward pass of your network at the tensor level, you cannot reliably debug it.**

**Mental models / first principles**
- Autograd does not make arbitrary stacks of differentiable blocks "just work". Saturated activations (flat tails of tanh/sigmoid) kill gradients, dead neurons, exploding/vanishing gradients in RNNs, and subtle bugs all come from not understanding what flows backwards.
- Example bug from the wild: someone clipped the *loss* at a max value when they meant to clip *gradients*. Clipping the loss of an outlier sets its gradient to zero, so outliers are silently ignored by training. Only visible if you think about the backward pass.
- Until ~2012-2014 everyone wrote backward passes by hand (Matlab, then numpy) and checked them with a numerical gradient checker. Autograd is now standard, but that habit built real intuition.
- Training as a "pulley system": the loss gradient tugs on logits; that tension propagates through the graph to weights, which slowly give in to the tug each update.

**Mechanisms (the reusable rules)**
- Gradient tensor always has the same shape as the tensor it belongs to (dlogprobs is 32x27 because logprobs is 32x27).
- Chain rule per node: local derivative times upstream gradient. Element-wise ops are scalar calculus: log -> 1/x, exp -> exp(x) (reuse the forward output), x^-1 -> -x^-2, x^-0.5 -> -0.5 x^-1.5, x^2 -> 2x, tanh -> 1 - out^2 (written in terms of the *output*).
- Indexing/plucking (e.g. picking the correct-class logprob): gradient is zero everywhere except the plucked positions; mean over n gives -1/n there.
- **Duality of sum and broadcast**: a sum along a dimension in the forward pass becomes a replication/broadcast in the backward pass along that dimension; a broadcast (variable reuse) in the forward pass becomes a sum in the backward pass. Use keepdim=True so shapes match (32x1 stays 32x1).
- Any node used in multiple branches accumulates gradients (+=). Your check will fail until every branch is done; that is expected, and confirms the checker detects errors.
- Addition is a gradient router: it copies the upstream gradient to all inputs. Max is a router to the argmax position (one_hot(argmax) * upstream).
- **Matrix multiply backward is a matrix multiply**: for D = A@B + c: dA = dD @ B^T, dB = A^T @ dD, dc = dD.sum(0). He does not memorise these; he derives them by **shape matching** (only one arrangement of transposes produces the right shape). If unsure, write out a 2x2 example by hand and generalise.
- view/reshape backward is just view back to the original shape. Embedding lookup backward: scatter-add each row's gradient into the embedding table row it came from (a row used many times accumulates).
- Max-subtraction for softmax stability: its gradient should be ~0 (1e-9) because shifting logits doesn't change probs; seeing that confirms correctness.
- **Fused cross-entropy gradient**: dlogits = (softmax(logits) - one_hot(y)) / n. Each row sums to zero: push down on wrong classes in proportion to their probability, pull up the correct one equally. Force is proportional to how wrong the prediction is; a perfect prediction gives zero gradient. Confidently wrong predictions get the strongest pull.
- Low-probability correct answers get boosted gradients through the 1/p term of log.
- Fused batchnorm backward derived on paper: count fan-out arrows (mu feeds 32 x-hats plus sigma^2; each x_i feeds mu, sigma^2, x-hat_i), sum all contributions; the dsigma^2/dmu term vanishes because mu is the batch mean. Result is one line using only dL/dy and x-hat, vectorised across all 64 neurons.
- Analytic fused backward passes (cross-entropy, batchnorm) are much shorter and faster than backpropagating through atomic pieces, because terms cancel.

**Decisions & trade-offs**
- Breaking the forward pass into many named intermediates makes debugging easy but is slow; fused ops (F.cross_entropy) are faster in both passes and more numerically stable. Use atomic decomposition to learn/debug, fused ops in production.
- Bessel's correction: batchnorm paper uses biased variance (1/n) in training and unbiased (1/(n-1)) for running stats at inference, i.e. a train/test mismatch. Karpathy treats this as a bug and uses 1/(n-1) consistently; it matters less with large batches.
- Bias before batchnorm is redundant (batchnorm removes the mean); he kept it only to check gradients.
- Once you supply gradients manually, wrap the training loop in torch.no_grad() so PyTorch doesn't build a graph.

**Rules of thumb / checklist**
- Build a gradient-comparison helper: exact equality, torch.allclose, and max abs difference vs a reference (PyTorch autograd or numerical gradient). Differences around 1e-9 are float noise.
- Initialise biases to small random numbers, not exactly zero, when testing gradients: zeros simplify expressions and can **mask a wrong gradient**.
- Always scrutinise shapes first, then reason about a tiny toy example, then write the general vectorised line.
- Run a short training loop (e.g. 100 steps), compare your gradients to autograd, then remove the check.

**Failure modes & gotchas**
- Forgetting the sum over broadcast dimensions, or forgetting keepdim, silently produces wrong shapes or wrong gradients.
- Forgetting a second branch (missing +=).
- Framework docs can be wrong (torch.var / BatchNorm1d documentation on biased vs unbiased estimators); verify behaviour.
- Odd framework backward implementations exist (he used x**-1 instead of 1/x because of an issue in division backward at the time). (note: likely fixed in current PyTorch.)

**Relevance to Sam**
- When a model trains badly, inspect gradient magnitudes per layer: near-zero means saturation or dead units; huge means instability. Gradient clipping should clip gradients, not losses.
- The cross-entropy "force proportional to error" intuition explains why noisy/mislabelled rows (common in scraped location/category data) dominate training: confidently wrong labels produce the strongest gradients. Clean labels or use robust losses rather than clipping loss.
- Writing small manual gradient checks is cheap on a laptop and catches custom-loss bugs (e.g. in ranking or matching losses) before burning hours of training.

## Andrew Ng (Stanford CS229) — Lecture 12: Debugging ML Models and Error Analysis (https://www.youtube.com/watch?v=ORrStCArmP4)
**ML development is a debugging workflow: run diagnostics that tell you which lever to pull, instead of picking fixes by gut feel.**

**Mental models / first principles**
- A learning algorithm almost never works the first time; most of the job is deciding *what to try next*. That decision dominates team efficiency (days of scraping data vs. tuning vs. switching models).
- Typical failing teams pick one fix at random (whatever they read last night, or whatever the loudest person wants). Merely brainstorming the full list of options and evaluating them puts you ahead.
- Turn "black art" into engineering: form hypotheses about what is broken and test them cheaply *before* investing in a fix, like binary-searching a crashing program to isolate the faulty component.
- This advice is for building things that work; research goals differ.

**Diagnostic 1: bias vs. variance (his single most-used tool)**
- Plot learning curves: x-axis = number of training examples, y = error; draw training error, dev error, and the desired performance line (often ~human-level or a business target). Use a dev/hold-out set, not the test set, for this.
- Training error *rises* with training set size (tiny sets are trivially fit); dev error falls, theoretically at roughly 1/sqrt(m) toward Bayes error (irreducible error from ambiguous data, e.g. blurry X-rays), assuming IID train/dev/test.
- **High variance**: large gap between training and dev error (strong signal); dev error still falling as m grows (weaker signal, hard to extrapolate). Fixes: more data, fewer features, more regularisation.
- **High bias**: training error itself is above the target, and the train/dev gap is small. More data will never get you there. Fixes: more/better features (e.g. email header/routing features for spam), bigger model, less regularisation.
- Workflow for a new problem: build a quick-and-dirty baseline (e.g. logistic regression), run bias/variance, then decide. Exception: domains where you or the literature already know data needs; then start more sophisticated.

**Diagnostic 2: optimisation algorithm vs. optimisation objective**
- You optimise a surrogate J(theta) (log-likelihood minus regulariser) but care about A(theta) (e.g. weighted accuracy where false-positive on non-spam costs far more). A is non-differentiable/NP-hard to optimise directly.
- Example: SVM beats regularised logistic regression on A, but you want to ship LR (cheaper, easier to update). Compute J on both parameter sets:
  - J(theta_SVM) > J(theta_LR): LR failed to maximise its own objective -> **optimiser problem** (run longer, Newton's method, fix the code).
  - J(theta_SVM) <= J(theta_LR): LR maximised J better yet loses on A -> **objective problem** (J doesn't track what you care about; change lambda weighting, cost weights, or switch objective e.g. SVM).
- "Train longer" almost never hurts if regularised, but this test tells you whether burning another 48 GPU-hours is worth it.
- Changing lambda is usually a bias/variance lever, though it technically changes the objective.

**Diagnostic 3: multi-component system (helicopter RL example)**
- Pipeline: simulator -> cost function (squared error to target position) -> RL algorithm -> controller. Controller flies worse than a human. Each of simulator / RL / cost function could be a PhD-thesis-sized project; which to work on?
- Logic: if the simulator were accurate, the optimiser minimised J, and J encoded "flies well", the result would fly well. So at least one assumption is false; find it.
  - Flies well in sim but badly in real life -> simulator mismatch (sim-to-real gap).
  - Compare J(theta_human) vs J(theta_RL): human gets lower cost -> RL algorithm not optimising well. Human gets higher cost but flies better -> cost function is wrong.
- The bottleneck moves: after fixing the simulator for a few weeks, rerun diagnostics; the next problem may be the optimiser, then the cost function. Use diagnostics continuously to drive prioritisation.
- Specifying a good cost function can be genuinely hard (what is a "good" banked turn?); Adam Coates' award-winning paper came from diagnostics proving this was the real bottleneck.
- Adding noise to simulators makes policies robust to reality; if sim performance is worse than real, you may have added too much noise.

**Error analysis for pipelines (ceiling analysis)**
- Face-recognition pipeline: background removal -> face detection -> eye/nose/mouth segmentation -> logistic regression. Overall 85%.
- On the dev set, replace each component's output with ground truth, one stage at a time (cumulatively), and measure end-to-end accuracy: perfect background removal 85.1% (+0.1), perfect face detection 91% (+~6), perfect eye segmentation +4, etc. Largest jump = most promising component to improve. Final stage perfect = 100%.
- Cautionary tale: someone wrote a whole PhD on background removal for a system where it barely moved the needle.
- Cumulative vs one-at-a-time can differ slightly; if worried, do both. Combine with judgment on how hard each improvement is. Results guide team allocation (parallel workstreams).
- If perfecting a component makes things *worse*, don't ignore it; investigate, like a weird UI glitch in software.

**Ablative analysis**
- Explains the gap between your system and a much simpler baseline: remove components one at a time and see how much accuracy drops. Spam example: plain LR 94% vs 99% with spelling correction, sender host, header, text parser, JavaScript parser, fetched-link features; removing the text parser dropped ~4%, the biggest contributor.
- Uses: insight for reports/papers, deciding what to double down on, and dropping expensive features (e.g. costly sender-host lookups) that contribute little. Order of removal is arbitrary; leave-one-out avoids ordering effects.

**Other points**
- Small data can still give great results (his teams get good results with ~100 examples) but needs more insightful pipeline design; big data is somewhat over-hyped. End-to-end monolithic models need huge data.
- More data almost never hurts, but the question is *how much* it helps; diagnose before spending days collecting.

**Relevance to Sam**
- Location matching pipeline (normalise -> block/candidate generation -> pairwise scoring -> threshold/cluster): do ceiling analysis by substituting hand-labelled perfect candidates or perfect normalisation on a labelled sample to see which stage limits final precision/recall. Often blocking recall is the ceiling, not the scorer.
- Use the J vs A test when a cheaper model (e.g. LightGBM or logistic regression) trails a heavier one on your business metric (e.g. weighted cost of false merges vs missed duplicates).
- Learning curves on subsamples are cheap on a 16GB laptop and tell you whether labelling more pairs/places is worth it.
- Ablation tells you which costly features (geocoding calls, external lookups) can be dropped from production.

## Percy Liang & Tatsu Hashimoto (Stanford CS336, Spring 2026) — Lecture 1: Overview, Tokenization (https://www.youtube.com/watch?v=JuoVZkPBiKk)
**Building the best model is an efficiency problem: accuracy is roughly efficiency x resources, so profile everything, scale by recipe, and treat data and tokenization as efficiency levers.**

**Mental models / first principles**
- Abstractions are leaky: prompting a model is fine until you hit something it can't do with no recourse. Real understanding (and fundamental research) requires knowing the whole stack, learned by building.
- Small-scale experiments may not represent the frontier: FLOP share in MLP layers goes from ~44% at small scale to ~80% at 175B; capabilities like in-context learning emerge only past a critical scale. So optimisations that matter small may not matter large.
- Three kinds of knowledge: **mechanics** (how transformers, parallelism, kernels work; transfers), **mindset** (squeeze hardware, take scaling seriously, profile and benchmark; transfers), **intuitions** (which data/model choices work; often do *not* transfer across scale and come only from experiments; e.g. SwiGLU paper attributing success to "divine benevolence").
- Bitter lesson, correctly read: not "scale is all that matters" but "algorithms that scale are what matter". Efficiency matters *more* at scale: a 5% gain on a $100M run is huge. OpenAI found ~44x algorithmic efficiency gain on ImageNet 2012-2019, multiplying with hardware gains.
- Framing question: what is the best model buildable with a given data and compute budget? Pretraining is usually compute-bound; if you hoard GPUs you may become data-bound and the calculus changes.

**Mechanisms**
- Training pipeline pieces: tokenizer -> architecture -> loss/optimiser -> training loop. Modern transformer refinements: activation functions, positional encodings, normalisation placement, cheaper attention (attention is O(n^2) in sequence length), state-space / linear attention (Mamba, Gated DeltaNet) often in hybrids with attention, mixture-of-experts replacing dense MLPs.
- Training choices: next-token loss (multi-token prediction helps), AdamW vs increasingly Muon (e.g. Kimi K2), initialisation, LR schedule, regularisation, batch size. These look like "just hyperparameters" but principled choices separate a run that blows up from state of the art.
- Core balance: expressiveness vs **stability** (keep parameter and gradient norms in a goldilocks zone, neither exploding nor vanishing) vs efficiency.
- Resource accounting: training FLOPs ~ 6 x params x tokens. Memory is not where compute is; moving parameters/activations between HBM and compute is usually the bottleneck. B200: ~2.25 PFLOP/s bf16, ~8 TB/s memory. Roofline analysis tells you whether you're compute- or memory-bound (usually memory).
- Kernels: every PyTorch op launches a kernel. Principle: organise compute to **minimise data movement**. Fusion = read once, do several ops, write once. Tiling extends this. Across GPUs data movement is even pricier; think in collectives (gather, reduce, all-reduce) and shard data / model / layers / sequence / experts with different trade-offs.
- Inference: prefill (like training, parallel over prompt, builds KV cache) and decode (one token at a time, memory-bound). Speedups: pruning, quantisation, distillation, speculative decoding (cheap draft model guesses tokens, big model verifies in parallel), inference kernels, dynamic batching of irregularly arriving requests. Inference also powers RL rollouts, synthetic data and evals.
- Scaling: think of a **scaling recipe** (FLOP budget -> config) not a single model. Run small experiments, fit a scaling law, predict loss at target scale before spending. Scaling laws are not natural laws; you engineer them via parameterisations enabling **hyperparameter transfer** (small-scale hparams equal or predictably map to large-scale). **Predictability is as important as optimality.**
- Chinchilla rule of thumb: ~20 tokens per parameter (70B -> ~1.4T tokens) for compute-optimal training; but inference cost pushes people to train smaller models on far more tokens. Marin project pre-registers predicted losses before big runs.
- Evaluation: internal metrics (e.g. perplexity) need smoothness across scale and relative comparisons; external metrics need ecological validity. Don't conflate them. Prefer eval data not on the internet (contamination). Use many diverse evals; a single averaged number hides differences.
- Data: doesn't fall from the sky; actively curate. Steps: transformation (HTML/PDF/code repos -> text), filtering, deduplication, mixing sources, synthetic rewriting. Legal issues (licences, unlicensed GitHub code). Stages: pretraining, mid-training (high-quality and long-context data at end of pretraining), post-training (conversations, agent traces). Filtering is an efficiency lever: compute spent on bad/redundant data is compute not spent on good data.
- Alignment: generate responses, score (human, verifier, LLM judge), update toward better ones (PPO, GRPO; DPO for preferences). Critiquing is easier than generating. RL is unstable and hard to tune; stay supervised as long as possible. RL at scale is an orchestration problem (inference server + trainer, off-policy lag vs throughput).

**Tokenization (detail)**
- Tokenizer = reversible map between strings (bytes) and integer sequences; must round-trip exactly. Compression ratio = bytes per token (e.g. 20 bytes / 8 tokens = 2.5). Higher ratio -> shorter sequences (good for quadratic attention), but bigger vocab -> sparsity; modern multilingual vocabs are 100k-200k.
- Character-level: ~150k Unicode chars, many rare, poor compression. Byte-level: vocab 256, compression 1, sequences too long. Word-level: meaningful units but unbounded vocab and UNK tokens at test time that break perplexity.
- **BPE**: start from bytes, repeatedly merge the most frequent adjacent pair into a new token (256, 257, ...), rewriting the sequence. Common strings become single tokens; rare strings split into smaller pieces, never UNK. Encoding new text = replay merges. Naive implementation loops over all merges (slow); index only relevant merges, pre-chunk text (regex) and tokenize chunks, handle special tokens, consider Rust/C.
- Quirks: " hello" and "hello" are unrelated IDs; digit grouping varies by tokenizer (per-digit is cleaner but more tokens).
- Any replacement for tokenizers must still provide abstraction over low-signal units and **adaptive computation** (variable-size chunks).

**Decisions & trade-offs / rules of thumb**
- For an application: prompt a model first, then fine-tune, pre-train only as a last resort.
- Profile and benchmark everything; know where FLOPs and memory go.
- Unit tests for each component give dense feedback; develop and verify correctness locally on a laptop, then use a cluster only for real runs/benchmarks.
- Use AI as a tutor with a pedagogical prompt, not to generate the thing you're meant to learn.

**Relevance to Sam**
- On a 16GB laptop the same mindset applies: most pipeline steps are memory-bandwidth bound; fuse passes (read each chunk once, apply all transforms), and measure before optimising.
- Data curation steps (transform, filter, dedupe, mix) mirror his location-data work; dedup and filtering are efficiency levers, not just hygiene.
- BPE/subword ideas matter for place-name matching: character n-gram or subword tokenisation handles rare/misspelled names without UNKs; be aware of leading-space and casing quirks if using LLM tokenizers for embeddings.
- Separate internal dev metrics (smooth, relative) from business-facing metrics; keep a private, non-public eval set to avoid contamination when testing LLM-based classifiers.
- Fit small "scaling" curves (data size vs metric) before committing to a big labelling or training job.

## Shreya Shankar (with Lukas Biewald, Gradient Dissent / CoreWeave) — Operationalizing Machine Learning: Interview with Shreya Shankar (https://www.youtube.com/watch?v=LdMydLBDgEQ)
**Production ML is mostly a data-management and process problem; what matters is velocity, validating as early as possible, and versioning, and the dominant real-world failure is corrupted data, not slow drift.**

**Mental models / first principles**
- "Production" = a model serving outputs people use regularly, where someone gets alerted if it breaks. That is a different game from research: on-call, reliability processes, tooling, team knowledge.
- Her interview study (~20 ML engineers across company sizes, self-driving to banking) found four stages: data collection (often another team), experimentation, evaluation & deployment, monitoring & response. Monitoring and evaluation/deployment were roughly equal pain points and blur together (e.g. monitoring staged deployments).
- **Three Vs** that govern success and judge tools: **Velocity** (iterate fast), **Validate early** (catch bad models before production, not at final A/B test), **Versioning** (manage many model versions, per-customer/per-population variants and edge cases over time).
- **A model is like a materialised view on a table**: as data changes it goes stale; the database answer is "refresh the view". Big companies retrain 6-8 times a day or hourly, so distribution shift per se is not their problem. Simply retraining fixes complaints about ~80% of the time, and busy engineers rationally choose that.
- With frequent retraining, the real problem becomes **training on corrupted data** and deciding when to **block a retrained model from promotion**.
- Many MLOps problems are recastable as classic data management problems (provenance, staleness, quality); ML pipelines are data pipelines.

**Mechanisms: how data corruption happens and why monitoring is hard**
- Features only get added, never removed: models accumulate hundreds to tens of thousands of features. Original authors leave; models stay.
- Schema-style validation (bounds, types, allowed values per feature) is correct in principle, but nobody maintains it as pipelines evolve.
- With that many columns, the probability that some record in some column is corrupted is near 1 -> **alert fatigue**. The real question is when corruption is bad enough to hurt the business, and detecting that precisely.
- Concrete causes she lived through: upstream jobs (e.g. pulling from Snowflake) fail for lack of spot capacity and features become all-null; clients send daily files that switch CSV/Parquet or reorder columns, corrupting a subset of rows/columns. Cryptic feature names (four-letter codes) make tracing regressions a nightmare.
- **Retraining trap**: if you retrain hourly on the last hour and validate on a held-out slice of that same hour, a fully corrupted hour passes validation (same distribution). Then someone fixes the upstream bug and production performance shifts. Validation must compare against other snapshots/time windows, at the timescale engineers can react.

**Decisions & trade-offs**
- Jupyter notebooks are bimodal: loved for velocity and storytelling, hated for validation/reproducibility. Conflicting opinions reflect conflicting priorities (initial velocity vs validation), which shift as a company matures. Stories: a quarter spent moving model evaluation out of forked notebooks into a standard system; someone spent 3-6 months reproducing notebooks to argue against them.
- Senior engineers accept tech debt and turnover as permanent; the goal is to keep shipping and keep old things running despite it, not to eliminate debt.
- Reproducibility is a spectrum, not binary. Exact reproduction is often impossible (GPU, infra, upstream data change); aim for similar *metrics*, not identical parameters. Biewald: most teams start at zero (can't say what code trained a 6-month-old model); every step along the axis (code version, data snapshot, config) pays off for debugging and governance, but perfect reproducibility is very expensive.
- Feature stores were almost never mentioned; mid-size companies get by with a Postgres feature table refreshed by a daily cron. Ask: how stale can features be before the business notices? Don't compute on demand unless needed.
- Tool layers (most to least frequently changed): **run layer** (per experiment), **pipeline layer** (components: feature generation, split, train; orchestrators like Airflow/Kubeflow), **infrastructure layer** (compute, on-prem vs cloud GPUs, Kubernetes). Changing lower layers is a huge organisational effort; tools that drop into the run layer (one engineer can adopt, e.g. experiment tracking) get adopted most. A tool must give a ~10x improvement on one of the Vs to be worth it.
- Starter stack advice for a resource-poor startup: cloud account with compute, a query engine for big data (Spark/EMR, though she prefers DBMS-style engines; notes the DuckDB trend), a DAG scheduler (Airflow: works, has backfill UI), then a good ML/data lead.
- Her critique of MapReduce/Spark: forces app developers to reason about record-level corruption, parallelism and consistency that a DBMS should abstract away.
- Hiring first ML person: prefer data-engineering experience (SLAs, scheduling, pipelines) over pure modelling experience; ML people without software/data engineering skills are hard to convert. Define ML SLAs, e.g. minimum accuracy before users complain.

**Rules of thumb / checklist**
- Validate as early in the pipeline as possible; gate promotion of retrained models with checks against historical windows, not just the same-batch holdout.
- Monitor for corruption/catastrophic breaks (nulls, schema/column-order changes, failed upstream jobs) before worrying about subtle drift; tie alerts to business impact to avoid fatigue.
- Make retraining cheap and automated so human attention goes elsewhere; the first model is hand-crafted, subsequent ones shouldn't be.
- Capture team knowledge (bad data sources, failed feature ideas) somewhere durable, not Slack streams.
- Be paranoid: ask to re-run the query/notebook rather than trust screenshots.
- Biggest research-to-production bottleneck: knowing whether a new method beats a simple off-the-shelf baseline enough to justify the integration headache. Always compare to a baseline.

**Underappreciated area she named**
- Interpretability as **data provenance**: which training examples drove this prediction (influence functions, "Understanding Black-box Predictions via Influence Functions", Koh & Liang). Current methods are expensive.

**Relevance to Sam**
- Supplier feeds (Viator, RateHawk, Coras) and GeoNames/OSM dumps are exactly the "client changes column order / sends nulls" risk: add lightweight ingestion checks (row counts vs previous run, null-rate per key column, schema/column order, value ranges for lat/lon) and fail loudly before matching/classification runs.
- When re-running location matching or classifiers on refreshed data, compare outputs to the previous snapshot (match rate, class distribution) before promoting, rather than trusting metrics on the same new batch.
- Version data snapshots + code commit + config per run (cheap with Parquet files and git) to get "similar metrics" reproducibility on a laptop.
- DuckDB-style single-node DBMS over Parquet fits his 16GB constraint better than Spark.

