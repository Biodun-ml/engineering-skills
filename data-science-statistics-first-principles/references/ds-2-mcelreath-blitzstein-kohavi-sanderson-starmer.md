# ds-2 — cross-cutting principles

1. **The causes aren't in the data.** Forks, pipes and colliders can all produce the same scatter plot. The direction of arrows comes from domain knowledge and the interventions you'd consider, so draw a DAG before choosing covariates. [McElreath L04, L05]
2. **Which covariates to adjust for depends on the variable's role.** Fork (common cause): adjust. Pipe (mediator): don't adjust for the total effect, do adjust for the direct effect. Collider (and its descendants): never adjust. "Control for everything" can *create* bias. [McElreath L05]
3. **Post-treatment bias ruins experiments in the analysis stage.** Don't add consequences of the treatment (other than the outcome) as covariates. About half of published political-science experiments did. [McElreath L05]
4. **Selection is conditioning on a collider.** Datasets of only funded, hired, surviving, booked or matched items show correlations that aren't in the full pool. Sample before the filter. [McElreath L05 — grants, restaurants]
5. **The estimate is a computed contrast, summarised last.** Take differences sample-by-sample (posterior or bootstrap), *then* compute means and intervals. Overlapping intervals don't mean "no difference", and an interval containing 0 doesn't mean "zero effect." [McElreath L04, L05]
6. **Simulate first.** Write a generative simulation with a known effect, intervene in it (do-operator: delete incoming arrows, set values) to get the true answer, and check your estimator recovers it before touching real data. [McElreath L04, L05]
7. **Use flexible local curves, not polynomials.** Polynomials are global, force symmetry and explode at the edges. Splines/GAMs smooth locally. Mechanistic models beat both when you have one. [McElreath L04]
8. **Agree on an OEC that is short-term measurable, predicts long-term value, and has an agreed *direction*.** Watch for proxies the treatment can game or reinterpret (revenue per email ignoring unsubscribes, clicks on a link that now hides the price, cannibalised widgets). [Kohavi]
9. **Most ideas fail (about ⅓ positive, ⅓ flat, ⅓ negative; 80–90% fail in mature areas), and experts can't predict which.** Ship through experiments, don't ship flat features, and replicate surprising wins. Only about 1 in 5,000 Bing experiments truly moved the north-star metric. [Kohavi]
10. **Trust the experimentation system only after validating it.** Run A/A tests (p-values should be uniform), check sample-ratio mismatch with a chi-square test, filter bots, and use the delta method for ratio metrics whose analysis unit differs from the randomisation unit. [Kohavi]
11. **Twyman's law: any figure that looks interesting or different is usually wrong.** Debug a surprising result before celebrating it (defaults, daylight saving, logging bugs). [Kohavi]
12. **Size experiments in advance: power, alpha, effect size give n.** Required n scales with 1/effect². Topping up after p = 0.06 is p-hacking. Small sites can only detect big effects. [Starmer Power; Kohavi]
13. **Larger samples shrink the sampling spread of the mean (≈1/√n), and the CLT makes this hold regardless of the underlying distribution.** That is the source of power. [Starmer Power]
14. **Evidence updates beliefs; it doesn't replace them.** Posterior ∝ prior × likelihood. A 4× likelihood ratio loses to a 20× base rate (Steve is probably a farmer). Base-rate neglect is the default human error. [3Blue1Brown; Starmer Bayes]
15. **Think in natural frequencies.** "40 out of 100" stops the conjunction fallacy (85% -> 0% error in the Linda task). Use counts when explaining results to people. [3Blue1Brown]
16. **Base rates make "significant" results and "match" flags less trustworthy than they look.** A low prior (rare true effects, rare true duplicates) means many positives are false. Compute precision from the base rate. [3Blue1Brown; Kohavi]
17. **Condition on the smallest thing that simplifies the problem (first-step analysis), and generalise the question to expose the recursion.** Small per-step edges compound: p = 0.49 with $100 each gives about a 2% chance of survival. [Blitzstein L7]
18. **Equal-likelihood counting needs a justified symmetry.** "Two possibilities" doesn't mean 50/50. Count with trees and the sampling table, and check on tiny cases. Combinatorial blow-up is why entity matching needs blocking. [Blitzstein L1]
19. **Variance reduction by aggregating varied models, with honest validation.** Random forests use bootstrap plus random feature subsets plus voting, and OOB error is free validation. It is optimistic when rows are near-duplicates or time-dependent. [Starmer RF]
20. **Separate the probability model from the decision.** Logistic regression gives P(y=1). The classification threshold, and "points" in general, are decisions driven by costs. Report probabilities and distributions; "points are decisions, not estimates." [Starmer Logistic; McElreath L04]

## Richard McElreath — Statistical Rethinking 2023, Lecture 04: Categories & Curves (https://www.youtube.com/watch?v=F0N4b7K_iYQ)
**One generative model can carry several questions (estimands). Each question may need a differently built statistical model, and the answer is almost always a contrast you compute from the whole posterior, not a number you read off a summary table.**

- **Mental models / first principles**
  - Linear regression works like the geocentric model: wrong about mechanism, very good at approximation. It can fit nearly any shape, so the *design* (what you stratify by) has to come from a causal model outside the regression (a DAG plus a generative simulation).
  - "The causes aren't in the data." A scatter plot can't tell you which way the arrows point. Decide direction by asking which interventions you would actually consider. Example: making someone taller makes them heavier, but losing weight doesn't make anyone shorter, so H -> W.
  - Mediation triangle: S -> H -> W plus S -> W. Sex affects weight in two ways: directly, and indirectly through height. **Total effect** and **direct effect** are different estimands and need different models.
  - Unobserved causes behind each variable can be ignored *unless they are shared* by two measured variables. A shared one is a confound. Example: in turtles, temperature drives both sex and body weight. Whenever you leave a variable out, you are making a strong assumption about which arrows it would add.
- **Mechanisms**
  - Workflow: (1) draw the DAG; (2) write a generative simulation as functions (H = f(S), W = f(H,S)) with noise; (3) find the true causal effect by *intervening in the simulation*: make everyone S=1, then everyone S=2, and difference the means. In his example the total effect was about 21 kg; (4) fit the estimator to simulated observational data and check it recovers that value; (5) only then run it on real data.
  - **Index variables vs indicator (dummy) variables.** Index coding gives category k its own parameter alpha[k]. Advantages: the same code works for 3 or 1,000 categories; every category gets the same prior symmetrically; it carries straight over to multilevel models. Indicator coding makes one category the baseline and the others "adjustments", which makes symmetric priors awkward.
  - **Total effect model:** W ~ alpha[S]. **Direct effect model:** W ~ alpha[S] + beta[S]*(H - Hbar). Stratifying by height blocks the S->H->W path. He avoids the word "control" because control is something you do in an experiment, not in statistics.
  - **Centering** (H - Hbar): alpha then means expected weight at average height. Priors are easier to set and fitting behaves better. Least-squares lines always pass through the point (mean x, mean y).
  - Direct-effect contrast: for each height, simulate expected weight for S=1 and S=2 from the posterior and take the difference. The resulting "bow tie" straddles zero, so almost all of the sex->weight effect runs through height.
  - "Full luxury Bayes": fit the whole DAG as one joint model (a weight submodel plus a height submodel), then simulate interventions p(W | do(S)) from it. This is equivalent to fitting one model per estimand. You write fewer models but do more simulation afterwards.
- **Decisions & trade-offs**
  - **Polynomials vs splines.** Polynomials are *global*: a data point anywhere can bend the curve far away from it. They force symmetry (a parabola must turn back), their uncertainty explodes at the edges, and they extrapolate absurdly. His example: a quadratic fit of weight on height predicts babies under ~50 cm get *heavier* as they get shorter. He says never use them.
  - **B-splines / GAMs / Gaussian processes** are sums of local basis functions times weights. Each weight changes only its own region of x. Use them when you need a flexible trend and have no mechanistic model. Prefer a mechanistic model when you have one (e.g. growth phases with monotonic constraints). Splines know no domain constraints, so they can produce impossible wiggles outside or between the data.
  - Age as a "cause": usually it stands in for accumulated causes over time. That's useful, but be clear about what you mean.
- **Rules of thumb**
  - Always compute the contrast. **Overlapping distributions do not mean "no difference."** Two correlated parameters can overlap heavily while their difference is reliably negative.
  - Summarise *last*. Take differences sample-by-sample across the whole posterior, then compute means and intervals. Differencing the means first gives wrong uncertainty.
  - Mean contrast vs individual contrast: men's mean weight is about 5–9 kg higher (reliable), yet a random man is heavier than a random woman only about 82% of the time. Report the one that matches the decision.
  - "Points are not estimates; points are decisions." The estimate is the whole distribution.
- **Failure modes**
  - Letting software build dummy variables silently, so you never learn what the model is doing.
  - Extrapolating any flexible curve beyond the data.
  - Calling a regression coefficient "the effect" when the estimand is total but the model conditions on a mediator, or the other way round.
- **Relevance for Sam**
  - Categorical stratification (market, supplier, device) should use index coding plus contrasts. To ask whether a supplier changes conversion *directly* or only through price, build separate total-effect and direct-effect models, with price as the mediator.
  - To judge whether a change worked, compute the posterior or bootstrap distribution of the *difference*. Don't eyeball overlapping confidence intervals.
  - For seasonality or trend lines in booking data, use splines/GAMs, not polynomial trend terms.
  - Simulate fake data with a known effect first and check that the analysis recovers it. This is cheap and catches most pipeline bugs.

## Richard McElreath — Statistical Rethinking 2023, Lecture 05: Elemental Confounds (https://www.youtube.com/watch?v=mBEA7PKDmiY)
**Every confound, however large the DAG, is built from four three-variable shapes: fork, pipe, collider, descendant. Whether you should adjust for a variable depends entirely on which shape it sits in, and the data alone can't tell you which shape that is.**

- **Mental models / first principles**
  - "Correlation is common in nature; causation is sparse." Examples: happiness vs metal bands per capita (Finland tops both); Waffle Houses vs divorce rate (both Southern); Maine divorce vs margarine consumption (r = 0.99). Time series make spurious correlation especially easy.
  - Estimand (the cake you want), estimator (the recipe), estimate (what came out). Confounding is any feature of the sample, or of how you analyse it, that makes the estimate differ from the estimand. He uses "confound" in its plain-English sense.
  - All four shapes can produce very similar scatter plots. The causal structure has to come from domain knowledge.
- **The four elemental confounds** (X = treatment, Y = outcome, Z = third variable)
  1. **Fork X <- Z -> Y** (common cause). X and Y are associated. *Condition on Z* and they become independent. In the simulation, r = 0.63 marginally and about 0 within each level of Z. Example: median age at marriage drives both marriage rate and divorce rate. Stratify by age to estimate marriage rate -> divorce.
  2. **Pipe X -> Z -> Y** (mediator). It looks the same statistically: associated marginally, independent given Z. But for the *total* effect of X you must **not** condition on Z. Example: plant experiment where antifungal treatment -> fungus -> growth. Adding fungus as a covariate removes the treatment's main pathway and makes a working treatment look useless. This is **post-treatment bias**. A survey found about half of published political-science experiments conditioned on post-treatment variables. Rule of thumb: don't put consequences of the treatment (other than the outcome) into the estimator. Condition on the mediator only when the estimand is the *direct* effect.
  3. **Collider X -> Z <- Y.** X and Y are independent marginally but *become associated when you condition on Z*. Simulated with a threshold: Z=1 if X+Y is large. Within each Z level X and Y are correlated, with opposite signs in the two strata.
     - *Selection colliders*: the sample already conditions on Z. Grants funded on newsworthiness plus rigour (independent in the applicant pool) show a negative correlation once you look only at funded grants. The most newsworthy funded grants had below-average rigour. Similar: great restaurants in bad locations (survival = food OR location), and attractive vs skilled actors.
     - *Endogenous colliders*: you create one by adding a variable to the model. Simulated example: age and happiness both cause marriage, and age has *zero* effect on happiness. Adding marital status as a "control" makes happiness appear to decline with age, inside both the married and unmarried strata.
     - Post-treatment covariate X with an unobserved U -> X and U -> Y: conditioning on X can make a treatment with no effect look effective.
  4. **Descendant**: a variable A caused by Z. Conditioning on A partly conditions on Z, so its effect depends on which shape Z belongs to. In the pipe example, conditioning on a descendant of the mediator *halved* the X–Y correlation (about 0.6 to about 0.3), which is a wrong estimate. Most measured variables are proxies, i.e. descendants of what you really care about. That also makes them useful: proxies of an unobserved confound can help de-confound (latent variables, measurement-error models, factor analysis).
- **Mechanisms**
  - With a continuous Z, "stratifying" means the regression gives each value of A its own intercept (alpha + beta_A*A), and beta_M is measured against that.
  - **Standardise** variables (z-scores) so that priors are easy to set and fitting is better behaved.
  - **Prior predictive simulation**: Normal(0,10) priors on standardised slopes produce absurd lines (one predictor explaining far more than all the variance). Use alpha ~ N(0, 0.2) and beta ~ N(0, 0.5). These allow strong positive, strong negative or flat relationships but rule out impossible ones.
  - Causal effect as **intervention**, p(D | do(M)): delete all arrows *into* M, set M to chosen values, keep the observed distribution of other causes (resample the 50 states' ages with replacement into 1,000 virtual states), simulate D for M=0 and M=1 SD, and take the difference. In simple linear models this matches the slope. In nonlinear or interaction models the estimand is never a single coefficient, so simulate.
  - For the total effect of A, drop M, because A -> M -> D is a pipe.
- **Decisions & trade-offs / rules of thumb**
  - Before adding any covariate, ask what shape it sits in. Forks: include. Mediators: exclude for total effects, include for direct effects. Colliders and their descendants: exclude. Proxies: use with care.
  - **An interval that includes zero does not show the effect is zero.** "Zero is not a special point that annihilates all other values." Here beta_M straddled zero but stretched well into negative values.
  - Running an experiment doesn't protect you. Randomisation can be undone in the analysis by conditioning on post-treatment variables.
  - Keep a small data list with only the variables the estimand needs.
- **Failure modes**
  - "Control for everything" regressions (the "causal salad").
  - Reading post-selection datasets (funded, hired, survived, published, *booked*) as if they were the population.
  - Using a noisy proxy for a mediator or confound and assuming it behaves exactly like the real thing.
- **Relevance for Sam**
  - **A/B tests:** never add post-treatment metrics (e.g. "pages viewed", "searched again", "clicked a supplier") as covariates when estimating a feature's effect on bookings. They are mediators or colliders. Pre-treatment covariates only.
  - **Selection colliders in travel data:** analysing only completed bookings, only listings that survived dedup, or only matches above a threshold can create correlations (e.g. price vs rating, distance vs name similarity) that don't exist in the full pool. Sample from the pre-filter population when you sanity-check matching quality.
  - **Forks:** season or destination often drives both a feature's exposure and conversion. Stratify by it.
  - When explaining findings, say which estimand was estimated (total vs direct) and report uncertainty without turning "CI includes 0" into "no effect."

## Joe Blitzstein (Harvard Stat 110) — Lecture 7: Gambler's Ruin and Random Variables (https://www.youtube.com/watch?v=PNrqCdslGi4)
**Condition on something simple, usually the first step, to turn a hard problem into a recursion. A random variable is a deterministic function from outcomes to numbers; all the randomness comes from the experiment.**

- **Mental models / first principles**
  - The course in five words: "Conditioning: the soul of statistics", and "random variables and their distributions."
  - When stuck, ask "what do I wish I knew?", then condition on the *smallest* thing that makes the problem simpler. Don't condition on the whole future.
  - Solving the general version can be *easier* than solving a specific one. Asking for P(A wins | starts with i) for every i exposes the recursion. A single question like "start with $100 of $212" hides it.
  - Learn to recognise structure: the gambler's ruin is the same problem as a random walk on 0..N with **absorbing states** at 0 and N. The pattern shows up in finance, physics and repeated success/failure processes.
- **Mechanisms**
  - Setup: A has $i and B has $N−i. Each round is independent with P(A wins) = p and q = 1−p, and $1 moves each round.
  - **First-step analysis** via the law of total probability: p_i = p·p_{i+1} + q·p_{i−1} for 1 ≤ i ≤ N−1, with boundary conditions p_0 = 0 and p_N = 1. This is a *difference equation*, the discrete analogue of a differential equation. Blitzstein argues these deserve more teaching, because real data arrive in discrete time.
  - Solve by guessing p_i = x^i. That gives px² − x + q = 0, whose discriminant 1−4pq = (2p−1)², so the roots are 1 and q/p. The general solution is A + B(q/p)^i. Applying the boundaries gives **p_i = (1 − (q/p)^i) / (1 − (q/p)^N)** for p ≠ q.
  - Fair game (p = q): take the limit with L'Hôpital to get **p_i = i/N**. Your chance of winning equals your share of the total wealth.
  - The game ends with probability 1: P(A wins) + P(B wins) = 1 (by symmetry, swap the names), so nothing is left for endless oscillation. The equations alone don't show this, but the probabilities do.
  - **Random variable** = function X: S -> R. It is a "numerical summary of an aspect of the experiment." The function itself is deterministic. Randomness enters through which outcome s happens. "X = 1" is shorthand for the event {s : X(s) = 1}.
  - **Bernoulli(p)**: values 0/1 with P(X=1) = p. **Binomial(n,p)**: number of successes in n independent Bernoulli(p) trials. PMF: P(X=k) = C(n,k) p^k (1−p)^{n−k}. Derivation: any particular arrangement has probability p^k q^{n−k}, and there are C(n,k) arrangements.
  - If X ~ Bin(n,p) and Y ~ Bin(m,p) are independent, then X+Y ~ Bin(n+m, p). Proof by story: count successes across n trials, then m more. (note: in the lecture he said "Y is binomial n,p" but meant m,p.)
- **Numbers worth remembering** (equal starting stakes, p = 0.49, a 1% edge against A)
  - N = 20: A wins about 40% of the time. N = 100: about 12%. N = 200: about 2%.
  - A tiny per-round disadvantage compounds into near-certain ruin over many rounds, and it is worse still when the other side holds more capital (a casino).
- **Rules of thumb**
  - Prefer **story proofs** to algebra. If you can explain why a result holds by counting or interpretation, you understand it.
  - Write down boundary conditions explicitly. They pin the general solution to the specific answer.
  - Check a limiting case (unfair -> fair should converge smoothly), then verify by plugging the answer back in.
- **Failure modes**
  - Vague definitions ("a variable that takes random values") make later work on distributions incoherent. Keep the random variable, its distribution and its realised value separate.
  - Assuming a process must terminate, or might not, without checking that the absorption probabilities sum to 1.
- **Relevance for Sam**
  - Small per-step edges compound. A 1% worse conversion per funnel step, a 1% match error per pipeline stage, or a slightly unfavourable pricing rule repeated across millions of rows adds up to a large aggregate effect. Model repeated processes explicitly rather than reasoning from one step.
  - First-step analysis and absorbing Markov chains fit funnel or retry modelling: P(booking eventually completes | current step), expected retries before supplier success or timeout.
  - Binomial thinking underlies conversion-rate tests and match-quality audits. Number of correct matches in a random sample of n ~ Bin(n, p_true), which tells you how precise an n-row manual audit can be.

## Joe Blitzstein (Harvard Stat 110) — Lecture 1: Probability and Counting (https://www.youtube.com/watch?v=KbB0FjPg0mw)
**Probability is the logic of uncertainty. The naive definition (favourable outcomes / total outcomes) holds only when outcomes are finite *and* equally likely by a justified symmetry. Counting with the multiplication rule and the sampling table supplies the numbers.**

- **Mental models / first principles**
  - "Math is the logic of certainty; statistics is the logic of uncertainty." Probability is how you quantify beliefs and update them.
  - Probability is counterintuitive even for experts. Newton did a dice calculation correctly but his intuition about it was wrong. Before sets and formal rules, people relied on heuristics, and most of those were wrong. Formalism exists because intuition fails.
  - The modern breakthrough was treating **events as subsets** of a **sample space** S (the set of all possible outcomes of an "experiment", meant very broadly). Unions, intersections and complements become the language of events.
  - Applications are broad, including history: Mosteller–Wallace used Bayes' rule to settle disputed authorship of the Federalist Papers.
  - Much of the skill is **pattern recognition**: learn individual techniques, then practise mixed problems where nobody tells you which technique applies. His analogy is chess tactics.
- **Mechanisms**
  - **Naive definition**: P(A) = |A| / |S|. Two assumptions: finite S, and all outcomes equally likely. Two coin flips give 4 outcomes, so P(TT) = 1/4 *if* the coin is fair and the tosses don't affect each other.
  - Counter-example: "life on Neptune: either there is or isn't, so 1/2." By the same logic "intelligent life" also gets 1/2, which breaks the requirement that a subset event can't be more likely than its superset. Two possibilities do not mean 50/50 without symmetry.
  - **Multiplication rule**: sequential sub-experiments with n1, n2, …, nr outcomes (for each earlier outcome) give n1·n2·…·nr outcomes in total. Visualise as a tree. Ice cream: 2 cones × 3 flavours = 6, in either order. Growth is exponential: 2^10 = 1,024, so listing outcomes stops being feasible quickly.
  - **Binomial coefficient** C(n,k) = n!/((n−k)! k!) counts subsets of size k with order ignored, and is 0 if k > n. Derivation: n(n−1)…(n−k+1) ordered picks, divided by k! orderings.
  - **Full house** (3 of one rank, 2 of another): 13·C(4,3)·12·C(4,2) / C(52,5). Build the numerator as a tree: pick the triple's rank, its suits, the pair's rank (12 left), then its suits. Anchor on a concrete example ("three 7s, two 10s").
  - **Sampling table** (choose k from n):
    - with replacement, order matters: n^k
    - without replacement, order matters: n(n−1)…(n−k+1)
    - without replacement, order doesn't matter: C(n,k)
    - with replacement, order doesn't matter: C(n+k−1, k). This is the hard one (stars and bars). Check it on small n, k.
- **Rules of thumb**
  - Before using equal-likelihood counting, state *why* outcomes are equally likely (physical symmetry, proper shuffling, random sampling). Without a reason, don't use it.
  - Count with a structured tree, not ad hoc. When in doubt, check the answer a second way.
  - Verify formulas on tiny cases by brute-force enumeration.
  - Write reasoning in words, not just equations. **Clarity and honesty**: saying "I don't know how to build this bridge" beats presenting a confident design that collapses. Don't reverse-engineer a sloppy argument to reach the answer you expected.
- **Failure modes**
  - Treating "two options" as 50/50.
  - Forgetting whether order matters, or whether sampling is with or without replacement. Each mistake changes the count by factors like k!.
  - Assuming independence between trials without justification (a "sticky" coin).
- **Relevance for Sam**
  - Combinatorial explosion is the reason entity matching needs blocking. n locations compared pairwise gives C(n,2) ≈ n²/2 comparisons, about 1.8×10^13 for 6M GeoNames rows, so restrict candidates by country, geohash or name prefix.
  - The sampling table matters for audits. Use draws *without replacement* for a manual match-quality check (no row reviewed twice). With replacement is a bootstrap.
  - Honest reporting: when a result is uncertain, say so plainly, and show the reasoning behind a metric as well as the number.

## Ronny Kohavi (Microsoft/Bing, ex-Amazon) — Trustworthy Online Controlled Experiments at Large Scale (Split DECISIONS keynote) (https://www.youtube.com/watch?v=kTAFOCynWIg)
**Most ideas fail and people are bad at predicting which will win, so ship through controlled experiments. Getting *trustworthy* numbers (the right OEC, A/A tests, sample-ratio checks, Twyman's law) is the hard part. Computing p-values is easy.**

- **Mental models / first principles**
  - Opening story: a "meh" idea (move an ad's first line into its title) sat in the backlog for months. An engineer built it in days, and the revenue alert fired because revenue was *too high*. It turned out to be +12% revenue (over $100M/yr) with no guardrail damage. Nobody had rated it highly. People rank ideas badly.
  - **HiPPO** (highest-paid person's opinion): give the HiPPO data, and controlled experiments are the best data.
  - **Trustworthy** is the key word. "It's very easy to show p-values with five digits after the decimal point and the first one is wrong."
  - Power vs size: a large product cares about tiny deltas (0.1% of Bing revenue ≈ $20–30M), so it needs about 1M+ users per treatment. A startup looking for *big* effects can get by with thousands, because required sample size scales with 1/effect² (his "quadratic" point).
- **Mechanisms (experiment lifecycle at Bing, ~2,000 experiments/month, ~300 treatments/week)**
  - Design templates: one recommended design for most experiments.
  - **Pre-launch gates**: automated QA, e.g. performance checks, and the top 1,000 queries must still return the right top result.
  - **Seed finder**: random splits are sometimes unlucky (a few heavy users skew one arm). Try hundreds of seeds against *historical* data and pick one with no pre-period differences across metrics.
  - **Staged ramp**: 1% traffic in a single data centre (this once caught a memory leak that would have taken down every data centre), then near-real-time "cheap" metrics (~10) at 15 minutes to catch egregious harm, then a few hours with more metrics, then full allocation (10–20% on Bing, 50/50 elsewhere for maximum power).
  - **Scorecard** of thousands of metrics (from ~10,000 available, 2–3K per experiment) so that unexpected side effects show up. Electronic data collection is cheap.
- **Five lessons**
  1. **Agree on the OEC (overall evaluation criterion).** It must be measurable short-term (1–2 weeks) and *predictive of long-term value*.
     - Amazon email: revenue-per-email was a bad OEC because it rises with more spam. After subtracting the lifetime value lost to unsubscribes, **half the emails were net negative**.
     - Microsoft support: half the room thought "time on site" should go up, the other half thought down. Agreeing on a metric is useless without agreeing on its *direction*.
     - Bing optimises long-term query share, but worse results *raise* short-term queries (users reformulate), and more ads raise short-term revenue while sessions/user fall within a week. So ads became a **constrained optimisation** with an average vertical-pixel budget per query.
     - Local optimisation trap: moving a widget left gave +109% clicks to that widget by cannibalising the others. Measure the page as a whole (clicks × value per user).
     - "Reduce zero-result queries to 0" produces junk results. Sometimes no result is the right answer.
     - Office homepage: OEC was clicks on revenue links and showed a 64% change, but only one variant showed price, so click→purchase conversion differed between arms. Measure the real outcome (purchases), not a proxy whose conversion the treatment can change.
     - Move towards a *single* ship/no-ship metric, which makes automated experimentation and parameter sweeps possible. Use surveys only to bootstrap metric discovery (they are noisy and annoying). Read *How to Measure Anything*.
  2. **Most ideas fail.** At Microsoft roughly a third are positive, a third flat and a third negative. In mature, well-optimised areas 80–90% fail. Don't ship flat features, because they add code bloat. Only about **1 in 5,000** experiments moves Bing's north-star metric (sessions/user) with replication. So with p < 0.05 the false-positive rate (5%) dwarfs the base rate (0.02%), and you must replicate. A 100-engineer, year-plus social-pane project produced zero value. Kiss more frogs, faster: cheap experimentation removes the "no time to test" excuse.
  3. **Small changes can have big impact**: search-box text +$5M/yr, ad site links +$50M, colour shade tweaks +$10M, a 100 ms speed-up worth a lot (each 4 ms pays a server engineer's yearly cost), a credit-card offer at Amazon worth tens of millions.
  4. **But most progress is inch by inch**: the relevance team's yearly goal is +2% on one metric. Ads gains of 15–25%/yr come from hundreds of small, tested ships combined. Controlled experiments detect improvements that seasonality would otherwise hide.
  5. **Validate the experimentation system.**
     - Remove **bots**: about 50% of Bing's US traffic and about 90% in China/Russia.
     - **A/A tests** (both arms identical): p-values should be uniform, with about 5% "significant" at 0.05. Simulated A/A runs exposed a p-value distribution piled up near 0, caused by a wrong variance formula for **ratio metrics** (randomised by user, analysed per page or click, so independence fails). Fix with the **delta method**.
     - **Sample ratio mismatch (SRM)**: a designed 50/50 split that comes out 50.2% at very large N had p ≈ 1e-6, about 1 in 500,000. Treat that as a bug, not luck. (note: the user counts in the transcript are garbled.) Always chi-square test the observed split before reading results.
     - **Twyman's law**: "any figure that looks interesting or different is usually wrong." Example: a 64% lift. Examples of artefacts: default dropdown values (birthdays 11/11/11, "astronaut" as the first alphabetical job), traffic "doubling" on daylight-saving day.
- **Failure modes**
  - Off-the-shelf tools that peek or use invalid stats (early Optimizely, according to him). Popular A/B books that get the statistics wrong.
  - Vanity metrics, and proxy metrics the treatment can change the meaning of.
  - Celebrating a surprising win without debugging it.
- **Relevance for Sam (travel product)**
  - Define an OEC for Trameter features before testing, e.g. bookings or booking value per user over a fixed window, with guardrails (cancellations, page latency, support contacts). "Clicks on a Drive Button" is a proxy. Check it isn't cannibalising other paths, and that the treatment doesn't change click→booking conversion.
  - Low traffic means power only for big effects. Accept that you can detect only large changes, or use longer runs and variance reduction.
  - Run an A/A test or SRM check on any new split mechanism. Filter bots from travel search logs, where scraper traffic is heavy.
  - Analysing per-search or per-click metrics while randomising per user requires delta-method or cluster-robust standard errors.
  - Apply Twyman's law to matching and pipeline stats: a sudden jump in match rate is a bug until proven otherwise.

## Grant Sanderson (3Blue1Brown) — Bayes theorem, the geometry of changing beliefs (https://www.youtube.com/watch?v=HZGCoVF3YvM)
**Evidence doesn't set your beliefs from scratch; it updates them. The posterior is the share of the evidence-consistent cases where the hypothesis holds, which you can picture as an area in a unit square.**

- **Mental models / first principles**
  - Three levels of understanding: plug in numbers; know why it's true; **recognise when you need it**. The third matters most.
  - "Rationality is not about knowing facts, it's about recognizing which facts are relevant." The usual failure is never thinking about the base rate at all, not getting it slightly wrong.
  - Underneath, probability maths is the maths of *proportions*. Bayes says: restrict to the cases where the evidence holds, then ask what fraction of those also have the hypothesis.
- **Mechanisms**
  - Steve (Kahneman & Tversky): "shy, tidy, detail-loving." Librarian or farmer? People say librarian and ignore the ~20:1 farmer-to-librarian ratio.
  - Representative sample: 210 people = 10 librarians + 200 farmers. If 40% of librarians and 10% of farmers fit the description, that's 4 librarians and 20 farmers, so P(librarian | description) = 4/24 ≈ **16.7%**. A 4× likelihood ratio can't overcome a 20× prior ratio.
  - Formula: P(H|E) = P(H)·P(E|H) / [P(H)·P(E|H) + P(¬H)·P(E|¬H)]. P(H) is the **prior** (1/21 here), P(E|H) the **likelihood**, P(E|¬H) the false-match rate in the other group, and P(H|E) the **posterior**. The population size (210) cancels.
  - Geometric version: a 1×1 square with the hypothesis taking width P(H) on the left. Evidence keeps a height of P(E|H) on the left and P(E|¬H) on the right. The posterior is left area / total kept area.
  - If P(E|H) = P(E|¬H), the evidence is irrelevant and the belief doesn't move. Beliefs move a lot only when the two likelihoods differ a lot.
  - In practice P(E) almost always has to be computed by splitting into the H and ¬H cases (law of total probability).
- **Rules of thumb**
  - Think in **natural frequencies**. The Linda problem: 85% of people rated "bank teller AND feminist" as more likely than "bank teller" (conjunction fallacy). Asked how many of 100 such people fall into each group, almost nobody makes the error. "40 out of 100" works better on intuition than "40%" or "0.4".
  - Don't memorise the formula. Redraw the square.
  - Disputes about such problems usually split into two parts: context or reference class shifts the *prior* (is Steve a random American, or someone you're likely to meet?), and stereotype accuracy shifts the *likelihoods*. Argue about each separately and explicitly.
- **Failure modes**
  - Base-rate neglect. The conjunction fallacy (a subset rated more likely than its superset). Choosing the wrong reference class for the prior.
- **Relevance for Sam**
  - **Matching QA:** a "match" flag from a fuzzy matcher is evidence, not truth. If true duplicates are rare (say 1% of candidate pairs) and the matcher fires on 95% of true pairs but also 2% of non-pairs, then precision = 0.0095 / (0.0095 + 0.0198) ≈ 32%. Always fold in the base rate of true matches among the candidates you're scoring.
  - **A/B testing:** with Kohavi's base rate (1 in 5,000 experiments truly moves the north star), a p < 0.05 result is still most likely a false positive. Bayes explains why replication is mandatory.
  - When explaining findings to non-statisticians, use counts ("of 1,000 flagged hotel pairs, about 320 are real duplicates"), not conditional percentages.

## Josh Starmer (StatQuest) — Power Analysis, Clearly Explained (https://www.youtube.com/watch?v=VX_M3tIyiYk)
**Decide your sample size *before* the experiment from three inputs: desired power, alpha, and effect size (how much the two distributions overlap). Adding a few more subjects after a near-miss p-value is p-hacking.**

- **Mental models / first principles**
  - Scenario: 3 patients on drug A vs 3 on drug B gives p = 0.06. The tempting move, adding one more person per arm and re-testing, is **p-hacking**. The right move is a power analysis that sets n for the *next* experiment.
  - **Power** = P(correctly reject H0 | a real difference exists). The usual target is 0.8.
  - Power is driven mainly by (1) **overlap** of the two populations (distance between means relative to spread) and (2) **sample size** per group. More overlap needs more n.
  - Why n helps: tests compare *summaries* (means), not raw points. With n = 1, the estimated mean is just one draw, so about 25% of estimates land far left and about 25% far right. With n = 2 a wild point is usually offset by the other. With n = 10 the distributions of sample means are narrow enough that two overlapping populations give *non-overlapping* sampling distributions of the mean. That is the standard error shrinking roughly as 1/√n.
  - The **central limit theorem** means this reasoning applies whatever shape the underlying distribution has, because sample means become roughly normal.
- **Mechanism: doing a power analysis**
  1. Choose power (0.8).
  2. Choose alpha (0.05).
  3. Estimate **effect size d** = (difference in means) / (pooled SD). A simple pooling is s_pooled = √((s1² + s2²)/2). Other effect-size definitions exist, so check which one your calculator expects.
  4. Get the means and SDs from prior data, published studies, or, worst case, an educated guess.
  5. Plug into a power calculator.
  - Worked example: difference = 10, SDs 7 and 6, so pooled ≈ 6.5 and d ≈ 1.5, giving **n = 9 per group** for 80% power at alpha = 0.05.
- **Rules of thumb**
  - Run the power analysis *before* collecting data. A fixed n means "regardless of the p-value, we used enough data to make a good decision."
  - A p just above 0.05 in a small pilot is a reason to plan a properly powered study, not to top up the sample.
  - Large overlap (small d) needs much more data, because n scales with 1/d². Halving the effect size roughly quadruples the required n (Kohavi's "quadratic" point).
- **Failure modes**
  - Optional stopping / topping up after peeking, which inflates false positives.
  - An underpowered study that reports "no significant difference" as if it were evidence of no effect.
  - Plugging in an optimistic effect size taken from a noisy pilot, which leaves the study underpowered.
- **Relevance for Sam**
  - Before an A/B test on Trameter traffic, compute the needed users per arm from baseline conversion and the minimum effect worth detecting. For proportions, n per arm ≈ 16·p(1−p)/δ² at 80% power and alpha = 0.05. Example: baseline 3% booking rate, detecting an absolute +0.3 pp needs about 16·0.0291/0.000009 ≈ 52k per arm. If traffic can't reach that, test bolder changes or a higher-frequency metric.
  - Fix the stopping rule in advance. If you must peek, use sequential methods, not repeated fixed-n tests.
  - For a manual audit of matching quality, the same logic tells you how many pairs to label to estimate precision within ±x%.

## Josh Starmer (StatQuest) — Bayes' Theorem, Clearly Explained (https://www.youtube.com/watch?v=9wCnvr7Xw4E)
**Bayes' theorem is just two ways of writing the same joint probability, P(A and B) = P(A|B)P(B) = P(B|A)P(A). It matters because you rarely have the full table, so you rebuild one conditional from the other plus marginal estimates or guesses (priors).**

- **Mental models / first principles**
  - Conditional probability = the probability of an event *scaled by what you already know*: P(A and B | B) = P(A and B)/P(B). The "redundant" notation P(not candy AND soda | soda) makes clear that both conditionals concern the *same* event (the same cell of the contingency table). Only the knowledge you divide by differs.
  - Example (StatLand, 14 people): 2 like both, 4 candy only, 5 soda only, 3 neither. P(no candy | soda) = 5/7 = 0.71. P(soda | no candy) = 5/8 = 0.63. Same numerator, different denominators.
- **Mechanism (derivation)**
  - P(A∩B) = P(A|B)·P(B) and P(A∩B) = P(B|A)·P(A). Set them equal and divide: **P(B|A) = P(A|B)·P(B)/P(A)**. You no longer need the joint probability.
- **Why it matters**
  - With the full contingency table, Bayes adds nothing: just count. In practice you usually have only some pieces, e.g. P(no candy | soda) = 0.71, a *guess* P(soda) ≈ 0.6, and P(no candy) = 0.57. Bayes gives P(soda | no candy) ≈ 0.75, not the true 0.63, because the guessed marginal was off.
  - That gap is the point of Bayesian statistics: you must guess (estimate) some quantities, such as priors, when you can't survey a whole population (e.g. asking everyone in India). Bayesian statistics studies what such guesses imply and how to make them well.
- **Rules of thumb / failure modes**
  - Keep P(A|B) and P(B|A) distinct. They share a numerator, not a denominator. Confusing them is the "prosecutor's fallacy" (my label, not his).
  - A wrong marginal or prior carries straight through to the posterior. Be explicit about which inputs are guesses.
- **Relevance for Sam**
  - Inverting conditionals is routine in data work. You know P(name similarity > 0.9 | true match) from a labelled sample but need P(true match | similarity > 0.9). That requires the base rate of true matches among candidates, so state where that number came from.
  - When a full cross-tab is available (e.g. bookings × device × market in DuckDB), compute conditionals directly by counting rather than via Bayes on estimated marginals.

## Josh Starmer (StatQuest) — Random Forests Part 1: Building, Using and Evaluating (https://www.youtube.com/watch?v=J4Wdy0Wc_xQ)
**A single decision tree fits its training data well but generalises badly. A random forest averages many deliberately varied trees (bootstrap rows plus random feature subsets at each split) and gets a free validation estimate from the rows each tree never saw (out-of-bag error).**

- **Mental models / first principles**
  - Trees are easy to build, use and interpret, but *inaccurate* on new data (Elements of Statistical Learning). They have high variance: small changes in the data give very different trees.
  - The fix is **variety plus voting**. Many decorrelated trees, aggregated, cut variance while each tree stays simple. "The variety is what makes random forests more effective than individual decision trees."
- **Mechanisms**
  1. **Bootstrap**: draw n rows *with replacement* from the n-row training set. Duplicates are allowed.
  2. **Grow a tree on the bootstrap sample, but at each split consider only a random subset of m features** (in the example, 2 of 4). Pick the best split among those, then repeat at each node.
  3. Repeat steps 1–2 hundreds of times.
  4. **Predict** by running a new case through every tree and taking the majority vote (for regression, the average). Bootstrap plus aggregate = **bagging**.
  - **Out-of-bag (OOB) data**: about one third of the original rows (≈ 1 − 1/e ≈ 36.8%) are left out of each bootstrap sample. Run each row through only the trees that didn't see it and take the vote. **OOB error** = share of rows misclassified this way. It acts as built-in cross-validation, with no separate holdout needed.
  - **Tuning m** (features per split): build forests with different m, compare OOB error, keep the best. Start near **√(number of features)** for classification and try a few values above and below. (note: the transcript says "square of the number of variables"; the standard default is the square root.)
- **Decisions & trade-offs**
  - Forest vs single tree: much better accuracy, less interpretability. Use variable importance or partial dependence (not covered here) to recover some explanation.
  - Small m gives more decorrelated trees and lower variance, but each tree is weaker. Large m gives stronger but more correlated trees. Choose m by OOB error.
- **Rules of thumb**
  - Use OOB error as a cheap first estimate of generalisation error. Still keep a true holdout for final reporting when the data has time or group structure.
  - More trees don't overfit. They just cost compute.
- **Failure modes / gotchas** (practitioner additions, beyond the video)
  - OOB and random CV assume rows are exchangeable. With duplicates or near-duplicates (the same hotel under several suppliers) or temporal drift, OOB is optimistic. Use group-aware or time-based splits.
  - Majority vote at 0.5 hides class imbalance. Look at vote proportions as scores and choose thresholds deliberately.
- **Relevance for Sam**
  - A random forest is a strong baseline for a **match/no-match classifier** on engineered pair features (name similarity, distance, category agreement, phone match). OOB error gives a quick read without a separate split. But group by entity when validating so the same place doesn't appear on both sides.
  - On a 16GB laptop: tree building parallelises well across cores (n_jobs=-1). Bootstrap with max_samples < 1.0 to save memory on millions of rows.

## Josh Starmer (StatQuest) — Logistic Regression (https://www.youtube.com/watch?v=yIYKR4sgzI8)
**Logistic regression fits an S-shaped curve giving P(true | inputs) for a yes/no outcome. It handles continuous and categorical predictors, is fitted by maximum likelihood (not least squares), and becomes a classifier once you pick a threshold.**

- **Mental models / first principles**
  - Recap of linear regression: fit a line, R² measures effect size, a p-value tests R², and the line can predict. Prediction from data *is* machine learning, so plain regression is ML.
  - Multiple regression adds predictors. Comparing a simple model to a complex one tells you whether an extra measurement earns its place.
  - Logistic regression does the same for a **binary outcome** (obese / not obese). The fitted curve runs from 0 to 1 and is read as a probability: heavy mouse means high P(obese), mid-weight about 50%, light mouse low.
- **Mechanisms**
  - **Classification**: threshold the probability, e.g. > 0.5 means "obese". The threshold is a decision, separate from the model.
  - Predictors can be continuous (weight, age) or discrete (genotype, star sign). Models can grow: weight -> + genotype -> + age -> + astrological sign.
  - **Variable usefulness**: test whether each coefficient differs from zero with the **Wald test**. Star sign isn't significant ("totes useless"), so drop it and save collection cost.
  - **Fitting by maximum likelihood**: there are no residuals in the least-squares sense. For a candidate curve, compute each observation's likelihood (P(obese) for obese mice, 1 − P for the rest), multiply them all together, shift the curve, repeat, and keep the curve with the highest likelihood. In practice you maximise the log-likelihood numerically.
- **Decisions & trade-offs**
  - Linear vs logistic: use logistic when the outcome is binary. A straight line can predict below 0 or above 1.
  - (note: the video says you "can't" compare simple vs complex logistic models or compute R². In fact you can, with **likelihood-ratio tests** and **pseudo-R²** such as McFadden's, which StatQuest covers in later videos. It's only least-squares R² and residuals that don't carry over directly.)
- **Rules of thumb**
  - Report probabilities, not just labels. Choose the threshold from the costs of false positives vs false negatives, not 0.5 by default.
  - A non-significant Wald test means the variable didn't help *in this model and sample*. It isn't proof of no effect (see McElreath: zero isn't special).
- **Failure modes**
  - Treating the fitted probability as calibrated without checking it (use calibration plots), especially after resampling for class imbalance.
  - Reading logistic coefficients as causal effects. Without a causal design, coefficients are associations, and covariate choice should follow the DAG (forks vs pipes vs colliders).
- **Relevance for Sam**
  - Logistic regression is the natural interpretable model for **booking conversion** (did a session book?) and for **pair-match scoring**. The coefficients show which signals matter, and the probabilities let you set a match threshold by precision/recall targets checked on a labelled sample.
  - For "did the change work", a logistic model of booking on treatment plus *pre-treatment* covariates (market, device) can improve precision. Never add post-treatment covariates.

