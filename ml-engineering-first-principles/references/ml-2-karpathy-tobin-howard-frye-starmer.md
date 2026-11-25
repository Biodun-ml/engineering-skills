# ml-2 — cross-cutting principles

1. **Check the starting loss before you train.** With uninformed outputs, cross-entropy should be about ln(K) (3.29 for 27 classes). A much higher value means the model is "confidently wrong" at the start, and early training will be wasted undoing that. Fix it with a zero output bias and small output weights. (Karpathy, makemore 3)
2. **Keep activations roughly unit-Gaussian at every layer.** Initialize weights with std = gain/√fan_in (Kaiming): gain 1 for linear, √2 for ReLU, 5/3 for tanh. If activations shrink, the layer does nothing. If they grow, units saturate or die and gradients vanish. (Karpathy)
3. **Watch the training dynamics, not just the loss.** Look at per-layer activation and gradient histograms and saturation %, and above all the update:data ratio (log10 around -3). Well below -3 means the learning rate is too low; well above means it is too high or a layer is mis-scaled. (Karpathy)
4. **Normalization layers help, but BatchNorm couples examples within a batch.** That makes it a regularizer and a source of bugs (train/eval mode, small batches, momentum). Drop the bias right before a norm layer. Prefer LayerNorm/GroupNorm when batches are small or inference is one row at a time. (Karpathy)
5. **Find out what actually limits performance before adding tricks.** BatchNorm gave nothing because the limit was the context length, not optimization. Troubleshoot in the order make it run → make it fast (profile) → make it right. Scale data for overfitting, scale the model for underfitting, and stick to proven architectures and hyperparameters. (Karpathy; Frye, FSDL L3)
6. **Use a model to find dirty data.** Train a rough model first, then sort examples by loss. Confidently wrong and unconfidently right items expose mislabels and hard cases far faster than manual review, and it is not specific to images. (Howard, Lesson 2; Frye, FSDL L3)
7. **Use the confusion matrix and named failure slices to locate the problem.** Group high-loss examples into named suites (e.g. "night", "reflection") and track metrics per slice as regression tests, the ML version of "add every prod bug to the tests". (Howard; Frye)
8. **Treat a test suite as a classifier that trades missed bugs against false alarms.** Before adding a test, compare the real bugs it would catch with the false alarms it could raise. No coverage targets. Minimal opt-in linting. Fast checks on push, slow checks on a schedule. (Frye)
9. **The minimum ML test set is three smoke tests:** loose data expectation tests (nulls, ranges, ordering), a memorization test (it can overfit a tiny batch, also time-boxed by steps or wall time), and loss-based regression suites. (Frye)
10. **Stay close to the data.** Model developers should label fresh production data on a regular rotation. (Frye, via Shreya Shankar)
11. **Deploy early and often, and keep it simple.** Offline metrics miss whether the model solves the user's problem. Path: prototype with a UI at a URL → separate the model from the app → scaling tricks/managed services → edge only if forced. (Tobin, FSDL L5; Howard)
12. **Batch prediction vs model service.** If inputs are finite and enumerable and some staleness is OK, precompute into the DB (simple, scalable), but monitor for silently failed or stale runs. If inputs are open-ended, run a separate model service (independent scaling, isolates failures, but adds a network hop and infrastructure). (Tobin)
13. **CPU first for inference.** A model trained on a GPU doesn't need a GPU to serve. Concurrency with tuned thread counts, caching of frequent inputs, quantization and distillation get you far (Roblox served ~1B BERT requests/day on CPUs). If you do use GPUs, use a serving framework for dynamic batching. (Tobin; Howard: 54 ms CPU inference)
14. **A prediction is weights + preprocessing code + exact library versions.** Package all three together (containers; Cog/BentoML/Truss). Custom functions referenced by a pickled model must ship with the serving code. (Tobin; Howard)
15. **Make rollouts safe:** shadow mode → gradual traffic shift → instant rollback, A/B splits, and a fallback (previous model, simpler model, or rules). Benchmark accuracy *and* latency on the real target runtime, and test size and latency in CI. (Tobin)
16. **Gradient boosting = start from a constant baseline, then add learning-rate-scaled small trees fitted to pseudo-residuals (negative gradients).** Many small steps beat a few large ones. Learning rate × number of trees × tree size control bias and variance. Tune with early stopping on validation. (Starmer, Gradient Boost 1)
17. **Every model needs a trivial baseline.** The mean or prior (GBM's F₀), uniform-softmax loss (Karpathy), a quick fine-tuned model (Howard). Without one you can't tell whether added complexity helps. (all)
18. **Watch out for overconfidence.** Longer training made a classifier flip from 50/50 to 100% on an out-of-distribution "dog-cat". Confidence is not calibration, so don't trust raw probabilities as uncertainty without checking. (Howard; Karpathy)

## Andrej Karpathy — Building makemore Part 3: Activations & Gradients, BatchNorm (https://www.youtube.com/watch?v=P6sfmUTpUmc)
**A network trains well only if its activations and gradients stay well-scaled through every layer; you get there by checking the starting loss, scaling initial weights to the number of inputs (fan-in), adding normalization layers, and watching diagnostic plots.**

**Mental models / first principles**
- Deep nets (and RNNs, which are deep nets once unrolled over time) are hard to optimize with first-order gradient methods. To see why, you have to look at the activations going forward and the gradients coming back, not just at the loss curve.
- Every problem has an **expected loss at initialization**. If nothing is known, the output should be roughly uniform. For a 27-way softmax that means loss = -ln(1/27) ≈ 3.29. Karpathy's model started at 27, meaning it was "confidently wrong" from the first step.
- The goal is roughly **unit-Gaussian activations at every layer at initialization**. If they shrink toward zero, the nonlinearity does nothing. If they grow, the units saturate and gradients die.
- A stack of linear layers with no nonlinearity collapses to a single linear function. You need the nonlinearity for expressiveness, and the price is that you must manage its squashing effect.

**Mechanisms**
- *Fixing output-layer overconfidence:* set the output bias to 0 and scale the output weights down (×0.01). The initial loss falls to about 3.3 and the "hockey-stick" loss curve goes away. That hockey stick is wasted training spent shrinking logits. Validation loss improved from 2.17 to 2.13.
- *Tanh saturation:* the local gradient of tanh is (1 - t²), so a unit near ±1 passes back almost no gradient. Karpathy plots |h| > 0.99 as a white/black image (examples × neurons). A column that is all white is a **dead neuron**: no input ever puts it in its active range, so it never learns. Sigmoid has the same issue. ReLU's flat region gives exactly zero gradient, so a ReLU unit can die at init or be "knocked off the data manifold" by a too-high learning rate and never recover ("permanent brain damage"). Leaky ReLU avoids flat tails. Fixing the pre-activation scale (W1 ×0.2) took validation loss from 2.13 to 2.10.
- *Principled scaling:* when you multiply x by W with both unit-Gaussian, the output std grows by √fan_in. Initialize W with std = gain/√fan_in (Kaiming/He). Gains: linear = 1, ReLU = √2 (because it discards half of the distribution), tanh = 5/3 (it squashes, so you boost). In the example, fan_in = 30 (10-dim embedding × 3 characters), giving about 0.3. This matched the hand-tuned result with no magic numbers. PyTorch's default `nn.Linear` init is uniform(±1/√fan_in) with gain 1.
- The Kaiming paper found that initializing the forward pass correctly also makes the backward pass roughly correct, so most people keep `mode='fan_in'`.
- *BatchNorm (2015):* standardize each unit's pre-activation over the batch (subtract the mean, divide by √(var + ε)), then apply a learnable scale γ (init 1) and shift β (init 0) so the network can still move the distribution. Its running mean and variance are **buffers**, updated by an exponential moving average under `no_grad`, not by backprop. At inference these running statistics are used, so a single example can be passed through. (The alternative is a separate calibration pass over the training set after training.)
- A bias placed just before BatchNorm is useless because the mean subtraction removes it and its gradient is zero. Use `bias=False` (as in ResNet's conv→BN→ReLU motif). BN's β does the biasing.
- PyTorch's BN momentum defaults to 0.1. With small batches (32), the batch statistics jump around, so use a smaller momentum (Karpathy used 0.001) so the running statistics settle.

**Decisions & trade-offs**
- Careful init matters less today because of three things: residual connections, normalization layers (Batch/Layer/Group/Instance), and better optimizers (RMSProp, Adam). In practice Karpathy just scales weights by 1/√fan_in.
- BatchNorm couples the examples in a batch: one example's output depends on whatever other examples were sampled with it. That noise acts as a regularizer (a kind of augmentation), which is why BN is hard to drop. It also causes many bugs (train/eval mode, small batches). Karpathy says he has "shot myself in the foot" with it repeatedly and prefers LayerNorm or GroupNorm.
- BN makes the forward and backward passes robust to weight scale, but **not the update scale**. Change the weight gain and you may have to retune the learning rate.
- Here BN gave no gain, because a single-hidden-layer MLP is not limited by optimization. The bottleneck was the 3-character context length, i.e. the architecture. Find out what is actually limiting you before adding tricks.

**Rules of thumb / diagnostic checklist**
1. Before training, compute the expected initial loss and check the first iteration against it.
2. Plot histograms of activations per layer, with mean, std and % saturated (|t| > 0.97). Aim for about 5% saturation and a similar std at every depth.
3. Plot gradient histograms per layer. They should look similar across depth, with no shrinking or exploding.
4. Check the weight grad:data std ratio per parameter.
5. Most useful: track the **update:data ratio** (lr × grad.std / param.std, log10) over time for each weight matrix. It should be around **-3** (an update is about 1/1000 of the value). Well below -3 means the learning rate is too low. Well above means it is too high or the layer is mis-scaled. This doubles as a way to set the learning rate.
6. Use `torch.no_grad()` for evaluation and buffer updates, so no autograd graph is built.
7. Keep a small amount of randomness in init instead of exact zeros, to break symmetry.

**Failure modes & gotchas**
- An output layer that is too confident at init. Saturated tanh units. Dead ReLUs after a learning-rate spike: forward the whole training set and look for units that never fire.
- Wrong gain in a deep linear+tanh stack: too small shrinks activations to zero, too large saturates them, and gradients become asymmetric across depth.
- The last layer can learn about 10× faster than the others under plain SGD if its scale differs. Adam hides much of this.
- Forgetting to switch BN to eval mode, or evaluating with batch statistics.

**Relevance to Sam**
- The "expected initial loss" check works for any classifier, for example place-category classification. Initial cross-entropy should be about ln(num_classes), or match the class prior if you set the output bias to log prior frequencies.
- Watching the update:data ratio is a cheap, model-agnostic way to tune the learning rate on a laptop without large sweeps.
- For small tabular or embedding MLPs (place matching, ranking towers), use LayerNorm or no normalization rather than BN when batch sizes are small or inference is one row at a time.

## Josh Tobin (Full Stack Deep Learning) — Lecture 05: Deployment, FSDL 2022 (https://www.youtube.com/watch?v=W3hKjXg7fXM)
**Ship a minimal model early, keep the setup simple, and move along a fixed path (prototype, then model separated from the UI, then scaling tricks, then edge) only when the simpler stage stops being enough. Deployment is also how you find out whether the model is any good.**

**Mental models / first principles**
- Offline evaluation misses subtle failures where the model does not solve the user's actual problem. You only find these by deploying, so deploying is part of making the model good.
- Mindset: "deploy early, deploy often; keep it simple, add complexity later."
- Generic architecture: client → network → server → database. The design decision is where the model sits.

**Mechanisms / the path**
1. **Prototype.** Gradio, Streamlit or Hugging Face Spaces. Include a basic UI, not just an API, because the goal is feedback. Put it behind a public URL, which makes it shareable and shows you the real latency. Spend about a day on it. Its limits are UI flexibility and handling many concurrent requests.
2. **Model-in-service** (model loaded inside the web server).
   - Pros: reuses existing infrastructure.
   - Cons: the web stack may be in another language; the model changes more often than the app, so each model update forces a full redeploy; a large model eats the server's resources; web servers rarely have GPUs; the app and the model scale differently.
3. **Batch prediction.** A scheduled job (Airflow, Dagster, Prefect, Metaflow) runs preprocessing, loads the model, predicts for every possible input, and writes the results to the database the app already reads.
   - Fits when the set of inputs is finite and enumerable: one prediction per user or lead, early-stage recommenders, lead scoring.
   - Pros: simple, scales like the database, low read latency, proven at large companies.
   - Cons: no good for large or open-ended input spaces; predictions are as old as the last batch run; **a failed batch job silently serves stale predictions**, so add data-quality and freshness checks.
4. **Model-as-a-service** (separate service, usually REST/JSON; gRPC also common).
   - Pros: a model bug does not crash the app; it can use its own hardware and scale on its own; it can be reused by other apps.
   - Cons: an extra network hop adds latency; more infrastructure to run.
   - The lecturer calls this the "sweet spot for most ML products." There is no standard request schema: GCP, Azure and SageMaker each format "instances" differently.
- **Dependencies.** A prediction depends on the weights *plus* the preprocessing code *plus* exact library versions. Small version bumps can change model behavior.
  - ONNX promises a framework-neutral model format, but in practice has translation bugs and does not handle Python preprocessing well.
  - Docker is the default. It is lighter than a VM, the convention is one container per task (web, DB, queue, worker), and the client/host/registry design lets you pull ready-made images.
  - Cog, BentoML and Truss wrap a `predict` class plus a YAML of dependencies into a standard container.
- **Performance.**
  - GPU vs CPU: being trained on a GPU does *not* mean you must serve on one. CPU with **concurrency** (several model copies per host, with tuned torch thread counts so copies don't fight over threads) can be cheap and fast. Roblox served about 1 billion BERT requests a day on CPUs.
  - **Distillation**: train a small model to imitate a large one. It is tricky to do yourself, so prefer pre-distilled models such as DistilBERT.
  - **Quantization** (fp16 or int8): small accuracy cost, large speed gain. Use the built-in tooling (PyTorch, Hugging Face Optimum, TF Lite). Quantization-aware training keeps more accuracy.
  - **Caching** of frequent inputs. `functools.lru_cache` on the predict function was a key part of the Roblox setup.
  - **Dynamic batching** on GPU: wait to collect requests into a batch, but with a timeout so a quiet period doesn't stall users. This trades throughput against latency. Don't build it yourself.
  - GPU sharing across models. Use servers such as TF Serving, TorchServe, NVIDIA Triton or Ray Serve.
- **Horizontal scaling**: multiple copies behind a load balancer, either via Kubernetes (KServe/Kubeflow, Seldon; "overkill to learn deeply just for ML") or via **serverless** (one container with a single `predict` entry point, pay per use, auto-scaling).
  - Serverless downsides: package-size limits, **cold starts** (seconds to minutes), awkward for multi-step pipelines, no state so caching is hard, weak deploy tooling, CPU-only, and time limits. (note: in 2022 serverless was CPU-only; serverless GPU offerings now exist, and some startups named in the lecture have since shut down. Check current options.)
- **Rollouts** need: gradual traffic shift (1% → 10% → 50% → 100%), instant rollback, traffic splitting for A/B tests, and **shadow mode** (the new model sees live inputs but its outputs are not shown to users).
- **Managed services**: SageMaker is easy on the happy path (HF or sklearn model, `.deploy()`), costs roughly 20% more than raw EC2, and is clunkier for custom containers.
- **Edge.**
  - Use it only for no/poor connectivity, strict privacy, or latency limits you can't meet otherwise. Latency = model time + network round trip.
  - Pros: lowest latency, works offline, private, users supply the compute.
  - Cons: weak hardware, immature tooling, hard to update, and you can't see the production data to debug.
  - Tools: TensorRT (NVIDIA), Core ML / Android NN, PyTorch Mobile, TF Lite, TF.js, Apache TVM. Use efficient architectures (MobileNet's 1×1 convolutions, DistilBERT).

**Decisions & trade-offs (summary)**
- Finite, enumerable inputs and tolerance for some staleness → batch predictions written to the database. Open-ended inputs → a model service.
- CPU first. Serverless if traffic is spiky or low-volume. GPU only with a proper serving framework.
- Edge only when forced. Choose the architecture for the target hardware from day one. Distillation plus quantization buys roughly **2–10×** at most; if the model is 100× too big or slow, drop that architecture.

**Rules of thumb / checklists**
- Add tests for model size and latency in CI so local iteration doesn't quietly blow the budget.
- Before rollout, benchmark accuracy *and* latency on the real production hardware or runtime, because converted models can behave differently.
- Always have a **fallback**: the previous model version, a simpler model, or a rule-based answer when the model errors or times out.
- Version the weights, the preprocessing code and the dependency versions together.

**Failure modes & gotchas**
- Batch jobs that fail silently (stale predictions). Dependency drift between notebook and server. ONNX conversion bugs. Batching latency spikes during low traffic. Cold starts. Debugging blind on edge devices.

**Relevance to Sam**
- Many Trameter ML outputs fit **batch prediction** well: classifying the about 6M GeoNames places, precomputing match/dedup clusters, scoring destination recommendations for a known set of locations. Run it as a chunked job on the laptop or a scheduled worker, write the results to the database or Parquet, and add a freshness/row-count check so a failed run isn't missed.
- Use a model service only for open-ended inputs (free-text search, matching a new supplier's inventory on the fly). On CPU, cache by normalized input, run several worker processes with limited torch/BLAS threads, and quantize embedding models to int8.
- For supplier-matching models, ship behind shadow mode first (log predictions next to the current rule-based matcher), then ramp traffic gradually. Keep the rules as the fallback.

## Jeremy Howard — Lesson 2: Practical Deep Learning for Coders 2022 (https://www.youtube.com/watch?v=F4tvM4Vb3A0)
**Train a quick model *before* you clean the data, then use its most-confused examples to find label errors. Ship it cheaply: train on a free GPU, export, run inference on CPU behind a small UI and API.**

**Mental models / first principles**
- **Model-assisted data cleaning.** The usual advice is to clean the data and then train. Howard reverses it: train a rough model first, then sort the data by **loss**. High-loss items are either wrong with high confidence (often a mislabel or junk item) or right with low confidence (ambiguous or hard cases). Checking only the top of this list finds most of the problems. He calls it a powerful technique almost nobody uses, and says it is not specific to images.
- The confusion matrix shows *which* classes get mixed up. That tells you where the problem is truly hard (grizzly vs black bear) as opposed to dirty data (teddy bears had no errors). Sometimes it shows the task is hard even for experts.
- The GPU is needed only for training. Inference on one input is fast on CPU: 54 ms on a laptop for a ResNet pet classifier. You need a GPU for inference only at thousands of predictions.

**Mechanisms**
- **Resize choices** for fixed input sizes: squish (distorts the shape, keeps everything), crop (keeps the shape, loses edges), pad (keeps both, wastes pixels). These usually make little difference.
- **RandomResizedCrop + aug_transforms** (rotation, warp, lighting, saturation) are applied on the fly in RAM each epoch. Nothing is copied to disk, but the model effectively sees endless variants. Use them when training for more than about 5–10 epochs. RandomResizedCrop also handles images too large to use whole: each epoch samples different patches.
- `ClassificationInterpretation`:
  - `plot_confusion_matrix`
  - `plot_top_losses`, which shows prediction, actual, loss and probability
  - `ImageClassifierCleaner`, which lists items per class sorted by loss so you can relabel or delete them, for train or validation separately
  After cleaning, retrain.
- **Export/load**: `learn.export()` pickles the learner (weights plus transforms). `load_learner()` restores it for `.predict`. Gotcha: custom functions used in labelling (e.g. `is_cat`) are not pickled as source. They must be defined again in the serving script.
- **Serving**: Gradio app on Hugging Face Spaces, deployed via git push of `app.py` plus the model file. Gradio needed plain Python floats, not tensors or NumPy, in the returned `{label: prob}` dict. nbdev's `#|export` cell markers plus `notebook2script` turn the notebook cells you need into `app.py`, so experiments stay in the notebook.
- Spaces exposes an **HTTP API** for the model. A static HTML+JS page hosted free on GitHub Pages can call it with `fetch` and handle single or multiple files. That separates the UI from the model (same idea as FSDL's model-as-service) and lets you **chain models**: a breed classifier feeding an NLP Q&A model.

**Decisions & trade-offs**
- How many epochs: train until the error is good enough, or you run out of time or compute, or the validation error starts rising (overfitting). For better results use more augmentation plus longer training.
- Gradio is the easiest option; Streamlit is more flexible but harder. Both are for prototypes. For a real app, use their API from your own frontend.
- Environment: use your own Python distribution (conda/mamba), not the system Python ("always leads to disaster"). Prefer conda/mamba over pip when you need a GPU, because pip may not install a CUDA-compatible build. (note: Mambaforge is now folded into Miniforge, and fastpages has been deprecated in favour of Quarto. Use current tools.)

**Rules of thumb**
- Before any manual cleaning pass, train a baseline and review the top-loss items for each class and split, validation included.
- Look at the first page of each sorted list. If it is clean, the rest probably is too.
- Use what you see to improve how you collect the *next* dataset, or to automate cleaning.
- GPU memory doesn't swap like RAM, so close idle notebooks that hold GPU memory.
- Use the same preprocessing at train and serve time, and ship helper functions with the model.

**Failure modes / gotchas**
- A missing labelling function at load time. Returning tensors where the UI expects plain floats. Overtraining makes the model overconfident on out-of-distribution inputs: the "dog-cat" hybrid went from 50/50 to 100% cat after retraining. Confidence is not calibration.

**Relevance to Sam**
- Directly useful for location matching and dedup. Train a quick matcher or place-type classifier on the noisy labels, then sort pairs and rows by loss. The confidently wrong ones are usually bad GeoNames feature codes, wrong supplier mappings or duplicates. Fix those first instead of hand-checking millions of rows. Fits a 16GB laptop because you review a ranked top-k list, not everything.
- The confusion matrix per place category (e.g. hotel vs hostel vs guesthouse) separates real ambiguity, which needs a taxonomy decision, from label noise.
- Serving pattern for internal tools: CPU inference behind a small API, with a plain JS frontend calling it.

## Charles Frye (Full Stack Deep Learning) — Lecture 03: Troubleshooting & Testing, FSDL 2022 (https://www.youtube.com/watch?v=RLemHNAO5Lw)
**Treat a test suite like a classifier that weighs catching bugs against false alarms. For ML, start with cheap smoke tests (data expectations, memorization, regression suites built from high-loss examples). Troubleshoot models in the order: make it run, then make it fast, then make it right.**

**Mental models / first principles**
- Python tests do not prove correctness. Following Nelson Elhage, think of a test suite as a **classifier of commits** ("bug or fine"). It has two error types: missed bugs and false alarms. The sign of a false alarm is a failing test followed by a commit that fixes the *test*, not the code.
- Before adding a test, list the real bugs it would catch and the false alarms it could raise. If the false alarms outnumber the bugs, drop it. The exception is high-stakes or regulated systems (medical imaging, self-driving, finance), where you do need high confidence.
- Software compiles source into a program. ML "compiles" **data into a model by training**. All three parts are harder to test: data is heavy and opaque, training is less mature than a compiler, and models are hard to inspect. So start with smoke tests: the 20% of tests that give 80% of the value.
- Gradient descent is already test-driven development: the loss is a fuzzy test the model trains against. But the loss says *that* you fail, not *why*. Structured test suites supply the why.
- Production differs from development, especially in its data, so you must also **test in production**. That means monitoring and fast fixes, not shipping junk.

**Mechanisms / tooling**
- pytest (marks, fixtures, parametrization). doctest keeps docstrings in sync with code. Smoke-test notebooks by running them end to end with asserts (nbformat).
- Use code coverage (Codecov) for visibility, but **do not gate merges on a coverage %**, because people then write low-value tests just to hit the number.
- Linting: black (auto-fixes formatting), flake8 with plugins (docstrings, types, security), shellcheck (explains bash pitfalls). Keep the rules minimal and **opt-in per file**, especially in legacy code. Pedantic linting backfires: 100k+ GitHub commits mention "placating flake8". (note: today ruff mostly replaces black and flake8.)
- Automation: GitHub Actions. Copy PyTorch's split: code checks under about 10 minutes run on every push/PR; slow or non-code jobs (long test suites, dependency checks, stale-PR cleanup) run on a schedule. Pre-commit hooks must finish in a few seconds or people stop committing. Bad automation, like bad tests, costs more than it saves, so give it to senior engineers.
- **Data: expectation tests** (Great Expectations). Assert simple properties: no nulls, end date after start date. Start with a few and make them *loose* so they don't fire falsely. Example: height ≥ 0 and < 30 ft catches inches typed in by mistake without assuming a distribution. Only alert on things worth paging someone for.
- **Stay close to the data.** Annotation setups ranked from worst to best: external benchmark or vendor → model devs labelling ad hoc at the start → internal annotation team with regular syncs → **an on-call rotation where model developers label fresh data themselves** (via Shreya Shankar).
- **Training: memorization tests.** The model must be able to overfit a tiny subset (Lightning `overfit_batches`). This catches only gross bugs: broken gradients, numerical problems, shuffled labels. To make it more sensitive, also assert on wall time and on the number of steps needed to reach a target loss. To speed it up (<10 minutes so it can run per PR), in order: faster hardware → data the size of one batch → turn off regularization (dropout, augmentation) → smaller layers or fewer of them → remove the most expensive components. You end up with tiers of tests from cheap to realistic.
- Rerunning old training jobs on new code is expensive: GPU CI cost about $24k/yr minimum on CircleCI, and GitHub Actions had no GPU runners then. Better: **regularly retrain on fresh production data**. That money builds the model instead of only testing code, but it needs the data flywheel and monitoring in place.
- **Models: regression suites.** Collect the highest-loss validation, test and production examples into a "hard" suite. High loss exposes *bad labels* as well as model weaknesses. Then **group the failures into named suites** (pedestrian in shadow, windshield reflection, night) and track metrics per suite, the ML version of "add every production bug to the test suite". Domino (foundation-model embeddings that find error slices) can help automate this. CheckList offers behavioral tests for NLP. Google's **ML Test Score** is a strict rubric for mature systems.
- **Troubleshooting, step 1: make it run.**
  - Shape errors: annotate expected shapes in comments and step through in a debugger.
  - OOM: use the lowest precision that works (fp16/bf16 by default), auto-scale the batch size, and use **gradient accumulation** if the batch gets too small for stable gradients. Then tensor parallelism or gradient checkpointing.
  - NaN/Inf: these usually start as gradient spikes or vanishing gradients, so log gradient norms. Switch to float64. If the problem disappears it was precision; if not, the numerical code is wrong and you should use a stable implementation. Normalization layers are common culprits (the DALL·E-mini spikes).
- **Step 2: make it fast.** Performance is counter-intuitive. In transformer layers the MLP takes longer than attention at typical sizes. Optimizer state can use more memory than the parameters. Data loading can be the bottleneck. **Profile**; there are often cheap fixes in plain Python code.
- **Step 3: make it right**, last, because models are never fully right. Scale fixes most things:
  - overfitting (production loss much higher than training) → more data
  - underfitting → bigger model
  - distribution shift (train and val low, test or production high) → more of both
  Scaling laws make this predictable. Without the budget, fine-tune a pretrained model. Other fixes are task-specific, so **stick to proven architectures and hyperparameters** (HF Hub, Papers with Code).

**Rules of thumb**
- Data expectations + memorization test + high-loss regression suites is a minimum ML test set you can build in a day.
- Grow named failure-slice suites over time. Report metrics per slice, not only the overall score.
- Keep CI under 10 minutes on push; schedule everything else.

**Failure modes & gotchas**
- Coverage targets. Strict expectation ranges that cause alert fatigue. Memorization tests that miss subtle bugs. Pre-commit hooks that are too slow. Model developers who never look at the data. (note: Lightning's batch-size finder now lives in the `Tuner` API.)

**Relevance to Sam**
- Expectation tests fit the GeoNames/OSM/GTFS pipelines well: lat in [-90, 90], lon in [-180, 180], non-null country code, unique IDs, row counts within ±X% of the last run, no (0, 0) coordinates. Use DuckDB or Pandera checks per chunk if Great Expectations is too heavy for 16GB.
- Build matching regression suites from named failure slices: transliterated names, same name in different admin regions, airport vs city, ferry terminal vs port. Track precision and recall per slice every time the matcher changes.
- Label fresh supplier mappings yourself on a schedule (the "on-call labelling" idea) to keep intuition about the data.

## Josh Starmer (StatQuest) — Gradient Boost Part 1 (of 4): Regression Main Ideas (https://www.youtube.com/watch?v=3CC4N4z3GJc)
**Gradient boosting for regression starts from the mean. It then repeatedly fits a small tree to the current residuals and adds a shrunken (learning-rate-scaled) copy of that tree to the prediction: many small steps that end with low bias and controlled variance.**

**Mental models / first principles**
- Boosting is *sequential error correction*. Each new learner models what the ensemble so far gets wrong.
- Compared with AdaBoost:
  - AdaBoost uses stumps (1 split) and gives each stump a different "amount of say" based on its accuracy.
  - Gradient boosting starts from a single **leaf** (the initial guess), uses **larger but still capped trees** (commonly 8–32 leaves; 4 in the toy example), and scales **every tree by the same learning rate**.
- The target of each tree is the **pseudo-residual**, observed minus current prediction. "Pseudo" because with squared-error loss the residual is the negative gradient of the loss with respect to the prediction (derived in Part 2). With other losses it is the gradient, not a literal residual. That is the "gradient" in gradient boosting.
- Bias/variance: one full-strength tree fitted to the residuals can reproduce the training targets exactly (71.2 + 16.8 = 88, the observed weight). That is low bias but high variance. Shrinkage trades a slower fit for better generalization. Friedman found empirically that many small steps beat a few large ones on test data.

**Mechanism (the common regression configuration)**
1. F₀ = mean of the target (71.2 kg in the example).
2. For m = 1..M:
   a. residuals rᵢ = yᵢ − F_{m−1}(xᵢ);
   b. fit a regression tree with at most L leaves to predict r from the features (height, favourite colour, gender);
   c. each leaf's output is the **mean of the residuals** in that leaf (several rows share a leaf because L < n);
   d. F_m(x) = F_{m−1}(x) + ν · tree_m(x), with learning rate ν ∈ (0,1], e.g. 0.1.
3. Stop at M trees or when new trees no longer reduce the residuals much.
4. Predict for a new x: F₀ + ν·Σ tree_m(x).
- Worked example: 71.2 → 71.2 + 0.1×16.8 = 72.9 → +0.1×15.1 = 74.4, getting closer to 88. All residuals shrink every round. Later trees can choose different splits.
- The algorithm has many configurable parts, but "99% of the time" one setup is used for regression (squared loss, mean start) and one for classification (log-loss, log-odds start; Parts 3–4).

**Decisions & trade-offs**
- **Learning rate vs number of trees**: a smaller ν needs more trees but usually generalizes better. Tune them together, typically with early stopping on a validation set. (Practical addition, not from the video: libraries like LightGBM/XGBoost expose `learning_rate`, `n_estimators`, `num_leaves`/`max_depth` and early stopping. Use ν around 0.03–0.1 plus early stopping.)
- **Tree size (max leaves)** sets how many features can interact in one tree and how much variance each tree adds. Stumps give an additive model; 8–32 leaves captures interactions.
- Don't confuse gradient boosting for regression with linear regression. They are related ideas (residuals) but different models.

**Rules of thumb**
- Always start from a sensible constant baseline (the mean for regression, log-odds of the base rate for classification). It is also the baseline the boosted model must beat.
- Watch the residuals: each round should shrink them on train, and validation error should be monitored to decide when to stop.
- Expect boosting to overfit if ν is too high or trees too deep. Shrinkage is the main regularizer covered here. (Subsampling rows/columns and leaf constraints are others, not covered in Part 1.)

**Failure modes & gotchas**
- One unshrunk tree fitted to the residuals reproduces the training data almost exactly, a classic overfit.
- Adding trees without a stopping rule keeps lowering training error while test error rises.

**Relevance to Sam**
- Gradient-boosted trees are the default strong baseline for Trameter's tabular problems: duplicate/match scoring on engineered pair features (name similarity, haversine distance, shared admin codes, feature-class agreement), place-type classification from structured attributes, and ranking (LightGBM `lambdarank` for travel recommendations).
- They are fast and memory-light on a 16GB laptop. LightGBM's histogram binning handles millions of rows, and categorical features like country code can be used directly.
- Use early stopping on a **group-aware** validation split (e.g. by country or region) so leakage across nearby duplicate places doesn't inflate validation scores.
