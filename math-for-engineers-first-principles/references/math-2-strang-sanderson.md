# math-2 — cross-cutting principles

1. **Ax is a weighted mix of A's columns.** Make this your default reading of matrix-vector products. It turns "solve Ax = b" into "which mix of columns reaches b?" and scales to any dimension. (Strang, L1 18.065 + Geometry of Linear Equations; 3B1B ch.3)
2. **A matrix is a transformation of space, and its columns are where the basis vectors land.** To build a matrix, decide where each basis vector should go. To read one, look at its columns. (3B1B, Linear transformations)
3. **Span, independence, basis, rank: one idea seen four ways.** Span = what you can reach. Independent = no redundant vectors. Basis = an independent spanning set. Rank = number of independent columns = dimension of the column space. (3B1B ch.2; Strang L1)
4. **Rank deficiency means structure or redundancy.** Random matrices are almost always full rank. Dependence comes from derived or duplicated columns (col3 = col1 + col2), and it makes solutions non-unique or impossible. (Strang L1, Geometry; 3B1B ch.2)
5. **Column rank = row rank, shown by A = CR.** C holds real independent columns of A. R (the RREF without zero rows) says how to rebuild every column. Reading the product by columns and by rows proves both ranks equal r. (Strang L1 18.065)
6. **AB = sum of rank-1 outer products (column k of A)(row k of B).** This is the basis of low-rank approximation (SVD, LoRA). Storage drops from m*n to r(m+n) numbers. The cost is m*n*p multiplications either way. (Strang L1 18.065)
7. **Sketch huge matrices by multiplying by random vectors.** Each Ax lies in the column space, so ~100 random products outline it. This is the idea behind randomized SVD and random projections, and it suits RAM-limited laptops. (Strang L1 18.065)
8. **Elimination, not determinants, is how systems are actually solved.** A becomes U through row operations E_ij (left-multiplication), then back substitution. A zero pivot with no row below to swap means the matrix is singular. det = product of pivots. Undo a step by adding back what you subtracted. (Strang, Elimination with Matrices)
9. **Left-multiply acts on rows, right-multiply acts on columns. Order matters, parentheses don't.** Associativity (E32 (E21 A) = (E32 E21) A) is behind many proofs and much code optimisation. (Strang, Elimination)
10. **The determinant is a volume-scaling factor with a sign.** |det| = how much area or volume is scaled. Negative = orientation flipped. 0 = dimension collapsed, so the columns are dependent and there is no inverse. det(AB) = det(A) det(B). For "how close to singular", prefer singular values or the condition number. (3B1B ch.6)
11. **A derivative is sensitivity: output nudge per input nudge, in the limit.** Gradient components are per-parameter sensitivities. A gradient component of 3.2 vs 0.1 means 32x more leverage on the loss. Check gradients numerically with (f(x+h) - f(x))/h. (3B1B Essence of calculus; Backprop)
12. **Integrals are limits of sums of thin pieces, and the derivative of an accumulated total is its rate (fundamental theorem of calculus).** Decompose along the problem's natural symmetry (rings for a circle) and let the slices shrink. (3B1B Essence of calculus)
13. **A neural network is repeated sigmoid(Wa + b): a linear map, then a shift, then a nonlinearity.** Each row of W is a template the neuron matches. The bias is its firing threshold. Without the nonlinearity, all the layers collapse into one matrix. Modern nets use ReLU instead of sigmoid because it trains much better when deep. (3B1B NN ch.1)
14. **Backprop = recursively passing "desired nudges" backwards.** A weight's change is proportional to (incoming activation) x (how much the receiving neuron must change). An upstream activation's wished change is the sum of (weight) x (downstream need). Average over examples to get the gradient. This is the chain rule as bookkeeping. (3B1B Backprop)
15. **Mini-batch SGD trades gradient accuracy for speed.** Shuffle, batch (~100), and step on each batch. Noisy but fast steps beat slow exact ones, and they also fit streaming or chunked data on 16GB. (3B1B Backprop)
16. **Labelled data is usually the real bottleneck,** not the algorithm. (3B1B Backprop)
17. **Every set of coordinates hides a choice of basis.** Embedding dimensions have no inherent meaning. Distances and angles do, and they survive rotations of the basis. PCA chooses a better basis on purpose. (3B1B ch.2, ch.3)
18. **Approximate first, then let the error vanish.** The same limit argument gives circle area (rings), integrals, derivatives, and why one determinant works for any shape (tiny grid squares). (3B1B Essence of calculus; ch.6)
19. **Prefer solve() over inv(), and treat "singular matrix" errors as a data problem** (collinear or duplicated features), not a library bug. (Strang, Elimination; Geometry)

---

## Gilbert Strang (MIT 18.065) — Lecture 1: The Column Space of A Contains All Vectors Ax (https://www.youtube.com/watch?v=YiqIkSHSmyc)
**Core idea in one line: read Ax as "a weighted mix of A's columns", and every big idea (column space, rank, basis, A = CR, AB as a sum of rank-1 pieces) falls out of that one view.**

- **Mental models / first principles**
  - A matrix is one object that turns a vector into a vector, not a grid of m*n numbers. Strang's request: stop thinking "row dot column" (component at a time, "low level") and start thinking column-wise.
  - Two readings of Ax for A = [[2,1,3],[3,1,4],[5,7,12]]: (a) each output entry is a row of A dotted with x; (b) the whole output is x1*(col 1) + x2*(col 2) + x3*(col 3). Same numbers, but (b) shows you the geometry all at once.
  - **Column space C(A)**: the set of all outputs Ax as x ranges over everything. Also called the *range*. It is always a "space" through the origin: all of R^3, a plane, a line, or just the zero point.
  - **Rank** = number of independent columns = dimension of the column space. Example: columns [1,1,1], [3,3,3], [8,8,8] all lie on one line, so rank 1. In Strang's main example, column 3 = column 1 + column 2, so the columns only fill a plane: rank 2.
  - A *random* square matrix almost surely has independent columns and is invertible. Rank deficiency means structure (a redundancy) in the data.
  - **Row space** = all combinations of the rows = column space of A^T. Keeping everything as column vectors matches the convention in MATLAB, Julia and NumPy.
  - Real data matrices are rarely square: e.g. columns = patients/samples, rows = features/diseases. So the column space and row space live in different spaces (R^m vs R^n).

- **Mechanisms**
  - **Building a basis greedily, left to right**: keep column 1 if it is non-zero; keep each next column only if it is *not* a combination of the columns already kept. The kept columns form C (here 3x2: [2,3,5] and [1,1,7]).
  - **A = C R**: R (2x3) records how to rebuild each column of A from C. Here R = [[1,0,1],[0,1,1]]: col1 = 1*c1 + 0*c2, col2 = 0*c1 + 1*c2, col3 = 1*c1 + 1*c2. R is the *row-reduced echelon form* (with the zero rows dropped): it has the identity in the independent-column positions.
  - **Why column rank = row rank (the "first great theorem")**: read C R two ways. Column view: each column of A is a combination of C's columns, so C's r columns are a basis for C(A). Row view: each row of A is a combination of R's rows (row 1 of A = 2*R-row1 + 1*R-row2, weights taken straight from row 1 of C). R's r rows are independent, so they are a basis for the row space. Both have r vectors, so the two ranks are equal.
  - **Rank-1 matrix = column times row**: [[1,3,8],[1,3,8],[1,3,8]] = [1,1,1]^T times [1,3,8]. Outer product u v^T: column space is the line through u, row space the line through v. These are "the building blocks of data science".
  - **AB as a sum of outer products**: AB = sum over k of (column k of A)(row k of B). Each term is rank 1. This is the matrix-matrix version of "Ax is a mix of columns".
  - **Cost**: (m x n)(n x p) takes m*n*p multiplications whether you do it by dot products (mp entries, n each) or by outer products (n rank-1 pieces, mp each). Same multiplications, just in a different order.
  - **Parentheses trick**: A B C x is in C(A) because it equals A times (BCx). Grouping things the right way is "the key to linear algebra".

- **Decisions & trade-offs**
  - **CR vs CUR**: CR uses real columns of A but R is not made of rows of A. CUR uses real columns *and* real rows of A, with a small mixing matrix U in the middle so the product comes out right. Picking real columns/rows keeps the factors interpretable (they are actual data points) and keeps sparsity.
  - **Sampling a huge matrix**: for a matrix around 10^5 x 10^5 that does not fit in fast memory, don't pick raw columns. Multiply A by random vectors x. Each Ax is a random mix of columns and lies in C(A), and about 100 of them often outline the column space well enough to work with. This is the idea behind randomized SVD / randomized linear algebra.

- **Rules of thumb**
  - When you see Ax, think "combination of columns". When you see AB, think "sum of rank-1 column-times-row pieces".
  - Rank = number of genuinely independent directions in your data. Duplicated or derived features (col3 = col1 + col2) add no rank.
  - A basis needs both properties: the vectors are independent (no extras), and their combinations fill the space.

- **Gotchas**
  - Rank deficiency can be invisible to the eye. Strang notes the row dependency in his example is not obvious, yet it *must* exist because the columns are dependent.
  - Writing a factorization is not the same as checking it. If you swap in different rows for R, A = CR breaks. CUR needs the middle U to fix this.

- **Relevance to Sam**
  - Feature matrix with a derived column (e.g. total_price = base + tax) has rank lower than its column count. Linear models then have no unique solution, and you get multicollinearity. Drop the derived column or regularise.
  - Embeddings: a batch of n embeddings stacked as columns spans at most a d-dimensional column space. Low rank means the embeddings live on a smaller subspace, which is what PCA/SVD exploit.
  - The outer-product view of AB is exactly how low-rank approximations (SVD truncation, LoRA adapters = B A with small inner dimension) save memory. A rank-r piece of an m x n matrix needs r(m+n) numbers instead of mn, which matters on a 16GB laptop.
  - Random projections (multiply by random x's) are a cheap way to sketch a huge location-feature matrix before clustering or deduplication.

## Gilbert Strang (MIT 18.06) — 2. Elimination with Matrices (https://www.youtube.com/watch?v=QVKj3LADCnA)
**Core idea in one line: solving Ax = b is "elimination to an upper-triangular U, then back substitution", and every elimination step is a simple matrix multiplying from the left.**

- **Mental models / first principles**
  - Elimination (Gaussian elimination) is how every numerical package solves linear systems, not determinants and not Cramer's rule. Getting from A to U is called "the most common calculation in scientific computing".
  - The matrix holds all the information. The x, y, z and the plus signs are just notation.
  - **Rows vs columns**: matrix times column vector gives a *combination of the columns*. Row vector times matrix gives a *combination of the rows*: [1,2,7] times A = 1*row1 + 2*row2 + 7*row3. Elimination is all row operations, so the elimination matrices multiply A *from the left*.
  - Matrix on the left acts on rows. Matrix on the right acts on columns. To swap columns, multiply by a permutation *on the right*.

- **Mechanisms (worked example)**
  - System: x + 2y + z = 2; 3x + 8y + z = 12; 4y + z = 2. So A = [[1,2,1],[3,8,1],[0,4,1]] and b = [2,12,2].
  - **Step (2,1)**: pivot 1. Multiplier 3. Row2 minus 3*row1 gives [0,2,-2]. Step (3,1): the entry is already 0, so the multiplier is 0 (software still checks it unless told it is zero).
  - **Step (3,2)**: pivot 2. Multiplier 2. Row3 minus 2*row2 gives [0,0,5].
  - U = [[1,2,1],[0,2,-2],[0,0,5]]. **Pivots 1, 2, 5. Determinant = product of pivots = 10.** You get the determinant for free.
  - **Augmented matrix [A | b]**: apply the same operations to b, giving c = [2, 6, -10]. (MATLAB finishes the left side first, then replays the steps on b.)
  - **Back substitution** on Ux = c, bottom up: 5z = -10 so z = -2; 2y - 2z = 6 so y = 1; x + 2y + z = 2 so x = 2. This works because U is triangular.
  - **Elimination matrices**: E21 = identity with -3 in position (2,1). E32 = identity with -2 in position (3,2). So E32 (E21 A) = U.
  - **Associativity**: E32 (E21 A) = (E32 E21) A. You can multiply the E's first to get one matrix E that does all of elimination. You can move parentheses, but you **cannot change the order** (AB is not BA in general). Strang says many proofs in linear algebra are just "moving the parentheses".
  - **Permutation matrix P**: do the row swap to the identity matrix, and the result is the matrix that swaps those rows. [[0,1],[1,0]] on the left swaps rows. The same matrix on the right swaps columns.
  - **Inverse of an elimination step**: E21 subtracts 3*row1 from row2, so E21^-1 *adds* 3*row1 back: identity with +3 in (2,1). Undoing a simple step is simple. This is why Strang prefers to think about going from U back to A (which leads to A = LU in the next lecture) instead of multiplying the E's together.

- **Decisions & trade-offs / failure modes**
  - **Pivot must be non-zero.** A 0 in the pivot position is only *temporary failure* if some row below has a non-zero entry there: swap rows. Example: if the 8 were a 6, the second pivot spot would be 0, but the 4 below allows a swap.
  - **Complete failure**: a 0 in the pivot position with only zeros below it. Example: if the 1 in position (3,3) were -4, the third pivot would become 0 and nothing could be swapped in. Then the matrix is **singular (not invertible)**: fewer than n pivots, determinant 0, no unique solution.
  - So elimination does two jobs: it solves the system, and it *diagnoses* whether A is a good (invertible) matrix.

- **Rules of thumb**
  - Index each elimination matrix by the position it zeroes: E_ij clears row i, column j.
  - The multiplier for step (i,j) is (entry being cleared) / pivot. E_ij holds *minus* the multiplier.
  - Need a matrix that does an operation X to rows? Apply X to the identity matrix.
  - det(A) = product of pivots (with a sign flip for each row swap).

- **Gotchas**
  - Order matters: E32 E21 is not E21 E32. Left-multiplying cannot swap columns.
  - (note: Strang does exact arithmetic here. Real solvers use *partial pivoting*: they swap in the largest available pivot even when the current one is non-zero, because tiny pivots amplify rounding error.)

- **Relevance to Sam**
  - Linear regression via the normal equations, numpy.linalg.solve, and scipy's LU all use exactly this algorithm under the hood. When you get "singular matrix" errors, a pivot hit zero: you have duplicated or perfectly collinear features.
  - Prefer `solve(A, b)` (LU plus back substitution) over computing `inv(A) @ b`. It is cheaper and more numerically stable.
  - Elimination costs about n^3/3 operations for an n x n matrix. Fine for hundreds of features, not for million-row Gram matrices. For those, use iterative or least-squares methods on chunks.
  - "Left multiply = row ops, right multiply = column ops" is the same mental model as `X @ W` in a neural layer. Each output column of XW is a combination of X's columns, weighted by one column of W.

## Gilbert Strang (MIT 18.06) — 1. The Geometry of Linear Equations (https://www.youtube.com/watch?v=J7DzL2_Na80)
**Core idea in one line: Ax = b has two pictures. The row picture is lines/planes crossing. The column picture is "which mix of A's columns lands on b". The column picture is the one that scales to n dimensions.**

- **Mental models / first principles**
  - Linear algebra's basic problem: solve n linear equations in n unknowns, Ax = b, with A the coefficient matrix, x the unknown vector, and b the right-hand side.
  - **Row picture**: each equation is a line (2D), a plane (3D), or a hyperplane (nD). The solution is where they all meet. Easy in 2D, harder to see with three planes, hopeless in 4D or more.
  - **Column picture**: x1*(column 1) + x2*(column 2) + ... = b. You are looking for the **linear combination** of the columns that produces b. Strang calls the linear combination "the most fundamental operation in the whole course".
  - **The big question**: can Ax = b be solved for *every* b? Put another way: do the combinations of the columns fill the whole space? Yes means A is **non-singular / invertible**. No means the columns lie in a lower-dimensional subspace, and only b's inside that subspace are reachable.

- **Mechanisms (examples)**
  - 2x2: 2x - y = 0, -x + 2y = 3. Row picture: the line through (0,0) and (1,2) meets the line through (-3,0) and (-1,1) at (1,2). Column picture: 1*[2,-1] + 2*[-1,2] = [0,3]. Draw it tip-to-tail: lay down column 1, then add column 2 twice.
  - 3x3: A = [[2,-1,0],[-1,2,-1],[0,-3,4]], b = [0,-1,4]. Row picture: two planes meet in a line, and the third plane cuts that line at one point. Column picture: b equals column 3, so x = (0,0,1) right away. With b = [1,1,-3] = col1 + col2, x = (1,1,0). The columns stay the same, only b changes, which is why the column view is good for "same system, many right-hand sides".
  - **Matrix times vector, two ways**: [[2,5],[1,3]] times [1,2]. Column way: 1*[2,1] + 2*[5,3] = [12,7]. Row way: dot products 2*1 + 5*2 = 12 and 1*1 + 3*2 = 7. Same answer.

- **Decisions & trade-offs / failure modes**
  - **Singular case**: if the three columns lie in one plane (e.g. column 3 = column 1 + column 2), every combination stays in that plane. Most b's are then unreachable, and you can solve Ax = b only when b happens to lie in the plane.
  - In 9 dimensions: 9 random columns almost surely fill R^9 (a random matrix is invertible "guaranteed" in practice). If column 9 duplicates column 8, the combinations fill only an 8-dimensional "plane" inside R^9.
  - The row picture helps with intuition in 2D. The column picture generalises. Elimination (the next lecture) is the systematic solver for either view.

- **Rules of thumb**
  - Always ask two questions: (1) Is b reachable? That is, is b in the span of the columns? (2) What combination reaches it? That combination is x.
  - Independence of the columns is what makes the solution exist *and* be unique.
  - "Ax is a combination of the columns of A." Strang repeats this as the habit to build.

- **Relevance to Sam**
  - A linear model y = Xw is the column picture: predictions are a weighted mix of feature columns. If the target is not in the column space (it almost never is with noisy data), least squares finds the closest reachable point, i.e. the projection of y onto C(X).
  - Duplicated features (the same field coming from two suppliers) make X singular-ish. The model weights become non-unique and unstable.
  - For matching or scoring: a score that is a linear combination of feature columns can only express what lies in their span. To capture interactions you need new columns (features) or nonlinearity (neural nets).

## Grant Sanderson (3Blue1Brown) — But what is a neural network? | Deep learning chapter 1 (https://www.youtube.com/watch?v=aircAruvnKk)
**Core idea in one line: a neural network is just a big function made of repeated "matrix times vector, add bias, squash" steps, and its ~13,000 weights and biases are the knobs that learning tunes.**

- **Mental models / first principles**
  - The problem: writing a program by hand that maps 28x28 pixel grids to digit labels is very hard, because the same "3" can light up completely different pixels. So instead we choose a flexible function family and *learn* its parameters.
  - **Neuron = a thing that holds a number** (its *activation*), here between 0 and 1. A more accurate view: each neuron is a *function* of all the activations in the previous layer.
  - **Architecture (plain vanilla multilayer perceptron)**: 784 input neurons (pixel greyscale values, 0 = black, 1 = white), two hidden layers of 16 neurons each (an arbitrary choice; the size is a design knob), and 10 output neurons (how strongly the network thinks the image is each digit). The prediction is the brightest output neuron.
  - **Why layers might work (the hope)**: hierarchical abstraction. Layer 2 detects little edges, layer 3 combines edges into parts (an upper loop, a long vertical line), and the output combines parts into digits (9 = upper loop + vertical line on the right). The same idea applies to speech: audio, then sounds, then syllables, then words, then phrases. Grant flags this as a *hope*, not a guarantee; chapter 2 checks what the trained network actually learned (spoiler from the series: mostly not clean edges).

- **Mechanisms**
  - **One neuron**: take a weighted sum of all 784 previous activations, add a bias, then pass it through a squashing function.
    - Weights act like a template over the pixels. Picture them as a grid: green = positive, red = negative. To detect an edge in one region, give positive weights there and negative weights to the pixels around it. The weighted sum is then largest when the centre is bright and its surroundings are dark.
    - **Bias** = a threshold shift. Adding -10 means the neuron only becomes meaningfully active once the weighted sum exceeds about 10.
    - **Sigmoid (logistic) function** 1/(1+e^-x): very negative inputs go to about 0, very positive inputs go to about 1, and it rises smoothly around 0.
  - **Parameter count**: layer 1 to 2 has 784*16 weights + 16 biases. Over the whole network that is about 13,000 parameters (784*16 + 16*16 + 16*10 weights, plus 16+16+10 biases, which comes to 13,002).
  - **Matrix form**: a^(1) = sigmoid(W a^(0) + b). Stack the activations into a column vector. Each *row* of W holds the weights into one next-layer neuron. Each output entry of W a is one weighted sum, i.e. a row of W dotted with a. Biases form a vector, and the sigmoid is applied to each entry separately.
  - Writing it with matrices is not just neat. It makes the code simpler and *faster*, because linear algebra libraries are heavily optimised for matrix multiplication (BLAS, GPUs).
  - **The whole network is one function**, R^784 to R^10, with about 13k parameters, built from repeated matrix-vector products and nonlinearities. "Learning" means searching for a setting of those parameters that solves the task (chapter 2 covers how: gradient descent).

- **Decisions & trade-offs**
  - **Sigmoid vs ReLU** (from the guest segment with Lisha Li): sigmoid was motivated by the biological idea of a neuron being on or off. Modern deep nets mostly use **ReLU(a) = max(0, a)**. It is the identity above the threshold and 0 below it. In practice it trains much more easily in very deep networks. (Background, not stated in the video: sigmoid saturates and its gradients vanish.)
  - The number of layers and neurons per layer are free design choices. Grant says there is a lot of room to experiment.

- **Rules of thumb**
  - Read each weight row as "the pattern this neuron is looking for". Read the bias as "how strong the evidence must be before it fires".
  - Understanding what the weights mean gives you somewhere to start debugging when the model misbehaves, or when it works for unexpected reasons. Don't treat the model as a total black box.

- **Gotchas**
  - The nice "edges, then loops, then digits" story is a design hope. Trained networks often do not learn human-interpretable parts. Always check.
  - (note: the sigmoid output layer shown here is old-school. Modern classifiers use softmax outputs with cross-entropy loss.)

- **Relevance to Sam (linear algebra link)**
  - Each layer is a *linear map* (a matrix, which per 3B1B chapter 3 is a transformation of space) followed by a nonlinearity. Without the nonlinearity, stacking layers would just compose into one matrix, i.e. a single linear model.
  - Embeddings that Sam uses for matching are the activations of some hidden layer. A dot product between embeddings is the same operation a neuron does with its weight row: "how well does this vector match this template?"
  - Parameter counting (input_dim * output_dim + output_dim per dense layer) is the quick check for whether a model fits in 16GB RAM. 4 bytes per float32 parameter, plus roughly 2-3x extra for gradients and optimiser state during training.

## Grant Sanderson (3Blue1Brown) — The essence of calculus (https://www.youtube.com/watch?v=WUvTyaaNkzM)
**Core idea in one line: chop a hard quantity into many tiny pieces (integral), notice how the total changes when you nudge the boundary (derivative), and see that the two are inverse operations (fundamental theorem of calculus).**

- **Mental models / first principles**
  - The aim of the series is that you feel you *could have invented calculus*, deriving its rules from pictures instead of memorising formulas.
  - **Respect the symmetry**: to get the area of a circle, slice it into concentric rings. That decomposition matches the circle's shape, and "math tends to reward you when you respect its symmetries".
  - **Approximate first, then make the approximation exact in the limit**: each ring, unrolled, is roughly a rectangle of width 2*pi*r and thickness dr. It is slightly wrong, but the error shrinks as dr shrinks.
  - Many problems are "a sum of many small quantities". Distance travelled = the sum of velocity(t) * dt over many tiny time steps. Whenever the pieces look like thin rectangles side by side, the exact answer is the **area under a graph**.

- **Mechanisms**
  - **Circle area**: ring area is about 2*pi*r*dr, with r running from 0 to R in steps of dr (e.g. 0.1). Stand the rectangles upright along an r-axis. Their heights trace the line y = 2*pi*r (slope 2*pi), and dr is both a factor in each area and the spacing between rectangles. As dr goes to 0, the sum becomes the area under that line: a triangle with base R and height 2*pi*R, so the area is 1/2 * R * 2*pi*R = pi*R^2. (For R = 3, the triangle's height is about 19.)
  - **Integral as a function**: A(x) = area under y = x^2 from 0 to x. This is hard to find directly.
  - **Nudge argument**: increase x by dx. The extra sliver of area dA is about x^2 * dx (a thin rectangle of height x^2 and width dx). So dA/dx is about x^2, and this gets more exact as dx goes to 0. Numeric check: (A(3.001) - A(3)) / 0.001 is about 3^2 = 9.
  - **Derivative** = what the ratio (tiny change in output) / (tiny change in input) approaches as the nudge shrinks. It measures *how sensitive a function is to small changes in its input*.
  - **Fundamental theorem of calculus**: for any area function, its derivative equals the height of the graph. So integrals and derivatives are inverses. If you know a function's derivative should be x^2, you can reverse-engineer the function (it turns out to be x^3/3, which comes later in the series).

- **Decisions & trade-offs / strategy**
  - The choice of decomposition matters. Rings are natural for a circle. A bad slicing makes the sum intractable.
  - **Problem-solving policy for hard questions**: don't attack the answer head-on. Play with the objects (here, the relationship between x^2 and its area function) until a structural clue appears, such as "dA/dx = x^2".
  - The trade-off between approximation and precision: a smaller dr gives more terms but a smaller error per term. The limit removes the error entirely, which is the conceptual leap of calculus.

- **Rules of thumb**
  - Think of dx as "a tiny nudge". Think of the derivative as "the output nudge per input nudge".
  - "Sum of f(x)*dx over small steps" means "area under f", which means an integral.
  - If a total accumulates something, its rate of change is that something.

- **Gotchas**
  - The rectangle approximation is wrong for any finite dr. The claim is only that the error vanishes in the limit. Don't confuse a finite-step numerical sum with the exact integral (relevant to numeric integration and discretised time steps).

- **Relevance to Sam**
  - **Gradient descent is built on the derivative-as-sensitivity idea**: dLoss/dw tells you how much the loss moves per tiny nudge to weight w. Finite-difference checks of the form (L(w + h) - L(w)) / h are exactly Grant's 3.001-vs-3 calculation, and they are how you verify a hand-written gradient.
  - **Integrals as accumulated sums** show up in data work as running totals: cumulative bookings from a booking rate, area under a ROC curve (AUC), and expected values (probability density times value, summed). The fundamental theorem says that if you have the cumulative curve, its slope gives you back the rate (e.g. CDF to PDF in probability).
  - Respecting symmetry when decomposing a problem maps to engineering: partition data along its natural structure (by country or geohash for GeoNames) so each chunk is simple and the results add up cleanly.

## Grant Sanderson (3Blue1Brown) — Backpropagation, intuitively | Deep Learning Chapter 3 (https://www.youtube.com/watch?v=Ilg3gGewQ5U)
**Core idea in one line: backprop works out, for each training example, how strongly each weight and bias should be nudged to reduce the cost. It does this by passing "desired changes" backwards layer by layer, weighted by activations and weights. Averaging those nudges gives the negative gradient.**

- **Mental models / first principles**
  - Recap: learning = finding the weights and biases that minimise a **cost function**. Here the per-example cost is the sum of squared differences between the output vector and the target (one-hot) vector, averaged over all training examples.
  - The **negative gradient** (a vector with about 13,000 components) says how to change every parameter to decrease the cost fastest. Backpropagation is *the algorithm that computes it*.
  - **Read the gradient as a sensitivity list, not a direction in 13k-D space.** If weight A's component is 3.2 and weight B's is 0.1, the cost is 32x more sensitive to A. The same small wiggle to A moves the cost 32 times as much. The relative sizes tell you where the "bang for the buck" is.
  - We can't change activations directly. We can only change weights and biases. But tracking *desired* changes to activations is the bookkeeping that makes backprop work.

- **Mechanisms (one example: an image of a "2", untrained network)**
  - Outputs look random, e.g. 0.5, 0.8, 0.2, and so on. We want the "2" neuron to go up and every other output to go down. **Each nudge is proportional to how far the current value is from its target.** A neuron already near its target barely matters.
  - One output neuron's activation = sigmoid/ReLU(weighted sum of the previous layer + bias). There are **three levers** to raise it:
    1. **Increase the bias.**
    2. **Increase the weights, in proportion to the previous-layer activation.** Connections from bright neurons have the most influence, because the weight multiplies a large activation. This loosely resembles Hebbian learning, "neurons that fire together wire together" (Grant adds caveats: only a loose analogy).
    3. **Change the previous layer's activations, in proportion to the weights**: brighten neurons with positive weights and dim those with negative weights. We can't do this directly, so record it as a *wish*.
  - **The "propagating backwards" step**: every output neuron has its own wish list for the second-to-last layer. Add them up, each weighted by its connecting weight and by how much that output needs to change. That gives the desired nudges for the second-to-last layer. Then apply the same recipe recursively to *its* weights and biases, and keep moving backwards through the network.
  - **Across examples**: one example alone would push the network to call everything a "2". So run backprop for every example and **average the desired nudges**. That average is (proportional to) the negative gradient.
  - **Stochastic gradient descent (SGD)**: summing over every example for each step is too slow. Instead, shuffle the data and split it into **mini-batches** (e.g. 100 examples), then take one step per mini-batch. Each step is only an approximation of the true downhill direction ("a drunk man stumbling quickly down a hill"), but it is much faster overall. Repeating over all mini-batches converges toward a *local* minimum of the cost, i.e. the network does well on the training data.

- **Decisions & trade-offs**
  - **Full-batch vs mini-batch**: full-batch gives the exact gradient but each step is very slow. Mini-batch gives a noisy gradient with cheap steps and much faster wall-clock progress. Mini-batches are the standard.
  - Convergence is to a *local* minimum of the training cost. Good training performance is the claim made here, not generalisation.

- **Rules of thumb**
  - Change in a weight is roughly proportional to (activation feeding into it) x (how much the receiving neuron needs to change).
  - Change wished on an upstream activation is roughly the sum over downstream neurons of (weight) x (that neuron's needed change).
  - Shuffle before batching, so each mini-batch is a fair sample.
  - Every line of backprop code corresponds to one of these intuitive "nudge" effects. The hard part is the notation and index-chasing, not the idea (the next video covers the chain-rule calculus).

- **Failure modes & gotchas**
  - **You need lots of labelled data.** MNIST is a good teaching example because tens of thousands of human-labelled digits exist. Getting labels is a common real bottleneck.
  - Listening to one example (or an unbalanced batch) biases the network toward that class.
  - (note: squared-error cost is used here for simplicity. Classification today normally uses cross-entropy.)

- **Relevance to Sam**
  - This is the **chain rule in action**. Each backward step multiplies local sensitivities (activation x weight x the downstream need). PyTorch autograd does exactly this bookkeeping for you.
  - The "gradient = per-parameter sensitivity" reading is also useful outside neural nets. In a matching-score model (e.g. a weighted mix of name similarity, distance and country match, trained with logistic regression), gradient sizes show which features the loss currently depends on most.
  - Mini-batch SGD fits Sam's 16GB constraint: stream chunks of millions of rows through the model, and never load the full dataset to compute an exact gradient.
  - The label bottleneck matters for supplier/location matching: budget time to build a labelled set of matched and unmatched pairs before training anything.

## Grant Sanderson (3Blue1Brown) — Linear transformations and matrices | Chapter 3, Essence of linear algebra (https://www.youtube.com/watch?v=kYB8IZa5AuE)
**Core idea in one line: a matrix *is* a linear transformation of space. Its columns are where the basis vectors land, and matrix-vector multiplication is "the same linear combination, applied to the landed basis vectors".**

- **Mental models / first principles**
  - "Transformation" is just another word for function (vector in, vector out). The word suggests *movement*: picture every input vector moving to its output.
  - Visualisation trick: draw each vector as the point at its tip, then watch the whole grid of points move. Keep a faint copy of the original grid behind it for reference.
  - **Linear transformation (visual definition)**: lines stay lines (no curving), and the origin stays fixed. Equivalently, **grid lines stay parallel and evenly spaced**.
    - Counter-examples: curving lines (not linear). Shifting the origin, i.e. a translation (not linear: that is *affine*). A transformation that keeps horizontal and vertical grid lines straight but bends diagonals is also not linear, so check diagonals too.
  - **The key consequence**: if v = -1*i-hat + 2*j-hat, then after the transformation v lands at -1*(where i-hat lands) + 2*(where j-hat lands). Linear combinations are preserved. So **you only need to record where the basis vectors go**, and everything else follows.
  - Grant calls this the one topic that makes all the rest of linear algebra click: matrix multiplication, determinant, change of basis and eigenvalues all build on it.

- **Mechanisms**
  - Example: i-hat lands at (1,-2) and j-hat at (3,0). Then v = (-1,2) lands at -1*(1,-2) + 2*(3,0) = (5,2). A general (x,y) lands at x*(1,-2) + y*(3,0) = (x + 3y, -2x + 0y).
  - **A 2D linear transformation = 4 numbers**, packed into a 2x2 matrix whose **columns are the landing spots of i-hat and j-hat**.
  - General formula: [[a,b],[c,d]] times (x,y) = x*(a,c) + y*(b,d) = (ax + by, cx + dy). The memorised textbook formula is just this column picture written out.
  - **Worked matrices**:
    - 90-degree counter-clockwise rotation: i-hat goes to (0,1), j-hat goes to (-1,0), so the matrix has columns (0,1) and (-1,0).
    - Shear: i-hat stays at (1,0), j-hat goes to (1,1).
    - Reading a matrix backwards: columns (1,2) and (3,1) mean "move i-hat to (1,2), move j-hat to (3,1), and drag the grid along evenly".
  - **Dependent columns**: if the two landing vectors are scalar multiples of each other (linearly dependent), the whole plane gets **squished onto a line**, their 1D span. Information is lost and the transformation cannot be undone. (This links to determinant = 0 in chapter 6 and to rank deficiency in Strang.)

- **Decisions & trade-offs**
  - Memorising (ax + by, cx + dy) vs understanding the columns as moved basis vectors: the second makes later topics intuitive. The first leaves you stuck at composition and eigenvectors.
  - Translation is not linear. To include a shift you need affine maps (Wx + b), which is why neural layers carry a separate bias term.

- **Rules of thumb**
  - **Every time you see a matrix, ask "what does it do to space?"** Then look at its columns.
  - To build a matrix for a transformation you want: work out where each basis vector should land and write those as the columns.
  - Linearity test: is the origin fixed, and do evenly spaced parallel grid lines stay that way, including diagonals?

- **Gotchas**
  - Some transformations look linear if you only check the axes. Always check a diagonal line too.
  - Dependent columns mean the map collapses dimension, and you cannot invert it.

- **Relevance to Sam**
  - A neural layer's W x is a linear transformation of the embedding space. The bias adds a shift (affine), and the activation function then bends space, which is the part linear maps can't do.
  - PCA is "pick new basis vectors (the principal directions) and express data in them". The projection matrix's rows and columns are those directions.
  - In geo work: rotating or scaling coordinates (e.g. a local tangent-plane approximation for small areas) is a 2x2 matrix. Converting lat/lon to projected metres is *not* linear globally, which is exactly why naive Euclidean distance on degrees misbehaves far from the equator.
  - Embedding similarity can change after a linear projection (e.g. reducing dimension), especially if the projection squishes along directions that distinguish your matches. Check how the projection's columns relate to the features you care about.

## Grant Sanderson (3Blue1Brown) — The determinant | Chapter 6, Essence of linear algebra (https://www.youtube.com/watch?v=Ip3X9LOh2dk)
**Core idea in one line: the determinant is the factor by which a linear transformation scales area (2D) or volume (3D). It is 0 when space gets squished into a lower dimension, and negative when orientation gets flipped.**

- **Mental models / first principles**
  - Some transformations stretch space and others squish it. The determinant measures *how much*: the scale factor applied to the area of *any* region.
  - **Why one number works for every shape**: grid lines stay parallel and evenly spaced, so every grid square is scaled by the same factor. Any blob can be approximated by tiny grid squares (the same limit argument as in "essence of calculus"), so the blob's area scales by that same factor.
  - **Watch the unit square** spanned by i-hat and j-hat. Its area after the transformation *is* the determinant. In 3D, watch the unit cube. It turns into a **parallelepiped**, whose volume is the determinant.
  - Grant's emphasis: *understanding what the determinant means matters much more than computing it*. Computation is a practice drill and "not the essence".

- **Mechanisms / examples**
  - Columns (3,0) and (0,2): the unit square becomes a 3x2 rectangle, so **det = 6**.
  - Shear with columns (1,0) and (1,1): the square becomes a parallelogram with base 1 and height 1, so **det = 1**. The shape is distorted but area is preserved.
  - det = 3 means areas triple. det = 1/2 means areas halve.
  - **det = 0**: space is squished onto a line or a point in 2D, or onto a plane, line or point in 3D. That means the **columns are linearly dependent**. Testing det = 0 tells you whether the map collapses dimension, i.e. is not invertible.
  - **Negative determinant = orientation flip**, like turning a sheet of paper over. In 2D, j-hat starts to the left of i-hat; if after the transformation j-hat sits on the right, orientation has flipped. |det| is still the area factor. Example: columns (1,1) and (2,-1) give det = -3, so space is flipped and areas scale by 3.
    - Why negative is natural: rotate i-hat steadily toward j-hat. The area shrinks to 0 when they line up, and if i-hat keeps going, the determinant keeps decreasing continuously into negative values.
  - 3D orientation uses the **right-hand rule** (forefinger along i-hat, middle finger along j-hat, thumb along k-hat). If after the transformation this only works with the left hand, det < 0.
  - **2x2 formula: det [[a,b],[c,d]] = ad - bc.** Intuition: if b = c = 0, a and d stretch the axes separately, so the area is ad (a rectangle). If only one of b or c is zero, you get a parallelogram with base a and height d, still area ad. When both are non-zero, the bc term accounts for the stretching or squishing along the diagonal.
  - **Teaser fact: det(M1 M2) = det(M1) det(M2).** The one-sentence reason: applying two transformations one after the other scales area by the first factor and then by the second.

- **Decisions & trade-offs**
  - Use the determinant as a *yes/no test for invertibility* (det != 0) and as a volume-scaling factor. Don't use it as a measure of "how close to singular" a matrix is: a determinant can be tiny just because of units or scale. (note: in numerical practice, the condition number or the smallest singular value is the right closeness-to-singular measure. This is outside the video.)
  - Computing determinants by hand: practise a few, then let software do it. Strang adds that software gets the determinant as the product of the pivots from elimination, not from cofactor formulas.

- **Rules of thumb**
  - |det| = how much volume is scaled. sign(det) = whether orientation is kept (+) or flipped (-).
  - det = 0 if and only if the columns are dependent, if and only if the map squashes dimension, if and only if there is no inverse, if and only if Ax = b is not solvable for every b.
  - The determinant of a product is the product of the determinants. So if any factor is singular, the product is singular.

- **Gotchas**
  - Determinants are signed. Use the absolute value when you mean an amount of area or volume.
  - A determinant near 0 in floating point is not a reliable singularity test for big matrices (see the note above).

- **Relevance to Sam**
  - **Probability / ML**: when you change variables (e.g. normalising flows, or transforming a Gaussian), densities get multiplied by 1/|det J|, the inverse of the volume scaling of the Jacobian J. The multivariate Gaussian's normalising constant includes det(covariance), the volume of the data's spread.
  - **PCA / covariance**: det(covariance) = product of the eigenvalues = "generalised variance". If it is about 0, your features have a near-linear dependency, so drop or merge features.
  - **Geo work**: the 2x2 cross-product determinant (x1*y2 - x2*y1) gives signed polygon area (the shoelace formula) and the orientation test (is point C left or right of line AB?). Both are core to point-in-polygon and spatial joins such as Sam's ferry spatial join.

## Grant Sanderson (3Blue1Brown) — Linear combinations, span, and basis vectors | Chapter 2, Essence of linear algebra (https://www.youtube.com/watch?v=k7RM-ot2NWY)
**Core idea in one line: coordinates are scalars that stretch basis vectors. Everything you can reach by scaling and adding a set of vectors is their span, and a basis is a set of non-redundant (independent) vectors that spans the space.**

- **Mental models / first principles**
  - Read the coordinates (3, -2) as *instructions*: stretch i-hat by 3, flip and stretch j-hat by 2, then add. A vector is a **sum of scaled basis vectors**.
  - **Basis vectors** = the things your coordinates scale. i-hat and j-hat (unit vectors along x and y) are the standard basis, but **any choice of basis is implicit in every set of numbers you write down**. Pick two other non-aligned vectors (one pointing up-right, one down-right) and you get a different but perfectly valid coordinate system. The same numbers then mean a different vector.
  - **Linear combination** = a*v + b*w. Why "linear": fix one scalar, let the other vary, and the tip traces a straight line. (Grant notes this is his intuition, not the word's actual origin.)
  - **Span** = the set of all linear combinations, i.e. everything reachable using only vector addition and scalar multiplication.
  - **Arrows vs points**: think of a single vector as an arrow, and a *collection* of vectors as the points at their tips. So "the span is a line" means the set of tips forms a line through the origin.

- **Mechanisms**
  - **Span of two 2D vectors**: usually the whole plane. If they line up (are collinear), just a line through the origin. If both are zero, just the origin.
  - **Span of two non-aligned 3D vectors**: a flat sheet (plane) through the origin. Picture turning two knobs (the scalars) and watching the tip sweep out the sheet.
  - **Adding a third 3D vector**:
    - If it already lies in that plane, the span doesn't grow. The vector is redundant.
    - If it points out of the plane (almost certain for a random vector), scaling it sweeps the plane through all of space, so the span becomes all of R^3. Three free scalars give access to three dimensions.
  - **Linearly dependent**: at least one vector can be removed without shrinking the span. Equivalently, one is a linear combination of the others. **Linearly independent**: each vector adds a new dimension.
  - **Formal basis**: a set of *linearly independent* vectors that *span* the space. Spanning means enough vectors to reach everything. Independent means no wasted ones.

- **Decisions & trade-offs**
  - Choice of basis is a modelling decision. The standard basis is convenient, but a basis aligned with the structure of the data (principal components, eigenvectors) can make the problem simpler. Grant previews change of basis for later chapters.
  - Redundant vectors cost you: they add parameters but no reach.

- **Rules of thumb**
  - Ask "what is the span?" to mean "what outputs can this set of building blocks produce?"
  - The number of independent vectors = the dimension of their span. For matrix columns, that is the rank (Strang's lecture 1).
  - Random vectors are almost always independent. Dependence comes from structure, such as duplicated or derived quantities.

- **Gotchas**
  - Numbers without a stated basis are ambiguous. The same coordinates describe different vectors in different bases.
  - Two vectors that look different can still be collinear (one a scaled copy of the other). That is the "unlucky" case, where the span collapses to a line.

- **Relevance to Sam**
  - **Embeddings**: each embedding dimension is a coordinate along some (learned) basis direction. The dimensions are not inherently meaningful. Only directions and distances are, and any rotation of the basis preserves cosine similarity and Euclidean distance.
  - **PCA** = choose a new basis whose first few vectors span most of the data's variance. Then drop the rest: keep a smaller span that still reaches almost every data point.
  - **Feature engineering**: a derived feature that is a linear combination of existing ones (a sum, difference or weighted total) is in the span of those features and adds nothing to a linear model. Only nonlinear features, or features with genuinely new information, expand the span.
  - **Supplier/location matching**: when combining scores linearly, independent signals (name similarity, geodistance, category match) each add a dimension. Two highly correlated signals (two string-similarity metrics on the same field) are nearly dependent and mostly duplicate each other.
