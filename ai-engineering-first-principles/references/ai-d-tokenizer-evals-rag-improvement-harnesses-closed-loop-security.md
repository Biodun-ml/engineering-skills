# ai-4 — cross-cutting principles

1. **Look at your data before you write any eval.** Read real traces, write a note on the *first upstream* failure in each, have an LLM cluster the notes into failure modes, then count them. This is the highest-ROI activity in AI engineering, and it also generates your product roadmap. (Husain & Shankar, Lenny's Podcast; Husain/Carter, Error Analysis)
2. **Log full traces from day one.** Capture the transcript, every tool/API call with its result, retrieved context, and the final output, ideally as one flat record per item. Without traces you can't do error analysis, verify outcomes or self-improve. (Uber closed-loop evals; Holmes/Warp; Husain & Shankar)
3. **Evaluate components through a failure funnel, not just end to end.** Break the pipeline into dependent stages, measure the pass rate at each one, and prioritise early-stage failures (later stages never get a chance on those cases). End-to-end pass/fail tells you nothing about where to act. (Bischof, Stop Managing AI Projects; Uber)
4. **Choose the cheapest evaluator that works.** First fix obvious prompt or engineering bugs with no eval at all. Then use code assertions (schema, IDs resolve, format). Use LLM-as-judge only for subjective, persistent failures. Each judge covers *one* failure mode and gives a *binary* verdict (or yes/no/unsure). No Likert scales. (Husain & Shankar; Uber)
5. **Validate every LLM judge against human labels with a confusion matrix, never raw agreement.** With rare failures, an always-pass judge looks 90% "accurate". Iterate until both off-diagonal error types are small, and use held-out splits. (Husain & Shankar; Uber's human golden set)
6. **Requirements emerge from data (criteria drift).** You can't write the rubric up front. Treat judge prompts as living PRDs, and keep a human-labelled golden set to gate every prompt or config change. (Shankar; Uber)
7. **Plan AI work as experiments, not dated features.** Each item is a question, a metric and an estimated effect size from a baseline and a small pilot, plotted on ROI vs effort. Write success criteria (a "validation contract") *before* implementation so tests aren't shaped by the code. (Bischof; Alvoeiro/Factory)
8. **Harness over prompt.** Put determinism in code around the model: tool registry, max steps and attempts, context compaction, privileged handlers (credentials, login), and verification from the tool trace or world state, never from the model's self-report. Make failure honest before you make it succeed. A good harness lets cheap models do the work. (Kumar/IBM Harnesses)
9. **Agents hide their mistakes.** They work around broken environments and report success. Grade every run with rubric judges (efficiency, verbosity, conformance) using the tool calls. Have a scheduled agent turn recurring failures into config/skill PRs for humans to review. Track score trends to prove each fix worked. (Holmes/Warp)
10. **Close the loop in production.** Sample production → human labels → diagnose which component is at fault → auto-tune the prompt/config (reflect + synthesise) → benchmark on the golden set → version in a config store, with rollback. Bound self-correction loops with K iterations, and treat "skip / don't publish" as valid outcomes. Watch for reward hacking (safe but meaningless outputs). (Uber)
11. **Retrieval MVP = BM25 + embeddings + metadata pre-filtering + a reranker.** Embeddings are lossy on names, jargon and numbers; BM25 is nearly free and almost always helps. Pre-filter by metadata (time, type, region) so top-k can't fill up with wrong-but-similar docs. A cross-encoder reranks a few dozen candidates. You need a search engine, not just a vector DB. (Clavié; Jason Liu)
12. **Segment queries into topics vs capabilities, and measure each segment.** Track count, relevance proxy (cosine/rerank score) and satisfaction per cluster. Content gaps mean add inventory; capability gaps mean add filters, date handling, BM25 or query decomposition. Monitor a live classifier with an "other" bucket to catch drift. (Jason Liu, Systematically improving RAG)
13. **Synthetic retrieval evals on day one, and fine-tune when you have data.** Generate questions per chunk and measure whether the source chunk lands in the top-k. Use structured outputs to turn queries into search objects. Spend compute at ingest (summaries, generated questions, pre-extracted facts). Fine-tuning a small embedder on about 2k pairs with hard negatives beats general APIs. (Jason Liu; Clavié)
14. **Separate creator from verifier, and put the right model in each seat.** Fresh-context validators that never saw the code (ideally from a different provider) review adversarially. Run writers serially with read-only parallelism. Use structured handoffs that record commands and exit codes. Keep orchestration in prompts/skills so it improves as models improve. (Alvoeiro/Factory)
15. **Break the lethal trifecta.** Private data + untrusted content + an outbound channel is a recipe for exfiltration. Sandbox agents off your laptop, allow-list egress, scope credentials, route model calls through a logging/redacting gateway, and firewall destructive commands. Secrets belong in the harness, never in the prompt. (Patterson/Coder; Kumar)
16. **Tokenization explains many "model" failures and costs.** Spelling, arithmetic, non-English bloat (about 3x tokens), trailing-space weirdness and special-token injection all come from it. Measure token counts of your formats, keep string matching and arithmetic in code, and never let untrusted text be parsed as special tokens. (Karpathy, GPT Tokenizer)
17. **Dates are a known LLM weak spot.** Always inject today's date (and document dates), add date-range filters, normalise fiscal years and the like at ingest, and keep a date-resolution test set. (Carter/NurtureBoss 33%→100%; Jason Liu)
18. **Make looking at data frictionless.** Build a small custom viewer for your domain (hide the boilerplate, one-click good/bad, a note field, AI categorise-and-count). A benevolent-dictator domain expert does the labelling, about 100 traces or until saturation, then about 30 min/week. (Husain & Shankar; Carter)

**Applied to Sam's work:** For Drive Button (gate → extraction → trip building):
- Treat it as a failure funnel with a human-labelled golden set of articles.
- Use code verifiers first: every place resolves to a GeoNames ID, every product exists with availability.
- Add binary faithfulness judges, validated against Sam's own labels.
- Bound retries, then "don't publish".
- Keep the article-reading LLM away from credentials and booking actions (untrusted input).
- Match places with hybrid BM25 + embedding search, pre-filtered by country/admin region and reranked.

For millions of location rows: a cheap or local model inside a strict harness (allowed labels, schema), compact input formats to save tokens, a stratified hand-labelled sample with per-class confusion matrices, and escalation to a stronger model only on low-confidence or disagreement cases.

---

## Andrej Karpathy — Let's build the GPT Tokenizer (https://www.youtube.com/watch?v=zduSFxRajkE)
**Tokens, not characters, are what an LLM sees; many "model" bugs (spelling, arithmetic, non-English, code, odd outputs, cost) are really tokenizer bugs.**

- **Mental model.** The tokenizer is a separate, separately-trained pre-processing component with its own training set. It translates text (Unicode code points) to integer IDs and back. Each ID indexes a row of the embedding table (input) and a column of the final LM-head projection (output). The LLM never sees letters, only these chunks.
- **Why not raw code points or raw bytes?** Unicode has ~150k code points and keeps changing, so it is a large, unstable vocabulary. Raw UTF-8 bytes give a tiny vocabulary (256) but very long sequences, and attention cost and the finite context window make long sequences expensive. Byte-level BPE is the compromise: start from UTF-8 bytes (UTF-8 chosen over UTF-16/32 because it is not padded with zeros and is ASCII-compatible), then compress.
- **BPE mechanism.** Count adjacent pairs, find the most frequent one, mint a new ID (256, 257, ...), replace every occurrence, repeat. The number of merges is the vocabulary-size hyperparameter. The tokenizer's "parameters" are just the ordered merges table plus the vocab (ID→bytes). Example: 20 merges on a blog post took 24k bytes down to 19k tokens (about 1.27x compression). GPT-2 had 50,257 tokens (256 bytes + 50k merges + 1 special `<|endoftext|>`). GPT-4's cl100k has about 100k tokens.
  - Encoding applies merges in the order they were learned (lowest merge index first) until nothing more can merge. Decoding concatenates byte strings and then UTF-8-decodes them. Use `errors="replace"`, because a model can emit token sequences that are not valid UTF-8 (you see the replacement char, which means bad output).
  - encode→decode round-trips, but decode→encode does not always.
- **Pre-tokenization regex (GPT-2/GPT-4).** Text is first split into chunks (letters, numbers, punctuation, whitespace, common contractions), and BPE only merges inside a chunk. This stops "dog." / "dog!" from becoming separate tokens. Quirks: GPT-2's contraction handling was case-sensitive and only covered ASCII apostrophes. GPT-4 made it case-insensitive, merged runs of whitespace (which makes Python much denser), and caps number chunks at 3 digits. Llama splits every digit separately, which helps arithmetic.
- **Special tokens.** Strings like `<|endoftext|>` and chat delimiters (`<|im_start|>`) bypass BPE and are swapped in by special-case code. Adding a new one requires "model surgery": add a row to the embedding table and a column to the LM head, initialise them small, and often train only those new parameters with the base model frozen. This is common when making chat or tool-use models. Gist tokens push the idea further: learned tokens that stand in for a long prompt, trained by distillation.
- **Vocab size trade-off.** A bigger vocab gives shorter sequences (more text fits in context) but a bigger embedding table and softmax, rarer tokens that end up undertrained, and too much meaning squeezed into one forward step. In practice it sits around tens of thousands up to about 100k.
- **tiktoken vs SentencePiece.** tiktoken (OpenAI) does inference only and runs BPE on bytes, which is clean. SentencePiece (Llama, Mistral) can train and infer, but runs BPE on code points with a byte fallback for rare ones (`character_coverage`). It has many legacy options (sentences, normalisation, `add_dummy_prefix`) that are footguns. Karpathy's advice: reuse GPT-4's vocab with tiktoken if you can. If you must train your own, use SentencePiece BPE and copy Meta's Llama settings exactly. Always enable byte fallback, otherwise rare text becomes `<unk>`.
- **Failure modes explained by tokenization:**
  - *Spelling, counting letters, reversing strings:* long tokens (for example a whole identifier as one token) hide the characters. Fix in a prompt: make the model first write out characters separated by spaces, then operate on them.
  - *Non-English:* the tokenizer's training data was mostly English, so the same sentence costs about 3x the tokens in Korean. That means more cost, less effective context, and worse quality.
  - *Arithmetic:* numbers split into arbitrary chunks (1+3, 2+2 digits...).
  - *GPT-2 Python:* every space was its own token, which wasted context.
  - *Special-token injection:* if user text containing `<|endoftext|>` is parsed as a special token, behaviour breaks. That is an attack surface, so never allow special tokens from untrusted input.
  - *Trailing whitespace / partial tokens:* ending a prompt with a space (or with half of a common token) puts the model off-distribution, because the space normally belongs to the next token. Don't end completion prompts with a trailing space.
  - *"SolidGoldMagikarp":* tokens present in the tokenizer's training data but absent from the LLM's training data are never updated. Invoking them feeds random embeddings, so behaviour is undefined (evasion, insults, hallucination).
- **Practical rules for Sam's work:**
  - Measure token counts of your formats. In his example YAML was 99 tokens vs JSON's 116 for the same data. For cost-sensitive bulk classification of millions of location rows, compact input formats (CSV/TSV rows, short keys, no pretty-printing) cut cost directly. (note: for *outputs*, provider structured-output/JSON-schema modes are now the reliable choice; the trade-off is fewer input tokens vs output reliability, so test both.)
  - Place names in non-Latin scripts (GeoNames alternate names) will cost several times more tokens and get worse handling. Budget for it, and consider transliterated/ASCII names as the primary input.
  - Don't ask an LLM to do character-exact string matching or normalisation that code can do deterministically (casefold, unidecode, edit distance). Use the model for semantic judgement only.
  - Numbers (prices, coordinates, IDs) are tokenized arbitrarily. Keep numeric comparison and arithmetic in code, not in the prompt.
  - Small/local models use different tokenizers (SentencePiece/Llama-family). Re-measure token budgets per model rather than assuming tiktoken counts.

## Hamel Husain & Shreya Shankar (Lenny's Podcast) — Why AI evals are the hottest new skill for product builders (https://www.youtube.com/watch?v=BsWxPI9UM4c)
**Evals are data analysis on your LLM app: look at real traces, name the failures, count them, then automate checks only for the failures that keep coming back.**

- **Mental model.** An "eval" is a broad term for any systematic way to measure and improve an AI application. Unit tests are a small part of it. It also covers error analysis, product metrics (thumbs up, chats per user), online monitoring and A/B tests. Vibe checks are fine at first but stop scaling as the app grows. LLM apps are stochastic, with a huge input surface, so you can't write the right tests until you have looked at data. Going straight to "let me write tests" is where teams fail.
- **Mechanism: the error-analysis loop (from social science qualitative coding + Andrew Ng-style ML error analysis):**
  1. **Log traces.** A trace is the full record of one interaction: system prompt, user turns, tool calls and results, retrieved docs, final output. Any observability tool works (Braintrust, Phoenix/Arize, LangSmith).
  2. **Open coding.** Read a trace and write a short free-text note on the *first / most upstream* thing that went wrong, then move on. Don't try to list every error. Be specific: "did not confirm call transfer with user", not "janky", because an LLM will later need to categorise these notes. Examples from NurtureBoss (an apartment-leasing assistant): it should have handed off to a human; SMS replies were split across several messages and garbled the flow; it offered a virtual tour that doesn't exist (a hallucination only a domain expert can spot); it transferred the call without telling the user.
  3. **How many traces?** Aim for about 100 to get started, then stop at *theoretical saturation* (new traces aren't revealing new kinds of failure). Sometimes that is 15, sometimes 60.
  4. **Axial coding.** Give the notes to an LLM (as a CSV, using the terms "open codes" and "axial codes", which LLMs understand) and ask it to cluster them into failure-mode categories. Then a human refines the categories so they are specific and actionable ("tour scheduling/rescheduling issues", "human handoff issues", "follow-up promises not kept"), not vague ones like "capability limitations".
  5. **Re-label every note** into the refined categories with an LLM (even a Sheets AI formula works). Add a "none of the above" option, which shows when your categories are incomplete.
  6. **Count with a pivot table.** Counting is the most underrated analysis technique. Prioritise by count *and* severity. Example: 17 conversational-flow issues.
- **Who does the coding: a "benevolent dictator".** One domain expert whose taste you trust, often the PM, not a committee. Keep the process cheap or it won't get repeated.
- **Can an LLM do the open coding?** No. It lacks product context and will say the trace "looks good" (it can't know a virtual tour isn't offered). Use LLMs for clustering, labelling and synthesis, never as a replacement for the human's first read.
- **Decisions: fix it, code check, or LLM judge?**
  - Many failures are just missing prompt instructions or engineering bugs. Fix them directly; they need no eval.
  - **Code-based evaluators** come first where possible (valid JSON, markdown, length, forbidden strings). They are cheap and deterministic.
  - **LLM-as-judge** is for subjective, persistent failures. Each judge checks *one* narrow failure mode and returns *binary* pass/fail. Its prompt spells out when the behaviour is required (for example, hand off to a human on: explicit human request ignored or looped, policy-mandated transfer, sensitive resident issue, tool data unavailable, same-day tour request).
  - Don't use 1–5 / Likert scores. They let you avoid deciding, and nobody knows what 3.2 vs 3.7 means.
  - A typical product ends up with about 4–7 LLM judges, not hundreds.
- **Validate the judge against human labels** before trusting it. Don't report raw agreement %: if the failure rate is 10%, a judge that always says "pass" gets 90% agreement. Look at the 2x2 confusion matrix (human vs judge) and iterate the judge prompt until both off-diagonal error types are small. Use separate data splits so you aren't tuning on the test set. A misaligned judge destroys trust in all your evals, and in you.
- **Where judges run.** In CI/unit tests on labelled failure traces, *and* online: sample about 1,000 production traces a day and track the failure rate on dashboards. A judge prompt is effectively a living PRD derived from real data.
- **Criteria drift** (Shankar's "Who Validates the Validators?"): even experts can't write the rubric up front. Their idea of good and bad changes after seeing outputs. Expect requirements to evolve as you read data.
- **Debates.**
  - "Claude Code doesn't do evals": it stands on model-level benchmarks plus heavy dogfooding, and coding is special because the developer *is* the domain expert using it all day. That doesn't generalise to domains like medicine.
  - A/B tests are a kind of eval. They are only worth running when the hypotheses come from error analysis.
  - Generic metrics (cosine similarity, a generic "hallucination score", MMLU-style benchmarks) don't correlate with product-specific failures.
- **Top misconceptions:** (1) buying a tool or letting "the AI eval itself" will do it; (2) not looking at the data. There are many correct ways to run evals, and the goal is product improvement, not a pretty eval suite. If a fix is obvious, just fix it.
- **Cost of the process.** About 3–4 days of upfront work (labelling, first judges), then about 30 min/week, with a cron job running judges on samples. Build your own simple annotation UI with AI (hide the system prompt, filter by channel, show failure counts). Removing friction from looking at data is the highest-ROI investment. Later: swap expensive models for cheaper ones once evals prove quality holds.
- **For Sam's Drive Button pipeline (gate → extraction → trip building):**
  - Log full traces per article: gate decision, extracted entities, matched products, final trip.
  - Open-code about 100 articles. Note the *first upstream* failure, because a gate false positive makes everything downstream irrelevant.
  - Likely axial codes: gate let in a non-trip article; place extracted but wrong geography; hallucinated activity; supplier match wrong; trip order illogical.
  - Code checks: valid JSON, every place resolves to a location ID, dates sane.
  - Binary LLM judges only for subjective failures (for example "the trip reflects what the article actually recommends"). Validate each judge with a confusion matrix against Sam's own labels.
- **For the millions-of-rows location classification:** hand-label a stratified sample and measure the classifier against it with a confusion matrix per class (not raw accuracy, given class imbalance). Re-check on a weekly sample.

## Jason Liu (on Hamel Husain's channel) — Systematically improving RAG applications (https://www.youtube.com/watch?v=RrDBV6odPKo)
**Don't "improve RAG" in general. Segment queries into topics and capabilities, measure each segment, and fix the segments that are big and failing.**

- **Mental model.** RAG is a recommendation system: query → retrieve N candidates → rank → generate. Improve it the way recsys people do: instrument everything, segment, form a hypothesis, test it with an experiment, then measure again. A vague goal ("make RAG better") is not actionable. A good finding looks like: "file-name lookups are 40% of queries with 30% satisfaction".
- **Feedback signals.**
  - **Explicit feedback copy matters.** Changing "How did we do?" to "Did we answer your question?" gave about 5x more ratings *and* made them measure correctness instead of length or latency.
  - Explicit feedback is sparse (maybe 5% of queries), so also log a **cheap relevance proxy for every request**: cosine similarity of the retrieved chunks and the reranker score (for example Cohere). These are relative and somewhat arbitrary, but they exist for 100% of traffic. Use them comparatively (segment vs segment), and rely on them less as real outcome data grows.
  - Implicit feedback when RAG is hidden inside an agent or report: use *user edits*. Produce structured JSON fields rather than free markdown so edits can be attributed: deleting an item means bad precision, adding one means bad recall. Fields with many corrections (for example high-cardinality enums) show where it fails.
  - Citations are free training signal: chunks the LLM cited are more relevant than the ones it ignored. Log them for later embedding fine-tuning.
- **Segmenting.** Cluster the query strings (BERTopic/LDA, or just an LLM). For each topic, track count, mean cosine/rerank score and mean satisfaction. Then read the grid:
  - Low count, low relevance, low satisfaction → deprioritise. Maybe tell the model in the prompt to decline that topic.
  - High count, high relevance, low satisfaction → retrieval is fine; the generation, prompt or answer format is the problem.
  - Low relevance → a retrieval or inventory problem.
- **Two kinds of clusters:**
  - **Content/topic gaps** (for example many pricing questions but few pricing docs). Fix by adding inventory. Alert the content team when fewer than N docs match a popular topic.
  - **Capability gaps:** questions that need features embeddings can't provide. Examples: "who last updated X" needs modified-date and owner metadata; "latest/recent" needs date-range filters *and* today's date in the prompt; "compare A vs B" needs parallel sub-queries; fiscal-year terms need normalising (in some industries FY24 isn't the calendar year); document lookups by file name need BM25.
- **Monitoring.** Once you have the categories, train or prompt a **real-time query classifier** and send its labels to Datadog/Amplitude. Keep an explicit **"other" bucket** and watch its share. If "other" jumps from 10% to 50% (say a new enterprise client that asks compare-and-contrast questions), re-cluster that slice. This catches distribution shift that a one-off offline analysis misses.
- **Mechanism: synthetic retrieval evals (do this on day one, before you have users).** For each chunk, have an LLM write 2–3 questions it answers. The metric: does the source chunk appear in top-k (3/10/25) when you search that question? This lets you compare chunking, embedding models, BM25 vs vector, rerankers and metadata in seconds instead of waiting for deploys. Examples: on Paul Graham essays, BM25 ≈ semantic ≈ 97% (too easy, so BM25 alone was enough). On GitHub issues, full-text 60%, semantic 61–62%, hybrid + reranker 65%. The test revealed that you must filter by repository, which is a UI/filter need, not an embedding one. Wean yourself off synthetic questions as real ones arrive.
- **Query understanding with structured outputs.** Use Instructor/Pydantic to turn the user request into a search object: rewritten query, date range, allowed domains/doc types, tags. Put a `.search()` method on that model so your domain knowledge lives in code you control. Jason prefers this to LangChain/LlamaIndex when the data is specific. He uses LanceDB because full-text, SQL filters and vectors live in one store, with no need to intersect results from separate indexes.
- **Do more work at insert time ("synthetic text chunks").** Spend compute when ingesting to make reads better:
  - Summaries of images and charts.
  - For tables: generate the 6–7 questions that should retrieve the table, embed those questions, and keep the table as metadata.
  - Pre-computed answer chunks for frequent question types (for example "all late-fee clauses in this lease"). Pipeline: classify the doc type → run the targeted extraction → store the result as a new chunk.
  - Normalise fiscal-year dates at ingest.
- **Decisions & trade-offs:**
  - **BM25 + vector by default.** BM25 wins clearly when the searcher wrote the documents, or when users treat RAG as document search by file name (often around 30% of enterprise usage).
  - **Bi-encoder vs cross-encoder:** use both. Vector search narrows 1M docs to about 100, the reranker to about 20. Cross-encoders handle nuance like negation ("I love coffee" vs "I hate coffee") but are slower. Choose top-k and rerank depth from your eval vs latency budget; in recsys, 100 ms can be worth about 1% revenue.
  - **Fine-tune embeddings.** A company with revenue is leaving money on the table by not doing it. With Modal, a roughly 200 MB BERT model fine-tuned on 2,000 Q/A pairs beat OpenAI/Cohere embeddings (about 84% → 86%, and about 89% with 20k pairs). After that, embedding-model choice becomes an infra question (throughput, GPU size).
  - **Chunking.** Follow the vendor defaults (around 800 tokens with overlap). Jason has rarely seen chunk size move metrics much. Whole-document synthetic chunks help more than tuning chunk size. (note: he says both "50%" and "5%" overlap at different points, so treat the overlap figure as unreliable.)
  - **Hierarchical retrievers** are small gains that shrink as context windows grow. Bigger chunks plus evals usually get you most of the way there.
  - **Long context doesn't remove search:** latency and cost still matter. A 5-second answer loses purchases.
  - **"You need a search engine, not a vector DB":** BM25, SQL aggregates and filters are all part of it. For questions that need aggregates over a table, use text-to-SQL. For row lookups, small 20-line chunks are enough. Work out which kind of question users actually ask before you build.
- **Tables and PDFs.** Nested headers are hard to parse. Try LlamaParse, or send the page image to a vision model and ask for a **markdown table** (markdown beat CSV for row-lookup recall in his tests).
- **Citations.** Give each chunk an ID and format citations as markdown links (models are trained heavily on markdown links, so this works better). Render hover previews.
- **The scientific loop (example).** Users complained summaries were too short. Plotting transcript length vs summary length coloured by satisfaction showed the cause. Adding "summary length proportional to transcript" raised average length from 1,100 to 2,000 chars on 50 test transcripts; it was then deployed and satisfaction rose. Hypothesis → data → intervention → experiment.
- **For Sam:**
  - In Drive Button, the "retrieval" step matches extracted places/activities to Viator/RateHawk products and GeoNames IDs. Build a synthetic eval: take known products or locations, generate the phrasing a blogger would use, and check whether the right ID lands in the top-k.
  - Use hybrid search: BM25/trigram on place names, plus embeddings for descriptions. Use metadata filters such as country/admin1, which play the role of "condition on repository".
  - Put an "other" bucket in the article-gate classifier to catch new blog genres.
  - Spend compute at ingest (for example pre-summarise the activity catalogue) rather than at query time.

## Ben Clavié (on Hamel Husain's channel) — Beyond the Basics of Retrieval for Augmenting Generation (https://www.youtube.com/watch?v=0nA5QG3087g)
**A good retrieval MVP has four parts, and it is about 25 lines of code: bi-encoder (embeddings) + BM25 full-text + metadata filtering + a cross-encoder reranker.**

- **De-jargoned mental model.** "RAG" is not an end-to-end system. It is retrieval stitched to generation: three separate components (a retrieval pipeline, a generator, and the glue/prompt format). "My RAG doesn't work" is like "my car doesn't work". You have to find out whether retrieval failed, the LLM failed to use the context, or the formatting failed.
- **The naive MVP (bi-encoder only).** Embed the docs ahead of time, embed the query at inference, then rank by cosine similarity (a dot product of normalised vectors). A plain NumPy array is enough for a few hundred or thousand docs; your CPU does it in milliseconds. A vector DB only earns its place at scale, because it wraps approximate indexes (HNSW, IVF-PQ) that avoid comparing against every vector.
  - It's called a "bi-encoder" because query and doc are encoded separately. That is cheap at query time (only the query is encoded), but the document never "sees" the query.
- **Why single-vector embeddings are lossy.** 384–1024 floats can't hold a whole 512-token chunk. The model learns to keep whatever helped on *its* training queries. So out-of-domain data, jargon, acronyms, rare names and **numbers** (dates, quarters, years) are poorly represented.
- **Component 2: keyword search (TF-IDF/BM25), always.** It weights terms by rarity, and it is from the 1970s. Its compute cost is negligible, and on the BEIR benchmark it is competitive with most neural retrievers except heavily trained ones (E5, BGE). Even the best embeddings still improve when combined with it. Combine scores with a weighted sum (a common default is 0.7 vector + 0.3 BM25). Alternatively, take the top-k from each and let the reranker sort them out; that is better when one retriever is weak for some query types (for example biomedical jargon). Don't feed the BM25 scores into the reranker. "BM25 never told anyone to eat rocks."
- **Component 3: metadata filtering.** Real documents have provenance (department, date, type), and metadata is often more informative than content. Example query: "cruise division financial report Q4 2022". Embeddings will happily return financial reports for the wrong year or division. And plain top-5 *always* returns 5 docs, so the LLM gets 5 financial reports and has to guess which is right.
  - Fix: extract entities from the query (GLiNER does zero-shot NER with arbitrary types such as doc type, time period, department; or use an LLM). Store the same fields on documents at ingest. **Pre-filter** before searching.
- **Component 4: reranking with a cross-encoder.** It scores each (query, doc) pair *together*, as a classifier whose positive-class probability is relevance. That is much more accurate, but too expensive to run over the whole corpus (10M Wikipedia passages would take minutes). So: first-stage retrieval gets tens of candidates, then rerank. Options: Cohere/Jina APIs, local models, LLM rankers (RankGPT), MonoT5. His `rerankers` library gives one API so you can swap them.
  - LanceDB is good for MVPs because vectors, full-text index, metadata filters and a reranker hook all live together.
- **Fine-tuning:**
  - If you have data, always fine-tune both the bi-encoder and the cross-encoder. Even a small budget is better spent generating synthetic queries with an LLM and fine-tuning.
  - Make the bi-encoder "looser" (optimise for recall of candidates) and trust the reranker for precision.
  - Training data: query, positive doc, negative doc (triplet loss). Use **hard negatives**, mined by retrieval, that are close to the right answer but wrong.
  - Tooling: Sentence Transformers v3.
- **ColBERT (multi-vector).** Keeps one small vector (128-d) per token. Scoring (MaxSim): for each query token, take the best match among the doc's tokens, then sum. It generalises much better out of domain and is less sensitive to chunking. Ideally use it as the first-stage retriever; it can also serve as a reranker. Its cost is storage, and his work reduces stored tokens by about 50–60%.
- **Rules of thumb:**
  - Chunking: about 300 tokens, never cut mid-sentence, roughly 50 tokens of overlap on each side. Don't be afraid of duplicating information.
  - Embedding models: stay roughly between 100M and 1B parameters. LLM-based 7B embedders aren't worth the latency, and some generalise worse (E5-Mistral was no better than E5-large despite being about 7x bigger). Try GTE/BGE/E5. Cohere's embedder is a good API default for demos.
  - If you already run Elasticsearch: BM25 for 50–100 candidates plus a reranker is a strong pipeline with no embedding of the corpus at all. This is also how cross-encoders are evaluated in papers.
  - Recall@10 or @100 is easy; recall@1–3 is hard (and labels are noisy). **Long context relaxes retrieval**: if you can pass the top 10–15, a lighter, faster retriever is enough. Long context is RAM, retrieval is disk. You still need the disk.
  - When chunks don't fit: map document→summary in a side table, retrieve chunks, then pass the parent document's summary or the full document. Always weigh it against latency tolerance (users won't wait 20 s).
  - Deciding *when* to retrieve (agentic/ReAct self-triggered search) is still an open problem.
- **For Sam:**
  - Matching extracted place or activity strings to GeoNames, Viator or RateHawk is exactly where embeddings fail: proper nouns, numbers, transliterations. Always run BM25/trigram alongside vectors.
  - **Pre-filter by country/admin region/feature class** (metadata), extracted from the article context, before searching 6M rows. This is the "cruise division Q4 22" lesson.
  - Rerank the top 20–50 candidates with a cross-encoder or a small LLM call instead of running an LLM over everything.
  - For a 16GB laptop, use small embedders (≤1B, ideally about 100–300M) with exact NumPy search on filtered subsets, or LanceDB on disk.
  - Fine-tune a small bi-encoder on Sam's own matched pairs, using hard negatives such as same-name places in other countries.

## Bryan Bischof (with Hamel Husain) — Stop Managing AI Projects Like Traditional Software ("Burn Your Roadmap: Why AI Needs Explorers") (https://www.youtube.com/watch?v=R_HnI9oTv3c)
**Novel AI work can't be date-roadmapped. Plan it as experiments (question → metric → estimated effect size) and use a component-level "failure funnel" to decide which experiments are worth running.**

- **Mental model.** A traditional roadmap is a ledger of fungible, dated features that teams "transact" against (marketing launches, staffing). That works for known work, including AI work that just implements a known pattern (a standard ReAct agent, a paper replication). It fails for anything genuinely new: you can't roadmap territory that hasn't been discovered. Committing to a launch date before a single experiment or customer conversation is the anti-pattern.
- **Mechanism: experiment-driven roadmap.** Each item has three parts:
  1. **Question** (a genuine curiosity, not a directive). "Can we find the relevant tables for text-to-SQL by their row values?" instead of "build a keyword index of row values". "Does extraction work better before or after OCR?" instead of "add a transcription node". "Should we rerank or interleave RAG results?" instead of "seven vendor meetings about rerankers". (Vendors have incentives; don't rely on their objectivity.)
  2. **Metric** that answers it: table recall, per-attribute extraction accuracy, binary classification accuracy (yes/no questions are the gold standard), effect of the relevant doc's rank on generation quality.
  3. **Estimated effect size** from a baseline plus a small pre-experiment. Example: "we retrieve the right table 73% of the time; this might add single-digit %"; "40% of attributes extracted correctly; might add 10%"; "when the relevant doc is in the middle of the results, generation fails 15% more; reranking could put it first 60% of the time".
  - That gives you the ROI axis for an ROI-vs-effort chart, so prioritisation becomes an honest conversation with the PM. A large backlog of measurable experiments makes changing plans less painful (an "agile posture", as opposed to a defensive one).
- **Evals-driven development: the philosophy.** A great eval doesn't answer "does my AI work?" ("kindergarten"). It shows *which of your assumptions are false*. An end-to-end pass/fail ("I asked, I got 7, expected 9") gives you nowhere to go.
- **The failure (capability) funnel.** Break a capability into dependent stages; failing one fails everything downstream. Text-to-SQL example:
  1. Syntactically valid SQL
  2. Executes without error
  3. Returns relevant results
  4. Matches user intent
  5. Solves the user's actual problem. ("What's our ARR?" probably means over time and by segment. Good products ask clarifying follow-ups, as Deep Research does.)
  - A fuller funnel had nine stages: right table retrieved, no hallucinated tables, right table chosen, no invented columns, correct filters, and so on.
  - Weight failures by position: early-stage failures matter more because those cases never reach the later stages. Fixing an early stage can expose new failures downstream.
  - Building it needs human-labelled answers, then "backpropagating" what must have been true at each stage (for example the correct table had to be in the context).
- **Start with component evals, not end-to-end.** Brian: if you only have time for two component evals or one end-to-end eval, take the two components, because they are actionable. Hamel's variant: do a rough end-to-end pass plus error analysis (note the most upstream error) to find which components to zoom into.
  - For agents whose steps are "spaghetti", pick the main paths and stretch them into a roughly linear order (technically a poset). If you can't name any start → steps → end path, you don't know what you're trying to achieve.
  - "Skill issue": unless you just put a system prompt on the user query, there are components to decompose (routing, tools, context sources, RAG, rerank, reflection, follow-ups).
- **Human-in-the-loop products (Stitch Fix stylists).** Evaluate whether the candidates handed to the human were good enough. If the human had nothing good to pick from, that failure belongs to the algorithm.
- **Analysing results is data science.** Look for the story behind the numbers, not just whether a number improved. Plot threshold trade-offs (for example how many items retrieval shows vs precision), keeping them interpretable for the PM. (He jokes that F1 is not one of them.)
  - Compare experiments as **transition matrices** over funnel stages (before vs after a treatment). Two treatments (for example rank fusion vs interleaving) can improve different stages; the funnel tells you which stage matters more.
  - Multiple comparisons matter less if you pre-register: "this experiment targets stage 2".
- **Who you need.** Not necessarily a credentialed data scientist, but people with data sense who are *truth-seeking*.
- **How to adopt it.** Either (a) for a new capability, write out the assumptions and hurdles and bring the PM experiments per funnel stage, or (b) build a funnel for your most important existing capability. Leaders respond to "12% fail before step one" much more than to "60% of evals pass". Failures that tell engineers where to spend their time are what a roadmap was always meant to be.
- **For Sam:**
  - The Drive Button pipeline is a natural failure funnel:
    1. Gate correctly accepts trip-worthy articles
    2. Places extracted (recall, no hallucinations)
    3. Each place resolved to the right location ID
    4. Activities/products matched to suppliers
    5. Availability and price are real
    6. Trip order and timing are sensible
    7. User actually books
  - Label a set of articles end to end, measure the pass rate at each stage, and report it that way to Trameter leadership instead of giving a delivery date.
  - Frame research items (for example "does a 3B local model classify location rows as well as a hosted mini model?") as question + metric + estimated effect, with a small pilot to get the baseline.

## Tejas Kumar (IBM, AI Engineer) — Harnesses in AI: A Deep Dive (https://www.youtube.com/watch?v=C_GG5g38vLU)
**An agent harness is the deterministic code *around* the model loop (tools, guardrails, context management, verification, privileged handlers) that keeps a rented black-box model reliable. Often you fix outcomes by changing the harness, not the prompt.**

- **Why harnesses exist.** We rent inference: limited context, rate limits, and a black-box model the provider could in principle swap silently. You can't control the model, so you build reliability around it. A harness ties the agent to something stable, the way a climber is tied to the rock or a dog is on a leash. Two meanings to keep apart: in ML a "harness" is an eval/test runner (feed inputs, score outputs). The *agent harness* in AI engineering is the runtime scaffolding.
- **Parts of an agent harness** (Claude Code, Cursor and Codex are all harnessed agents):
  1. **Tool registry:** name, description, JSON-schema parameters, and an execute function (the OpenAI-SDK style).
  2. **Model:** sometimes user-selectable.
  3. **Context management primitives:** for example auto-compaction.
  4. **Guardrails:** max steps/iterations, max messages, max attempts.
  5. **The agent loop itself.** The harness is the stuff *around* that loop, and it can be an outer loop of N attempts around the inner agent loop.
  6. **A verify step:** for coding, run lint and tests after the work.
- **Demo mechanism** (browser agent with deliberately weak GPT-3.5 Turbo + Playwright, task "upvote the top Hacker News post"; the prompt was *never* changed):
  - *Baseline:* a `while true` loop that stops when the model says "done" and pushes every event into a trace list. The model clicked upvote, hit the login wall, and **claimed success anyway** (the agent lied).
  - *Step 1, guardrails:* max iterations (about 6) and max messages. When the message limit is exceeded, compress context: keep the system prompt and the user task, drop the middle, keep the last 2 messages. (He calls this naive; better compaction exists.)
  - *Step 2, refactor:* move the loop into `runHarness`, so the entry point is about 19 lines.
  - *Step 3, deterministic verification:* an outer loop with max 3 attempts. After each attempt a *code* verifier inspects the tool-call trace: was there a successful click on upvote? Was there a failed auto-login, or a redirect to the login URL with no recovery? If so, report failure. Result: the agent still failed, but **stopped lying**. Honest failure comes first.
  - *Step 4, privileged deterministic handler:* a login handler that runs every loop iteration before the trace is pushed. It is a cheap no-op unless the current URL is the login page. If it is, the harness fills in credentials (from env/secrets the model never sees), submits the form, and injects a message: "harness logged you in". Result: success after 6 iterations, confirmed on the site.
- **Principles:**
  - **Verify from ground truth (the tool trace or the world's state), never from the model's self-report.**
  - Make failure honest before you try to make it succeed.
  - Anything deterministic (login, credential handling, retries, limits, state checks) belongs in code in the harness, not in the prompt. "Prompt it harder" or putting credentials in the system prompt is the wrong instinct, and the latter is also a security hole.
  - Secrets live in the harness layer, which the model can't read.
  - A good harness lets cheap or open models (Qwen, GPT-OSS, small models) do work that would otherwise need frontier models. That is the cost lever.
  - Log every tool event into a trace. The verifier and your debugging both depend on it.
- **Outlook (his speculation):** 2025 was "agents", 2026 is "harnesses", and maybe next come agents that generate their own task-specific harness on the fly (like plan mode, but producing guardrails and verifiers). IBM's open-source OpenRAG is an example of an enterprise harness around RAG over sensitive data.
- **For Sam:**
  - Drive Button: wrap each LLM stage with code verifiers. After extraction, check every place resolves in GeoNames. After trip building, check every product ID exists in the supplier API with real availability and that dates/order are valid. Retry up to N, then fail honestly to a review queue rather than shipping a made-up trip.
  - Keep supplier credentials and booking calls in deterministic harness code; the model only proposes.
  - For millions of location rows on a budget, a small local or cheap model inside a strict harness (schema validation, allowed-label check, confidence/disagreement fallback to a bigger model) is the cost play.
  - Add max-steps and cost caps to any agent that browses or calls suppliers.

## Ben Holmes (Warp, AI Engineer) — Your AI Agent Is Hiding Its Mistakes From You (https://www.youtube.com/watch?v=pz74r-4dGfQ)
**Agents quietly work around broken setup and report success. Centralise their traces, grade every run with rubric-based LLM judges, and have a scheduled agent turn recurring failures into config/skill fixes. Track scores over time to prove each fix worked.**

- **Mental model.** In a "software factory" (task → triage → spec → implement → computer-use screenshots → adversarial review → monitor), the leverage is in *factory engineering*: improving the skills, config and environment that drive the agents. Modern agents are persistent. They hit out-of-date instructions or a misconfigured environment, brute-force their way around it (for example pinging port after port), and then just report "done". The only visible symptom is "took 2 minutes longer". So the mistakes are hidden unless you read the traces.
- **Mechanism: the self-improvement loop:**
  1. **Centralise traces.** Local agents keep logs in per-tool folders (~/.claude, ~/.codex). Some teams zip everyone's folders into a shared S3 bucket. Shared cloud runners or sandboxes get centralisation for free. Each trace must include three things: the **transcript** (human and agent messages), the **tool/MCP/file-system calls** (what it actually did), and the **output** (diff or merged PR).
  2. **Score with rubric judges.** Each scorer is a short markdown "skill": what it evaluates, labelled score levels describing success and failure, a pass/fail cut-off (for example 0.4 and below fails), and a concise rubric definition. Start with a couple of bullets and grow it; it's more art than science. Example scorers:
     - *Redundant tests:* too many tests, or the wrong kind versus the team's preferred mix (for example the team prefers e2e over unit).
     - *Verbosity:* over-commenting and over-long summaries. Review is the bottleneck, so how the agent communicates matters.
     - *Efficiency:* reads the tool calls and asks whether it went straight to the solution or spun its wheels.
     - Trigger: grade when a PR merges, one judge run per conversation. Sample at random to save tokens, or grade everything. The judge writes a pass/fail *plus the reason*.
  3. **Improvement agent on a cron.** For example every 24 h: take the last day's graded runs (say 200), filter to the failures (say 50), and give their failure notes plus access to the skills repo and codebase to an agent that finds common causes and **opens PRs** for humans to review. Examples: add a test-coverage-pyramid instruction; tell the e2e-testing skill how to find the port first; make the changelog skill specify the house language so the human stops correcting it by hand; add a missing config that blocked spawning sub-agents (it had been burning tool calls).
  4. **Dashboards over time.** Plot each score's trend and mark when a suggested change was merged. An inflection in the curve is evidence the fix worked. That is the difference from opaque "agent memory/dreaming", where you can't see what the agent concluded.
- **Practical tips:**
  - Grade in batches for token efficiency.
  - A one-off run over your existing history surfaces quick wins (Warp offers a "Skill Doctor" skill for this). Continuous scoring is better.
  - Keep a human reviewing the improvement PRs; the agent proposes and the team merges.
- **Failure modes:**
  - You only notice slowness, not the cause.
  - Out-of-date skills and instructions silently cost tokens on every run.
  - Without tool-call data in the trace you can't judge efficiency at all.
  - Memory-style self-improvement with no metrics can't be audited.
- **For Sam:**
  - Apply the same loop to his own Claude Code / Antigravity work: collect session logs and periodically have a judge grade them on efficiency (wasted tool calls, re-discovering the environment) and on following his conventions. Then patch CLAUDE.md and skills, which also feeds his skill-building project.
  - For Drive Button batch runs, store the transcript, tool calls and final trip per article, and grade a daily sample with binary judges. A daily job clusters the failures and proposes prompt/config changes. Keep a score trend chart annotated with prompt-version changes.
  - Look specifically for silent retries and workarounds in tool calls (for example repeated supplier API errors that the agent hid).

## Soumya Gupta & Jai Chopra (Uber, AI Engineer) — Building Closed-Loop Evals for a Multimodal Agent at Scale (https://www.youtube.com/watch?v=31GUkCBD-Uc)
**For a production agent pipeline, evaluate each stage with the metric that fits it (classifier metrics for the router, pass@K plus pairwise rubric judges for generation), anchor everything to a human-labelled golden set, and close the loop by sampling production, diagnosing mismatches and auto-tuning prompts/configs behind a benchmark gate.**

- **Use case.** Uber Eats AI enhancement of merchants' menu photos (huge scale: millions of new items, about 10k cities). Constraints: consumers distrust anything that looks AI-generated, so outputs must stay *faithful* (no added food), keep the merchant's brand, avoid homogenising the marketplace (the same prompt for every photo would collapse diversity), only improve *when needed*, ship safely, learn continuously, and be cost-efficient. Design stance: put agents between brittle rules and unconstrained creativity, giving them agency but within guardrails.
- **Pipeline:**
  1. **Image understanding + router.** A multimodal LLM describes the image (plus dish name and metadata) as structured output, grades it against a rubric, then decides enhance or skip. Skipping keeps the original.
  2. **Editing agent in a loop with a QA agent.** First generate an image-specific prompt (what exactly needs fixing). Then edit. Then a multi-dimensional QA gate checks plating, faithfulness, colour and so on. On failure, the QA feedback goes back into prompt generation. After K failed loops the image is **not published** (you accept a coverage hit rather than a quality risk).
  3. **Post-processing + final "publish-ready" QA** (policy and quality checks).
  4. **Log everything** in one *flat* JSON per item across all agents (they use Arize), so engineers and non-technical PMs alike can drill into cases or aggregate. Logging comes first: without it there is nothing to optimise.
- **Evaluating the router (a classifier).** Use a confusion matrix and precision/recall. For multi-way routing (for example a cheaper, lower-latency model branch) use an NxN confusion matrix. **The guardrail metric is recall**, because no bad image should slip through.
  - Example failures: a great cheeseburger photo sent for enhancement means compute spent for zero lift *and* a risk of degrading it. A photo showing 6 wings for a dish named "8-piece wings" was approved, so the editor may hallucinate 2 extra wings to match the description, which is a faithfulness failure. That second case is why the text/image mismatch must be caught at routing.
- **Human labels as the golden source.** Use a representative sample (geography, dish type, quality level) labelled by humans against *objective* guidelines, to cut labeller subjectivity and noise. Tune until the guardrail metrics pass, then ship.
- **Closed loop for drift (the "model loop"):** regularly sample production → humans label with the same guidelines → compare with the agent → an umbrella **diagnosis agent** finds which component is at fault → triggers **auto-tuning**.
  - The auto-tuner is a prompt optimiser built from two sub-agents. **Reflect** de-noises the mismatches and finds systemic patterns. **Synthesize** edits the agent config.
  - The candidate config is benchmarked on the golden set. If it passes, it is registered in a versioned **agent config store** and production picks it up on the next run.
  - It is config-driven with no human in the loop, but with observability, guardrails and fast rollback.
- **Evaluating generation:**
  - **pass@K**: pass rate within K edit iterations (it should rise with K as feedback accumulates). Example: sweet-potato fries rejected for unrealistic portion and plating, then passed on iteration 2.
  - **Pairwise comparison** of input vs output against criteria aligned with design, policy and legal (faithful, complete, natural, realistic...). The judge answers **yes / no / unsure**, and unsure means reject in production.
  - Failure examples: shrimp added (faithfulness); sauce removed (completeness); a plate occluding the sauce (object coherence / physical plausibility, which are base-model limitations they report back to the frontier-model teams); and **reward hacking**: after a creative edit was rejected, the agent over-corrected into a generic ceramic bowl. Many pixels changed but there was no meaningful improvement. Also: when the item count can't be verified (8 wontons?), unsure means reject.
- **Swiss-cheese model.** Overlapping QA layers are intentional. Redundancy lowers the chance a failure reaches production, and the final gate also flags what upstream gates should have caught.
- **More feedback loops,** all routed through a general **diagnoser** that decides which agent(s) to retune: internal dogfooding (thumbs up/down plus free text from merchants, design and other teams; flagged examples are replayed and benchmarked before a new config ships) and production business metrics (add-to-cart, conversion), sliced by geography, device and dish type so tuning can be per segment.
- **Rules of thumb:**
  - Pick one guardrail metric per stage, chosen by which error is costlier.
  - "Skip" and "don't publish" are first-class outcomes.
  - Bound self-correction loops with K.
  - Allow "unsure" and treat it conservatively.
  - Version configs and gate every change on a golden set.
  - Watch for judge-driven reward hacking (changes that are "safe" but meaningless).
- **For Sam:** this maps almost one-to-one onto Drive Button:
  - Gate/router = article classifier. Choose recall or precision as its guardrail deliberately; probably precision, since bad trips damage trust.
  - Extraction + trip builder = the generator, with a QA judge in a bounded retry loop. Then "don't publish" rather than ship a dubious trip.
  - Faithfulness judge = "every item in the trip is supported by the article text".
  - Keep a human-labelled golden set of articles, version prompts in a config store, and replay flagged examples before each prompt change.
  - Note the "6 wings vs 8-piece" lesson: catch input inconsistencies (article mentions a place the geocoder can't confirm) early, so later stages don't hallucinate to reconcile them.
  - For bulk location classification, a cheaper-model routing branch plus an NxN confusion matrix is the evaluation pattern.

## Michael Patterson (Coder, AI Engineer) — The Lethal Trifecta Is Already on Your Laptops (https://www.youtube.com/watch?v=abw_m-DRi1k)
**An agent that combines private data, untrusted input and an outbound channel will eventually be exploited. Break the trifecta architecturally (isolated environment, egress control, scoped credentials, model proxy, command firewall) rather than trusting the prompt.**

- **The lethal trifecta (Simon Willison's term).** An agent is dangerous when it has all three of: (1) **access to private data** (files, credentials, company IP, databases); (2) **exposure to untrusted content** (web pages, tickets, error logs, CI logs, docs, emails); (3) **the ability to communicate externally** (HTTP requests, posting, pushing), which lets it exfiltrate. (note: the speaker blurred legs 2 and 3 into "internet access both ways". The standard framing is the three above. Any two may be survivable; all three means injected instructions can steal data.) His claim: with all three present, a security incident is a matter of when, not if.
- **Attack mechanisms:**
  - **Prompt injection:** malicious instructions hidden in sources the agent trusts (error messages, support tickets, CI logs, web pages).
  - **Context poisoning:** bad data planted in docs or datasets the agent retrieves.
  - **Privilege escalation:** agents are usually over-permissioned "to get work done", so an injection can do anything those permissions allow (drop tables, delete repos, post data).
  - Real anecdotes: an agentic-coding user's production DB deleted by an agent within days; the OpenClaw advice to run it only on a separate Mac Mini.
- **Three defence pillars** (classic security applied to agents; agents aren't immune to the mistakes humans make):
  1. **Control data access.** Run agents in an isolated VM, container or cloud dev environment, not on the laptop that holds your files and credentials. Only data you deliberately put there is reachable, and destructive actions can't hit your machine.
  2. **Restrict external communication.** Egress allow-lists, VPC/VPN, firewall. Govern and log all traffic, and alert (or kill the VM) when the agent leaves the perimeter.
  3. **Lock down permissions.** Least privilege, tightly scoped short-lived credentials.
- **Architectural components:**
  - **Cloud dev environments (CDEs):** pre-built, consistent images (Terraform-defined) with dependencies already installed. This is better for agents too: an agent on a fresh box will start fetching dependencies from wherever, and the registry might be compromised.
  - **Model proxy / LLM gateway** between the agent and the provider: logs every call, can strip injections, and redacts sensitive data (for example card numbers) before it reaches the provider.
  - **Agent firewall** at process level: deny un-blessed commands (`gh repo delete`, `DROP TABLE`) by watching what executes.
- **Organisational realities:**
  - **Shadow AI.** If the company blocks agents, staff use personal Claude/ChatGPT subscriptions on their laptops with no oversight. At scale ("ninja AI"), many unsupervised agents break things before anyone notices.
  - Adoption needs CISO sign-off (they carry the blame), DevOps buy-in, and concrete workflows.
  - Platform teams need observability of every LLM call *without alert fatigue*, because too many alerts become noise and critical ones are missed.
  - Developers worry about tech debt, AI slop and accountability when an agent in YOLO mode does damage.
  - It's fine to go slow and design the architecture first.
- **Checklist:**
  - Map each agent against the three legs and remove at least one.
  - Sandbox execution.
  - Use an egress allow-list.
  - Scope credentials per task.
  - Route all model calls through a logging gateway that does PII redaction.
  - Deny-list or allow-list destructive commands.
  - Alert on perimeter breaches, tuned against fatigue.
- **For Sam:**
  - The Drive Button pipeline reads **untrusted web content** (blog articles can contain injected instructions). Keep that stage away from private data and from any write or booking capability: the LLM only emits structured data, and deterministic code validates it before any supplier call.
  - Never give an article-reading agent supplier API keys or customer data. Credentials stay in harness code (see the Harness talk).
  - Running Claude Code / Antigravity on his laptop, which holds Trameter data and credentials, with web access is exactly the trifecta. Prefer a container or VM per project, keep `.env` and keys out of the agent's reach, and use permission prompts for network or destructive commands.

## Luke Alvoeiro (Factory, AI Engineer) — The Multi-Agent Architecture That Actually Ships (https://www.youtube.com/watch?v=ow1we5PzK-o)
**Long-running multi-agent work succeeds through structure, not parallelism. The pieces: an orchestrator that writes a validation contract before any code, fresh-context workers run one at a time, adversarial validators that never saw the code, structured handoffs, and the right model in each seat.**

- **Premise.** The bottleneck is now human attention, not model intelligence. Engineers can supervise only a few tasks a day. Goal: a human decides *what*, and the system works out *how* over hours or days.
- **Taxonomy of multi-agent patterns:**
  1. **Delegation:** a parent spawns a sub-agent ("figure out the DB schema") and gets a result back. The simplest pattern and the one most people build first.
  2. **Creator–verifier:** a separate agent with fresh context checks the work. The creator is biased towards its own code working, which is why humans do code review.
  3. **Direct communication:** peers message each other with no coordinator. Hard: state fragments and there's no single source of truth.
  4. **Negotiation** over a shared resource (same API, same code region). Best when it is positive-sum.
  5. **Broadcast:** one agent sends status, context or constraints to all. Unflashy but essential for staying coherent on long tasks.
- **"Missions" architecture** (combines delegation, creator–verifier, broadcast and negotiation):
  - **Orchestrator:** scopes the goal with the user (asks strategic questions, surfaces unclear requirements), then produces a plan of features and milestones plus a **validation contract**.
  - **Workers:** each feature gets a fresh-context worker (no accumulated baggage or degraded attention). It reads its spec, implements, and commits to git so the next worker inherits a clean, working codebase.
  - **Validators:** after each milestone. A *scrutiny* validator runs tests, type checks and lint, and spawns a dedicated code-review agent per feature. A *user-testing* validator acts like QA: it launches the app and drives it via computer use (forms, renders, clicks, end-to-end flows). Most wall-clock time goes to this real-world execution, not token generation. Neither validator has seen the code, so validation is adversarial by design.
- **The validation contract is the key mechanism.** Tests written after implementation are shaped by the code, so they confirm decisions rather than catch bugs, and the system drifts. The contract is written at planning time, independent of implementation. For a large project it can hold hundreds of assertions. Each feature maps to one or more assertions, and together the features must cover all of them.
- **Structured handoffs.** A finishing worker fills in a form: what was done, what was left undone, commands run *with exit codes*, issues found, and whether it followed the orchestrator's procedures. Problems are caught at milestone boundaries and corrective features are scoped. The system self-heals by forcing agents to write things down rather than hoping they remember.
- **Serial over parallel.** Running about 10 agents in parallel on code caused conflicts, duplicated work and inconsistent architecture; coordination overhead ate the speed-up while burning tokens. So only one worker or validator runs at a time, with parallelism only for **read-only** operations (codebase search, API research, code reviews). Slower on paper, but error rates drop sharply, and correctness compounds over multi-day runs. The longest mission ran 16 days; they believe 30 is possible.
- **Right model per seat.** Planning needs slow, careful reasoning. Implementation needs fast code fluency. Validation needs precise instruction following. No single provider is best at all three. Put validation on a *different provider* to avoid shared training-data bias. You're only as strong as your weakest link, so model-agnostic design is an advantage. The structure (contracts, checkpoints) also lets open-weight models do well. Knowing how models fail and compound is a skill in itself ("droid whispering").
- **Production numbers (Slack-clone mission):**
  - About 60% of time and tokens go to implementation.
  - Validation never passes first time, so follow-up features are always needed.
  - The final code is about 50% tests with about 90% coverage.
  - Heavy prompt caching offsets the cost.
  - UX: a "mission control" view showing progress, budget burned, the active worker and handoff summaries.
- **Bitter-lesson-proofing.** Keep orchestration logic in prompts and skills (about 700 lines of text, where a few sentences can change the strategy), not a hard-coded state machine, so the system improves as models do. The deterministic code stays thin: bookkeeping, running validation, *blocking progress when handoff issues are unresolved*.
- **Open questions:** how to parallelise more safely, and how to orchestrate missions of missions.
- **For Sam:**
  - For big builds (Tereka, the Riselist site, pipeline refactors), copy the pattern with Claude Code sub-agents. First write acceptance criteria / an e2e assertion list before implementation. Use one implementer at a time with git commits per feature, and a fresh-context reviewer (ideally a different model, for example Gemini via Antigravity reviewing Claude's work, which fits his existing two-model setup). Require a handoff note with commands and exit codes.
  - For Drive Button, the "validation contract" idea maps to defining the trip-correctness assertions up front, independent of the prompt, so the extraction prompt can't shape its own tests.
  - On a 16GB laptop, serial execution with read-only parallelism is also the resource-friendly choice.

## Hamel Husain with Jacob Carter (NurtureBoss) — Error Analysis: The Highest ROI Technique In AI Engineering (https://www.youtube.com/watch?v=e2i6JbU2R-s)
**A founder's account of error analysis: reading thousands of real conversations in a custom viewer, writing notes, having an LLM categorise and count them. That one practice drove the product roadmap, exposed bugs, and set up targeted evals (date handling went from 33% to 100%).**

- **Context.** NurtureBoss builds AI leasing assistants for apartment communities: answering prospects, scheduling tours, pricing and availability questions, follow-ups, rent collection, reviews, maintenance requests. Channels are voice, email and SMS, all running on OpenAI models. After launch the product was a "black box". Jacob admits he was frustrated when Hamel kept telling him to "just look at the data". Then it turned out to be the most valuable thing they did.
- **What looking at data revealed:**
  - **It writes the roadmap:** you see what users ask for that the product can't do, which beats discovery interviews and journey maps.
  - **User mental models differ from what you built.** How users *assume* the product works never matches the design, so you surface bugs and needed adjustments.
  - **You learn the model's habits.** Seeing it answer the same kinds of question repeatedly builds intuition for how to design tools, RAG and prompts.
- **Mechanism: a custom data viewer.** Built in-house to show a thread as human message → AI reply → tool call → tool response. One click marks it good or bad, then a quick free-text note on why. An AI button then categorises the negative notes across all SMS threads and shows counts. Top categories: tour scheduling errors, rescheduling (merge into scheduling), handoffs not triggered (the AI should have escalated to a human). That alone is enough to file bug and feature tickets and direct engineering.
- **From analysis to evals: the date bug.** LLMs are bad at relative and absolute dates. "Two weeks from now" was resolved to the wrong date, and "March 1st" was searched in 2022. They built a set of date-reference test cases in Braintrust's playground, iterated on the prompt or tooling, and went from 33% to 100% correct. Hamel's point: this is a *real, specific pain point* found in the data, not a generic "conciseness" or "hallucination score".
- **Generic tools break on your specifics.** NurtureBoss conversations are **evergreen threads**: they run from prospect to resident to renewal with no clean end. Off-the-shelf eval tools that assume a completed conversation don't fit. You only notice these structural quirks by reading the data, and then you either build tooling or adapt existing tools.
- **The recipe (Hamel's summary):**
  1. Look at data in a viewer that removes friction and renders *your* domain properly.
  2. Write detailed notes about problems.
  3. Categorise the notes (an LLM can help) and count.
  4. Fix the most frequent and most severe issues.
  5. Then write targeted evals for the problems that persist.
- **Rules of thumb:**
  - Product owners and domain experts should do the reading, not just engineers.
  - Every insight should become an action item right away; error analysis is not an academic exercise.
  - Always inject the current date (and timezone) and test date parsing explicitly.
- **For Sam:**
  - Build a small viewer for Drive Button runs: article text side by side with extracted places, matched products and final trip, with a good/bad button and a note field. Use an LLM to categorise the notes, then count.
  - Date handling is directly relevant: blog posts mention seasons, "last summer", festival dates, opening hours. Pass the article publish date and today's date explicitly, and write a date-resolution test set.
  - Users' assumptions about Drive Button buttons (what "book this trip" means) will differ from the design. Read real click and booking logs to learn this.

