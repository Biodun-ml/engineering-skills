---
name: data-science-statistics-first-principles
description: First-principles statistics and data science playbook from 32 lectures (McElreath, Brady Neal, Blitzstein, 3Blue1Brown, Kohavi, StatQuest). Use whenever analysing data, judging whether a change worked, running A/B tests, choosing controls, reading p-values, or building and evaluating a model.
---

# Data science and statistics from first principles

This skill turns 32 lectures and talks into working judgment for reasoning honestly with data.
This file holds the analysis procedure, the principles the teachers keep coming back to, and
checklists. `references/` (when present) holds detailed notes for every video, each credited
to its teacher.

Use it whenever numbers are about to drive a decision, even if the user doesn't say
"statistics": "did the release help bookings?", "is this match rate good?", "which supplier
converts better?", an A/B test, a dashboard that jumped, a regression, a classifier
threshold, or "how many rows do I need to label?".

**How to use it**
1. Follow the analysis workflow (section 1) for any question that ends in a claim.
2. Run the relevant checklists (section 3) against the design, query, notebook or write-up.
3. For depth, open `references/`. `references/INDEX.md` lists every video with its line
   number, or run `grep -n -i "keyword" references/*.md`. Lines marked "(note: …)", the
   "Relevance to Sam" notes, and "(practitioner note, not from the lectures)" in this file are
   helper additions, not the speaker's words. If `references/` is missing, clone the private
   repo `Biodun-ml/engineering-skills` (folder `data-science-statistics-first-principles/references/`).
4. Explain the *why*, and credit the teacher when you use a specific pattern.
5. Distrust intuition, including your own: compute, count or simulate. Newton got a dice
   answer right with wrong reasoning. *(Blitzstein — L1, L4)*

---

## 1. Analysis workflow ("draw the whole owl")

**Triage first** (so a quick question doesn't get the full owl):
- *Prediction* (score, classify, forecast)? Go to §2 Regression and the Model evaluation checklist.
- *Descriptive* ("is this match rate good?")? Count it, give an interval, and model how the sample was selected.
- *Causal, randomised*? Use the A/B checklist. *Causal, dated launch with no randomisation*? Use the
  quasi-experimental bullet in §2 Causal. *Causal, cross-sectional observational*? Run the full workflow below.

1. **Name the estimand.** What exact quantity answers the question: a proportion, a total
   effect, a direct effect, a mean contrast or an individual contrast? Each causal question
   usually needs its own model. *(McElreath — L01, L04)*
2. **Draw the causal model (DAG) before touching data.** Arrows come from domain knowledge and
   the interventions you'd actually consider, never from the scatter plot: forks, pipes and
   colliders can all produce the same plot. Include how the sample was selected (who answered,
   got scraped, booked, survived dedup). "No causes in, no causes out." *(McElreath — L01, L04, L05)*
3. **Decide identification, then estimation.** Turn the causal estimand into a statistical one
   using stated assumptions (randomisation, or a backdoor adjustment set plus overlap, no
   interference, consistency). Only then pick a model. For a before/after launch with no
   randomisation, use a quasi-experimental design instead (§2 Causal).
   *(Brady Neal — Potential Outcomes; McElreath — L06)*
4. **Write a generative simulation with a known effect** (including an effect of zero) and
   check the estimator recovers it: extremes (p=1 gives only water), large n, a sweep of
   parameter values. Use the same simulator to size the study. "If you test nothing, you miss
   everything." *(McElreath — L01, L02, L06)*
5. **Validate the data before reading results.** Join and label bugs, bots, sample-ratio
   mismatch, default values. A surprising number is a bug until proven otherwise (Twyman's law).
   *(Kohavi — Split keynote; Starmer — Hypothesis Testing, "mislabelled drug")*
6. **Run the planned analysis once.** Pre-planned metric, sample size and test; pre-treatment
   covariates only. *(Starmer — p-hacking, Power; McElreath — L05)*
7. **Compute the contrast and summarise last.** Take differences sample by sample (posterior or
   bootstrap), then report the mean and interval of the *difference*. Average predictions over
   the target population's covariate distribution, don't read one coefficient.
   *(McElreath — L04, L06; Neal — adjustment formula)*
8. **Report the effect, its uncertainty, the assumptions and every test you ran** (section 3,
   "Reporting results honestly"). Treat analysis code as software: versioned, rerunnable, no
   point-and-click. *(McElreath — L01)*

---

## 2. Principles that recur across the teachers

### Probability foundations
- **Conditioning is "the soul of statistics": filter, then renormalise.** P(A|B) is computed on
  the rows where B happened. P(A|B) ≠ P(B|A); they share a numerator, not a denominator.
  *(Blitzstein — L4, L7; Starmer — Bayes)*
- **Independence is not disjointness,** and pairwise independence is not full independence.
  Before multiplying two error rates (name similarity, distance), check the errors really are
  independent; correlated errors make the combined error far worse. *(Blitzstein — L4)*
- **"At least one" → 1 − P(none).** 1−(5/6)^6 ≈ 0.665 beats the 12- and 18-dice cases that
  intuition prefers. *(Blitzstein — L4, Newton–Pepys)*
- **Two possibilities don't mean 50/50.** Equal-likelihood counting needs a stated symmetry
  (shuffling, random sampling). Check counts on tiny cases by brute force. *(Blitzstein — L1)*
- **Count before you compare.** Pairwise comparison of n rows is C(n,2) ≈ n²/2; for 6M rows
  that is ~1.8×10^13, which is why matching needs blocking. *(Blitzstein — L1)*
- **Linearity of expectation always holds, even with dependence.** Write a count as a sum of
  indicators: E[count] = sum of probabilities (aces in a hand: 5 × 1/13). Calibrated match
  probabilities summed give the expected number of true matches. Linearity gives the mean,
  not the variance. *(Blitzstein — L9)*
- **First-step analysis:** condition on the first step and use self-similarity (geometric mean
  failures q/p; gambler's ruin). Solve the general version to expose the recursion.
  *(Blitzstein — L7, L9)*
- **Small per-step edges compound.** With p = 0.49 and equal stakes, A's chance of winning falls
  from ~40% (N=20) to ~2% (N=200). A 1% loss per funnel or pipeline step is not small.
  *(Blitzstein — L7)*
- **Variances add (for independent or uncorrelated terms), SDs don't.** SD of a sum is σ√n, SD
  of a mean is σ/√n: halving the error needs 4× the data. Correlated sessions of one user make
  the real SE bigger than σ/√n, so aggregate to the randomisation unit first. *(3Blue1Brown — CLT)*
- **The CLT licenses normal intervals on means, under conditions.** It needs independence,
  identical distribution and finite variance. "n ≥ 30" is a rule of thumb; heavy tails
  (whale bookings, latency spikes) converge slowly or not at all. Bootstrap the mean at your
  real n and look. *(3Blue1Brown — CLT; Starmer — CLT, with note)*
- **Bootstrapping done right** *(practitioner note, not from the lectures)*: resample the unit of
  randomisation (users or articles, not sessions or clicks); use ≥2,000 resamples with percentile
  or BCa intervals. It does not rescue tiny n with extreme whales, so check the bootstrap
  distribution's shape.

### Bayes and base rates
- **Evidence updates beliefs; it doesn't replace them.** Posterior ∝ prior × likelihood. Steve
  fits librarians 4× better but farmers outnumber them 20:1, so P(librarian) ≈ 4/24 ≈ 16.7%.
  Evidence moves you only when P(E|H) and P(E|¬H) differ. *(3Blue1Brown — Bayes)*
- **Recognising when you need Bayes is the hard part.** The usual failure is forgetting the
  base rate entirely. *(3Blue1Brown — Bayes)*
- **Speak in natural frequencies.** "40 out of 100" removes the conjunction fallacy (85% error
  in the Linda task, near zero with counts). *(3Blue1Brown — Bayes)*
- **Bayesian updating is counting.** Ways each hypothesis could produce the data, normalised.
  For a proportion with a flat prior: Beta(W+1, L+1). No minimum sample size; the estimate is
  the whole distribution; push ~1,000 posterior samples through every downstream calculation.
  *(McElreath — L02)*
- **Wrong priors carry through.** Say which inputs are guesses. With the full table, just count.
  *(Starmer — Bayes)*
- **Model measurement error and selection.** Symmetric label noise pushes proportions toward 0.5 *and*
  makes the naive estimate overconfident. A biased sample can still estimate a population if
  you model how it was selected and reweight (post-stratification). *(McElreath — L02)*

### Causal inference first
- **Association = causal association + confounding association.** E[Y|T=1] − E[Y|T=0] equals the
  ATE only under exchangeability. Randomisation makes it true by deleting every arrow into the
  treatment, including from variables you never thought of. *(Neal — Intro, Potential Outcomes)*
- **The four elemental shapes decide every control.** Fork X←Z→Y: adjust. Pipe X→Z→Y: don't
  adjust for the total effect; do for the direct effect *only if no unobserved variable
  confounds mediator and outcome* (otherwise the mediator becomes a collider and the direct
  effect may be unidentifiable: L06 grandparent example). Collider X→Z←Y: never adjust; doing so
  *creates* association. Descendant: conditioning on it partly conditions on its parent.
  *(McElreath — L05, L06)*
- **Use the backdoor criterion.** List every X–Y path ignoring direction; close those entering
  X; leave causal paths open; check with dagitty. Minimal adjustment set ≠ best set (pick for
  precision too). *(McElreath — L06)*
- **Bad controls are common** *(McElreath — L06, from Cinelli, Forney & Pearl)*: M-bias
  (a pre-treatment collider: "pre-treatment is safe" is false); confounded mediators; descendants
  of the outcome ("among users who reached checkout"); precision parasites (pure causes of X);
  bias amplifiers (adjusting for a pure cause of X when an unobserved U confounds X and Y; in
  his simulation this roughly doubled the existing bias).
- **Post-treatment bias undoes randomisation in the analysis.** About half of published
  political-science experiments conditioned on post-treatment variables. *(McElreath — L05)*
- **Selection is conditioning on a collider.** Funded grants show newsworthiness vs rigour
  trade-offs absent in the applicant pool; same for booked trips and surviving listings.
  *(McElreath — L05)*
- **Simpson's paradox is settled by the graph, not the numbers.** COVID-27: B has lower
  *mortality* in both strata (mild 10% vs A 15%, severe 20% vs 30%) yet higher overall (19% vs
  16%) because B got mostly severe cases. If condition → treatment, adjust: B 12.9% vs A 19.4%,
  choose B. If treatment → condition (waiting for scarce B worsens patients), use totals and
  choose A. *(Neal — Intro)*
- **Adjustment formula: predict each row under T=1 and T=0, then average** over the same
  population weights for every arm. In Neal's simulation (true ATE 1.05) the unadjusted
  difference was 5.33. Adjusting gave 0.85 either way: adjustment formula with a linear model,
  or the T coefficient in Y~T+X. The leftover error comes from the linear form. A coefficient
  equals the ATE only if the effect is constant and the model linear. With interactions or
  non-linearity, use a flexible model, then predict and average.
  *(Neal — Potential Outcomes; McElreath — L06)*
- **Unconfoundedness is untestable, and more controls erode overlap** (50% overlap in 1-D is 25%
  in 2-D). Check positivity: treated and untreated units in every segment. Also check no
  interference and a single version of treatment. *(Neal — Potential Outcomes)*
- **Table 2 fallacy:** control-variable coefficients are not causal effects. Report only the
  estimand the model was built for. *(McElreath — L06)*
- **No randomisation and a dated launch ("did the release help bookings?")**
  *(practitioner note, not from the lectures)*: before/after comparisons are confounded by
  seasonality, marketing and supplier changes. Use difference-in-differences (check parallel
  pre-trends), interrupted time series or Bayesian structural time series (CausalImpact-style),
  synthetic control, or regression discontinuity on a threshold. State the assumptions and get
  them reviewed.
- **Not covered by these lectures** (flag them, don't improvise): IPW / AIPW / double ML,
  instrumental variables, survival / time-to-event, forecasting, missing-data imputation.

### Hypothesis tests, p-values, power, p-hacking
- **Null testing is cleanest in a two-arm randomised test.** You reject or fail to reject;
  never "accept". In observational systems nulls aren't unique and rejecting one doesn't pick
  your mechanism. *(Starmer — Hypothesis Testing; McElreath — L01)*
- **p is P(data this extreme | null), not P(it works).** With α=0.05 and no real difference,
  ~5% of runs are false positives. Choose α from the cost of being wrong (0.2 for trivial
  calls, 0.00001 for high stakes). *(Starmer — p-values; Kohavi — Lenny's)*
- **p is not effect size.** A 6-point difference gave p=0.24, a 1-point difference with more
  data p=0.04. Report effect and interval. Same for R²: report R² *and* its p, and use
  adjusted R² or held-out error since added predictors always raise in-sample R².
  *(Starmer — p-values, Linear Regression)*
- **Size the study first:** power (0.8), α (0.05), effect size d = Δ/pooled SD → n. n scales
  with 1/effect²: halve the effect, quadruple n. d≈1.5 needed n=9 per group.
  *(Starmer — Power; Kohavi)*
- **p-hacking has two forms.** Testing many things and reporting the winner (fix: adjust *all*
  p-values together); topping up after p=0.06 (fix: power analysis, then a new experiment).
  100 null tests → ~5 "hits". *(Starmer — p-hacking)* Which correction *(practitioner note, not
  from the lectures)*: Holm (family-wise) for confirmatory ship decisions across variants;
  Benjamini–Hochberg FDR for exploratory segment/metric screens, then retest discoveries
  (`statsmodels.stats.multitest.multipletests(method='holm' | 'fdr_bh')`).
- **Zero is not special.** An interval containing 0 isn't "no effect"; overlapping intervals
  aren't "no difference"; 95% is a convention ("a superstition"). *(McElreath — L02, L04, L05)*

### A/B testing and online experiments
- **Most ideas fail and experts can't tell which.** ~⅓ positive/⅓ flat/⅓ negative at Microsoft;
  ~85% fail at Bing, 92% at Airbnb search; only ~1 in 5,000 Bing experiments moved the north
  star. Flat = no-ship. Decompose big redesigns (one factor at a time). *(Kohavi — both talks)*
- **The OEC must be short-term measurable, predict lifetime value, and have an agreed
  direction.** Pair it with guardrails. Revenue-per-email ignoring unsubscribes made over half
  of Amazon campaigns net negative; a widget moved left got +109% clicks by cannibalising
  others; for travel, include predicted post-stay satisfaction, not conversion alone.
  *(Kohavi — both talks)*
- **Trust the system only after validating it.** SRM chi-square on every test (~8% of Microsoft
  experiments failed it: bots, pipeline filters, mid-funnel triggering); A/A tests should give
  uniform p-values; filter bots (~50% of Bing US traffic); ratio metrics analysed per page while
  randomised per user need the delta method. *(Kohavi — both talks)*
- **Twyman's law:** if typical effects are <1% and you see 10%, ~9 in 10 such cases had a flaw.
  *(Kohavi — Lenny's)*
- **Peeking at a fixed-horizon test pushes false positives from 5% to ~30%.** *(Kohavi —
  Lenny's)* Use a fixed horizon (whole weeks, from the power analysis), or a real sequential
  design: group-sequential with O'Brien–Fleming alpha spending, or always-valid / mSPRT
  inference as built into current platforms (Statsig, Eppo, GrowthBook, Optimizely). Never
  repeat a fixed-n t-test daily. *(practitioner note, not from the lectures)*
- **False positive risk:** at an ~8% historical success rate, a p<0.05 win is ~26% likely to be
  false. Rerun when 0.01<p<0.05 and combine (Fisher/Stouffer); want p<0.01 for broadcast
  learnings. Track your own success rate as the prior. *(Kohavi — Lenny's; 3Blue1Brown)*
  Formula *(practitioner note)*: FPR = (α/2)(1−π) / [(α/2)(1−π) + power·π], π = historical
  success rate (α/2 because only wins in the good direction count). π=0.08 and power 0.8 give
  ≈26%; π=0.02 gives ≈60%.
- **Know your traffic.** Below tens of thousands of users only big effects are detectable;
  ~200k users to detect ~5% changes on a retail-like conversion rate. Hunt 5–10% effects.
  *(Kohavi — Lenny's)*
- **Variance reduction:** cap skewed metrics (revenue, nights booked) and use CUPED with
  pre-period data of the same metric. *(Kohavi — Lenny's)* CUPED only helps users with
  pre-period data. Mostly-new anonymous blog readers give little gain, so rely on capping, bolder
  changes and higher-frequency metrics. Capping is winsorising: state the cap and that the
  estimand changed. *(practitioner note, not from the lectures)*
- **Ramp in stages** (small slice first, cheap near-real-time metrics, then full power) and keep
  a searchable experiment archive with quarterly reviews of the biggest surprises.
  *(Kohavi — both talks)*

### Regression, classification and model evaluation
- **Bias vs variance is judged only on held-out data.** A perfect training fit is a red flag.
  Regularisation, bagging and boosting find the sweet spot. *(Starmer — Bias/Variance)*
- **10-fold CV for comparing models and tuning hyperparameters**; every row tested once.
  (note: keep an untouched test set or nested CV after selection; split by group or time to
  stop leakage.) *(Starmer — Cross Validation)*
- **Ridge trades a little bias for a lot less variance:** SSR + λΣβ² (intercept not penalised),
  λ by CV, solves p > n. (note: standardise first; coefficients are biased.) *(Starmer — Ridge)*
- **Logistic regression gives P(y=1); the threshold is a decision** from error costs. Fit by
  maximum likelihood. (note: compare models with likelihood-ratio tests and pseudo-R².) Check
  calibration before trusting probabilities. *(Starmer — Logistic; McElreath — "points are decisions")*
- **Trees split greedily on lowest weighted Gini (1 − Σp²);** a single-example leaf is
  overfitting, so set min leaf size or prune via CV. Single trees are high-variance.
  *(Starmer — Trees)*
- **Random forests = bootstrap rows + random feature subsets per split + vote.** OOB error
  (~⅓ of rows unseen per tree) is free validation but optimistic with near-duplicates or time
  drift. Tune m from ~√features by OOB error; more trees don't overfit. *(Starmer — RF)*
- **Read the full confusion matrix on test data,** checking its orientation (Starmer: rows =
  predicted; scikit-learn: rows = actual). *(Starmer — Confusion Matrix)*
- **ROC/AUC compares models; pick the operating point from costs.** With rare positives (entity
  matching), use precision–recall: FPR can look tiny while most flags are wrong.
  *(Starmer — ROC/AUC)* (note: AUC = P(a random positive outscores a random negative). A
  higher-AUC model can still be worse at your operating point, so compare in the region you'll
  use. Check calibration with a reliability plot, especially after resampling for imbalance.)
- **Many groups with uneven n (destinations, suppliers, articles): partially pool.**
  Multilevel/hierarchical models, or ridge/empirical-Bayes shrinkage, beat both per-group raw
  rates and ignoring groups. Don't rank destinations by raw conversion at n=12. *(McElreath —
  L04: index coding carries straight to multilevel models; rest is a practitioner note, not from
  the lectures)*
- **PCA:** centre (and standardise mixed units), find perpendicular max-variance directions,
  read loadings before naming an axis and the scree plot before trusting a 2-D view (PC1+PC2
  ≈ 90% is fine). High variance can be noise. *(Starmer — PCA)*
- **Neural nets are summed bent lines;** use them for genuinely non-linear shapes, at a cost in
  data and interpretability. *(Starmer — Neural Networks)*
- **Splines/GAMs over polynomials** for flexible trends: polynomials are global and explode at
  the edges; prefer a mechanistic model when you have one. *(McElreath — L04)*

---

## 3. Checklists

**Analysis workflow (quick form)**
- [ ] Estimand written in one sentence; total vs direct effect stated.
- [ ] DAG drawn, including the selection/sampling process and plausible unobserved causes.
- [ ] Adjustment set from the backdoor criterion; overlap checked per segment.
- [ ] Estimator tested on simulated data with known effects, including zero.
- [ ] Data validated (joins, labels, bots, defaults, time zones) before results are read.
- [ ] Contrast computed sample by sample; uncertainty carried through.

**A/B test**
- [ ] One OEC with agreed direction, predictive of long-term value; guardrails named
      (cancellations, latency, support contacts, cannibalised paths).
- [ ] Randomisation unit fixed; analysis unit matches it or uses the delta method/clustered SEs.
- [ ] Power analysis done: baseline, minimum effect worth shipping, n per arm, fixed duration.
- [ ] Not enough traffic? Test a bolder change, a higher-frequency metric, capping or CUPED.
- [ ] Randomisation unit chosen (cookie/user vs article/blog); if clustered, effective n = number
      of clusters, with cluster-robust SEs or a cluster bootstrap.
- [ ] A/A test on any new split mechanism; bots filtered.
- [ ] Staged ramp with a kill switch for egregious harm.
- [ ] SRM chi-square passes *before* any metric is read; alarm at p < 0.001
      (`scipy.stats.chisquare`), not 0.05. *(threshold: practitioner note)*
- [ ] Whole weekly cycles (Kohavi: 1–2 weeks); novelty/primacy checked by comparing the first
      days with the last.
- [ ] Any metric cap stated (it changes the estimand).
- [ ] No peeking with a fixed-horizon test; or a proper sequential design.
- [ ] Covariates are pre-treatment only.
- [ ] Every variant, segment and metric counted; Holm for ship decisions, FDR for screens;
      segment wins retested.
- [ ] Surprising result debugged first (Twyman's law); false positive risk estimated from the
      historical success rate; borderline wins replicated.
- [ ] Flat = no-ship; result logged in the experiment archive.

**Before claiming X causes Y**
- [ ] Was X randomised? If yes, was the randomisation undone by post-treatment conditioning?
- [ ] DAG drawn; every backdoor path closed; no causal path blocked.
- [ ] No controls that are colliders (incl. pre-treatment M-bias), mediators (for total
      effects), descendants of Y, or pure causes of X.
- [ ] Sample not selected on the outcome or a collider (booked only, matched only, survived).
- [ ] Simpson check: would stratifying flip the sign, and which direction does the arrow run?
- [ ] Overlap/positivity, no interference (shared inventory, prices), one version of treatment.
- [ ] Effect averaged over the target population, not read off a coefficient.
- [ ] Untestable assumptions (no unobserved confounder) stated explicitly.
- [ ] Where on the evidence ladder: anecdote < observational < natural experiment < RCT <
      replicated RCTs. *(Kohavi — Lenny's)*

**Reporting results honestly**
- [ ] Estimand and causal assumptions stated so others can critique them.
- [ ] Effect size with interval, not p alone; "inconclusive at this n", never "no effect".
- [ ] Mean contrast vs individual contrast matched to the decision (men's mean weight is
      reliably higher, yet a random man is heavier only ~82% of the time). *(McElreath — L04)*
- [ ] Counts in natural frequencies ("of 1,000 flagged pairs, ~320 are real").
- [ ] Every test run disclosed; multiple-comparison correction stated.
- [ ] Precision computed with the base rate; false positive risk, not "95% sure".
- [ ] Only the designed estimand reported (no Table 2 coefficients).
- [ ] Code and data versioned and rerunnable *(McElreath — L01)*; uncertain parts said plainly
      *(Blitzstein — L1)*.

**Model evaluation**
- [ ] Baselines: regularised logistic regression (interpretable) and gradient-boosted trees
      (LightGBM / XGBoost / sklearn HistGradientBoosting, the usual tabular winner); random
      forest as a cheap OOB sanity check. *(practitioner note, not from the lectures; Starmer's
      NN notes also favour simple models or boosting on tabular data)*
- [ ] Group- or time-aware folds; untouched final test set or nested CV.
- [ ] Confusion matrix (orientation checked); PR curve when positives are rare.
- [ ] Threshold chosen from error costs; probabilities calibrated.
- [ ] Regularisation/leaf size/m tuned by CV, never training error.

---

## 4. Failure modes to hunt for in reviews

- "Control for everything" regressions (causal salad); controls kept because significant;
  collinear variables dropped for being correlated.
- Post-treatment covariates in an experiment analysis ("pages viewed", "reached checkout").
- Naive treated-vs-untreated comparisons from observational data presented as effects.
- Peeking and stopping at p<0.05; topping up after p=0.06; reporting the best of many segments.
- Reading 1−p as the probability the treatment works; "CI includes 0, so no effect";
  eyeballing overlapping intervals instead of computing the difference.
- No SRM check; bots in the denominator; per-click analysis of a per-user split.
- A proxy OEC the treatment can game (clicks on a button that now hides price).
- Celebrating a 10% lift without hunting the bug.
- Ignoring the base rate when reading a match flag or a "significant" result.
- Normal-theory intervals on heavy-tailed revenue at small n; treating sessions of one user as
  independent.
- Training accuracy reported as performance; random folds over near-duplicate or time data;
  OOB treated as final.
- Accuracy alone on imbalanced data; ROC/FPR for rare-match problems; default 0.5 threshold.
- PCA on unstandardised mixed units; polynomial trend terms extrapolated.

---

## 5. Where the teachers disagree, and how to resolve it

- **Null tests and 0.05.** Starmer teaches α=0.05 as a sensible convention (chosen by cost);
  McElreath calls 95% a superstition and null rejection a weak goal. Resolve: in a clean
  randomised two-arm test, a pre-planned test with α set by cost is fine (Kohavi's practice);
  for observational or mechanistic questions, estimate the effect with its full uncertainty
  and compare process models instead.
- **Bayesian vs frequentist.** McElreath uses Bayes for practical reasons (measurement error,
  missing data, small n) and calls the war over; Kohavi and Starmer work in p-values. Resolve:
  use whichever answers the question, but always bring the prior back in when interpreting a
  win (false positive risk) and when inverting conditionals.
- **How big is "large n" for the CLT.** Starmer: n ≥ 30 is a generally safe heuristic (his demos
  work at n=20); his only fine print is that a mean must exist, and he has never met Cauchy in
  20 years. 3Blue1Brown and the notes: finite variance and independence are
  required; skewed revenue can need far more. Resolve: bootstrap the mean at your real n.
- **"Test everything" vs "not enough units".** Kohavi says wrap every change in an experiment,
  but also that below tens of thousands of users the statistics don't work. Resolve: at low
  traffic, test only bold changes on high-frequency metrics, use capping/CUPED, and build the
  platform and habit so you're ready.
- **Dropping variables by significance.** Starmer drops a non-significant predictor (Wald test)
  to save collection cost; McElreath says significance is no guide to causal structure. Resolve:
  for prediction, drop by held-out performance; for causal estimates, choose by the DAG only.
- **Word "control".** McElreath avoids it (control is what experiments do); others use it
  freely. Prefer "stratify by" or "adjust for" so nobody mistakes adjustment for an experiment.
- **OOB vs held-out data.** Starmer presents OOB as free validation; the notes warn it is
  optimistic with duplicates or drift. Use OOB for tuning, a group/time holdout for reporting.

---

## 6. Applying this to Sam's work

Sam builds large location-data pipelines (GeoNames ~6M rows, matching/dedup at scale), supplier
integrations (Viator, RateHawk), and the Drive Button (an LLM pipeline turning travel blog
articles into bookable trips), all on a 16 GB laptop. Use these notes only in that context.

- **Drive Button and booking-flow changes:** randomise exposure; OEC = bookings or booking value
  per user over a fixed window, guardrails = cancellations, latency, cannibalised paths; button
  clicks are a proxy. Trameter traffic is likely below ~200k per test, so hunt large effects
  and say "false positive risk" honestly. Exposure on long-haul articles is a fork *only if trip
  type drives exposure*: stratify by trip type. If the button changes which trips readers pick,
  trip type is a mediator: use totals for the total effect (Neal scenario 2). *(Kohavi; Neal)*
- **Drive Button randomisation unit** *(practitioner note, not from the lectures)*: decide it
  first. Use per-reader (cookie) if the injector can split traffic. If only per-article or
  per-blog, it is a cluster experiment: effective n = number of articles, analyse with
  cluster-robust SEs or a cluster bootstrap, and expect much lower power. Shared supplier
  inventory or prices across arms is interference. *(Neal — no interference)*
- **Power for proportions:** the notes give n per arm ≈ 16·p(1−p)/δ² (80% power, α=0.05);
  3% baseline and +0.3 pp needs ~52k per arm. *(Starmer — Power, Sam note)*
- **Matching QA at 6M rows:** block before pairing (C(n,2) explodes).
  - *Precision:* label a random sample (without replacement) of *accepted* matches and report
    Beta(W+1, N−W+1).
  - *Recall:* sample from the *pre-threshold candidate pool*, stratified by score band with
    inverse-probability weights. True pairs are rare (~1%), so a uniform sample yields almost none.
  - *Selection collider:* never study feature relationships (e.g. distance vs name similarity)
    only inside the accepted set.
  - *Expected precision before labelling:* fold in the base rate. 1% true pairs, 95% recall and
    a 2% false-match rate give ~32% precision.
  - Use PR curves with separate auto-accept and review thresholds; group k-fold by source location.
  *(McElreath L02, L05; 3Blue1Brown; Starmer; stratified recall sampling is a practitioner note)*
- **Small labelled sets, many features:** L2-regularised logistic regression or a shallow tree
  with min leaf size; gradient boosting or a random forest as baselines (`n_jobs=-1`,
  `max_samples < 1.0` on the 16 GB laptop); incremental PCA in chunks. Partially pool sparse
  destinations. *(Starmer — Ridge, Trees, RF, PCA)*
- **Tools (Python, late 2026)** *(practitioner note, not from the lectures)*:
  - statsmodels: power (`NormalIndPower`, `proportion_effectsize`), proportions tests,
    cluster-robust SEs, `multipletests`
  - `scipy.stats.chisquare` for SRM checks
  - DoWhy / pgmpy or dagitty for DAGs and adjustment sets
  - PyMC / Bambi / NumPyro for Bayesian models
  - scikit-learn `GroupKFold`, `TimeSeriesSplit`, `CalibrationDisplay`
  - DuckDB for counting conditionals
- **Supplier comparisons (Viator vs RateHawk) are Simpson-prone:** destination and product mix
  differ per supplier, so standardise to a common mix before comparing. *(Neal)*
- **A jump in match rate or bookings is a join or logging bug until proven otherwise.**
  *(Kohavi — Twyman; Starmer — mislabelled drug)*

---

## 7. Where to look in `references/`

Index of all 32 videos with line numbers: `references/INDEX.md`.

| Need | File |
|---|---|
| Golem of Prague (DAG-first workflow), Garden of Forking Data (Bayes as counting, measurement error), Neal potential outcomes, Blitzstein L9 expectation/linearity, CLT (3Blue1Brown, StatQuest), PCA, neural nets, hypothesis testing, p-values, bias/variance (11 videos) | `references/ds-1-mcelreath-neal-blitzstein-3blue1brown-starmer.md` |
| Categories & curves (contrasts, splines), elemental confounds, gambler's ruin, probability and counting, Kohavi Split keynote (OEC, A/A, SRM, Twyman), 3Blue1Brown Bayes, power analysis, StatQuest Bayes, random forests, logistic regression (10 videos) | `references/ds-2-mcelreath-blitzstein-kohavi-sanderson-starmer.md` |
| Kohavi on Lenny's (failure rates, SRM, peeking, false positive risk, CUPED), good & bad controls (backdoor, catalogue, Table 2), conditional probability, Neal intro (Simpson's, adjustment formula), linear regression/R², ridge, trees, ROC/AUC, p-hacking, confusion matrix, cross-validation (11 videos) | `references/ds-3-kohavi-mcelreath-blitzstein-neal-starmer.md` |
