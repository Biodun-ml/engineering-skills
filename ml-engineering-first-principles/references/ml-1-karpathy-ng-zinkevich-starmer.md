# ml-1 — cross-cutting principles

1. **A model is (representation, cost, optimiser).** Write down the hypothesis form, the single scalar loss that is low exactly when you're right, and how you minimise it. Linear regression, MLPs and boosted trees all fit this frame. (Ng, CS229 L2; Karpathy, micrograd)
2. **Frame the problem as task T, metric P, experience E before building.** If you cannot name the metric and the labelled data, you are not ready to train. (Ng, CS229 L1)
3. **Launch without ML first; a heuristic baseline gives data from the real problem.** Move to ML when the heuristic becomes harder to understand than a model. (Zinkevich, Rules of ML)
4. **Backprop = local derivative x upstream gradient, applied in reverse topological order. Gradients accumulate (+=).** Plus nodes route gradients, multiply nodes swap in the other input, so a zero input means a zero gradient for its weight. (Karpathy, micrograd)
5. **Training loop: forward -> loss -> zero_grad -> backward -> update with p -= lr * grad.** Forgetting zero_grad or flipping the sign are classic silent bugs. (Karpathy; Ng L2 on the sign)
6. **A model can train despite bugs on easy problems.** A toy run converging is not proof of correctness; gradient-check numerically and compare against a reference implementation. (Karpathy, micrograd)
7. **The learning rate is the first knob.** Too high overshoots or diverges, too low crawls. Scale features, start near 0.01, search on a doubling/log scale, and watch the loss curve: a rising loss means lr is too big. Decay lr over time to calm SGD oscillation. (Ng L2; Karpathy)
8. **Batch GD for small data, mini-batch/SGD for big data.** Full-dataset steps are prohibitive at millions of rows; noisy steps reach "good enough" parameters much faster. This also fits streaming chunks through a 16GB machine. (Ng L2; Karpathy demo)
9. **Know when closed form exists.** Least squares is a convex bowl with the normal equation theta = (X^T X)^-1 X^T y. A singular X^T X means duplicated or collinear features, so fix the features. (Ng L2)
10. **Be strategic: diagnose before acting.** Teams lose months on hopeless directions. Profile the ML system (data? model? objective?) before collecting data or buying compute. (Ng L1; Zinkevich on looking at the system weekly)
11. **Most of the work is data and training/serving consistency, not algorithms.** Log features at serving time instead of re-joining IDs offline; audit filters when copying pipelines, because dropped data is invisible. (Zinkevich)
12. **Gate every model export.** Validate new models against production on a fixed check; a stale model is an inconvenience, a broken model is an outage. (Zinkevich)
13. **First objective: simple, observable, attributable. First model: interpretable and calibrated.** Optimising one reasonable proxy usually lifts the others early; tune objectives only when offline gains stop showing up online. (Zinkevich)
14. **Measure seed-to-seed variance.** You cannot claim an improvement smaller than the noise between retrains of the same model. (Zinkevich)
15. **Ranking data only shows what you displayed.** Choose negatives from the true candidate set and handle position effects, or you build feedback loops. (Zinkevich Q&A)
16. **Heuristic patches on top of a model rot.** Constraints and utility weights are fine, but "prediction x fudge factor" hurts once the model improves. (Zinkevich)
17. **Gradient boosting = sequential small trees on residuals, each scaled by eta.** XGBoost splits by gain in a regularised similarity score, prunes with gamma, and shrinks leaves with lambda, which penalises leaves built on few rows. gamma=0 still prunes negative-gain splits. (Starmer, XGBoost Pt1)
18. **Regularisation targets sensitivity to individual observations.** L2 on weights in neural nets and lambda on leaf values in XGBoost do the same job: stop a few points from dictating the model. (Starmer; Karpathy mention of L2)
19. **Abstraction level is a free choice if you know forward + local gradient.** Composite ops such as tanh, or custom similarity functions, can be single autograd nodes. (Karpathy)

---

## Andrej Karpathy — The spelled-out intro to neural networks and backpropagation: building micrograd (https://www.youtube.com/watch?v=VMj-3S1tku0)
**Core idea: a neural net is just a differentiable math expression; backprop is the chain rule applied recursively in reverse topological order, and everything else (tensors, GPUs, PyTorch) is efficiency.**

- **Mental model.** Training = forward pass (inputs + weights -> predictions -> one scalar loss), backward pass (gradient of loss w.r.t. every node), update (nudge weights against the gradient), repeat. Backprop does not care about neural nets at all; it works on any expression graph. Neural nets just happen to be one class of expression where some leaves are data (fixed) and some are weights (tunable). Gradients on data leaves exist but are not used.
- **What a derivative tells you.** Slope = how much the output moves if you nudge an input by a tiny h, normalised by h. Sign tells direction of influence, magnitude tells sensitivity. Example: f(x)=3x^2-4x+5 has slope 14 at x=3, about -22 at x=-3, zero near x=2/3. Numerical estimate (f(x+h)-f(x))/h with h~0.001 works, but making h too tiny breaks due to floating point.
- **Mechanism: the Value node.** Each scalar node stores `data`, `grad` (dLoss/dnode, init 0 = "no effect"), its children (`_prev`), the op that made it (`_op`), and a `_backward` closure that pushes `out.grad` into its children's grads. Leaves have a no-op backward.
- **Local derivatives (memorise these):**
  - `+`: local grad is 1 for both inputs, so a plus node just *routes/copies* the upstream gradient to all children.
  - `*`: grad of a is b's value times upstream grad and vice versa. Consequence: if an input x is 0, the weight multiplying it gets zero gradient (it cannot affect the output right now).
  - `tanh`: 1 - t^2 where t is the tanh output (e.g. t=0.707 gives local grad 0.5).
  - `exp`: local grad = e^x itself (already stored in out.data).
  - `x^k` (constant k): k*x^(k-1). Division = multiply by x^-1; subtraction = add a negation (multiply by -1). Build new ops from existing ones where possible.
- **Chain rule in one line:** global grad of a node = local derivative x upstream (global) grad. Karpathy's intuition: car 2x bicycle, bicycle 4x walker -> car 8x walker; rates multiply.
- **Ordering.** Never call a node's backward until every node that depends on it has pushed its gradient. Topological sort (DFS, append a node only after all its children) then iterate in reverse; seed the output's grad with 1.0.
- **Abstraction level is your choice.** You can implement tanh as one op (with its own derivative) or decompose it into exp, +, /, pow; both give identical forward values and gradients (verified: w1/x1 grads 1 and -1.5, w2/x2 0 and 0.5). The only requirement for a new "lego block" is: forward, plus local gradient. PyTorch's `autograd.Function` works the same way (define forward and backward).
- **Neuron/Layer/MLP.** Neuron = tanh(sum(w_i x_i) + b), weights init uniform in [-1,1], bias = "trigger happiness". Layer = list of independent neurons all connected to the same input. MLP = layers applied in sequence. A 3-4-4-1 MLP has 41 parameters. `parameters()` collects all weights/biases so the optimiser can loop over them (mirrors `nn.Module.parameters()`).
- **Loss.** One scalar that is low exactly when predictions match targets. MSE = sum of (pred - target)^2 (squaring discards sign; abs would also work). For classification in practice: binary cross-entropy or max-margin; for next-token prediction (GPT), cross-entropy. Same machinery at billions of parameters; only the update rule (fancier than plain SGD) and loss differ.
- **Update sign.** Gradient points toward *increasing* loss, so update is `p.data -= lr * p.grad`. Getting the sign wrong maximises loss.
- **PyTorch equivalent.** Tensors are just arrays of these scalars, packed for parallelism. Leaf tensors default to `requires_grad=False` (efficiency, because you rarely want grads on inputs); default dtype is float32 (Karpathy cast to double to match Python). `.item()` pulls a scalar out. Production kernels (CPU/CUDA, bfloat16, complex) are huge and hard to read; micrograd is ~100 lines engine + ~50 lines nn.

**Decisions & trade-offs**
- Learning rate: too low = slow convergence; too high = overshoot into an unrelated region of the loss surface, instability, loss explosion. The gradient is only *local* information, so big steps are a gamble. In the demo 0.01 was too slow, 0.1 "dangerously high", something in between was stable. Learning-rate decay (high early, lower later to fine-tune) is used in the fuller demo.
- Mini-batches: with millions of examples, forward/backward/update on a random subset each step rather than the full dataset.
- Activation choice (tanh vs ReLU vs sigmoid) matters little for small MLPs; tanh used because its derivative is more instructive.
- L2 regularisation mentioned as the knob for overfitting (not covered).

**Rules of thumb / checklist**
- Gradient-check new ops numerically: nudge each input by h and compare (f(x+h)-f(x))/h to the analytic grad. micrograd's tests compare every forward and backward against PyTorch.
- Training loop order: forward -> loss -> **zero grads** -> backward -> update -> repeat. Print loss every step.
- Verify one step reduces the loss before running many.

**Failure modes & gotchas**
- **Overwriting instead of accumulating gradients.** If a variable is used more than once (b = a + a, or a feeds two branches), `grad =` overwrites; must be `grad +=` (multivariate chain rule: contributions sum). a+a should give grad 2, buggy code gives 1.
- **Forgetting zero_grad.** Because grads accumulate, not resetting them each iteration silently sums gradients across steps, acting like an ever-growing step size. Karpathy made this bug on camera; it still "worked" only because the toy problem was trivially easy. Lesson: a net can train *despite* bugs on easy problems and fail on real ones, so passing a toy run is not proof of correctness.
- Storing the closure call (`out._backward = _backward()`) instead of the function itself.
- Python operator plumbing: wrap raw numbers into Value inside `__add__`/`__mul__`; define `__rmul__`/`__radd__` so `2 * a` works.

**Relevance to Sam**
- Understanding gradients at this level is what lets you debug a model that "trains but badly": check loss decreases on a tiny batch, check grads are non-zero (dead inputs/weights show zero grad), and suspect accumulation or sign bugs first.
- Custom differentiable pieces (e.g. a learned similarity score for matching place names/coordinates) only need forward + local derivative to plug into PyTorch.
- On a 16GB laptop, the "everything else is efficiency" point is literal: same math, but batch sizes and tensor packing decide whether it fits in memory.

## Andrew Ng (Stanford Online) — Stanford CS229: Machine Learning Lecture 1 (Autumn 2018) (https://www.youtube.com/watch?v=jGwO_UgTS7I)
**Core idea: ML is a set of tool families (supervised, unsupervised, RL, deep learning) and the scarce skill is not knowing the algorithms but being *strategic* about what to try next.**

(Most of the lecture is course logistics; the substance is the map of the field and the "ML strategy" argument.)

- **Definitions / framing.**
  - Arthur Samuel: giving computers the ability to learn without being explicitly programmed. His checkers program, via self-play, learned which board patterns lead to wins and ended up beating its own author.
  - Tom Mitchell's "well-posed learning problem": a program learns from experience E for task T under performance measure P if P on T improves with E. Practical value: before building anything, write down T, P and E explicitly. For checkers: E = games of self-play, T = playing checkers, P = probability of winning the next game. (For Sam: T = "decide if two place rows are the same place", P = precision/recall on a labelled holdout, E = labelled match pairs.)
- **Supervised learning** (where most of the economic value has come from): given inputs X *with* labels Y, learn a mapping X -> Y that generalises to new X.
  - *Regression* = Y continuous (house size -> price). *Classification* = Y discrete (tumour benign/malignant; or K classes).
  - Choosing model complexity (straight line vs quadratic) is a core recurring decision, made automatically or by hand.
  - Real problems are high-dimensional (many features: tumour size, age, clump thickness, cell-size uniformity, adhesion...) so you cannot just plot them; you rely on algorithms and diagnostics instead of eyeballing.
  - Logistic regression fits a linear decision boundary between classes; SVMs with kernels can effectively use infinite-dimensional feature representations without storing them.
  - ALVINN example (CMU self-driving): ~10 frames/s, grayscale low-res image is X, human steering is Y; a neural net trained with backprop/gradient descent goes from a blurry "don't know" output to a sharp steering choice. A second-level "arbitrator" picked between separate one-lane and two-lane models: a simple form of model routing/ensembling by context.
- **Unsupervised learning**: only X, no Y; find structure. Not just clustering.
  - Clustering (k-means): grouping news articles about the same story (Google News), genetic microarray data, market segmentation of customer databases, communities in social graphs, organising compute clusters, galaxies.
  - ICA (independent components analysis) for the cocktail-party problem: several microphones each record overlapping voices; separate the sources.
  - Learning from unlabelled text (word analogies like man:woman :: king:queen, Tokyo:Japan :: Washington DC:USA).
- **Reinforcement learning**: no one knows the optimal action, so you give a reward signal ("good dog / bad dog") and the algorithm learns to maximise it. Autonomous helicopter, legged robot over obstacles, Atari/Go; Ng is more excited about robotics and logistics optimisation than game stunts.
- **Deep learning** is one fast-moving subset of ML; AI > ML > DL. Know many tools, not just neural nets.

**Decisions & trade-offs: ML strategy as engineering discipline**
- The same algorithm in two teams can give wildly different results. Ng has seen strong teams spend six months on directions that could have been ruled out quickly by looking at the data.
- Typical decision menu on every project: collect more data? which kind of data? try another algorithm? train longer / buy more GPU? change architecture? Picking well among these is the high-leverage skill.
- Analogy 1: deleting every line with a compiler error is not debugging. Analogy 2: experienced engineers run a profiler and fix the actual bottleneck before optimising. Do the same for ML: diagnose (e.g. bias vs variance, error analysis) before acting.
- Goal: move ML from "black magic / tribal knowledge" to a systematic process. This is the theme of his *Machine Learning Yearning* book and the "learning theory / strategy" part of CS229.

**Rules of thumb**
- A learning algorithm almost never works the first time; plan for an iteration loop and make each iteration cheap.
- Before committing months, quickly check whether the approach can work at all (look at data, quick baseline).
- Prefer supervised formulations when you can get labels; that is where value has come from. Use unsupervised methods for exploration, grouping, and when labels are impossible.

**Failure modes**
- Working long on a direction without diagnostics.
- Treating ML as only deep learning.

**Relevance to Sam**
- Place deduplication across millions of GeoNames/OSM rows is naturally framed as supervised pairwise classification (match / no match) once you have labelled pairs; clustering (grouping duplicates into one entity) is the unsupervised follow-on, like Google News grouping stories.
- Place-type classification = multi-class supervised classification; travel ranking can borrow the "arbitrator" idea (route by context, e.g. city vs rural model).
- The strategy point matters most on a 16GB laptop: each experiment is expensive, so diagnose before scaling.

## Andrew Ng (Stanford Online) — Stanford CS229: Linear Regression and Gradient Descent, Lecture 2 (Autumn 2018) (https://www.youtube.com/watch?v=4b4MUYve_U8)
**Core idea: every supervised learner = (hypothesis representation, cost function, optimiser). Linear regression makes all three explicit; gradient descent (batch / stochastic / mini-batch) is the optimiser you will reuse everywhere, and the normal equation is a closed-form shortcut that only works for least squares.**

- **Workflow framing.** Training set -> learning algorithm -> hypothesis h. h takes a new input and outputs a prediction. The design choices that recur in *every* algorithm: what is the dataset, how is h represented, what cost is minimised, how is it optimised.
- **Hypothesis.** h_theta(x) = sum_{j=0..n} theta_j x_j with a dummy feature x_0 = 1 so the intercept folds into the vector (technically affine). Notation: m = number of training examples (rows), n = number of features, x^(i), y^(i) = i-th example (superscript is an index, not a power), theta and x are (n+1)-dimensional. Example data: 49 Portland houses (size in sq ft, bedrooms -> price in $1000s; 2104 sq ft -> 400).
- **Cost.** Ordinary least squares: J(theta) = 1/2 * sum_i (h(x^(i)) - y^(i))^2. The 1/2 only simplifies the derivative. Squared error corresponds to assuming Gaussian noise (justified later via generalised linear models); other losses (absolute, 4th power) are possible.
- **Gradient descent.** Start at theta = 0 (or random), repeatedly step in the direction of steepest descent: theta_j := theta_j - alpha * dJ/dtheta_j, for all j simultaneously. For least squares, dJ/dtheta_j = sum_i (h(x^(i)) - y^(i)) * x_j^(i): the error times the feature. Derived via the chain rule; for one example the 2 from the square cancels the 1/2.
  - Subtract, not add: at theta=1 on J=theta^2 the derivative is positive and you want theta to shrink.
  - In general, gradient descent can land in different local minima depending on initialisation. But least-squares J is a quadratic "bowl" (convex): the only local optimum is the global one, and contours are ellipses; the steepest-descent direction is perpendicular to the contours.
- **Batch vs stochastic vs mini-batch.**
  - *Batch GD*: each step sums over all m examples. Clean convergence, but each step must scan the whole dataset (painful at millions/hundreds of millions of rows or terabytes; hundreds-to-thousands of steps = hundreds-to-thousands of full scans).
  - *SGD*: update after each single example. Noisy, wandering path that heads toward the minimum on average and never fully settles (keeps oscillating), but makes progress far faster on big data. The parameters it ends at are usually "good enough" for prediction.
  - *Mini-batch* (e.g. 100 examples per step) is what is used most in practice (covered in CS230).
  - Reduce oscillation by **decaying the learning rate** over time (smaller steps, bounces in a smaller region near the optimum). Ng says decay is used far more often than switching from SGD to batch at the end.
  - SGD's learning rate is *not* 1/m of batch's; usually much bigger, otherwise it is as slow as batch.
- **Normal equation (closed form, linear regression only).** Stack examples as rows in the design matrix X (m x (n+1)), labels into vector y. Then J = 1/2 (X theta - y)^T (X theta - y); set the gradient X^T X theta - X^T y to 0 and get X^T X theta = X^T y, so theta = (X^T X)^{-1} X^T y. One step to the global optimum, no learning rate. Matrix-calculus tools used: gradient w.r.t. a vector/matrix has the same shape as it; trace facts tr(A)=tr(A^T), d/dA tr(AB) = B^T, tr(AB)=tr(BA), cyclic tr(ABC)=tr(CAB).
  - If X^T X is not invertible, features are linearly dependent (often the same feature twice). Pseudo-inverse "works", but the real fix is to find and remove the duplicated feature.

**Decisions & trade-offs**
- Small dataset (hundreds to thousands of rows) and batch GD is cheap: use batch, "one less thing to fiddle with". Large data: SGD/mini-batch.
- Linear regression exactly: consider normal equation (note: (X^T X) inversion cost grows ~n^3 in features, so it suits modest n; not stated in lecture but standard). Everything else (GLMs, neural nets): gradient descent.

**Rules of thumb**
- Scale features to roughly [-1, 1] or [0, 1]; then start alpha around 0.01 and search on an exponential/doubling scale (0.01, 0.02, 0.04, 0.08...) to see which drives J down fastest. Choosing alpha is empirical.
- Plot J(theta) over iterations. If it *increases*, alpha is too big. When it stops going down, stop (simple convergence check for SGD).
- Initialisation does not matter much for convex problems; it matters a lot for non-linear models like neural nets, where convergence debugging is "more acute".

**Failure modes & gotchas**
- alpha too large: overshoot past the minimum, divergence. Too small: very slow.
- Unscaled features make the bowl elongated and gradient descent zig-zag (implied by the scaling advice).
- Redundant/collinear features break the normal equation.

**Relevance to Sam**
- With millions of location rows on 16GB, batch GD over all rows per step is wasteful; mini-batch/SGD with chunked streaming fits the memory constraint naturally (each step only needs one chunk in RAM). Libraries like scikit-learn's `SGDRegressor/SGDClassifier.partial_fit` are this algorithm.
- Duplicate or derived columns in a feature table (e.g. lat/lon plus a projected coordinate) can cause singular matrices in linear models; dedupe features as well as rows.
- Always plot the loss curve; it is the cheapest diagnostic you have.

## Martin Zinkevich (Google Research, via rzykov) — Rules of Machine Learning (https://www.youtube.com/watch?v=wvB4B_jb5Oc)
**Core idea: to build a great ML system, be a great engineer first. Launch simple, get the pipeline and metrics right, guard every model export, and treat training-serving skew and dropped data as the main dangers. Algorithm cleverness comes last.**

(A talk summarising Google's "Rules of ML" document, which has many more rules than the talk covers. Bias toward web ranking/recommendation.)

- **Guiding principle.** Good ML engineering is ordinary engineering discipline: keep it simple, document, test, fix "the inevitable sign mistake", peer review, version control, monitor, and keep a tight launch -> iterate -> test cycle. Reliability, technical debt, and fast-but-stable progress are intertwined.
- **Before ML.**
  - *Don't be afraid to launch without ML.* ML is not a magic hat; it combines an objective with past data. For ranking, start with a heuristic (recency, stars, popularity), gather data *from the actual problem*, then learn on it.
  - *Make metric design and implementation a priority.* Instrument lots of metrics before you have a model. Doctor analogy: health is many numbers, not one; treat the product as a patient you keep healthy long-term.
  - *ML as a way to reduce technical debt.* When the heuristic grows into a tangle ("installs times ratings to the 23rd power..."), it is harder to understand than a model. Saying *what* you want formally ("maximise P(install | search)") is clearer and survives staff turnover better than hand-coding *how*. Compiler analogy: hand-coded policy is machine code; an objective plus data is the high-level language, and it re-adapts automatically when data changes. Feature engineering is where the work goes.
- **Reality of the job.** Most time goes into gathering solid data and making sure the deployed model sees what it saw at training, not into the learning algorithm.
- **Dropped data when copying pipelines** (real Google cases):
  - Google+ copied the "What's Hot" pipeline for general ranking; it silently kept a filter that dropped older posts (sensible for freshness, wrong for general ranking).
  - A team logged only examples the user saw, then tried to predict *whether* something was shown: the negatives they needed were exactly the dropped ones.
  - Google Play reused code across books/movies/apps and accidentally copied data along with it.
  - Dropped data is by definition invisible, so audit filters whenever you reuse a pipeline.
- **Gatekeep model exports (the "most basic" rule).** In continuous training, a broken model pushed to users = middle-of-night page. A stale model = a mild inconvenience. So put a validator before export: does the model run, is it at least as good as production on basic checks? If not, skip it silently; alert only if several in a row fail and staleness grows.
- **ML failures are silent.** Unlike a lost email (the recipient notices), a bad feature or bad training example just makes the model a bit worse and is very hard to pin to a specific example/feature. Monitor training data and serving-time feature values explicitly.
- **Objectives.**
  - *Don't overthink which objective to optimise first.* Clicks vs time-spent: optimising one usually lifts the other early on because both proxy "system got better". Worry about trade-offs only once basic features are in and you are near the Pareto frontier.
  - *First objective: simple, observable, attributable.* Clicks/purchases rather than "is the user happy" or "DAU next week". Attributable = you know the click happened because you showed it.
  - *Launch decisions are proxies for long-term goals* (product alive and healthy in ~5 years). That is why many A/B metrics (downloads/day, revenue/day) can veto a launch that improved offline accuracy; humans connect long-term reward to those metrics.
  - When an offline metric improves but online metrics or side-by-sides get worse, *stop adding features and fix the objective*; it no longer represents what you care about.
- **Start with an interpretable model** whose mean prediction matches the mean label (calibrated, e.g. logistic regression), not "a seven-layer neural factor model with a ranking loss". Easier to debug.
- **Log features at scoring time.** Join features + query at serving time and log the whole instance, later join with labels. Do *not* log only IDs and re-join features offline to save storage: it causes skew (features change between serving and join), more code, and fixing it gave some teams dramatic improvements.
- **Under-practised but valuable:**
  - *Weekly manual look at the system.* Poke around pipelines and outputs; Zinkevich often spots something plainly wrong just by looking.
  - *Learning stability.* Retrain with several random seeds and look at the *distribution* of performance, not just the mean. If daily/weekly retrains vary a lot, users may get a bad day, and you cannot detect any improvement smaller than seed-to-seed noise.
  - *Business logic and ML must play nicely.* Ranking by perfect P(click) is optimal; hard constraints (contractual positions) and multiplying by a utility (value per click) are also fine. But "prediction x heuristic fudge" or "+ heuristic" that patches a current model flaw becomes harmful once the model improves; document and revisit them.
- **Q&A additions.**
  - Ranking data has inherent skew: you only observe what you showed. Gmail autocomplete example: the 5 shown-but-not-clicked names are not the only negatives; the whole contact list is. Choose negatives that reflect the real candidate set.
  - Avoid feedback loops from positional features (position influences clicks; handle position explicitly).
  - Human relevance ratings are too few to train on but useful as an evaluation metric; give raters precise guidelines, which evolve. They often disagree with engagement metrics.
  - Stability analysis for deep models was "under active development" (2017).

**Rules of thumb / checklist**
1. Heuristic baseline first; collect data from the real problem.
2. Define and instrument many metrics before modelling.
3. Simple, observable, attributable first objective; interpretable calibrated first model.
4. Log the exact features used at serving time.
5. Audit filters when copying pipelines.
6. Validate every model before export; fail closed to the previous model.
7. Look at the system weekly by hand.
8. Run multiple seeds; know your noise floor before claiming wins.
9. When offline gains don't show online, tune the objective, not features.

**Relevance to Sam**
- Travel ranking/recommendation (Drive Button trip buttons, supplier product ranking): start with heuristics (popularity, rating, distance), log every impression with its features, use clicks/bookings as the first attributable label, and beware that you only see feedback on items you showed.
- Location matching pipelines: the "dropped data" warning maps to dedup filters (e.g. drop rows without coordinates or with certain feature codes) silently removing true matches; count rows in and out at every filter.
- Automated retrains (e.g. place classifier) need a gate: compare against the current model on a fixed holdout before replacing it.

## Josh Starmer (StatQuest) — XGBoost Part 1 (of 4): Regression (https://www.youtube.com/watch?v=OtD8wVaFm6E)
**Core idea: XGBoost is gradient boosting where each tree is fitted to residuals using a regularised "similarity score". Splits maximise gain, gamma prunes low-gain branches, lambda shrinks both scores and leaf outputs, and eta scales each tree's contribution.**

(Assumes you already know plain gradient boosting and regularisation. Toy data: 4 points, drug dosage -> effectiveness, residuals -10.5, 6.5, 7.5, -7.5.)

- **Boosting loop.** Start with an initial prediction (default **0.5** for both regression and classification). Compute residuals (observed - predicted). Fit a tree to the residuals. New prediction = old prediction + eta x tree output. Recompute residuals, fit another tree, and repeat until residuals are tiny or you hit the max number of trees. Each tree is a small corrective step.
- **Similarity score (regression, squared-error loss):** (sum of residuals)^2 / (number of residuals + lambda). The residuals are summed *before* squaring, so residuals with opposite signs cancel. A node full of mixed residuals scores low; a node of similar residuals (or a single residual) scores high. Example: root with all 4 residuals = 4 (with lambda=0).
- **Gain of a split** = Similarity_left + Similarity_right - Similarity_parent. Try every threshold: the midpoints between adjacent sorted feature values. Keep the one with the highest gain. Example: dosage<15 gain 120.33 beats dosage<22.5 (4) and dosage<30 (56.33). Second level: dosage<30 gain 140.17 beats dosage<22.5 (28.17). (note: on large data XGBoost uses approximate/quantile-sketch or histogram candidate splits rather than every midpoint; that is covered in later parts, not this one.)
- **Depth.** Demo limited to 2 levels; XGBoost default max depth is **6**.
- **Pruning with gamma (tree-complexity parameter).** Work bottom-up: if gain - gamma < 0, remove the branch; if positive, keep it. Once a lower branch survives, its parent is kept even when the parent's own gain < gamma. Example: gamma=130 keeps everything (140.17 > 130). gamma=150 removes the lower branch and then the root (120.33 < 150), leaving only the initial prediction: "extreme pruning".
- **Lambda (L2 regularisation on leaf weights).** Adds to the denominator of both similarity and output value.
  - Similarity shrinks *more for nodes with few residuals*: with lambda=1 the single-residual leaf drops 50% (110.25 -> 55.12), the 3-residual leaf drops 25% (14.08 -> 10.56), the 4-residual root drops 20% (4 -> 3.2). So lambda penalises splits that isolate a few points.
  - Gains shrink (120.33 -> 66), so the same gamma prunes more aggressively.
  - Gain can go **negative** with lambda>0 (example: -16.06). Therefore **gamma=0 does not turn off pruning**; a negative-gain branch is pruned even at gamma=0. That is lambda doing its job against overfitting.
- **Leaf output value** = (sum of residuals) / (number of residuals + lambda) (like similarity, but the sum is not squared). With lambda=0 it is just the mean residual in the leaf (e.g. 7 and -7.5); with lambda=1 a single residual -10.5 becomes -5.25. So lambda reduces sensitivity to individual observations. Default lambda in this video's setup is 0. (note: in the current xgboost library the default `reg_lambda` is 1, not 0; check your version.)
- **Learning rate eta** (default **0.3**). Example: dosage=10 goes 0.5 + 0.3 x (-10.5) = -2.65; its residual shrinks, i.e. a small step in the right direction for every point.

**Decisions & trade-offs**
- gamma: higher = simpler trees (prunes branches whose gain is below it). Good for noisy data.
- lambda: higher = more conservative leaf values and more pruning, especially of leaves built on few rows. Protects against one outlier driving a leaf.
- eta: smaller = each tree contributes less, so you need more trees but get better generalisation (standard boosting trade-off).
- max_depth: deeper trees capture interactions but overfit faster.

**Rules of thumb**
- Read XGBoost hyperparameters as: eta (step size), max_depth/gamma (tree complexity), lambda (leaf shrinkage).
- If a tree hinges on leaves containing one or two rows, raise lambda (and/or a minimum-child-weight constraint) before anything else.
- Monitor residuals shrinking per round; stop when they plateau (use early stopping on a validation set in practice).

**Failure modes & gotchas**
- Thinking gamma=0 means "no pruning": with lambda>0 negative gains are still pruned.
- Forgetting the 0.5 base prediction when hand-checking outputs.
- Default values differ between explanations and library versions; set them explicitly.

**Relevance to Sam**
- Gradient-boosted trees (XGBoost/LightGBM) are the default first serious model for tabular problems like place-match classification (features: name similarity, distance in km, admin-code agreement, feature-class match, population ratio) and place-type classification. They handle mixed-scale features without normalisation and train fast on a laptop.
- For millions of candidate pairs on 16GB, use histogram-based training (`tree_method="hist"`) and float32 features; the split-search intuition here (sum residuals per candidate threshold) is why histogram binning is so cheap.
- Lambda/gamma are the levers against overfitting to rare, noisy match pairs.
