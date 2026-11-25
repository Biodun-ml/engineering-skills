# ds-3 — cross-cutting principles

1. **Most ideas fail, so default to measurement over opinion.** 2/3 of ideas failed at Microsoft, ~85% at Bing, 92% in Airbnb search; big bets fail ~80% of the time. Test every change, ship in small increments (one factor at a time), and treat "flat" as no-ship. *(Kohavi)*
2. **A p-value is P(data | null), not P(treatment works).** Convert to a false positive risk using your historical success rate: at an 8% success rate, a p<0.05 win is ~26% likely to be false. Replicate borderline wins (0.01<p<0.05) and combine with Fisher/Stouffer. *(Kohavi; Bayes' rule from Blitzstein)*
3. **Conditioning = filter and renormalise; Bayes' rule is just P(A|B)=P(B|A)P(A)/P(B).** Never confuse P(A|B) with P(B|A); independence (multiply) is not disjointness (mutually exclusive), and pairwise independence is not full independence. *(Blitzstein)*
4. **One planned test per question, sized in advance.** Peeking and stopping at p<0.05 pushes false positives from 5% to ~30%; topping up data after p=0.06 is p-hacking. Do a power analysis first; if you peek, use proper sequential methods. *(Kohavi; Starmer p-hacking)*
5. **Correct for multiple comparisons, and include every test you ran.** α=0.05 means ~5 false positives per 100 null tests; use FDR across all variants/segments/metrics, and treat segment "wins" as hypotheses to retest. *(Starmer p-hacking)*
6. **Validate the experiment before reading it.** Run a sample-ratio-mismatch check on every A/B test (~8% of Microsoft experiments failed it: bots, pipeline filters, mid-funnel triggering); run A/A tests on new platforms. *(Kohavi)*
7. **Twyman's law: an interesting or surprising number is usually wrong.** If typical effects are <1% and you see 10%, hunt for the bug first (9 of 10 times there is one). *(Kohavi)*
8. **Pick an OEC that predicts long-term value, with countervailing guardrails.** Revenue or clicks alone get gamed (more ads, email spam). Price the side effects (an unsubscribe's lost LTV showed over half of Amazon email campaigns were net negative). For travel, use predicted post-stay satisfaction, not only conversion. *(Kohavi)*
9. **Know when you have enough units.** Below tens of thousands of users, only large effects are detectable; ~200k users to detect ~5% changes on conversion. Shrink the sample you need with variance reduction: cap skewed metrics, use CUPED with pre-period data. *(Kohavi)*
10. **Association = causal + confounding association.** Only randomisation (which removes all arrows into the treatment, including from unobserved variables) or a correct adjustment set recovers causal effects. *(Neal; McElreath)*
11. **Simpson's paradox is resolved by the causal graph, not by the data.** If the stratifier causes the treatment, adjust (use subgroups). If the treatment causes the stratifier, don't (use totals). Adjusted estimate = per-stratum outcome averaged with the *same* population weights for every arm. *(Neal)*
12. **Choose controls with the backdoor criterion.** List all X–Y paths, close the ones entering X, and leave causal paths open. Use dagitty to check. Then marginalise over the controls for the target population; a regression coefficient alone is often not the causal effect. *(McElreath; Neal)*
13. **Bad controls are real and common.** Avoid conditioning on colliders (including pre-treatment ones, M-bias), on mediators when you want the total effect, on descendants of the outcome (selection on outcome, e.g. "among users who reached checkout"), and on pure causes of the treatment (they cost precision and can amplify confounding bias). Never pick controls by significance, by collinearity, or by "pre-treatment is safe". *(McElreath)*
14. **Table 2 fallacy: control-variable coefficients are not causal effects.** Report only the estimand the model was designed for. *(McElreath)*
15. **Report effect size and reliability together.** R² (share of variance explained) needs a p-value from the F-test, because two points always give R²=1. More parameters always raise in-sample R², so use adjusted R² or held-out error. *(Starmer, linear regression)*
16. **Trade a little bias for a lot less variance when data are scarce.** Ridge adds λΣβ² (intercept excluded), shrinks coefficients and still solves p>n problems; trees need minimum leaf size or pruning. Tune λ, leaf size and similar hyperparameters by cross-validation, never on training error. *(Starmer ridge, trees, CV)*
17. **Evaluate on unseen data with k-fold CV (10-fold default).** Every point is tested once; avoid arbitrary single splits. Add your own guardrails: group/time-aware folds to prevent leakage, and a final untouched test set or nested CV after model selection. *(Starmer CV)*
18. **A classifier is scores plus a threshold chosen from error costs.** Read the full confusion matrix (check its orientation). Use ROC/AUC to compare models and pick operating points. With rare positives (entity matching), prefer precision/recall, because FPR hides how many flagged items are wrong. *(Starmer confusion matrix, ROC/AUC)*
19. **Build institutional memory and an honest evidence hierarchy.** Anecdote < observational < natural experiment < RCT < replicated RCTs. Keep a searchable archive of experiments and review the biggest surprises (gap between expected and actual, in either direction) every quarter. *(Kohavi)*
20. **Distrust intuition; compute or simulate.** Newton got the dice answer right but his reasoning wrong. Test any estimator by simulating its assumed data-generating process, including a true effect of zero. *(Blitzstein; McElreath)*


## Ronny Kohavi (Lenny's Podcast) — The ultimate guide to A/B testing (https://www.youtube.com/watch?v=hEzpiDuYFoE)
**Core idea in one line: most ideas fail, humans are bad at predicting which ones, so a trustworthy controlled experiment (with a well-chosen OEC and automatic validity checks) is the only reliable arbiter.**

- **Mental models / first principles**
  - Humility is the base rate. Kohavi's published failure rates: about 2/3 of ideas fail at Microsoft overall, ~85% at Bing (a heavily optimised product is harder to improve), 92% for Airbnb search relevance (only 8% of ~250 experiments moved the target metric). Booking, Google Ads etc. report 80–90%. Every new team believes it will be the exception and is humbled.
  - "Test everything": any code change, even a small bug fix, can have surprising impact, so wrap it in an experiment. You cannot experiment too much; you can over-focus on incremental ideas.
  - Portfolio thinking: most gains are inch-by-inch (Bing relevance targets ~2%/year built from 0.1–0.2% wins; Airbnb search got ~6% revenue from many small wins). Reserve some effort for high-risk, high-reward bets but expect ~80% of them to fail (Bing's social-search integration ran ~1.5 years, hundreds of experiments, flat-to-negative, eventually killed).
  - Rare home runs exist: moving the second ad line into the title line at Bing raised revenue ~12% (~$100M/year then) without hurting guardrails; it sat low on the backlog for months. Opening search results in a new tab won repeatedly (Hotmail UK, MSN, later Airbnb) and was nearly forgotten — institutional memory matters.
  - The experiment platform plays two roles: a **safety net** (abort bad launches fast) and an **oracle** (trustworthy verdict on metrics at the end of, say, two weeks).

- **Mechanisms**
  - **OEC (overall evaluation criterion)**: the metric(s) you optimise must be *causally predictive of lifetime value*, not short-term revenue. Pair a goal metric with countervailing metrics. Bing: more ads = more short-term money but more churn and slower time-to-successful-click; fix via a constrained problem (fixed budget of vertical pixels for ads per query on average) or by pricing the user-experience cost.
  - Amazon email example: crediting the email team with any purchase after an email click led to spam. Modelling the lifetime-value loss of an unsubscribe (a few dollars) showed over half of campaigns were net negative. Follow-on feature: default the unsubscribe to "this campaign type only", shrinking the cost.
  - Travel example (directly relevant): don't optimise booking conversion alone; include predicted post-stay satisfaction/rating. Since the stay is months away, build a model that predicts the later rating from data available now.
  - **Sample ratio mismatch (SRM)**: if you design a 50/50 split and get 50.2/49.8 on ~1M users, a chi-square style test can say this happens about once in half a million experiments by chance — so the experiment is broken. ~8% of Microsoft experiments had SRM; third-party vendors later reported 6–10%. Common causes: bots hitting arms differently (e.g. a changed page breaks a bot's parser so it retries), data-pipeline filtering that interacts with treatment, triggering mid-funnel while a campaign pushes traffic in from elsewhere. Platform response escalated from a warning banner (ignored) to a blanked scorecard, to red lines on every number so screenshots carry the warning.
  - ~10% of experiments are aborted on day one, mostly bugs, not bad ideas.
  - **Peeking**: computing p-values continuously and stopping when p<0.05 (early Optimizely) inflates false positives from 5% to around 30%. A/A tests exposed this. Use fixed-horizon analysis or proper sequential methods.
  - **p-value misreading**: p is P(data this extreme | null true), not P(treatment better). 1−p is *not* the probability the treatment wins. To get what you actually want, apply Bayes with a prior success rate → the **false positive risk**. With Airbnb's ~8% success rate, a p<0.05 "win" has ~26% chance of being false, not 5%.
  - **Replication**: Airbnb rule — if 0.01<p<0.05, rerun; combine p-values via Fisher's or Stouffer's method for a much lower joint p. Prefer p<0.01 for wins you'll broadcast as learnings.
  - **Variance reduction** (fewer users needed, faster results): cap skewed metrics (revenue capped at e.g. $1000; nights booked capped ~30/month since agencies book hundreds); **CUPED** — adjust with pre-experiment data of the same metric; unbiased with lower variance.

- **Decisions & trade-offs**
  - When to start: below tens of thousands of users the statistics don't work for most metrics. For a retail-like conversion rate and detecting ≥5% effects you need ~200,000 users; startups should hunt 5–10% effects, not 1%. Below that, build culture and platform so you're ready.
  - Not everything is testable: one-off events like M&A lack units.
  - **Flat = no-ship.** Shipping a neutral change adds code and maintenance cost. Exception: legal requirements — but even then try ~3 implementations and ship the least harmful.
  - Big redesigns usually lose; decompose into steps (OFAT, one factor at a time) and test along the way. Bundling 17 changes when most ideas fail makes a net negative likely. Beware sunk cost after 6 months of building.
  - Crises (e.g. COVID travel collapse) are a reason to test *more*: if you're wrong 2/3–80% of the time in peace, you're not suddenly right in war. Replicate later to check external generalisability.
  - Build vs buy: today's vendors are good; usually a mix. Seed culture in a team that ships often (weekly) and has a clear OEC. If a room can't even agree on the direction of a metric (microsoft.com support site: is more time on site good or bad?), it is not an OEC.
  - Aim for marginal cost of an experiment → 0 via self-service platform; scorecard within a day, not a week waiting for an analyst. Metric templates when metric count explodes (Bing could define ~10,000).

- **Rules of thumb / checklist**
  - Run an SRM check on every experiment before reading any metric.
  - **Twyman's law**: any figure that looks interesting or different is usually wrong. If typical effects are <1% and you see 10%, investigate before celebrating; ~9 in 10 such cases had a flaw.
  - Track your org's historical success rate; use it as the prior for false positive risk.
  - Define a "surprise" as |expected − actual| large, in either direction; hold quarterly reviews of the most surprising experiments (including surprising losers: a Windows indexer improved relevance offline but killed battery life online). Keep a searchable experiment archive.
  - Hierarchy of evidence: anecdote < observational study < natural experiment < controlled experiment < multiple replicated controlled experiments. Many observational findings were later reversed by RCTs.

- **Failure modes & gotchas**: peeking; ignoring SRM; misreading p as win probability; crediting a team with a metric lacking a counterweight (email spam); presenting too-good-to-be-true results; analysis bottlenecked on analysts; shipping flat results to keep morale.

- **Relevance to Sam / Trameter**: Trameter's traffic is likely below 200k users per test, so focus on large effects, use capping and CUPED, and treat small "wins" sceptically (high false positive risk). For booking-flow changes, combine conversion with a guardrail or downstream-satisfaction proxy (cancellations, refunds, ratings). Run SRM checks when splitting traffic, including bot filtering. When reporting, say "false positive risk" honestly rather than "95% sure it works".

## Richard McElreath — Statistical Rethinking 2023, Lecture 06: Good & Bad Controls (https://www.youtube.com/watch?v=uanZZLlzKHw)
**Core idea in one line: which variables to "control for" is decided by a causal model (DAG + backdoor criterion), never by correlations, significance or timing heuristics — the wrong control can create bias that wasn't there.**

- **Mental models / first principles**
  - "Causes are not in the data." The same associations arise from different structures, so you must bring assumptions from outside the data, write them down (a DAG), and deduce their implications transparently ("be boring, not clever").
  - Three elemental structures among X, Y, Z:
    - **Fork** X←Z→Y: X and Y associated via common cause; stratifying on Z removes the association.
    - **Pipe** X→Z→Y: Z mediates; stratifying on Z also removes the association (you cannot tell a fork from a pipe from data alone).
    - **Collider** X→Z←Y: X and Y independent; stratifying on Z *creates* an association.
    - **Descendant**: conditioning on a child of a variable partially conditions on the variable itself.
  - Randomisation is the gold standard because it deletes every arrow into X: the treatment is caused only by the randomiser. The do-operator `do(X)` formalises this intervention. Goal of observational analysis: find a statistical procedure equal to P(Y | do(X)). Partial compliance ("intent to treat") breaks perfect randomisation.
  - **Do-calculus** (Pearl): three rules that tell you whether, for a given DAG, the causal effect is identifiable without functional-form assumptions. It's both a worst case (non-parametric, so with justified functional assumptions you can sometimes do more) and a best case (if it says yes, critics can't attack your functional assumptions).

- **Mechanisms**
  - **Causal effect = marginalise over controls**: P(Y|do(X)) = Σ_U P(Y|X,U) P(U). The regression coefficient alone is generally not the causal effect once there are interactions/non-linearity; you must average predictions over the control distribution of the *target population*. Example: effect of baboons on gazelle depends on whether cheetahs are present (cheetahs suppress baboon hunting), so the effect in a given reserve requires that reserve's cheetah distribution.
  - **Backdoor criterion** (graphical shortcut from do-calculus):
    1. List all paths from X to Y, ignoring arrow direction (association flows both ways).
    2. Paths starting with an arrow *into* X are backdoor (non-causal) paths.
    3. Find a (minimal) adjustment set that blocks all backdoor paths without blocking causal (front-door) paths, using the fork/pipe/collider rules.
  - Example: U unobserved → Z → X, U → Y. Stratify on Z: Z carries all of U's influence on X, so it blocks the backdoor. Simulation (n=200, true effect 0): naive regression biased upward; adding Z recovers ~0. Z's own coefficient (≈0.24, "significant") is meaningless.
  - Complex DAG walk-through: adjustment set {C, Z, A or B}. Conditioning on Z (a needed fork control) opens a collider path through A/B, so one of them must also be added. B is preferable for efficiency, not identification — **minimal adjustment set ≠ best adjustment set**. Use dagitty.net to check.
  - Sometimes you can get the total effect but not the direct effect: grandparent→parent→child education with an unobserved neighbourhood confound U of parent and child. Stratifying on parent (to isolate direct effect) makes P a collider-opened path → could even show grandparents *hurt* outcomes. "You can't always get what you want." Multiple regression answers only a narrow set of causal questions; often you need multiple simultaneous equations / a full generative model.

- **Catalogue of bad controls** (from Cinelli, Forney & Pearl, "A Crash Course in Good and Bad Controls"):
  1. **M-bias / pre-treatment collider**: U→X, U→Z, V→Z, V→Y. Z is pre-treatment but a collider; conditioning opens X←U→Z←V→Y. Example: friendship network (Z) caused by hobbies; analysing only friends conditions on it. "Pre-treatment is safe" is false.
  2. **Confounded mediator**: X→Z→Y with U→Z, U→Y. Total effect is fine without Z; adding Z (e.g. happiness when studying lottery win → lifespan) gives a biased, can-be-"significant" negative estimate even when the true effect is zero. Statistical significance is not a guide to model structure.
  3. **Hidden collider via post-treatment variable**: education → values ← family background → income. Conditioning on measured "values" creates spurious associations.
  4. **Descendant of the outcome** (case-control bias / selection on outcome): X→Y→Z. Stratifying on Z (income, when studying education→occupation) roughly halved the estimated effect in simulation. Within a level of Z, Y barely varies, so X has nothing to explain.
  5. **Precision parasite**: Z→X→Y only. No bias, but adding Z removes useful variation in X and widens the estimator's sampling distribution.
  6. **Bias amplification**: Z→X, plus unobserved U→X and U→Y. Already biased, and adding Z roughly *doubles* the bias: within levels of Z, X varies less, so the confound's relative share of X–Y covariation grows. Example: occupation→income, controlling for education (a cause of occupation) with unmeasured regional/cultural confounds — a reviewer might ask for exactly this control.

- **Decisions & trade-offs**: backdoor criterion tells you what's *needed*; estimation efficiency (finite samples, precision) is a separate, equally hard concern. Adding non-required variables can help (more precise, e.g. predictors of Y only) or hurt (parasites, amplifiers). Generative "full luxury Bayes" can do the do-calculus implicitly but is cumbersome; do-calculus tells you which sub-models matter.

- **Rules of thumb / checklist**
  - Draw the DAG first. For every pair of variables, ask whether an unobserved common cause is plausible; draw it and redo the analysis.
  - Never: add everything and keep what's significant ("YOLO"); drop collinear variables because they're correlated (collinearity has many causal origins); assume any pre-treatment/baseline variable is safe; keep adding controls until all are significant.
  - Don't control for mediators if you want total effect; don't control for descendants of the outcome; don't control for colliders; be wary of pure causes of treatment.
  - Test your estimator by simulating the DAG (e.g. 10,000 sims), varying effect sizes including zero, and checking bias and spread of estimates.
  - **Table 2 fallacy** (Westreich & Greenland 2013): coefficients on control variables aren't causal effects of those variables. HIV→stroke with age and smoking as confounders: adjustment set {age, smoking}. In that model, the smoking coefficient is only its *direct* effect (HIV mediator blocked), age's is only direct; with an unobserved lifestyle confound of smoking and stroke, smoking becomes a collider and its coefficient is outright biased, while the X estimate stays valid. McElreath's preference: don't report control coefficients, or interpret each with its own causal model.

- **Relevance to Sam**: when analysing Trameter observational data (e.g. "users who used feature F booked more"), list confounders (traffic source, device, trip type, intent) and avoid conditioning on post-treatment variables (e.g. "among users who reached checkout" is selection on a descendant and can manufacture or hide effects). Restricting analysis to converted users is conditioning on a collider/outcome. When reporting regression outputs, present only the estimand you designed for. Prefer randomised tests wherever feasible.

## Joe Blitzstein (Harvard Stat 110) — Lecture 4: Conditional Probability (https://www.youtube.com/watch?v=P7NE4WF8j-Q)
**Core idea in one line: conditioning — restricting the world to what you've observed and renormalising — is "the soul of statistics", and Bayes' rule falls out of it in one line of algebra.**

- **Mental models / first principles**
  - Statistics is the study of uncertainty; the core question is how to *update* beliefs coherently when evidence arrives. Updating is sequential: today's posterior is tomorrow's prior.
  - **Independence ≠ disjointness.** Independent: knowing A tells you nothing about B, P(A∩B)=P(A)P(B). Disjoint: if A happens B *cannot* — maximal dependence. Confusing them is a classic blunder.
  - For 3+ events, pairwise independence is not enough: you also need P(A∩B∩C)=P(A)P(B)P(C), and in general the product rule for every subset. "Independence means multiply" (for intersections).
  - Intuition is unreliable even for geniuses: Newton computed the Newton–Pepys answer correctly but gave a wrong intuitive argument. Stigler's critique: Newton's argument didn't use the dice being fair, yet the answer changes with biased dice, so the argument can't be right. Good test for any explanation: does it depend on the things the answer depends on?

- **Mechanisms**
  - **Matching problem (de Montmort)**: shuffle n labelled cards; P(at least one card in its own position) via inclusion–exclusion. Symmetry: any k specific matches has probability (n−k)!/n!, there are C(n,k) such sets, so each term collapses to 1/k!. P(no match) = Σ(−1)^k/k! → 1/e ≈ 0.37. Converges very fast (n=10 is accurate to ~1e-8). Lesson: exploit symmetry by picking a concrete case ("the first k") instead of general subscripts; factorials in denominators suggest the e^x Taylor series.
  - **Newton–Pepys (1693)**: P(≥1 six in 6 dice)=1−(5/6)^6≈0.665; P(≥2 sixes in 12 dice)=1−(5/6)^12−12·(1/6)(5/6)^11≈0.619; P(≥3 in 18)=1−Σ_{k=0..2} C(18,k)(1/6)^k(5/6)^{18−k}≈0.597. Most people (and Pepys) guess the 18-dice case; the 6-dice case wins. Technique: "at least one" → union → take complement → intersection of independent events → multiply. The C(n,k)p^k(1−p)^{n−k} form is the binomial probability; understand its derivation (choose positions, multiply), don't memorise.
  - **Definition**: P(A|B)=P(A∩B)/P(B), P(B)>0.
    - *Pebble world*: outcomes are pebbles with masses summing to 1. Observing B deletes every pebble outside B; renormalise the remainder by dividing by P(B) so it sums to 1 again.
    - *Frequentist world*: repeat the experiment many times, keep only runs where B occurred, and compute the fraction where A also occurred.
  - **Theorems**: P(A∩B)=P(B)P(A|B)=P(A)P(B|A). Chain rule: P(A1…An)=P(A1)P(A2|A1)P(A3|A1,A2)… — valid in any of n! orderings; choosing a convenient order can make a hard problem easy. Independence ⇔ P(A|B)=P(A).
  - **Bayes' rule**: P(A|B)=P(B|A)P(A)/P(B). Trivial proof, deep implications (an entire school of statistics).

- **Rules of thumb**
  - Faced with "at least one", compute 1 − P(none).
  - Use symmetry and concrete instances to simplify.
  - Sanity-check terms with an extreme case (e.g. 1/n! = all cards in order, exactly one arrangement).
  - Don't trust intuition on probability; compute, or simulate.

- **Failure modes**: equating disjoint with independent; assuming pairwise independence implies full independence; confusing P(A|B) with P(B|A).

- **Relevance to Sam**: the frequentist "keep only the runs where B happened" picture is exactly what filtering a dataframe does — so P(booked | clicked) is computed on the filtered rows, and it is not P(clicked | booked). This is the same inversion Kohavi warns about with p-values (P(data|H0) vs P(H0|data)). For matching-quality checks: if two match signals (name similarity, coordinate distance) are treated as independent to multiply their error rates, check that independence actually holds; correlated errors make combined error far higher than the product suggests. Chain rule is the basis for multi-step funnel probabilities (search→view→checkout→book).

## Brady Neal — A Brief Introduction to Causal Inference (Course Preview) (https://www.youtube.com/watch?v=DXBPtpBhGqo)
**Core idea in one line: an observed association is a mix of causal and confounding association; potential outcomes define what we want, randomisation or a sufficient adjustment set (plus marginalisation) is how we get it, and Simpson's paradox can only be resolved by knowing the causal graph.**

- **Mental models / first principles**
  - Causal inference = estimating the effect of some X (treatment, policy, intervention) on some Y.
  - **Total association = causal association + confounding association.** Correlation (strictly, *linear* dependence; "association" = any statistical dependence) can't separate the two. ML version: prediction does not imply causation.
  - "Correlation = causation" is a cognitive bias we use constantly, fed by the availability heuristic (whatever explanation you read most recently) and motivated reasoning (blaming the in-laws for the headache). Some correlations are pure chance (Nicolas Cage films vs pool drownings).
  - Two equivalent views of why naive comparisons fail: groups aren't *comparable* (shoe-sleepers were mostly drinkers), or a *common cause confounds* (drinking → shoes on, drinking → headache).
  - **Potential outcomes**: Y_i(1) is the outcome if unit i were treated, Y_i(0) if not; individual treatment effect = Y_i(1) − Y_i(0). **Fundamental problem of causal inference**: only one of these is ever observed; the other is a counterfactual.
  - Average treatment effect ATE = E[Y(1)] − E[Y(0)] = E[Y(1) − Y(0)] (linearity of expectation). In general ATE ≠ E[Y|T=1] − E[Y|T=0] because the latter contains confounding.

- **Mechanisms**
  - **Simpson's paradox (COVID-27 example)**: treatment A: 240/1500 died (16%); B: 105/550 (19%) — A looks better overall. Within mild patients: A 15% (210/1400) vs B 10% (5/50). Within severe: A 30% (30/100) vs B 20% (100/500). B is better in *both* subgroups. Cause: unequal weighting — A's overall rate is dominated by mild cases (1400/1500), B's by severe (500/550), and severe patients die more.
  - **Which is right depends on the graph, not the numbers**:
    - Scenario 1, condition → treatment (doctors save scarce B for severe cases): condition is a confounder; use subgroup numbers → choose **B**.
    - Scenario 2, treatment → condition (scarce B means waiting, and patients worsen while they wait): condition is a mediator of B's harm; use the total numbers → choose **A**.
  - **Randomised controlled trials**: the treatment is set by a coin flip, so T has no causal parents, confounding vanishes (including from *unobserved* variables) and ATE = E[Y|T=1] − E[Y|T=0]. Randomisation makes the groups comparable (drunk and sober people spread evenly across shoes/no-shoes).
  - **Why observational studies anyway**: RCTs can be unethical (smoking), infeasible (assigning countries to economic systems), impossible (DNA at birth), or just expensive/slow.
  - **Adjustment formula**: if W is a sufficient adjustment set (blocks all confounding paths), E[Y(t) | W=w] = E[Y | T=t, W=w], and E[Y(t)] = E[Y | do(T=t)] = E_W[ E[Y | T=t, W] ]. Condition on W, then average over W's *marginal* distribution.
  - Worked numbers (scenario 1): weights P(mild)=1450/2050, P(severe)=600/2050, the same for both treatments. E[Y|do(A)] ≈ 0.15·0.707 + 0.30·0.293 ≈ 19.4% deaths; E[Y|do(B)] ≈ 0.10·0.707 + 0.20·0.293 ≈ 12.9%. The naive totals used different weights per arm (that's the bug).
  - Multiple valid adjustment sets often exist (e.g. {C, W2} or {C, W1}); adding an irrelevant W3 doesn't hurt identification. Warning: in a V-shaped (collider) structure you must *not* condition on the collider.

- **Rules of thumb**
  - Before choosing aggregate vs stratified numbers, ask: does the stratifying variable cause the treatment (adjust) or is it caused by the treatment (don't adjust for total effect)?
  - The standardised estimate uses the same population weights for every arm.
  - When you can randomise, do; it handles confounders you never thought of.

- **Relevance to Sam**: classic travel analytics trap — e.g. "users who saw the Drive Button converted less" when the button is shown mainly on long-haul or expensive-trip articles (trip type → exposure and → conversion). Stratify by the confounder and re-weight to a common distribution before comparing. Conversely, if a feature changes which trips users pick (feature → trip type → conversion), stratifying by trip type hides part of the effect. Supplier comparisons (Viator vs RateHawk conversion) are Simpson-prone because the destination/product mix differs per supplier.

## Josh Starmer (StatQuest) — Linear Regression, Clearly Explained!!! (https://www.youtube.com/watch?v=7ArmBVF2dCs)
**Core idea in one line: linear regression = least-squares fit + R² (how much variance the fit explains) + an F-test p-value (whether that R² is distinguishable from chance); you need both a big R² and a small p.**

- **Mental models / first principles**
  - Three steps: (1) fit a line by least squares; (2) compute R²; (3) compute a p-value for R².
  - R² compares two models of the same outcome: the "mean-only" model (horizontal line, 1 parameter) and the fitted model. It's the fraction of variation removed by using the predictor.
  - The p-value exists because R² alone can be meaningless: with two points any line fits perfectly (R²=1) regardless of reality.

- **Mechanisms**
  - **Least squares**: residual = vertical distance from point to line. Try lines (conceptually by rotating), sum squared residuals, pick the minimum. Parameters estimated: intercept and slope; non-zero slope means the predictor helps guess the outcome.
  - **R²** = (Var(mean) − Var(fit)) / Var(mean) = (SS_mean − SS_fit)/SS_mean (the 1/n cancels). Mouse example: Var(mean)=11.1, Var(fit)=4.4 → R²≈0.6, i.e. weight explains 60% of size variation (SS version: 100 and 40). Perfect fit → R²=1; useless predictor → Var(fit)=Var(mean) → R²=0. Works for any model equation, not just lines.
  - **Multiple regression**: with two predictors (weight, tail length) least squares fits a plane; three parameters. A useless predictor gets coefficient ≈0, so **adding parameters never increases SS_fit** — R² can only stay the same or rise. Random junk (coin flip, favourite colour, star sign) can reduce SS_fit by chance → inflated R². Hence **adjusted R²**, which penalises parameter count.
  - **F statistic** = [(SS_mean − SS_fit)/(p_fit − p_mean)] / [SS_fit/(n − p_fit)] = variance explained per extra parameter ÷ unexplained variance per remaining degree of freedom. Degrees of freedom: p_mean=1; p_fit=2 for a line, 3 for a plane. Dividing residual SS by n−p_fit reflects that more parameters need more data (2 points define a line, 3 a plane).
  - **p-value for F** (conceptual): generate many random datasets with no relationship, compute F for each, build the histogram; p = fraction of random F values at least as extreme as the observed F (e.g. observed F=6). In practice, use the theoretical F distribution with those degrees of freedom. With more samples relative to parameters (larger n−p_fit), the distribution's tail falls off faster → smaller p for the same F.

- **Rules of thumb**
  - Report R² (effect size: is the relationship strong?) *and* p (reliability: is it real?). Large R² with large p (tiny n) or tiny R² with tiny p (huge n) are both incomplete stories.
  - Use adjusted R² when comparing models with different numbers of predictors.
  - Keep n well above the parameter count.

- **Failure modes**: celebrating R² from a handful of points; adding predictors to push R² up; treating a better in-sample R² as evidence of a better model (it's guaranteed by construction).
  - (note: R² measures association/explained variance, not causation; and in-sample R² says nothing about out-of-sample performance — that needs cross-validation, see the CV video.)

- **Relevance to Sam**: when modelling e.g. booking value vs lead time or price vs distance, with millions of rows almost any slope is "significant" — focus on R²/effect size and practical importance. With small samples (e.g. a 50-row hand-checked match sample) be wary of high R². Adding many features to a price or match-score regression will always raise in-sample R²; judge with adjusted R² or held-out data.

## Josh Starmer (StatQuest) — Regularization Part 1: Ridge (L2) Regression (https://www.youtube.com/watch?v=Q81RR3yKn30)
**Core idea in one line: ridge regression deliberately adds a little bias (shrinking coefficients toward zero via a λ·Σβ² penalty) to buy a large drop in variance, giving better predictions on new data when data are scarce or parameters are many.**

- **Mental models / first principles**
  - Bias–variance trade-off: least squares on very little data fits the training points perfectly but generalises badly (high variance / overfitting). Example: fit a line through 2 training points → training SSR=0, but large SSR on the held-out test points.
  - Ridge's idea: accept a slightly worse fit on training data to get more stable, less sensitive predictions — "slightly worse start, better long-term predictions".
  - Slope = sensitivity: a steep slope means small changes in the input swing predictions a lot. Shrinking slopes makes predictions less sensitive to the (noisy) training data.

- **Mechanisms**
  - Objective: minimise SSR + λ·(slope²) (in general λ·Σ of all squared coefficients, **excluding the intercept**).
  - Worked example (λ=1): least-squares line through 2 points: SSR 0, slope 1.3 → 0 + 1.69 = 1.69. A ridge line: residuals 0.3, 0.1, slope 0.8 → 0.09+0.01+0.64 = 0.74 → ridge line wins under the penalised objective.
  - λ ∈ [0, ∞). λ=0 → identical to least squares. As λ grows, slopes shrink asymptotically toward 0 (never exactly 0 — unlike lasso), and predictions become less sensitive to the predictors.
  - **Choose λ by cross-validation** (typically 10-fold): try many λ values, pick the one with the lowest CV error.
  - Works with categorical predictors: for diet (normal vs high-fat), the coefficient is the difference between group means; ridge shrinks that difference when the sample is small.
  - Works with logistic regression: penalty is added to the (log-)likelihood objective instead of SSR, shrinking the weight→obesity slope.
  - **Solving the "unsolvable"** (p > n): least squares needs at least as many data points as parameters (2 for a line, 3 for a plane, 10,001 for 10,000 gene predictors + intercept). With only 500 mice, OLS has infinitely many perfect solutions; the ridge penalty picks a unique one favouring small coefficients, with λ set by CV.

- **Decisions & trade-offs**
  - Use ridge when n is small relative to the number of predictors, or when predictors are many and noisy; you give up unbiased coefficients for lower variance.
  - (note: standardise predictors before ridge in practice, since the penalty is scale-dependent; the video doesn't mention this. Ridge coefficients are biased, so don't read them as causal or unbiased effect sizes.)

- **Rules of thumb**: tune λ by CV, never by training error; don't penalise the intercept; reach for regularisation whenever a model has many features compared with the number of labelled rows.

- **Relevance to Sam**: training a match classifier (GeoNames ↔ supplier location) on a few hundred hand-labelled pairs with dozens of engineered features is exactly the small-n/many-p regime — use L2-regularised logistic regression with CV-chosen λ. Same for price or demand models per destination where some destinations have few rows.

## Josh Starmer (StatQuest) — Decision and Classification Trees, Clearly Explained!!! (https://www.youtube.com/watch?v=_L39rN6gz7Y)
**Core idea in one line: a classification tree greedily picks, at each node, the question (feature + threshold) whose split gives the lowest weighted Gini impurity, stops when splitting no longer helps, and must be constrained (pruning or min leaf size chosen by CV) to avoid overfitting.**

- **Mental models**
  - A tree asks a true/false question at each node and routes the example; convention: true → left. Classification trees output categories; regression trees output numbers.
  - Terminology: root (top), internal nodes/branches (arrows in and out), leaves (arrows in only).
  - Trees mix numeric and yes/no features freely; the same feature can recur at different thresholds; the same class can appear in several leaves.

- **Mechanisms (building from data)**
  - For each candidate feature, build a one-split "stump", run all rows down it, and count classes per leaf. Leaves with mixed classes are **impure**.
  - **Gini impurity of a leaf** = 1 − Σ_k p_k². Example: leaf with 1 yes/3 no → 1 − (1/4)² − (3/4)² = 0.375; leaf with 3 people → 0.444.
  - **Split impurity** = leaf impurities weighted by leaf size: (4/7)·0.375 + (3/7)·0.444 = 0.405 for "loves popcorn"; "loves soda" scores 0.214.
  - **Numeric features**: sort values, take midpoints between adjacent values as candidate thresholds, compute Gini for each (e.g. age<9.5 → 0.429); best ages 15 and 44 tie at 0.343.
  - Choose the feature/threshold with lowest impurity for the root (soda, 0.214). Recurse on each child using only the rows that reach it (among soda lovers: popcorn split 0.25 vs age<12.5 → 0, so use age).
  - A node becomes a leaf when splitting can't reduce impurity. Leaf output = majority class.
  - Alternatives (entropy, information gain) behave numerically similarly to Gini.

- **Decisions & trade-offs / failure modes**
  - A leaf containing a single training example is a strong overfitting signal: you can't trust its prediction.
  - Remedies: **pruning**, or growth limits such as a minimum number of samples per leaf (e.g. ≥3). You accept impure leaves but get an honest confidence (e.g. "75% of this leaf loved it").
  - Hyperparameters like min-samples-per-leaf aren't knowable in advance → pick via cross-validation.
  - (note: greedy splitting is locally optimal, not globally; single trees are high-variance, which is why ensembles like random forests and gradient boosting are standard in practice.)

- **Relevance to Sam**: a shallow tree on match features (name similarity, distance in km, admin-level agreement, feature class) is an interpretable first match classifier and produces human-readable rules ("distance < 2 km and name similarity > 0.9 → match"). Leaf class proportions give a natural confidence score; set min leaf size so rules aren't built on one or two labelled examples. Report leaf purity, not just the majority label.

## Josh Starmer (StatQuest) — ROC and AUC, Clearly Explained! (https://www.youtube.com/watch?v=4jRBRDbJemM)
**Core idea in one line: a probabilistic classifier needs a threshold; the ROC curve plots true-positive rate vs false-positive rate across all thresholds so you can pick one for your cost trade-off, and AUC summarises the whole curve for comparing models.**

- **Mental models**
  - Logistic regression outputs P(class), not a class. Classification = probability + threshold. 0.5 is a default, not a law.
  - The right threshold depends on error costs: for something like Ebola screening, missing a positive is catastrophic → lower the threshold (e.g. 0.1), accept more false positives. In the mice example a 0.9 threshold beat 0.5 (same true positives, no false positives).
  - Many thresholds give identical confusion matrices; only thresholds between distinct scores matter. Rather than juggling dozens of matrices, summarise them in one graph.

- **Mechanisms**
  - **TPR (sensitivity, recall)** = TP/(TP+FN): share of actual positives caught. Y-axis.
  - **FPR (1 − specificity)** = FP/(FP+TN): share of actual negatives wrongly flagged. X-axis.
  - Sweep the threshold from "everything positive" (point (1,1)) to "everything negative" (point (0,0)), computing a confusion matrix and plotting (FPR, TPR) at each. Example points: (1,1) → (0.75,1) → (0.5,1) → … → a point with TPR 0.75, FPR 0 (no false positives) → (0,0).
  - The diagonal TPR = FPR is a no-skill classifier; points above/left are better. Moving left at constant TPR = fewer false positives for the same catch rate.
  - **AUC** = area under the ROC curve (example 0.9). Higher AUC = better ranking overall; use it to compare model families (e.g. logistic regression vs random forest).
  - (note: AUC also equals the probability that a randomly chosen positive gets a higher score than a randomly chosen negative — a useful interpretation not stated in the video.)
  - **Precision** = TP/(TP+FP): share of predicted positives that are right. It ignores true negatives, so under heavy **class imbalance** (rare disease, many more negatives) precision is more informative than FPR → precision–recall curves.

- **Decisions / rules of thumb**
  - Use ROC to choose the operating threshold given how many false positives you can tolerate; use AUC to choose between models.
  - With rare positives, look at precision (PR curves); FPR can look tiny while most flagged items are wrong.
  - (note: AUC is threshold-free, so a model with higher AUC can still be worse at the specific operating point you need; check the region of the curve you'll actually use.)

- **Relevance to Sam**: for location/entity matching, true matches among all candidate pairs are rare → use precision–recall, not ROC/FPR, and pick the score threshold from the cost of a wrong merge (bad booking destination) vs a missed match (manual review). Different thresholds for auto-accept vs send-to-review is a natural use of the curve.

## Josh Starmer (StatQuest) — p-hacking: What it is and how to avoid it! (https://www.youtube.com/watch?v=HDCOUXE3HMM)
**Core idea in one line: a 0.05 threshold only guarantees ~5% false positives if you compute one pre-planned p-value per test; testing many things and reporting the winner, or adding data until p dips below 0.05, quietly multiplies false positives.**

- **Mental models**
  - **p-hacking** = misuse of analysis choices that gets you fooled by false positives.
  - Under the null (both groups from the same distribution), p-values are spread out uniformly: repeated 3-vs-3 comparisons from one distribution of recovery times (95% between 5 and 15 days) gave p = 0.86, 0.63, 0.06… and eventually 0.02 — a false positive by pure chance.
  - α=0.05 ⇒ ~5% of null tests are "significant": 100 tests → ~5 false positives; 10,000 → ~500. This is the **multiple testing problem**.

- **Two forms of p-hacking**
  1. **Testing many candidates, reporting only the one that looks good** (drug A, B, … Z until one gives p=0.02). Fix: compute a p-value for *every* test and adjust all of them together, e.g. with the **false discovery rate** (Benjamini–Hochberg): adjusted p-values are usually larger, and some false positives move above 0.05. The correction only works if you feed in *all* p-values — no cherry-picking which tests to run.
  2. **Optional stopping / adding data after a near miss**: p=0.06, add one observation per group, get p=0.02. The decision now rests on two p-values, so the true false-positive rate exceeds 5%; when p is near 0.05, there is a surprisingly high chance one more point pushes a null result under the line.

- **Fix for #2: power analysis before the experiment** — decide sample size up front so you have a high probability of correctly rejecting the null if the effect is real, then collect that many and test once. If you got a borderline p, use the data you have as input to a power analysis to plan a properly sized *new* experiment, rather than topping up.

- **Rules of thumb / checklist**
  - Pre-register (at least informally): hypotheses, metrics, sample size, analysis.
  - Count all comparisons you ran (variants, segments, metrics) and correct for them.
  - Never "peek and extend" with a fixed-horizon test. (note: sequential testing designs exist that allow valid peeking; this ties to Kohavi's point that real-time p-value monitoring pushed the false positive rate to ~30%.)

- **Relevance to Sam**: when slicing Trameter experiment results by country, device, supplier and channel, some segment will show p<0.05 by chance; treat segment findings as hypotheses, apply FDR, and confirm with a new test. When comparing many matching-rule variants on the same labelled sample, the best one's score is optimistically biased; validate the winner on fresh labels. Decide test length/sample size before launching and don't extend a test because it is "almost significant".

## Josh Starmer (StatQuest) — Machine Learning Fundamentals: The Confusion Matrix (https://www.youtube.com/watch?v=Kdsp6soqA7o)
**Core idea in one line: a confusion matrix cross-tabulates predicted vs actual classes on held-out data, showing exactly what kinds of mistakes a model makes; it is the raw material for every classification metric.**

- **Mechanisms**
  - Workflow: split data into training and testing (ideally cross-validation), train each candidate method (logistic regression, k-NN, random forest…), then build a confusion matrix per method **on the test data**.
  - Starmer's layout: rows = predicted, columns = actual truth. 2×2: TP top-left (has disease, predicted disease), TN bottom-right, FN bottom-left (has disease, predicted none), FP top-right (healthy, predicted disease). Diagonal = correct, off-diagonal = errors.
  - (note: scikit-learn and many texts use the transpose — rows = actual, columns = predicted. Always check orientation before reading off FP/FN.)
  - Heart disease example: random forest TP=142, TN=110, FN=29, FP=22. k-NN TP=107, TN=79 → clearly worse on both classes, so prefer the random forest. Logistic regression's matrix was very close to the random forest's — the raw matrix can't separate near-ties; that needs summary metrics (sensitivity, specificity, ROC/AUC).
  - Multi-class: an n-class problem gives an n×n matrix (3 movies → 3×3, 40 classes → 40×40); diagonal still = correct, and off-diagonal cells show *which* classes get confused with which.

- **Rules of thumb**
  - Always evaluate on data not used for training.
  - Look at the full matrix, not only accuracy: the cost of FN and FP is rarely equal.
  - When two matrices look similar, move to metrics tied to your costs (sensitivity/recall, specificity, precision) and to threshold-independent views (ROC/PR).

- **Relevance to Sam**: when auditing a sample of location matches, record a confusion matrix of (predicted match / non-match) × (human label), not just "x% correct". With multi-class outputs (e.g. predicted place type: city / airport / region / POI), the off-diagonal cells show systematic confusions (e.g. airports labelled as cities) that point straight at the rule to fix.

## Josh Starmer (StatQuest) — Machine Learning Fundamentals: Cross Validation (https://www.youtube.com/watch?v=fSytzGwwBVw)
**Core idea in one line: you need data both to train and to test a model; cross-validation rotates which block is held out so every observation gets used for testing once, giving a fairer estimate of real-world performance for comparing methods and tuning hyperparameters.**

- **Mental models**
  - Two jobs for data: **training** (estimating parameters, e.g. the logistic curve's shape) and **testing** (checking performance on data the model hasn't seen).
  - Testing on training data is a terrible idea: it tells you nothing about new data.
  - A single 75/25 split is better but arbitrary — why the last 25%? Results depend on the luck of the split.

- **Mechanisms**
  - **k-fold CV**: split data into k blocks; for each block, train on the other k−1 and test on it; record performance; summarise across all k. Every observation is tested exactly once. Example: 4-fold comparing logistic regression, k-NN and SVM → SVM did best → choose it.
  - **Leave-one-out CV (LOOCV)**: k = n, each sample is its own test block.
  - **10-fold CV** is the common practical default.
  - **Tuning parameters** (hyperparameters not estimated from the data in the usual fit, e.g. ridge's λ): use 10-fold CV to choose the value with best held-out performance.

- **Rules of thumb / gotchas**
  - Use CV to compare model families and to tune hyperparameters.
  - (note, not in the video but essential: if you use CV to pick the model or λ, the CV score of the winner is optimistically biased — keep a final untouched test set or use nested CV. Folds must respect structure: for time-dependent data use time-based splits; for grouped data — e.g. many candidate pairs for the same place, or many bookings per user — split by group so the same entity doesn't sit in both train and test (leakage). Stratify folds by class when classes are imbalanced.)

- **Relevance to Sam**: for match classifiers trained on labelled pairs, split folds by source location (group k-fold) so near-duplicate pairs don't leak. For forecasting demand or prices, use forward-chaining time splits rather than random folds.
