---
name: ai-engineering-first-principles
description: First-principles AI engineering playbook from 43 talks (Karpathy, Hamel Husain & Shreya Shankar, Chip Huyen, Jason Liu, Anthropic and AI Engineer conference speakers). Use when building, evaluating, debugging or costing LLM features: prompts, structured outputs, RAG, agents, tools/MCP, evals, guardrails, model choice.
---

# AI engineering from first principles

This skill turns 43 talks and lectures (44 note sections) into working judgment for building products on LLMs.
This file holds the procedure, the principles the teachers keep coming back to, and
checklists. `references/` (when present) holds detailed notes for every video, each credited
to its teacher.

Use it whenever an LLM is part of a system, even if the user doesn't say "AI engineering":
extracting fields from documents, classifying rows, a chatbot or search feature, an agent with
tools, choosing a model, cutting LLM cost, or asking "is this output good enough?".

**How to use it**
1. Follow the build procedure for any new LLM feature.
2. Run the relevant checklists against the design, prompt or code.
3. If a `references/` folder sits next to this file, use it for depth. `references/INDEX.md`
   lists every video with its line number, or run `grep -n -i "keyword" references/*.md`. Each section ends with a "Relevance to Sam" or "For Sam" note: Sam builds an AI pipeline that turns travel blog
   articles into bookable trips (gate → extraction → trip building), classifies millions of
   location rows on a budget, and uses small/local models on a 16 GB laptop. Use those parts
   only in that context. Lines marked "(note: …)" and those Sam notes are helper additions,
   not the speaker's words. If `references/` is missing, the full notes live in the user's
   private GitHub repo `Biodun-ml/engineering-skills` (folder
   `ai-engineering-first-principles/references/`); clone it when you need depth.
4. Explain the *why*, and credit the speaker when you use a specific pattern.
5. Model names, prices and product features in the talks date quickly. Check current ones
   before recommending a specific model or API.

---

## 1. Build procedure

1. **Define the outcome and its cost of error.** What concrete result proves the feature works
   ("the trip has the right city and dates", not "good summaries")? How bad is a wrong answer?
   Internal drafts tolerate errors; anything customer-facing or money-moving needs strict
   evals and guardrails. Brainstorm worst cases first. *(Chip Huyen; Shreya Shankar)*
2. **Pick the paradigm per sub-task:** plain code (1.0), a small trained model (2.0), or a
   prompt (3.0). Arithmetic, counting, lookups, string handling and ID checks go to code. Use
   the LLM only for the fuzzy part. *(Karpathy — Software 3.0, Deep Dive)*
3. **Prefer a workflow to an agent.** If you can draw the decision tree, if volume is high,
   budget small, or errors costly and hard to detect, build a mostly deterministic pipeline.
   Use an agent only for open-ended problems where the path can't be mapped.
   *(Barry Zhang — Effective Agents; Dex Horthy — 12-Factor Agents)*
4. **Put the truth in the context.** Retrieve or pass in the source text, current prices,
   availability and IDs. Never rely on the model's memory for facts that must be right or
   current. *(Karpathy)*
5. **Make the output typed.** Define a schema (e.g. Pydantic) with an explicit
   "unknown/not found" option, reasoning or evidence fields *before* the answer, use the provider's strict JSON-schema/constrained decoding where available, then validate
   meaning in code (IDs exist, quotes appear in the source) and retry with the errors. *(Jason Liu — Pydantic; Karpathy — "models need
   tokens to think")*
6. **Get a baseline working, then build evals before optimising.** A capable model is fine for
   prototyping; escalate or downgrade models against the eval bar, don't default to the
   biggest. Fix obvious bugs before writing any eval. Instrument full traces from day one.
   *(Hamel Husain; Chip Huyen; Karpathy)*
7. **Do error analysis, then fix the biggest bucket** (section 2). Repeat.
8. **Then optimise cost and latency against the eval bar:** cheaper models, cascades, caching,
   batching, splitting the task. Fine-tune last, if ever. *(Chip Huyen; Shreya Shankar —
   cascades; Revolut)*
9. **Ship with a human in the loop, then widen autonomy** as acceptance rates prove it:
   AI suggests → humans accept → automate the categories with high acceptance.
   *(Karpathy — autonomy slider; Chip Huyen — crawl-walk-run)*

---

## 2. Principles that recur across the teachers

### Context and prompting
- **Context is working memory; weights are vague recall.** Hallucination starts where recall is
  rare. Ground answers in provided text and allow "I don't know". *(Karpathy; John Berryman)*
- **Models need tokens to think.** Each token gets a fixed slice of compute, so ask for reasoning or evidence before the answer, never answer-first. With native
  reasoning models, skip a separate chain-of-thought field but keep an evidence/quote field
  for verification. Use reasoning models for hard,
  verifiable problems; plain models are faster for knowledge and simple tasks. *(Karpathy)*
- **Context engineering is the main lever.** Gather, rank, trim, then assemble into a familiar
  document shape. Fewer, denser tokens beat stuffing; extra tokens distract and cost. Compact
  errors and drop stale history. *(Berryman — Copilot pipeline; Horthy; Richmond Alake)*
- **Few-shot examples in delimited (XML-style) blocks are the cheapest accuracy boost.** Save
  prompts as versioned templates and treat changes as tracked experiments. Framework default
  prompts can change under you. *(Karpathy; Chip Huyen; Shreya Shankar)*
- **Byte-level details matter.** Whitespace, templates and tokenizers swing results; keep eval
  and production prompts identical. Tokenization explains failures with spelling, counting and
  some languages. *(Hailey Schoelkopf; Karpathy — tokenizer)*

### Structured outputs and verification
- **Most production LLM work is language → typed data.** Docstrings are prompt; validators are
  guardrails. *(Jason Liu)*
- **Make verification structural.** Require quotes that must literally appear in the source,
  check entities and IDs against your own database (models "dream" plausible IDs), and record
  provenance end to end so every claim traces back to a passage. *(Liu; Karpathy; Douwe Kiela)*
- **For closed label sets, constrain the output** (enums, structured decoding, log-likelihood
  over choices) to get valid labels plus a confidence score for routing. *(Schoelkopf; Berryman
  — logprobs)*

### Evals (the core discipline)
- **Look at your data first.** Start with 20–30 real traces and keep going to saturation (often ~100), in a friction-free viewer, write
  free-form notes, cluster them into a failure taxonomy, count, fix the biggest bucket. Note the *first upstream* failure in each trace. A round ends when 5–10 more traces show
  nothing new. *(Hamel Husain & Shreya Shankar — error analysis)*
- **Evals are bespoke.** Benchmarks and generic "helpfulness/hallucination" scores aren't
  actionable. Measure the concrete outcome users care about. *(Husain & Shankar; Chip Huyen)*
- **Tier your evals:** code assertions → aligned LLM judges → humans. Prefer assertions
  wherever a deterministic signal exists; they can also run in production to trigger retries.
  *(Husain & Dan Becker; Teresa Torres)*
- **An LLM judge is a model you must evaluate.** One narrow binary pass/fail judge per failure
  mode, a few labelled examples, validated against human labels (train/dev/test) with a
  confusion matrix — never raw agreement, since an always-pass judge looks 90% accurate when
  failures are rare. Pin and version the judge's model and prompt. No 1–5 scales.
  *(Husain & Shankar; Bryan Bischof)*
- **Evaluate each pipeline stage separately, as a failure funnel** (retrieval as retrieval,
  extraction as precision/recall, final answer on its own). Fix early-stage failures first:
  later stages never get a chance on those cases. *(Chip Huyen; Eugene Yan; Bischof; Uber)*
- **Small offline sets are regression tests, not accuracy estimates.** Keep a hard suite
  (~60–70% pass), delete evals that never flip, replay real traffic, and monitor online.
  Criteria drift as you grade and as outputs improve — treat them as living. *(Husain &
  Shankar; Bischof; Shankar — EvalGen)*
- **Domain experts own prompts and labels.** Build a small custom annotation UI that shows
  outputs as they really appear next to their sources. Buy trace logging; build the viewer.
  *(Husain & Shankar)*
- **Optimisers and judges get gamed.** Auto-tuning against one narrow judge produces "safe but
  meaningless" outputs; keep broad evals and sanity-check results against reality. Swap the
  order of candidates to catch judge position bias. Repeated "do not X" prompt edits are
  ready-made assertions. *(Uber; Kapoor; Karpathy; Husain & Bischof; Shankar — SPADE)*
- **Your taste is the differentiator.** Don't ask an agent to "evaluate my app"; have AI build
  the review UI, cluster your notes and propagate your labels. Make human verification fast
  (diffs, side-by-side sources). Always read AI-generated analysis code. *(Shankar; Karpathy)*
- **Synthetic data: generate with one model, verify with independent ones,** and eyeball
  50–100 samples per stage. Prefer precision over volume. *(Thakur)*
- **Reliability, not capability, is the job.** pass@k is not "right every time"; 80% right can
  be a product failure. Retry-until-a-verifier-passes inherits the verifier's false positives.
  *(Sayash Kapoor)*

### Retrieval (RAG)
- **Retrieval is classic information retrieval.** Build your own judged set (query, doc,
  label), measure recall@k / MRR / nDCG, and always keep a tuned BM25 baseline. A better
  retriever alone can move end results more than a better LLM. *(Nandan Thakur — ~60%→~90% from the retriever alone; Jo Bergum; Jerry Liu)*
- **Fix the data layer first:** layout-aware parsing, sensible chunks (small, with context
  metadata), metadata filters inferred from the query, hybrid keyword + dense search, then
  rerankers. More retrieved tokens ≠ better answers. Allow zero results — vector kNN always returns something. *(Jerry Liu; Eugene Yan;
  Bergum)*
- **Segment questions and improve by segment.** Cluster real queries, find which segments fail
  (missing data vs. poor retrieval vs. capability), and build specific indexes or tools per
  segment. *(Jason Liu — systematically improving RAG)*
- **Retrieval MVP = BM25 + embeddings + metadata pre-filters + a reranker.** Embeddings are lossy
  on names, jargon and numbers; pre-filter by region/type/time so top-k can't fill with
  wrong-but-similar documents; a cross-encoder reranks a few dozen candidates. Generate
  synthetic questions per chunk on day one to measure retrieval; fine-tune a small embedder
  once you have real pairs. *(Ben Clavié; Jason Liu)*
- **Dates are a known weak spot.** Inject today's date and document dates, normalise dates at
  ingest, add date filters, and keep a date-resolution test set. *(Jacob Carter with Hamel;
  Jason Liu)*
- **Spend compute at ingest** (summaries, generated questions, pre-extracted facts) and use
  small-to-big retrieval. *(Jerry Liu; Jason Liu)*
- **Use graphs only when relationships already exist** and a concrete query needs them.
  *(Eifrem — vendor talk; caution from Shreya Shankar)*

### Agents and tools
- **Agents are just software.** Own the prompts, context window, control flow and state; tool
  calls are JSON plus a switch. Use a mostly deterministic graph with small micro-agents,
  pausable and resumable from persisted state. *(Horthy — 12-Factor Agents)*
- **Keep the core simple:** environment + tools + system prompt, and debug from inside the
  agent's context window. *(Barry Zhang)*
- **Compile recurring behaviour into code.** When an agent keeps doing the same thing, turn it
  into a saved script, a rule or a skill. Package expertise as versioned skills plus MCP for
  connectivity instead of building a bespoke agent per domain. *(Blumenfeld — graph
  engineering; Barry Zhang & Mahesh Murag — Skills)*
- **Design tools for agents, not as REST mirrors.** Few tools per agent (tool search or
  specialist sub-agents beyond that), retries/auth/rate limits handled server-side, read vs
  write marked, and let the model write one script over typed functions instead of many
  round-trips. *(Murag — MCP workshop; David Soria Parra — Future of MCP)*
- **Harness over prompt.** Put determinism in code around the model: tool registry, max steps
  and attempts, context compaction, credentials handled by privileged code (never in the
  prompt), and success checked from the tool trace or world state — never from the model's
  self-report. A good harness lets cheap models do the work. *(Tejas Kumar)*
- **Agents hide their mistakes.** They work around broken environments and report success.
  Grade every run from its tool calls, and turn recurring failures into config/skill fixes
  reviewed by humans. *(Ben Holmes — Warp)*
- **Separate creator from verifier.** A fresh-context validator that never saw the work
  (ideally a different model) reviews adversarially; handoffs record commands and exit codes.
  *(Luke Alvoeiro — Factory)*
- **Close the loop in production:** sample live traffic → human labels → find the faulty
  component → tune prompt/config → benchmark on the golden set → version with rollback. Bound
  self-correction to K tries and allow "skip / don't publish" as a valid outcome.
  *(Uber — closed-loop evals)*
- **Measure agent efficiency:** tool calls per task, repeated queries, loop length. Long loops
  correlate with failure — cap turns and dedupe. *(Thakur)*
- **Persist state and failures** as memory/lookup tables so pipelines stop repeating mistakes.
  *(Alake; Torres)*

### Security
- **All third-party content is untrusted input.** Hidden text in pages, docs and images can
  inject instructions; rendered URLs can exfiltrate data. Never let retrieved content choose
  tool calls or URLs. Avoid the "lethal trifecta" in one agent: private data access +
  untrusted content + an outbound channel. Sandbox agents, allow-list egress, scope credentials, keep secrets in the harness
  (never the prompt), log model calls through a redacting gateway, and firewall destructive
  commands. Treat MCP tool descriptions and outputs from third-party servers as untrusted too,
  and pin the servers you use (current practice; not from the talks). *(Patterson; Kumar)*
  *(Karpathy; Michael Patterson — Lethal Trifecta, restated per Simon Willison)*

### Cost and models
- **Report cost and latency next to accuracy and choose from the Pareto frontier.** A model with
  similar accuracy at 10× lower cost wins. *(Kapoor)*
- **Restructure the task before waiting for a better model:** split → map → reduce, split
  multi-field prompts, push cheap filters (keywords, embeddings, code) first. *(Shreya Shankar
  — DocETL)*
- **Cascade and route.** Send items through a cheap model or simpler proxy question first and
  escalate low-confidence cases, with thresholds calibrated on a labelled sample (DocETL showed
  ~86% cost cut at a 90% accuracy target). Routers send easy items to cheap/local models, hard
  ones to strong models, sensitive ones to humans. *(Shankar; Chip Huyen; Karpathy)*
- **Small fine-tuned classifiers** (PII, hallucination/NLI checks) can replace LLM calls at
  scale for fast checks. *(Eugene Yan; Chip Huyen)*
- **Mind tokens:** non-English text can cost ~3× the tokens; never let untrusted text be parsed
  as special tokens. *(Karpathy — tokenizer)*
- **Self-hosted open models can make the cheap tier nearly free at scale,** once evals prove
  they meet the bar. *(David Abitbol — Revolut)*
- **Climb the complexity ladder only as far as needed:** prompt → examples → retrieval/data
  prep → fine-tuning last (it carries hosting/maintenance cost and is quickly overtaken by new
  base models). *(Chip Huyen)*

### Building with AI (coding and teams)
- **Software fundamentals scale AI coding:** agree on a design concept first, keep a shared
  glossary, small test-driven steps, deep modules with clear interfaces. *(Matt Pocock)*
- **Plan AI work as experiments, not dated features.** Each item is a question, a metric and an
  expected effect from a baseline and a small pilot; write success criteria before building so
  tests aren't shaped by the code. *(Bryan Bischof with Hamel Husain — Stop Managing AI Projects;
  Luke Alvoeiro — Factory)*
- **Make your own systems legible to agents:** markdown docs, llms.txt, API equivalents of UI
  steps, MCP. *(Karpathy — Software 3.0)*

---

## 3. Checklists

**New LLM feature**
- [ ] Concrete success outcome and worst-case harm written down.
- [ ] Each sub-task assigned to code, small model or prompt; exact work in code.
- [ ] Workflow vs agent justified.
- [ ] Facts, prices, IDs supplied in context; "unknown" allowed.
- [ ] Typed output schema, reasoning/evidence before answer, validation + retry.
- [ ] Full traces logged (inputs, retrieved context, prompts, outputs, costs, latency).

**Evals**
- [ ] 20–30+ traces read to saturation; first upstream failure noted; taxonomy with counts.
- [ ] Code assertions for every deterministic check.
- [ ] One binary judge per remaining failure mode, validated against human labels.
- [ ] Per-stage metrics (retrieval, extraction, final).
- [ ] Regression set run on every prompt/model change; online monitoring.
- [ ] Judge and prompts versioned; eval prompts byte-identical to production.

**RAG**
- [ ] Judged query set (synthetic questions per chunk to start); BM25 baseline; recall@k/MRR tracked.
- [ ] Metadata pre-filters + reranker; today's date injected for time questions.
- [ ] Parsing and chunking checked by eye.
- [ ] Zero-result path; citations to source passages.
- [ ] Failing query segments identified and handled.

**Agents and tools**
- [ ] Deterministic steps are code, not model-chosen tools.
- [ ] Small tool set per agent; read vs write marked; server-side retries/auth/rate limits.
- [ ] State persisted; resumable; turn cap; loop detection.
- [ ] Success verified from tool results/world state, not the agent's own report.
- [ ] No agent combines private data + untrusted content + outbound channel without a human gate.
- [ ] Destructive or money-moving actions require confirmation.

**Cost**
- [ ] Cost/latency per task measured alongside accuracy.
- [ ] Cheaper model tested against the eval bar; cascade/routing with calibrated thresholds.
- [ ] Prompt/prefix caching (stable instructions and examples first, variable input last);
      batch APIs for offline bulk jobs; upstream steps cached on a fixed dev set while
      iterating on evals *(Torres)*.

---

## 4. Failure modes to hunt for in reviews

- Asking the model to do maths, counting or exact lookups in its head.
- Answer-first schemas; no "unknown" option; free-text parsing with regex.
- Trusting model-produced IDs, URLs or prices without checking them against your data.
- A single 1–5 "quality" score from an unvalidated LLM judge.
- Six hand-picked examples treated as an accuracy measurement.
- Stuffing more retrieved chunks to fix bad answers; no retrieval metrics.
- An agent where a fixed workflow would do; dozens of tools on one agent.
- Retrieved web/blog text allowed to trigger tool calls or links.
- Using the biggest model for every row of a million-row job.
- Fine-tuning before prompts, examples and retrieval are exhausted.
- Prompt auto-tuning against one judge with no broad regression set.

---

## 5. Where to look in `references/`

| Need | File (see INDEX.md) |
|---|---|
| Karpathy deep dive + Software 3.0, DocETL/cascades, prompt engineering workshop, eval mistakes, 12-factor agents, Pydantic, GraphRAG, software fundamentals | `ai-a-*` |
| Instrumenting & evaluating LLMs, error analysis UI, Chip Huyen ×2, Karpathy intro, Bergum RAG basics, Kiela RAG lessons, effective agents, skills, Kapoor, graph engineering | `ai-b-*` |
| Karpathy "How I use LLMs", MCP ×2, eval talks (Torres, Thakur, Schoelkopf, Shankar), Jerry Liu ×2, agent memory, Revolut self-hosting | `ai-c-*` |
| Tokenizer, evals on Lenny's Podcast, systematically improving RAG, Clavié retrieval, Bischof experiments, error analysis (NurtureBoss), harnesses, agents hiding mistakes, Uber closed-loop evals, lethal trifecta, Factory multi-agent | `ai-d-*` |
