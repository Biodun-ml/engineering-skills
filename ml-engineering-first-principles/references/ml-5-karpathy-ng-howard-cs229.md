# ml-5 — cross-cutting principles

1. **Compute the error numbers before acting.** Human-level (a Bayes-error proxy), train, train-dev, dev and test error. The gaps tell you which problem you have: avoidable bias, variance, distribution mismatch, or dev-set overfitting. Fix that one. (Ng, Nuts and Bolts)
2. **Dev and test must come from the distribution you will deploy on.** The dev set is the problem spec. Put large off-distribution data in train and carve out a train-dev slice to measure mismatch. Never change the test set midway. (Ng)
3. **Use the strongest human level as the reference** (e.g. a team of experts, not a typical person). After you beat it overall, look for slices where you are still worse. (Ng)
4. **In deep learning, bias and variance are no longer a strict trade-off.** A bigger model cuts bias, more data cuts variance, so you almost always have a move. A new architecture is the hard, last-resort lever. (Ng)
5. **End-to-end only pays with lots of paired (x, y) data.** Otherwise, build a pipeline that injects known structure and use learning on the sub-steps (bone detection → age table; detection → planner → steering). (Ng)
6. **Data synthesis is powerful but must match the real distribution.** It looking realistic to a human is not enough (e.g. 20 GTA cars). (Ng)
7. **Transfer learning removes most "big data / big compute" needs.** Fine-tune pretrained weights. Train tabular models and collaborative filtering from scratch, because no pretrained model fits arbitrary tables. (Howard, Lesson 1)
8. **Most practical work is data plumbing.** Answer the DataBlock questions: items, label source, validation split, per-item transforms. Look at the data at every step, delete broken files, and always hold out validation. (Howard)
9. **Every model is the same loop:** forward pass → loss → gradient → small update. Only the forward pass changes as models grow, from lookup table to transformer. Logits → softmax → NLL/cross-entropy is the standard classification head. (Karpathy, makemore; Howard; CS229)
10. **Average negative log-likelihood is the single quality number.** Know its uniform baseline (ln K) and beat a count-based baseline before going bigger. (Karpathy)
11. **Zero probabilities give infinite loss.** Smoothing with fake counts in a count model is the same thing as L2 regularisation toward zero weights in a neural model. Strength controls how uniform the output is. (Karpathy)
12. **Respect broadcasting and check shapes.** Missing keepdim=True silently normalises columns instead of rows. Write out the shape of every tensor and assert invariants such as row sums. (Karpathy; CS229)
13. **Match the head and loss to the label structure.** Independent sigmoids + summed BCE for multi-label; softmax + cross-entropy for exclusive classes (add a "none" class if needed); linear/ReLU + L1/L2 for regression. The label ordering defines what each output neuron learns. (CS229)
14. **Backprop is the chain rule run from the loss backwards, reusing cached intermediate gradients.** The forward equations determine the valid derivative paths. (CS229; Karpathy)
15. **Choose architecture size empirically.** Try several candidates and pick on the validation metric, using intuition about task complexity. Harder tasks need more data and depth. (CS229)
16. **Hidden layers learn hierarchical features** (edges → parts → objects) that would be impractical to hand-engineer. That is the core advantage over classical feature-engineering pipelines. Classical pipelines still win with little data, where feature engineering decides the outcome. (Howard; CS229; Ng)
17. **Simple, cheap baselines first.** Bigram/n-gram counts, logistic regression, a pretrained ResNet fine-tuned for seconds. Scale the same code from one example to the full dataset. (Karpathy; Howard; CS229)
18. **Organisation:** one unified data warehouse (argue about access, not about pooling), a systems team next to the ML team, and practice built on replicating papers plus doing the "dirty work". (Ng)

**For Sam's work specifically:**
- Matching millions of locations: hand-label target-feed pairs for dev/test, use weak/auto labels for train, and keep a train-dev split. Synthesise noisy name variants tuned to real supplier noise. Baseline with logistic regression or GBM over pair features before any deep model.
- Place classification: multi-label sigmoid heads (rare-class weighting), and softmax only for exclusive schemes. Fine-tune pretrained text encoders on hundreds of examples per class.
- Character n-gram NLL works as a cheap junk/garbled-name detector and as a matching feature.
- Collaborative filtering is the starting point for travel recommendations.

---

## Andrej Karpathy — The spelled-out intro to language modeling: building makemore (https://www.youtube.com/watch?v=PaCmpygFfXo)
**A model is just a parameterised way to assign probabilities to data; train it by minimising average negative log-likelihood, and a counting table and a one-layer neural net trained by gradient descent converge to the same model.**

- **Mental models / first principles**
  - Each training string holds many examples. "isabella" teaches: i likely starts a name, s likely follows i, ..., and the name likely ends after the final a. Add explicit start/end tokens so "where does it start/stop" is learned too (Karpathy uses a single "." token at index 0 for both).
  - A bigram model only sees the previous character, so its samples look name-ish but broken (it emits a lone "h" as a name because it has no memory that h was the first letter). Weak models are still useful baselines: compare samples from a uniform model (pure garbage) with the bigram model (clearly better) to confirm the model learned *something*.
  - Likelihood = product of probabilities the model gives to the observed data. Work with log-likelihood (sum of logs, avoids underflow; log is monotonic so the optimum is unchanged), negate it to get a loss where lower is better, and average it so the number is comparable across dataset sizes. Floor is 0.
  - Sanity yardstick: with 27 tokens a uniform guess gives each ~4% (NLL = ln 27 ≈ 3.3). Any probability above 1/K means the model learned signal. The bigram model reaches ~2.45 NLL on 228k bigrams from 32k names.
- **Mechanisms**
  - Counting model: 27x27 int tensor N of bigram counts; normalise rows to probabilities P; sample with torch.multinomial (pass replacement=True; seed a torch.Generator for reproducibility); loop until the end token is sampled.
  - Neural version: inputs are integer indices; one-hot encode (F.one_hot with num_classes set explicitly, then cast to float; it returns int64), multiply by W (27x27), treat outputs as **logits = log-counts**, exponentiate to fake counts, normalise rows = **softmax**. Loss = -probs[arange(n), ys].log().mean(). Backprop with requires_grad=True, zero grads (set to None is cheaper), loss.backward(), then W.data -= lr * W.grad.
  - One-hot times W just selects a row of W, so this "network" is a lookup table; at convergence exp(W) row-normalised equals the count matrix and the loss matches (~2.45-2.47). Learning rate on this tiny convex problem could go to ~50.
  - Batching: matrix multiply of (batch x features) by (features x neurons) computes every neuron's dot product on every example at once.
- **Decisions & trade-offs**
  - Explicit counting is exact and instant but does not scale: context of 10 characters means a table of 27^10 cells. The gradient-based path is flexible: only the forward pass (logits from context) changes as models grow to MLP/RNN/transformer; softmax + NLL + gradient descent stay the same.
  - Smoothing vs regularisation: adding fake counts (e.g. +1) avoids zero probabilities, which would give infinite loss on unseen pairs (e.g. "jq"). The neural equivalent is L2 regularisation (loss + λ·mean(W²)): pulling W toward 0 pushes logits toward equal, i.e. uniform output. Larger λ = more fake counts = smoother, less peaked model.
  - Use NLL/cross-entropy for classification, MSE for regression.
- **Rules of thumb**
  - Reduce the model quality to one number (average NLL) and track it; evaluate per-example log-probs to see which items the model finds surprising.
  - Start with the dumbest model that works on one example (one word, 5 bigrams), then scale to the full dataset without code changes.
  - Precompute normalised probability matrices rather than renormalising in a loop; prefer in-place ops (P /= ...) to avoid new allocations.
  - Read docs and check dtypes: torch.tensor (infers int) vs torch.Tensor (always float32).
- **Failure modes & gotchas**
  - **Broadcasting bug**: P / P.sum(1) (keepdim=False) gives shape (27,), which broadcasting right-aligns into (1,27) and copies down the rows, so it silently normalises *columns*, not rows. It runs with no error and gives wrong output. Always use keepdim=True and check that a row sums to 1. Align shapes from the right; size-1 or missing dims get stretched.
  - Zero-probability events give infinite loss; smooth or regularise.
  - Guessing random weights and keeping the best ("guess and check") is not optimisation; use gradients.
- **Relevance to Sam**
  - Character n-gram counts with smoothing are a cheap, strong baseline for place-name work: scoring how "name-like" a string is, finding junk or garbled GeoNames entries (very high per-character NLL), and making blocking keys or features for fuzzy matching. A 27x27 or 27^3 table fits in memory easily; use sparse dicts for bigger alphabets and Unicode.
  - The same pipeline shape (indices → embedding/lookup → logits → softmax → cross-entropy) is the template for place-type classifiers. Always compare against the uniform baseline (ln K) and a count-based baseline before training anything bigger.
  - The keepdim/broadcasting trap applies equally to numpy/pandas normalisation across millions of rows: assert row sums.

## Andrew Ng (Lex Fridman channel) — Nuts and Bolts of Applying Deep Learning (https://www.youtube.com/watch?v=F1ka6a13S9I)
**Measure human-level, train, train-dev, dev and test error, read the gaps between them, and let those gaps decide your next move. Don't guess.**

- **Mental models / first principles**
  - **Scale drives progress.** Older algorithms (logistic regression, SVMs) plateau as you add data. Bigger neural nets keep improving, so top performance needs *both* a large model and lots of data. With small data the ranking is unclear: whoever engineers features harder often wins, because most of the knowledge comes from hand-engineering.
  - Almost all industry value (2016) came from three buckets: plain fully-connected nets, sequence models (RNN/LSTM/GRU/attention) and image models (CNNs). Unsupervised learning and RL were the "maybe future" bucket.
  - **End-to-end learning** (raw input → rich output: audio→transcript, image→caption, English→French) is powerful, but it needs huge amounts of labelled (x, y) pairs. Without that, a pipeline wins because it lets you inject hand-built knowledge. Example: child's age from a hand X-ray. Use deep learning to find the bones, then a known bone-length→age table, rather than image→age directly. The self-driving modular pipeline (detect cars/pedestrians → planner → steering) beats image→steering at today's data volumes.
  - **Human-level performance ≈ proxy for the Bayes (optimal) error.** Progress is fast up to human level and slows after, because (a) humans are close to the noise floor on perception tasks, and (b) below human level you still have the levers: human labels, error analysis using human insight, and a reference point to separate bias from variance.
- **Mechanisms: the bias/variance recipe**
  - Compute: human-level error, train error, dev error. Gap human→train = **avoidable bias**. Gap train→dev = **variance**. Examples: 1/5/6% means high bias. 1/2/6% means high variance. 1/5/10% means both.
  - Whether 8% train / 10% dev is a bias or a variance problem depends on human level: at 7.5% human it is variance; at 1% it is bias.
  - Flowchart: Is train error high? → bigger model, train longer / better optimiser, (hard) new architecture. Then is dev error high? → more data, regularisation, (hard) new architecture. Repeat.
  - Deep learning weakened the bias–variance *trade-off*: "bigger model" cuts bias without much variance cost (with regularisation), and "more data" cuts variance without hurting bias, so you almost always have a move.
- **Mismatched train/test distributions (production reality)**
  - Example: 50,000 hours of general speech data plus 10 hours from the real product (in-car rear-view mirror). Do *not* make the general data train and the target data dev/test across a split. Put the large general data in train, and split the small in-distribution data into **dev and test from the same distribution**. The dev set is the problem spec, and a dev/test mismatch is like "go north" when you meant San Francisco.
  - Carve a **train-dev** set (e.g. 20 hours) out of the training distribution. Five numbers: human / train / train-dev / dev / test. Gaps: human→train = bias; train→train-dev = variance; train-dev→dev = **distribution mismatch**; dev→test = **overfitting the dev set** (fix: get more dev data).
  - Mismatch fixes: get more training data like the target, data synthesis/augmentation to make train look like test, or (hard) new architecture.
  - In the DL era it is usually better to pool all data into one big model than to build per-product models. Big enough models rarely lose from extra off-distribution data.
- **Decisions & trade-offs**
  - Which "human level" to use: for analysis, use the *best* achievable (a team of experts: 0.5% beats an expert's 0.7%, a typical doctor's 1% and a layperson's 3%), because it bounds Bayes error. Papers often pick a weaker definition to claim "superhuman".
  - Once past human level overall, find **slices** where you are still worse than humans (e.g. accented speech) and apply the levers there.
  - **Data synthesis** works (OCR text pasted on random images; clean speech + car noise added as waveforms; synthetic ungrammatical sentences), but it takes painful tuning (blur, contrast) to match the real distribution. Trap: synthetic data that looks rich to humans can be poor for the model (20 GTA car models look like plenty to a person and are a tiny dataset to a model).
- **Rules of thumb**
  - Always compute those error numbers before choosing an action. "Bigger model / more data" alone solves a lot.
  - Build **one unified data warehouse**. Argue about access rights, not about whether data gets pooled.
  - Pair the AI team with a systems/HPC team, because scale is a systems problem.
  - Product heuristics: if a typical person can do it in under 1 second of thought, it is likely automatable. Also: predicting the next outcome in a frequently repeated event (ad clicks, delivery ETA from restaurant/destination/time).
  - Career: read and replicate many papers (20–50 before original ideas come), *and* do the "dirty work" (data cleaning, debugging, tuning). Do this weekend after weekend.
- **Failure modes**
  - Dev and test from different distributions waste months.
  - Changing the test set midway (it is the spec). Change the training data freely instead.
  - End-to-end projects chosen because x and y exist rather than because there is enough paired data: they make cherry-picked demos that fail at scale.
  - Treating bias/variance as easy: it is simple to state and hard to apply.
- **Relevance to Sam**
  - Place matching/classification: training labels often come from one source (GeoNames/OSM heuristics, supplier data) while production is a new supplier feed (Viator/RateHawk/Coras). Hand-label a few hundred pairs *from the target feed* as dev/test, keep the bulk weak/auto labels as train, and hold out a train-dev slice to tell mismatch apart from overfitting.
  - Get a human-level number: have a person (or two people plus a tie-breaker) label ~200 match/no-match pairs. That is your Bayes-error proxy, and it tells you whether to grow the model or fix the data.
  - Prefer pipelines over end-to-end when labels are scarce: blocking (geohash/country/name n-grams) → feature scoring (name similarity, distance, type) → classifier, using known structure such as distance thresholds.
  - Synthesise training pairs by corrupting names (transliteration, abbreviations like "St."/"Saint", accents, typos), but tune the corruptions to match real supplier noise.
  - Slice-based error analysis (by country, script, place type) matters more than the aggregate F1.

## Jeremy Howard — Practical Deep Learning for Coders: Lesson 1 (https://www.youtube.com/watch?v=8SF_h3xF3cE)
**Start from a pretrained model and fine-tune it on a small, checked dataset with a held-out validation set. Most practical work is getting the data in the right shape, not designing architectures.**

- **Mental models / first principles**
  - An xkcd joke from 2015 says "is this a bird photo?" is nearly impossible. In 2022 it takes about 2 minutes on a laptop: ~200 web images per class (birds vs forests), fine-tune a pretrained ResNet, and get ~100% validation accuracy within seconds of training.
  - **Why deep learning won:** classic ML (e.g. a 2012 Stanford breast-cancer survival project) used thousands of hand-engineered features built over years by teams of experts, fed to logistic regression. Neural nets **learn the features**. Layer 1 finds edges and colour gradients, layer 2 corners, curves and circles, deeper layers repeating patterns, flower edges, text (Zeiler & Fergus visualisation). Depth = composing features.
  - **Arthur Samuel's machine learning loop (1950s):** inputs + weights → model → results → loss. Then a mechanism to adjust weights so the loss gets a little better. Repeat. A neural net is a stack of multiply-add (matrix multiply) layers with ReLU (negatives → 0) between them. With enough layers it can approximate any computable function, so the only hard part is the update step (gradient descent).
  - After training, the loss is no longer needed and the weights are frozen. The model then *is* a program: inputs → results. Deployment is basically `learn.predict(x)` inside normal code.
  - **Transfer learning** is the main reason you rarely need big data or big compute. Start from weights trained on ImageNet (1M+ images, 1,000 classes), then `fine_tune` carefully adjusts them to the differences in your dataset. Jeremy argues the "you need lots of data" myth is pushed by vendors selling storage and compute.
- **Mechanisms (fastai API, built on PyTorch)**
  - **DataBlock** splits what changes between projects into five questions: input type (ImageBlock), output type (CategoryBlock), how to get items (get_image_files), **how to split validation** (RandomSplitter 20%; fastai refuses to train without a validation set), how to label (parent folder name), and item transforms (Resize 192, "squish" vs crop). Then DataLoaders feed **mini-batches** to the GPU. Check them with show_batch.
  - Learner = data + model (ResNet family is fine for most things; timm offers hundreds of alternatives). Vision and NLP use `fine_tune` on pretrained weights. **Tabular** uses `fit_one_cycle` with declared categorical vs continuous columns, because no pretrained model fits arbitrary tables. **Segmentation** uses unet_learner (labels every pixel; decent after ~20s of training). **Collaborative filtering**: (user, item, rating) rows, set y_range slightly wider than the real range (0.5–5.5 for 1–5 ratings) for reasons explained later in the course, metric MSE, under 1 second per epoch.
  - Collaborative filtering defines "similar users" by shared tastes, not demographics. It is the basis of most recommender systems.
- **Decisions & trade-offs**
  - Use high-level libraries: less code means fewer bugs and built-in best practices (fastai's AdamW is a few lines vs a long block in plain PyTorch). Drop down to PyTorch when you need control.
  - Small images (resize to ≤400px when downloading, 192 for training): opening and decoding the image often costs more than the forward pass.
  - Turn other data into images where that helps: sound waveforms/spectrograms, time series, mouse-movement traces (clicks = dots, speed = colour) fed to image classifiers have hit state of the art.
  - What deep learning is good at: things a human (even an expert) can do fairly quickly (looking at a Go board). Weak at long chains of logical reasoning with little data (e.g. "who will win the election").
- **Rules of thumb**
  - **Look at your data at every step**: view samples before bulk download, show_batch, show_results.
  - Verify and delete broken files before training. Corrupt images break training.
  - Always hold out a validation set (random 20% is the default; fastai enforces it).
  - Pin or upgrade library versions at the top of cloud notebooks (`pip install -Uq fastai`) to avoid version-mismatch bugs.
  - Learn top-down ("whole game first"): build and deploy something end to end, then go deeper as needed. Hear the same idea in several forms (book + video).
- **Failure modes**
  - Spending time on architecture tweaks instead of data pipelines. A small set of model families covers most applications.
  - Training on unverified scraped data (broken files, wrong labels).
  - (note: in 2022 the lecture said PyTorch had overtaken TensorFlow in research. That held through 2026.)
- **Relevance to Sam**
  - Tabular is "the most widely used kind of model in industry". For place classification (place type, POI category) and match/no-match on engineered features, use tabular learners or gradient boosting with `fit` from scratch, not fine-tuning.
  - Fine-tune pretrained *text* models (or sentence-embedding models) for place-name/description classification instead of training from scratch. Hundreds of labelled examples per class can be enough.
  - Collaborative filtering (user × item interactions) is the baseline for travel recommendations ("travellers who booked X also booked Y"). Set y_range just beyond the real rating range.
  - The DataBlock questions are a useful checklist for any of Sam's pipelines: what are the items, what is the label source, how is validation split (by country or region to avoid leakage between near-duplicate places), and what transforms run per item.
  - Laptop-friendly: small inputs, pretrained weights, a few epochs. Kaggle and Colab notebooks give free GPUs when the 16GB laptop is not enough.

## Stanford CS229 (Kian Katanforoosh) — Lecture 10: Introduction to Neural Networks (https://www.youtube.com/watch?v=MfIjxPh6Pys)
**A neuron is a linear part plus an activation, and a model is an architecture plus its parameters. Choose the output activation and loss to match the label structure, check the shapes, and train with backprop (the chain rule applied from the loss backwards while reusing stored values).**

- **Mental models / first principles**
  - Deep learning took off because of three things together: compute (GPUs, parallelised code), data (digitisation), and algorithms. Neural nets are flexible, so they keep improving as data grows.
  - **Logistic regression = a one-neuron network.** Flatten a 64×64×3 image to a 12,288-vector x. Then ŷ = σ(wx + b), with w of shape 1×12,288, giving 12,289 parameters. Count parameters as edges (weights) plus nodes (biases). The parameter count grows with input size, which is one reason to move on to other architectures later.
  - **Model = architecture + parameters.** "Shipping a model" means shipping an architecture file and a weights file.
  - The loss is a *proxy*: you can't search directly for the function you want, so you minimise a loss whose minimum gives good parameters. Logistic loss comes from maximum likelihood.
  - **Hidden layers learn features.** Early layers see raw pixels, so the most they can find is local patterns like edges. Later layers combine them into ears, mouths, faces. In the house-price example you *could* hand-wire features (zip + wealth → school quality, zip → walkability, size + bedrooms → family size). Fully connected layers let the network discover such intermediate features itself, often better than a human, at the cost of interpretability ("black box"). "End-to-end" = train only on input and output with no constraint on the middle.
- **Mechanisms**
  - **Multi-label (several classes can be present at once):** k independent sigmoid outputs, label vectors like [1,1,0], loss = sum of the k binary cross-entropies. This is k logistic regressions side by side. Each output's gradient depends only on its own weights. Which neuron "means" cat is decided purely by the label ordering, and with enough data each output becomes robust to co-occurring animals. Weakness: a rare class (one iguana in the zoo) barely trains, because its term mostly pushes toward "absent".
  - **Mutually exclusive classes:** softmax over logits z_k: ŷ_k = e^{z_k} / Σ_j e^{z_j}. Outputs sum to 1 and are coupled (every output depends on every weight through the denominator), so use **softmax cross-entropy** −Σ_k y_k log ŷ_k with one-hot labels. Add an explicit "none" class if the image might contain none of them. Feeding multi-hot labels to a softmax fails (the label sums to 2, the output can't, so it learns 0.5/0.5). Predict with argmax.
  - **Regression** (e.g. a cat's age): change the output activation to linear or **ReLU** (no negative ages) and the loss to L1/L2. Each loss is matched to its task because that makes its shape easier to optimise. Or treat age as a softmax over age buckets.
  - **Forward propagation with shapes:** z[1] = W[1]x + b[1] (W[1] is 3×n), a[1] = σ(z[1]). W[2] is 2×3, W[3] is 1×2, and the biases are 3×1, 2×1, 1×1. Notation: [l] = layer, (i) = example, subscript = neuron. Writing out every shape is the main debugging tool.
  - **Batching / vectorisation:** stack m examples as columns, X (n×m), Z[1] = W[1]X + b[1] (3×m). The parameter count does not change with batch size. Adding a 3×1 bias to a 3×m matrix works through **broadcasting** (numpy copies b across the m columns).
  - **Cost vs loss:** loss is for one example, cost J = (1/m)Σ L_i is for the batch. Differentiation is linear, so compute per-example gradients and average.
  - **SGD vs batch GD:** SGD updates after each example (noisy direction). Batch GD uses the whole batch (accurate direction, more compute per step). Mini-batch sits in between.
  - **Backprop:** update W[l] ← W[l] − α ∂J/∂W[l]. Start at the layer closest to the loss (W[3]): ∂J/∂W[3] = ∂J/∂a[3] · ∂a[3]/∂z[3] · ∂z[3]/∂W[3], where the last factor is a[2]ᵀ. For W[2], reuse ∂J/∂z[3], then multiply by ∂z[3]/∂a[2] · ∂a[2]/∂z[2] · ∂z[2]/∂W[2]. Store the intermediate values instead of recomputing them. The forward equations tell you the valid path through the chain rule. σ' = σ(1−σ).
- **Decisions & trade-offs**
  - Sigmoid-per-class (multi-label) vs softmax (exactly one class): choose based on whether labels can co-occur. Softmax handles imbalance among exclusive classes better because all neurons learn together.
  - Output layer size = number of classes (classification) or 1 (scalar regression).
  - Architecture size (depth, width): nobody knows in advance. Try several (~10) and pick by **validation accuracy**, guided by experience about task complexity. Harder tasks (cat vs no-cat, with many breeds, is harder than day vs night) need more data and depth. Day/night gets harder too if indoor scenes are included.
  - Neural net vs PCA: PCA is unsupervised and finds directions of variance in X. A supervised net learns features that matter for y.
- **Rules of thumb**
  - Write down every tensor shape before coding, and verify that parameter counts match edges + biases.
  - Match the loss to the output: sigmoid + BCE, softmax + CE, linear/ReLU + MSE/L1.
  - The label encoding *defines* what each output neuron learns. Keep the class index mapping fixed and versioned.
- **Failure modes**
  - Multi-hot labels with a softmax head, or one-hot labels where classes actually co-occur.
  - Rare classes in a multi-sigmoid setup barely learn.
  - Shape mismatches hidden by broadcasting (see Karpathy's keepdim bug).
- **Relevance to Sam**
  - Place classification: a place can have several tags (beach + resort + UNESCO), so use multi-label sigmoid heads with per-class BCE. Use softmax only for truly exclusive schemes (e.g. the GeoNames feature class A/P/H/L/T…), with an explicit "other/none" class.
  - Deal with long-tail categories: in a multi-sigmoid setup, rare POI types get few positives. Use class weighting, resampling or hierarchical labels.
  - Match/no-match is a binary logistic problem over pair features (name similarity, distance, type agreement, country). A logistic regression or small MLP on these features is a fast, interpretable baseline that runs comfortably on 16GB, vectorised over millions of candidate pairs in mini-batches.
  - Price or duration prediction (tours, transfers): linear/ReLU output with L1/L2 loss, or bucketed softmax when the distribution is multimodal.

