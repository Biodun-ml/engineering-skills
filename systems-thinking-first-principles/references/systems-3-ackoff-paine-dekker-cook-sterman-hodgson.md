# systems-3 — cross-cutting principles

1. **A system's essential properties belong to the whole and come from interactions, not parts.** Take it apart and both the whole and the parts lose those properties; you can't verify "a confirmed, honoured booking" by testing services one at a time. (Ackoff)
2. **Analysis gives know-how; synthesis gives understanding.** To debug, take it apart. To explain or redesign, first name the larger system it belongs to and its role there; the "why" is usually outside the thing. (Ackoff)
3. **Producers are necessary, not sufficient: the environment completes every explanation.** No universal laws, only laws that hold in a given environment. Ask what surrounding conditions made a result possible. (Ackoff / Singer)
4. **Model a system as the kind of system it is.** Machines have no purpose of their own; organisms seek survival and growth; social systems have purposes at three levels (the whole, its members, the wider environment). Treating companies, teams or creator networks as machines is the usual costly mistake. (Ackoff)
5. **Structure generates behaviour.** Structure = physical plumbing + what information is available + the decision rules people use. Read situations as events -> patterns -> structure, and fix the structure. (Sterman; Paine)
6. **Close the loop: there are no side effects, only effects you didn't model.** Decisions change the state of the system, which changes later decisions and goals. Other actors with their own goals push back, and every link has delays. (Sterman; Paine)
7. **Fundamental attribution error is low-leverage.** Start by assuming people are intelligent, care, and act rationally on the information they have. Model their information and incentives until their behaviour makes sense, then change the system around them. (Sterman; Paine; Dekker)
8. **Stocks are memory and change only through flows.** You cannot snap-change a stock (backlog, debt, data quality, trust). Only inflow or outflow moves it. Check whether a variable is a stock or a flow by how it behaves, not by its name (an interest rate is a price, so a stock). (Paine; Hodgson; Matt P)
9. **Loop polarity drives dynamics.** Reinforcing loops run away in either direction (virtuous or vicious). Balancing loops seek a goal, and their delay sets how badly they oscillate. If a link's sign could flip, there is missing structure in between. Test whether a link is causal by asking: does pushing X move Y the same way every time, all else equal? (Paine; Hodgson; Matt P)
10. **Interconnections and rules beat elements as leverage.** Changing the rules, incentives or information flows changes a system far more than replacing its people or components. A system's real purpose is what it actually does. (Hodgson / Meadows)
11. **Watch for system traps.** *Shifting the burden / addiction*: outside fixes cause the system's own capacity to waste away. *Escalation*: rivals in a reinforcing loop bleed each other while neither gains share. Structure, not villainy, produces both. (Matt P / Meadows)
12. **Models exist for leverage, not prediction.** System dynamics predicts *modes* (oscillation, overshoot) and which structures change them, not point values. Build in a spiral, one reference behaviour at a time. A good fit to history isn't the goal. (Paine; Sterman)
13. **Every system runs at capacity, and drift toward failure is the default.** Efficiency pressure and least-effort pressure push the operating point toward the accident boundary. Rules nudge it back briefly. Accidents push it back hard, but the effect fades. Safety campaigns do almost nothing. (Cook / Rasmussen / Hirschhorn)
14. **Normalization of deviance: crossing the margin without harm proves luck, not safety.** People keep moving the safety margin outward ("we did it 100 times"). The real accident boundary hasn't moved, so ordinary variation eventually finds it. Keep explicit, instrumented margins and make distance-to-failure visible. (Cook / Vaughan)
15. **Counting the absence of negatives predicts nothing and breeds risk secrecy.** Zero-incident streaks hide bad news (Deepwater Horizon went six years without injuries before the blowout). The airline with the most reported incidents had the lowest passenger mortality. Aim for risk competence. (Dekker)
16. **Study success, because failures and successes contain the same "errors".** What separated good outcomes from bad ones was the presence of adaptive capacities: someone able to say stop under pressure, not trusting past success in a new context, diversity and dissent, and keeping the risk conversation alive. (Dekker)
17. **Put armour where there are no holes (survivorship bias).** The incidents you can count come from the survivors. The fatal paths are where your monitoring shows nothing. (Dekker / Wald)
18. **Work-as-imagined ≠ work-as-done.** Every rule-bound job needs "get stuff done" adaptations. They are the source of success and also where drift starts. Study them, don't just ban them. Give people freedom within a frame, and test changes with safe-to-fail, reversible micro-experiments that have abort criteria (Woolworths: a ~1,000-store rollout after a randomized trial). (Dekker)
19. **Resilience is the ongoing operator work of monitoring, reacting, anticipating and learning.** It explains why systems mostly stay up. Invest in operators' ability to see where the operating point is, rather than in exhortation. (Cook)
20. **Match evaluation horizons to feedback delays.** If consequences arrive after the decision-maker has been rewarded or promoted, the system teaches short-term fixes that cause long-term damage. (Sterman)

---

## Russell L. Ackoff — From Mechanistic to Systemic Thinking (https://www.youtube.com/watch?v=yGN5DBpW93g)
**Core idea: a system is a whole that cannot be split into independent parts, so analysis gives you know-how (how it works) but only synthesis (looking outward at the containing system) gives you understanding (why it is the way it is).**

- **Mental models / first principles**
  - Every explanation sits on a "world view" (an unconscious theory of reality). Ackoff argues the Machine Age world view (Renaissance to mid-20th century) rested on three pillars: (1) the universe is completely understandable, (2) analysis is *the* method of thought, (3) cause-and-effect is the only relationship needed. Together they give determinism, reductionism (everything reduces to indivisible elements: atoms, cells, phonemes, Taylor's "work elements") and an *environment-free* theory of explanation (laws of "freely" falling bodies in a vacuum; the lab is a place built to exclude the environment).
  - **Definition of a system** (three conditions): (a) every part can affect the behaviour of the whole; (b) how each part affects the whole depends on at least one other part (parts form a connected set, none has an independent effect); (c) any subgroup of parts also has properties (a) and (b). Poetic version: a whole that cannot be divided into independent parts.
  - Consequences: the *essential properties of a system are properties of the whole that no part has* (no part of a car can carry you; your hand can't write, you write). Take a system apart and it loses its essential properties — and so do the parts (an engine on the floor moves nothing). A system is not the sum of its parts but the *product of their interactions*.
  - **Analysis vs synthesis.** Analysis: take apart -> explain each part -> aggregate. Synthesis: (1) identify the larger system it is part of, (2) explain the behaviour of that containing system, (3) disaggregate to explain the role/function of the thing within it. Analysis yields knowledge/know-how (instructions, how to repair). Synthesis yields understanding (explanations, why). Example: no amount of dismantling cars explains why the UK drives on the left or why early cars had six seats (average US family ~5.6); the answer is outside the car. Systems thinking = using both, depending on whether you need knowledge or understanding.
  - **Expansionism**: understanding grows as you comprehend larger containing systems; knowledge grows as you comprehend smaller elements. Neither terminates.
  - **Producer–product (Singer, 1898)** replaces strict cause-effect: a producer is *necessary but not sufficient* (an acorn needs soil, moisture, etc. to make an oak). The other necessary conditions *are* the environment, so all explanation becomes environment-dependent; there are no universal laws, only laws valid within an environment. "Probabilistic causality" is a contradiction in terms. Producer-product and cause-effect are complementary slices through the same reality (orange cut two ways); producer-product makes purpose and choice representable.
  - **Three kinds of system**: *machines* (no purpose of their own; serve an external purpose — the owner/God), *organisms* (own purpose: survival, which requires growth; parts have functions, not purposes; environment assumed passive), *social systems* (the whole, its parts, AND the containing system all have their own purposes). Managers now face three levels of purpose that are rarely compatible: the enterprise's, its members', and the wider society/environment's (worker movements, consumer and ecological movements = parts and neighbours pushing back). The common error is treating organisms and social systems as machines ("the only business of business is business" is the machine view).
  - Technology of the systems age is about symbols, not energy: instruments (observation), telegraph/radio (communication), computer (logical symbol manipulation). Put together these approximate a mind; *automation* of mental work replaces *mechanization* of physical work.

- **Mechanisms / examples given**: Taylor's work analysis reduced jobs to elementary tasks, mechanized what was cheap to mechanize, gave the rest to people, then aggregated into a production line — an assembly line is literally analytic thinking made physical. Side effect: people were made to behave like machines -> alienation, and (per Ackoff) falling labour productivity once separated from capital productivity. Firms went public after WWI because growth opportunities exceeded what reinvested profit could fund, which dissolved the "owner as god" and shifted business language to biology (head, corporation, "brain of the firm").

- **Decisions & trade-offs**
  - Use analysis when the question is "how does it work / what broke?" (debugging, repair). Use synthesis when the question is "why is it like this / what should it be?" (design, strategy).
  - Choose the model that fits the system type: mechanical model for a machine, organism model for a person, social-system model for a company or market. Modelling a company or a team as a machine is the cheap, common, costly mistake.

- **Rules of thumb**
  - Before optimizing a component, name the containing system and the component's function in it. A part optimized in isolation can make the whole worse because its effect depends on other parts.
  - When you cannot explain behaviour by looking inside, look outside: environment, upstream, customers, regulations.
  - Something with no effect on the whole isn't a part (Ackoff's appendix joke) — useful test for scope.
  - Expect stakeholders inside (staff, contractors) and outside (customers, suppliers, regulators) to have purposes of their own; design for them or they will "mess you up".

- **Failure modes**: reductionist debugging that never reaches understanding; believing in complete understanding/forecastability (the 1850 scientists predicting science would be complete by 1900); environment-free explanations; dehumanising work by splitting it into machine-like tasks.

- **Relevance to Sam**
  - *Booking flow / supplier integrations*: a booking's essential property ("customer gets a confirmed, honoured trip") belongs to the whole chain — search, pricing, supplier API (Viator/RateHawk), payment, confirmation email. No single service has it; testing each service alone can't prove it. Use end-to-end checks on the emergent property.
  - *Pipelines*: why a GeoNames/OSM match rate is what it is usually lies outside the code (how upstream data was collected, what the product needs). Ask "what larger system is this dataset a part of and what is its role?" before tuning the matcher.
  - *Side businesses*: Riselist and Tereka are social systems — creators, clients, users each have purposes. Treating creators as interchangeable machine parts produces churn; aligning the three levels of purpose (business, members, market) is the management job.
  - *AI*: Ackoff frames computers as automation of observation, communication and reasoning — the framing for Sam's "use AI to build" approach: automate symbol work, keep humans for purpose-setting.

## James Paine (MIT System Dynamics Group) — System Dynamics: Systems Thinking and Modeling for a Complex World (https://www.youtube.com/watch?v=o-Yp8A7BPE8)
**Core idea: structure generates behaviour — close the loops in your "open-loop" decision model, look below events to patterns to structure, and fix the system rather than blaming the people in it.**

- **Mental models / first principles**
  - Paine's one-line definition: system dynamics is *control theory applied to social systems*. Founded by Jay Forrester (inventor of core memory/RAM, Whirlwind) with *Industrial Dynamics* (1958). First case: a GE appliance plant with a 3-year hire/quit boom-bust cycle — the cause was internal policy structure, not the market. Later WORLD2 -> *Limits to Growth* (often misread as point predictions; it was about overshoot-and-collapse *modes*).
  - **Open-loop vs closed-loop thinking.** The standard project/DMAIC picture is linear: goal -> decision -> change system state -> done. Reality: the new state changes your future decisions and even your goals; it triggers *side effects*; other actors (suppliers, customers, other teams) change *their* decisions in response, which feeds back into your state; and every link has *delays*. "There are no side effects, only effects you haven't thought about yet."
  - A system = interdependent parts with a common purpose; since everything connects to everything, you must *pick a purpose and draw a boundary* — the boundary is a modelling choice.
  - Social/economic systems are dynamic, tightly coupled, feedback-governed, non-linear, with limited information and ambiguity — usually more complex than engineered physical systems.
  - **Iceberg: events -> patterns of behaviour -> structure.** Headlines about oil prices (a drunk trader, OPEC rumours) are events; the boom-bust oscillation is the pattern (with growing noise); the structure underneath is physical structure + information availability + actors' mental models (decision rules — "tiny PID controllers").
  - **Fundamental attribution error**: our first instinct is to blame the person, not the system (the driver who cut you off; you do the same when your kid is sick). MIT SD group's "basic assumption": everyone is intelligent, capable, cares, acts with integrity and wants to learn. Accept that, and what must change is the system they work in.
  - Learning in dynamically complex settings is hard: limited information and time delays are the big reasons. Expecting people to act on information they don't have is expecting omniscience.

- **Mechanisms (tools)**
  - **Causal links** with polarity: production +-> inventory, shipments −-> inventory, price −-> orders. If a link's sign could flip ("more salespeople -> more orders… until they annoy customers"), there is *missing structure* in between — split the link.
  - Causal ≠ correlated: ice cream sales and murder rate both driven by temperature. Paine's working test: push one variable, does the other move the same way *consistently, every time, all else equal*? If not, it is not a direct link.
  - **Loops**: Reinforcing (R) — employee skill -> customer satisfaction -> fewer complaints -> less manager firefighting -> more coaching time -> more skill; runs away in either direction (virtuous *or* vicious). Balancing (B) — market attractiveness -> more competitors -> lower price -> lower profit -> lower attractiveness; goal-seeking loops close a gap to a target. Polarity shortcut: count negative links; even = reinforcing, odd = balancing (only valid if each link is labelled correctly — trace it by hand to be sure).
  - **Stocks and flows**: a stock is the system's *memory* (accumulation of inflow minus outflow) — employees, inventory, cash, CO2 in the atmosphere. Bathtub: the level only changes by changing inflow or outflow; for atmospheric CO2 the only levers are cutting net emissions or increasing net removal. Balance sheet = stocks; cash-flow statement = flows; wealth vs income/expenditure. Trick: an interest rate is a *price*, i.e. a sticky stock, despite "rate" in the name — classify by behaviour, not by the word.
  - **Spiral modelling**: pick one *reference mode* (an observed behaviour over time), build structure that reproduces it, add the next reference mode, add structure, rework. When interviews give six stories and three outcomes, model one at a time then reconcile. Revise your own mental model as you go.
  - Simulation is cheaper than experimenting on real social systems ("takes a long time and makes people grumpy"). Management flight simulators (Fishbanks — a commons/overfishing game; the Beer Game) teach by experience.

- **Decisions & trade-offs**
  - "All models are wrong, some are useful." The purpose is not fit-to-history but finding **high-leverage policies**. Paine admits over-spending time curve-fitting to reference data — being right ≠ being useful.
  - SD is good at predicting *modes* of behaviour (oscillation, overshoot, amplitude/period and which structure changes them), poor at point predictions far from today. Use it to ask "what structure would change this pattern?", not "what will the price be on date X?".
  - Tools: whiteboard first; Vensim, Stella Architect for stock-flow; NetLogo for agent-based ("I understand one agent, not the system"); Python/TensorFlow are fine — SD is a way of thinking, not a diagram type. En-ROADS climate simulator as a policy-alignment tool.

- **Rules of thumb**
  - How to start: write the variable you care about in the middle of a whiteboard; ask what pushes it up or down; draw arrows with +/−; keep going until loops close.
  - When someone's behaviour looks irrational, model *their* information and incentives until it looks rational, then place that model in the larger system.
  - Information helps but doesn't save you: even professors who teach the Beer Game reproduce its oscillations. Changing behaviour durably needs a change in **structure** (delays, information flows, decision rules), not exhortation.

- **Failure modes**: Vasa warship (1628) — late design additions (bigger cabin, more sails, more cannon) made it unstable; it sank on launch. Scope creep without re-checking whole-system stability. Mistaking correlation for cause. Treating side effects as someone else's problem. Six Sigma's 3.4 DPMO target applied arbitrarily (Paine's aside).

- **Relevance to Sam**
  - *Pipeline incidents*: write a blameless review from the iceberg — the bad deploy is the event; recurring similar failures are the pattern; the structure is missing validation, delayed feedback (bad rows found weeks later by users), and decision rules under deadline. Fix structure: shorten the feedback delay (row-count/match-rate checks per chunk), not "be more careful".
  - *Supplier integrations / inventory*: availability caches, booking retries and rate-limits are stocks and flows with delays — Beer-Game oscillation (over-ordering, retry storms) is the expected mode when you react to stale information.
  - *Riselist (TikTok agency)*: reinforcing loop — good campaigns -> case studies -> better clients -> better creators -> better campaigns; also a balancing loop — more clients -> less attention per client -> worse results. Identify which loop dominates at each growth stage.
  - *Tereka (money planner)*: personal finance is literally a stock-and-flow model (balance = inflows − outflows; savings stock; debt stock with an interest "price"). The bathtub framing is a natural UI/teaching metaphor.
  - Use the "push it, does it move consistently?" test before putting a metric on a dashboard as a driver.

## Sidney Dekker — The human factor: Pursuing success and averting drift into failure (DDD Europe 2018) (https://www.youtube.com/watch?v=9fwJ9xgvu3A)
**Core idea: counting and suppressing negatives (errors, incidents, rule violations) does not predict or prevent big failures; study how success is actually created — the adaptive capacities and workarounds that bridge work-as-imagined and work-as-done — because that is also where drift into failure lives.**

- **Mental models / first principles**
  - You cannot understand a complex system from a God's-eye position; you are part of it (echoing Dave Snowden). Dekker learned to fly the 737 after becoming a professor to see it from inside. Engineers share this: whoever pushes the deploy button is "first at the scene".
  - **Absence of negatives ≠ presence of safety.** "31 days without a mishap" boards are aspirations, not evidence. They predict nothing and actively encourage hiding bad news -> a *culture of risk secrecy*, which quickly becomes a dumb culture (bosses don't hear bad news, nobody learns). Aim for *risk competence* instead.
  - Evidence: DuPont-style "reduce errors when walking/driving" signs at a plant where a *gas release* killed four. Deepwater Horizon had ~six years without a recordable injury before the blowout killed 11. MIT data cited: the airline with the highest *incident* rate had the lowest *passenger mortality* risk. Small-incident counts and catastrophic-risk are different things; low-frequency incident counts can even correlate with higher catastrophe risk (because reporting is suppressed).
  - **Hospital study (1 in 13 patients harmed).** Investigations of harm found human error, guideline violations, communication failures, miscalculations, procedural violations. Countermeasures: "try harder", war on error, auditors. But when researchers studied the 12 that went *well*, they found the **same** errors and violations. A ward nurse faces ~600 rules ("then we stopped counting"). So the negatives do not discriminate success from failure.
  - What did discriminate: **presence of positive adaptive capacities**:
    1. The ability to **say stop**, even under acute production/deadline pressure and against seniority.
    2. **Not taking past success as a guarantee of future safety** — noticing that today's context differs subtly from yesterday's.
    3. **Diversity of opinion and the ability to dissent** — especially diversity of professional background and psychological make-up, not only demographics.
    4. **Keeping a discussion about risk alive** when everything looks fine — double-checking questions that acknowledge one's own fallibility ("I'm not sure, but have we…?"). Balanced against team harmony and production; being "too safe" is also unsafe (the driver who never enters the roundabout).
  - Erik Hollnagel's numeric argument: far more goes right than wrong, so success is where most of the data is.
  - **Abraham Wald (1943, survivorship bias)**: bombers returning with flak holes — commanders wanted armour where the holes were; Wald said armour where there are *no* holes, because planes hit there didn't come back. Analogy: the small holes you count (incidents) are on the survivors; the fatal paths are where you see nothing.
  - **Work-as-imagined vs work-as-done.** Nick, an outback foreman, wears a hard hat in an open field, hi-vis in bright sun, permit stickers for each "hole" a bureaucracy patched — and a "GSD" (get stuff done) sticker. Every job has a gap between how rule-writers imagine it and how it gets done; something must give. Most sacrifices are harmless (the nurse knows the 2–3 rules out of 600 that truly matter) — but this gap is where both success and future catastrophe come from.

- **Mechanisms / experiments**
  - **Shared space (Drachten, NL, Hans Monderman)**: a square with full signage/lights had ~10 bad accidents per year and gridlock (~20,000 cars/day). Removing all signs and paving uniformly let users self-organize: they read each other's intentions and adapt to the slowest user (an elderly pedestrian sets everyone's speed). No top-down rule produces this. Such systems absorb disturbances ("chaos monkey") better than over-constrained ones.
  - **Woolworths micro-experiment (Australia)**: three store groups — (1) control, (2) all national top-down safety procedures removed, (3) removed *plus* team training in "safety differently" (teams identify their own key risks, e.g. customer slips at small floor grades). Group 3 had the lowest incident rates and best ownership, engagement, locus of control and leadership; group 2 also did well. "Freedom within a frame" beat both full control and plain removal. Took ~18 months with 14 regulators plus ethics approval; built with a kill switch ("pull the plug any moment"). Later replicated across ~1,000 stores.
  - **Drift example (Greek airport)**: a displaced threshold forces landing past a painted bar over hills; no taxiway beyond exit Charlie means a 7–8 minute backtrack if you miss it; turnarounds are squeezed to ~25 minutes ("the bodies or the bags?"). Pilots predictably land short — photo shows a jet low over a road with a coincidentally red traffic light — using informal, socially taught techniques (line a windscreen screw up with a light on the terminal). It is never reported as an incident; it's how work gets done. This is *drift*: locally rational adaptations under production pressure that gradually erode margins, invisible to incident counts.

- **Decisions & trade-offs**
  - Control-by-procedure vs adaptive capacity: more rules reduce small, visible holes but constrain people, push workarounds underground and hide the real risk. Removing constraints needs a frame (shared risk conversation) and a safe-to-fail design.
  - Safe-to-fail micro-experiments (small, randomized, reversible, with clear abort criteria) generate evidence "from the inside out" instead of arguing ideology.

- **Rules of thumb**
  - Stop celebrating zero-incident streaks; track signals of capacity: were "stop" calls made and honoured? Were dissenting views raised? Did anyone ask "is today different?"
  - Investigate successes as rigorously as failures: ask how the work actually got done, what was sacrificed, what was improvised.
  - When you write a rule to plug a hole, ask what it costs the person doing the work and what workaround it will provoke.
  - Make it safe to report; reporting volume going up is often a good sign.

- **Failure modes**: risk secrecy; broken-windows logic (fixing cosmetic negatives to prevent catastrophe); survivorship bias in incident data; replicating a past success in a new context; over-constraint leading to cynicism (parody safety posters) and hidden workarounds; drift into failure through normalized shortcuts under time pressure.

- **Relevance to Sam**
  - *Pipelines and integrations*: "zero failed jobs this week" says little about the risk of silently wrong data (e.g. a mis-joined GeoNames/OSM batch that passes all checks). Look where there are no holes: which outputs have no validation at all? Put "armour" (sample-based correctness checks, reconciliation against source counts) there.
  - *Work-as-done on a 16GB laptop*: shortcuts like skipping a dedupe step to fit RAM, or hand-patching rows, are GSD adaptations — fine, but write them down; they're exactly where future drift starts.
  - *Booking flows*: supplier quirks handled by ad-hoc code paths (retry here, manual override there) are successful adaptations; catalogue them, because they reveal where the official integration model is wrong.
  - *Teams and agencies (Riselist, Starting Out)*: give creators/staff a "freedom within a frame" — few hard rules that truly matter, a shared conversation about risk, and explicit permission to say stop (e.g. refuse a brand brief that would damage a creator's account).
  - *Using AI agents*: a chain of AI-built steps that "always worked" is exactly where Dekker's warning about past success applies — re-check context when inputs shift.

## Richard Cook — Resilience in Complex Adaptive Systems (Velocity NY 2013) (https://www.youtube.com/watch?v=PGLYEDpNu60)
**Core idea: every system runs at the edge of its capacity, economic and workload pressure push its operating point steadily toward the accident boundary, and what keeps it up is not rules or campaigns but operators' continuous resilience work (monitor, react, anticipate, learn).**

- **Mental models / first principles**
  - **Law of systems development** (Larry Hirschhorn, 1997): every system always operates at its capacity — as soon as something improves, we stretch it. Systems are too expensive and important to be allowed slack, so they always run near failure.
  - Lines are merging: "web apps" become business-critical, and business-critical becomes safety-critical (dispatching fire trucks, finding critical info). Assume your system will become safety-critical.
  - Complex: unexpected behaviours, unexpected responses to interventions, new forms of failure, constant change. Adaptive: not just changing but *responding* to pressures — planned change plus unplanned local responses, structural and functional, on time-scales from seconds to decades, all at once.
  - What operators actually do (Cook is an anaesthesiologist): continuously **monitor**, **exploit** opportunities, **estimate distance to failure**, **react** to threats, **anticipate** future conditions, **learn** system behaviour. These aren't separate steps; they're words for one continuous activity. Safety plays out on these short time-scales.
  - Safety-I vs Safety-II: old 20th-century models (reliability, stability, control) explained why accidents happen; new models (resilience, dynamics, anticipation, adaptation, work analysis) explain why they mostly *don't*. The surprise isn't that your system goes down; it's that it's ever up given all its known flaws.

- **Mechanism: Rasmussen's dynamic safety model (late 1980s, post-Three Mile Island)**
  - Three boundaries enclose the feasible operating space: **economic failure** boundary (cross it and you go out of business), **unacceptable workload** boundary (people literally drop from exhaustion), and **acceptable performance / accident** boundary (you define it: MTBF, no outages, nobody dies).
  - The **operating point** is always moving (quasi-Brownian) because it must adjust to situations. An accident = the point crossing the performance boundary.
  - Two gradients: management pressure for **efficiency** pushes away from the economic boundary (accountants always think you're near bankruptcy; the closer you are, the stronger the push); a **least-effort** gradient pushes away from the workload boundary. These vectors add up and the point drifts toward the accident boundary.
  - **Counter-gradients**: new rules (small push back, then drift resumes); **accidents** (huge push back — "healthcare.gov moment: I don't care what it costs, get it up", the point slams toward the economic boundary — but the effect decays over weeks/months as memory fades); **safety campaigns / "uptime matters" meetings** (almost no effect; "people who tell you to work harder have no idea what's going on").
  - **Marginal boundary**: an imagined line inside the accident boundary (speed limits, change rules, agreements). Approaching it is a signal to act. We know a lot about the margin and very little about where the real accident boundary is — we only learn that by crossing it.
  - **Normalization of deviance** (Diane Vaughan, Challenger): ordinary operations take you past the margin; nothing bad happens; repeated many times, people conclude the margin is too conservative and formally move it outward. The accident boundary hasn't moved (technology unchanged), so the gap shrinks and normal random motion now finds the accident. The three-year-old toeing the kitchen line until they're beside the stove. Cook calls this **"flirting with the margin."**
  - Resilience, scientifically, is the study of how the operating point moves and of the relationship between margin and accident boundary — and in the short run it lives entirely in operator communities.

- **Decisions & trade-offs**
  - Post-incident spending spikes are temporary; if you want lasting margin, build it into structure (capacity headroom, automated guardrails) rather than relying on remembered pain.
  - Rules move the point a little and briefly; investing in operator ability to see distance-to-failure moves it more durably.

- **Rules of thumb**
  - Make distance-to-failure visible: capacity headroom, queue depth, error-budget burn, memory headroom — instruments for where the operating point is.
  - Treat "we crossed the margin and nothing happened" as information about luck, not about the accident boundary. Before relaxing a limit, ask what new evidence says the real boundary moved.
  - If you find yourself scheduling a meeting about how important reliability is, you've failed — change structure instead.
  - Expect the efficiency push to resume after every outage; plan for it.

- **Failure modes**: drift under efficiency and least-effort gradients; normalization of deviance; decaying lessons from accidents; reliance on campaigns and exhortation; confusing the marginal boundary with the accident boundary.

- **Relevance to Sam**
  - *16GB laptop pipelines*: the laptop literally sits between a workload/RAM boundary and a performance boundary. Each run that squeezed through at 15.5GB with bigger chunks "proves" larger chunks are fine — that's normalization of deviance; the OOM/kernel-kill boundary didn't move. Keep an explicit margin (e.g. cap at ~70–80% RAM) and log peak memory per run so distance-to-failure is visible.
  - *Booking/supplier systems*: rate-limit headroom, supplier timeouts and payment-confirmation lags are margins. "We exceeded Viator's rate limit last week with no penalty" is not evidence the limit is safe.
  - *Startups and side businesses*: Trameter and Riselist will always push toward the economic boundary (do more with less); Sam personally is near the workload boundary with two jobs, study and several ventures — the same model says drift toward an "accident" (missed deadline, burnout, a client lost) is the default unless deliberately counter-pushed. Use it to decide where to keep slack.
  - *Incident response*: after an outage, use the brief window of "I don't care what it costs" to fix structural margins, since that force fades.

## John Sterman (MIT 15.871) — Introduction to System Dynamics: Overview (https://www.youtube.com/watch?v=AnTwZVviXyY)
**Core idea: policies fail not despite our cleverness but because of it — open-loop mental models ignore feedback, delays and other actors, so "side effects" (really just unmodelled effects) push back against our goals; the cure is to widen mental model boundaries and test them with simulation.**

(Short course-intro lecture; Sterman is author of *Business Dynamics*. Overlaps heavily with the Paine session above, which borrows his slides.)

- **Mental models / first principles**
  - Learning claim: "I can't teach you anything" — capability comes only from doing, practising and trying things out; reading and attending are necessary but not sufficient.
  - The problem isn't just acceleration; it's that more and more policies fail, and thoughtful leaders say it's *because of* our analytic power.
  - **Squeezed manager cartoon**: a manager crushed by pressures acts "decisively", pushes one wall out, gets relief — and the push comes back from elsewhere. Why does it persist? Short time horizons; feedback delays long enough that consequences arrive after the decision-maker has been **promoted** (the evaluators reward short horizons too); and blaming "outside" effects.
  - Saying "it wasn't my fault, it was an unanticipated side effect" actually reveals a narrow mental model. Most bad outcomes are at least partly the result of the person's own earlier decisions via feedback they didn't see.
  - **Open-loop project picture** (identify -> gather data -> evaluate -> select optimal -> implement): nearly everyone has seen it and ~3/4 have drawn it. It encodes a false beginning-middle-end view. Real projects iterate constantly because you discover you misunderstood the problem.
  - **Closed loop**: knowing your goal is insufficient — you need feedback about where you are now (Sterman's bike commute: knowing you must stay on the right doesn't tell you how to turn the handlebars; you need to see where the bike is). Running a firm is harder than riding a bike because the intended-effects loop is only part of the structure.
  - **Side effects don't exist**: there are effects you counted on (in your model) and everything else, which usually feeds back *against* your goals.
  - **Other actors**: customers, suppliers, employees, investors, competitors, communities, nature — each with goals that differ from yours (market share is zero-sum). Every effective move toward your goal pulls the world away from theirs, motivating them to act, with their own limited mental models and their own side effects.
  - The job: expand the boundaries of mental models to include more of this structure. All models are wrong (only reality is reality), but we can do much better than current mental models.

- **Mechanisms / method**: elicit and articulate mental models (in the context of busy people); represent feedback, stocks and flows, delays, non-linearities explicitly; simulate — not because the model gives "the answer", but because it gives insight that improves the mental models of everyone who must be involved for change to happen, leading to **high-leverage policies**. Implementation is the point: SD is useless unless it catalyses change in real organizations.

- **Three core ideas he leaves with**
  1. **Structure generates behaviour**: structure = physics of the system + information available + decision rules that turn information into action. Changing only physics, or only the IT system (information), or only incentives is usually insufficient.
  2. **Mental models matter**: having the right answer is not enough; the people who must act need to hold an adequate model.
  3. **Fundamental attribution error**: my failures have reasons (the customer, the sun in my eyes); yours are because you're incapable. Almost always wrong and low-leverage. Default assumption: everyone is intelligent, capable, cares and wants to learn.

- **Rules of thumb**
  - For any decision, list who else's goals it moves the world away from and what they will do in response.
  - Make evaluation horizons at least as long as feedback delays; otherwise you reward people for creating delayed problems.
  - Expect to loop back in any project; plan iteration rather than treating it as failure.
  - When someone says "side effect", ask which part of their mental model was missing.

- **Relevance to Sam**
  - *Delays and promotion*: in a startup, quick fixes to data pipelines (manual patches, skip-validation flags) look good this sprint and bite after the person has moved on to the next project — make the cost visible in the backlog.
  - *Supplier and partner ecosystem*: pricing or commission changes in Trameter's booking flow move suppliers, affiliates and travellers away from their goals; anticipate their counter-moves (supplier rate changes, affiliate churn).
  - *Learning plan*: Sterman's "I can't teach you anything — you must do it" matches Sam's stated non-tutorial learning approach: build, simulate, practise.
  - *Riselist/Tereka*: competitors and creators are actors with their own goals; a growth push (lower agency fees) invites competitor response — the "squeezed manager" loop.

## Ashley Hodgson — Thinking in Systems, Key Ideas (Ch. 1) (https://www.youtube.com/watch?v=XBFpasY2Gd8)
**Core idea: a system is elements + interconnections + purpose; the interconnections (rules, incentives, information flows) and the *real* purpose (what it actually does) matter far more than the elements.**

(Short economist's summary of Donella Meadows, *Thinking in Systems*, chapter 1.)

- **Mental models / first principles**
  - Meadows' three components: **elements** (what you'd see in a snapshot), **interconnections** (relationships that create dynamics), **purpose/function**. Hodgson's economist addition: *incentives live in the interconnections* — they are what make the system move.
  - Examples: school system (teachers; salary structures/contracts linking teachers and administrators; purpose: education); legal system (laws; review of precedent; purpose: contain crime); digestive system (stomach, intestines; body signals — chemical and nervous — as connectors, chemicals can be both element and connector; purpose: nourishment); health-care system (insurers, doctors, patients, drug firms linked by contracts).
  - **Stated vs actual purpose.** Health care's official "Triple Aim" is population health, cost per capita (Hodgson would rather say sustainability) and patient experience — but the system may achieve none of these and instead serve *its own growth*, benefiting those at the top. "The purpose of a system is what it does" (POSIWID, though she doesn't use the acronym).
  - A system is more than the sum of its parts; systems can be **self-organizing and self-repairing** over some range of disruption (immune response healing a cut; stabilizing forces in markets).
  - Many interconnections operate through **information flows** — increasingly important in an information economy.
  - **Elements are the least leverage**: replace every player on a basketball team and the game barely changes; change the *rules* and the system changes completely. Our minds gravitate to elements because they're easy to visualize.

- **Mechanisms**
  - **Bathtub stock and flow**: stock = water level (snapshot), inflow = faucet, outflow = drain; the stock is the accumulated history of inflows and outflows. You cannot change a stock instantly — reductions must flow out over time, so "change the system now" usually meets inertia.
  - **Balancing loop example — personal budget**: stock = bank balance; inflow = monthly income; outflow = spending. If the balance falls below your goal, the discrepancy makes you cut spending, restoring the stock. Checking your account monthly *is* the loop (its delay = checking interval).
  - **Reinforcing loop example**: Meadows and her brother — each poke provokes a bigger poke back, eroding the stock of goodwill until something breaks the spiral (escalation).

- **Rules of thumb**
  - To understand an organization, map its connectors (contracts, incentives, information flows) before its org chart.
  - Judge purpose by outcomes, not mission statements.
  - Expect stocks to need time to drain; plan change as flow adjustments, not instant switches.

- **Relevance to Sam**
  - *Tereka (money planner)*: Hodgson's budget loop is the core product loop — the app's job is to make the stock and the goal-gap visible and to shorten the delay (weekly or per-transaction rather than monthly) so the balancing loop corrects earlier. Each "cup" is a stock with its own inflow and outflow.
  - *Riselist*: changing creators (elements) matters less than changing payment terms, briefs and performance incentives (connectors). If the agency's real behaviour is "maximize campaigns sold", that is its purpose regardless of the website copy.
  - *Trameter data*: the information flows between suppliers, the master location dataset and the booking UI are the high-leverage connectors; swapping one supplier changes less than changing the matching rules or the data contract.

## Matt P — Systems Thinking 101: Stocks, Flows, and Feedback Loops (Donella Meadows Summary) (https://www.youtube.com/watch?v=zjwBwOmF6HE)
**Core idea: stubborn problems are produced by feedback structure (stocks, flows, balancing and reinforcing loops), so well-meant fixes often backfire via predictable traps such as addiction (shifting the burden) and escalation; work with structure and leverage points rather than forcing outcomes.**

(Short popular summary of Meadows; low novelty but clear framing of two system traps.)

- **Mental models / first principles**
  - The paradox: helping can weaken the thing you help. Linear A-causes-B thinking misses the web.
  - A system is interconnected things that achieve something; the relationships matter most (vs a heap of sand).
  - **Stocks** = measurable level at a moment (bank balance, library books, water in the tub). **Flows** = inflows/outflows (births/deaths; deposits/withdrawals). System "personality" appears when the stock level itself influences its flows — that's a **feedback loop**, the engine of behaviour.
  - **Balancing loops** = goal-seeking stability: thermostat, body temperature (~37°C / 98.6°F), a manager holding inventory at target. **Reinforcing loops** = amplification: compound interest (virtuous), soil erosion -> fewer roots -> more erosion (vicious).

- **System traps described**
  - **Addiction / shifting the burden**: an outside intervention relieves a symptom and the system's own capacity atrophies. Meadows' example: families traditionally cared for elders; social security and nursing homes helped, but over generations families lost the "muscle" for elder care and became dependent on outside help. (note: this is presented as a structural pattern, not a judgement that pensions are bad — the lesson is to design interventions that strengthen rather than replace the system's own capacity.)
  - **Escalation**: two actors in a reinforcing loop, each responding to the other's gain by trying harder. Ad-spend war: A raises ads, B raises more, A responds… both bleed money, market shares unchanged. Not greed or stupidity — the loop wiring makes it almost inevitable.

- **Habits / rules of thumb**
  1. Stop reacting to single events; look for patterns of behaviour over time.
  2. Zoom out to the whole board, not just your piece.
  3. Ignore the stated purpose; behaviour reveals the true purpose.
  4. You can't force or control a complex system — "dance with it": observe its rhythm, find **leverage points** where a small push makes a big change, and guide toward healthier patterns.

- **Relevance to Sam**
  - *Shifting the burden in engineering*: manual fixes by one person (Sam hand-cleaning supplier data, or an AI agent silently patching outputs) relieve the symptom while the pipeline's own validation capacity never gets built. Each manual fix should also produce an automated check, so capacity grows rather than atrophies.
  - *Escalation*: discount/commission wars in travel distribution or creator-rate bidding wars in TikTok marketing — check whether a competitive response is just feeding a loop where nobody's share changes.
  - *Tereka*: compound interest (reinforcing) and debt spirals are the obvious loops to teach users; "addiction" maps to habitual borrowing to cover recurring shortfalls.
