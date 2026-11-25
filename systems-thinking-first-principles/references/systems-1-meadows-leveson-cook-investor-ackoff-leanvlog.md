# systems-1 — cross-cutting principles

1. **Structure and goals drive behaviour, not the people in the system.** If different actors in different places keep producing an outcome no one wants, fix the incentives, information flows and goal; swapping people changes little. Leverage order: paradigm/goal > feedback structure and information flows > elements. (Meadows)
2. **Watch stocks, not just flows.** Revenue, catch or throughput can look best right before collapse; the leading indicator is the underlying stock (fish, supplier goodwill, data-quality debt, team energy). (Meadows)
3. **Every reinforcing loop needs a balancing loop that is strong and fast enough.** A slightly late balancing signal gives oscillation; a much later one lets the system cross a non-linear threshold where regeneration turns into erosion, and it doesn't recover. (Meadows)
4. **Beware of subsidising away your own signals.** Subsidies, price distortion and noisy information weaken the feedback that would correct a system. Clean, fast, true information is a precondition for managing anything. (Meadows)
5. **Success to the successful.** Winners gain the means to win more. Build counter-loops on purpose if you want diversity or fairness. (Meadows)
6. **Safety and security are emergent; treat them as a control problem.** Enforce constraints on system behaviour and on interactions, not just component reliability. Many accidents have no failed part. (Leveson)
7. **Process-model mismatch is the core failure pattern.** Each controller (software or human) acts on its belief about current state; accidents happen when that belief diverges from reality. Ask how each component knows the state and when that knowledge goes stale. (Leveson)
8. **Work top-down from unacceptable losses.** Define the losses, derive constraints, look for unsafe control actions and design them out early (fixing late can cost ~1000x). Distrust single probability numbers for complex software-heavy systems. (Leveson; matches Munger's inversion)
9. **System as imagined vs system as found.** Designs and post-mortems use a static, deterministic picture; real operations are dynamic and stochastic, always under maintenance, and kept running by people. (Cook, Velocity)
10. **Resilience = monitoring + responding + adapting + learning, and it lives in operators.** Design for it: expose the controls, show lift points, support mental simulation, open black boxes, have no "don't touch" zones, and make maintenance continuous. (Cook, Velocity)
11. **Resilience is a costly, continuous balance of destruction and renewal, guided by local signals.** Strength comes from always repairing small damage. Starve it of energy, feedback or slack and it fails; load beyond repair rate and damage accumulates (burnout). (Cook, Bone)
12. **Use the resilience that's already there before trying to engineer it.** Type 1 (align and stabilise so natural repair gives good results) is cheap and proven. Type 2 (tinkering with the control signals themselves) is powerful and creates new hazards. (Cook, Bone)
13. **Latent time bombs and common-mode failure.** Reasonable features plus reasonable setup can produce a fleet-wide failure on a date (expiries, scheduled flips). Big-bang rollouts make it worse. Rarely used capabilities "don't work as advertised" until tested. (Cook, Velocity)
14. **The whole is the product of the parts' interactions.** Improving parts separately does not reliably improve the whole; best-of-breed parts may not even fit. Change a part only when the change improves the whole. (Ackoff)
15. **Aim at what you want, not at removing what you don't.** Eliminating defects doesn't produce the desired state. Do an idealised design ("what would we do now if unconstrained?"), then work back to what's feasible. (Ackoff)
16. **Effectiveness before efficiency.** Doing the wrong thing right is worse than doing the right thing wrong. Discontinuous, creative leaps beat incremental imitation. (Ackoff)
17. **Find and feed the constraint.** Throughput is set by one constraint. Identify it, exploit it (never idle, never fed bad input), subordinate everything else, elevate it only after that, then repeat. Improving non-constraints only builds inventory. (Goldratt via LeanVlog)
18. **Invert, avoid ruin and protect compounding.** List how a plan certainly fails and avoid those paths; anything times zero is zero. Compounding processes (skills, code, audience, capital) are mainly lost through leakages and interruptions. (Munger via Swedish Investor)
19. **Use many models; crush your cherished ones.** A man with a hammer tortures reality to fit his tool. Keep a small latticework of big ideas from many fields, and write down disconfirming evidence right away. (Munger via Swedish Investor)
20. **Models are what-if tools, not predictions.** About 90% of the insight comes from the first 10% of modelling, so sketch the small stock-and-flow model before building the big one. (Meadows)

---

## Donella (Dana) Meadows — Sustainable Systems (https://www.youtube.com/watch?v=hhSpzQhvFS8)
**Behaviour comes from structure and goals, not from the people in the system; a growth loop whose balancing signal arrives late will look healthy right up until it crosses an erosion threshold and collapses.**

- **Mental models / first principles**
  - A system = elements + interrelationships + function/goal. The *goal* is what the system reliably produces, not what anyone says it is for (the national economy's de facto goal is growth: it "throws a fit" when growth stops).
  - Ranking of what drives behaviour (and so where to intervene): (1) goal/purpose, (2) structure — feedback loops, information flows, rules, rewards, prices, who sees what, (3) elements — people, machines. Swapping elements rarely changes behaviour (US presidents converge; a university stays itself as students turn over). Swapping a person only matters if that person changes structure or goal.
  - Layers of observation: events (what the news shows) -> behaviour over time -> structure -> goals -> mindset/paradigm. Mindset is the "progenitor" of systems: goals, rules and incentives are invented from it.
  - Recurring unwanted behaviour across different actors, scales and places = a systems problem, not a people problem. Everyone producing an outcome nobody wants, through rational responses to local incentives, is the signature. Systems thinkers rarely blame people; they ask what about the structure makes the perverse act rational.
  - Holism vs reductionism: "trade-off between environment and economy" is a category error when the two are one system. Analogous software trap: "trade-off between speed and data quality" when bad data is what slows you down.
  - Strict sustainability rules (from Herman Daly): use renewables no faster than they regenerate; non-renewables no faster than a renewable substitute is developed; emit pollution no faster than it can be absorbed; plus social equity (Robèrt / Natural Step).

- **Mechanisms (the fishery model)**
  - Two stocks: fish (inflow regeneration, outflow harvest) and boats (inflow investment, outflow depreciation ~20-year life). Fleet wants ~10%/yr growth, funded by profit = catch x price.
  - Two non-linear relationships drive everything: (a) regeneration vs fish population is hump-shaped — zero at zero fish and at carrying capacity, peak around half capacity; (b) catch-per-boat vs fish density (technology curve).
  - Reinforcing loop: more boats -> more catch -> more profit -> more investment -> more boats. Balancing loop: more boats -> fewer fish -> less catch per boat -> less profit -> investment stops.
  - Run 1 (weak tech): balancing loop bites early, boats level off, sustainable equilibrium. Tragedy of the commons did *not* happen because the feedback arrived before the threshold.
  - Run 2 (better tech, e.g. development-bank loans for radar/bigger boats): balancing loop slightly too slow relative to growth loop -> predator-prey oscillations (overshoot, crash in profit, fleet depreciates, fish recover, repeat). Oscillation = a slightly delayed/weak balancing loop.
  - Run 3 (drift nets, onboard freezers): fish pushed past the peak of the regeneration hump. Below that point the regenerating loop flips into an *erosion loop*: more catch -> less regeneration -> fewer fish. The fleet keeps straggling on with too many boats for the remnant. No recovery.
  - Key observation: if you only watch catch/revenue, the system looks best just before collapse. The leading indicator is the *stock* (fish), not the *flow* (catch). Most major real fisheries and Philippine timber harvests show this overshoot-and-collapse shape.
  - Price feedback ("scarcity raises price, demand drops, system self-corrects") depends on income distribution: with even incomes demand collapses fast; with high inequality a few rich buyers pay anything for the last fish (Atlantic bluefin ~95% fished out, sizes from ~1500 lb to ~300 lb, still worth chasing one fish at ~$120/lb).
  - Government subsidies weaken both balancing loops: consumer subsidies blunt the demand signal; producer subsidies keep unprofitable boats out. Her (possibly dated) figure: ~$70B catch value vs ~$120B cost, gap subsidised. Systemic and politically locked in, so changing the government doesn't fix it.
  - **Success to the successful** archetype (Monopoly): winners get the means to win more; losers lose capacity to recover. Interest flows, education, campaign money, healthcare, poverty<->family size. Counter-loops: equal education, progressive/inheritance taxes, safety nets, periodic "redistribution" (potlatch / game reset).

- **Decisions & trade-offs**
  - Villains of the fishery are the three things culture says will save us — growth, technology, the market — plus government intervention. Not because they're bad tools, but because of the goal/mindset running them.
  - Model runs are "what-if" explorations, not predictions. 90% of insight comes from the first 10% of modelling: build the tiny model first.

- **Rules of thumb (her list for sustainable systems)**
  1. A meaningful goal with a defined sense of *enough*; pure growth on a finite base guarantees overshoot.
  2. Every reinforcing loop (growth, erosion, success-to-successful) needs a balancing loop strong and fast enough to hold it.
  3. Information must be clean, clear, quick, compelling, and true. Distorted prices (not full cost) and entertainment-as-news break management.
  4. Treat the resource base as sacred, culturally, not just by rule — including the system's capacity for resilience, self-organisation and evolution (diversity is the store of it).
  5. Equity: no actors pushed to desperation.
  - Ask of any growth plan: growth of what, for whom, at whose cost, for how long, and how much is enough?

- **Failure modes & gotchas**
  - Watching flows instead of stocks; a delayed balancing signal; crossing a non-linear threshold that flips a regenerating loop into an eroding one; subsidising away the very signal that would stop you; assuming price will save a scarce resource when buyers are unequal.
  - Fishermen's fix at Georges Bank (via Dee Hock): after collapse they rewrote their own first principle — "we are stewards of the whole food chain" — a paradigm change from which new rules, monitoring and information flows followed.

- **Relevance to Sam**
  - Stocks vs flows in a travel business: bookings/revenue are flows; supplier goodwill, API quota, data-quality debt, team energy, creator trust (Riselist) and user trust (Tereka) are stocks. Monitor the stocks; flows look great until they don't.
  - Rate limits and supplier relationships are a "fishery": aggressive scraping or polling (better "technology") can push a partner past their tolerance and get access revoked — the erosion loop.
  - Pipelines: a backlog that grows faster than it drains is an overshoot pattern; oscillating retry storms are a delayed balancing loop.
  - Leverage ordering: before hiring/replacing people or tools (elements), change information flows (who sees which metric, how fast) or the goal (what "done" or "success" means).
  - Success-to-the-successful: early marketplace winners (top creators, top suppliers) accumulate ranking advantage; design counter-loops deliberately if diversity matters.
  - Ask "how much is enough?" when picking what to build across many side projects.

## Nancy Leveson — A Paradigm Change for Safety Engineering (and Security) (https://www.youtube.com/watch?v=cbaYX_Diu1s)
**Safety (and security) are emergent properties of interactions, so treat them as a control problem — enforce constraints on system behaviour — rather than as a component-reliability problem.**

- **Mental models / first principles**
  - An accident ("mishap") is any unplanned, undesired event causing a loss someone cares about: life, property, mission, data/security, or business impact. Intent doesn't matter, so the same method covers security.
  - History: 1800s guards on machines -> 1920s–WWII "everything is human error" -> post-WWII "accidents are chains of component failures" (FMEA 1940s/50s; fault trees, HAZOP, event trees in the 1960s). From ~1980, computers enabled exponential growth in complexity and shifted humans from button-pushing to monitoring and complex cognitive decisions (airline pilots hand-fly ~7 minutes per flight). The old assumption — accidents = component failures — no longer holds.
  - Three ways to cope with complexity: analytic decomposition (works when components are loosely coupled and independent), statistics (works when behaviour is highly random and unorganised), and systems theory (for the large middle: tightly coupled, software-intensive, automated, connected — where almost all modern systems live).
  - Emergence: safety isn't in any component, so you can't sum component reliabilities to get system safety.

- **Mechanisms**
  - Many modern accidents have **no failed component**: everything did what it was specified to do.
    - Navy jet test: pilot told to fire a dummy missile at the plane ahead; "smart" fire-control software, finding an antenna blocking the shot, switched to the next missile — a live one. Requirements were "right"; nobody saw the interaction.
    - Lufthansa A320 at Warsaw: software blocked reverse thrust unless it believed the plane had landed (weight-on-wheels, wheel spin). Hydroplaning on a wet runway -> software believed "airborne" -> refused pilot command -> overran runway. Certified software, no failure.
    - Tupolev at Moscow: soft/late touchdown + crosswind -> same blocking; pilots, expecting reversers, also commanded high engine power to stop faster; software ignored the reverser command but not the power command -> went off the end faster.
  - **Control-loop model (STAMP)**: controller (human or software) issues control actions to a process and gets feedback; it holds a *process model* (its belief about current state). Unsafe behaviour usually arises when the process model diverges from reality (software "thinks" not landed; pilot thinks reversers will deploy). This frames software errors, human errors and flawed requirements in one structure.
  - Controls include physical design (redundancy, interlocks, fail-safe), processes (manufacturing, maintenance, operations) and social controls (management, regulation) — socio-technical, not just technical.
  - **STPA** (hazard analysis built on STAMP): start top-down from the losses/hazards you won't accept, identify the constraints that prevent them, then find control actions that could be unsafe (provided, not provided, wrong timing, wrong duration) and the flawed process-model/feedback scenarios behind them. **CAST** is the accident-analysis counterpart. Applied to a DoD programme's full chain (programme office -> buyer -> vendor -> OS -> software), it found vulnerabilities later exploited by SolarWinds; that programme wasn't compromised.
  - Probabilistic risk assessment critique: fault trees multiply component failure probabilities to hit 1e-9 for certification, but need independence (hard to show), can't handle non-failure accidents, and assign software an arbitrary 1e-4 from a misused paper. A 2002 evaluation found expert teams disagreeing by 3–4 orders of magnitude on the same system. A drone rated at ~1e-12 (roughly 500,000 years between accidents) crashed five times in its first year. Every accident she has investigated had a PRA saying it shouldn't happen. "If you know enough about a design error to put a number on it, fix it instead."

- **Decisions & trade-offs**
  - Constraint enforcement includes the old reliability toolbox (prevent component failures when needed) but adds controlling interactions — a paradigm change subsumes the old one.
  - Do it at concept/requirements/architecture time: fixing in test can cost ~1000x more than doing it right up front; late fixes in 100M-line systems introduce new problems. Retrofitting safety by removing devices until it's "reliable enough" (the torpedo that shut itself off and sank) is the wrong order.
  - Human vs automation override: someone must have final authority. Humans err but adapt and handle the unforeseen; software is "intelligent like a five-year-old". She's sceptical of near-term road autonomy; constrained environments (factory AGVs) are fine.
  - Industry evidence: many shadow-project comparisons; GM sceptics tried it against their method and became the largest users. Reportedly cheaper and more powerful than traditional analyses.

- **Rules of thumb / checklist**
  - Start from the loss you refuse to accept, work backwards to worst-case scenarios, and design them out — rather than enumerating every way each part can fail forward (FMEA-style).
  - For each controller: what does it believe about the process, where does that belief come from (feedback), and when can it be wrong?
  - Don't analyse hardware, software and humans in separate silos; mode confusion and lost situational awareness live at the interfaces.
  - Look above engineering: management decisions, procurement and supply chain are controllers too (737 MAX was, in her view, a management-decision accident).
  - Distrust single risk numbers for complex, software-heavy systems.

- **Failure modes & gotchas**
  - Large, distributed teams (F-35: ~8,000 engineers, ~4,000 software, spread across every state for political funding) make unseen interactions likely.
  - Post-incident reflex to "fix the component that let the target escape" instead of the design that allowed it.
  - Blaming the operator ("don't let the stupid pilot...") produces interlocks that cause the next accident.

- **Relevance to Sam**
  - Booking flows are control loops: the booking service holds a process model of supplier state (availability, price, booking status). Most booking incidents — double bookings, charging for an unconfirmed booking, showing stale prices — are process-model divergence, not crashes. Ask "how does this component know the state, and when could that be stale?"
  - Use STPA-style thinking for a new supplier integration (Viator, RateHawk): list losses (customer charged with no booking; booking with no charge; data leak; rate-limit ban), derive constraints ("never capture payment without supplier confirmation ID"), then examine unsafe control actions (retry sends a second booking; timeout treated as failure while supplier actually booked).
  - Pipelines: a job that "succeeded" but wrote wrong data is a non-failure accident; checks should assert constraints on outputs (row counts, key uniqueness, geographic bounds) rather than only catching exceptions.
  - Automation vs human override in AI features (e.g. Drive Button injecting bookable links): decide explicitly who has final say and design feedback so the human's model stays accurate.

## Richard Cook — A Few Observations on the Marvelous Resilience of Bone & Resilience Engineering (https://www.youtube.com/watch?v=AOH2aG9AEFw)
**Resilience is not a static property but a continuous, energy-consuming balance of destruction and renewal, steered by local signals; most "resilience engineering" is arranging conditions so an existing resilience produces good outcomes, not creating resilience itself.**

- **Mental models / first principles**
  - Bone is the *archetype* of resilience, not a metaphor for it. The skeleton looks static but is replaced roughly every 10 years: osteoclasts dissolve old bone (with acid), osteoblasts lay new bone. Durability comes from constantly repairing micro-fractures, not from being impervious to damage.
  - Remodelling is **directed by mechanical strain**: sensors in bone detect strain and new bone is laid along strain lines. The internal lattice of the femoral head isn't coded in any gene — it emerges from the repeated loads of walking upright. Structure records history: you can read what happened to a person from their bones, as you can read an organisation's history in its software architecture.
  - Bone does more than one job: it is also the body's store for calcium (~1 kg, ~99% in bone) and phosphorus. Calcium is the most tightly regulated ion; the calcium-control network (kidney, gut, vitamin D, parathyroid hormone) borrows from or deposits into bone. Resilient subsystems are coupled to other goals.
  - **No master controller**: a messy, layered signalling network with lots of crosstalk and cell-level integration. "Locally directed, globally modulated."
  - Links to David Woods' four concepts of resilience; the two that matter most are **graceful extensibility** (soft, stretchable performance boundary rather than a cliff) and **sustained adaptability** (never-ending adaptation).
  - Resilience is expensive (continuous energy and materials), delicate (lost if feedback is lost), prone to disease, and finite (it has a sell-by date).

- **Mechanisms**
  - Fracture repair: periosteum injury -> haematoma -> cartilage cells bridge the gap -> bone-forming cells lay a thick callus -> over months to years strain-guided remodelling sculpts the callus back to near-original shape.
  - Repair heals whatever alignment it's given: a ~1400 AD skeleton shows a femur healed overlapped (muscles pulled the ends past each other), leaving a leg ~1.5 inches shorter.
  - Medicine's answer (known since the Edwin Smith papyrus, ~1600 BC, and Egyptian splints): **reduction** (align the pieces) + **fixation** (hold them still long enough) — casts, external fixators, plates and screws, intramedullary nails. Surgeons don't heal bone; they arrange the conditions so the body's healing yields function. Nutrition is a known input.
  - Too much strain outpaces repair: fractures accumulate faster than they heal (his hint at burnout).
  - Pathologies: osteogenesis imperfecta, Paget's (excess resorption), osteoporosis (resorption > formation), osteosarcoma, rickets (malnutrition), hyperparathyroidism (a signalling fault: too much signal -> bone chewed up, fractures, kidney stones).
  - Modern drugs (abaloparatide, teriparatide) mimic the PTHrP signal in microgram doses given intermittently, tipping the balance to bone formation; roughly 91% vertebral-fracture risk reduction vs placebo. But at high continuous doses in rats they caused cancers — manipulating the control signals creates new hazards.

- **Decisions & trade-offs: two kinds of resilience engineering**
  - **Type 1** — engineering applied *to* a resilient system: rely on resilience being present; you can succeed without understanding its mechanism in depth; improved by knowing its inputs and modulators. ~3,500 years old. Cheap, safe, proven.
  - **Type 2** — engineering *the resilience itself* (modulating the signals): needs very deep mechanistic understanding, small inputs have huge effects, and it generates new kinds of hazard. Only a few years old in bone.
  - People excited about resilience tend to jump to Type 2 ("build/boost resilience") and undervalue Type 1, which is what operations teams already do daily. Incident tooling (paging, runbook automation) is Type 1: it channels the resilience held by the people who run systems so it flows to where it's needed.

- **Rules of thumb**
  - Find the resilience already present (usually in people's adaptive capacity) and create the "reduction and fixation" so it plays out productively: right information to the right people, stable conditions, time to heal.
  - Feed it: resilience needs ongoing energy and resources (slack, training, time).
  - Protect feedback loops; resilience dies when the signal is lost.
  - Treat background churn — staff turnover, onboarding, re-architecture, on-call rotation — as the remodelling that keeps capacity alive, not just as cost.
  - Be wary of directly tweaking powerful control signals (incentives, auto-scalers, global retry policies): small changes, big and sometimes harmful effects.

- **Failure modes & gotchas**
  - Load beyond repair rate -> accumulating damage (burnout, tech-debt piling faster than it's paid).
  - Healing without alignment locks in a bad shape (a rushed hotfix that becomes permanent architecture).
  - Organisational "diseases" that rob resilience: starving maintenance, cutting slack, breaking feedback.

- **Relevance to Sam**
  - A data pipeline over ~6M GeoNames rows stays healthy through continuous small repair (regular validation, re-matching, dedupe passes) — not a one-off big clean. Budget energy for ongoing remodelling.
  - Build where strain actually is: invest in the parts of the booking flow or dataset that real usage stresses, the way bone lays material only along strain lines; don't over-engineer unstressed paths.
  - In a small team and side businesses, Sam himself is the resilience; Type 1 work is making his adaptive capacity effective (runbooks, alerts with context, checklists) and protecting it from overload rather than trying to automate judgment away.

## Richard Cook — How Complex Systems Fail (Velocity 2012) (https://www.youtube.com/watch?v=2S0k12uZR14)
**The surprise is not that complex systems fail but how rarely they do; success comes from people monitoring, responding, adapting and learning in the "system as found", so design to give operators the controls and knowledge they need.**

- **Mental models / first principles**
  - Flip the question: given how systems are built and run, we "deserve" far more outages than we get. Why so few? Nobody calls ops at 3 a.m. to report things are fine, so success goes unstudied.
  - Research base (~25 years): ORs, ERs, ICUs, airline and military cockpits, military software, semiconductor fabs, power generation/distribution, Israeli mass-casualty response to bus bombings.
  - Properties of real operational worlds: genuine surprises (existential, not latency blips); highly variable demand/op-tempo; "stuff don't work as advertised" (features unused for years fail when first needed); novel conditions are common; constant tailoring (tweaking settings, toggling services); unanticipated uses (email used to move huge files); continuous change of tech and people; both shallow conflicts (squeeze a bit more out) and deep ones (take it down to fix, or fix live?). Summary: the normal world is not well behaved, yet we succeed a lot.
  - **System as imagined vs system as found**:
    - Imagined: state diagrams, rack layouts, fault trees, PRA — static and deterministic. Used in design, development, and (crucially) in post-incident reviews.
    - Found: dynamic and stochastic, predictable only statistically; with people at screens holding it together ("do not touch any of these wires"). Encountered in implementation, operations, maintenance, recovery. Mean time between maintenance is effectively zero — something is always being changed. Hollnagel's FRAM is one way to draw it.
  - **Resilience = monitoring + responding (anticipating, not just reacting) + adapting + learning**, and it lives mostly in operator communities (often shift workers, hourly, anonymous in the org chart) that designers rarely meet.
  - **Reliability vs resilience**: reliability is built at design time from stiff boundaries, layers, formalisms, defence in depth, redundancy, abstraction and information hiding, assurance and accountability. Resilience is what you actually want at run time: absorb transients, recover smoothly and quickly, prioritise high-level goals when something must be sacrificed, recognise and handle situations never imagined at design, and adapt systems designed 4–7 years ago to the next 3–5.

- **Mechanism example (infusion pumps)**
  - Hospital standardised on one vendor, replaced every pump on one day. A config field for "mandatory software upgrade date" was set one year out during commissioning. At that date (00:20, a year after install) pumps stopped accepting new commands until updated. Within ~25 minutes ~20% failed; one biomed tech swapped and fixed them; every pump needed touching within 24 hours. Without that person, 600–700 pumps would progressively have become unusable over 7–8 hours. Afterwards little attention was paid: the tweak was reset and life moved on.
  - Lessons: latent time bombs from reasonable-seeming features + reasonable setup; common-mode failure from a single-vendor, single-day rollout; recovery depended entirely on a human on site.

- **Design for resilience (his list)**
  1. Support continuous maintenance as part of the design, not a periodic bolt-on.
  2. Reveal the controls to operators — at 3 a.m. they're the only ones who can help; trust them rather than locking them out.
  3. Show the "lift points": mark where a component can be picked up, moved out and replaced (in code, infrastructure and org structure).
  4. Support mental simulation: operators need deep knowledge of configuration and current status to predict what an action will do.
  5. Open the objects and methods: heavy abstraction/black boxes block reasoning during incidents (cf. fiddling with Wi-Fi driver settings).
  6. Abolish "don't touch" zones — no part of the system should be inoperable.
  7. Empower operator learning.
  - Agenda: hand ops the keys (and build that confidence if absent); make resilience the first design priority for next-generation systems; fund discovering and supporting resilience across the lifecycle.

- **Decisions & trade-offs**
  - Reliability measures (hiding, locking down) can reduce resilience by removing operator access and understanding. Trade some protection-from-people for capacity-of-people.
  - Post-incident reviews done in the "as imagined" frame will misread what happened; study normal work, not just failures.

- **Failure modes & gotchas**
  - Expiry dates, certificates, licence keys, scheduled config flips — time-triggered latent faults that fire simultaneously across a fleet.
  - Big-bang rollouts create common-mode failure.
  - Assuming rarely-used features (failover, backups, DR runbooks) work as advertised.

- **Relevance to Sam**
  - API keys, OAuth tokens, SSL certs and supplier contract/sandbox expiries in Viator/RateHawk integrations are infusion-pump time bombs: keep a register with alerts well before expiry.
  - Stage rollouts of pipeline or schema changes; avoid touching every supplier connector on one day.
  - For a one-person or small-team setup, Sam is the "operator community": build scripts and dashboards that expose real state (row counts, last successful sync, queue depth) and controls (re-run a stage, replay a date range, pause a supplier) instead of opaque cron jobs.
  - Test restore and replay paths before they're needed — "stuff don't work as advertised."
  - Design the GeoNames/locations pipeline with lift points: each stage writes inspectable Parquet/CSV so a broken stage can be swapped out or re-run alone.

## The Swedish Investor (on Charlie Munger) — Mental Models for the Rest of Your Life (https://www.youtube.com/watch?v=ywyQ_eNNCJU)
**Carry a latticework of a few big models from many fields, avoid stupidity before seeking brilliance (invert), protect compounding from interruptions, and actively destroy your own outdated beliefs.**

(Short, secondary-source explainer; lighter on technical substance.)

- **Mental models / first principles**
  - "The company that needs a new machine tool and hasn't bought it is already paying for it" — the same holds for thinking tools: the cost of a missing model is paid invisibly in bad decisions.
  - **Swiss Army knife / latticework**: Munger reckons he uses ~100 models regularly, but "a few big ideas carry most of the freight." Single-model thinking = man-with-a-hammer syndrome: reality gets tortured to fit the tool. Organisational version: departments optimising their own discipline (marketing over product quality). Example given: efficient-market theory built on a rational-agent model while ignoring psychology.
  - **Learn from the eminent dead**: ~117 billion humans have lived; the best thinking is in their records. Choose sources carefully (giants, not hype channels) and learn mentors' backgrounds to understand their biases. Munger's model was Benjamin Franklin.
  - **Invert, always invert**: hard to specify how to get X; easier to list how to get not-X and avoid those. "Tell me where I'll die so I never go there." Harvard 1986 talk: guaranteed misery = mood-altering chemicals, envy, resentment; guaranteed failure = unreliability, learning only from your own experience, quitting after a first few tries, not inverting.
  - Asymmetry: downside often outweighs upside — anything times zero is zero. Being "consistently not stupid" beats trying to be very intelligent.
  - **Compounding**: growth proportional to the stock. Snowball phases — (1) early: returns are tiny (30% of $1,000 is $300), so outside inflow (earned income) dominates; (2) mid (~$100k): returns become a meaningful share of income; (3) late (~$1M in his example): returns exceed earned income. Inverted: protect it from leakages — management fees, frequent-trading costs, avoidable taxes.
  - **Crush cherished beliefs**: Galbraith — people prefer proving they needn't change their mind. Darwin wrote down disconfirming evidence within ~30 minutes, before his mind discarded it. Unlearning is the hard part.

- **Decisions & trade-offs**
  - Breadth of models vs depth: aim for the big, obvious ideas across many fields rather than exotic depth in one.
  - Avoiding ruin first, optimising second.

- **Rules of thumb**
  - Before a plan, run a pre-mortem: "How would this certainly fail?" then design those paths out.
  - Track leakages that silently kill a compounding process.
  - Log contradicting evidence immediately.

- **Gotchas**
  - The video's numbers are illustrative, not rules (note: 30%/yr returns are exceptional, not a planning assumption).

- **Relevance to Sam**
  - Systems thinking itself is a "machine tool": stocks/flows, constraints and control loops are a few big models that carry most of the freight across pipelines, booking flows and businesses.
  - Inversion = Leveson's top-down hazard approach: list the losses Trameter/Tereka/Riselist must never suffer (double charge, data leak, banned by supplier, creator unpaid, user's money plan wrong) and guard those first.
  - Compounding applies to skills, reusable code/libraries, datasets and audience (YouTube, Substack): early phase needs outside input and patience; protect against leakages like context switching across too many side projects.
  - Man-with-a-hammer risk: reaching for an LLM or a new framework for every problem when a join, a constraint or a cron job would do.

## Russell Ackoff — "If Russ Ackoff had given a TED Talk..." / Beyond Continual Improvement (https://www.youtube.com/watch?v=OqEeIG8aPPk)
**A system's performance is the product of how its parts interact, not the sum of how they act separately — so improving parts separately cannot be relied on to improve the whole; aim at what you want (not at removing what you don't), and at doing the right thing, not just doing things right.**

- **Mental models / first principles**
  - Context: national surveys found over two-thirds of managers who launched quality/improvement programmes judged them failures. Since quality means meeting the customer's expectations, and these managers were the customer, the programmes failed — whatever the experts say. His hypothesis: they failed because they were anti-systemic.
  - **Definition of a system**: a whole made of parts, each of which can affect the whole's behaviour, and none of which has an independent effect — each part's effect depends on what other parts are doing. So a system can't be split into independent parts.
  - **Essential properties belong to the whole, not to any part**: a car moves you; no wheel, axle or engine can (the engine can't even move itself). You live, see, write, think; your hand, eye and brain cut off do none of it. Disassemble a system and it loses its essential properties even with every part in the room.
  - **The best-parts car**: test all 457 car models on sale in the US, pick the best engine (Rolls-Royce), transmission (Mercedes), battery (Buick)... and assemble them. You don't get the best car; you don't get a car at all, because the parts don't fit. Performance depends on fit and interaction.
  - **Architect's principle**: design the house first, then rooms to fit it. Change a room to improve the house; never improve a room in a way that doesn't simultaneously improve the house.
  - **Removing what you don't want ≠ getting what you want** (critique of Shewhart-style defect elimination). TV example: probability a random channel shows what you want ≈ 0.01; switching away from a bad one leaves you at ≈ 0.01. Improvement must be aimed at the desired state.
  - **Idealised design**: ask what you'd do *right now* if you could do anything you wanted; if you don't know that, you can't know what to do under constraints.
  - **Discontinuous > continuous improvement**: creativity breaks the chain. You don't overtake a leader by imitating them slightly better; you leapfrog.
  - **Efficiency vs effectiveness (Drucker)**: doing things right vs doing the right thing. Doing the wrong thing right is worse than doing the right thing wrong. His example: high-quality cars still choking cities such as Mexico City and Santiago — quality must include value, not just efficiency. Efficiency ~ knowledge; effectiveness ~ wisdom.

- **Rules of thumb**
  - Before optimising a component, state the whole's purpose and check the change improves the whole.
  - Specify the target state positively ("what good looks like") instead of a list of defects to eliminate.
  - Periodically do an idealised redesign from scratch, then work backwards to what's feasible.

- **Failure modes & gotchas**
  - Local optimisation by teams/departments with separate KPIs; "best of breed" component choices that don't integrate.
  - Polishing a process that shouldn't exist.

- **Relevance to Sam**
  - Picking the "best" tool per layer (best geocoder, best DB, best LLM, best supplier API) doesn't give the best travel product; integration fit (data formats, IDs, latency, failure semantics) decides it.
  - In matching/cleaning, a defect-elimination mindset (remove bad rows) doesn't guarantee a useful master location dataset; define what a correct, complete record looks like and design toward it.
  - Ask "right thing?" before "done right?" across side projects: perfecting a pipeline matters less than whether that dataset drives bookings.

## LeanVlog — The Theory of Constraints: A Complete Introduction (https://www.youtube.com/watch?v=LiAHtgFyRkA)
**Manage a business by three money measures and focus improvement only on the constraint that governs flow, using Goldratt's five focusing steps.**

(Very short summary video; ends with a course plug, skipped.)

- **Mental models**
  - From Eli Goldratt's *The Goal*: replace dozens of KPIs and scattered initiatives with three measures and a few targeted actions at the constraint.
  - Three measures (all in money): **output** (Goldratt's "throughput" — measured as money generated, not units produced), **inventory** (raw material cost only, excluding value added in process), **operating expenses** (all of them, no direct/indirect split).
  - Goal of the firm: raise output, lower inventory and operating expenses. Judge every investment/proposal by its effect on these three.
  - **Priority order**: output first (it lifts cash flow, profit and ROI at once), then inventory, then operating expense — cut the latter two carefully, since too little inventory or laying off skilled people can reduce output.
  - Most firms have only a few constraints; the constraint sets the pace of the whole flow (see drum-buffer-rope).

- **Mechanism: five focusing steps**
  1. **Identify** the constraint.
  2. **Exploit** it: don't lose a minute there — buffer in front of it, quality checks before it (don't waste constraint time on bad input), better maintenance (TPM).
  3. **Subordinate** everything else to it: everyone knows where it is; measure its performance; feed upstream work to it promptly; smaller batches to speed flow.
  4. **Elevate** it: only after 2–3, spend decisively (duplicate the machine, overtime); if it's the true constraint, this pays back amply.
  5. **Repeat** from step 1 once it's broken — and don't let inertia (old rules built around the old constraint) become the new constraint.

- **Gotchas**
  - Improving a non-constraint adds no output; it can just build inventory.
  - Cost-cutting that starves the constraint reduces throughput.
  - (note: the video doesn't explain constraint vs bottleneck or drum-buffer-rope in depth.)

- **Relevance to Sam**
  - On a 16GB laptop, find the true constraint of a pipeline (RAM, disk I/O, a single-threaded join, a rate-limited supplier API) before parallelising anything. Exploit: never let the rate-limited API sit idle (keep a queue ready, validate requests before sending so quota isn't wasted on bad calls). Subordinate: size chunks to what the constraint can absorb. Elevate: only then buy more quota/compute.
  - Business level: for Riselist or Tereka, the constraint may be sales/outreach or Sam's own hours, not code. Work that doesn't move the constraint is inventory (half-built features).
  - Measure money-based throughput (bookings revenue, paying clients), not units of activity.

