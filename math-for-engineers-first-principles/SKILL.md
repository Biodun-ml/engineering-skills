---
name: math-for-engineers-first-principles
description: First-principles maths playbook from 19 lectures (3Blue1Brown, Gilbert Strang MIT 18.06/18.065, Joe Blitzstein Stat 110). Use whenever explaining or applying linear algebra, calculus, gradients/backprop or probability: embeddings, PCA/SVD, low rank, solvers, distances, NOUN coursework.
---

# Maths for engineers from first principles

This skill turns 19 lectures into working intuition for the maths underneath data, ML and
geo work: linear algebra as geometry (3Blue1Brown, Strang), calculus as sensitivity and
accumulation (3Blue1Brown), neural nets as applied calculus (3Blue1Brown), and probability by
story (Blitzstein). This file holds how to explain and learn a concept, how to use the maths in
engineering work, the principles, checklists and gotchas. `references/` holds detailed notes
for every video, each credited to its teacher.

Use it whenever maths is the hidden question, even if nobody says "maths": "why did cosine
scores change after PCA?", "singular matrix error", "loss is NaN", "is this distance right
near the poles?", "why won't this average settle?", explaining a concept for a video, or a
NOUN BSc Mathematics topic.

**How to use it**
1. To explain or learn a concept, follow section 1. To apply maths to a real problem, follow
   section 2. Then run the checklists in sections 4–5.
2. If a `references/` folder sits next to this file, use it for depth. `references/INDEX.md`
   lists every video with its line number, or run `grep -n -i "keyword" references/*.md`. Each section ends with a "Relevance to Sam" or "For Sam" note: Sam builds
   large location-data pipelines (GeoNames ~6M rows, matching/dedup), supplier integrations,
   embeddings for matching, and an LLM pipeline for travel articles, all on a 16 GB laptop.
   Use those parts only in that context. Lines marked "(note: …)" and those Sam notes are
   helper additions, not the speaker's words. If `references/` is missing, the full notes live
   in the user's private GitHub repo `Biodun-ml/engineering-skills` (folder
   `math-for-engineers-first-principles/references/`); clone it when you need depth.
3. Lead with the picture, then the numbers, then the code. Credit the teacher when you use
   their framing.
4. The videos teach intuition, not numerical practice. Section 6 and the box in section 8
   cover what the sources leave out; say so when you rely on them.

---

## 1. How to explain or learn a concept

**When explaining to Sam (or for his channel):**
1. **Start from a question in his work**, not the definition ("why does PCA keep cosine
   scores only sometimes?"). Grant's stated aim is that you feel you could have invented the
   idea yourself. *(3B1B — Essence of calculus)*
2. **Draw it in 2D first, then translate to numbers.** Every linear-algebra idea is built from
   adding and scaling; picture arrows, then write coordinates. *(3B1B — Vectors Ch.1)*
3. **One worked example with small integers** that he can check by hand (e.g. columns (3,0)
   and (0,2) → det 6; t³ at t = 2 → slope 12). *(3B1B — Determinant; Derivative Ch.2)*
4. **Then the code version:** a 10–20 line numpy check that confirms the picture.
5. **Then where it bites in his work** (section 3 table) and **one failure mode** (section 6).
6. **Name the honest caveat** the teacher gave (Cramer's rule isn't practical; the "edges →
   loops → digits" story is a hope, not what nets learn). *(3B1B — Cramer Ch.12; NN Ch.1)*
7. Skip proofs unless asked, but keep the one-sentence "why" (det(AB) = det(A)det(B) because
   areas scale by one factor then the other). *(3B1B — Determinant Ch.6)*

**Learning a maths topic without boredom (Sam's procedure).** Sam finds tutorials and toy
projects boring and has about 40 minutes on weekdays. So learning is driven by building and
explaining, in short sessions:
1. **Pick a live bug or question** from current work, not a chapter number. One per session.
2. **Predict before you look.** Write a one-line guess ("rotating embeddings won't change
   cosine scores"). Being wrong is what makes it stick.
3. **Watch only the one short video that answers it** (a single 3B1B chapter), or
   read its notes section in `references/`, not the whole series.
4. **Break it in code (15 min).** Build the smallest numpy experiment that confirms or kills
   the prediction: make a rank-deficient matrix and call `solve`; rotate vectors with an
   orthonormal Q and compare cosines; finite-difference a loss.
5. **Teach it back (5 min).** Explain it out loud or as a 60-second script. Explaining holds his
   attention better than reading. If the explanation needs a hand-wave, that is the gap to
   fill next session.
6. **Ship one artefact:** a test, an assert, a comment in a pipeline, or a short clip. Keep
   personal learning logs out of the shared `engineering-skills` repo.
7. **Spiral, don't march.** Return to the same idea from a new problem a week later (Strang's
   rank appears again in PCA, LoRA and least squares).

**Alongside NOUN BSc Mathematics coursework:**
- Use this skill to get the *picture* for each topic before the formal version in the course
  material: vectors, span, linear maps, determinants and eigenvalues (3B1B + Strang),
  derivatives, Taylor series and the fundamental theorem (3B1B), distributions (Blitzstein).
- Exams and assignments still want the hand methods (cofactor determinants, Cramer's rule,
  characteristic polynomials, Taylor coefficients). Practise those by hand; then check every
  answer with numpy or sympy. Practice and exams are different tools, not a contradiction.
- Turn each proof into a picture: "column rank = row rank" is A = CR read two ways *(Strang —
  18.065 L1)*; "dim N(A) = n − r" is counting free variables *(Strang — 18.06 L11)*.
- Claude explains, generates fresh practice problems, and checks working; Sam writes the
  submitted answers himself.

---

## 2. How to use the maths in engineering work

1. **Name the object.** Is it a vector (a row, an embedding), a linear map (a matrix, a layer,
   a projection), a function to minimise (a loss), or a random quantity (a count, a ratio)?
2. **Ask the geometric question.** For a matrix: what does it do to space, and does it
   collapse dimension? For a loss: which way is downhill, and how far does the linear picture
   hold? For data: what story generated it? *(3B1B — Linear transformations Ch.3; Taylor
   Ch.11; Blitzstein — Lec 20)*
3. **Check the rank and conditioning before solving anything.** Dependent columns mean no
   unique answer. *(Strang — 18.06 L1, L2)*
4. **Prefer the library routine that matches the structure:** `solve`/`lstsq` not `inv`;
   `eigh` for symmetric matrices; SVD for PCA; `logsumexp` for probabilities.
5. **Check numerically:** finite-difference gradients, reconstruct A from its factors,
   simulate a distribution and compare with the formula.
6. **State the approximation and its radius** (Taylor order, flat-earth distance, linear
   model): where is it valid, where does it break? *(3B1B — Taylor Ch.11)*

---

## 3. Principles (credited)

### Linear algebra as geometry
- **A vector is anything you can add and scale.** Arrows, number lists, matrices and
  functions all qualify; the power is in translating between the geometric and numeric views.
  *(3B1B — Vectors Ch.1; Strang — 18.06 L11)*
- **Coordinates hide a choice of basis.** (3, −2) means "3 of î, −2 of ĵ". Embedding
  dimensions have no inherent meaning; directions, angles and distances do, and they survive a
  rotation of the basis. *(3B1B — Span Ch.2)*
- **Span, independence, basis and rank are one idea.** Span = what you can reach; independent
  = no redundant vectors; basis = independent and spanning; rank = number of independent
  columns = dimension of the column space. Random vectors are almost always independent;
  dependence comes from structure (duplicated or derived quantities). *(3B1B — Ch.2; Strang —
  18.065 L1)*
- **A matrix is a linear map, recorded by where the basis vectors land (its columns).** To
  build one, decide where each basis vector goes. Linear = origin fixed, grid lines parallel
  and evenly spaced (check a diagonal too). A shift (Wx + b) is affine, which is why layers
  carry a bias. *(3B1B — Ch.3)*
- **Ax is a weighted mix of A's columns.** Make this the default reading; "solve Ax = b"
  becomes "which mix of columns reaches b?", and it scales to any dimension where the row
  picture (planes meeting) doesn't. *(Strang — 18.06 L1; 18.065 L1)*
- **Multiplication is composition, read right to left.** AB applies B first. Order matters
  (AB ≠ BA); grouping doesn't, so (AB)C = A(BC). Column j of AB = A × column j of B.
  *(3B1B — Composition Ch.4; Strang — 18.06 L2)*
- **Left-multiply acts on rows, right-multiply on columns.** To build a matrix that does an
  operation to rows, do it to the identity. *(Strang — 18.06 L2)*
- **The determinant is the signed volume scale factor.** |det| = area/volume scaling, sign =
  orientation kept or flipped, 0 = dimension collapsed, so columns dependent and no inverse.
  det(AB) = det(A)det(B). *(3B1B — Determinant Ch.6)*
- **Cramer's rule is the geometric bridge, not the method.** Coordinates are signed areas, so
  xᵢ = det(A with column i replaced by b)/det(A). Use elimination in practice. If A is
  orthonormal, dot products survive and x = Aᵀb. *(3B1B — Cramer Ch.12)*
- **Elimination is how systems are really solved.** Reduce A to upper-triangular U with row
  operations E_ij, then back-substitute. Pivots 1, 2, 5 → det = 10, free. A zero pivot with a
  nonzero below → swap rows; zeros all the way down → singular. Elimination solves *and*
  diagnoses. *(Strang — 18.06 L2)*
- **Eigenvectors are the directions a map only stretches.** Av = λv; find λ from
  det(A − λI) = 0, then v from the null space of A − λI. In an eigenbasis A is diagonal and
  Aⁿ = P Dⁿ P⁻¹. Rotations have no real eigenvectors (complex λ = rotation); shears have too
  few to diagonalise. |λ| > 1 grows under repetition, < 1 decays. *(3B1B — Eigen Ch.14)*
- **Rank-1 matrices (u vᵀ) are the building blocks.** A rank-r matrix is a sum of r of them;
  AB = Σₖ (column k of A)(row k of B). Rank-r matrices and unions of subspaces are *not*
  subspaces; combine subspaces with the sum. *(Strang — 18.06 L11; 18.065 L1)*
- **A = CR proves column rank = row rank.** C = the independent columns of A (kept greedily,
  left to right); R = the reduced echelon form without zero rows, saying how to rebuild every
  column. CUR uses real rows too, which keeps factors interpretable. *(Strang — 18.065 L1)*
- **dim N(A) = n − r and dim N(Aᵀ) = m − r.** Show a set is a subspace by exhibiting it as a
  null space: "components sum to 0" is N([1 1 1 1]), dimension 3 — exactly centred data.
  *(Strang — 18.06 L11)*
- **Sketch huge matrices with random vectors.** Each Ax lies in C(A); ~100 random products
  often outline the column space. This is the idea behind randomised SVD. *(Strang — 18.065 L1)*

### Calculus as sensitivity
- **A derivative is the limit of Δf/Δx, not an "instantaneous change".** Read f′(a) as "near
  a, f moves about f′(a) per unit input". Zero derivative = locally flat, not frozen. In code
  it is a finite difference. *(3B1B — Derivative Ch.2)*
- **Integrals are limits of sums of thin pieces; the derivative of an accumulated total is its
  rate (fundamental theorem).** Slice along the problem's symmetry (rings for a circle). CDF →
  PDF and cumulative bookings → booking rate are the same move. *(3B1B — Essence of calculus)*
- **Taylor series turn derivatives at a point into a local approximation.** Coefficient n is
  f⁽ⁿ⁾(a)/n!; adding a term never disturbs earlier ones; cos x ≈ 1 − x²/2. Valid only inside
  the radius of convergence: ln x about 1 converges only for 0 < x ≤ 2. *(3B1B — Taylor Ch.11)*
- **Gradient descent is a first-order Taylor bet.** C(w + Δ) ≈ C(w) + ∇C·Δ holds only for small
  steps, which is why too large a learning rate blows up. *(3B1B — Taylor Ch.11, note)*

### Neural networks as applied calculus
- **A network is repeated σ(Wa + b).** Each row of W is a template the neuron matches; the bias
  is its threshold. Without nonlinearities, stacked layers collapse into one matrix. 784-16-16-10
  has 13,002 parameters. Modern nets use ReLU and softmax + cross-entropy, not the sigmoid and
  squared error used for teaching. *(3B1B — NN Ch.1; Composition Ch.4)*
- **Learning = minimising an average cost by stepping along −∇C.** Sign of each component =
  which way to nudge; magnitude = how much that parameter matters now (3.2 vs 0.1 → 32× more
  leverage). The cost must be smooth, which is why activations are continuous. *(3B1B — DL
  Ch.2, Ch.3)*
- **Backprop = chain rule as bookkeeping.** Multiply local sensitivities along a path, sum
  across paths, reuse upstream results. ∂C/∂w = a^(L−1) · σ′(z^L) · 2(a^L − y): input activation
  × local slope × upstream error. *(3B1B — Backprop calculus DL Ch.4)*
- **Products of many factors vanish or explode.** Sigmoid σ′ ≤ 0.25, so deep sigmoid stacks
  shrink gradients; ReLU, initialisation and normalisation fix this. *(3B1B — DL Ch.4, note)*
- **Mini-batch SGD trades exactness for speed:** shuffle, batch (~100), step per batch — "a
  drunk man stumbling quickly down a hill". *(3B1B — DL Ch.3)*
- **Low training cost ≠ learned structure.** Nets memorise shuffled labels, find
  uninterpretable minima, and answer "5" to random noise with confidence. Validate on held-out
  data; distrust confidence out of distribution. Labelled data is the usual bottleneck.
  *(3B1B — DL Ch.2, Ch.3)*

### Probability by story
- **Recognise structure before integrating.** Z1 − Z2 ~ N(0, 2) (variances add, even for a
  difference), so E|Z1 − Z2| = 2/√π ≈ 1.13 with a 1-D integral. Sums of *independent* normals
  are normal (MGF proof). *(Blitzstein — Lec 20)*
- **The multinomial counts n objects into k buckets.** Each count is Bin(n, pⱼ), so E = npⱼ and
  sd = √(npⱼ(1−pⱼ)) immediately. Lumping buckets gives another multinomial; conditioning on X1 =
  n1 renormalises the rest by pⱼ/(1 − p1). *(Blitzstein — Lec 20)*
- **Ratios of noise can be Cauchy: no mean, no variance.** X/Y for iid N(0,1) is Cauchy; the
  mean of n Cauchys is still Cauchy, so averaging never converges. Guard denominators and use
  medians or trimmed means. *(Blitzstein — Lec 20)*

---

## 4. Concept → where it shows up in Sam's work

| Concept | Where it shows up | Rule |
|---|---|---|
| Dot product, cosine similarity | Embedding matching of place names, supplier records | Cosine = dot of unit vectors; rotations (orthonormal Q) preserve it; general projections, dropping dimensions and mean-centring don't *(3B1B Cramer Ch.12, Span Ch.2; section 8)* |
| Change of basis, PCA | Compressing embeddings/features to k dims on 16 GB | Centre first; PCA = top eigenvectors of the (symmetric) covariance, computed via SVD in practice *(3B1B Eigen Ch.14 + note; Strang L11)* |
| Rank-1 pieces, SVD | Truncated SVD, matrix-factorisation recommenders | Keep the top-r u vᵀ pieces; storage r(m+n) not mn *(Strang 18.065 L1)* |
| Low rank, LoRA | Fine-tuning an embedder or small LLM | Update ΔW = BA with small inner dimension r *(Strang 18.065 L1, note)* |
| Rank deficiency, collinear features | total_price = base + tax; same field from two suppliers; two string metrics on one name | Derived columns add no rank → non-unique, unstable weights; drop or regularise *(Strang 18.06 L1; 3B1B Ch.2)* |
| "Singular matrix" error | Regression, solvers | A data problem (duplicated/collinear features), not a library bug *(Strang 18.06 L2)* |
| Condition number | Near-duplicate features, ill-posed fits | Use singular values, not det, to judge "close to singular" *(3B1B Ch.6, note)* |
| Composition | Preprocessing chains, GIS transforms, NN layers | Pre-multiply linear steps into one matrix; order matters (standardise-then-project ≠ project-then-standardise) *(3B1B Ch.4)* |
| Determinant / 2×2 cross product | Polygon area (shoelace), left/right-of-line test in spatial joins | Signed area; sign = orientation *(3B1B Ch.6)* |
| Taylor, local approximation; sphere vs plane | Flat-earth distance vs haversine; polygon areas; log1p, exp ≈ 1 + x | Trust near the expansion point only; areas need a projected/geodesic method *(3B1B Taylor Ch.11, note; section 8)* |
| Dominant eigenvector | PageRank-style ranking on transit or link graphs | Repeated Aⁿv (normalised each step) converges to the eigenvector with largest \|λ\| *(3B1B Eigen Ch.14, note)* |
| Small-world graphs | GTFS stop graphs with hub shortcuts | A few shortcuts collapse path lengths *(Strang 18.06 L11)* |
| Finite differences | Gradient checks for custom losses or match scores; per-hour rates | Central difference, float64 *(3B1B Derivative Ch.2, note)* |
| Gradient as sensitivity | Feature importance in a differentiable match scorer | Bigger \|∂L/∂w\| = more leverage now, if features are on comparable scales *(3B1B DL Ch.3)* |
| Multinomial | Bookings per supplier, places per feature class, matches per bucket | Binomial error bars per bucket *(Blitzstein)* |
| Cauchy | Price ratios, rates with small denominators, distance ratios in scores | Medians; guard denominators *(Blitzstein)* |
| Sum of normals | Errors stacking through pipeline stages | Variances add if independent *(Blitzstein)* |
| Parameter count | Will this model fit in 16 GB? | in×out + out per dense layer; float32 = 4 bytes/param *(3B1B NN Ch.1, note)* |

---

## 5. Checklists

**Before solving or fitting anything linear**
- [ ] Shapes written down; (m×n)(n×p) chain makes sense.
- [ ] Rank checked (`matrix_rank` or singular values); derived/duplicated columns removed.
- [ ] Features centred/standardised where the method is scale-sensitive.
- [ ] `solve` or `lstsq`, never `inv(A) @ b`.
- [ ] Result checked: residual ‖Ax − b‖ small, or factors reconstruct A.

**Before trusting embeddings or distances**
- [ ] Normalised vectors for cosine; same model and preprocessing on both sides.
- [ ] Any projection (PCA, learned) checked against a labelled set of matches before/after.
- [ ] Feature order consistent ((lat, lon) vs GeoJSON's (lon, lat)).
- [ ] Distance formula fits the scale: flat approximation only for short ranges; haversine or
      geodesic otherwise; antimeridian wrap handled.

**Before training or debugging a model**
- [ ] Loss is smooth and differentiable; gradient checked by finite differences on a tiny batch.
- [ ] Held-out set exists; confidence not trusted out of distribution; "no match" class allowed.
- [ ] NaNs/zero gradients traced to a layer (dead ReLUs, saturated sigmoids, huge LR).
- [ ] Memory: params × 4 bytes, plus gradients and optimiser state (section 8).

**Before averaging or reporting a statistic**
- [ ] Denominators can't be near zero; medians for ratios.
- [ ] Count data modelled as binomial/multinomial with error bars.
- [ ] Independence actually holds before adding variances.

---

## 6. Numerical gotchas (mostly outside the videos; standard practice)

- **Float precision.** float32 has ~7 significant digits (machine epsilon ≈ 1.2e-7); float64 ~16
  (≈ 2.2e-16). Longitudes near ±180 stored as float32 are only resolved to ~1.5e-5° ≈ 1.7 m:
  keep coordinates in float64. Never test floats with `==`; use `np.isclose`.
- **Catastrophic cancellation.** Subtracting nearly equal numbers loses digits; a finite-
  difference step that is too small hits this, too large biases the slope. Use the central
  difference (f(x+h) − f(x−h))/2h with h ≈ 1e-5 in float64. *(3B1B Derivative Ch.2, note)*
- **Near-singular matrices.** A tiny determinant may just be units; a huge one may hide
  dependence. Use the condition number κ = σ_max/σ_min: you lose roughly log10(κ) digits.
  Libraries use partial pivoting because tiny pivots amplify rounding. *(Strang 18.06 L2, note;
  3B1B Ch.6, note)*
- **Don't form AᵀA if you can avoid it:** it squares the condition number. `lstsq` (QR/SVD) is
  safer than the normal equations.
- **Standardise before distance, PCA or regularisation.** Scaling a component changes the
  geometry and so the matching result; metres vs kilometres can dominate a distance. *(3B1B
  Vectors Ch.1)*
- **log-sum-exp.** Compute log Σ exp(xᵢ) as m + log Σ exp(xᵢ − m) with m = max xᵢ; softmax the
  same way. Multiply probabilities in log space. Use `log1p`/`expm1` for small x. *(3B1B Taylor
  Ch.11, note)*
- **Cost of elimination** is ~n³/3 for n × n: fine for hundreds of features, not for million-row
  Gram matrices. Chunk, stream, or use iterative/randomised methods. *(Strang 18.06 L2, note)*
- **Matrix-chain order.** A(B(Cv)) is matrix-vector work; building ABC first is far costlier.
  *(3B1B Ch.4, note)*
- **Eigen in practice.** Nobody solves the characteristic polynomial for big matrices: use
  `eigh` for symmetric, `eigsh`/power iteration for the top few. *(3B1B Eigen Ch.14, note)*

---

## 7. Failure modes and where the teachers differ

**Failure modes**
- Memorising (ax + by, cx + dy) instead of "columns = where the basis lands", then getting stuck
  at composition and eigenvectors. *(3B1B Ch.3)*
- Treating dt as 0 (0/0) or the derivative as literal instantaneous change. *(3B1B Ch.2)*
- Assuming variances subtract for differences; assuming averages converge for heavy tails.
  *(Blitzstein)*
- Conditioning a multinomial without renormalising (p2..pk don't sum to 1). *(Blitzstein)*
- Trusting a Taylor or flat approximation outside its radius. *(3B1B Taylor)*
- Writing a factorisation without checking it reconstructs A. *(Strang 18.065 L1)*
- Reading low training loss as success. *(3B1B DL Ch.2)*

**Where the teachers differ, and how to resolve it**
- **Rows vs columns.** Strang pushes the column view (Ax = mix of columns) as the habit; 3B1B's
  neuron reads each output as a row of W dotted with the input (a template match). Use columns
  for span, rank and solvability; rows for similarity and "what does this unit detect".
- **Determinants.** Grant: meaning first, geometric Cramer's rule. Strang: compute via
  elimination, det = product of pivots. Use det for meaning and invertibility yes/no;
  elimination for computing; singular values for "how close to singular".
- **Factorisations.** Grant's eigenbasis diagonalises a square map (if enough eigenvectors);
  Strang's CR/CUR and rank-1 sums work for any rectangular data matrix. For data, use SVD
  (section 8); eigen-decomposition only for square, especially symmetric, matrices.
- **Exact vs noisy gradients.** DL Ch.2 frames the full average gradient; Ch.3 says use noisy
  mini-batches. Use mini-batches in practice; the exact gradient is the thing being estimated.
- **Teaching choices vs current practice.** Sigmoid and squared error appear in the videos;
  use ReLU-family activations and cross-entropy for classification. *(notes' corrections)*
- **Structure vs brute force.** Blitzstein solves by recognising the story; engineers often
  simulate. Do both: derive by story, then confirm with a quick Monte Carlo.

---

## 8. Not covered by the sources, but know this

> These are standard results the 19 videos don't teach (or only touch in helper notes). Say
> so when you rely on them; they are not credited to the teachers.
>
> - **Dot product and projection.** a·b = ‖a‖‖b‖cos θ. The projection of b onto a is
>   (a·b / a·a) a. Cosine similarity = dot product of unit vectors, and for unit vectors
>   ‖a − b‖² = 2 − 2 cos θ, so cosine and Euclidean ranking agree after normalising.
>   Cosine is *not* translation-invariant: mean-centring (as PCA does) changes it.
> - **Orthogonality and least squares.** With noisy data b is not in C(A); least squares picks
>   x̂ so the residual b − Ax̂ is orthogonal to every column: AᵀAx̂ = Aᵀb. Ax̂ is the projection
>   of b onto the column space. Compute with `lstsq`/QR, not by inverting AᵀA. Orthonormal Q
>   has QᵀQ = I and preserves lengths and angles.
> - **SVD, the factorisation for data.** Any m × n matrix A = UΣVᵀ = Σ σᵢ uᵢ vᵢᵀ, Strang's
>   rank-1 pieces sorted by importance. Keeping the top k gives the best rank-k approximation
>   (Eckart–Young). PCA = SVD of the centred data matrix; σᵢ²/(n−1) are the variances along each
>   component. Rank = number of nonzero σ (in floats: above a tolerance). κ = σ₁/σᵣ.
> - **Norms.** L2 (Euclidean), L1 (robust, sparsity-inducing as a penalty), L∞ (worst case).
>   Frobenius norm = √(Σ entries²) = √(Σ σᵢ²). Regularisation (ridge = L2) also fixes
>   collinear features by making AᵀA + λI invertible.
> - **Matrix-calculus shapes.** The gradient has the same shape as the parameter. For Y = XW:
>   ∂L/∂W = Xᵀ(∂L/∂Y), ∂L/∂X = (∂L/∂Y)Wᵀ. If shapes don't line up, the formula is wrong. The
>   Jacobian of f: Rⁿ → Rᵐ is m × n; the Hessian holds curvature (second-order Taylor term).
> - **Gradient checks in numbers.** Compare analytic and numeric gradients with relative error
>   |a − n| / max(|a|, |n|); around 1e-7 or smaller in float64 is healthy, above ~1e-3 usually
>   means a bug. Check on a few random coordinates of a tiny batch, without dropout.
> - **Convexity.** A convex loss (linear regression, logistic regression) has every local
>   minimum global, so the answer doesn't depend on initialisation. Neural nets are non-convex,
>   which is why 3B1B's local-minimum caveats apply.
> - **Sphere vs plane.** Lat/lon are angles, not a linear space. Haversine distance with
>   R ≈ 6371 km: d = 2R·asin(√(sin²(Δφ/2) + cos φ₁ cos φ₂ sin²(Δλ/2))). The flat (equirectangular)
>   approximation scales Δλ by cos φ and is fine for short ranges away from the poles and the
>   antimeridian. Never run the shoelace formula on raw degrees: project to an equal-area CRS
>   or use a geodesic library (e.g. pyproj `Geod`) for areas. Handle ±180° wrap explicitly.
> - **Distributions every engineer needs.** Bernoulli/binomial (yes/no and counts), Poisson
>   (events per interval, mean = variance), normal (sums of many small effects), exponential
>   (waiting times), uniform, Beta (a rate with few observations), and heavy tails
>   (log-normal/power law: prices, populations — use medians and log scales). Central limit
>   theorem: averages of finite-variance data become normal; Cauchy is the counterexample.
> - **Bayes intuition.** P(match | high score) depends on the base rate. Among millions of
>   candidate pairs with few true matches, even an accurate scorer flags mostly false
>   positives unless you block/pre-filter first. For priors, A/B tests and evaluation, use the
>   `data-science-statistics-first-principles` skill.
> - **Training memory.** float32 Adam needs ~16 bytes per parameter (weights, gradients, two
>   moment estimates) plus activations; bf16/fp16 and LoRA cut this.

---

## 9. Applying this to Sam's work

- **Matching (GeoNames ↔ supplier places):** independent signals (name similarity, distance,
  country, category) each add a dimension; two string metrics on one field are nearly
  collinear. Normalise embeddings, block by country/geohash before scoring (Bayes base rate),
  allow "no match", and calibrate on a labelled pair set built first. *(3B1B Ch.2, DL Ch.2–3)*
- **Compressing millions of rows on 16 GB:** chunked matmuls with one pre-multiplied transform
  matrix; randomised SVD/PCA with k components; float32 for embeddings, float64 for
  coordinates. *(3B1B Ch.4; Strang 18.065 L1)*
- **Geo pipelines (ferry spatial join, transit graphs):** orientation tests and areas via the
  2×2 determinant in a projected CRS; haversine for distance; hub shortcuts shrink GTFS paths.
  *(3B1B Ch.6; Strang 18.06 L11)*
- **Rates and ratios (prices, conversions, supplier shares):** multinomial counts with binomial
  error bars; medians for ratios with small denominators. *(Blitzstein)*
- **Fine-tuning a match scorer or embedder:** gradient-check custom losses, watch for vanishing
  or exploding gradients, prefer LoRA when memory is tight. *(3B1B DL Ch.4; Strang 18.065 L1)*

---

## 10. Where to look in `references/`

| File | What it covers |
|---|---|
| `math-1-blitzstein-strang-sanderson.md` | Blitzstein Stat 110 Lec 20 (multinomial, Cauchy, sums of normals); Strang 18.06 L11 (matrix spaces, rank 1, dimension, small-world graphs); 3B1B gradient descent (DL Ch.2), Taylor series, the derivative paradox, eigenvectors, Cramer's rule, vectors, composition, backprop calculus (DL Ch.4) |
| `math-2-strang-sanderson.md` | Strang 18.065 L1 (column space, rank, A = CR, rank-1 products, random sketching); Strang 18.06 L2 (elimination, pivots, E and P matrices) and L1 (row vs column picture); 3B1B neural network (DL Ch.1), essence of calculus (FTC), backprop intuition and SGD (DL Ch.3), linear transformations, determinant, span and basis |
| `INDEX.md` | Line number of every video section in both files |
