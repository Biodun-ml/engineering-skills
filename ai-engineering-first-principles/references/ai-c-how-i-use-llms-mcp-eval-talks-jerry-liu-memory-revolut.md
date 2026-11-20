# ai-3 — cross-cutting principles

1. **Context is the product.** An LLM is a lossy, dated compression of the internet; whatever must be correct (recent facts, prices, availability, exact numbers) has to be put into the context window via retrieval or tools, not recalled. Keep each call's context short and on-task — extra tokens distract and cost. (Karpathy — How I use LLMs; Alake — Agent Memory)
2. **Route exact work to code, not the model.** Arithmetic, counting, plotting and lookups belong in a code tool; models without one "do it in their head" and are close but wrong. Always read generated analysis code — it silently invents assumptions. (Karpathy)
3. **Escalate models, don't default to the biggest.** Start with a fast/cheap model, escalate to a reasoning or frontier model only for items that fail; self-hosted open weights can make the cheap tier nearly free at scale. (Karpathy; Abitbol — Revolut)
4. **Few-shot examples, delimited with XML-like tags, are the cheapest accuracy boost; save the prompt as a fixed template.** (Karpathy)
5. **Evals = scientific method: error analysis → measure prevalence → improve, then repeat.** Look at real traces yourself, name failure modes, fix the most prevalent first (~80/20). (Shankar — Automate Evals; Torres — Noob to Evals)
6. **Your taste is the differentiator; AI scales it but can't supply it.** Don't ask an agent to "evaluate my app"; have it build review UIs, taxonomise your free-text notes, and propagate your labels to other traces. (Shankar)
7. **Prefer code assertions over LLM judges wherever a deterministic signal exists; validate every judge against human labels before trusting it.** Judges are weakest exactly on the hard, subjective calls you most want them for. (Torres; Schoelkopf — Deep Dive on LLM Evaluation)
8. **Feed evaluators minimal, pre-extracted context and cache expensive upstream steps on a fixed dev set**, so eval iteration is fast and cheap; A/B every change (prompt, model, temperature, workflow) on the same dev set and read the per-eval table. (Torres)
9. **Loop over your data more than once, and analyse the whole user-facing output for prioritisation.** New failure modes appear only after you've seen many traces; per-component annotation is for building component evals. (Shankar; Torres)
10. **Set the accuracy bar per feature from worst-case harms.** Internal summaries need little; customer-facing outputs need strict evals and guardrails — brainstorm worst cases first. (Shankar)
11. **Implementation details swing results: prompt whitespace, templates, tokenizers.** Keep eval and production prompts byte-identical; trust evals you ran yourself on your own downstream task over vendor tables. (Schoelkopf)
12. **For closed label sets, constrain the output** (log-likelihood over choices, enum/structured decoding) to get guaranteed-valid labels plus a confidence score for routing. (Schoelkopf)
13. **Retrieval quality often matters as much as the LLM.** Same LLM went ~60%→~90% on a search benchmark just by swapping BM25 for a strong retriever; always pin and ablate the retriever, keep a tuned BM25 baseline, and measure with IR metrics (hit rate, MRR, NDCG, recall). (Thakur — Search Agents; Liu — Production RAG)
14. **Fix the data layer first: layout-aware parsing, sensible chunking, metadata filters inferred from the query, small-to-big retrieval** — before rerankers, agents or fine-tuning. More retrieved tokens ≠ better answers. (Liu — Production RAG; Liu — Knowledge Assistants)
15. **Measure agent efficiency, not just accuracy:** search/tool calls per task, repeated queries, novelty of retrieved docs. Long loops correlate with failure; cap turns and dedupe. (Thakur)
16. **Keep each agent's tool set small** (models handle ~tens to ~100 tools); beyond that use tool search / progressive discovery or specialist agents with 5–10 tools each — which also lets you use smaller, cheaper models. (Murag — MCP Workshop; Soria Parra — Future of MCP; Liu — Knowledge Assistants; Alake)
17. **Compose with code instead of chaining tool calls.** Give the model typed functions in a sandbox and let it write one script (programmatic tool calling) — fewer inference round trips, lower latency and tokens. (Soria Parra)
18. **Design tools for agents, don't mirror REST one-to-one.** Put retries, auth, rate limits and response shaping on the server side; mark read vs write tools; deterministic steps should be plain calls, not model-chosen tools. (Murag; Soria Parra)
19. **Synthetic data: generate with one model, verify with independent ones; prefer precision over volume.** Eyeball 50–100 samples per stage. (Thakur — ORBIT)
20. **Persist state and failures.** Store failures and resolved entities as memory/lookup tables and feed them back so pipelines stop repeating mistakes; pass prior-step outputs forward to avoid cross-call duplication. (Alake; Torres)

---

## Andrej Karpathy — How I use LLMs (https://www.youtube.com/watch?v=EWvNQjAaOHw)
**An LLM is a lossy, dated "zip file" of the internet with an assistant persona bolted on; everything useful beyond that comes from what you put in the context window (documents, tool outputs) and from picking the right model tier/mode.**

- Mental models / first principles:
  - Underneath the chat bubbles is one 1-D token sequence (user and model take turns appending; special tokens mark turn boundaries and "I'm done"). Example: a haiku exchange was 42 tokens total; GPT-4o vocabulary is ~200k tokens.
  - Pre-training = compress the internet into parameters (~1 TB file ≈ ~1T params; tens of millions of dollars, months) → knowledge is probabilistic, vague and frozen at a cutoff. Post-training = conversations written by labelers give the assistant persona/style. Default model has no calculator, no interpreter, no browser.
  - Recall quality tracks frequency on the internet: common, stable facts (caffeine in an Americano) are fine from memory; rare or recent facts are not.
  - Context window = working memory. Anything in it (search results, uploaded PDFs, tool output) is "directly accessible"; parametric knowledge is hazy recollection. Grounding = moving facts from parameters into the window.
  - Thinking models = an extra RL stage on math/code problems where the model discovers strategies (try, backtrack, revisit assumptions) that humans could not hand-label. You pay in latency (can be minutes) and tokens for accuracy on hard problems.
  - Multimodality: audio (spectrogram windows quantised to a ~100k vocab) and images (patch grid quantised to a codebook) become token streams; the transformer does not know which tokens are which modality — only encoders/decoders do. "Native" (omni) audio vs "fake" audio (separate speech-to-text / text-to-speech around a text model). Image generation in ChatGPT at the time was a caption sent to a separate image model.
- Mechanisms:
  - Tool use = model emits special tokens (e.g. a search call or code block); the app pauses sampling, runs the tool, pastes the result as tokens into the context, and resumes. Labs teach *when* to use tools by curated examples.
  - Search: query → fetch ~10 pages → thousands of tokens stuffed into context → answer with citations.
  - Deep research = search + thinking run for ~10+ minutes (one run: 5 min, 27 sources) producing a report with citations.
  - Python interpreter: big arithmetic should go through code; models without the tool (or unwilling to use it) "do it in their head" and are remarkably close but wrong.
  - Memory feature = a list of text strings prepended to every conversation. Custom GPTs / custom instructions = saved prompts.
- Decisions & trade-offs:
  - Model size: bigger = more world knowledge, better writing, fewer hallucinations, more cost. Free tiers may silently give you a smaller model. Check which model you are actually talking to.
  - Try the fast non-thinking model first; escalate to a reasoning model when the answer looks weak and the task is math/code/logic. Thinking adds nothing for travel ideation-type tasks.
  - Use search whenever the answer is recent, changes over time, or is niche/esoteric. Use deep research for product comparisons and literature surveys, but treat output as a first draft.
  - Professional coding: use an IDE agent with repo context (Cursor etc.) rather than pasting snippets into a chat UI.
- Rules of thumb:
  - Start a new chat when switching topics: long contexts both distract the model (lower accuracy) and cost slightly more per token.
  - Few-shot beats zero-shot: describe the task AND give several concrete input/output examples; delimit examples with XML-like tags.
  - For images with numbers (labels, lab reports), do it in two steps: first ask for a transcription into text/table, verify it, then ask questions on the text.
  - Ask several models the same question ("LLM council") for ideation and cross-checking.
  - Always read the generated analysis code: in one demo the model silently replaced an N/A with 0.1 and then reported 1.7T when the variable actually held ~20T.
- Failure modes & gotchas:
  - Hallucination persists even with citations/deep research (a "US LLM labs" table omitted xAI and included non-US/minor orgs).
  - Different apps/models have different tools wired in; a model without search will confidently use stale knowledge; without code it fabricates arithmetic.
  - PDF uploads are probably converted to text with images dropped; very long files can exceed limits.
  - Features differ between web and mobile, and between pricing tiers, and change fast. (note: specific products/tiers in the video are Feb-2025 snapshots and already outdated.)
- Relevance to Sam:
  - Travel-blog → trip pipeline: never rely on the model's memory for venue facts, opening times, prices or availability; ground with supplier API results or search in context, and keep each call's context tight (one article + task + schema), not a growing chat.
  - Millions of location rows: classification of common, stable facts (country, place type) can use a small/cheap model; push anything requiring exact numbers or lookups (coordinates, distances, counts) to code, not to the LLM.
  - Few-shot examples in XML tags are the cheapest accuracy boost for an extraction or gate prompt; save them as a fixed prompt template (the "custom GPT" idea in code form).
  - Escalation pattern: cheap fast model first, reasoning model only for rows/articles that fail validation.
  - Two-step "transcribe then reason" applies to any OCR/screenshot or scraped HTML input: extract to a verifiable intermediate form before acting on it.

## Mahesh Murag (Anthropic) — Building Agents with Model Context Protocol, Full Workshop (https://www.youtube.com/watch?v=kQmXtrmQ5Zg)
**MCP is a standard client/server layer (like LSP for IDEs) that turns the N×M "every app integrates every tool its own way" mess into N+M, and gives the model, the application and the user each their own control surface.**

- Mental models / first principles:
  - "Models are only as good as the context we give them." Before MCP, context arrived by copy-paste or bespoke integrations; every team re-implemented prompt logic, tool wiring and access to the same data (e.g. each team writing its own access to the company vector DB).
  - Precedents: REST APIs standardised front-end↔back-end; LSP let one Go language server serve every LSP-capable IDE. MCP does the same for AI app ↔ external systems.
  - Three primitives, split by WHO controls them:
    - Tools — model-controlled: the LLM decides when to call (read, write, update DB, write files).
    - Resources — application-controlled: data the server exposes (files, JSON, images, static or dynamic); the app decides whether to attach it (Claude Desktop shows them as attachments; can also auto-attach). Clients can subscribe and get notifications when a resource changes.
    - Prompts — user-controlled: server-provided templates invoked like slash commands (e.g. Zed's PR-summary command takes a PR id and expands a long prompt).
  - Why not make everything a tool? MCP is about richer app↔server interaction, not just more context for the model; separating model/app/user control lets each be governed differently.
  - Agents = "augmented LLM" (retrieval + tools + memory) running in a loop until the task is done; MCP is that whole bottom augmentation layer, and lets an agent gain capabilities after it was built.
- Mechanisms:
  - Clients (Claude Desktop, Cursor, Windsurf, Goose…) speak a standard set of calls: list/call tools, list/read resources, list/get prompts. The model does not have to be in the loop; an app can call these deterministically.
  - Community servers for GitHub/Asana were a couple hundred lines each; Anthropic's ~15 launch servers took ~45 min each to write with Claude. Demo: Asana server had ~30 tools; model chained list_workspaces → search_projects → create tasks without being told the project name.
  - mcp-agent (LastMile AI) demo: ~80-line orchestrator + three sub-agents (researcher with search/fetch/filesystem servers, fact-checker, report writer with only filesystem/fetch). Orchestrator writes a plan, then dispatches. Agent builder declares task + servers; focuses on loop, context management, model choice.
  - Sampling: a server can ask the client to run an LLM completion (passing system/task prompt, model preferences like "big vs small", temperature, max tokens). Client keeps control of model choice, cost, privacy and can refuse. Lets a server be "intelligent" without hosting a model or holding API keys.
  - Composability: client vs server is logical, not physical; an agent can be an MCP server to its caller and an MCP client to further servers → hierarchies of specialised agents, with sampling requests federated back to the app that owns the LLM.
  - Transport-agnostic; convention: stdio for local, SSE/HTTP for remote. OAuth 2.0 added: server does the handshake with the upstream (e.g. Slack), holds the upstream token, gives the client a session token.
  - Roadmap at the time: official registry API (metadata layer over npm/PyPI/Docker, verification of "official" servers, versioning/changelog, self-hostable like Artifactory); `.well-known/mcp.json` for top-down discovery on a domain; stateless/short-lived connections; streaming; tool namespacing; elicitation/server-initiated interaction; tool annotations such as read-only vs write. (note: early-2025 snapshot; several of these have since shipped/changed.)
- Decisions & trade-offs:
  - Use a tool when it is ambiguous whether retrieval is needed; if you would always query the vector DB, just call it deterministically, no tool needed.
  - MCP does not replace agent frameworks: frameworks own the loop and knowledge management; MCP standardises how context/tools get in (LangGraph has MCP adapters).
  - Put retries, logging, auth and business logic on the server side: the server is closest to the system and clients may have never seen it before.
  - MCP vs plain REST: REST stays for stateless data passing; MCP adds LLM-shaped formatting, context-size control, notifications, sampling, multi-step agent-to-agent interaction.
- Rules of thumb:
  - Today's models handle ~50–100 tools well (Claude up to a couple hundred in his experience). Beyond that: a "search tools" tool (RAG/keyword over tool descriptions) or hierarchical/progressive exposure of tool groups.
  - In multi-layer agent systems, each layer must validate/normalise what it fans in before passing upward, to stop compounding errors.
  - Treat a new server version like a dependency upgrade: pin versions, rerun your tool-call evals against the new version and diff.
  - Use the MCP Inspector to debug connections and logs.
- Failure modes & gotchas:
  - Trust: connecting to arbitrary servers is like installing arbitrary packages; use allow-lists, self-hosted registries, verified servers.
  - Tool-name collisions across servers (prefix with server name until namespacing lands).
  - Observability across nested servers is not enforced by the protocol; good servers must return useful metadata/logs.
- Relevance to Sam:
  - Wrap Trameter supplier APIs (Viator, RateHawk etc.) and the master location dataset as internal MCP servers: one team-owned server with retries, auth, rate limits and LLM-friendly response shaping, reusable by every agent/app instead of each pipeline re-implementing it.
  - Mark tools read-only vs write (search availability vs create booking) and gate the write ones.
  - For the blog→trip pipeline, deterministic steps (always look up candidate places) should be direct calls, not tools; reserve model-chosen tools for genuinely ambiguous steps.
  - Keep each agent's tool list small; if exposing many supplier endpoints, add a tool-search layer.

## Hamel Husain (with Teresa Torres) — From Noob to Automated Evals In A Week (as a PM) (https://www.youtube.com/watch?v=N-qAOv_PNPc)
**Evals are just the scientific method applied to a non-deterministic product: look at real traces, name the failure modes, write the cheapest check that catches each one, validate it against your own labels, then measure every change before shipping.**

- Context: Torres built an "interview coach" that grades customer-interview transcripts on 4 rubric dimensions (story-based question, set the scene, build the timeline, redirect generalisations), returning a score + real excerpts + tips per dimension. Started as one prompt in a Claude project; grew to a 5–9 LLM-call workflow on AWS Lambda + Step Functions.
- Mental models / first principles:
  - Vibe-checking felt fine until she annotated traces systematically — then "my coach sucks." Annotation reveals problems that spot checks hide.
  - Every change (model, prompt, temperature, chunking, workflow) improves some metrics and worsens others; you need a table of per-eval results to make the release judgment call. "When I can measure it, I can improve it."
  - Domain expertise drives eval design. Only someone who knew what students get wrong would see that "general question" failures can be caught by keyword rules; an engineer would likely have built an LLM judge.
  - Writing a precise eval forces you to sharpen the rubric — in one case the code eval was more correct than her human labels, so she fixed her labelling guidelines.
- Mechanisms / workflow:
  1. Collect traces somewhere familiar (Airtable). Start data: ~35 transcripts (own, instructors', alumni). Airtable interface as first annotation UI: trace, free-text note, failure-mode tags, "reviewed" checkbox.
  2. Open coding → failure modes. Two catastrophic ones: coach suggests a leading question; coach suggests a general ("typical morning") question.
  3. First evals in a Jupyter notebook (markdown brainstorm + several alternative implementations per eval, run side by side):
     - Leading question → LLM-as-judge (definition + list of suggested questions → pass/fail). Iterated on input shape: all questions at once vs one at a time vs full response, keeping whichever best matched human labels.
     - General question → code assertion on red-flag keywords; near-zero false positives, catches nearly everything she catches.
  4. Pre-processing helpers: code extracts coaching tips from the JSON response; an LLM extracts suggested questions from tips. Evals receive only what they need → fewer input tokens, cheaper, faster, more accurate. Cache helper outputs for the dev set so iterating on an eval does not rerun expensive calls.
  5. Run all evals × all traces → matrix (rows traces, columns evals). First run: the two key failures at ~20–25%. One column "failing everywhere" turned out to be a bug in a helper function; another exposed a label problem.
  6. Score judges vs human labels (agreement), iterate to align.
  7. A/B runner: run dev set (sampled from production) through production config vs candidate change, compare eval tables.
  8. Later: eval code moved into the main repo; some evals run in production as guardrails on a percentage of traffic; notebook becomes a pure analysis/diagnostic tool (pick a trace → see helper outputs + each eval's verdict and reasoning).
- Decisions & trade-offs:
  - Code assertion vs LLM judge: prefer code wherever a deterministic signal exists (cost, speed, no drift).
  - One monster prompt confused criteria across dimensions; splitting into 4 calls raised overall quality but introduced "greedy dimension" errors (each call judged the whole interview by its own dimension). No change is purely positive.
  - Annotate whole traces (what the customer sees) for error analysis and prioritisation; per-dimension annotation produced many non-critical errors and endless work. Use per-component annotation only when building a component-specific eval.
  - Built-from-scratch tooling (notebook + custom annotation UIs) over eval platforms, to understand it end-to-end.
- Rules of thumb:
  - Make annotation effortless: custom UI with progress bar, keyboard shortcuts, next/back, failure-mode pills; she spins up a new annotation tool per eval in minutes with Claude Code. Or let Claude Code present items one at a time in the terminal: ~100 labels in <10 min.
  - Measure before fixing. Cross-call failure (same excerpt praised in one dimension, criticised in another): 81% of transcripts. Fix was a workflow change — pass the list of already-used excerpts to subsequent calls and forbid reuse → 3%. She first had to write an eval to detect near-duplicate (not exact) excerpts.
  - A day and a half on a question classifier went nowhere; the simple fix came after a break. Try the simplest workflow change before building a new model component.
  - Pair-program with coding agents: plan mode first, compare patterns and pros/cons, reviewed project plan, then small diffs ("four-line diffs, not thousand-line diffs"). Never run code you do not understand; keep markdown docs/README for future you and the AI.
- Failure modes & gotchas:
  - Synthetic transcripts were hard to generate realistically; real data first.
  - Users sent real (sensitive) customer interviews instead of practice ones → compliance problem (she was storing in Airtable, not SOC 2) → moved the product into a compliant partner platform.
  - AWS Step Functions: JSONPath errors are not catchable/retryable; sub-workflows that return errors in the payload instead of throwing break parent error handling. Integration tests caught it right before launch.
  - Thumbs up/down only tells you which trace to look at; aim for UX that captures granular feedback (which excerpt/tip was wrong).
  - Synthetic-data dimensions should come from real customer use cases/pain points.
- Relevance to Sam:
  - Blog→trip pipeline: build the same matrix — each stage (LLM gate, extraction, trip building) gets cheap code checks first (e.g. extracted place exists in master location table, dates parse, price fields numeric) and LLM judges only for fuzzy failures (e.g. "extracted activity not actually recommended by the author").
  - Cache the expensive upstream outputs (article fetch, extraction) for a fixed dev set so iterating on later-stage prompts/evals is cheap.
  - Pass prior-stage outputs forward to avoid cross-call duplication/contradiction (e.g. send already-extracted places to the next chunk's extraction).
  - Error-analyse the final trip a user would see, not each sub-call, to prioritise.
  - Watch for unexpected sensitive inputs once a tool is public.

## Hamel Husain (with Nandan Thakur) — How To Build And Evaluate Search Agents (https://www.youtube.com/watch?v=pi-IW4HYJwU)
**Agentic search = an LLM that calls a retrieval tool in a loop; its quality is as much a retrieval problem as an LLM problem, and you must evaluate accuracy, retrieval recall AND trajectory efficiency (search calls, repetition), not just the final answer.**

- Mental models / first principles:
  - Naive RAG: retrieve top-k once → generate. Agentic search: LLM first decides whether to search, issues queries to a retrieval tool, reads results, refines queries, repeats, then answers.
  - Old QA benchmarks (HotpotQA ≤2 hops, Natural Questions, BEIR) are factoid-easy and cannot measure multi-hop reasoning-plus-retrieval.
  - "Riddle-style" questions (BrowseComp): easy to verify, hard to find. Solving them is a funnel: a broad constraint gives ~100 candidates, a second constraint cuts to ~20, the final one to 1. BrowseComp questions were kept only if another human with search couldn't solve them in 10 min and GPT-4 failed.
  - Reported accuracy without stating the retriever is not reproducible or comparable: the search tool can swing results hugely.
- Mechanisms:
  - BrowseComp-Plus: fixed ~100k-doc corpus = human-verified evidence pages (gathered with o3, then human-annotated per question part) + hard negatives; per-query relevance judgments. Metrics: exact-match answer accuracy, number of search calls, retrieval recall of positive docs.
  - Findings: the LLM and the retriever both matter. Same LLM, moving BM25 → dense embedding (Qwen3) raised accuracy and cut search calls. With GPT-5 fixed, BM25 ~60% vs a reasoning-tuned late-interaction (ColBERT-style) retriever ~88–90%. A well-tuned BM25 can still reach 60–80% in some systems — never skip a tuned lexical baseline.
  - Long-form answers (TREC RAG): evaluate with "nuggets" (atomic key facts expected in the answer), support/attribution (does the cited doc actually support the sentence), source credibility. Moving from engineered LLM-judge setups to agent judges, validated by Kendall's tau against prior years' human labels. Observed bias: agent judges over-use "partially supported"; humans are more decisive (not relevant / fully relevant).
  - Synthetic training data for multi-hop search:
    - WebDancer: crawl link graphs and have an LLM write Q/A; or E2HQA — iteratively add constraints to a simple question using retrieved docs to make it harder while keeping the answer.
    - InfoSeeker: entity graph + blur nodes to build constraint-satisfaction questions (needs an entity graph — expensive prerequisite).
    - ORBIT (his, frugal, ~20k pairs): seeds = Wikipedia pages from ~100 categories; free DeepSeek chat UI driven by Selenium does search+reasoning to write a multi-hop Q/A with few-shot examples; self-verification by the same model; external verification by scraping the cited sources and checking that small open models (Qwen3-4B, gpt-oss) can answer from them. Accept losing hard examples to avoid false positives. Quality check: eyeballed 50–100 samples per stage; stage-2 output had wrong answers, stage-4 was accurate.
  - Training: RL with verifiable rewards (GRPO/PPO), reward = 1 if exact-match answer else 0. Simple reward works well as a baseline (Search-R1 lineage). Trajectory pattern: think → search → information → … → answer. ORBIT-4B (Qwen3-4B base), same pipeline with only the dataset changed, beat comparable data on Wikipedia benchmarks; FRAMES is the most trustworthy (needs 5+ docs).
- Decisions & trade-offs:
  - Leaderboard top (95% accuracy) used ~210 search calls per question; an entry with 90% accuracy and 12 calls is better in practice (tokens, latency, cost).
  - Strict multi-stage verification reduces dataset size/hardness but prevents training on wrong labels — choose precision over recall for training data.
  - Benchmarks: mix human queries (short, ambiguous) and LLM-generated queries (long, context-rich — which is what agents actually send to the retriever). Generating synthetic queries over your own corpus (e.g. cluster related docs, write a question needing all of them) solves the enterprise cold-start/privacy problem.
- Rules of thumb:
  - Always report/pin the retriever when evaluating an agent; ablate retriever with LLM fixed and vice versa.
  - Track per trajectory: number of search calls, query repetition (semantic duplicates), query provenance (from original question / retrieved context / model memory / earlier queries), document novelty (new vs already-seen docs), topic shift.
  - Correct answers tend to come with fewer search rounds; long loops correlate with failure — a cap on search turns plus a "no new docs" stopping signal are sensible.
  - Read even one or two trajectories yourself; it shows where the agent loops.
- Failure modes & gotchas:
  - Agents repeat semantically identical queries a lot (in one study the dominant query category showed heavy repetition), wasting tokens.
  - Self-verification with the same model that generated the data is biased (it may "remember" its own answer).
  - Exact-match metrics only fit short verifiable answers.
- Relevance to Sam:
  - Matching blog mentions to the master location dataset / supplier catalogues is the retrieval half of the problem: build a small labelled set (mention → correct GeoNames/product ID) and measure recall@k of BM25 vs embeddings vs hybrid before blaming the LLM.
  - If an agent searches supplier APIs (e.g. "find a Viator tour for this activity"), log every query and count calls/repeats; cap turns; dedupe queries; cache results.
  - The ORBIT recipe (generate with one model, verify by having a different small model answer from the retrieved sources) is a cheap way to make labelled data or to cross-check LLM classifications of location rows.
  - Small 4B models trained or prompted well can do competent tool-calling search — relevant to running local models.

## Hailey Schoelkopf (EleutherAI, on Hamel Husain's channel) — A Deep Dive on LLM Evaluation (https://www.youtube.com/watch?v=IsZVCnViwhk)
**Evaluating LLMs is hard because scoring free-form language is itself unsolved, and tiny implementation details (prompt whitespace, tokenizer, answer format) swing scores enough to flip conclusions — so prefer downstream evals you run and control yourself.**

- Mental models / first principles:
  - Core difficulty: you need a reliable, repeatable way to judge natural-language output. Naive string match fails — "it's definitely not Joe Biden" contains the right string but the wrong meaning. If you could always score free text perfectly, you would have solved hallucination.
  - Three ways to probe a model: (1) log-likelihoods of given answers, (2) perplexity on a dataset, (3) generate text and score it.
  - Transformers produce next-token distributions for every position in one forward pass (causal mask), so P(y | x) for any continuation y is one pass: feed x+y, sum log-probs of y's tokens. Also can check whether y is the greedy continuation in one pass.
  - Model evals (MMLU-style general capability; overfitting is bad because it destroys the signal) vs downstream evals (your actual task on your real data; "overfitting" to it is fine — that is just being good at your job).
- Mechanisms:
  - Multiple-choice via log-likelihood (MMLU, most Open LLM Leaderboard tasks): compute P(A|q), P(B|q)… pick the max. Pros: cheap (no long generations), never an invalid/abstaining answer, works for small base models that can't generate coherently. Cons: no chain-of-thought, unlike real usage, models never have to produce the answer themselves.
  - Perplexity: average per-token log-prob of true next tokens over a corpus; essentially a validation loss on any distribution you care about. Useful diagnostic for pretraining / domain fit; misleading for chat models whose style differs from the corpus.
  - Tokenizer dependence: summed/averaged log-likelihood and perplexity aren't comparable across models with different tokenizers (bigger vocab → fewer tokens to predict); normalise (e.g. per byte/character) if comparing.
  - Generation scoring options: regex/extract "The answer is X" (penalises models that don't follow your format — a confound), or LLM-as-judge (fallible).
  - HumanEval example: prompt ending with "\n\n\t" vs no trailing whitespace changed scores by ~5% for many models, because the model must now emit a token like "return" without its usual leading indent.
- Decisions & trade-offs:
  - LLM-as-judge is most needed exactly where it is weakest: complex/subjective judgments. Judges do simple checks (extracting an answer) far better than hard ones. Don't use a judge on a task it cannot do itself; measure judge performance before relying on it.
  - To force a clean multiple-choice answer: structured/constrained generation (mask disallowed tokens) or log-likelihood scoring when you have logit access; otherwise strong system prompt and "ask nicely" — less reliable because models are trained to chat/think aloud. Closed APIs often expose only partial logprobs.
  - Human evaluators for subjective quality: expensive but sometimes worth it.
- Rules of thumb:
  - The best eval is one you ran yourself; better still, one you designed for your own use case. Don't trust vendor tables — self-reported numbers often use different settings per model (e.g. the Llama 3 launch comparison).
  - Share/version the evaluation code; without code, results aren't reproducible. Use vetted harnesses (lm-evaluation-harness, HELM, OpenCompass) for standard benchmarks.
  - Fair comparisons: equal prompt-engineering effort for baseline and new system, or use each model's own chat template; at minimum record every detail.
  - Check that eval-time prompts are byte-identical to production prompts (whitespace, templates) or your eval lies.
- Failure modes & gotchas:
  - Label errors in benchmarks (HellaSwag reportedly ~26% of samples with some error): once scores approach the noise ceiling, gains are overfitting.
  - Data contamination, saturation, and measurement validity (does it measure what you care about?).
  - Many benchmarks reward memorised facts more than multi-step reasoning (contrast with ARC).
  - (note: 2024 talk; "current president" example is dated, the point stands.)
- Relevance to Sam:
  - For LLM classification of millions of location rows with a local/open model, log-likelihood scoring over a closed label set (or constrained decoding / JSON schema with an enum) gives guaranteed-valid labels, is cheaper than free generation, and yields a confidence score you can threshold to route uncertain rows to a bigger model or a human.
  - Keep the eval harness and production on the exact same prompt template/serialisation; a stray newline can cost several points.
  - Your downstream eval (real blog articles → correct bookable items) matters far more than public leaderboard ranks when choosing a model; it is fine to tune hard against it, but keep a held-out slice.

## Hamel Husain (with Shreya Shankar) — How to Automate AI Evals (Correctly) (https://www.youtube.com/watch?v=tqUDjc1HzO4)
**You cannot outsource evals end-to-end to an AI, because "what good means" lives in your head, not in the traces; use AI to externalise and scale your judgment (build review UIs, taxonomise your notes, propagate your labels), not to supply it.**

- Mental models / first principles:
  - Evals = systematic measurement of application quality. Output quality is reported as the top barrier to productionising agents.
  - Epistemic argument: if a tool could fully find and fix your product's problems from traces alone, it could do it for every competitor — your judgment/taste is the differentiator.
  - Lifecycle: Analyze (error analysis: discover failure modes in traces) → Measure (prevalence of each) → Improve (prompt change, model switch, fine-tune…), repeated as the product evolves.
  - Error analysis is the hardest step because, unlike supervised ML, there is no label; failures are "know it when I see it" (e.g. AI-writing slop). AI is weakest at this early taste-heavy step and increasingly useful at measure/improve (judging at scale, prompt optimisation, hill-climbing).
  - Pareto: ~80% of problems come from ~20% of failure modes — measure prevalence and fix the big ones first.
- Mechanisms — her open-source error-analysis skill for a coding agent:
  1. Agent loads a data sample and infers shared structure/fields.
  2. Designs a visual encoding (e.g. colour by message role) for human review.
  3. Builds a local review app with three views: one-at-a-time trace viewer, map view of clusters, progress view (tree-map of found failure modes).
  4. Clusters traces on features it chooses (first version forced embeddings + k-means; she relaxed it because embeddings aren't always a good representation) and samples representatives so the human doesn't review everything (demo: 33 articles → 21 samples; app built in ~5 min).
  5. Human annotates in situ (highlight text, write free-form note). Agent monitors the annotation file, builds a failure-mode taxonomy from the notes (e.g. "informal fragments / colon interjections", "contrastive 'it's not X, it's Y' framing"), and pushes for breadth (many modes) and depth (many examples per mode).
  6. Agent proposes other instances of an existing failure mode in already-reviewed and unreviewed traces; human accepts/rejects. Agent does NOT invent new failure categories — she found validating the agent's taste worse than stating her own.
- Common mistakes:
  1. "Claude, read my traces and evaluate my app." It finds generic issues, misses taste-specific ones, invents priorities ("extremely repetitive voice is the biggest issue"), and nothing persists — next batch starts from scratch. Fix: keep your workflow, use AI on specific steps, persist failure-mode definitions in code/memory.
  2. One pass over the data. Re-reading surfaces new modes once more traces are in your head (she only noticed the overused word "matters" deep into review, then saw it in every article). Use an outer loop over data and an inner loop over hypotheses per trace; have the agent back-apply each new label to earlier traces.
  3. Uniform accuracy bar across all AI features. Internal Slack-summary tools need little eval investment; customer-facing features need a high bar. Work backwards from worst cases — ask a coding agent, given traces + app description + code, to list worst-case user harms (e.g. sabotaged citations, leaking a journalist's private notes) → turn those into evals and guardrails.
- Decisions & trade-offs:
  - Sampling via LLM-chosen clusters is a guess; useful to reduce fatigue, not a guarantee of coverage.
  - Agent-built UIs are rough (duplicated colours, unneeded chips) — fine, they're disposable; iterate the skill instructions when the UI annoys you.
  - Early findings: general coding agents (Claude Code, Codex) customised with your domain prompt were often more exhaustive at finding failure modes than dedicated vendor discovery tools; no tool finds all modes with good precision. Vendor tools are "someone's prompt plus a harness."
- Rules of thumb:
  - Discovering a failure mode often means a missing instruction you assumed the model "should just know" — feed it back into the system prompt in the Improve step.
  - Agents flagging more instances is not exhaustive; still review.
  - Keep a human in the loop for taste; delegate tedium (categorising, counting, visualising, propagating).
- Relevance to Sam:
  - For the blog→trip pipeline, have a coding agent build a throwaway reviewer that shows article text beside extracted items and generated trip, with in-place notes; let it taxonomise your notes and find more examples, while you define what "wrong" means (e.g. activity not bookable, wrong city, author was warning against it).
  - Different bars: internal research-scout summaries need light evals; anything that produces bookable trips shown to travellers or prices needs strict evals and guardrails. Brainstorm worst cases (wrong destination, fake prices, unsafe recommendations, leaking affiliate/private data).
  - For million-row location classification, cluster rows (by country/feature code/source) and sample per cluster for review rather than random sampling; revisit early samples after learning new failure patterns.

## Jerry Liu (LlamaIndex, AI Engineer) — Building Production-Ready RAG Applications (https://www.youtube.com/watch?v=TRjq7t2Ms5I)
**Naive RAG (chunk → embed → top-k → stuff prompt) fails on retrieval precision/recall; fix it by first defining retrieval and end-to-end evals, then climbing a ladder from table-stakes tweaks (chunking, metadata filters, hybrid) to advanced retrieval (small-to-big) to agents and fine-tuning.**

- Mental models / first principles:
  - Two ways to give an LLM unseen data: retrieval augmentation (fixed model, data pipeline injects context into the prompt) vs fine-tuning (bake knowledge into weights/adapters).
  - RAG stack = ingestion (load, chunk, embed, index) + querying (retrieval + synthesis). Learn the low-level pieces even if a framework does it in five lines.
  - Failure sources: bad retrieval — low precision (irrelevant chunks → hallucination, lost-in-the-middle, fluff) and low recall (needed info not retrieved, top-k too low), outdated data — plus LLM-side issues (hallucination, irrelevance).
  - Retrieval evaluation is a decades-old IR problem; reuse IR metrics rather than inventing new ones.
- Mechanisms:
  - Eval data: (query → relevant doc IDs) from human labels, user feedback or LLM-synthesised questions. Retrieval metrics: hit rate/success rate, MRR, NDCG. End-to-end: (query → reference answer) dataset; run full pipeline; LLM-based evals with or without labels.
  - Metadata filtering: attach structured JSON (page, title, year, summaries of neighbours, even LLM-"hallucinated" questions the chunk answers) to chunks; infer filters from the query (e.g. "risk factors in 2021" → year=2021 as a WHERE clause) and combine with semantic search. Raises precision, prevents cross-document/year contamination.
  - Small-to-big retrieval: embed small units (sentences, summaries, or synthetic questions) that point to a parent chunk; retrieve on the small unit, synthesise over the expanded parent. More precise retrieval, allows small top-k (e.g. k=2 vs k=5 with big chunks), reduces lost-in-the-middle.
  - Multi-document agents: model each document as tools (summarise this doc, QA over this doc); with many docs, retrieve over tools first, then let the agent call them — for comparisons/summaries that top-k QA can't do.
  - Fine-tuning embeddings: generate synthetic queries from your chunks with an LLM, fine-tune the embedding model or just a query-side adapter (adapter needs no base weights and avoids re-indexing the corpus).
  - Fine-tuning/distilling the LLM: use a strong model to generate data that teaches a weaker model better synthesis, chain-of-thought and structured output. (note: model names like GPT-3.5/Llama-2-7B are 2023-era; principle stands.)
- Decisions & trade-offs:
  - Start with cheap fixes before heavy machinery: chunking strategy and size, hybrid search, metadata filters → then reranking, recursive retrieval, small-to-big, embedded-table handling → then agents and fine-tuning (higher latency, cost and complexity).
  - More retrieved tokens ≠ better answers; there is an optimal chunk size per dataset, and in one experiment adding reranking actually worsened error metrics — measure, don't assume.
  - Evaluate end-to-end and per-component; when end-to-end is bad, diagnose whether retrieval or synthesis is the culprit.
- Rules of thumb:
  - Be task-specific and have a benchmark before trying any technique.
  - Embed something "retrieval-friendly" (small, focused) but return enough context for synthesis.
  - Use structured metadata whenever the query contains structured constraints (dates, locations, types).
- Failure modes & gotchas:
  - Pre-trained embeddings are not tuned to your domain; may retrieve the wrong thing consistently.
  - Lost-in-the-middle when stuffing many chunks.
  - Mixed collections (multiple years/docs) poison pure semantic search.
- Relevance to Sam:
  - Matching blog mentions to places/products: use metadata filters (country, admin region, feature class, supplier) inferred from article context before semantic/fuzzy matching; this is the structured "WHERE + similarity" pattern and slashes false matches among millions of GeoNames rows.
  - Small-to-big for articles: embed paragraphs/sentences for locating where an activity is mentioned, but give the extraction LLM the whole surrounding section.
  - Build a retrieval eval (mention → correct place ID / product ID) and track hit rate/MRR before swapping embedding models; a query-side adapter or fine-tuned small embedding model is a budget-friendly upgrade.
  - Distilling a big model's labels into a small model is a viable path for classifying millions of rows cheaply.

## David Soria Parra (Anthropic, AI Engineer) — The Future of MCP (https://www.youtube.com/watch?v=v3Fr2JR47KA)
**Agents in 2026 need connectivity, and connectivity is a stack (skills + CLI/computer use + MCP), not one tool; the remaining work is in the harness (progressive tool discovery, programmatic tool calling) and in designing servers for agents rather than mirroring REST APIs.**

- Mental models / first principles:
  - Timeline: 2024 demos; 2025 coding agents (the ideal agent case — local, verifiable with a compiler, a developer present to fix things); 2026 general knowledge-worker agents (finance, marketing) whose key need is connecting to several SaaS apps and shared drives, not a compiler.
  - Pick the right layer per need:
    - Skills = packaged domain knowledge/instructions in a simple file, mostly portable.
    - CLI / computer use = great for local agents with a sandbox and code execution, especially for tools already in pre-training (git, gh); composable in bash, self-discoverable.
    - MCP = when you need rich semantics, UI, long-running tasks, resources, platform independence, no sandbox, authorization/governance/policy ("boring but important enterprise stuff").
  - "Context bloat" isn't a protocol problem: the protocol just moves data; the client chose to dump every tool into the context.
  - Adoption: ~110M monthly SDK downloads (pulled in by OpenAI Agents SDK, Google ADK, LangChain…); most servers are private, connecting company systems to agents.
- Mechanisms:
  - MCP Applications: a server ships its own interactive UI (rendered by web-based clients — Claude, ChatGPT, VS Code, Cursor) AND tools, so a human uses the UI while the model uses the tools on the same thing. Delivered via the protocol's extension mechanism (CLIs won't support HTML rendering).
  - Progressive discovery: give the model a tool-search/tool-loading tool; defer loading tool definitions until needed. Big context reduction shown in Claude Code before/after. Can be built yourself on any API.
  - Programmatic tool calling ("code mode"): instead of model → tool → result → model → tool round trips (each an inference step, slow and token-heavy), give the model a code-execution environment (V8 isolate, Lua interpreter, etc.) with tools as functions; it writes one script that calls and composes them, filters JSON, and returns only what matters. MCP's structured output (declared return schemas) supplies types for composition; if a tool lacks it, have a cheap model coerce output to an expected type.
  - Server-side variant: expose an execution environment rather than many tools (Cloudflare's MCP server cited).
  - Roadmap (mid-2026 spec): stateless transport (Google proposal) so servers deploy like ordinary stateless HTTP services on Cloud Run/Kubernetes — current streamable HTTP is hard to scale; better async "tasks" primitive (agent-to-agent); v2 TypeScript and Python SDKs (he concedes FastMCP is better than the Python SDK he wrote); cross-app access via the company identity provider (log in once with Okta/Google, no per-server re-auth); server discovery on well-known URLs; skills over MCP (servers ship and update usage skills alongside tools — you can approximate today with a load-skills tool). Also elicitation, centralised authorization already added. Project now under a foundation.
- Decisions & trade-offs:
  - Don't auto-convert REST APIs one-to-one into MCP tools — produces poor agent tools. Design the interface as you'd want to use it as a human; that is a good start for an agent.
  - When steps need orchestration, prefer code execution over chained tool calls (lower latency and tokens, more composable).
  - Use MCP's richer primitives (apps, tasks, elicitation, skills) where they add value; otherwise a CLI may be enough.
- Rules of thumb:
  - Best agents will use all connection methods seamlessly.
  - Ship usage guidance (skills) with large tool sets.
  - Declare output schemas on tools so code-mode composition works.
- Failure modes & gotchas:
  - Loading all tools into context → bloat and worse tool choice.
  - Stateful streamable HTTP is hard to scale horizontally (until stateless transport lands).
- Relevance to Sam:
  - Supplier integrations: build a purpose-designed "trip-building" MCP server (e.g. search_activities(place_id, date, budget) returning trimmed, typed results) rather than exposing every Viator/RateHawk endpoint raw.
  - Pipeline efficiency: where the LLM would otherwise chain many lookups per article (resolve place → search supplier → check availability), let it write one script against typed functions, or do the orchestration in your own code — fewer inference round-trips, lower cost.
  - Pair a skill document (how to map blog entities to Trameter IDs, booking rules) with the tools so agents use them correctly.
  - A local pipeline with a sandbox can often just use CLIs/scripts; reserve MCP for shared, governed, multi-client access.

## Jerry Liu (LlamaIndex, AI Engineer) — The Future of Knowledge Assistants (https://www.youtube.com/watch?v=zeAyuLc_f3Q)
**Moving from naive RAG to a real knowledge assistant takes three layers: (1) high-quality parsing/chunking/indexing, (2) agentic query flows where the LLM plans and treats data services as tools with memory, (3) multiple specialist agents deployed as services.**

- Mental models / first principles:
  - Enterprise LLM use cases: document processing/tagging/extraction, knowledge search & QA, conversational agents with history, agentic workflows that take actions.
  - Knowledge assistant = any task in (simple question, complex question, vague research task) → any output (short answer, report, structured output).
  - Naive RAG gaps: crude data processing, no query understanding/planning, no interaction with other services, stateless (no memory). Plain RAG is "a glorified search system."
  - Garbage in, garbage out: an LLM app is only as good as its data processing layer (parse → chunk → index).
- Mechanisms:
  - Parsing: generic PDF text extractors (e.g. PyPDF) collapse tables, mixing numbers and text → hallucinations. A layout-preserving parser (spatially laid-out text the LLM can read) let the model answer train-timetable questions correctly where naive parsing hallucinated. Good parsing alone reduces hallucination, before any retrieval tricks.
  - Agent ingredients on top of RAG, ordered by cost/latency: routing/function calling and tool use → query planning (sequential or DAG) → conversation memory (stateful) → full agents → multi-agent.
  - "Agentic RAG": use the LLM during query understanding (rewrite, decompose, route), with every data source (vector index, SQL, APIs, other agents) wrapped as a tool.
  - Reasoning loops: the most stable is a while-loop over function calling / ReAct; fancier DAG- or tree-based planners exist.
  - Multi-agent benefits: specialisation (more reliable on focused tasks), parallelism, and each agent seeing only ~5–10 tools so you can use a smaller, faster, cheaper model. A single agent with ~1,000 tools struggles.
  - llama-agents (alpha at the time): each agent is a microservice; communication through a message queue; a control plane orchestrates either explicitly (defined flows) or implicitly (an LLM orchestrator delegates). Demo: query-rewrite service → RAG agent service, handling many concurrent requests. (note: early 2024 alpha; the specific repo has since evolved — the architectural idea is what matters.)
- Decisions & trade-offs:
  - Constrained vs unconstrained multi-agent interaction: let agents converse freely or explicitly force flows given inputs; production generally needs some explicit constraints.
  - Agents in notebooks ≠ production; need a service architecture (scaling, concurrency, deployment, reuse across tasks).
  - Each step up the agent ladder adds capability but costs latency and money; choose the lightest that handles your question types.
- Rules of thumb:
  - Fix parsing first, especially tables, charts and multi-column layouts.
  - Specialist agents with small tool sets outperform generalists with huge ones.
  - Treat structured and unstructured sources uniformly as tools for comparison/multi-document questions.
- Relevance to Sam:
  - Blog ingestion: article HTML → clean, layout-aware text (keep headings, lists, tables of prices/itineraries) before extraction; bad parsing will look like LLM hallucination.
  - The blog→trip pipeline maps naturally onto specialist services: gate/classifier (small model) → entity extraction → place resolution (tool over location DB) → product search (tool over suppliers) → trip assembly; each with few tools, cheaper models where possible, connected by a queue for throughput.
  - Prefer explicit orchestration for this pipeline (known steps) over an LLM orchestrator; reserve LLM routing for genuinely variable articles.

## Richmond Alake (MongoDB, AI Engineer) — Architecting Agent Memory: Principles, Patterns, and Best Practices (https://www.youtube.com/watch?v=W2HVdB4Jbjs)
**Agents become reliable and believable by persisting state: memory management (generate, store, retrieve, integrate, update, forget) decides what relevant memory enters the context window, rather than stuffing everything into a large context.** (note: much of the talk is a MongoDB pitch; the reusable content is the memory taxonomy and patterns below.)

- Mental models / first principles:
  - Evolution: chatbots → RAG (domain knowledge) → scaled models with emergent reasoning/tool use → agents. "Agenticity" is a spectrum (like self-driving levels): minimal = LLM in a loop; top = autonomous multi-agent with tools.
  - Agent = perception of environment + LLM cognition + action via tools + memory (short- and long-term). Memory underpins reflection, interactivity, proactivity, autonomy.
  - Large context windows are not a reason to dump all data in; memory management means selecting and structuring relevant memories for each step.
  - Human-memory analogy: short-term, long-term, working, semantic, episodic, procedural (skills — e.g. how to do a backflip).
- Mechanisms — memory types and how to model them (each as a document/collection with metadata):
  - Persona memory: the agent's personality/role, to make it consistent and relationship-building.
  - Toolbox memory: store tool JSON schemas in a database; just before each LLM call, retrieve only the relevant tools via search. Vendor guidance at the time suggested only ~10–20 tool schemas in context; a toolbox lets you scale beyond.
  - Conversation memory: timestamped turns with conversation ID, plus signals like recall count and recency to support forgetting.
  - Workflow memory: record step sequences and failures as experience; retrieve past failures on the next run so the LLM avoids that path or explores alternatives.
  - Episodic, long-term, entity memory; an agent registry (agent → tools, persona).
  - Memory-management operations: generation, storage, retrieval (most important), integration into context, updating, and "forgetting" — decay/deprioritise rather than hard-delete.
  - Retrieval is not only vector search: combine text, vector, graph, geospatial and metadata queries; agentic RAG = retrieval as a tool the agent chooses to call. Rerankers and good embeddings reduce hallucination.
- Decisions & trade-offs:
  - Dedicated memory tools (MemGPT/Letta, Mem0, Zep) exist, but there is no single way to solve memory; often you need a flexible store and custom memory logic for your use case.
  - Keeping memory in the same operational database as your data simplifies stack (his pitch) vs specialised stores.
- Rules of thumb:
  - Persist intermediate state so agents are stateful across runs.
  - Attach recency/frequency signals to memories to rank and forget.
  - Store failures, not just successes.
- Relevance to Sam:
  - Workflow/failure memory for the blog→trip pipeline: log per-article failures (e.g. "place X resolved to wrong country", "supplier returned none for this city") and feed known corrections back as context or a lookup table so the pipeline stops repeating mistakes — cheaper than prompting harder.
  - Toolbox pattern: if exposing many supplier endpoints, keep schemas in a table and retrieve the few relevant ones per step.
  - Geospatial + text + vector retrieval in one query is directly relevant to location matching (filter by bounding box/country, then fuzzy/semantic match).
  - Entity memory: cache resolved entities (blog name string → place_id/product_id) across articles; reuse saves LLM calls across millions of mentions.

## David Abitbol (Revolut, AI Engineer) — How Revolut Cut AI Coding Costs 90% With Self-Hosted Open Models (https://www.youtube.com/watch?v=Ok6QdZHCcI8)
**Self-hosting a strong open-weights model on dedicated GPUs gave ~2,000 engineers effectively unlimited, flat-cost tokens at ~1/10 of frontier API spend — but only after heavy inference tuning; frontier models become the exception, not the default.**

- Mental models / first principles:
  - Metered tokens with budget caps made engineers ration AI (spend ranged $1k–$10k/month per engineer with unclear ROI); unlimited flat-cost tokens changed behaviour — people used AI for everything, not just the hardest tasks.
  - Measured on ~2,000 engineers: Gemini (first AI with private data) → ~10% lead-time, ~20% coding-time improvement; budget-capped Claude Code → little impact; self-hosted GLM-5.1 with unlimited use → ~60% coding-time and ~70% lead-time improvement. Lead time improved more than coding time: the bottleneck isn't just writing code (digging through tickets/docs/inbox).
  - Inference performance is workload-dependent; naively loading weights on GPUs performs terribly. Tune before you scale.
- Mechanisms / numbers:
  - Capacity: ~56 NVIDIA B200 GPUs to serve ~2,000 engineers with GLM (large MoE model).
  - Latency targets: time-to-first-token < 3 s (complaints start past ~5 s); ~200 tokens/s output is enough because users run tasks in parallel and check back rather than watch streams (off-peak they got ~800 tok/s).
  - Cost: blended ~$0.30 per million tokens; invoice ~10% of previous Anthropic/OpenAI spend.
  - Tuning result on the same 56 GPUs (April→August): requests/s ~3 → >7 while TTFT >10 s → ~1.6 s (~88% faster despite ~76% more demand).
  - Knobs: cap context (serve 256k, not 1M — "long context is a scam" for this workload); quantise (on Blackwell, NVFP4; FP4 vs FP8 ≈ 40% smaller GPU footprint with no measurable precision loss for them); tensor parallelism chosen for KV-cache headroom (model fits TP4 but TP8 gave enough KV cache); prefix-cache hit rate for coding should exceed 90%; speculative decoding, prefill/decode tuning (P/D disaggregation).
  - Setup: OpenAI-compatible endpoints (incl. Responses API) used from open-source coding harnesses (OpenCode, Codex); text-only main model, with a local router sending image tasks to a vision model (e.g. Gemini or a small GLM vision "flash" model).
- Decisions & trade-offs:
  - Prefer the smartest text model and route the occasional image task elsewhere, rather than a weaker multimodal model for everything.
  - Frontier closed models kept for exceptional tasks.
  - Chinese-origin open weights: risk is about inference endpoints, not weights; hosting weights on US/EU infrastructure keeps data local.
- Rules of thumb (governance, EU-regulated bank):
  - Keep a human in the loop and prove it with a sandbox: allow/deny rules per operation (e.g. allow grep, block gradle/deploy) so regulators see it is not an autonomous agent; also protects against rogue insiders.
  - Host GPUs in jurisdictions within data-adequacy frameworks (US/EU/UK/Israel); providers default to zero data retention.
  - Strong OpenTelemetry logging for incident forensics and analytics.
  - Prompt injection: lower risk with strong models, but still consider it.
- Failure modes & gotchas:
  - Over-provisioning context length wastes KV cache and kills throughput.
  - Scaling GPUs before tuning parameters wastes money.
- Relevance to Sam:
  - Same economics for bulk jobs: millions of location rows or blog articles are a throughput workload — an open-weights model on rented GPUs (or a quantised small model locally) with flat cost may beat per-token APIs. Benchmark on your eval set first.
  - For a 16GB laptop: quantisation (4-bit) and a short max context are the main levers; keep prompts short and shared so prefix caching helps (put the fixed instructions/few-shot examples first, the row last).
  - Batch/offline work doesn't need low latency — optimise for throughput, run at off-peak when renting.
  - Route by task: cheap/local model for classification and gating, a frontier model only for the hard residue.

