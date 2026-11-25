# math-1 — cross-cutting principles

1. **"Vector" means "can add and scale".** Arrows, lists of numbers, matrices and functions are all vectors, and linear algebra's power lies in translating between the geometric and numeric views. Every embedding or feature row is a vector. *(3B1B Vectors Ch.1; Strang Lec 11)*
2. **A matrix is a transformation, fully defined by where the basis vectors land (its columns).** Matrix × vector = the columns weighted by the entries. *(3B1B Composition Ch.4)*
3. **Matrix multiplication = composition, read right to left.** Order matters (AB ≠ BA) and grouping doesn't (associativity). Pre-multiply chains of linear transforms into one matrix. Without nonlinearities, stacked NN layers collapse into one matrix. *(3B1B Composition Ch.4)*
4. **Rank-1 matrices (u·vᵀ) are the building blocks.** A rank-r matrix is a sum of r of them. This is the seed of SVD, PCA, low-rank compression and matrix-factorisation recommenders. Rank-r matrices and set unions are *not* subspaces; use sums. *(Strang Lec 11)*
5. **Dimension = number of free parameters, and dim N(A) = n − r.** The easiest way to show a set is a subspace is to exhibit it as a null space (e.g. "components sum to 0" = the null space of [1 1 1 1], which describes centred data). *(Strang Lec 11)*
6. **The determinant is the area/volume scale factor.** det = 0 means space collapses: no unique solution, and the matrix isn't invertible. Cramer's rule falls out geometrically, but use elimination or a library solver in practice. *(3B1B Cramer Ch.12; Eigen Ch.14)*
7. **Orthonormal maps preserve dot products, lengths and angles. Their inverse is the transpose.** Rotating embeddings keeps cosine/Euclidean matching scores intact; general projections do not. *(3B1B Cramer Ch.12)*
8. **Eigenvectors are the directions a map only stretches, with Av = λv found via det(A − λI) = 0.** In an eigenbasis the matrix is diagonal and powers are trivial. Rotations have no real eigenvectors and shears too few. PCA = eigenvectors of the covariance matrix, and repeated application converges to the dominant eigenvector (PageRank, Markov chains). *(3B1B Eigen Ch.14)*
9. **A derivative is the limit of Δf/Δx as the step → 0: the best local linear approximation, not an "instantaneous change".** In code it's a finite difference, which is how you gradient-check a custom loss. *(3B1B Derivative Ch.2)*
10. **Taylor series turn derivatives at one point into local approximations.** The nth coefficient is f⁽ⁿ⁾(a)/n!, and adding a term never disturbs earlier ones. Valid only within a radius of convergence (ln x diverges beyond x = 2 when expanded at 1). Gradient descent is a first-order Taylor bet, which is why steps must stay small. *(3B1B Taylor Ch.11)*
11. **Learning = minimising an average cost over data by stepping along −∇C.** The gradient's sign says which way to nudge each parameter; its magnitude says how much each parameter matters. The cost must be smooth, which is why activations are continuous. *(3B1B Gradient descent DL Ch.2)*
12. **Low training cost ≠ learned structure.** Nets can memorise shuffled labels, land in uninterpretable local minima, and be confidently wrong on noise. Always validate on held-out data and don't trust confidence on out-of-distribution inputs (e.g. unfamiliar place-name scripts). *(3B1B Gradient descent DL Ch.2)*
13. **Backprop = chain rule: multiply local sensitivities along a path, sum across paths, reuse upstream results.** Weight gradient = input activation × σ'(z) × upstream error. Products of many small or large factors explain vanishing/exploding gradients. *(3B1B Backprop calculus DL Ch.4)*
14. **Before integrating, recognise structure.** Z1 − Z2 is N(0,2), which turns a 2D integral into 1D. Independent normals add with variances adding, even for differences. Use a distribution's generative story to get marginals and conditionals for free. *(Blitzstein Stat 110 Lec 20)*
15. **The multinomial models counts in k buckets.** Each count is Binomial(n, pj), lumping categories gives another multinomial, and conditioning on one count renormalises the rest by 1 − p1. It fits bookings per supplier, place feature classes, and match-type buckets. *(Blitzstein Stat 110 Lec 20)*
16. **Ratios of noisy quantities can be Cauchy: no mean, no variance, and averages never converge** (the mean of n Cauchys is still Cauchy). Guard denominators near zero and prefer medians or robust statistics for ratio metrics. *(Blitzstein Stat 110 Lec 20)*
17. **Approximations are local, so know the radius.** This holds for Taylor polynomials, flat-earth distance approximations, linearised gradient steps, and models outside their training distribution. *(3B1B Taylor Ch.11; Gradient descent DL Ch.2)*
18. **Think geometrically before memorising formulas.** Associativity, AB ≠ BA, Cramer's rule and eigen-decomposition all become obvious from the picture, and that makes them usable when debugging real systems. *(3B1B Composition, Cramer, Eigen)*

---

## Joe Blitzstein (Harvard Stat 110) — Lecture 20: Multinomial and Cauchy (https://www.youtube.com/watch?v=xiVWNkQUqKk)
**Before you integrate, look for structure: the story of a distribution gives you most answers for free. The multinomial is "counting things into k buckets"; the Cauchy is "a ratio of noise", and it has no mean at all.**

- **Mental model: simplify before computing.** Want E|Z1 - Z2| for two iid standard normals? You could write a 2D LOTUS double integral. Blitzstein says no: first recognise that Z1 - Z2 is itself normal, N(0, 2). Variances add even for a difference, because -Z2 has the same variance as Z2. So the problem becomes sqrt(2)·E|Z|, a 1D integral that equals sqrt(2/pi). Result: E|Z1 - Z2| = sqrt(2)·sqrt(2/pi) = 2/sqrt(pi) ≈ 1.13.
- **Mechanism: sum of independent normals is normal (MGF proof).** The MGF of N(mu, sigma²) is exp(mu·t + sigma²t²/2). For independent X and Y the MGF of X+Y is the product of the two, and exponents add, so you get N(mu1+mu2, sigma1²+sigma2²). An MGF pins down a distribution, so the proof is done. Independence is required. Location-scale trick: any normal is mu + sigma·Z, so you only ever need to think about the standard one.
- **Multinomial story.** You have n objects, and each one independently falls into exactly one of k categories with probabilities p = (p1..pk). Each pj ≥ 0 and they sum to 1. X = (X1..Xk) holds the counts per category. Binomial is the case k = 2 (success/failure).
  - Joint PMF: P(X1=n1,...,Xk=nk) = n!/(n1!...nk!) · p1^n1 ··· pk^nk when n1+...+nk = n, otherwise 0. The derivation: take the probability of one specific ordered sequence, then multiply by the number of rearrangements. That count works like the letters of "Mississippi": n! divided by the factorial of each repeated group.
  - **Marginal:** Xj ~ Bin(n, pj). Each object is either in category j or not, so you need no summing over the joint. That gives E[Xj] = n·pj and Var(Xj) = n·pj(1-pj) immediately.
  - **Lumping property:** merging categories gives another multinomial with the probabilities added. In his example a country has 10 parties and 2 are dominant: (X1, X2, X3+...+X10) ~ Mult3(n, (p1, p2, p3+...+p10)). The only requirement is that each object still lands in exactly one category.
  - **Conditioning:** given X1 = n1, the remaining counts are Mult(k-1) with n - n1 objects and renormalised probabilities pj' = pj/(1 - p1). **Gotcha:** writing just p2..pk is a common mistake. Those don't sum to 1.
- **Cauchy distribution.** T = X/Y with X, Y iid N(0,1). PDF is f(t) = 1/(pi(1+t²)) for all real t, and the CDF involves arctan. It is the "evil" distribution:
  - It has **no mean and no variance**.
  - The scariest property: **the average of n iid Cauchys is still Cauchy**, with exactly the same distribution. Averaging more data does not converge to anything, so the law of large numbers does not apply. Naive averaging is the wrong tool for such data; robust alternatives exist (note: e.g. the median, which he hints at but doesn't name).
- **Mechanism: deriving the Cauchy PDF.** This is a common interview question and tests joint-PDF fluency.
  1. Use symmetry to write P(X/Y ≤ t) = P(X ≤ t|Y|). That avoids flipping the inequality when Y < 0.
  2. Write it as a double integral of the joint PDF (product of marginals by independence). The inner integral is just Phi(t|y), the normal CDF.
  3. You can't integrate Phi in closed form. But you want the PDF, so differentiate with respect to t **under the integral sign** (allowed for well-behaved integrands). The chain rule pulls out a factor y, and Phi' = phi.
  4. What's left is ∫0..∞ y·exp(-(1+t²)y²/2) dy. Substitute u = (1+t²)y²/2 and you get 1/(pi(1+t²)).
  - Alternative method: the law of total probability, conditioning on Y = y. You get ∫ Phi(t|y|)·phi(y) dy, the same integral. Independence is what lets you drop the condition after plugging in y.
- **Rules of thumb:**
  - Before integrating, ask whether this is a known distribution in disguise.
  - Use the generative "story" (what physically happens) to get marginals and conditionals instead of summing joint PMFs.
  - Check that a derived PDF integrates to 1.
  - When a probability involves dividing by a random variable, handle its sign: use symmetry or split cases.
- **Failure modes:** Forgetting a factor in a long integral. He drops an exp(-y²/2) term live and a student catches it, which is a reminder to re-check each factor. Assuming variances subtract for differences. Assuming averages converge for heavy-tailed data.
- **For Sam's work:**
  - Category counts are multinomial: bookings by supplier, places by feature class in GeoNames, matches by match-type bucket. Each per-category count is binomial, which gives quick error bars: sd = sqrt(n·p(1-p)).
  - Lumping "other" categories is legitimate and keeps the model simple. Conditioning on a known category means renormalising the rest.
  - The Cauchy warning matters for **ratios of noisy quantities**: price ratios, conversion rates with small denominators, distance ratios in matching scores. If a denominator can be near zero, means blow up and averaging won't stabilise them. Use medians or trimmed means, or guard the denominator.
  - Sums of independent normal errors stay normal with variances adding. That's useful when errors stack up through pipeline stages.

## Gilbert Strang (MIT 18.06) — Lecture 11: Matrix Spaces; Rank 1; Small World Graphs (https://www.youtube.com/watch?v=2IdtqGM6KWU)
**Linear algebra applies to anything you can add and scale (matrices, functions, solutions of equations). Rank-1 matrices (a column times a row) are the building blocks of every matrix.**

- **Mental model: "vector" means "can add and scale", not "column of numbers".** The set M of all 3x3 matrices is a vector space. You only need addition and scalar multiplication; matrix multiplication is not part of the vector-space picture. M is essentially R^9 with the 9 numbers written in a square.
- **Subspaces of M and their dimensions** (count the free parameters):
  - All 3x3 matrices: dim 9. The standard basis is 9 matrices, each with a single 1.
  - Symmetric S: dim 6 (3 diagonal + 3 above-diagonal entries; the rest are forced).
  - Upper triangular U: dim 6.
  - S ∩ U is the diagonal matrices: dim 3.
  - **Gotcha:** a basis for the big space does not generally contain a basis for a subspace. Only 3 of the 9 standard basis matrices are symmetric, so S needs new basis elements such as matrices with 1s in both (i,j) and (j,i). For U the big basis happens to contain one (an accident).
- **Union vs sum.** S ∪ U is **not** a subspace. It's like two lines in a plane, and adding one vector from each can land outside both. The right operation is the **sum** S + U = {s + u}, all combinations. Here S + U is all 3x3 matrices (dim 9).
  - **Formula:** dim S + dim U = dim(S ∩ U) + dim(S + U). In this example 6 + 6 = 3 + 9.
- **Functions as vectors.** The solutions of y'' + y = 0 are exactly y = c1·cos x + c2·sin x. That is a 2-dim vector space, and it is really a **null space** (of a differential operator). {cos, sin} is a basis; {e^{ix}, e^{-ix}} is another. The dimension equals the order of the equation. Strang's framing: a linear-ODE course is mostly "find a basis for the solution space".
- **Rank-1 matrices.** Example: rows (1,4,5) and (2,8,10). The second row is a multiple of the first, so the rank is 1. Every rank-1 matrix is A = u·vᵀ, a column times a row: here (1,2)ᵀ·(1,4,5).
  - The column space is span(u) and the row space is span(v). Each has dimension 1, consistent with dim col space = dim row space = rank.
  - **Building blocks:** a matrix of rank r is a sum of r rank-1 matrices. For example, a 5x17 matrix of rank 4 is the sum of 4 terms u_i·v_iᵀ. (note: this is the seed of the SVD and of low-rank approximation.)
- **Rank-r matrices do not form a subspace.** Two rank-1 matrices usually add to rank 2. In general rank(A+B) ≤ rank(A) + rank(B), and the rank is also capped at min(m, n). For 5x17 matrices the rank is at most 5.
- **Worked example: the four fundamental subspaces of A = [1 1 1 1].** S = {v in R^4 : v1+v2+v3+v4 = 0} is a subspace because it is the **null space of A**.
  - Rank r = 1, so dim N(A) = n - r = 4 - 1 = 3.
  - Basis via special solutions: the pivot is column 1 and the free variables are 2, 3, 4. Set one free variable to 1 at a time to get (-1,1,0,0), (-1,0,1,0), (-1,0,0,1).
  - Row space: dim 1 in R^4. Column space: all of R^1. N(Aᵀ) = {0}, dim 0, with the empty set as its basis.
  - Check: row + null = 1 + 3 = 4 = n, and column + left null = 1 + 0 = 1 = m.
- **Small-world graphs (teaser for the next lecture).** A graph is nodes plus edges, for example people plus friendships, or websites plus links. Its structure is captured by an incidence matrix (5 nodes, 6 edges → a 5x6 matrix). "Six degrees of separation": a few random shortcut edges shrink typical distances dramatically. Modelling the web graph is a big motivation.
- **Rules of thumb:**
  - Dimension = number of free parameters you can choose.
  - To show a set is a subspace, check closure under + and scaling, or better, show it is the null space (or column space) of some matrix. Then everything follows from the rank: dim N(A) = n - r and dim N(Aᵀ) = m - r.
  - Use sum, not union, to combine subspaces.
- **For Sam's work:**
  - **Rank-1 / low-rank thinking is the core of embeddings and recommenders.** A user x item matrix approximated as a sum of a few u·vᵀ terms is matrix factorisation. Truncated SVD and PCA keep the top-r rank-1 pieces, which compresses millions of rows into r numbers per row and fits a 16GB laptop.
  - "Components sum to zero" describes centred features. Mean-centring data (as PCA requires) projects it onto exactly this kind of n-1 dimensional null space.
  - Small-world graphs connect to location/transit networks. With a few hub shortcuts (airports, ferry hubs), path lengths in a GTFS stop graph collapse, which matters for route search and connectivity metrics.

## Grant Sanderson (3Blue1Brown) — Gradient descent, how neural networks learn | Deep Learning Ch. 2 (https://www.youtube.com/watch?v=IHZwWFHWa-w)
**"Learning" means minimising a cost function: repeatedly nudge all the parameters a small step along the negative gradient.**

- **Setup (MNIST).** Input: a 28x28 grid, so 784 grayscale values in [0,1]. Two hidden layers of 16 neurons (an arbitrary choice). Output: 10 neurons, and the prediction is the brightest one. Each neuron applies sigmoid or ReLU to (weighted sum + bias). In total about 13,000 weights and biases.
- **Three nested functions (the key mental model):**
  1. **The network**: 784 numbers in, 10 numbers out, parameterised by the 13k weights.
  2. **The cost**: 13k parameters in, 1 number out. For one example, sum the squared differences between the output and the one-hot target (1 for the true digit, 0 elsewhere). Then **average over all training examples**. Small cost = confidently right; large = clueless.
  3. **The gradient of the cost**: 13k numbers telling you which nudge to each parameter lowers the cost fastest.
- **Mechanism: gradient descent.**
  - In 1D: check the slope. Step left if it's positive, right if it's negative, and repeat. You end up in a local minimum, like a ball rolling downhill.
  - In 2D and beyond: the gradient points in the direction of steepest *ascent*, and its length is how steep that slope is. Step along **-∇C**, repeat.
  - For 13k dimensions, stack all the parameters into one giant column vector. -∇C is a direction in that space.
  - **Step size ∝ gradient**: steps shrink automatically as the surface flattens near the minimum, which reduces overshooting.
- **Second reading of the gradient: relative importance.** The sign of each component says whether to nudge that parameter up or down. The magnitude says how much that parameter matters right now. A gradient of (3, 1) means nudging x has 3x the "bang for buck" of nudging y, at least locally. So the gradient tells you which weights matter most for the current data.
- **Why artificial neurons are continuous.** Gradient descent needs a **smooth** cost to take small downhill steps. Binary on/off neurons (like biological ones) give zero or undefined slopes, so activations are continuous instead.
- **Backpropagation** is the efficient algorithm for computing ∇C (covered in the following chapters).
- **Results.** Accuracy on unseen images is about **96%**, and about **98%** with tweaks to the hidden layers. Generalisation is tested on held-out labelled data the network never saw.
- **Failure modes / honest caveats:**
  - **Local minima.** Where you end up depends on the random start, with no guarantee of the global minimum.
  - **Hidden layers don't learn the hoped-for features.** Visualised first-layer weights look almost random rather than edge detectors. The network found "a happy local minimum" that classifies well without human-interpretable structure.
  - **Overconfidence on garbage.** Feed in random noise and it confidently answers "5". The cost never rewarded uncertainty, and its whole universe was centred, clean digits. It can recognise digits but has no idea how to draw them.
  - **Memorisation.** (Interview with Lisha Li) A deep net trained on **shuffled labels** still reaches the same training accuracy by memorising, so low training cost ≠ learned structure. But with real labels the loss drops fast after a slow start, while with random labels it falls slowly, almost linearly. Another paper suggests the local minima these nets find tend to be of roughly equal quality, so structured data is easier to optimise. (note: this is 2017-era framing; the conclusion "train accuracy alone proves nothing; check held-out data" still holds.)
  - The plain network is 1980s–90s technology and a starting point, not the state of the art.
- **Rules of thumb:**
  - Always evaluate on held-out data.
  - Choose a smooth, differentiable loss.
  - Expect the dependence on initialisation.
  - Don't trust confidence scores on out-of-distribution inputs.
  - Read gradient magnitudes as local sensitivity.
- **For Sam's work:**
  - Any model with a loss works this way, whether a learned matching score between supplier and GeoNames places, a ranking model, or a fine-tuned embedding: data → loss averaged over examples → gradient → step. Because the cost is an *average over data*, mini-batches estimate it cheaply, which is how training fits on a 16GB laptop. (note: that is stochastic gradient descent, implied rather than covered here.)
  - **Overconfidence on out-of-distribution input** is a real risk for location matching. A classifier trained on clean European place names may confidently mis-match Nigerian or transliterated names. Calibrate the model and add an "unknown / no match" option.
  - The gradient-as-sensitivity view is basically feature importance for differentiable models.

## Grant Sanderson (3Blue1Brown) — Taylor series | Essence of calculus Ch. 11 (https://www.youtube.com/watch?v=3d6DsjIBzJ4)
**A Taylor series turns derivative information at one point into approximation information near that point, by building a polynomial whose derivatives match the function's.**

- **Why polynomials.** They are easy to compute, differentiate and integrate. Ugly functions (cos, exp, ln) get replaced locally by friendly ones. Motivating example: a pendulum's potential energy ∝ 1 - cos θ is awkward. Replacing cos θ ≈ 1 - θ²/2 makes it a simple quadratic and reveals the link to other oscillators.
- **Mechanism: building the quadratic for cos x near 0.** Use P(x) = c0 + c1·x + c2·x².
  - **Match the value:** P(0) = c0 = cos 0 = 1.
  - **Match the slope:** P'(0) = c1 = -sin 0 = 0, so the tangent is flat.
  - **Match the curvature:** P''(0) = 2·c2 = -cos 0 = -1, so c2 = -1/2.
  - Result: cos x ≈ 1 - x²/2. Check: cos(0.1) ≈ 0.995, very close to the true value.
  - Each coefficient controls exactly one derivative at 0, because higher terms all contain x and vanish when you plug in 0. **Adding terms never disturbs earlier ones.**
- **Higher terms.** The third derivative of cos at 0 is sin 0 = 0, so c3 = 0. That makes 1 - x²/2 the best cubic too. The fourth derivative is cos 0 = 1, so c4 = 1/24, giving 1 - x²/2 + x⁴/24. Derivatives of cos at 0 cycle 1, 0, -1, 0, so the coefficients follow that pattern.
- **Where factorials come from.** Differentiating xⁿ n times gives n!. To make the nth derivative of the polynomial equal f⁽ⁿ⁾(0), the coefficient must be f⁽ⁿ⁾(0)/n!.
- **General formula:** P(x) = Σ f⁽ⁿ⁾(a)/n! · (x - a)ⁿ. Writing in powers of (x - a) just makes the point a behave like 0, so the same cancellation works.
  - Term meanings: constant = matching value, linear = matching slope, quadratic = matching how the slope changes, and so on.
  - More terms = closer fit but a more complex polynomial (the trade-off).
- **e^x example.** Every derivative is e^x, which equals 1 at 0, so e^x ≈ 1 + x + x²/2! + x³/3! + ...
- **Geometric picture of the second-order term (via the fundamental theorem of calculus).** Take an area-under-graph function A(x) whose derivative is the graph. Area(x) ≈ Area(a) + rectangle (graph height × (x - a)) + the little triangle on top. That triangle is ½·slope·(x - a)², and the slope of the graph is A''(a). So each Taylor term is a visible piece of the area diagram.
- **Series and convergence.** You can't literally add infinitely many terms. A series "equals" a value if its partial sums approach it.
  - e^x equals its Taylor series at **every** x (x = 1 gives e, x = 2 gives e²). So do sin and cos.
  - **Gotcha:** ln x expanded around 1 converges only for 0 < x ≤ 2 (note: 0 < x ≤ 2 including the endpoint 2; the video says "between 0 and 2"). Just outside that range the partial sums swing wildly and **diverge**, even though ln is perfectly well defined there. The derivative information at x = 1 "doesn't propagate" that far. The maximum distance over which it works is the **radius of convergence**.
- **Rules of thumb:**
  - A Taylor approximation is local. Trust it near a, and check the radius or the error elsewhere.
  - First order = the linear (tangent) approximation, second order adds curvature.
  - For small angles: sin θ ≈ θ, cos θ ≈ 1 - θ²/2.
- **For Sam's work / ML:**
  - **Gradient descent is a first-order Taylor bet.** C(w + Δ) ≈ C(w) + ∇C·Δ, so stepping along -∇C lowers the cost, but only while the step is small enough for the linear approximation to hold. That's why too large a learning rate blows up: you leave the region where the approximation is valid. The second-order term (curvature, the Hessian) is what Newton-type optimisers and Adam-style adaptive scaling try to account for.
  - **Haversine / geo distance:** for small distances, the flat-earth approximation (lat/lon scaled by cos(latitude)) is a Taylor-style simplification of the spherical formula. It's fine for nearby-place matching and wrong across long distances. Same lesson: local approximation, known radius.
  - Numeric tricks such as log1p(x) ≈ x, or exp(x) ≈ 1 + x for small x, are first-order Taylor terms that matter for numerically stable probability code.

## Grant Sanderson (3Blue1Brown) — The paradox of the derivative | Essence of calculus Ch. 2 (https://www.youtube.com/watch?v=9vKqVkMQHKk)
**"Instantaneous rate of change" is an oxymoron. The derivative is what Δs/Δt approaches as Δt shrinks toward 0 (never equal to 0): the slope of the tangent line, best read as the best constant approximation of the rate near a point.**

- **The paradox.** Velocity is distance per unit time and needs two moments to compare. A snapshot has no speed. Yet we want a velocity function v(t) that takes a single t. Calculus resolves this without ever computing "change in an instant".
- **Running example.** A car goes 100 m in 10 s, speeding up then slowing to a stop. The distance graph s(t) is shallow, then steep, then shallow. The velocity graph is a bump that peaks mid-journey.
- **Mechanism, the engineering version:**
  - **A real speedometer** sidesteps the paradox. It measures distance over a tiny but *finite* interval (e.g. 3.00 s to 3.01 s) and divides: ds/dt with dt = 0.01.
  - **Grant's plotting code** works the same way. Pick dt = 0.01, then for many t compute (s(t+dt) - s(t))/dt. That's a finite-difference derivative, which works for any distance curve.
  - Graphically, this is the rise-over-run slope of a line through two close points.
- **Mechanism, the pure math version.** The derivative is **not** the ratio for any specific dt. It is the **limit** of the ratio as dt → 0, and dt is always nonzero and finite. As the two points merge, the secant line approaches the **tangent line**.
  - The notation ds/dt announces "we will let dt → 0". It isn't literally a fraction, but thinking of ds and dt as real tiny nudges is a good working intuition.
- **Worked example: s(t) = t³ at t = 2.** Expand ((2+dt)³ - 2³)/dt = 3·2² + 3·2·dt + dt². As dt → 0 the dt terms vanish, leaving 12. In general d(t³)/dt = 3t².
  - **Key insight:** taking the limit *simplifies*. The messy terms involving dt drop out, and that's the heart of why calculus is useful. You derive this once and then reuse the rule.
- **The paradox revisited.** With s = t³, is the car moving at t = 0? The derivative there is 3·0² = 0, a flat tangent. "If not at t = 0, when does it start moving?" The question is malformed because it presumes change in an instant. Over 0 to 0.1 s the car does move 0.001 m, an average of 0.01 m/s. That's tiny *relative to* the time step, and the ratio → 0 as the step shrinks. A zero derivative means the best constant-velocity approximation is 0, not that the car is frozen.
- **Rules of thumb:**
  - Read f'(a) as "near a, f changes about f'(a) units per unit of input".
  - A derivative of 0 means locally flat, not static.
  - In code, derivatives are finite differences with a small step. In math, they're limits.
- **Failure modes:**
  - Treating dt as literally 0, which gives 0/0.
  - Treating the derivative as a literal instantaneous change, which leads to paradoxes like "when does the car start".
  - (note: in numerical practice, a dt that is too small hits floating-point cancellation, and one too large gives a biased slope. A central difference (f(x+h) - f(x-h))/2h is more accurate. That's standard practice, not from the video.)
- **For Sam's work / ML:**
  - **Finite differences are how you sanity-check gradients.** Nudge one weight by h, re-evaluate the loss, and compare (L(w+h) - L(w))/h with backprop's value. That's the standard "gradient check" for a custom loss or a hand-written matching score.
  - Rates in pipelines (bookings per hour, rows processed per second, price change per day) are finite-difference derivatives over a sampling interval. Interval choice matters: a tiny window gives noisy rates, a large window blurs changes.
  - "Best local linear approximation" is the idea that generalises to gradients in many dimensions (see the gradient descent and Taylor notes).

## Grant Sanderson (3Blue1Brown) — Eigenvectors and eigenvalues | Essence of linear algebra Ch. 14 (https://www.youtube.com/watch?v=PFDu9oVAE-g)
**An eigenvector is a vector that a transformation only stretches or squishes, never knocking it off its own line (span). The eigenvalue is that stretch factor. Eigenvectors describe what a matrix really does, independent of your coordinate system.**

- **Prerequisites.** Grant's claim: confusion about eigen-things usually comes from shaky foundations, not the topic itself. You need matrices as transformations, determinants, linear systems, and change of basis.
- **Geometric definition.** Matrix A with columns (3,0) and (1,2): î lands on (3,0), ĵ on (1,2).
  - Most vectors get rotated off their span.
  - Vectors on the x-axis stay on it and are stretched ×3 (eigenvalue 3).
  - Vectors along (-1,1) stay on that diagonal and are stretched ×2 (eigenvalue 2).
  - By linearity, an entire line of vectors shares each property.
  - Eigenvalues can be negative or fractional. For example, -1/2 means flipped and halved, but still on the same line.
- **Why care.** For a 3D rotation, an eigenvector *is* the axis of rotation (eigenvalue 1, since rotations don't stretch). "Axis + angle" is far easier to understand than 9 matrix entries. In general, eigenvectors and eigenvalues reveal a transformation's essence; reading the columns depends on the basis.
- **Mechanism: computing them.**
  1. Av = λv. Rewrite the right side as (λI)v, so (A - λI)v = 0.
  2. A nonzero v can only be sent to 0 if A - λI squashes space into a lower dimension. That means **det(A - λI) = 0**.
  3. For A = [[3,1],[0,2]]: det = (3-λ)(2-λ), so λ = 2 or 3. Plug λ = 2 back in and solve the linear system: the solutions span (-1,1).
  - Another example: columns (2,1), (2,3). "Turn the knob" on λ until the determinant hits 0; that happens at λ = 1, and there v stays fixed.
- **Edge cases to remember:**
  - **90° rotation** [[0,-1],[1,0]]: det = λ² + 1, whose roots are ±i. There are no real eigenvectors, since every vector gets rotated off its span.
  - **Shear** with columns (1,0), (1,1): det = (1-λ)², so the only eigenvalue is 1. Only the x-axis consists of eigenvectors, which is **not enough to span the plane**.
  - **Uniform scaling by 2**: a single eigenvalue, 2, but *every* vector is an eigenvector.
- **Eigenbasis and diagonalisation.**
  - If the basis vectors are themselves eigenvectors, the matrix is **diagonal**, with the eigenvalues on the diagonal (e.g. diag(-1, 2)).
  - Diagonal matrices make powers trivial: A^100 just raises each eigenvalue to the 100th power. A non-diagonal A^100 is a nightmare by hand.
  - If you have enough eigenvectors to span the space, put them as the columns of a change-of-basis matrix P. Then P⁻¹AP is diagonal. To compute A^n: change to the eigenbasis, take powers there, then change back (A^n = P·Dⁿ·P⁻¹).
  - Not always possible: a shear has too few eigenvectors.
- **Rules of thumb:**
  - Eigenvalue magnitude > 1 means repeated application blows up along that direction; < 1 means it decays; = 1 means it persists.
  - Complex eigenvalues signal rotation.
  - det(A - λI) = 0 is the test, and the null space of (A - λI) gives the eigenvectors.
  - (note: for large matrices nobody solves the characteristic polynomial. Libraries use iterative methods: numpy.linalg.eigh for symmetric matrices, or power iteration / scipy.sparse.linalg.eigsh for the top few.)
- **Gotchas:**
  - Not every matrix has real eigenvectors (rotations).
  - Not every matrix has enough eigenvectors to diagonalise (shears).
  - The zero vector is never counted as an eigenvector.
- **For Sam's work:**
  - **PCA = the eigenvectors of the covariance matrix.** The covariance matrix is symmetric, so it always has a full orthogonal eigenbasis. The top eigenvectors are the directions in which your embeddings or features vary most, and the eigenvalues are the variance along each. Keeping the top k compresses millions of rows to k dimensions on a 16GB laptop. (note: in practice it's done with SVD on centred data, which is mathematically equivalent and more stable.)
  - **Repeated application → dominant eigenvector.** PageRank and Markov chains (A^n v) converge toward the eigenvector with the largest eigenvalue. That's useful for ranking places or suppliers in a link or transition graph (e.g. a transit network).
  - Exploding or vanishing gradients in deep nets are this same story: repeated multiplication by weight matrices with eigenvalues above or below 1.

## Grant Sanderson (3Blue1Brown) — Cramer's rule, explained geometrically | Essence of linear algebra Ch. 12 (https://www.youtube.com/watch?v=jBsC34PxzoM)
**You can encode a coordinate as a signed area or volume. A linear map scales every area by the same factor, det(A). So each unknown = (determinant with that column replaced by the output) / det(A).**

- **Framing.** Solving Ax = b means: which input vector (x, y) does the transformation A send to the known output b? Equivalently, which combination x·(col 1) + y·(col 2) equals b? The video assumes **det(A) ≠ 0**, so every output has exactly one input. If det = 0, space gets squashed, and there are either no solutions or infinitely many.
- **Honest caveat.** Cramer's rule is **not** the practical method; Gaussian elimination is always faster. Grant offers it as a "cultural excursion" that ties determinants and linear systems together, and as a test of real understanding.
- **The wrong-but-useful first idea.** The coordinates are dot products with the basis vectors: x = v·î, y = v·ĵ. You'd hope that dotting the *output* with the *transformed* basis vectors (the columns of A) gives x and y. But most transformations **don't preserve dot products**: angles change, perpendicular vectors stop being perpendicular, lengths stretch.
  - **Exception: orthonormal transformations** (rotations and other rigid motions, with columns perpendicular and of unit length) do preserve dot products. Then the solution is just x_i = (column i)·b, i.e. x = Aᵀb. Solving is trivial.
- **The clever idea: coordinates as signed areas.**
  - The parallelogram spanned by î and the unknown v = (x, y) has base 1 and height y, so its signed area = **y**.
  - The parallelogram spanned by v and ĵ has signed area **x**.
  - In 3D, the z-coordinate = the signed volume of the parallelepiped (î, ĵ, v), and so on. The order of the vectors matters (right-hand rule), and the sign handles negative coordinates.
- **Mechanism.** After applying A, every area gets multiplied by det(A).
  - The transformed parallelogram (î, v) is spanned by (col 1 of A, b). So det([col1 | b]) = det(A)·y, giving **y = det([col1 | b]) / det(A)**.
  - Likewise **x = det([b | col2]) / det(A)**.
  - Everything you need comes from the output side: the columns of A and b.
- **Numeric check from the video.** Output b = (-4, -2), det(A) = 2. The altered determinants are 6 and 4, so x = 3 and y = 2, matching the original input.
- **Generalisation.** For n×n systems, replace column i with b, take the determinant, and divide by det(A). Grant leaves the 3D reasoning as an exercise and argues that working through it yourself is where the learning happens.
- **Rules of thumb:**
  - det(A) is the universal area/volume scale factor of the map.
  - If a matrix is orthonormal, its inverse is its transpose, so you solve by dot products.
  - Use elimination or LU (via a library solver) for real computation. Cramer's rule costs roughly n+1 determinants, which is hopeless for large n.
- **Gotchas:**
  - det(A) = 0 means division by zero, which matches "no unique solution".
  - Signed area or volume, not plain area: the order of the columns flips the sign.
  - (note: Cramer's rule is also numerically poor when det(A) is near 0, i.e. ill-conditioned. Libraries like numpy.linalg.solve or lstsq are both faster and more stable.)
- **For Sam's work:**
  - **Orthonormal = dot products and distances preserved.** Rotating embeddings with an orthonormal matrix (e.g. aligning two embedding spaces, or PCA's eigenvector basis) keeps cosine similarities and Euclidean distances intact. A general matrix does not, so a learned linear projection changes your matching scores.
  - The determinant as a volume scale factor appears in probability: a change of variables multiplies densities by 1/|det J|, e.g. when transforming coordinates or using normalising flows.
  - For solving least-squares fits (e.g. calibrating a geocode offset), call a solver rather than computing an inverse or using Cramer's rule.

## Grant Sanderson (3Blue1Brown) — Vectors | Essence of linear algebra Ch. 1 (https://www.youtube.com/watch?v=fNk_zzaMoSs)
**A vector is at once an arrow rooted at the origin and an ordered list of numbers. Linear algebra's power comes from translating between the two views, and everything is built on two operations: addition and scaling.**

- **Three perspectives:**
  - **Physics:** an arrow with length and direction that can be slid anywhere.
  - **Computer science:** an ordered list of numbers. For example, a house is (square footage, price): a 2D vector, where order matters and the dimension is the list length.
  - **Mathematician:** anything you can sensibly add and multiply by numbers (abstract; covered later in the series, and the same idea as Strang's matrix and function spaces).
  - **Grant's working default:** an arrow inside a coordinate system, *tail at the origin* (unlike physics). Then translate to coordinates.
- **Coordinates as instructions.** (x, y) means walk x along the x-axis, then y parallel to the y-axis. Written as a vertical column in square brackets to distinguish vectors from points. Pairs correspond one-to-one to 2D vectors, and triples to 3D vectors (add a z-axis perpendicular to both).
- **Vector addition.** Tip-to-tail: place the second vector's tail at the first one's tip, and the sum goes from the first tail to the second tip. This is the one time vectors leave the origin.
  - **Why this definition:** a vector is a *movement*. Doing step 1 then step 2 equals doing the sum, just like 2 + 5 on a number line.
  - Numerically you add componentwise: (1,2) + (3,-1) = (4,1). Reorder the walk into all horizontal moves, then all vertical moves.
- **Scalar multiplication.** ×2 stretches, ×1/3 squishes, ×(-1.8) flips then stretches. Numbers that act this way are called **scalars**. Numerically you multiply each component.
- **Why both views matter.**
  - Data analysts get a *visual, geometric* handle on lists of numbers, which clarifies patterns and what operations do globally.
  - Physicists and graphics programmers get a numeric language for describing space. Grant's animations work this way: think about the space, then compute the pixels.
  - The usefulness lies in the translation, not in either view alone.
- **Rules of thumb:**
  - When meeting a new linear-algebra concept, first picture the arrows in 2D or 3D, then translate to numbers.
  - Everything later (span, basis, linear maps, matrices) is built from addition and scaling only.
- **Gotchas:**
  - Feature order is part of the vector's meaning. Two pipelines emitting (lat, lon) vs (lon, lat) produce different vectors. (note: a classic GIS bug; GeoJSON uses lon, lat.)
  - Linear-algebra vectors are rooted at the origin, so "position" and "displacement" get mixed up if you're careless.
- **For Sam's work:**
  - Every row in a dataset is a vector: a GeoNames place as (lat, lon, population, feature-code one-hot…), or an embedding of a place name as a 384- or 768-dim vector.
  - **Addition and scaling are exactly what makes embeddings useful:** averaging vectors (scale by 1/n and add) gives a centroid, e.g. of a city's POIs. Differences between vectors capture relations.
  - Distance and similarity measures (Euclidean, cosine) only make sense if features are on comparable scales. Scaling individual components (standardisation) changes geometry, and therefore matching results. The geometric picture makes that obvious.

## Grant Sanderson (3Blue1Brown) — Matrix multiplication as composition | Essence of linear algebra Ch. 4 (https://www.youtube.com/watch?v=XkY2DOUCWMU)
**A matrix is a linear transformation, recorded by where the basis vectors land (its columns). Multiplying matrices means applying one transformation after another, read right to left.**

- **Recap: what a matrix is.** A linear transformation keeps grid lines parallel and evenly spaced, and keeps the origin fixed. So it is fully determined by where î and ĵ land: (x, y) = x·î + y·ĵ lands on x·(new î) + y·(new ĵ). Write new î and new ĵ as the matrix columns. **Matrix × vector = the columns weighted by the vector's entries.**
- **Composition.** First rotate 90° counter-clockwise, then shear. The overall effect is one new linear map. Track î and ĵ through both steps: î ends at (1,1), ĵ at (-1,0), so those are the columns of the composite matrix.
  - Applying rotation then shear to any vector (Shear·(Rot·v)) equals applying (Shear·Rot)·v. That's why we call it the **product**.
  - **Read right to left:** the rightmost matrix acts first, because we write functions to the left of their inputs, f(g(x)).
- **Mechanism: computing the product by tracking basis vectors.** M1 has columns (1,1), (-2,0); M2 has columns (0,1), (2,0). For M2·M1 (M1 first):
  - î → M1's first column (1,1) → M2·(1,1) = (2,1).
  - ĵ → (-2,0) → M2·(-2,0) = (0,-2).
  - The composite has columns (2,1), (0,-2).
  - **General rule:** column j of AB = A × (column j of B). This makes the "rows-times-columns" recipe make sense instead of being memorised.
- **Order matters (non-commutative).** Shear then rotate sends î to (0,1) and ĵ to (-1,1), close together. Rotate then shear sends î to (1,1) and ĵ to (-1,0), far apart. Different results, and you can see it in your head without arithmetic.
- **Associativity for free.** (AB)C = A(BC): both mean "apply C, then B, then A". Grant says this is an honest proof, and a better *explanation* than grinding through entries, which he calls horrible and unenlightening.
- **Rules of thumb:**
  - Before memorising a formula, ask what it does geometrically.
  - To find a composite matrix, push each basis vector through the chain.
  - Never assume AB = BA.
  - Feel free to regroup products (associativity) to save computation.
- **Gotchas:**
  - Reading order: AB applies B first.
  - Dimension bookkeeping: (m×n)(n×p) works because the n-dim output of B is the input of A.
  - (note: associativity also matters for cost. For a matrix chain with a vector at the end, A(B(Cv)) is matrix-vector work instead of building ABC first, which is much cheaper when the matrices are large.)
- **For Sam's work / ML:**
  - **A neural-network layer is a matrix (a linear map) followed by a nonlinearity.** Without the nonlinearities, stacking layers would collapse into a single matrix (the product), with no extra expressive power. Composition explains why activations are essential.
  - Pipelines of linear feature transforms (centring, scaling, PCA projection, a rotation) can be **pre-multiplied into one matrix** and applied once over millions of rows. That's cheaper on a 16GB laptop: chunk the rows and do one matmul per chunk.
  - Order of preprocessing steps matters (non-commutativity). Standardise-then-project ≠ project-then-standardise.
  - Coordinate conversions in GIS work (rotations, scalings, projection approximations) are compositions. Getting the order wrong is a silent bug.

## Grant Sanderson (3Blue1Brown) — Backpropagation calculus | Deep Learning Ch. 4 (https://www.youtube.com/watch?v=tIeHLnjs5U8)
**Backprop is the chain rule applied to a chain of tiny nudges: w → z → a → C. Multiply the local sensitivities, sum over every path a variable influences, and reuse each layer's result to step one layer back.**

- **Simplest case: one neuron per layer** (3 weights, 3 biases). Focus on the last link.
  - z^L = w^L·a^(L-1) + b^L, then a^L = σ(z^L).
  - Cost for one example: C0 = (a^L - y)².
  - Superscripts are layer indices, not exponents.
  - Picture each quantity on its own number line: a nudge to w moves z, which moves a, which moves C.
- **Mechanism: the chain rule as a product of nudge ratios.**
  - ∂C0/∂w^L = (∂z^L/∂w^L)·(∂a^L/∂z^L)·(∂C0/∂a^L) = **a^(L-1) · σ'(z^L) · 2(a^L - y)**.
  - **2(a^L - y):** the gradient is proportional to the error. A badly wrong output means small changes matter a lot.
  - **σ'(z^L):** the slope of the activation function.
  - **a^(L-1):** a weight's influence scales with how active the incoming neuron is. That's "neurons that fire together wire together".
  - **Bias:** the same expression, but ∂z/∂b = 1, so ∂C0/∂b^L = σ'(z^L)·2(a^L - y).
  - **Previous activation:** ∂z^L/∂a^(L-1) = w^L. This is the "propagate backwards" step. You can't change a^(L-1) directly, but knowing ∂C/∂a^(L-1) lets you repeat the same chain-rule step for the weights and biases one layer earlier, and so on back to the input.
- **From one example to the gradient.** The full cost is the **average over training examples**, so each partial derivative is averaged over examples too. Each partial is one component of ∇C. The weight derivative is "more than 50% of the work"; the bias and previous-activation derivatives reuse most of the same factors.
- **Multiple neurons per layer.** Index neurons in L-1 by k and neurons in L by j. w^L_jk connects neuron k to neuron j; the index order matches row j, column k of the layer's weight matrix.
  - z^L_j = Σ_k w^L_jk·a^(L-1)_k + b^L_j, and C0 = Σ_j (a^L_j - y_j)².
  - The derivative for a single weight looks the same as before.
  - **The one real change:** ∂C0/∂a^(L-1)_k = Σ_j w^L_jk·σ'(z^L_j)·∂C0/∂a^L_j. A neuron feeds the cost through **several paths** (every neuron in the next layer), so you **sum** the contributions over the paths.
- **Rules of thumb:**
  - Name the intermediate weighted sum z. It splits the chain into clean local derivatives.
  - Every gradient for a weight = (input activation) × (local slope) × (upstream sensitivity).
  - Multiply along a path, add across paths.
  - Compute the backward pass layer by layer, reusing upstream results. That reuse is what makes backprop efficient compared with differentiating each weight separately.
- **Failure modes / intuition for known problems:**
  - Because gradients are *products* of σ' and w factors across layers, small factors shrink them toward zero (sigmoid σ' ≤ 0.25), and large ones explode them. (note: this is the vanishing/exploding gradient problem, which the video doesn't name. It's why ReLU, careful initialisation and normalisation are standard.)
  - A dead input neuron (a^(L-1) = 0) means its outgoing weights get zero gradient for that example.
- **For Sam's work:**
  - Autodiff libraries (PyTorch, JAX) do exactly this mechanically. Understanding it helps you debug NaNs, zero gradients and exploding losses when fine-tuning an embedding model or training a learned match scorer.
  - The "sensitivity = product of local ratios" pattern also works for pipeline reasoning outside ML: how much does a final matching score change if one feature's weight changes? Multiply the local sensitivities along the path, and sum over the features it feeds.
  - The error term 2(a - y) shows that squared loss weights big mistakes heavily, which makes it sensitive to outliers. Consider absolute or Huber loss for noisy labels such as crowd-sourced place matches.

