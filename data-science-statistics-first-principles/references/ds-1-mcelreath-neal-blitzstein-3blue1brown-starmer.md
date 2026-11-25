# ds-1 — cross-cutting principles

1. **No causes in, no causes out.** Data alone can't tell you causes; write a causal model (DAG) before choosing a statistical model or control variables. Each causal question may need a different model. (McElreath, L01 Golem of Prague)
2. **Controls can hurt.** Some covariates block confounding (good controls), others create bias (bad controls); "add everything" is unsafe. Choose the adjustment set from the DAG. (McElreath L01; Brady Neal, Potential Outcomes)
3. **Association is not an effect.** E[Y|T=1] - E[Y|T=0] equals the ATE only under exchangeability. Randomisation makes it true by design by cutting every arrow into treatment. (Brady Neal)
4. **Identification before estimation.** First turn the causal estimand into a statistical one using explicit assumptions (unconfoundedness, positivity/overlap, no interference, consistency); then estimate it, e.g. with the adjustment formula: predict outcome under T=1 and T=0 for every row and average. Unconfoundedness is untestable; more covariates make it more plausible but erode overlap exponentially. (Brady Neal)
5. **A regression coefficient as "the effect" silently assumes a constant effect for everyone** (linear form). Prefer flexible outcome models plus averaging. (Brady Neal)
6. **Simulate, then estimate.** Write a generative simulation, check the estimator recovers known truth (extremes, large-n, parameter sweeps), use the simulator to plan sample size, and only then touch real data. Treat analysis code as tested software. (McElreath L01 and L02)
7. **Bayesian updating is counting.** Plausibility of each explanation is proportional to the number of ways it could produce the data; for a proportion, p^W(1-p)^L, giving Beta(W+1, L+1) under a flat prior. No minimum sample size, the posterior carries n, and you can update without the raw data. (McElreath L02)
8. **The estimate is the whole distribution.** Push posterior samples (not a point) through every downstream calculation; intervals (50/89/95%) are arbitrary summaries and nothing happens at their edges. (McElreath L02)
9. **Model measurement error and selection.** Label noise biases proportions toward 0.5 *and* makes naive estimates overconfident. A sample needn't be representative if you model how it was selected and reweight. (McElreath L02)
10. **Null hypotheses: useful in clean experiments, weak in complex systems.** In a two-arm randomised test, "no difference" is the natural, data-free hypothesis. You reject or fail to reject; you never prove equality. In observational science the null is not unique, and rejecting it does not pick out your mechanism. (Starmer, Hypothesis Testing; McElreath L01)
11. **What p and alpha mean.** With alpha = 0.05 and no true difference, about 5% of experiments give false positives. Pick alpha by the cost of being wrong (0.2 for trivial calls, far smaller for high-stakes claims). Run many tests and you will get "significant" noise. (Starmer, p-values)
12. **p-value is not effect size.** Big samples make tiny effects significant and small samples hide big ones. Always report the effect with its uncertainty. (Starmer, p-values; McElreath L02 on interval superstition)
13. **Variances add; SDs scale with sqrt(n).** SD of a sum is sigma*sqrt(n) and SD of a mean is sigma/sqrt(n), so halving error needs 4x data. (3Blue1Brown CLT)
14. **CLT: means of IID finite-variance draws are approximately normal**, whatever the source shape. This licenses CIs, t-tests and ANOVA on means. It fails or converges slowly with dependence, non-identical draws, or heavy tails (infinite variance). "n >= 30" is only a rule of thumb; check by simulation or bootstrap on your own data. (3Blue1Brown; Starmer CLT)
15. **Linearity of expectation + indicators.** E[sum] = sum of E, even with dependence; E[indicator] = P(event). So expected counts (correct matches, duplicates, successes) are sums of probabilities. Use symmetry and first-step analysis for the rest. (Blitzstein, Stat 110 L9)
16. **Bias-variance and held-out evaluation.** Flexible models (squiggles, neural nets) cut bias but raise variance. Judge only on held-out data; perfect training fit is a red flag. Regularisation, bagging and boosting find the sweet spot. (Starmer, Bias-Variance; Starmer, Neural Networks)
17. **Neural nets are just composed bent lines.** Weights and biases reshape activation functions, which are summed into a curve. Use them for genuinely non-linear relationships, at the cost of data and interpretability. (Starmer, Neural Networks)
18. **PCA = centre, then find perpendicular max-variance directions.** Loadings say which variables drive each axis; the scree plot says how much a 2-D view can be trusted. Standardise variables on different scales first. (Starmer, PCA)

---

## Richard McElreath — Statistical Rethinking 2023 - 01 - The Golem of Prague (https://www.youtube.com/watch?v=FdnMWdICdRs)
**Core idea: statistical models are mindless "golems"; only an explicit causal model (DAG, then a generative simulation) tells you which model to fit, which controls to add, and what the numbers mean.**

- **Mental models / first principles**
  - "No causes in, no causes out" (Nancy Cartwright). A dataset alone cannot reveal its causes; for any sample an analysis can "find" whatever cause it is pointed at. You must bring an external causal model.
  - Association is bidirectional and carries no direction; causation is about interventions. Correlation does not imply causation, and causation does not even imply correlation (an effect can be hidden). Prefer the word "association": correlation is only one narrow linear measure of it.
  - Two faces of causal inference: (1) predicting the consequences of an intervention ("if I add wind, the trees sway; shaking the tree does not make wind"), which differs from pure prediction (seeing swaying trees, predict wind); (2) causal imputation: reconstructing unobserved counterfactuals ("what would have happened if ...").
  - Description is not a causal-free zone. If you want to describe a population from a sample, the sample was *caused* by something (who answered, who got scraped, who booked), and you need a causal story of why the sample differs from the population to correct for it (post-stratification).
  - Research design also needs causal knowledge of the system.
- **Mechanisms: DAGs**
  - A directed acyclic graph holds only variable names and arrows. Arrow X -> Y: intervening on X changes Y, not the reverse. No functional forms, no linearity assumption; by default everything may interact.
  - Roles relative to treatment X and outcome Y: competing causes of Y (B), causes of the treatment (A), common causes of X and Y (C = confound). The relationships among A, B, C also matter for which controls work.
  - Each causal question (effect of X on Y, effect of A on Y, ...) generally needs a *different* statistical model / adjustment set. One regression rarely answers all questions; reading every coefficient of one big regression as causal is a classic error (the "Table 2 fallacy", not named here but implied).
  - Good controls exist (they block confounding) and bad controls exist (adding them creates bias). "Throw everything in and see what happens to the coefficients" is unsafe. In his example DAG the correct adjustment set is {B, C}, not all variables.
  - DAGs are also "intuition pumps": they pull you out of the numbers and into the science, and their logical implications are testable.
- **Golems (statistical models)**
  - Powerful, no wisdom, execute instructions blindly; same category as ML and AI systems. Harm comes from using them in the wrong context.
  - The intro-stats flowchart of tests (t-test, Spearman, etc.) is a "zoo" of narrow golems suited to quality control. The t-test came from Gosset at Guinness for small-batch consistency checks: perfectly good for that.
  - Null-hypothesis rejection is a weak research goal in observational systems because null models are not unique. Examples: neutral vs selection models in population genetics (a fluctuating-selection model predicts the same distribution as the neutral-equilibrium null); permutation nulls for species co-occurrence and social networks (no unique way to permute a matrix; low power plus high false positives); Neanderthal DNA (rejecting "zero Neanderthal DNA" is consistent with both interbreeding and ancient-substructure process models).
  - Lesson: rejecting a null does not pick out your preferred process. You must write multiple process models and find where their predictions differ.
- **Decisions & trade-offs**
  - Bayes is chosen for practical, not philosophical, reasons. For simple linear regression it is "cutting a birthday cake with a chainsaw" (works, more mess). It pays off for measurement error, missing data, latent variables, regularisation: routine problems in real work. Bayesian models are generative, so they map closely onto the causal model.
  - The Bayes-vs-frequentist "war" is treated as over; the real question is how you justify a procedure.
- **Workflow ("drawing the owl", don't skip steps)**
  1. Define the estimand: what exactly are you trying to learn?
  2. Build a scientific/causal model: start as a DAG, make it generative (can simulate data).
  3. Use 1 and 2 to design statistical model(s) justified by the causal model.
  4. Simulate synthetic data from the generative model and check the estimator recovers the known truth.
  5. Only then analyse real data. Loop back and revise, documenting everything.
  - Reuse the simulation code to compute hypothetical interventions.
  - Treat analysis scripts as software: version, test, comment, be able to rerun them. Avoid point-and-click analysis.
- **Failure modes**
  - Choosing controls without a causal model; adding every available covariate.
  - Treating "rejected null" as confirming a specific mechanism.
  - Leaving measurement error and missing data to "later": they are in nearly every project.
  - Unreproducible analysis code ("I'm not sure which script it was").
- **Relevance to Sam**
  - "Did the change work?" is an intervention question; write the DAG first (seasonality, marketing pushes, supplier price moves, traffic mix are common causes of both the change and bookings).
  - Descriptive stats on scraped/supplier/GeoNames samples: model why your sample differs from the target population (e.g. coverage biased to popular destinations) before generalising.
  - Matching pipelines: build a synthetic set with known true matches, check the matcher/estimator recovers them, then run on real data (step 4 of the owl).
  - Honest reporting: state the causal assumptions explicitly so others can critique them.

## Richard McElreath — Statistical Rethinking 2023 - 02 - The Garden of Forking Data (https://www.youtube.com/watch?v=R1vcdhPBlXA)
**Core idea: Bayesian inference is just counting: for each possible explanation, count the ways it could have produced the observed data, normalise to probabilities, and carry the whole distribution (not a point) into every downstream calculation.**

- **Setup and generative model**
  - Estimand: p, proportion of Earth covered by water, estimated by tossing a globe and recording water (W) or land (L) under your finger, N tosses.
  - DAG: N -> W, N -> L, p -> W, p -> L. Intervening on N or p changes W/L; fabricating W/L does not change N or p. p is unobserved (drawn circled).
  - Write the generative function first from what you know about the sampling process; the estimator falls out of it.
- **Mechanism: garden of forking data**
  - Toy "4-sided globe": possible p in {0, .25, .5, .75, 1}. Observed W, L, W. Ways each p produces it: 0x4x0=0, 1x3x1=3, 2x2x2=8, 3x1x3=9, 0. Any hypothesis that makes one observation impossible gets zero.
  - Things that happen together multiply; independent tosses mean the garden branches the same way each time.
  - Updating = multiply existing counts by the ways each hypothesis produces the new observation. After the 9-toss sample (6 W, 3 L): 0, 27, 512, 729, 0. General form for continuous p: ways proportional to p^W (1-p)^L.
  - Normalise counts to probabilities by dividing by their sum; counts explode fast (30 tosses at p=.5 gives over a billion paths), probabilities don't. Over infinitely many p the sum becomes an integral and the result is the Beta distribution; the factorial term is just the normalising constant, the shape comes entirely from p^W(1-p)^L. With continuous p you talk of density, not probability.
  - With a flat prior and one W the posterior is a straight line rising in p; each new toss reshapes it; it gets taller and narrower as evidence accumulates.
  - Conditional on the generative model being correct, this estimator is optimal: no other procedure uses the data better or is more honest about uncertainty ("small world" optimality; if the model is wrong, all bets are off).
- **Properties that conflict with frequentist intuitions**
  - No minimum sample size: n=1 gives a valid, appropriately vague posterior. The posterior already encodes the sample size and structure, so you can update from the posterior alone without the raw data.
  - No point estimates: the estimate *is* the distribution. Mode and mean are communication summaries only; compute predictions, causal effects, interventions over the *entire* posterior, never from a single point.
  - No "true" interval: 50%, 89% (his habit), 95% are arbitrary summaries of shape. Nothing special happens at an interval boundary. "95% is a superstition"; whether zero falls inside an arbitrary interval says nothing about robustness. Robustness is a property of the estimator and model, not of the interval width. NHST's only job is Type I error control, which is not estimation.
- **Workflow and testing (apply to every project)**
  - Write a simulation function (sim_globe(p, N)) and an estimator function (compute_posterior(sample, possibilities)); test the estimator on simulated data where the truth is known. "If you test nothing, you miss everything."
  - Tests: extremes where intuition is certain (p=1 must give only water; p=0 only land); large-N check (N=10,000, sample proportion should be close to p); sweep across values of p; check the posterior converges to truth as N grows and is appropriately wide when N is small.
  - Rubber-duck debugging: explain your code out loud.
  - Use the simulator for design: decide how many tosses you need to reach a target precision before collecting data.
- **Sampling from the posterior (the practical engine)**
  - Draw e.g. 1,000 samples (rbeta(1e3, W+1, L+1)); any calculation you can do for one value of p you can then do for all samples, which averages over uncertainty and replaces integral calculus. 1,000 is usually enough; draw more for a smoother approximation. MCMC later returns only samples, so this skill carries over.
  - Posterior predictive distribution: for each sampled p, simulate a new experiment with the same simulator (e.g. 10 more tosses), pool the outcomes. It is wider than the predictive distribution for any single p because it includes parameter uncertainty. Use it for forecasts, counterfactuals, prior predictions.
- **Bonus: misclassification / measurement error**
  - If the recorder flips the label with rate x, the observed W* depends on true W and the measurement process M. P(observe W) = p(1-x) + (1-p)x; P(observe L) = (1-p)(1-x) + p x. Plug these into the same likelihood. Alternatives that cannot co-occur add; sequential events multiply.
  - The naive model (ignores misclassification) is overconfident *and biased*: misclassification pushes the estimate toward 0.5, so the aware posterior is wider and shifted outward (to the right in his example). With known x and growing N, the aware estimator recovers the true p.
  - Same structure applies to diagnostic tests, labelling, any classifier output used as data.
  - Also model missing data, non-compliance, inclusion/selection. A sample does not need to be representative to estimate a population; you need to model *how* it differs and reweight (post-stratification). Sometimes the bias is unrecoverable; modelling the selection process tells you which case you are in.
  - Subjective inputs (assumptions) + objective machinery; any method claiming more is hiding assumptions.
- **Relevance to Sam**
  - Matching-quality audits: if you hand-label a sample of N matched pairs and see W correct, the posterior Beta(W+1, N-W+1) directly gives the plausible range of the true match precision; you can stop and report after any number of labels (no minimum n). Use sim to decide how many labels you need for, say, +/-3% width.
  - If your "ground truth" labels are themselves noisy (human labellers, heuristic rules, scraped flags), the naive accuracy estimate is biased toward 50% and overconfident; model the label error rate.
  - Booking-conversion or click rates are the same globe-tossing problem; report the full posterior or a stated interval, not "significant or not".
  - Scraped/supplier data is never a random sample of the world; model the inclusion process before claiming population numbers.

## Brady Neal — Potential Outcomes, Week 2 (https://www.youtube.com/watch?v=5x_pPemAVxs)
**Core idea: a causal effect is a difference between potential outcomes you can never both observe; you identify it from data only by invoking four explicit assumptions (unconfoundedness, positivity, no interference, consistency), then estimate it with the adjustment formula.**

- **Mental models / first principles**
  - Potential outcomes: Y_i(1) is unit i's outcome if treated, Y_i(0) if untreated. Individual causal effect = Y_i(1) - Y_i(0) (headache pill: 1 - 0 = 1).
  - Fundamental problem of causal inference: only one potential outcome is ever observed (the factual); the other is a counterfactual. Equivalently, causal inference is a missing-data problem: a table of Y(1), Y(0) columns with a "?" in every row.
  - Average treatment effect ATE = E[Y(1) - Y(0)] = E[Y(1)] - E[Y(0)] (linearity of expectation). The tempting substitute E[Y|T=1] - E[Y|T=0] is the *associational difference*, not a causal quantity.
  - Why they differ: confounding. Sleeping with shoes on is associated with morning headaches because drinking the night before causes both. The treated and untreated groups are not comparable (treated group is mostly drunk people).
- **Mechanisms: assumptions for identification**
  - Identification = turning a causal estimand (with potential outcomes) into a statistical estimand (conditional expectations of observed data). Estimation = turning a statistical estimand plus data into a number. Keep these two steps separate in your head (identification-estimation flowchart).
  - Ignorability / exchangeability: (Y(1), Y(0)) independent of T. Then E[Y(1)] = E[Y|T=1], so the associational difference equals the ATE: you can "ignore the question marks". Exchangeability view: swapping which group got treated would not change the expected outcomes.
  - Randomised controlled trials make exchangeability true by design: a coin flip assigns T, which deletes every arrow into T (removes confounding). Note the treatment definition changes ("slept with shoes on" not "went to bed with shoes on"): the experimenter controls it.
  - Conditional exchangeability / unconfoundedness: (Y(1), Y(0)) independent of T given covariates X. Identifies the conditional ATE: E[Y|T=1,X] - E[Y|T=0,X]. Marginalise over X to get the ATE: the **adjustment formula** ATE = E_X[ E[Y|T=1,X] - E[Y|T=0,X] ].
  - Unconfoundedness is **untestable**: an unobserved confounder W could always exist.
  - Positivity / overlap / common support: 0 < P(T=1|X=x) < 1 for every x present in the population. Rewritten with Bayes' rule, the adjustment formula has P(T=1|x) in a denominator; violation means dividing by zero. Intuition: if everyone with X=x was treated, you have no data on what untreated looks like for them.
  - Under positivity violations, models f1(x), f0(x) for each arm must extrapolate into regions with no data, giving garbage estimates.
  - No interference (SUTVA part): my outcome depends only on my treatment, not others' (counter-example: friends getting dogs affects my happiness).
  - Consistency / no multiple versions of treatment: observed Y equals Y(t) when T=t. Violated when "treated" hides variants (golden retriever vs chihuahua both coded as "got a dog" give different outcomes).
  - Full identification proof uses all four: no interference, linearity, iterated expectations, unconfoundedness + positivity, consistency.
- **Decisions & trade-offs**
  - Unconfoundedness vs positivity tension: conditioning on more covariates makes unconfoundedness more plausible but shrinks overlap exponentially (50% overlap in 1-D becomes 25% in 2-D, and so on: curse of dimensionality).
  - Regression coefficient as ATE: if you assume Y = alpha*T + beta*X (linear), the coefficient alpha on T is the ATE and matches the adjustment-formula estimate. But that linear form silently assumes the effect is the *same for every unit* (no heterogeneity). Generally prefer the adjustment formula with a flexible outcome model (any model minimising MSE of Y given T and X) then average predictions over the empirical X. (See Morgan and Winship sections 6.2-6.3 for critique.)
- **Worked example (simulated, true ATE = 1.05)**
  - Effect of sodium intake (binarised at a threshold) on systolic blood pressure, covariates age and urinary protein. Naive regression of Y on T: 5.33 (about 407% off). Adjustment formula with linear regression on T and X: 0.85 (about 19% off). The coefficient on T in a Y ~ T + X regression also gives 0.85; same if T stays continuous.
  - Estimation recipe: fit E[Y|T,X] with a model; for each row predict with T=1 and T=0; average the difference over all rows (the empirical mean replaces E_X).
- **Failure modes**
  - Reporting E[Y|treated] - E[Y|untreated] from observational data as an effect.
  - Claiming "we controlled for everything": unconfoundedness can never be verified from data.
  - Adjusting for many covariates and silently extrapolating outside overlap.
  - Fuzzy treatment definitions (multiple versions) and spillovers between units.
- **Relevance to Sam**
  - "Users who saw the new Drive Button booked more" is an associational difference; heavy readers/high-intent visitors may be more likely both to see it and to book. Randomise exposure (A/B) whenever possible; it is the only way to make exchangeability true by construction.
  - When you can't randomise, write down X (device, traffic source, destination, season), check overlap (are there treated and untreated units in each segment?), then use the adjustment formula: predict outcome under both arms for each row and average.
  - Interference is real in marketplaces: a pricing change for some users can change availability/prices for others; caching or shared inventory breaks unit independence. Consider cluster/time-based randomisation.
  - Consistency: define "treatment" precisely (which button variant, which supplier version) before analysis.

## Joe Blitzstein (Harvard Stat 110) — Lecture 9: Expectation, Indicator Random Variables, Linearity (https://www.youtube.com/watch?v=LX2q356N2rU)
**Core idea: linearity of expectation (true even under dependence) plus indicator variables turns hard expected-value problems into "count the indicators, multiply by one probability".**

- **CDF recap (mental model)**
  - CDF F(x) = P(X <= x), defined for all real x even for discrete X. For a discrete variable it is a step function; jump sizes are the PMF.
  - The CDF fully determines the distribution: P(a < X <= b) = F(b) - F(a). For discrete variables, strict vs non-strict inequalities matter; for continuous they don't.
  - Valid CDF iff (1) non-decreasing, (2) right-continuous (closed dot on the upper value at each jump), (3) tends to 0 at minus infinity and 1 at plus infinity.
- **Independence of random variables**
  - X, Y independent iff the joint CDF factors: P(X<=x, Y<=y) = P(X<=x)P(Y<=y) for all x, y. Discrete case: joint PMF factors, P(X=x, Y=y) = P(X=x)P(Y=y). Slogan: "independence means multiply". Intuition: knowing X tells you nothing about Y.
- **Expectation as a weighted average**
  - Averaging five 1s, two 3s, one 5: sum/8 equals the grouped weighted average (5/8)(1)+(2/8)(3)+(1/8)(5). Expectation is the same idea with probabilities as weights: E[X] = sum over x of x P(X=x).
  - Mean of 1..n is (n+1)/2 (average first and last of any arithmetic series; Gauss's 1..100 = 5050).
  - Expectation is more than a one-number centre summary: variance and other spread measures are themselves expectations of functions of X.
- **Mechanisms / key tools**
  - Bernoulli(p): E[X] = p.
  - **Fundamental bridge**: for an indicator I_A (1 if event A occurs, else 0), E[I_A] = P(A). Any probability can be rewritten as an expected value and vice versa.
  - **Linearity**: E[X+Y] = E[X] + E[Y] and E[cX] = cE[X], *always*, even when X and Y are dependent (proof promised next lecture).
  - Binomial(n,p) mean: the direct sum over k with the committee-chair identity k C(n,k) = n C(n-1,k-1) and the binomial theorem gives np; the better way: X = sum of n Bernoulli(p) indicators, so E[X] = np by linearity (independence not even needed).
  - Hypergeometric example: aces in a 5-card hand. X = sum of 5 indicators (card j is an ace); by symmetry each has P = 4/52 = 1/13; E[X] = 5/13. The indicators are dependent (if four aces already, the fifth can't be), yet linearity holds. So hypergeometric has the same mean as the matching binomial (variance differs).
  - Strategy combo: **indicators + linearity + symmetry + fundamental bridge**.
  - Geometric(p) (his convention: number of failures before first success; some books count the success, so check conventions): PMF q^k p, k = 0,1,2,...; sums to 1 via geometric series. E[X] = q/p.
    - Calculus route: differentiate sum q^k = 1/(1-q) to get sum k q^(k-1) = 1/(1-q)^2, multiply by q, times p, gives q/p.
    - Story / first-step route: let c = E[X]. With prob p first trial succeeds (0 failures); with prob q you get one failure and the problem restarts (memoryless). c = p*0 + q(1 + c), solve: c = q/p.
- **Rules of thumb**
  - Before brute-forcing a sum, ask: can X be written as a sum of indicators? Then E[X] = sum of probabilities.
  - Symmetry arguments ("no reason card 2 differs from card 5") collapse n terms to n times one term.
  - First-step analysis: condition on the first outcome and use self-similarity.
  - Watch convention differences (geometric with or without the success; first-success distribution has mean 1/p).
- **Gotchas**
  - Linearity gives the mean, not the variance: for variance, dependence matters (hypergeometric variance is smaller than binomial).
  - E[X] alone can hide a complicated distribution; one number isn't enough.
- **Relevance to Sam**
  - Expected number of correct matches in a matched dataset = sum over rows of P(row is correct match). If a matcher outputs calibrated probabilities, summing them gives the expected count of true matches without labelling everything, even though rows are dependent (e.g. one-to-one constraints).
  - Expected duplicates/collisions (e.g. how many place pairs share a name) are sums of pair indicators: C(n,2) times the per-pair probability.
  - Geometric distribution models "attempts until first success": retries until an API call succeeds (expected failures q/p, so with 20% failure rate per call, p=0.8, expected failures 0.25), or sessions until first booking.
  - Conversion rate as the mean of a 0/1 indicator: the fundamental bridge is why averaging booking flags gives the booking probability.

## 3Blue1Brown (Grant Sanderson) — But what is the Central Limit Theorem? (https://www.youtube.com/watch?v=zeJD6dqJ5lo)
**Core idea: sum many independent draws from the same finite-variance distribution, standardise by mean n*mu and SD sigma*sqrt(n), and the shape converges to the standard normal no matter what the starting distribution looked like.**

- **Mental model**
  - A single event (one ball on a Galton board, one die roll) is unpredictable, but the distribution of sums of many such events is highly predictable.
  - Idealised Galton board: each peg adds +1 or -1 with 50/50; final bucket = sum of those choices. More rows = bigger sum = closer to a bell curve.
  - Works for any starting distribution: a die skewed toward low values, or one with nearly all mass on 1 and 6. Sums of 2 still look like the original; sums of 10 look mostly bell-shaped (slight skew, or spiky for the extreme 1/6 die); by sums of about 50 the starting shape is essentially washed out.
- **Mechanisms**
  - Distribution of a sum of two independent variables = convolution: for each target sum, add up products of probabilities of all pairs that produce it (weighted version of counting dice pairs on diagonals).
  - Mean mu = E[X] (centre of mass). Variance = E[(X - mu)^2]; squaring keeps signs positive and makes the maths nice but gives squared units; SD sigma = sqrt(variance) is in the original units.
  - Means add: sum of n draws has mean n*mu (distributions drift right).
  - **Variances add, not SDs** (for independent variables): Var(sum of n) = n*sigma^2, so SD = sigma*sqrt(n). Spread grows only with the square root of n.
  - Standardise: Z = (sum - n*mu) / (sigma*sqrt(n)) = "how many SDs from the mean is this sum". Draw histograms with area (not height) = probability so they line up with a density.
  - Normal density construction: e^(-x^2) gives a smooth two-sided decay (e^(-|x|) has a sharp peak); a constant in the exponent rescales width (any base works; e is chosen so sigma reads as the SD); area under e^(-x^2) is sqrt(pi), so normalise by 1/(sigma*sqrt(2*pi)); subtract mu to shift. Standard normal: (1/sqrt(2 pi)) e^(-x^2/2). With a continuous density you ask for the probability of an interval (area), never of a point.
  - Formal statement: P(a < Z_n < b) tends to the integral of the standard normal density from a to b as n goes to infinity.
- **Worked example (practical recipe)**
  - Sum of 100 fair die rolls. Single die: mu = 3.5, variance 2.92, sigma 1.71. Sum: mean 350, SD 10 x 1.71 = 17.1. 68-95-99.7 rule: 95% range is roughly 350 +/- 2 x 17.1, about 316 to 384. Only two numbers (mu, sigma) of the base distribution were needed.
  - Divide by 100 to get the sample mean: centred at 3.5 with SD sigma/sqrt(n) = 0.171. The CLT tells you how close the sample average will be to the expected value; that spread shrinks as 1/sqrt(n) (this is the standard error).
- **Three assumptions (and how they break)**
  1. Independence between the terms.
  2. Identically distributed (1 and 2 together = IID).
  3. Finite variance. Heavy-tailed distributions with infinite variance can be valid and occur in practice, and their sums need not converge to a normal.
  - The real Galton board violates 1 and 2 (each bounce depends on the previous trajectory; pegs differ), yet looks roughly normal; generalisations relax assumption 2 especially. But do not assume normality without justification: many people do.
- **Rules of thumb**
  - SD of a sum scales sqrt(n); SD of a mean scales 1/sqrt(n). To halve the error, quadruple the sample.
  - 68/95/99.7% within 1/2/3 SD for normal variables.
  - "n = 10" may not be enough if the base distribution is lopsided; n around 50 usually is for bounded variables (his visual demo; heavier skew needs more).
- **Relevance to Sam**
  - Average booking value per session, daily revenue, mean API latency over many calls: the *mean* is approximately normal for large n even though individual values are skewed, so mean +/- 2 SE intervals and z/t tests on means work.
  - But travel data is often heavy-tailed (a few huge group bookings, latency spikes, extreme prices): finite-variance can be practically violated, so means converge slowly and a single whale can dominate. Consider medians, trimmed means, log-transforms, or bootstrap, and check by simulation.
  - Independence breaks in practice: sessions from the same user, rows from the same supplier, days with shared seasonality. Clustered data has a larger effective SE than sigma/sqrt(n); aggregate by the unit of randomisation.
  - Planning precision: to get the mean conversion within +/- 0.5 percentage points at 95%, solve 2*sigma/sqrt(n) = 0.005.

## Josh Starmer (StatQuest) — Principal Component Analysis (PCA), Step-by-Step (https://www.youtube.com/watch?v=FgakZw6K1QQ)
**Core idea: PCA centres the data and finds successive perpendicular directions that capture the most spread; projecting onto the first few gives a low-dimensional picture whose faithfulness you read off the scree plot.**

- **Mental model**
  - Rows = samples (mice, students, businesses, hotels), columns = variables (genes, test scores, features). Beyond 3 variables you can't plot; PCA compresses to 2 axes where similar samples cluster together, tells you which variables drive the separation, and how accurate the 2-D picture is.
- **Mechanism (via SVD)**
  1. Compute the mean of each variable; shift so the centre sits at the origin (relative positions are unchanged).
  2. Consider any line through the origin; project each point onto it.
  3. Goodness of fit: by Pythagoras, for a fixed point (fixed distance to origin), minimising its distance to the line is the same as maximising the distance from its projection to the origin. PCA maximises the latter (easier to compute): sum of squared projected distances (SS).
  4. Rotate to the line with the largest SS: that is PC1. In the example its slope is 0.25 ("4 parts Gene 1, 1 part Gene 2"): data are spread mostly along Gene 1.
  5. A PC is a **linear combination** of the original variables. Scaled to unit length (divide by sqrt(4^2+1^2)=4.12) the PC1 vector is (0.97, 0.242): the singular vector / eigenvector; its entries are **loading scores** (how much each variable contributes).
  6. Eigenvalue for a PC = SS of projections divided by (n-1) (a variance); singular value = sqrt(SS).
  7. PC2 is perpendicular to PC1 (in 2-D, no further optimisation): recipe (-0.242, 0.97). In higher dimensions each next PC is the best-fitting line perpendicular to all earlier ones.
  8. PCA plot: rotate so PC1 is horizontal, place each sample at its projected coordinates.
- **Reading the output**
  - Proportion of variation per PC = eigenvalue / sum of eigenvalues (example: 15 and 3 gives 83% and 17%). The **scree plot** shows these percentages.
  - 3-gene example: PC1 79%, PC2 15%, PC3 6%; 2-D plot keeps 94%. A 4-gene example with PC1+PC2 = 90% is also fine for 2-D.
  - If later PCs (PC3, PC4) carry a lot of variation, the 2-D plot is a poor representation, though it can still hint at clusters.
  - Number of PCs = min(number of variables, number of samples).
- **Decisions & trade-offs / gotchas**
  - PCA captures linear structure and variance, not necessarily what matters for your outcome; high-variance directions can be noise.
  - (note: not covered in this video but essential: PCA is scale-sensitive. Variables in large units dominate PC1; standardise variables (z-scores) when they're on different scales, e.g. price in naira vs rating out of 5.)
  - Loadings tell you which original variables drive each axis; check them before naming a cluster.
- **Relevance to Sam**
  - Exploratory view of high-dimensional travel data: hotels or destinations described by price, rating, review count, distance to centre, amenities; project to 2-D to spot clusters, duplicate entities, or outlier suppliers.
  - Embedding diagnostics: PCA on name/address embeddings from a matching pipeline to visualise whether matched pairs sit close and non-matches separate; the scree plot tells you how much a 2-D view can be trusted.
  - Dimension reduction before clustering or nearest-neighbour matching on a 16GB laptop: keep the top k PCs explaining, say, 90-95% of variance to cut memory and compute. Incremental/randomised PCA works in chunks.

## Josh Starmer (StatQuest) — The Essential Main Ideas of Neural Networks (https://www.youtube.com/watch?v=CqOfi41LfDw)
**Core idea: a neural network is a "big fancy squiggle-fitting machine": weights and biases slice, flip and stretch copies of one bent activation function, then add them up and shift the result to fit non-linear data.**

- **Problem it solves**
  - Drug dosage example: low dose ineffective (0), medium effective (1), high ineffective (0). No straight line fits all three; a straight line can get at most two right. You need a curve that rises and falls: a "squiggle".
- **Mechanism (fully worked toy network: 1 input, 2 hidden nodes, 1 output, softplus activation, dosage scaled 0-1)**
  - Top hidden node: x = dosage * (-34.4) + 2.14; y = softplus(x) = log(1 + e^x). Dosage 0 gives x = 2.14, y = 2.25; dosage 0.1 gives x = -1.3, y = 0.24. Sweep 0-1 to get a curve, then multiply by output weight -1.3 (dosage 0 gives -2.93).
  - Bottom hidden node: x = dosage * (-2.52) + 1.29; dosage 0 gives y = 1.53. Multiply the resulting curve by +2.28.
  - Output: add the two reshaped curves, then add the final bias -0.58. Dosage 0.5 gives 1.03, closer to 1, so "effective".
  - Each node uses only a narrow window of the activation function (the input range 0-1 maps into a small x-range), so different weights expose different parts of the same curve.
  - Terminology: multipliers are **weights**, additive terms are **biases**; nodes between input and output are **hidden layers**; bent functions are **activation functions** (softplus, ReLU, sigmoid). "log" means natural log.
  - Parameters are like slope/intercept in a line fit; they are estimated by backpropagation (covered in Part 2).
- **Decisions & trade-offs**
  - Activation choice: sigmoid is common in teaching; ReLU and softplus are more common in practice.
  - Architecture (number of hidden layers and nodes) is mostly guess-and-check from rules of thumb: start small, add layers/nodes if performance is poor.
  - More nodes/layers let you fit almost any squiggle (universal approximation in spirit), which is also the overfitting risk (see bias-variance).
- **Gotchas**
  - The "neural" name is historical (1940s-50s analogy to neurons and synapses); there is no brain magic, just compositions of simple functions.
  - Inputs were scaled to 0-1 to keep the maths simple; input scaling matters in real networks.
- **Relevance to Sam**
  - Use NNs when the relationship is genuinely non-linear (e.g. booking probability vs lead time or price rises then falls). But for tabular travel data, a simple model or gradient boosting is often a stronger baseline; a NN's flexibility costs data, tuning time and interpretability.
  - Mental model for explaining to non-technical colleagues: "it adds up a few bent lines into a curve that matches the data".
  - Any flexible fit must be judged on held-out data (cross-validation), not training fit.

## Josh Starmer (StatQuest) — Hypothesis Testing and The Null Hypothesis, Clearly Explained (https://www.youtube.com/watch?v=0oc49DyA3hU)
**Core idea: because random, uncontrollable differences between subjects make every result wobble, we test the one hypothesis that needs no preliminary data, "no difference", and either reject it or fail to reject it; we never "accept" it.**

- **Mental model**
  - Same virus, same drug, different recovery times: diet, exercise, stress, pollution, etc. act as noise you can't control. Any single experiment's difference between groups is partly noise.
  - Drug A vs B: preliminary data (n=3 per arm) suggests A is 15 hours faster; replications repeatedly show the *opposite* (e.g. A 35 hours slower), after re-checking labels. Strong contrary evidence: **reject** the hypothesis.
  - Drug C vs D: first experiment says C is 13 hours faster; replications give 12 and 13.5. Data are close but not identical. You can't confirm "exactly 13" because 12, 13.5, 12.25, 13.1 are all equally supported; the best you can do is **fail to reject**.
- **Why the null hypothesis**
  - A specific hypothesis (13 hours) is arbitrary: it just copies one noisy result, and requires preliminary data (you wouldn't know whether to test 13 or 13,000 hours). The null hypothesis ("no difference", value 0) is the single non-arbitrary value and needs no pilot data.
  - Drugs E vs F: 0.5-hour difference favouring F; small random changes in a few subjects could flip it to 0.25 hours favouring E. The data don't overwhelmingly argue against "no difference", so fail to reject.
  - With many subjects, small random factors barely move the result, so a real difference can be confidently detected (sample size drives the ability to reject).
- **Decision rule**
  - Reject the null if the observed difference would be hard to explain by random subject differences (e.g. so many more fast recoveries with C than D that it's implausible all the healthy people landed in C).
  - Fail to reject if little random things could shift the result back and forth.
  - The alternative hypothesis is covered in a separate video.
- **Gotchas**
  - Mislabelling and procedure errors are real alternative explanations for surprising results: double-check before believing a reversal.
  - "Fail to reject" is not "proved equal": it means the data are not decisive.
  - (note: McElreath's lecture warns that in complex observational systems the null is not unique and rejecting it doesn't identify a mechanism; "no difference" is cleanest in randomised two-arm experiments like this one.)
- **Relevance to Sam**
  - A/B test of a booking-flow change: the natural null is "no difference in conversion". Tiny n (a handful of sessions or one day) yields results that flip direction on replication: do not act on them.
  - Before celebrating a surprising result in a pipeline metric (match rate jumped), check for labelling/join bugs first: the "mislabelled drug" explanation is common in data engineering.
  - Report non-significant results as "inconclusive at this sample size", not "no effect".

## Josh Starmer (StatQuest) — p-values: What they are and how to interpret them (https://www.youtube.com/watch?v=vemZtEM63GY)
**Core idea: a p-value threshold controls how often you'd get a false positive when there is truly no difference; a small p-value says "probably different", never "how different".**

- **Mental model**
  - One patient per drug tells you nothing (allergy, drug interaction, missed dose, placebo, mislabelling). Two per drug barely more. You need many subjects so random oddities average out.
  - Obvious case: drug A cures 1043 of 1046 (99.7%), drug B cures 2 of 1434 (0.1%): chance can't plausibly explain it. Ambiguous case: 37% vs 31% cured: here you need a p-value.
- **Mechanism / correct interpretation**
  - p-values lie in [0, 1]; smaller means stronger evidence against "no difference". (note: the video loosely calls this "confidence the drugs are different". Strictly, p is the probability of data at least this extreme *if the null were true*; it is not the probability that the null is true.)
  - Threshold 0.05: if there is no real difference and you repeat the experiment many times, about 5% of runs give p < 0.05. Those are **false positives**.
  - Demonstration: give the *same* drug to two groups. Usually p is large (e.g. 0.9). Occasionally, all the allergic people land in one group and the strong placebo responders in the other, giving p = 0.01 even though there is no difference.
  - The ambiguous 37% vs 31% example gives p = 0.24: not confident they differ.
- **Choosing the threshold (trade-off)**
  - High-stakes claims: use something like 0.00001 (false positive about 1 in 100,000 null experiments).
  - Low-stakes decisions ("will the ice-cream truck be on time?"): 0.2 is fine (2 in 10).
  - 0.05 is common because pushing false positives lower usually costs more (bigger samples) than it's worth. It is a convention, not a law.
  - Terminology: deciding whether drugs are the same = hypothesis testing; null hypothesis = the drugs are the same; p-value helps decide to reject it or not.
- **Biggest gotcha: p-value is not effect size**
  - A 6-point difference with few people gave p = 0.24; a 1-point difference with many more people gave p = 0.04. Small p can come with a tiny effect, large p with a big but noisy one. Always report the effect size and its uncertainty alongside p.
- **Relevance to Sam**
  - With large traffic, trivial conversion changes (0.05 percentage points) become "significant". Decide in advance the minimum effect worth shipping, and judge results by effect size and interval, not p alone.
  - Running the same comparison daily, or across many segments/destinations, means about 5% will show p < 0.05 by chance alone: expect false positives and correct for multiple comparisons or pre-register the main metric.
  - Pick the threshold by cost of a wrong call: a reversible UI tweak can tolerate 0.1-0.2; a pricing or supplier-switch decision deserves stricter evidence.

## Josh Starmer (StatQuest) — The Central Limit Theorem, Clearly Explained (https://www.youtube.com/watch?v=YAlJCEDH2uY)
**Core idea: "even if you're not normal, the average is normal": sample means drawn from almost any distribution are approximately normally distributed, which is what licenses confidence intervals, t-tests and ANOVA on means.**

- **Mechanism (simulation)**
  - Uniform(0,1): repeatedly draw samples of size 20, compute each sample's mean, histogram the means. By about 100 means the histogram is clearly bell-shaped and matches an overlaid normal curve. The means are *not* uniformly distributed.
  - Exponential (strongly skewed): same procedure, same result: the means are normally distributed, not exponential.
  - Claim: regardless of the source distribution, means of samples are (approximately) normal.
- **Why it matters in practice**
  - You often don't know the data's distribution. The CLT says you don't need to for inference on means: use the normal distribution of the sample mean to build confidence intervals, run t-tests (two means), ANOVA (three or more means), and basically any test built on the sample mean.
- **Rules of thumb**
  - Folk rule "n >= 30" is a generally safe heuristic, not a requirement; his demos use n = 20 successfully.
  - Fine print: the distribution must have a mean. He cites the Cauchy distribution as the example without one and says he has never met it in 20 years of biostatistics.
  - (note: this understates the risk. As the 3Blue1Brown video states, the classic CLT also needs finite *variance*. Heavy-tailed distributions with a mean but infinite variance (e.g. Pareto with tail index between 1 and 2) converge slowly or to non-normal limits. Strongly skewed real data, such as revenue per user, can need far more than 30 observations. Independence is also required.)
- **Relevance to Sam**
  - Mean booking value or mean session revenue per variant can be compared with t-tests/intervals even though individual values are exponential-like, provided n is reasonably large and there isn't an extreme whale tail.
  - Quick sanity test for your own data: bootstrap or simulate sample means at your actual n and look at the histogram; if it's still skewed, n is too small for normal-theory intervals.
  - Per-user aggregation first (one mean per user) keeps the "independent draws" assumption closer to true.

## Josh Starmer (StatQuest) — Machine Learning Fundamentals: Bias and Variance (https://www.youtube.com/watch?v=EuBBz3bI-aA)
**Core idea: bias is a model's inability to capture the true relationship; variance is how much its fit changes across datasets; the best model sits in the sweet spot, which you can only see on held-out test data.**

- **Setup**
  - Predict mouse height from weight. True relationship rises then plateaus (heavier mice get fatter, not taller). Split data into training (blue) and test (green) sets.
- **Mechanism**
  - Linear regression (least squares) fits a straight line: can never bend to follow the arc, so it has relatively **high bias**.
  - A very flexible "squiggly line" passes through every training point: training sum of squared residuals = 0, **low bias**.
  - Fit quality is measured by sum of squared distances (squared so signs don't cancel).
  - Training-set contest: squiggle wins. Test-set contest: straight line wins; the squiggle does terribly.
  - **Variance** (ML usage): the difference in fit across datasets. The squiggle's error swings wildly between train and test (high variance, unpredictable future performance). The straight line's errors are similar on both (low variance): "consistently good, not great".
  - Fits training data well but test data poorly = **overfit**.
- **Decisions & trade-offs**
  - Ideal: low bias (captures the real shape) and low variance (consistent across datasets), found between simple and complex models.
  - Three standard tools for finding the sweet spot: **regularisation**, **boosting**, **bagging** (random forests are bagging).
- **Rules of thumb / gotchas**
  - Never judge a model on training error alone; always hold out data (or cross-validate).
  - Perfect training fit is a warning sign, not a success.
  - A biased but stable model can beat a flexible one on new data, especially with small or noisy datasets.
- **Relevance to Sam**
  - Matching/classification models (e.g. a place-name matcher with many hand-crafted features): a model that hits 100% on the labelled sample you tuned on may be overfit to it. Keep a separate untouched labelled set to report real precision/recall, and evaluate on data from different regions/suppliers to test variance.
  - Demand or price models trained on one season can fail next season: test on a later time period (time-based split), not random rows.
  - For small labelled sets, prefer simpler/regularised models; flexible models need much more data.

