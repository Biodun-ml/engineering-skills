# ai-1 — cross-cutting principles

1. **Context is working memory; weights are vague recall.** Put the source text, docs and facts in the prompt instead of relying on what the model "knows". Rare facts are where recall fails and hallucination starts. (Karpathy, Deep Dive; Berryman, "LLMs are not psychic")
2. **Models need tokens to think.** Each token gets a small, fixed amount of compute. Ask for reasoning or evidence *before* the answer field, and never train or few-shot "answer first". Optional CoT fields in a schema let you trade quality against latency. (Karpathy, Deep Dive; Berryman, CoT; Liu, CoT-in-component)
3. **Send arithmetic, counting, string ops and lookups to code or tools, not to the model's head.** Tokenization makes character-level work unreliable, and the "Swiss cheese" failures are random. (Karpathy, Deep Dive)
4. **Most production LLM work is language → typed structured data.** Define the output as a typed schema (Pydantic) whose docstrings are the prompt. Validate in code, retry with the errors, and always include an explicit "unknown/not found" escape hatch. (Liu, Pydantic; Horthy, Factor 1)
5. **Make verification structural, not hoped for.** Require evidence quotes that must literally appear in the source; validate entities against a database; teach or allow "I don't know". (Liu, citation validators; Karpathy, Llama-3 factuality probing)
6. **Agents are just software.** Tool calls are JSON plus a switch statement. Own the prompts, the context window, the control flow and the state. Use a mostly deterministic DAG with small 3–10-step micro-agents, pausable/resumable through persisted state. (Horthy, 12-Factor Agents; Liu, plans-as-DAGs)
7. **Context engineering is the main lever.** Gather, rank, trim, then assemble into a familiar document shape. Fewer, denser tokens beat stuffing. Compact errors and drop stale history. Have a fallback budget policy. (Berryman, Copilot pipeline; Horthy, Factors 3/9)
8. **Restructure the task instead of waiting for a better model.** Split long documents (split → map → reduce, with "gather" context per chunk), split multi-field prompts, fuse easy steps, replace trivial steps with code, and push cheap filters (keywords/embeddings) down first. (Shankar, DocETL rewrite directives)
9. **Cascade for cost.** Send each item through cheap proxies first: a smaller model, a smaller slice, or a simpler correlated question. Escalate only low-confidence cases, with thresholds calibrated on a labeled sample against the expensive "oracle" (~86% cost cut at 90% target accuracy). (Shankar, task cascades; Berryman, logprobs)
10. **Evals are bespoke to the product.** Benchmarks are SAT scores. Generic hallucination/toxicity/1–5 scores aren't actionable. Measure the concrete outcome ("was the order actually cancelled?"). (Husain & Shankar, Common Mistakes; Husain, 15-min Concepts)
11. **Error analysis first.** Instrument full traces, read and annotate about 30+ until you hit saturation, cluster the notes into a failure taxonomy (an LLM can help), count, and fix the biggest bucket. Expect your criteria to drift while you grade. (Husain; Shankar, EvalGen/DocWrangler)
12. **Use LLM judges as narrow binary classifiers, one per observed failure mode,** with a few pass/fail examples. Validate against human labels on a train/dev/test split, tracking missed failures vs false alarms. No ungrounded Likert scales. (Husain & Shankar)
13. **Small offline sets are regression tests, not performance estimates.** Six test examples can't measure accuracy, and prompt optimizers overfit them. Monitor online. (Husain & Shankar, Common Mistakes)
14. **Domain experts own prompts and labels.** Build a custom, vibe-coded annotation and prompt-admin UI that renders outputs natively next to their sources. Outsourcing this outsources your moat. (Husain & Shankar; Shankar, three gulfs)
15. **Build partial-autonomy products, not demo agents.** The AI generates and the human verifies, so make verification fast (GUIs, diffs, side-by-side sources), keep changes small, add an autonomy slider, and use auditable intermediate artifacts to keep the AI on a leash. (Karpathy, Software 3.0)
16. **Pick the paradigm per sub-task:** code (1.0), a trained small model (2.0), or a prompt (3.0). Once evals exist, test cheaper and smaller models against a fixed quality bar, which is often the easiest win. (Karpathy, Software 3.0; Husain, 15-min Concepts)
17. **Graphs help when relationships already exist.** Vector search finds the entry node, traversal adds context, structure ranks. Don't LLM-extract a graph unless a concrete query needs one. (Eifrem, GraphRAG; Shankar Q&A)
18. **RL-trained "thinking" models are for verifiable hard reasoning.** Plain models are fine (and faster) for knowledge and simple tasks. Reward-model-based optimization gets gamed if pushed too far, and so do eval-driven auto-optimizers without broad evals. (Karpathy, Deep Dive; Husain, hill-climbing caveat)
19. **Software fundamentals scale AI coding.** Agree on a shared design concept first ("grill me"), keep a ubiquitous-language glossary, use small TDD steps, and build deep modules with owned interfaces and delegated implementations to cut review fatigue. (Pocock)
20. **Make your own systems legible to agents:** markdown docs, llms.txt, API/curl equivalents of "click here", MCP. (Karpathy, Software 3.0)

---

## Andrej Karpathy — Deep Dive into LLMs like ChatGPT (https://www.youtube.com/watch?v=7xTGNNLPyMI)
**An LLM is a lossy, stochastic next-token simulator: pretraining = knowledge, SFT = persona of a human labeler, RL = discovered problem-solving strategies; design around its fixed per-token compute, token-level vision and "Swiss cheese" reliability.**

**Mental models / first principles**
- Three training stages, like a textbook: pretraining = reading exposition (knowledge base), SFT = studying worked examples (imitate experts), RL = doing practice problems where only the final answer is known (discover your own method).
- Base model = "internet document simulator". Its parameters are a lossy zip of the web: frequent facts are recalled well, rare facts are vague and probabilistic. It is not an assistant; it just continues token sequences.
- An SFT assistant's answer is best thought of as an instant statistical simulation of a skilled human labeler following the company's labeling instructions (often hundreds of pages, written to be "helpful, truthful, harmless"). Thinking (RL) models are something more: emergent strategies not copied from humans.
- Parameters = vague long-term recollection; context window = working memory. Anything you place in context is directly accessible and far more reliable than recall.
- The model has no persistent self; "who are you" answers are either hardcoded via SFT examples (OLMo used ~240 identity conversations) or a system prompt, otherwise it hallucinates (often claims to be ChatGPT because the internet is full of ChatGPT text).

**Mechanisms**
- Data: Common Crawl (2.7B pages by 2024) → URL blocklists → HTML text extraction → language filter (FineWeb keeps pages >65% English) → dedup → PII removal. FineWeb ≈ 44 TB ≈ 15T tokens.
- Tokenization: bytes then Byte Pair Encoding merges frequent pairs to trade sequence length for vocabulary size; GPT-4 uses ~100,277 tokens. Tokens are case- and whitespace-sensitive ("hello world" vs "Hello World" tokenize differently).
- Training: sample windows (e.g. up to 8k tokens), predict next token, nudge parameters so the true next token's probability rises; watch the loss fall. GPT-2 (2019): 1.6B params, 1,024 ctx, 100B tokens, ~$40k then; Karpathy reproduced it for ~$600 in a day on an 8×H100 node (~$3/GPU-hour), likely ~$100 possible. Llama 3: 405B params, 15T tokens.
- Inference: sample from the probability distribution token by token, so outputs are remixes, not copies, and differ each run. Highly upsampled sources (e.g. Wikipedia seen ~10x) can be regurgitated verbatim.
- Base models can still be used with few-shot prompts (in-context learning), e.g. 10 English→Korean pairs then "teacher:"; or a fake web-page conversation to make a base model act as an assistant.
- Chat format: conversations become 1-D token streams with special tokens (e.g. `<|im_start|>user<|im_sep|>...<|im_end|>`); the server appends `assistant` start and samples. Post-training is ~hours vs pretraining's months; same algorithm, different data.
- Modern SFT data is mostly synthetic (LLM-drafted, human-edited), millions of conversations (e.g. UltraChat).
- Hallucination mitigation 1 (Meta Llama 3 "factuality"): take a doc paragraph, have an LLM generate factual Q&A, ask the target model several times, use an LLM judge to compare; where it is consistently wrong, add training examples whose correct answer is "I don't know". This wires the model's internal uncertainty signal to an explicit refusal.
- Mitigation 2: tools. The model emits special tokens like `<search_start>query<search_end>`; the inference harness pauses, runs the search, pastes results into context, resumes. A few thousand demonstration examples teach tool use.
- RL: for each prompt sample many solutions (thousands), score against the known answer, train on the good ones. DeepSeek R1 showed response length grows during RL as the model learns to re-check, backtrack, try other framings ("wait, let me re-evaluate") — chain-of-thought emerges without being hand-written.
- RLHF: humans rank ~5 outputs per prompt; a separate reward model (transformer outputting a 0–1 score) learns to match the rankings; RL runs against this simulated human. Ranking is easier than writing (discriminator–generator gap).

**Decisions & trade-offs**
- Thinking/RL models for hard math/code/reasoning; plain SFT models for knowledge and simple questions (Karpathy: ~80–90% of his use is a non-thinking model; thinking is overkill and slower for factual queries).
- Verifiable domains (exact answers) allow RL to run indefinitely; RLHF against a learned reward model must be stopped after a few hundred steps because RL will find adversarial inputs (e.g. "the the the" scored as a top joke). "RLHF is not real RL" — it's a small fine-tune.
- Smaller local models: use distilled variants at lower precision (quantized) in LM Studio; base models are rarely hosted.
- Leaderboards (LM Arena) are a first pass only and can be gamed; test models on your own task.

**Rules of thumb**
- Models need tokens to think: each token gets a fixed, small amount of compute (~100 layers). Do not force the answer into the first token; ask for intermediate steps before the final answer. Training/few-shot examples that state the answer first teach the model to guess.
- For arithmetic, counting, spelling, character indexing: tell the model to use code. Copying a string into Python is easy for it; counting tokens-in-its-head is not.
- Paste the source material into the prompt instead of relying on recall ("summarize chapter 1, attached below" beats "summarize chapter 1").
- Treat the model as a tool: use for drafts and inspiration, verify outputs, own the result.

**Failure modes & gotchas**
- Confident hallucination on unknown entities, because SFT data always answers "who is X" confidently.
- Swiss-cheese capability: solves olympiad problems yet says 9.11 > 9.9 (interpretability work found Bible-verse features firing). Character-level tasks fail because the model sees tokens, not letters ("ubiquitous" = 3 tokens; strawberry r's).
- Reward-model gaming; regurgitation of memorized text; random sampling means non-reproducible answers.
- Future pressure points: context windows are finite and precious; no test-time learning yet; agents will need human supervision ("human-to-agent ratio").

**Relevance to Sam**
- Trip-extraction pipeline: put the full article text in context and ask the model to extract only from it (working memory beats recall); never ask it to "know" place facts — ground them via GeoNames/supplier lookup tools. Expect confident fabrication when a place or tour isn't in the article; build an explicit "not found / unknown" option into the output schema.
- Have the LLM reason/quote evidence before emitting the final label or JSON field (tokens to think), or use a separate reasoning field.
- Counting, distance, price math, string matching on place names: do in code, not in the model.
- Location classification at scale: small/local distilled quantized models are viable for simple, frequent patterns; rare locations are where recall fails — feed context (country, admin region, feature code) rather than relying on model memory.
- Sampling is stochastic: use low temperature, and for evaluation run several samples to see variance; self-consistency (ask 3x, compare) is a cheap uncertainty probe, the same trick Meta used.

## Shreya Shankar (Hamel Husain channel) — How to Process Documents at Scale with LLMs (https://www.youtube.com/watch?v=t6r4U0SlnPc)
**Treat LLM document processing as a query-optimization problem: compose semantic map/filter/reduce operators, then rewrite the pipeline (decompose, cascade, push down, replace with code) to hit a target accuracy at a fraction of the cost, and steer it by looking at real outputs.**

**Mental models / first principles**
- "Semantic data processing": classic dataflow operators (map, filter, reduce/group-by, join) whose logic is a natural-language prompt plus an output schema. Now shipped in Databricks, BigQuery AI SQL, Snowflake Cortex, DuckDB. DocETL is her open-source system for this over text.
- Two core problems: (1) systems — make pipelines accurate and cheap at scale; (2) steerability — users can't specify fuzzy tasks precisely up front.
- "Optimize" means three axes now: accuracy, latency and dollar cost. There is no single optimal plan, only a Pareto frontier; show users several plans along the cost/accuracy curve.
- Don't wait for a better model: even frontier models fail when an operator is poorly scoped (e.g. "extract every implicit-bias statement" from a 300-page transcript → poor recall). And if only expensive models can do the op, every plan is expensive. The fix is restructuring the task.

**Mechanisms**
- Operators: map (1→1, adds schema fields), filter (yes/no, drops rows), reduce (group by key, one output per group). Datasets are lists of JSON dicts.
- Rewrite directives (13 in the first paper, 30+ now), instantiated by an LLM agent that writes the new prompts/schemas/chunk sizes for the specific data:
  - Data decomposition: split → map per chunk → reduce (e.g. ~1,500-word chunks) to raise recall on long docs.
  - Task decomposition: one map per sub-aspect of a long, detailed prompt, then a unify step; also suggested when one of several fields in a single call is inaccurate.
  - Fusion: merge two easy ops into one LLM pass to cut cost.
  - Code replacement: agent-synthesized Python for trivial steps (concatenate lists, sums) — zero LLM cost.
  - Projection pushdown: a cheap keyword or embedding filter finds the relevant parts of a document first, so every downstream LLM call reads less.
- New operators: **gather** (give each chunk surrounding context: previous chunk, running summary, doc metadata/table of contents) and **resolve** (first-class entity resolution before grouping, because LLM-extracted keys come out inconsistently).
- Task cascades (generalized model cascades): route each document through cheap proxies first — a cheaper model, a smaller slice of the document, or a simpler correlated question ("is any lower court mentioned at all?") — and accept the proxy answer when confident; otherwise escalate to the expensive "oracle" plan. Confidence from log-probabilities; thresholds chosen on a sample by comparing proxy vs oracle labels to meet a target accuracy. Result: ~86% average cost reduction while staying within a 90% target accuracy. Optimal cascade is NP-hard → greedy construction.
- Global plan search (MCTS/UCB-inspired): nodes are full pipelines, edges are rewrites; pick promising Pareto-frontier pipelines, have an agent rewrite, run on samples, update estimates, repeat until budget. Local optimization of one operator is not enough because other operators can compensate. Public-defender workload: GPT-5 baseline F1 47% → 1.5x F1 with 95% lower cost. Beat Lotus, Palimpzest/Abacus and a "GPT-5 agent with the docs" baseline.

**Decisions & trade-offs**
- Accuracy estimation is the expensive part: you usually must run candidate plans on labeled samples (they used ~40 docs). Cascades are attractive because accuracy is bounded by construction.
- LLM judges are a fallback when you have no labels; she prefers user labels.
- Knowledge graphs/graph DBs: usually make life harder. Ask what you'll do with the graph; often you only needed extracted entities, summaries or a group-by on an attribute.
- Multi-hop across a long doc: split-map-reduce plus gather inverts the problem — scan every chunk, aggregate at the end — rather than retrieving the right chunks.

**Rules of thumb (steering: the "three gulfs" from DocWrangler)**
- Gulf of comprehension (what's in my data?): read outputs next to source docs, write notes (open coding). Users routinely accumulate 25–30 notes and discover requirements they couldn't have predicted (e.g. "also extract dosages", "exclude over-the-counter drugs").
- Gulf of specification: turn those notes into prompt edits (AI-assisted diffs).
- Gulf of generalization: works on 100 docs, breaks at scale; run an LLM judge on a sample after each run and suggest decompositions when a field is weak.
- Users invent throwaway exploration pipelines ("summarize these", "extract key ideas") and add boolean/numeric fields purely to get histograms for quick sanity checks — copy this.
- Criteria drift (EvalGen, "Who Validates the Validators?"): people change their evaluation criteria while grading (e.g. "no hashtags as entities… unless it's a notable entity"). Label iteratively until criteria stabilize; don't freeze a rubric before seeing outputs.

**Failure modes**
- One giant prompt asking for many fields from a huge doc → missed items.
- Grouping on LLM-extracted values without entity resolution → fragmented groups.
- Chunks lacking context ("who is 'he'?").
- Optimizing each step in isolation.

**Relevance to Sam**
- Blog → bookable trip pipeline maps directly: filter (is this a travel article with bookable intent?) → map (extract places/activities/dates) → resolve (normalize place names against GeoNames/OSM, dedupe "NYC" vs "New York City") → reduce (assemble trip per article). Add gather context to chunks of long posts (title, destination, previous section).
- Cheap gate first: keyword/regex or small-model "does this mention a destination/hotel/tour?" before any expensive extraction (projection pushdown + cascade).
- Million-row location classification: a task cascade is the template — rules/code first, small local model with logprob confidence second, frontier model only for the low-confidence tail; calibrate thresholds on a labeled sample of a few hundred rows against the big-model "oracle".
- Split multi-field prompts when one field is weak; replace any aggregation/concatenation step with code.
- Read 30+ outputs next to sources before writing the final prompt; expect your own criteria to move.

## John Berryman (Hamel Husain channel) — Prompt Engineering Workshop (https://www.youtube.com/watch?v=htBTho6oEJA)
**An LLM app is a transformation layer: convert the user's problem into a document the model has "seen before", let it complete, then convert the completion back into action — and the hard engineering is gathering, ranking, trimming and assembling context to fit.**

(Speaker: ex-GitHub Copilot engineer, co-author of an O'Reilly prompt-engineering book. Talk dated mid-2024; model specifics are from that era.)

**Mental models / first principles**
- The model only predicts the next token in a document. Every prompting technique works by making the desired output the most likely continuation.
- "LLMs are dumb mechanical humans": (1) they understand familiar language/constructs best; (2) they get distracted — attention is finite, so stuffing the prompt hurts and very long intermediate context can make the model forget the original request; (3) they are not psychic — if the info isn't in training or the prompt, they can't know it (private docs, recent events, API data like travel availability must be injected); (4) prime directive: if a human can't make sense of the prompt, the model can't either.

**Mechanisms / techniques**
- Few-shot: a pattern of handcrafted examples ending in the open slot. Risk: examples "bleed" into answers and bias them; they cost context.
- Chain of thought: few-shot examples that show the reasoning; then "let's think step by step" (zero-shot CoT) does it without examples, avoiding bleed and saving space. Works because the model has no hidden monologue; written reasoning is its scratchpad.
- Document mimicry (most important): frame the prompt as a familiar document type — a transcript with a header ("between an award-winning IT support rep and a customer"), markdown headings for roles, code comments. The framing conditions tone and role.
- Copilot completions prompt pipeline: collect context (current file, open tabs, symbol definitions, file path) → rank (file path first — tiny and informative; then current doc; then neighbouring tabs; symbols weren't statistically significant so were shelved) → trim (drop tabs first; cut the top of the current file, never the text right before the cursor) → assemble as a plausible file (path comment at top, tab snippets as language-appropriate comments with source labels, then prefix, plus suffix).
- Chat (ChatML): `<|im_start|>role ... <|im_end|>` special tokens users can't type, so role injection is hard; system message is specifically trained to be obeyed; assistant stops on its own.
- Tool calling: functions with names, descriptions, typed parameters. Loop: model emits a call → your app executes it → append a `tool` message with the result (or an error, which lets the model recover) → call the model again until it answers the user. Parallel calls allowed. Example: thermostat "make it 2° warmer" = get_temperature then set_temperature; "put it back" works because prior messages carry state.
- Under the hood (OpenAI, as reverse-engineered): function definitions are rendered as a TypeScript-like `namespace functions` with comments for descriptions; nested object descriptions and JSON-Schema constraints like min/max are stripped. Every generated token is effectively a classifier: call a tool or not → which tool → each argument.
- Copilot Chat context budget: must-haves = system message, tool definitions you'll actually use, latest user message; nice-to-haves = prior tool calls/results (trim), references attached to messages (shrink/drop least relevant), history (fill newest-first until budget then truncate). Fallback: drop tool definitions, keep system + user message, so the model can at least reply gracefully rather than a 500.

**Decisions & trade-offs**
- Temperature ("blood alcohol content"): 0 = deterministic and a bit dull; Copilot Chat used 0.7; 1.0 = raw distribution; 1.5+ = gibberish.
- `n` completions: use with temperature turned up to get many samples (votes) for evaluation or post-processing; n with temp 0 is pointless.
- How many few-shot examples: inspect logprobs of the expected tokens with 1, 3, 5... examples; when they plateau, more examples add nothing.
- Few-shot placement in chat: system message is fine, but examples may need to be near the bottom of long conversations; can be put in fake user/assistant turns, carefully.
- Frameworks (LangChain, DSPy) can hide the prompt and remove knobs; spend real time talking directly to the model to build intuition, then use tools like DSPy for things like example selection.

**Rules of thumb**
- For structured extraction use function calling / a fake "save to database" function with a shallow schema; never paste your full nested API spec as a tool definition. If it still fails, break it down — at the extreme, one question per call.
- Name and describe tools and arguments plainly.
- Code completion: write a comment describing what the next lines do ending with a colon; the model mirrors your code style (sloppy in → sloppy out).
- ReAct (tools in a loop with an "answer" tool) and Reflexion (run the output through a checker such as unit tests, feed errors back, retry a few times) are the other techniques worth knowing.

**Failure modes**
- Overstuffed RAG context distracts; few-shot bleed; deep nested schemas where descriptions get stripped; dropping the user's message when context overflows.

**Relevance to Sam**
- Trip-extraction prompt: frame it as a familiar document (e.g. markdown "Travel article" → "Extracted itinerary" sections) or a `save_trip(...)` tool with a flat schema; keep schema shallow and describe each field.
- Write an explicit context-budget policy for long blog posts: must-have = instructions + schema + article core; trim boilerplate, comments, nav; truncate in a known order.
- Supplier APIs (Viator, RateHawk) as tools: return errors to the model as tool messages so it can retry; don't expose raw supplier schemas — wrap them in small, well-named functions.
- For classification of millions of rows: temp 0 for consistency; use logprobs of the label token as a confidence score (feeds a cascade); for eval/self-consistency, sample n>1 at higher temperature.
- Add a checker loop (Reflexion style): validate extracted places against GeoNames/supplier inventory and feed mismatches back for a retry.

## Andrej Karpathy (Y Combinator) — Software Is Changing (Again) (https://www.youtube.com/watch?v=LCEmiRjPEtQ)
**Prompts are a new kind of program (Software 3.0) running on a fallible "people spirit" OS; build partial-autonomy products with an autonomy slider and a fast human generate–verify loop, keep the AI on a leash, and make your own systems legible to agents.**

**Mental models / first principles**
- Software 1.0 = code; 2.0 = neural-net weights (you curate data, run an optimizer; Hugging Face is its GitHub); 3.0 = prompts in English programming an LLM. Same task (e.g. sentiment) can be done in any of the three — be fluent in all and choose deliberately. At Tesla, 2.0 "ate" the C++ stack (e.g. multi-camera/time fusion moved into the net); expect 3.0 to eat stacks similarly.
- LLMs as utilities (capex to train, opex metered per million tokens, demand for uptime/latency/consistent quality; OpenRouter is the "transfer switch"; outages are "intelligence brownouts"), as fabs (huge capex, centralizing R&D), but best as operating systems: LLM = CPU, context window = RAM, orchestrating tools and memory. Closed providers ≈ Windows/macOS; open weights (Llama ecosystem) ≈ possible Linux. Apps (Cursor) run on any "OS" via a dropdown.
- We're in the 1960s of this computer: expensive, centralized, time-shared, batch; personal LLM computing not yet economical (though Mac minis suit memory-bound batch-1 inference). Chat is the terminal; a general GUI hasn't been invented.
- Diffusion is flipped: consumers adopted before corporations and governments.
- LLM psychology: stochastic simulations of people; encyclopedic memory (Rain Man) but hallucination, weak self-knowledge, jagged intelligence (9.11 > 9.9, r's in strawberry), anterograde amnesia — no consolidation of learning, so you must program working memory (context) directly (Memento, 50 First Dates). Gullible: prompt injection, data leakage.

**Mechanisms — anatomy of a good LLM app (Cursor, Perplexity)**
1. The app does context management for the user.
2. Orchestrates multiple model calls (embeddings over files, chat model, diff-apply model).
3. Application-specific GUI for auditing: red/green diffs, accept/reject shortcuts, cited sources. Visual review uses the brain's "vision GPU"; reading text is effortful.
4. Autonomy slider: Cursor tab-complete → Cmd-K (selection) → Cmd-L (file) → Cmd-I agent (whole repo). Perplexity: search → research → deep research.

**Decisions & trade-offs**
- Iron Man suit (augmentation) over Iron Man robot (full agent) for now; build partial-autonomy products with a path to slide autonomy up over time. "2025 is the year of agents" is wrong framing — it's the decade of agents. Waymo gave him a perfect zero-intervention drive in 2013 and autonomy still isn't fully solved; demos ≠ products.
- Human is the verification bottleneck: a 10,000-line diff is useless. Two levers: make verification faster (GUIs, visual representations) and keep generation small (leash).
- Vague prompts → failed verification → spinning; spend more effort on concrete prompts to raise first-pass success.
- Use auditable intermediate artifacts to constrain the AI: e.g. for education, one app creates a course (reviewed), another serves it; the AI stays within the syllabus instead of "getting lost in the woods".

**Rules of thumb**
- Work in small, incremental, concrete chunks; spin the generate–verify loop fast.
- Ask of every product: can an LLM see everything a human sees? act everywhere a human acts? can a human supervise? What is the "diff" for your domain?
- Build for agents as a third consumer (after humans via GUI and programs via API): `llms.txt` markdown describing your domain; docs in markdown (Vercel, Stripe); replace "click X" instructions with curl/API equivalents; MCP servers; URL-swap ingestion tools (gitingest, DeepWiki) that flatten repos into LLM-readable text. Agents can click, but meeting them halfway is cheaper and more reliable.
- Vibe coding is great for one-off custom tools, but the hard part of shipping (MenuGen) was auth, payments, domains, deployment — DevOps clicking in browsers, a week vs. a few hours for the demo.

**Failure modes**
- Over-eager agents producing huge unreviewable changes; trusting demos; HTML-scraping instead of LLM-friendly sources; forgetting the model doesn't learn between sessions.

**Relevance to Sam**
- Drive Button / trip builder: design it as a partial-autonomy product. Autonomy slider = (a) suggest bookable entities for a human editor to accept/reject, (b) auto-insert high-confidence buttons, (c) fully automated publishing once evals justify it. Build a review GUI that shows source sentence ↔ extracted entity ↔ matched supplier product side by side (fast verification).
- Use an auditable intermediate artifact (a structured "trip draft" JSON) between extraction and booking, rather than one end-to-end agent.
- Decide per sub-task between 1.0/2.0/3.0: string normalization and geo-joins in code; small trained/fine-tuned classifier for high-volume location typing; LLM prompts for fuzzy long-tail.
- Keep providers swappable (OpenRouter-style abstraction) for outages and cost; plan for brownouts with fallbacks.
- Make Trameter's own data/APIs agent-legible (markdown docs, llms.txt, MCP) — useful for both internal agents and partners.

## Hamel Husain & Shreya Shankar — LLM Evals: Common Mistakes (https://www.youtube.com/watch?v=GL0XhAj5LPE)
**Application evals are bespoke, built from your own labeled traces via error analysis; generic metrics, outsourced labeling, unvalidated Likert judges and tiny holdout sets are the common traps.**

**Mental models / first principles**
- Foundation-model benchmarks are the SAT/GMAT of LLMs — interesting, but you hire on job-specific tasks. Application evals are the job tasks, and they're usually "boring" checks (summary within length limit; each of the 3 instructions in the prompt followed on every turn) that make or break the business.
- Off-the-shelf scores (hallucination, toxicity, conciseness) are at best a starting sanity check; relied on, they mislead.
- Prompts and labeled data are your moat. Delegating prompt-writing and labeling to developers (often offshore, far from the business) silently outsources taste and business judgment.
- You cannot trust any judge (LLM or otherwise) without checking it against labels. There is no way around labeling some data.

**Mechanisms**
- Integrated prompting environment: an admin mode in the product where a domain expert can edit the prompt and see what the user sees.
- Make traces accessible to domain experts; have ~3 experts label/verify the same traces and look at disagreement — disagreement usually reveals an ambiguous spec. "You can't send any ambiguity to an LLM."
- Build your own annotation app (vibe-codeable in ~10 minutes): render outputs in native form (an email as an email, a chart as a chart, markdown rendered, images shown), domain-specific filters, collapsible parts, fast feedback from thumbs up/down to free-text failure notes. Log everything to an observability tool and build the labeling UI on top of it (pull, annotate, push back). Label a sample, not everything. Target ~90–95% label accuracy; outsourced labeling may give 50–70%.
- Error analysis (social scientists call it "coding"): read traces, take notes on failures, aggregate notes into categories — LLMs can now cluster your notes into top failure modes and assign traces. Stop at theoretical saturation, often after ~30 traces; even chatbots rarely have more than ~10 query types (lookup, general Q&A, navigation…).
- LLM-as-judge done right: one judge per failure mode actually observed (not brainstormed a priori — that's infinite); narrow scope ("check only tone", "check tool used correctly"), not "is this trace correct?" (an intern given that instruction wouldn't beat the generator). Binary pass/fail with 2–3 examples each of pass and fail; segment by context (e.g. tone judge per recipient type: friend / family / exec). Measure agreement with your own labels empirically.

**Decisions & trade-offs**
- Binary over Likert (1–5): ungrounded scales mean nothing to the judge or to humans, graded in isolation they drift; grounding 5 levels needs examples for each; the business ultimately makes a go/no-go call anyway (seen in prod: "score ≥ 3 → alert", which is just a badly built binary judge). Likert is acceptable only when you train/ground a judge with examples per level (as in the "Judging LLM-as-a-Judge"/Chatbot Arena paper).
- Offline eval sets with 25–30 labeled traces split 80/20 → a 6-example test set; one example = ~17 points of "accuracy". Prompt-optimization tools hill-climbing on such sets overfit easily (paraphrases leak). Don't report offline accuracy as pipeline performance; monitor online.
- Hand-curated offline sets are robustness/regression sets (like unit tests in CI: designed to pass 100%, guard regressions), deliberately loaded with hard cases — not random samples of production, so they can't estimate production error. Keep both: a regression set and a production-sampled estimate.

**Rules of thumb**
- Domain experts write prompts and labels; developers make that easy.
- Look at data first; 30 traces yields actionable fixes immediately.
- One binary judge per real failure mode, with few-shot pass/fail examples, validated for agreement.
- Online monitoring is the ground truth for performance.

**Relevance to Sam**
- Trip pipeline: build a tiny labeling app that shows the blog article (rendered) next to the extracted trip and the matched bookable products; label 30–100 articles; write failure notes ("hallucinated hotel", "wrong city disambiguation", "non-bookable activity", "missed date range"); cluster with an LLM; then build one binary judge per top failure mode.
- Get a travel/content person (not only engineers) to own the gate and extraction prompts via an admin prompt editor.
- Location classification: replace any 1–5 "confidence/quality" scoring with binary judgments per category check; keep a curated hard-case regression set (ambiguous names, ferry terminals vs. ports, transliterations) separate from a random-sample accuracy estimate over the millions of rows.
- Beware reporting "92% accuracy" from 25 examples; size eval sets so a single error is a small change, or report counts.

## Dex Horthy (AI Engineer) — 12-Factor Agents: Patterns of Reliable LLM Applications (https://www.youtube.com/watch?v=8kMaTybvDUw)
**Production "agents" are mostly ordinary software with small LLM steps: the LLM turns language into JSON, and you own the prompts, the context window, the control flow and the state.**

**Mental models / first principles**
- Frameworks get you to 70–80% quality fast; past that you end up "seven layers deep in a call stack" reverse-engineering prompt construction, and often rewrite from scratch. The 12 factors are less anti-framework than a wish list for frameworks.
- Not every problem needs an agent: his first DevOps "run the Makefile" agent needed so much step-by-step prompt detail that a bash script would have taken 90 seconds.
- From 100+ builders: most production agents aren't very agentic; they apply small modular patterns to existing code, no greenfield rewrite, no AI background needed — like Heroku's 12-factor apps for cloud.
- LLMs are pure functions (tokens in → tokens out). The only lever on output quality without retraining is what tokens go in: "everything is context engineering" (prompt, memory, RAG, history).

**Mechanisms (the factors he covered)**
- F1 Natural language → structured output: the most magical capability is turning a sentence into JSON. Usable in your app today.
- F4 "Tool use is harmful" (as a mystical abstraction): a tool call is just JSON the model emits that deterministic code consumes via a switch statement, maybe feeding a result back.
- F8 Own your control flow: an agent = prompt (choose next step) + switch (execute JSON) + context builder + loop with exit conditions. Owning the loop lets you break, summarize, insert LLM-as-judge, pause for approval, etc. The naive "LLM decides next step until done" loop fails on longer workflows, mainly because the context grows; fewer, curated tokens always give tighter results (even if a 2M-token prompt technically returns something).
- F5/F6 Unify execution state and business state; launch/pause/resume via plain APIs: put the agent behind REST or MCP; when it calls a long-running tool, serialize the context window to a DB with a state ID; when the callback arrives, load state, append result, continue. The agent doesn't know time passed.
- F2 Own your prompts: prompt-generator abstractions give a good start, but past a quality bar you hand-write every token. Try many variants and evaluate.
- F3 Own your context window: you need not use the standard messages format; you can stringify the whole event history into one custom, dense user message (he shows XML-ish event traces). Optimize density and clarity of every token.
- F9 Compact errors into context: feeding a tool error back so the model retries is useful but can spiral. Once a valid call succeeds, clear pending errors; summarize instead of pasting stack traces.
- F7 Contact humans with tools: make the first output decision a natural-language intent (done / need clarification / escalate to manager / call tool) rather than a hidden binary between "tool call" and "message"; enables outer-loop agents that ask humans for approval.
- F11 Trigger from anywhere, meet users where they are: email, Slack, Discord, SMS rather than yet another chat tab.
- F10 Small, focused micro-agents: a mostly deterministic DAG with tiny agent loops of 3–10 steps. HumanLayer deploy bot: CI/CD is deterministic; after merge + tests pass, a micro-agent proposes deploy order, human says "backend first" (language → JSON next step), approves, deploys, then deterministic e2e tests against prod; failures hand off to a similar rollback agent.
- F12 Agent as stateless reducer (really a transducer): state lives outside; agent is a function of state.
- Prefer scaffold-and-own code (shadcn-style "create-12-factor-agent") over opaque wrappers.

**Decisions & trade-offs**
- Expect LLMs to take over bigger chunks over time (sprinkle into deterministic code, expand scope as models improve), but engineering skill still decides quality.
- NotebookLM lesson: pick tasks right at the edge of what the model can do reliably, then engineer reliability around it — that's where differentiated products come from.
- Tools should take away the non-AI hard parts (state, durability, human contact) so you can spend time on the AI hard parts (prompts, flow, tokens) — not the reverse.

**Failure modes**
- Long naive loops lose coherence; error-retry spirals; framework opacity; agents for problems a script solves.

**Relevance to Sam**
- The blog → trip pipeline should be a deterministic DAG (fetch → clean → LLM gate → LLM extraction to JSON → code: geo-match/supplier search → LLM micro-step only where judgment is needed → human review) rather than one autonomous agent.
- Supplier API calls (Viator/RateHawk availability, booking) are async/long-running: persist pipeline state by ID, resume on callback — no context held in memory.
- Build context per step deliberately: pass the extraction step only the article text plus a compact summary, not the whole history; when retrying a failed supplier lookup, give a one-line error summary.
- Human-in-the-loop via Slack/email for low-confidence trips or booking confirmations.
- Write prompts by hand in your own repo (versioned), not hidden inside a framework.

## Jason Liu (AI Engineer) — Pydantic Is All You Need (https://www.youtube.com/watch?v=yj-wSRJwrrc)
**Most LLM apps are "language → structured data" systems; define the output as typed Pydantic models so the schema, prompt, validation and behaviour live in one reviewable piece of code, and handle failures with validation + retry instead of prayer.**

(2023 talk; since then providers ship native structured outputs/strict JSON schema, but the design patterns remain current.)

**Mental models / first principles**
- ~90% of production LLM work is asking for JSON/structured output to feed existing systems and APIs you can't change — LLMs must be backwards-compatible with software. We build systems, not chatbots.
- String prompts → string outputs → regex/`json.loads` is "praying to the LLM gods": inconsistent keys (`user` vs `username`), missing fields, wrapper text ("here you go, here's some JSON") — and you only notice with good logging.
- Structured prompting: the Pydantic model *is* the prompt. Class docstrings and field descriptions are serialized into the JSON schema sent to the model, so code quality, prompt quality and data quality stay in sync. Good names, descriptions, docs become a prompt-engineering requirement. "Domain modeling more than prompt engineering."

**Mechanisms**
- Pydantic: type-hint-driven validation and coercion (string timestamp → datetime, list of strings → tuple of ints), IDE autocomplete/type checking, emits JSON Schema (the language of function calling). ~70M downloads/month at the time.
- Instructor: patch the client, pass `response_model=YourModel`, get back a validated instance instead of a dict. Marvin offered similar with more providers.
- Validators: plain-code field validators (e.g. lowercase check, char limit, DB lookup via HTTP) raise errors; `llm_validator("don't say mean things")` uses an LLM call as a validator. `max_retries`: collect all validation errors, append them to the conversation, ask the model to fix — "validation error handling and re-asking", not prompt chains.
- Escape hatch: model "MaybeUser" with `result: Optional[User]`, `error: bool`, `message: str` rather than checking for the string "I DON'T KNOW". Optional fields (e.g. role may be None).
- Reusable components: `TimeRange(start, end)` used for both work_time and leisure_time; add a `chain_of_thought: str` field inside a component when it parses poorly — toggle it on in testing, off in production to measure latency/quality trade-off.
- Arbitrary key/values: `List[Property(key, value)]`; validators enforce key consistency; add an index field and say "count to N" to reliably cap the number of items.
- Graph/structure outputs: entities with `id` and `friends: List[int]` produce a network; knowledge graph models mirroring the visualization library's API (nodes/edges) → ~40 lines end to end.
- RAG query understanding: `Search(title, query, before_date, type: Literal["email","video"])` with an `execute()` method; extract `List[Search]` to decompose a question into backend-specific searches with time filters; run concurrently (asyncio map). The system prompt can be trivial because the schema carries the instructions.
- Query planning in one call: `Query(id, question, dependencies: List[int])` → a DAG you execute with classical code (parallelize independent nodes) instead of an open-ended agent loop.
- Citation-grounded answers: `Fact(statement, substring_quotes: List[str])`; validator checks every quote actually appears in the source chunk and drops facts without one; answer-level validator keeps only grounded facts. Anti-hallucination as verification logic, not "please don't hallucinate".

**Decisions & trade-offs**
- Nested, typed models are far more maintainable than hand-written JSON schemas for POST bodies; still keep them as shallow as the task allows (see Berryman).
- CoT fields cost latency/tokens; make them optional per component.
- Let the LLM output data structures (plans, DAGs) that ordinary code executes — e.g. something you could hand to Airflow — rather than an agent loop "hoping it terminates".

**Rules of thumb**
- Every LLM call returns a typed object or raises.
- Put instructions in docstrings/field descriptions; keep the system prompt short.
- Validate with code first (cheap, deterministic), LLM validators second; retry with the error messages.
- Always include an explicit "not found / error" path in the schema.

**Relevance to Sam**
- Trip extraction schema: `Trip(destination: Place, items: List[TripItem])`, `TripItem(kind: Literal["hotel","tour","restaurant","transport","attraction"], name, city, country_code, evidence_quote, date_hint: Optional[...])`; validator: `evidence_quote` must be a substring of the article (kills hallucinated venues); country_code must be valid ISO; city must resolve in GeoNames (HTTP/DB validator) → re-ask on failure.
- LLM gate as `GateResult(is_travel_article: bool, bookable_intent: Literal[...], reason: str)` with the reason field before the boolean if you want CoT.
- Location classification: `Literal[...]` enum of allowed categories forces valid labels; `Optional` + `unknown` escape hatch avoids forced guesses on ambiguous rows; batch several rows per call as `List[RowLabel(row_id, label)]` with an id validator to ensure every input id is returned exactly once.
- Supplier search: decompose a trip into `List[SupplierQuery(provider, query, dates, location_id)]` and execute concurrently in code.

## Emil Eifrem (AI Engineer) — GraphRAG: The Marriage of Knowledge Graphs and RAG (https://www.youtube.com/watch?v=knDDGYHnnSI)
**GraphRAG = RAG whose retrieval path uses a knowledge graph: vector search finds entry nodes, graph traversal expands related context, graph structure ranks it; the payoff is accuracy and explainability, the cost is building the graph.**

(Speaker is Neo4j's CEO — a vendor talk; accuracy figures are cited from others' studies and should be weighed accordingly. Shreya Shankar, in the DocETL talk above, warns most people reach for graphs without needing them.)

**Mental models / first principles**
- Search history as analogy: keyword/BM25 engines (AltaVista era) → "too many results" → PageRank (a graph algorithm, eigenvector centrality) ranked the top 10 links → 2012 Knowledge Graph "things not strings" (entities and relationships, not just text and links) → now LLM + knowledge graph (e.g. Google's AI travel itineraries).
- Property graph: nodes and relationships, both carrying arbitrary key–value properties.
- Vector vs graph representations are complementary. Vectors are opaque to humans; the graph is explicit, deterministic, visual — you can see and debug it.

**Mechanisms**
- Core pattern: (1) embed the question, vector search to get initial nodes (treat it almost like a primary-key lookup into the graph); (2) traverse relationships to pull in related context (sibling products in the hierarchy, articles by a highly ranked author, etc.); (3) optionally rank the expanded set by graph structure (even PageRank) to top-K; (4) pass to the LLM. Example: Wi-Fi router support articles linked to product nodes in a product hierarchy and to the authoring engineer.
- Data types for graph construction: structured (Postgres/Snowflake) → easy mapping to a property graph; unstructured (PDF, web text) → hard, historically immature tooling, now LLM entity/relation extraction (Neo4j Knowledge Graph Builder: ingest PDFs/YouTube/Wikipedia, chunk, embed, extract entities, link chunks ↔ entities); mixed (structured records with long text fields) → where most enterprise use cases live.
- Two graph kinds mentioned but not explained: lexical graphs (documents/chunks structure) vs domain graphs (business entities).

**Claimed benefits**
- Accuracy: data.world study ~3x average accuracy over 43 questions with KG + vectors; LinkedIn paper ~75–77% improvement (also a good reference architecture); Microsoft GraphRAG: better answers and a new class of questions (global/summary questions) baseline RAG can't answer.
- Easier development once the graph exists: a fintech porting from a vector DB to Neo4j said related items were just relationship hops and visualizing the cache fixed bugs.
- Explainability, auditability, governance from the explicit structure.

**Decisions & trade-offs**
- The hump is creating the knowledge graph; worth it if you already have, or naturally have, entity relationships.
- Not a replacement for vector search — use both.

**Failure modes / caveats**
- Vendor framing; no discussion of cost of extraction, entity-resolution errors, or graph maintenance. Pair with Shankar's advice: first ask what you'll do with the graph; often grouping by an extracted attribute suffices.

**Relevance to Sam**
- Trameter's location data is already naturally a graph (GeoNames hierarchy: country → admin1 → admin2 → city → POI; transit terminals ↔ cities; supplier products ↔ locations). That's the structured case — cheap to build. Pattern for trip building: vector/name search to find the place mentioned in the article, then traverse to parent city/region and to nearby bookable products/terminals, rank by proximity/popularity, pass to the LLM.
- Don't LLM-extract a graph from blog text unless a concrete query needs it; extract entities and link them to existing GeoNames IDs instead (entity resolution against the master dataset).
- A graph view of article → places → products is a strong debugging/QA tool for the Drive Button.

## Hamel Husain — AI Eval Concepts Explained in 15 Minutes (https://www.youtube.com/watch?v=3rlYBCDK0K4)
**The eval loop: instrument traces → annotate failures → group and count them → sample smartly → build binary automated graders validated against human labels (train/dev/test) → use them in CI, monitoring and guardrails to actually improve the product (including moving to cheaper models).**

**Mental models / first principles**
- Foundation benchmarks (MMLU, Terminal-Bench, Humanity's Last Exam) are IQ tests, usually shown as cost-vs-performance Pareto charts. On your product-specific evals the Pareto frontier reshuffles — models low on general benchmarks can rise to the top for your task.
- A product eval measures a concrete outcome: "user asked to cancel → is the order actually cancelled?"
- Evals exist to make the product better, not for their own sake; they encode the ship/no-ship threshold.

**Mechanisms**
- Trace = full record of a session or workflow run: system prompt, all context, every LLM call, tool calls and results, sub-agents. Instrument code, store traces in a DB, and have a human-readable trace viewer. Traces serve three uses: find issues, test changes, debug failures.
- Annotation: write what went wrong in plain words (e.g. "cancel tool failed but assistant said order cancelled").
- Grouping: have an AI propose a failure taxonomy (false confirmations, wrong order, missed handoff), then refine by hand — e.g. split "false confirmation" into "tool failed" vs "tool never called".
- Analysis: start by counting failures per group; counts set priorities.
- Sampling traces — a spectrum: random (best for unknown unknowns, always keep some), clustering (structured exploration of themes), data analysis (stats like turn count, latency, number of tool calls → look at outliers). More exploration when new; more signal-guided later.
- Synthetic data (limited; not first resort): generate *inputs* only, run them through the real system, log traces, review. Use scenario dimensions (cancel before vs after shipping; unclear order; missing order number). Make inputs as messy as real users ("cancel the blue one"), not "cancel order #42". For retrieval: generate a plausible question from each document, then test whether retrieval returns that document (reverse generation).
- Grading options: code assertions (e.g. trace must not contain a certain user ID / tool call), a classifier (decision model), or LLM-as-judge used as a binary classifier. Binary pass/fail only.
- Validating a judge: human ground-truth labels (AI can propose, human accepts). Error types: missed failures (false negatives) and false alarms (false positives). Split labeled data: train (source of few-shot examples in the judge prompt), dev (~40%; iterate prompt to maximize human–judge agreement; hill-climb, AI can help), test (final untouched check; poor agreement here = overfit, redo).
- Deploy trusted evals: CI/CD, production monitoring (near-real-time error detection), and guardrails (in the request/response path, blocking bad outputs, usually safety).

**Decisions & trade-offs / when to build an eval**
- Evals are expensive to build and maintain; don't eval everything. Build one when: you've observed or can reproduce the problem (brainstormed worries are fine only if reproducible), and it's something you'll iterate on without an obvious fix — that's where eval signal pays.
- Automated hill-climbing (agent edits prompt, reruns evals): only with a comprehensive eval suite, else metric hacking; a human should review, simplify and harvest ideas from the agent's prompt rather than adopt it blindly; check on unseen cases.
- Model downsizing: set a minimum quality bar per important task, then test smaller/cheaper models on the same evals — frequently the easiest win once evals exist.

**Traps**
- Generic 1–5 or 0–100 metrics (what is 4.2 vs 4.5?) are not actionable; measure specific product failures.
- Synthetic inputs that are too clean.
- Overfitting the judge to the dev set.

**Relevance to Sam**
- Define a trace for the Drive Button: article id → cleaned text → gate prompt+output → extraction prompt+output → geo-match results → supplier queries/results → final buttons. Store per-run in DB; build a viewer.
- First failure taxonomy to expect: false positives at the gate, hallucinated places, wrong disambiguation (Paris, TX), non-bookable items, supplier mismatch, "button inserted but product unavailable". Count them before optimizing.
- Data-analysis sampling: outliers in number of extracted items, articles with 0 matches, long latency, many supplier retries.
- For location classification: train/dev/test split on a few hundred human labels to validate a cheap judge or a small model; then compare small local model vs. API model at a fixed quality bar — classic downsizing win given the budget.
- Synthetic: generate messy article snippets (multi-city trips, ambiguous names, non-English place names) to stress extraction.

## Matt Pocock (AI Engineer) — Software Fundamentals Matter More Than Ever (https://www.youtube.com/watch?v=v4F1gFy-hqg)
**"Code is cheap" is wrong — bad code is the most expensive it's ever been, because AI only performs well in codebases that are easy to change; use classic design practice (shared design concept, ubiquitous language, fast feedback/TDD, deep modules) to direct AI coding agents.**

**Mental models / first principles**
- Specs-to-code (edit the spec, regenerate, never read the code) degrades with each regeneration — "vibe coding by another name". It's software entropy (Pragmatic Programmer): changes made without regard to whole-system design make the codebase steadily worse.
- Ousterhout's definition: complexity is whatever makes a system hard to understand and modify. Good code = easy to change. AI amplifies a good codebase and drowns in a bad one.
- Role split: AI is the tactical "sergeant on the ground"; you are the strategist who owns design. Kent Beck: invest in the design of the system every day — specs-to-code divests from it.

**Failure modes and fixes**
1. "AI didn't do what I wanted" → no shared *design concept* (Brooks, The Design of Design): the invisible shared theory of what's being built; not a file. Fix: a "grill me" skill — "interview me relentlessly about every aspect of this plan until we reach shared understanding; walk each branch of the design tree, resolving dependencies between decisions one by one." It asks 40–100 questions; the transcript becomes a PRD or issues for an unattended agent. He prefers this to Claude Code's plan mode, which rushes to produce a plan artifact.
2. "AI is too verbose / talking past me" → no shared language. Fix: DDD *ubiquitous language* — a markdown glossary (tables of domain terms) used consistently in code, conversation and prompts; a skill scans the codebase to build it. Keeps planning and AI reasoning tighter and implementation aligned with the plan.
3. "It built the right thing but it doesn't work" → feedback loops: static types (TypeScript), browser access for front-end, automated tests. But AI writes huge batches and only then checks — "outrunning your headlights"; the rate of feedback is your speed limit. Fix: TDD (red → green → refactor) forces small steps.
4. Testing is hard (unit size, what to mock, which behaviours — all interdependent) → good codebases are easy to test. Use Ousterhout's *deep modules*: few modules with lots of functionality behind simple interfaces, vs shallow modules (little functionality, complex interfaces) that AI tends to generate and then can't navigate or understand. An "improve codebase architecture" skill finds related code and wraps it in deep modules; test at the interface.
5. "My brain can't keep up" (developer fatigue from reviewing AI output) → treat deep modules as grey boxes: design and own the interface, delegate the implementation, verify via boundary tests; review internals only for critical areas (e.g. finance). Make the module map part of the ubiquitous language and write PRDs that specify which module interfaces change.

**Rules of thumb**
- Align (grill) before generating; maintain a glossary; small TDD steps; design interfaces yourself; let AI fill implementations behind tested boundaries; keep investing in architecture.

**Relevance to Sam**
- With a 16GB laptop and large pipelines built mostly via AI, Sam's codebases (locations_transform, Drive Button, Tereka) benefit directly: wrap pipeline stages in deep modules with simple interfaces (e.g. `resolve_place(name, country_hint) -> PlaceMatch`, `extract_trip(article) -> Trip`, `search_supplier(query) -> List[Product]`), test at those interfaces, let the agent own internals.
- Keep a ubiquitous-language file for Trameter (trip, itinerary item, bookable product, supplier, location id, terminal, gate, match confidence) and feed it to coding agents and to the LLM prompts themselves.
- Use "grill me" before asking an agent to build a new pipeline stage; turn the Q&A into a short PRD.
- Matches his stated concern about user/dev fatigue: reduce review load by designing interfaces and boundary tests rather than reading every generated line.
