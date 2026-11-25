---
name: systems-thinking-first-principles
description: Systems thinking playbook from 21 talks (Meadows, Sterman/MIT, Ackoff, Senge, Snowden, Cabrera, Cook, Dekker, Leveson, Goldratt, Munger). Use when a fix keeps failing, before changing a pipeline, booking flow, policy or incentive, for postmortems, or for business and learning strategy.
---

# Systems thinking from first principles

This skill turns 21 talks and lectures into working judgment for seeing and changing systems:
software (pipelines, booking flows, incidents), businesses, teams and Sam's own learning.
This file holds the procedure, the principles the teachers keep coming back to, and checklists.
`references/` (when present) holds detailed notes for every video, each credited to its teacher.

Use it whenever behaviour comes from interactions rather than one part, even if the user doesn't
say "systems": a bug that comes back after every fix, a retry storm, a metric stuck despite effort,
a "trade-off" that may be fake, a launch of a new rule, price, incentive or rollout, an outage
review, choosing what to build next across several ventures, or "why does this keep happening?".

**How to use it**
1. Map the system before changing it (section 1). Small maps beat big ones.
2. Run the relevant checklists (section 3) against the plan, incident or design.
3. If a `references/` folder sits next to this file, use it for depth. `references/INDEX.md`
   lists every video with its line number, or run `grep -n -i "keyword" references/*.md`. Each
   section ends with a "Relevance to Sam" note: Sam builds large location-data pipelines
   (GeoNames ~6M rows, matching/dedup) on a 16 GB laptop, supplier integrations (Viator,
   RateHawk), an LLM pipeline that turns travel blog articles into bookable trips, and runs side
   businesses (Riselist, a TikTok creator agency; Tereka, a money planner). Use those parts only
   in that context. Lines marked "(note: …)" and those Sam notes are helper additions, not the
   speaker's words. If `references/` is missing, the full notes live in the user's private GitHub
   repo `Biodun-ml/engineering-skills` (folder `systems-thinking-first-principles/references/`);
   clone it when you need depth.
4. Explain the *why*, and credit the teacher when you use a specific idea.
5. Keep it practical. Name the stock, the loop, the delay or the constraint; don't lecture.
6. The Goldratt clip is an intro only (~680 words, "focus"), and the LeanVlog video is a short
   summary. The five focusing steps below come from LeanVlog plus helper background, not from
   Goldratt's own clip. The Munger material is a secondary-source explainer (Swedish Investor).
   Hodgson and Matt P are summaries of Meadows' book, not Meadows herself.
7. Standard material the videos don't cover (Meadows' 12 leverage points and eight traps, Senge's
   five disciplines and archetypes, current Cynefin, drum-buffer-rope, Cook's 18 points) is in
   section 7, marked as background. Don't credit it to the videos.

---

## 1. Map a system before changing it

1. **Name the behaviour over time, not the event.** Draw the graph with time on the x-axis:
   growth, decline, goal-seeking, S-curve, overshoot, oscillation. One outage is an event; "the
   sync breaks every few weeks" is a pattern. *(Meadows — Philosophical Look; Paine — iceberg)*
2. **Pick a purpose, a boundary and a time horizon.** Everything connects, so the boundary is a
   choice. The horizon decides which delays matter. The hard part is where to stop.
   *(Paine; Meadows)*
3. **List elements, interconnections and the real purpose.** Interconnections are rules,
   contracts, incentives and information flows. Judge purpose by what the system actually does
   and is paid for, not the mission statement. *(Hodgson on Meadows; Ackoff — health care is
   "sickness care")*
4. **Find the stocks and the valves.** Stocks are memory (backlog, cash, data-quality debt,
   trust, supplier goodwill); they change only through inflows and outflows and can't be
   snap-changed. Classify by behaviour, not name: an interest rate behaves like a stock.
   *(Meadows; Paine; Hodgson)*
5. **Close the loops.** Write the variable you care about in the middle, ask what pushes it up
   or down, draw +/− arrows until loops close. For every "A causes B", ask how B feeds back to A.
   Polarity: odd number of negative links = balancing, even = reinforcing (trace by hand to be
   sure). *(Paine; Meadows)*
6. **Test every arrow for cause.** Each arrow names a mechanism. Test: push X, does Y move the
   same way every time, all else equal? If a sign could flip, there's missing structure: split
   the link. Correlated variables often share a hidden driver. *(Paine; Meadows)*
7. **Mark delays and nonlinearities.** Squiggle nonlinear arrows (returns saturate; regeneration
   is hump-shaped). Delays turn balancing loops into overshoot and oscillation.
   *(Meadows — fishery and Philosophical Look)*
8. **Add the other actors.** Suppliers, customers, staff, competitors each have goals; every move
   toward yours pulls the world from theirs, and they respond. *(Sterman)*
9. **Model each controller's information.** What does each actor (human or code) believe about
   the current state, where does that belief come from, and when does it go stale? Assume people
   are intelligent and acting rationally on what they can see. *(Leveson; Sterman; Paine)*
10. **Do synthesis, not just analysis.** Name the larger system this thing is part of and its
    role there; the "why" is usually outside. Use analysis to debug, synthesis to explain or
    redesign. *(Ackoff — Mechanistic to Systemic)*
11. **Sort the domain (Cynefin).** Ordered: write the recipe and automate. Complex: run parallel
    safe-to-fail probes. Chaotic: act to stabilise first. Don't call everything complex.
    *(Snowden — the talk uses these three)* Ordered splits into clear vs complicated, and "I
    don't know which" is its own domain (confused): see section 7.
12. **Stop at the small model.** ~90% of insight comes from the first 10% of modelling. Models
    show modes and leverage, not point predictions. Build in a spiral: one reference behaviour,
    then the next. Simulate once you have more than a few loops. *(Meadows; Paine; Sterman)*

---

## 2. Principles that recur across the teachers

### Structure generates behaviour
- **Same outcome from different people = a systems problem.** When everyone, everywhere,
  produces an outcome nobody wants through rational local responses, fix the structure. Swapping
  people (elements) rarely changes behaviour. *(Meadows — Sustainable Systems; Hodgson —
  basketball rules)*
- **Structure = physical plumbing + information available + decision rules.** Changing only
  one (just the IT, just incentives) is usually not enough. *(Sterman)*
- **Read reinforcing vs balancing from the curve.** Exponential growth or decline: find the
  reinforcing loop. Stuck despite pushing: find the balancing loop and its hidden goal. Both
  without delay: S-curve. Add delay: overshoot and oscillation. *(Meadows)*
- **Loop dominance shifts.** The oil argument "higher prices find more oil" holds while the
  reinforcing loop dominates and fails once depletion dominates. *(Meadows — oil example)*
  (note: "easy matches first" in dedup works the same way: each later hour buys fewer matches.)
- **There are no side effects, only effects you didn't model.** They usually push back against
  your goal. When someone says "side effect", ask what their model was missing. *(Sterman; Paine)*
- **Fundamental attribution error is low-leverage.** Blaming the person is the reflex and almost
  always wrong. Model their information and incentives until the act looks rational, then change
  the system around them. *(Sterman; Paine; Meadows)*

### Stocks, flows, delays
- **Watch stocks, not flows.** In Meadows' fishery the catch looks best just before collapse;
  the leading indicator is the fish stock. Revenue, throughput and bookings are flows; supplier
  goodwill, API quota, data-quality debt and energy are stocks. *(Meadows)*
- **Every reinforcing loop needs a balancing loop strong and fast enough.** Slightly late:
  oscillation. Much later: the system crosses a threshold where regeneration flips into erosion
  and doesn't recover. *(Meadows — fishery runs 1–3)*
- **Don't subsidise away your own signals.** Subsidies, distorted prices and noisy information
  weaken the feedback that would correct the system. Information must be clean, quick and
  true. *(Meadows)*
- **Match evaluation horizons to feedback delays.** If the consequence lands after the
  decision-maker is rewarded or promoted, the system teaches short-term fixes. *(Sterman)*
- **Ask "how much is enough?"** Pure growth on a finite base guarantees overshoot. Ask growth of
  what, for whom, at whose cost, for how long. *(Meadows)*

### Leverage points and traps
- **Leverage order: paradigm/goal > feedback structure, rules and information flows >
  elements.** Mindset produces goals, rules and incentives. Our minds go to elements because
  they're easy to see. *(Meadows; Hodgson)* (note: the full 12-point list is in section 7,
  not in the videos.)
- **Traps seen in the videos:** *success to the successful* (winners gain means to win more;
  build counter-loops on purpose) *(Meadows)*; *overshoot and collapse of a shared resource*
  (fishery run 3; with weak technology, run 1, the commons tragedy did *not* happen because
  feedback arrived in time) *(Meadows)*; *shifting the burden/addiction* (outside fix lets the
  system's own capacity waste away) and *escalation* (each side answers the other's gain, both
  bleed, shares unchanged) *(Matt P on Meadows)*. Structure produces these, not villainy. The
  other standard traps (drift to low performance, rule beating, seeking the wrong goal, policy
  resistance) are in section 7.
- **Be wary of tweaking powerful control signals directly.** Incentives, auto-scalers and global
  retry policies: small changes, big and sometimes harmful effects. Prefer Type 1 work: align
  and stabilise so existing repair produces good results. *(Cook — Bone)*

### Whole vs parts (Ackoff)
- **A system is the product of its parts' interactions, not their sum.** Essential properties
  belong to the whole; take it apart and both lose them. The car of best parts doesn't run.
  Change a part only if the whole improves (architect's principle). *(Ackoff, all four talks)*
- **Reality is a mess, not a problem.** A problem is an abstraction pulled out of a system of
  interacting problems; an exercise is a problem with the formulating information removed.
  Tackle messes by design: whole first, parts fitted to it. *(Ackoff — speech)*
- **Effectiveness before efficiency.** Doing the wrong thing right makes you "wronger"; doing
  the right thing wrong gives errors you can correct. *(Ackoff via Drucker)*
- **Aim at what you want, not at removing what you don't.** Removing defects leaves you where
  you were (the 0.01 TV-channel example). Write the target state positively. *(Ackoff)*
- **Idealised design.** Ask what you'd do right now if unconstrained; then work back to the
  feasible. Leaps come from discontinuity; continuous improvement is imitation. *(Ackoff)*
- **Dissolve before you solve.** Absolve (ignore), resolve (good enough), solve (optimal),
  dissolve (redesign so it can't happen: the striker on the back of the matchbook). The frame
  sets the solution space: elevator waits were boredom, mirrors cost ~$500. Problems are often
  best solved somewhere other than where they're noticed. *(Ackoff — speech)*
- **Model the system as what it is.** Machines have no purpose of their own; organisms seek
  survival; social systems have purposes at three levels (whole, members, environment).
  Treating companies, teams or creator networks as machines is the costly common error.
  *(Ackoff — Mechanistic to Systemic)*

### Mental models and learning
- **"The system" is a pattern we re-enact daily; culture is the assumptions we can't see.**
  Reorgs and programmes touch the surface and the pattern snaps back. Go upstream to where the
  pattern forms; include the people most affected. *(Senge, culture definition from Schein)*
- **Measures serve learning when the doers own them.** Toyota measures heavily but doesn't
  report up for control; ranking and incentive pay breed fear and "unknowable" losses. No
  feedback at all also kills learning. *(Senge via Deming)*
- **Thinking moves: DSRP.** Distinctions (what it is and isn't), Systems (parts and wholes),
  Relationships, Perspectives. Don't let over-specified instructions, or AI, take the
  structuring off your desk (the Lego-kit trap). *(Cabrera)*
- **Capability only comes from doing.** "I can't teach you anything." *(Sterman)*
- **Latticework and inversion.** A few big models from many fields carry most of the freight;
  one model = man with a hammer. Invert: list how the plan certainly fails and avoid it;
  anything times zero is zero. Protect compounding from leakages and interruptions. Write
  disconfirming evidence down at once (Darwin, ~30 minutes). *(Munger via Swedish Investor)*

### Complexity (Snowden, Leveson)
- **In complex systems you manage only boundaries, catalysts and energy.** Set flexible
  boundaries (rigid ones snap), introduce probes, feed attractors that form, dampen harmful
  ones. Attractors can be catalysed, not created. *(Snowden — children's party)*
- **Change interactions, not people.** Use cognitively diverse trios run in parallel; collect
  many raw micro-stories to catch weak signals before a threshold flips. *(Snowden)*
- **Pick the method by the kind of complexity.** Loosely coupled, independent parts:
  decompose and analyse. Many random, unorganised interactions: statistics. Tightly coupled,
  software-heavy, organised complexity (most modern systems): systems methods. *(Leveson)*

### Failure, resilience and safety (Cook, Dekker, Leveson)
- **System as imagined vs system as found.** Designs and postmortems use a static picture;
  real operations are dynamic, always under maintenance, held together by people. Work-as-done
  always differs from work-as-imagined; the gap is where success and drift both come from.
  *(Cook — Velocity 2012; Dekker)*
- **Every system runs at capacity and drifts toward failure.** Efficiency and least-effort
  gradients push the operating point toward the accident boundary. Rules push back a little,
  accidents push back hard but fade, safety campaigns do almost nothing. *(Cook 2013, after
  Rasmussen and Hirschhorn)*
- **Normalization of deviance.** Crossing the margin without harm proves luck, not safety. The
  real boundary didn't move. Ask for new evidence before relaxing a limit. *(Cook via Vaughan)*
- **Absence of negatives is not safety.** Zero-incident streaks hide bad news (Deepwater
  Horizon: ~6 years without a recordable injury). The airline with the most reported incidents
  had the lowest passenger mortality. *(Dekker)*
- **Study success: it contains the same "errors".** What separated good outcomes: someone able
  to say stop, not trusting past success in a new context, diversity and dissent, and keeping
  the risk conversation alive. *(Dekker — hospital study)*
- **Armour where there are no holes.** Counted incidents come from survivors; fatal paths are
  where monitoring shows nothing. *(Dekker via Wald)*
- **Resilience = monitor + respond + anticipate + adapt + learn, and it lives in operators.**
  Reveal the controls, show lift points, support mental simulation, open black boxes, no "don't
  touch" zones, continuous maintenance. *(Cook — Velocity 2012)*
- **Resilience is costly and continuous.** Like bone, strength comes from always repairing
  small damage, guided by local strain. Starve it of energy, feedback or slack and it fails;
  load beyond repair rate and damage piles up. Aim for *graceful extensibility* (a boundary
  that stretches, not a cliff) and *sustained adaptability*. *(Cook — Bone, citing Woods)*
- **Safety is a control problem.** Many accidents have no failed part (A320 at Warsaw: software
  blocked reverse thrust because it believed the plane was airborne). Failure = a controller's
  process model diverging from reality. Work top-down from losses you won't accept (STPA).
  Fixing late can cost ~1000×. Distrust single probability numbers: "if you know enough about
  a design error to put a number on it, fix it instead." *(Leveson)*
- **Latent time bombs.** Reasonable settings plus a single-day, single-vendor rollout gave the
  infusion-pump fleet failure a year later. Rarely-used features "don't work as advertised".
  *(Cook — Velocity 2012)*

### Constraints
- **Find and feed the constraint.** Throughput is set by one constraint. Five steps: identify;
  exploit (never idle, never fed bad input, buffer and check quality before it); subordinate
  everything else; elevate (spend) only after that; repeat, and don't let old rules become the
  new constraint. Improving a non-constraint only builds inventory. Judge by money throughput
  first, then inventory, then operating expense. *(LeanVlog on Goldratt; Goldratt's clip only
  says it all reduces to "focus")* (note: in Goldratt's own definition throughput is sales
  minus truly variable costs, not revenue; drum-buffer-rope, which schedules flow around the
  constraint, is in section 7.)

---

## 3. Checklists

**Where's the leverage?**
- [ ] Is this an event or a pattern? Did I draw behaviour over time?
- [ ] Am I changing an element (person, tool, supplier) when a rule, incentive, information flow
      or goal would do more? *(Meadows; Hodgson)*
- [ ] Which stock really matters, and who can see it, how fast? Can I shorten that feedback
      delay? *(Meadows; Paine)*
- [ ] Which loop dominates now, and which balancing loop will take over later?
- [ ] What is the one constraint on throughput? Am I working on it or on a non-constraint?
      *(LeanVlog)*
- [ ] Can the problem be dissolved by redesign, or solved somewhere other than where it shows?
      *(Ackoff)*
- [ ] Does the change improve the whole, not just my part? *(Ackoff)*
- [ ] Is this clear (recipe), complicated (expert analysis), complex (probes), chaotic
      (stabilise) or confused (split it up first)? *(Snowden; section 7)*
- [ ] Am I fixing a symptom in a way that atrophies the system's own capacity? *(Matt P)*
- [ ] Am I only tuning a parameter (bottom of Meadows' list) when a delay, an information
      flow, a rule or the goal could change? *(section 7)*

**Before launching a policy, price, incentive or rollout (unintended consequences)**
- [ ] Inverted: written down how this certainly fails, and designed those paths out. *(Munger)*
- [ ] Listed every other actor whose goals this moves away from, and their likely response.
      *(Sterman)*
- [ ] Looked for gaming: does it rank people on one metric or report up for control?
      *(Senge/Deming)*
- [ ] Checked for escalation (they'll answer in kind) and success-to-the-successful loops.
- [ ] Will people hit the metric without hitting the purpose (rule beating, seeking the wrong
      goal, Goodhart)? Does the target quietly drop each time it's missed (drift to low
      performance)? *(section 7)*
- [ ] Does it weaken a signal I need (subsidy, hidden cost, delayed feedback)? *(Meadows)*
- [ ] Evaluation horizon at least as long as the feedback delay. *(Sterman)*
- [ ] What workaround will the rule provoke in work-as-done? What does it cost the doer?
      *(Dekker)*
- [ ] Run as a safe-to-fail, reversible micro-experiment with abort criteria and a control
      group, not a big bang (Woolworths: trial first, then ~1,000 stores). *(Dekker; Snowden)*
- [ ] Staged rollout; no single-day fleet-wide change; expiry dates and scheduled flips
      registered. *(Cook)*

**Incident / postmortem without blame**
- [ ] Iceberg: event → pattern (has this happened before?) → structure (plumbing, information,
      decision rules). *(Paine)*
- [ ] Assume people acted rationally on what they could see; reconstruct what each actor and
      component believed, and why. *(Sterman; Leveson)*
- [ ] Was it a non-failure accident: every part did what it was told, the interaction was
      unsafe? *(Leveson)*
- [ ] Unsafe control actions: provided, not provided, wrong timing, wrong duration. *(Leveson)*
- [ ] Describe work-as-done, including GSD adaptations, not the runbook version. *(Dekker; Cook)*
- [ ] Was a margin crossed before without harm and quietly relaxed? *(Cook — normalization)*
- [ ] No single "root cause": list the contributing conditions and the defences that held.
      Write the timeline from what people knew then, not with hindsight. *(Cook's 18 points,
      section 7; Leveson's CAST is the same idea)*
- [ ] Fix structure (validation, shorter feedback delay, headroom, guardrails), never "be more
      careful" or a reliability meeting. *(Paine; Cook)*
- [ ] Use the post-incident window to buy structural margin; the urgency fades. *(Cook 2013)*
- [ ] Look for where there are no holes: which outputs have no checks at all? *(Dekker/Wald)*
- [ ] Also study a recent success the same way. *(Dekker)*

**Resilience and safety in design**
- [ ] Losses we refuse listed first; constraints derived from them. *(Leveson)*
- [ ] Each controller's process model and its feedback source known; staleness handled.
- [ ] Distance-to-failure visible (headroom, queue depth, error-budget burn). *(Cook)*
- [ ] Operators can see state and act: re-run, replay, pause; no "don't touch" zones. *(Cook)*
- [ ] Restore, failover and replay paths tested before they're needed. *(Cook)*
- [ ] Someone can say stop and is heard; dissent is welcome. *(Dekker)*

---

## 4. Failure modes to hunt for

- Blaming the person or swapping a tool when the same outcome recurs across people.
- Dashboards of flows (revenue, rows processed) with no view of the underlying stocks.
- Reacting to stale state through a delay: over-ordering, retry storms, overcorrection.
- Optimising one stage, team or tool by its own metric; best-of-breed parts that don't fit.
- Defect-hunting as strategy; polishing a process that shouldn't exist.
- Open-loop plans (goal → decision → done) with no feedback about where you are now.
- Ordered tools (targets, milestones, best-practice docs) applied to complex situations, or
  "it's complex" used as an excuse where a recipe would work.
- Celebrating zero-incident streaks; suppressing reports; rules that push workarounds underground.
- Relaxing a limit because "we crossed it 100 times and nothing happened".
- Manual heroics that never become an automated check (shifting the burden).
- Single probability numbers for complex software; big-bang rollouts; untracked expiries.
- Reliability talks and "try harder" campaigns instead of structural change.
- A man with a hammer: an LLM or new framework for what a join, a constraint or a cron would do.
- Eroding goals: the match-rate or latency target quietly lowered to whatever last month did.
- Releasing work faster than the constraint can take it (no rope): growing queues, memory
  spikes, rate-limit bans.

---

## 5. Where teachers disagree, and how to resolve it

- **Control vs freedom.** Leveson enforces constraints through control loops; Dekker and Snowden
  favour fewer rules, freedom within a frame and self-organisation. Resolve by domain: hard
  constraints for unacceptable losses (money, data, safety) and ordered work; freedom within a
  frame for complex, people-heavy work. Both agree: control interactions, not just components.
- **Reliability vs resilience.** Locking down and hiding (reliability) can remove the operator
  access that resilience needs *(Cook)*. Keep guardrails on catastrophic actions, but expose
  state and controls to whoever is on call.
- **Measure vs don't.** Goldratt/LeanVlog want three money measures; Senge warns that what's
  measurable isn't what's most real and that upward-reported metrics breed fear; Dekker says
  negative counts predict nothing. Resolve: measure throughput and stocks, let the doers own
  the numbers, and track capacity signals (stop calls, dissent) over absence of incidents.
- **Continuous vs discontinuous.** Ackoff dismisses continuous improvement as imitation; Cook
  shows continuous small repair is what keeps systems strong. Resolve: continuous maintenance
  for health, idealised redesign for direction.
- **Model vs probe.** Sterman and Paine simulate; Snowden says complex systems can't be
  engineered, only probed. Model when structure is knowable (stocks, flows, rate limits);
  probe when it isn't (markets, creator virality, adoption).

---

## 6. Applying this to Sam's work

- **Pipelines (GeoNames, matching, GTFS).** Unmatched rows are a stock; ingest and match rates
  are flows; expect diminishing returns as easy matches deplete. A job that "succeeded" but
  wrote wrong data is a non-failure accident: assert row counts, key uniqueness and geographic
  bounds on outputs. Each stage writes inspectable Parquet/CSV (lift points). Prefer
  dissolving: tighten the ingest contract so bad rows can't enter. *(Leveson; Cook; Ackoff)*
- **16 GB laptop.** Find the true constraint (RAM peak, disk I/O, a single-threaded join, a
  rate-limited API) before parallelising. Log peak memory per run and keep an explicit margin
  (helper heuristic: ~70–80% RAM); a run that squeezed through at 15.5 GB is luck. Use
  drum-buffer-rope: the constraint sets the pace, a small bounded queue sits in front of it,
  and new chunks are released only as it drains (bounded executor or queue, not "submit all").
  *(LeanVlog; Cook — normalization; DBR from section 7)*
- **Booking flows (Viator, RateHawk).** Most incidents are process-model divergence: stale price,
  timeout treated as failure while the supplier booked, retry sending a second booking. List
  losses first ("never capture payment without a supplier confirmation ID"), test the emergent
  property end to end, keep an expiry register for keys, certs and sandboxes, and treat rate
  limits as a fishery: aggressive polling erodes supplier goodwill. *(Leveson; Ackoff; Meadows)*
- **Drive Button LLM pipeline.** Decide explicitly who has final say over injected links and
  keep the human's model accurate; a chain of AI steps that "always worked" needs re-checking
  when inputs shift. *(Leveson; Dekker)*
- **Riselist.** Social system: creators, clients and the agency have their own purposes. Change
  connectors (payment terms, briefs, incentives) before swapping creators; avoid single-metric
  creator rankings; watch rate-bidding escalation; for growth, set boundaries, seed catalysts,
  feed the formats that take off. *(Ackoff; Hodgson; Snowden)*
- **Tereka.** Personal finance is a bathtub: balance = inflows − outflows; each cup is a stock.
  The product's job is to show the gap to goal and shorten the checking delay so the balancing
  loop corrects earlier. *(Hodgson; Paine)*
- **Sam's own time and learning.** Two jobs, study and several ventures put him near the
  workload boundary; drift toward burnout or a dropped client is the default unless slack is
  kept deliberately. Find the one constraint (often his hours or distribution, not code);
  unfinished features are inventory. Learn by doing, keep DSRP on his own desk when building
  with AI, protect compounding (skills, code, audience) from context-switching leakage, and
  turn every manual fix into an automated check. *(Cook 2013; LeanVlog; Sterman; Cabrera;
  Munger; Matt P)*

---

## 7. Not covered by the sources, but know this

> **Background box.** Standard field knowledge the 21 videos don't contain. Use it, but credit
> the book or paper named here, never the videos or `references/`.

**Meadows' 12 leverage points** ("Leverage Points: Places to Intervene in a System", 1999;
*Thinking in Systems*, 2008), weakest to strongest: (12) constants and parameters (timeouts,
thresholds, prices; where almost all attention goes, rarely changes behaviour); (11) buffer and
stabilising-stock sizes relative to flows (queues, caches, cash reserve); (10) structure of
material stocks and flows (plumbing, hard to change after build); (9) delays relative to the
rate of change (often fixed; slow the system instead); (8) strength of balancing loops
(monitoring, validation); (7) gain of reinforcing loops (slowing a runaway loop beats a
stronger brake); (6) information flows, who sees what when (cheap, high leverage); (5) rules,
incentives, contracts; (4) power to self-organise and change structure; (3) goals; (2) the
paradigm the system arises from; (1) power to transcend paradigms. Meadows warns people often
find the right point and push it the wrong way; treat the order as a guide, not a law.

**Meadows' eight system traps and the way out** (*Thinking in Systems*, ch. 5)
- *Policy resistance* (actors pull toward their own goals): find a shared goal or stop pushing.
- *Tragedy of the commons* (erosion feedback delayed or invisible): educate, privatise, or
  regulate by mutual agreement; make the erosion signal visible.
- *Drift to low performance / eroding goals* (target anchored to past results): keep
  standards absolute; anchor to the best performance, not the worst.
- *Escalation*: refuse to compete, or negotiate a balancing loop (caps, agreements).
- *Success to the successful*: diversify, cap the advantage, reset periodically.
- *Shifting the burden / addiction*: strengthen the system's own capacity; plan the exit.
- *Rule beating* (letter obeyed, purpose defeated): point rules at the purpose.
- *Seeking the wrong goal* (indicator replaces purpose; Goodhart): measure the outcome you
  want, not effort.

**Senge's five disciplines** (*The Fifth Discipline*, 1990): personal mastery, mental models,
shared vision, team learning, and systems thinking (the fifth, which integrates the others).
His archetypes overlap with Meadows' traps; also worth knowing: *fixes that fail* (fix relieves
the symptom, delayed side effect worsens it), *limits to growth* (push the limit, not the
growth), *growth and underinvestment* (capacity built too late, so standards drop).

**Cynefin, current naming** (Snowden / The Cynefin Co; decision models from Snowden & Boone, HBR
2007). The talk only uses ordered/complex/chaotic; clear + complicated = "ordered".
- *Clear* (earlier "simple", then "obvious"): sense–categorise–respond, best practice,
  automate. Risk: complacency, then a sudden fall into chaos.
- *Complicated*: sense–analyse–respond, good practice, experts; several right answers.
- *Complex*: probe–sense–respond, emergent/exaptive practice, safe-to-fail probes.
- *Chaotic* (Chaos): act–sense–respond, novel practice; stabilise first, then move it.
- *Confused* (centre; earlier "disorder"; also called confusion/aporia): you don't know which
  domain you're in. Break the situation into parts and assign each part to a domain.

**Drum-buffer-rope** (Goldratt, *The Goal* / *The Race*). *Drum*: the constraint sets the pace
for the whole flow. *Buffer*: a time/work buffer in front of the constraint (and before
shipping) so upstream variation never starves it. *Rope*: release new work only as the
constraint consumes it, which caps work-in-progress. Software mapping: a rate-limited supplier
API or peak RAM is the drum; a bounded queue is the buffer; backpressure, bounded worker pools
or token buckets are the rope. TOC measures: throughput = sales minus truly variable costs;
inventory = money tied up in things meant to be sold; operating expense = money spent turning
inventory into throughput. Constraint ≠ any busy resource: it's the one that caps system output.
For projects, Goldratt's critical chain pools task buffers at the end instead of padding each task.

**Cook, "How Complex Systems Fail" (1998 paper, 18 points)**. The Velocity 2012 talk covers
the theme, not the list. (1) Complex systems are intrinsically hazardous; (2) heavily and
successfully defended against failure; (3) catastrophe needs multiple failures, single points
are not enough; (4) they contain changing mixtures of latent failures; (5) they run in degraded
mode; (6) catastrophe is always just around the corner; (7) post-accident attribution to a
"root cause" is fundamentally wrong; (8) hindsight biases assessments of human performance;
(9) operators are both producers and defenders against failure; (10) all practitioner actions
are gambles; (11) actions at the sharp end resolve all ambiguity; (12) practitioners are the
adaptable element; (13) human expertise is constantly changing; (14) change introduces new
forms of failure; (15) views of "cause" limit the effectiveness of defences; (16) safety is a
property of systems, not components; (17) people continuously create safety; (18) failure-free
operations require experience with failure. Most useful in reviews: 3, 5, 7, 8, 14, 17.

**Other tools worth naming**
- *Little's law*: items in system = arrival rate × time in system. Use it to size queues and
  read backlog delays.
- *Hollnagel's four resilience abilities* (Safety-II): respond, monitor, anticipate, learn.
  *Ashby's requisite variety*: a controller needs as much variety as its disturbances.
- *Modelling software*: causal loops on a whiteboard first; Vensim PLE, Stella, Insight Maker
  for stock-flow; PySD runs Vensim/XMILE models in Python; NetLogo or Mesa for agent-based.

---

## 8. Where to look in `references/`

| Need | File (see INDEX.md) |
|---|---|
| Meadows fishery and sustainable systems, Leveson STAMP/STPA, Cook on bone and resilience types, Cook "How Complex Systems Fail" (Velocity 2012 talk, not the 18-point paper), Munger latticework/inversion, Ackoff TED-style talk, LeanVlog five focusing steps | `systems-1-meadows-leveson-cook-investor-ackoff-leanvlog.md` |
| Senge Fifth Discipline, Ackoff education speech (dissolving problems, elevator, four treatments), Meadows system dynamics method, Snowden Cynefin, Cabrera DSRP, Ackoff Beyond Continual Improvement, Goldratt "focus" clip | `systems-2-senge-ackoff-meadows-snowden-cabrera-goldratt.md` |
| Ackoff analysis vs synthesis and system types, Paine (MIT) causal loops and stocks, Dekker drift and Safety-II, Cook Rasmussen model and normalization of deviance, Sterman intro, Hodgson elements/interconnections/purpose, Matt P system traps | `systems-3-ackoff-paine-dekker-cook-sterman-hodgson.md` |
