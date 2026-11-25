---
name: startup-product-first-principles
description: Startup and product judgment from 19 talks (Paul Graham, Seibel, Snyder, Todd Jackson, Mom Test, Dunford, Alströmer, Epstein, Blomfield, Altman). Use for any idea, MVP, user interview, PMF, first customers, positioning, pricing or focus call, e.g. Tereka, Riselist, Trameter.
---

# Building products people want, from first principles

This skill turns 19 talks and lectures into working judgment for finding, testing, launching,
selling and pricing products with very little capital. This file holds the procedures, the
principles the teachers keep returning to, checklists, and a reviewer's box of things the talks
don't cover (Nigeria, payments, regulation, unit economics). `references/` (when present) holds
detailed notes for every video, each credited to its teacher.

Use it for any product or business call, even unnamed: idea quality, v1 scope, first users, price,
"they love it but don't come back", "build X?", which project to focus on, a pitch or cold email.

**How to use it**
1. Pick the procedure that matches the question (sections 1–4) and walk it step by step.
2. Run the relevant checklist (section 6) against the plan, interview notes, metrics or copy.
3. If a `references/` folder sits next to this file, use it for depth. `references/INDEX.md` lists
   every video with its line number, or run `grep -n -i "keyword" references/*.md`. Each section
   ends with a "Relevance to Sam" or "For Sam" note: Sam is a solo builder in Lagos with two jobs
   and almost no capital, running Tereka (a free money-organising planner for Nigerians), Riselist
   (a service-first TikTok creator-marketing agency) and Trameter product work (Drive Button,
   booking flows, "where can I go"). Use those parts only in that context. Lines marked "(note: …)"
   and those Sam notes are helper additions, not the speaker's words. If `references/` is missing,
   the full notes live in the user's private GitHub repo `Biodun-ml/engineering-skills` (folder
   `startup-product-first-principles/references/`); clone it when you need depth.
4. Explain the *why*, and credit the teacher when you use a specific pattern.
5. Benchmarks in the talks (conversion rates, ARR bands, price thresholds) are mostly US, B2B-SaaS
   and in dollars. Treat them as shape, not as targets for Nigerian consumer products. Dollar
   figures from 2014 talks are dated.

---

## 1. The core loop: learn → confirm → grow

Do these stages in order. Doing them out of order (selling before learning, growing before a small
group loves it) is the number one killer. *(Fitzpatrick; Altman)*

1. **Pick the person and problem first, spend real time on it.** Founders spend 99% of their time
   building and 1% picking, yet the pick bounds the next decade. *(Todd Jackson citing Kopelman)*
   Name a *specific person* with a blocked project, not a segment. *(Snyder)*
2. **Learn.** Talk about their life, past facts and current workarounds, never your idea.
   *(Fitzpatrick; Alströmer)* Watch them work where you can. *(Caldwell & Seibel)*
3. **Confirm with behaviour or money.** Ask for payment, or ship something tiny and watch if they
   come back. Behaviour doesn't lie; words do. *(Fitzpatrick; Epstein)*
4. **Iterate cheaply and fast.** Expect around 27 loops, about 10x more than you plan for, so keep
   burn low enough to afford them. *(Rusenko)* Kill bad ideas fast. *(Reinhardt)*
5. **Grow only after a few people love it.** Love expands; mass liking doesn't. *(Altman)*
6. **Repeat one case study.** "They were trying to do X, options failed because Y, we unblocked them
   with Z", run through a factory of pipeline → sales → delivery. Work only on the current
   bottleneck. *(Snyder)*

Hold the user and problem tight and the solution loose. When it fails, fix the screwdriver; don't
hunt for new uses for it. When you do change course, say which variable you moved: product *or*
market, never both silently. *(Seibel "MVP"; Reinhardt)*

---

## 2. Procedure: validate an idea in two weeks

Assembled from Snyder, Fitzpatrick, Alströmer, Todd Jackson and Friedman. Fits evenings and lunch
breaks: customer learning runs on calendar time, not clock time, so start while employed.
*(Fitzpatrick)*

**Days 1–2: write the hypothesis.**
- Score the idea on Friedman's ten questions (checklist 6.1). Google it and find why predecessors
  failed (tar-pit check). *(Friedman)*
- Write a PULL card for one named kind of person *(Snyder)*: **P**roject on their to-do list (a job,
  not "buy software") · **U**navoidable now (deadline, regulator, boss, consequence) · **L**ist of
  options they've considered (including doing nothing, a spreadsheet, a person) · **L**imitations
  that leave them stuck. All four must hold. P without U: they complain but don't buy. Good-enough
  options: they do it themselves.
- Use AI for baseline domain knowledge so real conversations go deep *(Caldwell & Seibel)*, and map
  the buyer's day: where they spend time, email or not, groups, events *(Max)*.

**Days 3–10: about 10–15 conversations.**
- Sources: your network first (most responsive, least honest), then co-workers, then communities
  where they self-identify (LinkedIn titles, Reddit, Facebook/WhatsApp groups, forums, events).
  *(Alströmer; Max)*
- Ask for 20 minutes with a short, specific hook, never a pain-point pitch *(Alströmer; Snyder)*;
  run checklist 6.2 and log exact quotes and emotions *(Fitzpatrick)*.
- Review every ~5 conversations: zero pull and nothing learned means change the hypothesis (persona,
  problem or promise). *(Snyder; Todd Jackson)* Six consistent answers confirm a guess; "three
  clusters of two" means mixed segments, so narrow and do six more. *(Fitzpatrick)*

**Days 8–12: test a promise, not a product.**
- Snyder's two slides: slide 1 states their pull in plain words ("You're trying to do X now because
  of Y; A and B fail because Z. Did I get that right?"); slide 2 is a rough sketch of how you'd
  help. You want "oh my god, yes", then "how do we start?".
- Or a mock-up / landing page / Wizard-of-Oz manual delivery. Lattice sold its first 5–10 customers
  on Figma mock-ups. *(Todd Jackson; Rusenko)*
- When a call goes lukewarm, ask the Hail Mary: "Is there anything on your to-do list you'd pay me
  to take off your plate right now?" *(Snyder)*

**Days 12–14: ask for commitment and decide.**
- Ask for money, a deposit, a signed intent, a calendar slot, or an intro to a colleague. Test
  willingness to *transact* before fine-tuning price. *(Snyder; Alströmer)*
- **Go** if several people show pull-shaped actions ("how do we get started?", "wait, you can do
  *what*?", asking for the deck to share internally). **Kill or change one P** if all you got was
  "interesting, send more info", "maybe next quarter", "no budget". *(Reinhardt; Todd Jackson)*
- Stop *validating* once you can predict 70–80% of answers *(Todd Jackson)*, but keep *exploring*
  for surprises *(Caldwell & Seibel)*.

---

## 3. Procedure: plan and launch an MVP

1. **Name the first users today.** Yourself, people you know, or a list you can reach. If you can't
   name any, question the idea. *(Seibel "MVP" and "Mistakes")*
2. **Pick their single highest-order problem** and ignore the rest. Vision for everyone, MVP for a
   few. One feature that works beats many that don't. *(Seibel; Caldwell & Seibel)*
3. **Time-box the spec** (e.g. 3 weeks), **write it down** (unwritten specs drift silently from 3
   weeks to 3 months), and **cut it** when you're over scope a week in. *(Seibel "MVP")*
4. **Use less surface area than feels minimal.** Airbnb had no payments or map; Stripe had no bank
   deals; Brex v1 was a virtual card; OpenSea used third-party wallets. Use third-party parts for
   anything non-core. *(Seibel; Caldwell & Seibel; Alströmer; Altman)*
5. **Aim for "minimum remarkable"**: better than what's out there at the one job that matters most,
   not on a feature grid. *(Rusenko citing Buchheit)*
6. **Instrument it from day one** (actives, returning usage, key action) *(Seibel; Altman)*, and run
   3–5 silent UX tests: give a task, don't touch, say nothing; three find the critical bugs *(Rusenko)*.
7. **Launch to real users within weeks** (under a month for most consumer and B2B). Separate the
   "any customers" launch (now) from a press launch (later). Nobody remembers launch day. *(Seibel)*
   Heavy MVPs (banking, lending, insurance, hard tech) still start with a website explaining what
   you do. *(Seibel)*
8. **Deliver to the first users by hand.** Stripe integrated in customers' offices. Onboarding is a
   funnel stage; skipping it causes churn. *(Seibel; Alströmer)*
9. **Find the "X" moment** after which churning would be weird, and make onboarding a bullet train
    to it. *(Snyder)*
10. **Run the feedback engine weekly:** feedback → decision → back in front of users. Aim for 10%
    better per week. *(Altman)*

---

## 4. Procedure: price it

1. **Charge, early.** Not charging is the most common mistake. Charging tells you whether anyone
   pays, which segment pays most, how much, and which channels you can afford. *(Epstein;
   Alströmer)*
2. **Find the order of magnitude, not the perfect number.** $10 vs a willingness of $100 must be
   fixed; $10 vs $15–20 is fine for now. *(Epstein)*
3. **Anchor on the alternatives and the value, never on cost.** Bounds come from what the status quo
   costs them and what finishing the project is worth (six staff doing it by hand meant a ~$1M/yr
   problem, not a $200/month one). *(Snyder; Fitzpatrick; Epstein)*
4. **B2B value equation.** Write the value down with the champion (time saved, cost cut, revenue
   gained) and price at 25–50% of it: they keep about two-thirds. The same equation becomes the
   pilot's success criteria. *(Blomfield)*
5. **Cost is a floor.** Target 80–90% gross margin; count cloud and AI credits as cash. If your
   one-third share is below cost, show more value, change what you build, or exit. *(Blomfield)*
6. **Ask high and let them negotiate.** Segment asked $120K/yr and settled at $18K, ~150x its
   sheepish $10/month; customers had worried a cheap tool couldn't be trusted with their data. Low
   price can signal low trust. *(Reinhardt; Epstein)*
7. **Raise until they complain but still pay.** Instant enthusiastic yes means too cheap. If unsure,
   raise ~50% per new pitch until you lose more than 25% of deals on price alone *(Blomfield)*, or
   keep doubling until people stop paying, then undo the last doubling *(Fitzpatrick)*.
8. **Make price, channel and model fit.** A salesperson must close ~5x their pay in new ARR. Around
   $75/month is a dead zone (too cheap for sales, too dear for impulse); sales-led needs roughly
   $200–300/month or more, self-serve roughly $25/month or less. Slow sales cycle plus low ticket is
   the "quadrant of death". *(Blomfield; Rusenko; Todd Jackson)* (note: US numbers; translate by
   local purchasing power.)
9. **Pilots: short, paid, decisive.** No long free trials or unpaid pilots. Use 2–4-week paid proofs
   with value-equation criteria, or an annual contract with a 30–60-day money-back guarantee, priced
   under the champion's signing authority. *(Blomfield; Alströmer)*
10. **Mirror how they already pay** (flat, per seat, per campaign) and cap any usage pricing. Prefer
    committed recurring revenue. Keep plans simple. *(Blomfield; Epstein)* Consumer trials work best
    with a card up front. *(Alströmer)*
11. **Pricing isn't permanent.** Grandfather early users; raise for new ones. *(Epstein;
    Fitzpatrick)* Don't compete on "X but cheaper"; differentiate instead. *(Epstein; Blomfield)*

---

## 5. Principles that recur across the teachers

### Ideas
- **Don't brainstorm; get close to reality.** Brainstormed ideas are "bad and plausible". ≥70% of
  top-100 YC companies' ideas came organically; ~90% of Snyder's $1M+ ARR companies came from
  founders getting first-hand exposure (shadowing, embedding, doing the job). *(Graham "Before";
  Friedman; Snyder)*
- **Founder–market fit is the most important question.** Start from what you're unusually good at or
  unusually placed to see; list every job and life experience and ask what you know that others
  don't. *(Friedman)* Play to your existing skills and network; a programmer entering banking spends
  months getting meetings. *(Fitzpatrick)*
- **Avoid solutions in search of a problem and tar pits.** "AI is cool, what can I apply it to?"
  finds made-up problems. *(Friedman)* Copying a winner with no theory of which customer they leave
  blocked is the tarpit Snyder sees most.
- **Welcome schleps, boring spaces and competitors.** No competitors often means no demand. A fresh
  spin on a budgeted category ("a better CLM") beats inventing a category. *(Friedman; Todd Jackson
  on Ironclad)*
- **Have a "why now?"** and prefer a small fast-growing market to a big slow one. Dominate a small
  market, then expand. *(Altman; Friedman)* Attack incumbents when the paradigm shifts, not head-on.
  *(Graham "Ambition")*

### Talking to users
- **Ask about past facts, never opinions of your idea.** Compliments are the "fake phone number" of
  feedback. *(Fitzpatrick)*
- **Customers are experts on the problem; you are the expert on the solution.** Don't be a waiter
  taking orders. Users have good problems and bad solutions: "inbox and email on one screen" meant
  Gmail was slow. *(Caldwell & Seibel; Alströmer)*
- **Hunt for surprise.** If you can predict everything users say, you'll find nothing new. Users
  normalise pain ("17 clicks is just my job"); you only see it by watching. *(Caldwell & Seibel)*
- **In B2B, learn how the customer makes money** and how your user contributes; saving three clicks
  isn't value. *(Caldwell & Seibel)*
- **Test money, not opinions.** Probe existing budget, approval path, and fair / expensive /
  prohibitive price. "No budget" and "talk next year" mean no. *(Todd Jackson)* If you feel you're
  pushing or convincing, that's a red flag. *(Snyder)*

### Product-market fit
- **PMF is unmistakable pull.** Every metric jumps at once; customers run ahead of you. If you have
  to ask whether you have it, you don't. Pity usage and idle interest are not PMF. *(Reinhardt;
  Snyder)*
- **But it's also a ladder.** Nascent (3–5 paying customers with an urgent problem) → developing
  (~25) → strong (~100) → extreme. Optimise satisfaction first, then demand, then efficiency. Even
  great founders rarely *feel* it, so use metrics. *(Todd Jackson, sales-led B2B)*
- **Efficiency is part of fit.** Selling $2 for $1 (the "$100 vending machine") is not PMF. The next
  customer should keep getting easier and cheaper to win. *(Todd Jackson)*
- **Retention is the truth.** Track returning usage (D1/D3/D7/D30), cohort renewal, actives and NPS
  (above ~40–50 suggests fit); never total sign-ups. Press spikes decay; watch the baseline between
  them. *(Rusenko; Altman)* 95% vs 90% monthly retention leaves 54 vs 28 of 100 customers after a
  year. *(Epstein)*
- **Watch for users bending the product to a new job, or loving a side piece** (Segment's
  analytics.js). A foothold expands: "5 of 5 customers hand-build the same pipeline" names the next
  product. *(Rusenko; Reinhardt)*
- **When stuck, change one of the four Ps:** persona, problem, promise, product. Cheap levers
  (promise, mock-ups) before rebuilding. Founders do 10% pivots when they need 200% ones. *(Todd
  Jackson, citing Jack Altman)*
- **With mass churn, find the segment that stays** and go vertical and premium. *(Snyder)*

### First customers
- **Founders sell.** You can't outsource it or know the product is bad until you've genuinely tried
  selling it. Writing more code to avoid selling doesn't work. *(Alströmer)*
- **Customers 1–3 from your network; 4–10 from unscalable work** (visits, community replies, small
  dinners, give-before-ask); tools (Apollo, Clay) only from about 10–50. Early buyers buy the
  founder. *(Max)*
- **Channel follows the buyer's day.** Offline buyers aren't won by cold email: 3 days at a trade
  show beat 3 months of email for one founder. *(Max)*
- **Sales is a numbers funnel; work backwards.** 500 sends → 250 opens → ~20 replies → 10 demos → 2
  customers. 100 sends tells you nothing. Track every stage, onboarding included. *(Alströmer)*
- **Copy:** under 75 words (or 6–8 sentences), plain text, founder voice, show social proof, one
  call to action, give value first, follow up 3–4 times, read it aloud to de-AI it. Map the buyer's
  day first; exhaust warm and second-degree intros before tools. *(Max; Alströmer)* Snyder hasn't
  seen AI personalisation work; it reads as machine-written.
- **Easiest customers first** (people you know, startups); most people aren't early adopters, so you
  need volume to find the few. Don't fake size. *(Alströmer; Blomfield)*

### Positioning
- **Derive it in order:** competitive alternatives → differentiated capabilities → value ("so
  what?") → best-fit customers → market category → sales story. "Better" means "what value, for
  whom". *(Dunford)*
- **The status quo is your biggest competitor.** About 40% of B2B deals are lost to "no decision".
  To move someone off a spreadsheet you must be dramatically better. *(Dunford; Alströmer)*
- **Keep it loose early, tighten on a pattern.** Market the tuna net for "big fish" until you see it
  catches grouper. Tighten when different-looking customers share a thread. *(Dunford)*
- **Arm the champion** with a story they can sell to their boss. Test positioning on qualified
  prospects, not friends. *(Dunford)*
- **Weak positioning tells:** "can you back up?", "so you're like Salesforce?", "I could do that in
  a spreadsheet". Write it down; re-run it as the market drifts. *(Dunford)*

### Business models
- **Copy a proven model; innovate on product.** SaaS, transactional and marketplaces are 67% of YC's
  top 100. Prefer recurring revenue, close to the transaction, scaling with software. *(Epstein)*
- **Weak primary models:** ads (only if you'll be a top-10 site), affiliate (far from the
  transaction, paid 30–90 days later), services (scale with people), anything that depends on
  another platform's goodwill. Services are fine as a way to *learn*. *(Epstein; Friedman Q9)*
- **Pick the company type that fits your goal:** scalable (investor-funded, years of losses),
  reliable (someone pays within about a month, profit-funded, no investors, no loans on your house),
  or lifestyle. *(Fitzpatrick)*

### Focus, ambition and the founder
- **Steak, not sizzle.** Press, conferences, hiring, branding and fundraising are sizzle; shipping
  and talking to users are steak. Don't "play house". *(Seibel "Mistakes"; Graham)*
- **Shipping speed is still the best predictor, even with AI**; the bottleneck includes deciding
  *what* to build. *(Graham "Ambition")*
- **Discipline beats genius;** act with 12-months-of-runway urgency now. *(Caldwell & Seibel)*
- **Cost of mistakes, not size, decides survival.** Stay a cheap "cockroach" so a failed launch is a
  blunder, not death. Stage spending to milestones. *(Fitzpatrick; Reinhardt; Graham)*
- **Pick a problem and users you'll care about for 5+ years**; lost motivation kills many startups.
  *(Seibel "Mistakes")* Choose the idea you think about when you're not trying to. *(Altman)*
- **Startups are all-consuming;** at most one venture can be the real startup. A side project
  becomes a startup when it takes an alarming share of your life. *(Graham "Before")*
- **Know when to stop.** 4–5 years without pull rarely turns around. *(Todd Jackson)*
- **Validation from investors, press, grants, peers or competitions is a driver's licence, not
  proof.** "It worked in country X" doesn't transfer. *(Caldwell & Seibel)*

---

## 6. Checklists

**6.1 Idea scorecard** *(Friedman, plus Snyder and Altman)*
- [ ] Founder–market fit: are we obviously the right team?
- [ ] Market big now, or small and growing fast; can it expand beyond the first niche?
- [ ] Acute: is the current alternative literally nothing, or painful?
- [ ] Competitors exist (demand proven), and we have a new insight.
- [ ] We want it, or know people who do; would have paid for it.
- [ ] Recently became possible or necessary ("why now?").
- [ ] Proxy: a non-competing company elsewhere proves the model.
- [ ] We'd work on it for years.
- [ ] Scales with software, or has a plan to stop scaling with people.
- [ ] Fertile idea space with adjacent ideas if this one fails.
- [ ] One-sentence explanation; a PULL card with all four letters.

**6.2 User interview** *(Fitzpatrick; Alströmer; Snyder; Todd Jackson)*
- [ ] Goal written down: what must I learn?
- [ ] Opening: "Why did you take this call?" or "Tell me how you do X today."
- [ ] Follow-ups: hardest part? why? how often? why does it matter? what do you do now? what does
      that cost? "last time it happened, what did you do?"
- [ ] Ask to see it: screen-share the spreadsheet, notebook, bank app, workflow.
- [ ] Idea kept out until the end, or entirely; no yes/no, "would you use", or "which features".
- [ ] Probe U and L: "Is it OK if this waits?" "Are your current options good enough?"
- [ ] Money: existing spend, who approves, fair / expensive / prohibitive price.
- [ ] Exact quotes, emotions and past spending logged; P/U/L/L filled live.
- [ ] Ended with a next step (intro, follow-up, deposit), or the Hail Mary.
- [ ] Feature requests translated into the underlying problem.

**6.3 Is this PMF yet?**
- [ ] Pull-shaped actions without pushing: "how do we start?", intros, unsolicited payment.
- [ ] Retention curve flattens: a cohort keeps returning at its natural frequency.
- [ ] Users would be *very* disappointed if it vanished (ask one-on-one: "don't be nice; are we a
      necessity?"). *(Todd Jackson citing Rick Song; Altman)*
- [ ] Organic word of mouth; inbound you can't trace. *(Altman; Todd Jackson)*
- [ ] Customers share the same core use, not a different "most important feature" each (otherwise
      you're a consultancy). *(Todd Jackson)*
- [ ] Paying (B2B), or a credible path to revenue that doesn't rely on subsidy.
- [ ] The next customer is easier and cheaper than the last.
- [ ] Red flags absent: pity usage, signups up but actives flat, "great, I understand the value", "a
      partnership will save us". *(Reinhardt; Rusenko; Altman)*

**6.4 Weekly founder review** (30 minutes, same time each week)
- [ ] What did I ship *to users* this week? *(Graham "Ambition")*
- [ ] How many user conversations? What surprised me? *(Caldwell & Seibel)*
- [ ] The one metric per venture: actives / retention cohort / revenue / funnel stage counts.
- [ ] Which station is the bottleneck: pipeline, sales or delivery? Next week works only on it.
      *(Snyder)*
- [ ] Am I doing sizzle (branding, logos, social accounts, grants, tools) instead of steak?
- [ ] Did the written spec change? Was the change deliberate? *(Seibel)*
- [ ] Which P am I testing next, and what result would make me kill it? *(Todd Jackson)*
- [ ] Is each venture still the one I'd care about for 5 years? Which one is the real startup?

---

## 7. Failure modes to hunt for

- Months of code before any customer touches it; many half-working features in a money product.
  *(Reinhardt; Seibel; Caldwell & Seibel)*
- Compliments, idle interest, sign-ups, press spikes, pity trials or design-partner enthusiasm read
  as fit. *(Reinhardt; Snyder; Rusenko)*
- Free pilots and trials; underpricing; cost-plus; "X but cheaper". *(Snyder; Blomfield; Epstein)*
- 100 cold emails then "outbound doesn't work"; cold email to offline buyers; tools before
  second-degree intros. *(Alströmer; Max)*
- Sliding product and market at once; 10% pivots when 200% is needed. *(Reinhardt; Todd Jackson)*
- Growth machines, hiring or fundraising before love; public presales before you can deliver.
  *(Altman; Fitzpatrick; Caldwell & Seibel)*

---

## 8. Where the teachers disagree, and how to resolve it

| Tension | Resolution |
|---|---|
| Stop interviewing at 70–80% predictability *(Todd Jackson)* vs. hunt for surprise; predictable answers mean wrong questions *(Caldwell & Seibel)* | Stop *validating* a hypothesis once answers repeat; keep *exploring* for anomalies in every later conversation. |
| "10x better" is the wrong test; if they aren't blocked, 100x won't sell *(Snyder)* vs. be dramatically better than the spreadsheet *(Alströmer)*, minimum remarkable *(Rusenko)* | Check urgency first (U), then size of improvement over the status quo. Both must hold; urgency is the one founders skip. |
| No pilots or design partners *(Snyder)* vs. paid 2–4-week proofs *(Blomfield)*, heavy pre-sale work (Looker) *(Todd Jackson)* | Never *offer* free partnerships. If procurement requires a trial, make it short, paid and judged on the value equation. |
| Don't pitch; ask about their life *(Fitzpatrick; Alströmer)* vs. sell from day one *(Alströmer, First Customers; Snyder)* | Learning comes first in each conversation; "earn the right" to ask, then keep the pitch tiny once pull appears. Selling is also learning. |
| Pivot fast, kill ideas *(Reinhardt; Todd Jackson)* vs. pivots rarely become big; ideas matter *(Altman)* | Kill fast *before* commitment; once you have a loved foothold, iterate on the solution, not the problem. Big wins usually pivot into something the founders wanted. |
| Charge from day one *(Epstein; Alströmer)* vs. free early, monetise after traction *(Rusenko's Weebly)* | B2B: charge. Free consumer products may defer, but must prove retention and design a revenue path early (efficiency). Weebly's "revenue caps valuation" view is dated. |
| Service-first is fine *(Snyder; Todd Jackson L1)* vs. services rarely become giants *(Epstein; Friedman Q9)* | Fine as a cash engine and learning wedge. Watch for every client wanting a different core deliverable; productise what repeats. |
| How many interviews: ~5 per hypothesis *(Snyder)*, 6 *(Fitzpatrick)*, 5–10 *(Rusenko; Alströmer)* | ~5–6 per hypothesis per segment; more only if answers split into clusters. |

---

## 9. Not covered by the sources, but know this

> **Reviewer's box** (helper addition, not from the 19 talks; verify current rules and fees).
>
> **Distribution in Nigeria.** WhatsApp is the default channel: broadcasts, Status, Channels and
> community groups (church, alumni, estate, work, ajo) beat ads. Design a "send this to a friend"
> moment into the product. Answer public complaints on X, TikTok, Instagram and Nairaland (Max's
> Reddit play, localised); use events, campuses and NYSC groups as micro-events. Build for low-end
> Android, costly data and patchy power.
>
> **Payments.** Paystack and Flutterwave are the main naira rails; many users prefer bank transfer
> to cards. Collecting USD from foreign clients usually needs a registered business and a
> domiciliary account or a receiving-account service. Register before you need business payment
> accounts.
>
> **Naira volatility.** Price locals in naira, foreign clients in USD/GBP. Review naira prices on a
> schedule (e.g. quarterly), avoid long fixed naira contracts, add FX clauses to big ones. Small
> frequent tickets (weekly, per campaign) suit tight cash flow better than annual plans.
>
> **Trust and regulation for money products.** Holding, moving or pooling user money needs a CBN
> licence or a licensed partner; investment advice or collective schemes can fall under SEC Nigeria.
> A planner that only organises, and says plainly what it does not do, stays outside most of that.
> Personal data falls under the Nigeria Data Protection Act 2023 (NDPC): lawful basis and consent,
> purpose limitation, minimisation, security, breach notification. Financial data is sensitive:
> sharing user-level data without explicit, revocable consent is a legal and a trust risk;
> aggregated, anonymised, opt-in insights are the defensible version. Get a lawyer's read before
> anything touches money flows.
>
> **Unit economics.** CAC = acquisition spend ÷ new customers. Contribution margin = revenue −
> direct costs (creator fees, API, payment fees). LTV ≈ monthly margin ÷ monthly churn. Aim for LTV
> ≥ 3× CAC and payback under ~12 months. For services, price your own hours into margin.
>
> **Retention cohorts.** Group users by start week or month and plot the share still doing the key
> action later. The curve must *flatten*. Measure at the natural frequency: for a salary planner
> that's payday, so "planned again next payday" beats D7. Sean Ellis's ≥40% "very disappointed"
> survey is a common quick PMF check.
>
> **Solo with two jobs.** Hours are the capital. One main venture, the rest on maintenance; fixed
> weekly blocks for building, outreach and calls; batch outreach; let AI research, draft and code
> but keep customer conversations and judgment yourself. Check employment contracts for conflict and
> IP clauses. Protect sleep; the talks mention burnout (Reinhardt, Moskovitz).

---

## 10. Applying this to Sam's work

**Tereka (free money planner).** First users are people Sam knows *(Seibel)*. Position against the
status quo (mental maths, notebook, WhatsApp note, ajo, bank app), not other apps *(Dunford)*. Find
the U (salary week, rent, school fees); the job is "don't run out before month-end", not "use a
planner" *(Rusenko)*. Being free, the signal is behaviour: payday-cadence retention and referrals,
never sign-ups *(Fitzpatrick; Alströmer)*. One reliable job beats many cups; a money tool that is
ever wrong is the hated-company outcome *(Caldwell & Seibel)*. Consumer budgeting is a known tarpit
(Plaid began as a failed budgeting app), and copying a US/UK app is the "worked in country X"
mistake *(Friedman; Todd Jackson; Caldwell & Seibel)*. *Making money without breaking trust:* never
charge users and never hold money. Ads and affiliate are weak primary models; "transactional" means
moving funds, i.e. licensing *(Epstein)*. Trust-safe paths: (1) paid B2B for employers, cooperatives
or HR, priced on a value equation *(Blomfield)*; (2) clearly labelled, opt-in partner referrals; (3)
aggregated, anonymised, consented reports (the "how Nigerians spend" idea), never user-level data.
Publish a plain "what we do not do" page, and design the revenue path early so growth isn't a $100
vending machine *(Todd Jackson)*.

**Riselist (service-first agency).** Endorsed as a learning wedge and cash engine *(Snyder;
Epstein)*; PMF level 1 by design *(Todd Jackson)*. With US/UK/Canada brands as the focus, cold email
and LinkedIn fit the buyer's day *(Max)*; work volume back from the goal (hundreds of sends for a
couple of clients) *(Alströmer)*. Give before ask: a three-point TikTok teardown or a sample creator
shortlist *(Max)*. Price per campaign or retainer on a value equation (sales or CPA vs current
influencer spend), in USD, upfront or with a guarantee, never free pilots *(Blomfield; Alströmer)*.
Productise only what every client keeps asking for *(Todd Jackson; Graham)*.

**Trameter features.** Name the person and project per feature *(Snyder)*. Drive Button:
hand-onboard the first bloggers, position against "plain affiliate links / do nothing", find the X
moment (first booking click?) *(Seibel; Dunford; Snyder)*. Owning the booking moves Trameter closer
to the transaction than affiliate commissions *(Epstein)*; make one flow bulletproof first
*(Caldwell & Seibel)*. If partners hand-build the same mapping, that's the product *(Reinhardt)*.
LLM-native discovery is a paradigm-shift attack *(Graham)*; count token costs as cash *(Blomfield)*.

**Sam, solo in Lagos with two jobs.** Fitzpatrick's "reliable, keep the day job" founder: favour
ideas that pay within about a month or cost nearly nothing. His edge is data, AI and pipelines;
ideas from what he built for himself or Trameter beat list-made ones *(Graham; Friedman)*. Run the
weekly review, pick one real startup, and treat names, logos, socials and grants as sizzle unless
they unblock users; grants are a driver's licence, not PMF *(Seibel; Graham; Caldwell & Seibel)*.

---

## 11. Where to look in `references/`

| File | What it covers |
|---|---|
| `startup-1-snyder-reinhardt-graham-caldwell-seibel.md` | Snyder (PULL, case-study factory, outreach benchmarks, X moment), Reinhardt (Segment, what PMF feels like, kill ideas fast), Graham ×2 (Before the Startup; ambition, shipping speed, lean in the AI era), Caldwell & Seibel (simple products, rookie mistakes), Seibel ×2 (MVP planning; biggest founder mistakes) |
| `startup-2-jackson-rusenko-friedman-epstein-blomfield-caldwell.md` | Todd Jackson (PMF levels, four Ps, dollar-driven discovery), Rusenko (iteration loop, retention metrics, channel–price fit), Friedman (getting and scoring ideas), Epstein (business models, pricing), Blomfield (B2B value equation, pilots), Caldwell & Seibel (talking to customers, hunting surprise) |
| `startup-3-dunford-fitzpatrick-altman-alstrmer-max.md` | Dunford (positioning), Fitzpatrick (Mom Test, company types, cost of mistakes), Altman & Moskovitz (idea, product, love vs like, why start), Alströmer ×2 (first customers funnel; talking to users), Max (first 10 customers, channels, copy) |
| `INDEX.md` | Every video with its line number |
