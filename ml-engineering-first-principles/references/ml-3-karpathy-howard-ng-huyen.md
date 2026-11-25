# ml-3 — cross-cutting principles

1. **Train a trivial baseline on day one, then add one component at a time and measure each.** Karpathy goes bigram 2.5 → +attention 2.4 → +multi-head 2.28 → +FFN 2.24 → +residual 2.08 → +LayerNorm 2.06 → scaled up 1.48 val loss. Howard says most industry projects wait far too long to train their first model. (Karpathy GPT; Howard L3)
2. **Spend early effort on data, not architecture.** Start with a small, fast model (ResNet18/34) so iterations are quick. Try fancier architectures last, and pick them by speed, memory and accuracy against your latency budget. (Howard L3)
3. **Sanity-check the loss at initialisation.** A uniform guess over C classes should give −ln(1/C) (4.17 for 65 classes). A higher value means the starting predictions are overconfident and wrong. (Karpathy GPT)
4. **The learning rate is the most important hyperparameter.** A loss that climbs or diverges means the LR is too high. Painfully slow progress means it is too low. Bigger or attention-based models need a lower LR, and AdamW at ~3e-4 is a sane default. (Howard L3; Karpathy GPT)
5. **Deep learning = ReLUs (via matmul) + a loss + gradient descent.** Everything else makes it faster or more data-efficient. Linear layers stacked without a nonlinearity are still linear. (Howard L3)
6. **Each loss follows from an assumption about the noise.** Least squares is MLE under Gaussian IID errors, and logistic/cross-entropy is MLE for Bernoulli labels. Write the assumption down first, and switch to a GLM (e.g. Poisson for counts) when it doesn't hold. (Ng CS229 L3)
7. **Don't use linear regression with a threshold for classification.** One far-away point moves the boundary. Logistic regression has a concave log-likelihood (no local optima) and is a top-two everyday baseline. (Ng CS229 L3)
8. **Pick the optimiser by parameter count.** Newton's method converges quadratically and wins with tens of parameters. With thousands or more, the Hessian is infeasible, so use gradient methods. (Ng CS229 L3)
9. **Parametric vs non-parametric affects memory and latency.** Non-parametric methods (locally weighted regression, k-NN) keep all the training data and need spatial indexes (KD-trees) at millions of rows. Kernel bandwidth controls the underfit/overfit trade-off. (Ng CS229 L3)
10. **Feature hygiene for tabular data:** n−1 dummy columns for categoricals (including numeric-looking levels), scale numerics so no column dominates, log-transform heavy-tailed money or count features, and add a constant column for the bias. (Howard L3)
11. **Deep networks only optimise with residual connections and normalisation.** The residual stream gives gradients a clean path to the inputs, and pre-norm LayerNorm keeps activations well scaled. If adding depth makes results worse, add these first. (Karpathy GPT)
12. **Control variance at initialisation.** Scale attention logits by 1/√d_k so softmax doesn't collapse to one-hot, and initialise parameters as small random values around zero. (Karpathy GPT; Howard L3)
13. **Evaluate properly.** Average the loss over many batches for both train and val. Switch to eval mode and no_grad when evaluating. Watch for train loss pulling ahead of val as the first sign of overfitting, and use dropout or more data when it appears. (Karpathy GPT)
14. **IID is usually an approximation.** Errors for nearby or same-street items are correlated. Accept the assumption for efficiency, but your validation split must respect the correlation (time, region or group splits). (Ng CS229 L3; Karpathy's held-out tail split)
15. **A deployable model = preprocessing pipeline + weights + label vocab, exported together.** Training needs a GPU, inference usually doesn't. (Howard L3)
16. **Latency and staleness are different things.** Batch predictions are fast but stale. Online inference on batch features is fresh, but the features are not. Streaming or event-triggered features get close to zero staleness at a compute cost. Decide freshness per feature. (Chip Huyen)
17. **Streaming data is a CSV that keeps changing underneath you.** Enforce schemas and data contracts at ingestion and treat the compute engine as a deliberate choice, unlike the stationary-data habits of pandas. (Chip Huyen)
18. **Pair matching is classification.** "Are these two texts the same concept?" can be framed as one input (text1 + text2) going into a classifier. Use encoder models (bidirectional, no causal mask) to embed strings, and decoder models for generation. (Howard L3; Karpathy GPT)

---

## Andrej Karpathy — Let's build GPT: from scratch, in code, spelled out (https://www.youtube.com/watch?v=kCc8FmEb1nY)
**A GPT is a stack of blocks that alternate "communication" (masked self-attention between tokens) with "computation" (a per-token MLP), wrapped in residual connections and layer norm, trained only to predict the next token.**

**Mental models / first principles**
- A language model completes sequences. Pre-training produces a *document completer*, not an assistant; turning it into an assistant is a separate fine-tuning stage (supervised Q&A examples, then a reward model trained on human rankings, then PPO against that reward).
- Tokenization is a trade-off between vocabulary size and sequence length: character level gives 65 symbols and long sequences; GPT-2's BPE (tiktoken) has ~50k tokens, so "hi there" becomes 3 integers. Subword tokenizers (SentencePiece, BPE) are the usual choice in practice.
- Attention is a **communication mechanism on a directed graph**: each node holds a vector and pulls in a data-dependent weighted sum of the nodes that point to it. In language modelling the graph is causal (token t sees tokens 1..t). Nothing in attention knows about position or space, so position has to be added explicitly. Convolution is different because it has spatial layout built in.
- Query = "what am I looking for", Key = "what do I contain", Value = "what I will tell you if you find me interesting". The raw x is private to the token; v is what it shares.
- Encoder block = no mask (every token talks to every other, useful for classification/sentiment). Decoder block = triangular mask (autoregressive). Cross-attention = queries come from x, keys/values come from another source (e.g. an encoder output). The original paper was encoder-decoder because it did translation; GPT is decoder-only because it conditions on nothing.
- Batch elements never talk to each other. Batching exists only to keep GPUs busy.

**Mechanisms**
- Data: encode the whole corpus as one long integer tensor, split 90/10 train/val (keep val to measure overfitting, since the goal is to generalise rather than memorise). Never feed the whole text: sample random chunks of `block_size` (context length). A chunk of block_size+1 tokens holds block_size training examples (context lengths 1..T). This also teaches the model to predict from as little as one token of context, which inference needs.
- Bigram baseline: an embedding table of vocab×vocab gives logits straight from the current token. Loss = cross-entropy. PyTorch wants (B·T, C) logits and (B·T) targets, so reshape with `.view`.
- **Sanity-check initial loss**: with 65 classes a uniform guess gives −ln(1/65) ≈ 4.17. Seeing 4.87 at init means the starting predictions are a little over-confident and wrong.
- The "weighted aggregation" trick: averaging over all previous tokens is just a matmul with a row-normalised lower-triangular matrix. Version 3: start with zeros, `masked_fill` the future with −inf, softmax each row. Making those zeros *data-dependent* (q·kᵀ) is self-attention.
- One head: `k, q, v = Linear(C, head_size, bias=False)(x)`; `wei = q @ k.transpose(-2,-1) * head_size**-0.5`; mask; softmax; `out = wei @ v`. The mask matrix is registered as a buffer, not a parameter.
- **Why scale by 1/√d_k**: with unit-variance q and k, the dot products have variance ≈ head_size. Large logits make softmax close to one-hot, so each token would pull from just one other token at init. Scaling keeps the variance at ~1 and the attention spread out.
- Multi-head = several smaller heads in parallel, concatenated (like grouped convolution), e.g. 4 heads × 8 dims = 32. Each head is a separate communication channel.
- Feed-forward = per-token Linear → ReLU (GELU in GPT-2) → Linear, inner width 4× n_embd. "Tokens gathered data, now they think about it individually."
- **Residual connections**: x = x + attn(ln1(x)); x = x + ffwd(ln2(x)). Addition passes gradient equally to both branches, which gives a "gradient superhighway" from loss to input. Blocks start out contributing little and come online as training goes. A projection Linear maps each branch back into the residual stream.
- **LayerNorm** = BatchNorm that normalises rows (per token, across features) rather than columns, so there are no running buffers and no train/eval difference. Modern Transformers use **pre-norm** (norm before the sublayer), which differs from the paper's post-norm. There is also a final LayerNorm before the LM head.
- Dropout goes on the attention weights after softmax, on the attention projection, and on the FFN output before it rejoins the residual. It works like training an ensemble of subnetworks.
- Generation: crop the context to the last `block_size` tokens (the position table only covers that many), take the last time step's logits, softmax, `torch.multinomial`, concatenate, repeat.

**Numbers given (val loss on tiny Shakespeare, char level)**
- Bigram 2.5 → one attention head 2.4 → 4 heads 2.28 → +FFN 2.24 → +residual & 4× FFN 2.08 (train loss now pulling ahead of val, the first sign of overfitting) → +LayerNorm 2.06 → scaled up (batch 64, block 256, n_embd 384, 6 heads of 64, 6 layers, dropout 0.2, lower LR) **1.48**. About 10M parameters, ~15 minutes on an A100. He says not to try this config on a CPU or laptop without shrinking layers and width.
- GPT-3: 175B params trained on 300B tokens. That is roughly 1,000,000× the tiny-Shakespeare token count, and the architecture is nearly identical.

**Decisions & trade-offs / rules of thumb**
- AdamW at ~3e-4 is a good default. Tiny nets tolerate 1e-3 or higher. Attention nets tolerate less, so lower the LR and train longer. Lower the LR again as the model grows.
- Don't trust a single batch's loss. Write an `estimate_loss()` that averages over many batches for both splits.
- Use `model.eval()`/`model.train()` around evaluation (dropout and batchnorm behave differently) and `torch.no_grad()` there (it skips storing activations, which saves memory).
- Move data, model and generated context to the same device.
- Deep nets stop being optimisable without residuals and normalisation. If stacking blocks makes the result *worse*, add these before touching anything else.
- Build up incrementally from a trivial baseline (bigram) and measure val loss at each step. Each architectural piece has to earn its place with a number.

**Failure modes & gotchas**
- Shape conventions: cross_entropy expects channels in dim 1. Batched matmul broadcasts (T,T) @ (B,T,C).
- If generation feeds more than block_size tokens, the position-embedding lookup goes out of range.
- A reasonable-looking loss is not the same as reasonable text. Always sample and look at the output.

**Relevance to Sam**
- The baseline → measure → add one component loop is the template for any model, including place-matching classifiers.
- For deduplicating or matching place names, attention and embeddings are the reason pretrained small encoders (no causal mask, full bidirectional attention) beat bigram/n-gram features. Use an *encoder* model for embedding place strings, not a decoder.
- On a 16GB laptop, train small: shrink n_embd and n_layer, use char/subword level, and check the init loss against ln(vocab) before spending compute.
- A 90/10 split taken from the *end* of a sequential corpus is a time-style split. With location data, split by region or source rather than at random, so near-duplicate rows can't leak across splits.

## Jeremy Howard — Lesson 3: Practical Deep Learning for Coders 2022 (https://www.youtube.com/watch?v=hBBOjCiFcuo)
**Deep learning = add up lots of ReLUs (via matrix multiplication) into a flexible function, define a loss, and use gradient descent to set the parameters. Everything else is tweaks to make it faster or need less data.**

**Mental models / first principles**
- A trained model is two things: the **preprocessing pipeline** (data blocks, normalisation, the category vocab) plus the **parameters**. fastai's `model.pkl` (a Learner) bundles both. Training needs a GPU. Inference usually doesn't, so training and deployment are separate steps.
- Machine learning means fitting a function to data. The worked example: recover 3x²+2x+1 from noisy points. By hand: nudge one coefficient, check whether the fit improved, keep the direction that helps. Make "improved" objective with a **loss** (MSE). Replace manual nudging with the **derivative**, which says which direction and how hard to push each parameter. That is gradient descent.
- **ReLU** = max(0, mx+b). Two summed ReLUs make a hockey-stick shape. Enough of them approximate any squiggly function, in any number of dimensions. "There is nothing between draw two circles and draw the rest of the owl": ReLUs + gradient descent + input/output examples, and the computer draws the owl.
- Summing two linear functions just gives another linear function. The nonlinearity is what gives a network its expressive power.
- **Matrix multiplication** computes many sum-products (many linear units) at once. It is almost the only linear algebra needed in DL, and it is what GPU tensor cores do (4×4 matmuls tiled across larger matrices).

**Mechanisms**
- PyTorch loop: params = tensor(...).requires_grad_(); loss = f(params); loss.backward() fills params.grad; then inside `torch.no_grad()` do params -= lr * params.grad (and zero the grads). Every optimiser is built on this.
- Gradient units: "if this parameter rose by 1 and the slope stayed constant, the loss would change by this much". The slope flattens as you approach the minimum, and near any optimum the function looks roughly quadratic. Stepping by the raw gradient overshoots, so multiply by a small **learning rate**. That LR is a *hyperparameter*: a parameter used to find the parameters.
- LR too high → you overshoot, bounce, and the loss gets worse and worse (divergence). If training loss *rises*, suspect the LR first. LR too low → painfully slow, because steps shrink as the slope flattens.
- Initialise parameters to small random values either side of zero (he uses rand − 0.5).

**Tabular from scratch (Titanic in Excel)**
- Drop clearly useless ID/name columns. Rows missing age or embarked were simply dropped for the demo.
- Categorical → binary **dummy variables**. A variable with n levels needs n−1 columns (Pclass 1/2/3 → IsClass1, IsClass2). Pclass is a number, but it is a level rather than a quantity, so it should still be one-hot.
- **Normalise** features to similar scales (age / max age). Without this, the large-magnitude column (fare, up to $512) dominates everything.
- **Log-transform** heavy-tailed money-like features (a few huge values, many small). log(fare) comes out roughly even.
- Bias trick: add a constant column of 1s, and its coefficient becomes the intercept.
- Linear regression solved by gradient descent (Excel Solver): mean squared loss 1.03 → 0.10. Two linear units → ReLU each → sum: 0.08. Rewritten as a single MMULT with the transposed parameter matrix, it gives the same result. Predictions can fall outside [0,1], which motivates a sigmoid later.

**Decisions & trade-offs / rules of thumb**
- **Start every project with a small, fast model (ResNet18/34) and spend the time on data**: cleaning, augmentation, external data. Trying better architectures comes *last*. Jumping to fancy models first is "the number-one beginner mistake."
- When you do choose an architecture, judge it on three axes: **speed, memory, accuracy**. Use a speed-vs-accuracy plot (timm has 500+ models) and pick for your latency budget. In his pets example, ConvNeXt (via `timm`, passed to `vision_learner` as a name string) cut error from 7.2% to 5.5%, roughly 30% relative, with epochs going from 20s to 27s. (Note: he says "accuracy" on screen but means error rate.) "in22ft1k" models are pretrained on ImageNet-22k and usually better on natural photos.
- **Train your first model on day one** with whatever data you can hack together. You learn early whether the problem is already solved well enough, or impossible. Most industry projects wait far too long.
- "Do I have enough data?" can't be answered until you train. Also separate *inputs* from *labels*: unlabelled inputs are often cheap and labels expensive (medical imaging masks). Semi-supervised learning and augmentation help get more out of unlabelled data.
- Predict outputs come back in the order of `dls.vocab`. Zip them with the probabilities to label them.
- NLP is next: document classification (sentiment, routing, legal relevance). Even "are these two phrases the same concept?" (patent phrase-matching, scored 0/0.25/…/1) is framed as classifying the pair "text1 + text2".

**Failure modes & gotchas**
- Unscaled features silently let one column dominate.
- Linear models stacked without a nonlinearity are still linear.
- Loss going up during training usually means the LR is too high.
- Don't brute-force all 500 architectures. Pick from the trade-off plot once the data work is done.

**Relevance to Sam**
- Place deduplication can be framed exactly like the patent task: a **pair classifier** on (name_a, name_b, plus features) predicting same/not-same. Start with a simple model on day one and iterate on the data (normalising names, transliteration, blocking) before upgrading the model.
- For tabular place features: one-hot the low-cardinality categoricals (feature class/code), log-transform heavy-tailed counts like population, scale the numerics, and add a bias column for linear baselines.
- On 16GB, pick small, fast pretrained backbones and spend the effort on data. Inference runs on CPU fine, so ship the export (preprocessing + weights) together.

## Andrew Ng (Stanford CS229) — Locally Weighted & Logistic Regression, Lecture 3 (https://www.youtube.com/watch?v=het9HFqo1TQ)
**Squared error and logistic loss are not arbitrary. Each is maximum-likelihood estimation under a stated assumption about P(y|x;θ). Locally weighted regression avoids hand-picking features by fitting a fresh weighted line around every query point. Newton's method trades expensive steps for very few iterations.**

**Mental models / first principles**
- **Feature choice is model choice.** A linear model can fit curves if you add features (x², √x, log x). A quadratic eventually curves back down, so √x or log x may suit "rises then flattens" data such as house price vs size. Feature selection algorithms automate this choice.
- **Parametric vs non-parametric**: parametric models (linear/logistic regression) keep a fixed set of θ, and you can throw the training data away afterwards. Non-parametric models (locally weighted regression, and k-NN in spirit) must keep the training data, and storage grows linearly with it. That makes them costly at scale.
- **Probabilistic view of linear regression**: assume y = θᵀx + ε, with ε ~ N(0, σ²) and IID. The likelihood L(θ) = Πp(yᵢ|xᵢ;θ). Taking logs turns the product into a sum, and maximising it is the same as minimising ½Σ(yᵢ − θᵀxᵢ)². **Least squares is MLE under Gaussian IID noise.** Gaussian noise is a reasonable default because of the central limit theorem (many small, roughly uncorrelated causes add up). If the noise is far from Gaussian, use a different distribution via GLMs (e.g. Poisson for counts).
- IID is often false. Errors for houses on the same street are correlated. You assume it anyway when it gives an efficient algorithm that is good enough, and only model the dependence when the violation is severe.
- Likelihood vs probability: it is the same number. Viewed as a function of θ with the data fixed, it is "likelihood of the parameters". Viewed as a function of the data with θ fixed, it is "probability of the data". The semicolon (p(y|x;θ)) means θ is a parameter, not a random variable (frequentist convention).

**Mechanisms**
- **Locally weighted linear regression (LOESS-style)**: for each query x, minimise Σ wᵢ(yᵢ − θᵀxᵢ)², with wᵢ = exp(−(xᵢ−x)²/(2τ²)). Nearby points get weight ≈1, distant ones ≈0. This bell curve is *not* a probability density (it doesn't integrate to 1). The bandwidth τ is a hyperparameter: too wide over-smooths (underfit), too narrow gives a jagged fit (overfit). Variants include triangle kernels and τ that adapts to local density. Each prediction means solving a weighted normal-equation system. That's fine for thousands of points. For millions, use KD-trees or similar.
- **Don't use linear regression + 0.5 threshold for classification**: a single far-right, obviously-positive point drags the fitted line and moves the decision boundary. Outputs outside [0,1] also make no sense.
- **Logistic regression**: h(x) = g(θᵀx), with g(z) = 1/(1+e^(−z)) (sigmoid = logistic). Assume P(y=1|x;θ) = h(x), which compresses to p(y|x) = h^y (1−h)^(1−y). The log-likelihood is ℓ(θ) = Σ[y log h + (1−y) log(1−h)], i.e. negative cross-entropy. Maximise by **gradient ascent**: θⱼ := θⱼ + α Σ(yᵢ − h(xᵢ))xᵢⱼ.
- That update has *the same surface form* as the linear-regression LMS rule. Only h differs. Every generalised linear model shares this form.
- The logistic log-likelihood is **concave**, so there are no local optima and the only maximum is global. That is part of why the sigmoid is chosen. There is **no closed form** (no normal-equation equivalent), so it has to be solved iteratively.
- **Newton's method**: to solve f(θ)=0, step θ := θ − f(θ)/f′(θ), i.e. follow the tangent line to where it crosses zero. To maximise ℓ, set f = ℓ′, giving θ := θ − ℓ′/ℓ″. Vector form: θ := θ − H⁻¹∇ℓ, where H is the Hessian. It has **quadratic convergence**: near the optimum the number of correct digits roughly doubles each step (0.01 → 0.0001 → 0.00000001).

**Decisions & trade-offs / rules of thumb**
- Use LWR when the dimension is low (n ≈ 2–4), there is plenty of data, and you don't want to engineer features. It is poor at extrapolation, but so are most learners.
- Newton vs gradient descent: with tens of parameters (10–50), Newton converges in about 10 iterations and wins. With 10k–100k+ parameters, building and inverting a d×d Hessian is infeasible, so use gradient methods.
- Ng's two most-used algorithms are linear and logistic regression. They are strong baselines.
- Write down the noise/label assumption first, then derive the loss with MLE. The loss follows from the assumption.

**Failure modes & gotchas**
- Linear regression on 0/1 labels gives a fragile boundary.
- Non-parametric methods blow up memory and latency as the data grows.
- An overly narrow kernel or bandwidth overfits. An overly wide one underfits.
- Correlated errors (same street, same region) break the IID assumption behind the loss and behind naive validation.

**Relevance to Sam**
- **Place-pair matching**: logistic regression on pair features (name similarity, distance, same country/admin code, type match) is the right first baseline. It is calibrated-ish, gives probabilities, has a concave objective, and trains fast on millions of rows with a small parameter count. With few features, Newton or L-BFGS solvers (e.g. sklearn's `lbfgs`/`newton-cg`) converge in a handful of iterations.
- **Geospatial locality = LWR idea**: weight by distance to the query (a Gaussian kernel on haversine distance) for local estimates such as price or popularity smoothing. Use a KD-tree or BallTree to keep it tractable on millions of points on 16GB.
- Location rows are spatially correlated, so they are not IID. Validate with spatial/group splits and don't trust random-split metrics.
- For heavy-tailed or count targets (reviews, bookings), consider Poisson or log-transformed targets rather than plain squared error.

## Chip Huyen — Real-time Machine Learning (MLOps Coffee Sessions #133) (https://www.youtube.com/watch?v=6i6t-HHL9So)
**Latency and staleness are separate properties of an ML system. Streaming lets you cut feature and prediction staleness to near zero, and the conceptual leap from pandas thinking is mostly about data that keeps arriving, enforced schemas, and a different compute engine.**
(Mostly a conversational podcast. Technical content is concentrated in two segments, plus a few practitioner lessons.)

**Mental models / first principles**
- **Latency ≠ staleness.** Latency = time from receiving a prediction request to returning a prediction. Staleness = how old the information behind that prediction is. You can have low latency with very stale data, and the reverse.
- The spectrum of designs:
  1. **Batch predictions** (precomputed, then looked up): latency is just a fetch, so very low, but the predictions are stale (computed yesterday, say).
  2. **Online predictions on batch features**: latency = fetch precomputed features + run the model. Predictions are fresh, but the features are only as fresh as the last batch job (e.g. daily).
  3. **Online predictions on streaming features**: features are recomputed on a short cadence (every minute or 10 minutes) or **event-triggered**. Example: update a driver's rating the instant a user submits a rating, so that feature has essentially zero staleness. The cost is compute, since you recompute far more often.
- The batch/streaming split is blurry. Batch can run on a fast cadence (micro-batch) and streaming can be slow. The useful question is how fresh each feature needs to be, and whether event-driven updates are worth the compute.
- **When don't you need real time?** When the data isn't changing (her example is archaeology). In general, people prefer information sooner, and adoption follows tooling: teams don't go real-time until tools make it cheap, and good tools then drive more adoption.

**Mechanisms: streaming vs the pandas mindset**
- Any data processing setup has three parts: a **schema** (shape and types), a **data source** (CSV, table, stream), and a **compute engine** (pandas, swappable for Dask/Modin/cuDF behind a similar interface; Flink/Spark/Ray for streams).
- In pandas the data is **stationary**: it only changes when you change it. So people ignore the schema (pandas happily lets a float column take a string) and never think about the engine.
- In streaming the data **keeps arriving without your action**, so you need **enforced schemas** (and data contracts / schema evolution) and a stream-capable engine. "It's like a CSV file that keeps getting new rows, and sometimes new columns."
- Conceptually, streaming is just "an event arrives, you process it". The hard part is habit and tooling, not the idea.

**Practitioner lessons (product/company)**
- User feedback: 100 users give you 100 opinions. The hard part is choosing which to act on and turning that into engineering milestones. Balance opinion against data, and decide without waiting forever for more data.
- Every hire is a bet. A startup has a limited "failure budget" for wrong bets, and each one costs far more in a 10-person team than in a 100-person one.
- Learning method: write about a topic *while* learning it, because beginner's eyes spot what's confusing. Brainstorm with experts, circulate notes for feedback, and give it months. Publish before you get bored of it. Use forcing functions (commitments to others) to make yourself learn.

**Decisions & trade-offs / rules of thumb**
- For each feature, decide its freshness requirement separately: daily batch, cadence (minutes), or event-triggered. Pay for streaming only where staleness hurts the prediction.
- Batch prediction is fine when inputs are known in advance and staleness doesn't matter. Use online inference when request-time context matters.
- Treat incoming data as a contract: validate types and schema at ingestion, because nobody is watching each row as it arrives.

**Failure modes & gotchas**
- Confusing low latency with fresh predictions.
- Carrying pandas habits (no schema enforcement) into pipelines whose sources change underneath you.
- Over-engineering real time where daily batch would do. Equally, defaulting to batch out of habit when tooling now makes streaming cheap.

**Relevance to Sam**
- **Travel ranking/recommendation**: precompute static place features (popularity, category, embeddings) in batch, but update fast-changing signals (availability, price from supplier APIs, recent clicks) on a cadence or on events. Track staleness per feature explicitly.
- **Supplier integrations (Viator, RateHawk)**: supplier feeds are "a CSV that keeps changing". Enforce schemas and contracts at ingestion and alert on new or changed columns and type drift, so the matching pipeline doesn't silently degrade.
- On a 16GB laptop, a chunked or micro-batch design (DuckDB/Polars on a cadence) gives most of the freshness benefit without running Flink or Kafka.

