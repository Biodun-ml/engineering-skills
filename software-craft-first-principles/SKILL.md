---
name: software-craft-first-principles
description: First-principles code-craft playbook from 28 talks (Ousterhout, Hickey, Metz, Bernhardt, Rhodes, Fowler, Beck, Feathers, Rainsberger, Hettinger, Muratori). Use whenever writing, reviewing, refactoring or testing code, especially AI-written Python, pipelines and API adapters.
---

# Software craft from first principles

This skill turns 28 talks into working judgment for writing and changing code well: design,
naming, refactoring, legacy code, testing, idiomatic Python and performance. This file holds
the procedure, the principles the teachers keep coming back to, and checklists. `references/`
(when present) holds detailed notes for every video, each credited to its teacher.

Use it whenever code is being written or changed, even if the user doesn't say "design" or
"refactor": reviewing an AI diff, adding a supplier adapter, cleaning a messy script, deciding
how to test something, "should this be a class?", "is this abstraction worth it?", or "why is
this slow?".

**How to use it**
1. Follow the working procedure for any non-trivial change.
2. Run the relevant checklist (section 3) against the design, diff or test suite.
3. If a `references/` folder sits next to this file, use it for depth. `references/INDEX.md`
   lists every video with its line number, or run `grep -n -i "keyword" references/*.md`.
   Each section ends with a "Relevance to Sam" or "For Sam" note: Sam builds large
   location-data pipelines (GeoNames ~6M rows, matching/dedup), supplier API integrations
   (Viator, RateHawk, Coras) and an LLM pipeline that turns travel blogs into bookable trips,
   all on a 16 GB laptop, with AI writing much of the code. Use those parts only in that
   context. Lines marked "(note: …)" and those Sam notes are helper additions, not the
   speaker's words. If `references/` is missing, the full notes live in the user's private
   GitHub repo `Biodun-ml/engineering-skills` (folder
   `software-craft-first-principles/references/`); clone it when you need depth.
4. Explain the *why*, and credit the teacher when you use a specific pattern.
5. Some talks predate modern Python. Section 8 (reviewer addition, not from the talks) covers
   typing, concurrency, packaging, profiling, pytest and updated idioms as of late 2026.
   Items marked *(reviewer)* elsewhere are also additions, not the speakers' words.

---

## 1. Working procedure (writing or changing code)

1. **State the problem in writing.** Problem, facts and constraints (RAM, row counts, rate
   limits), question marks, prior art, and two or more candidate solutions with trade-offs.
   No question marks means you skipped a step. Sleep on important decisions: misconceptions
   are the costliest bugs and no test catches them. *(Hickey — Hammock Driven Development)*
2. **Design it twice.** "Suppose you're forbidden from doing what you just proposed": the
   second idea usually wins, for about 1–2% of project time. *(Ousterhout; Rhodes)*
3. **Choose the data shape first.** Parse and reshape external input into types built for
   your problem at the edge; the core then needs few conditionals. *(Rhodes citing Brooks;
   Schlawack; Arjan)*
4. **Lay out a functional core and an imperative shell.** Decisions in pure functions, I/O as
   thin glue at the top. Ask: can I call this without I/O happening? *(Bernhardt; Rhodes)*
5. **Design interfaces deep.** Write the interface docstring first; if it's hard to describe,
   the design is wrong. *(Ousterhout)*
6. **Tidy first, in a separate commit.** "Make the change easy, then make the easy change."
   Never mix structure and behaviour changes. *(Beck; Fowler; Metz)*
7. **Test behaviour at the public contract.** See it fail as predicted, dirty green fast,
   refactor with no new tests. *(Cooper; Farley)*
8. **Leave it better and delete what's dead.** Put what you learned into names and extracted
   functions. Code is a liability. *(Fowler; Ousterhout; Diederich)*
9. **Review generated code like a stranger's.** AI writes tactical code, not decomposition;
   it's "generate and test" and verification is yours. Use section 3. *(Ousterhout; Feathers)*

---

## 2. Principles that recur across the teachers

### Complexity and depth
- **Complexity is incremental**, so fight it continuously with ~10–20% extra effort, not
  rewrites. Strategic beats tactical; Ousterhout guesses (no data) payback at 6–12 months.
  *(Ousterhout; Fowler — design stamina)*
- **Judge modules by depth, not length.** Depth = functionality hidden ÷ interface a caller
  must learn (signatures, side effects, ordering rules). A shallow method costs more to call
  than to inline. "Split anything over N lines" breeds classitis; hundreds of lines behind a
  simple interface are fine. Combine entangled pieces into one deeper unit. Make the common
  case trivial (unlike Java I/O's three wrappers for buffering). *(Ousterhout)*
- **Define errors out of existence, sparingly.** Choose semantics where the "error" is normal
  (idempotent unset/delete, substring returning the overlap). Collapse exception types into the
  few reactions callers really have. Never ignore real failures, never map an unknown response
  to success, and sometimes crashing is right. *(Ousterhout)*
- **Simple is not easy.** Simple = not intertwined (objective); easy = familiar (relative).
  Judge the running artifact, not the typing. God classes and DRY-at-any-cost are easy, not
  simple. *(Hickey — Simple Made Easy; Schlawack)*
- **Coupling is the cost driver.** Two elements are coupled with respect to a change if
  changing one forces changing the other. Change costs follow a power law: a few changes
  cascade (1 → 4 → 75 → 4,000). Every coupling decision trades the cost of coupling against
  the cost of decoupling. *(Beck — Tidy First?; Cooper)*
- **Design economics.** Defer decisions that can wait (spending later is spending less);
  invest in structure that creates options, e.g. one internal supplier interface, since
  options are worth more under uncertainty. *(Beck)*

### Data, state and I/O
- **Hoist I/O, don't hide it.** Burying `requests.get` in a helper hides it but keeps every
  caller coupled to it, which is why people reach for `mock.patch`. Extract the logic
  (`build_url`, `pluck_definition`) and leave I/O as glue at the top. *(Rhodes)*
- **Functional core, imperative shell.** Core: many paths, no dependencies → many fast unit
  tests, no mocks. Shell: many dependencies, few paths → a few integration tests. "80%
  functional is a heck of a lot better than 0%." Build many small core/shell systems rather
  than one giant one. *(Bernhardt)*
- **Values are the best boundaries; state leaks.** Plain data can be tested, cached, queued
  and parallelised, so boundaries can move between classes, processes and machines. A method
  hiding mutable state that returns different results for the same input spreads complexity
  to every caller. Prefer frozen dataclasses and tuples. *(Bernhardt; Rhodes; Hickey)*
- **Verbs and nouns.** Most logic in stateless, idempotent functions (verbs). Stateful data
  (nouns) kept legible (printing shows all state) and shallow (methods are leaves of the call
  graph). Combine, don't mix. Keep call graphs shallow: helpers return data to `main`, which
  takes the next step. *(Rhodes — Skip the Design Patterns)*
- **Rules as data; queues between stages.** Gather business rules (match thresholds, mapping
  rules) into declarative tables instead of conditionals strewn about. If A calls B directly,
  A is tied to where and when B runs; put a queue or an intermediate Parquet file between
  stages. *(Hickey)*
- **Represent information as plain data** (dicts, sets, sequences, DataFrames, simple
  dataclasses) so generic transforms work everywhere. Pass data, not a handle to go find it
  (`process_payment(reference, amount)`, not `order_id` plus a repository). *(Hickey; Arjan —
  7 Smells #5)*
- **Make illegal states unrepresentable.** Separate types per state (open/paid/overdue
  invoice) instead of one class full of `X | None`. Don't let the API schema or DB schema
  become your domain model; explicit mapping between duplicate-looking per-layer types is
  fine. Receive API payloads as Pydantic discriminated unions; repositories return plain
  dataclasses. It depends: a pure pass-through lookup service needs no domain model, and a
  dict is the right shape there. *(Schlawack)*

### Objects, abstraction and duplication
- **A class with `__init__` plus one method is a function.** Use `functools.partial` for a
  bound first argument. Classes earn their place as mutable state plus several functions on
  it. Keep namespaces flat. *(Diederich — Stop Writing Classes)*
- **Duplication is far cheaper than the wrong abstraction.** Tolerate it (tag copies
  `# DUP-3`) until real cases reveal the right abstraction. When a shared abstraction starts
  sprouting special-case `if`s and parameters, inline it back into callers and re-extract later.
  Test concrete features, not the shared abstraction, so it can be inlined without deleting
  tests. *(Metz — All the Little Things; Abramov — The WET Codebase)*
- **Type-switch → factory plus polymorphism; mapping → configuration.** A conditional over a
  type code becomes a factory choosing a class plus one small class per variant; the
  name→class mapping is a dict (later YAML/DB). New behaviour means *adding* a class and a
  registry entry. Shared method prefixes/suffixes (`normal_tick`, `brie_tick`) signal a missing
  object. *(Metz)* Use this for *open* sets (suppliers keep arriving). For a *closed* set of
  states, a union of dataclasses + `match` + `typing.assert_never` is simpler and the type
  checker proves it exhaustive. *(reviewer)*
- **Inheritance is for specialisation, not code sharing** (only shallow, narrow, leaf-node).
  For variation, make variants alike to reveal what differs, name the *role*, inject players;
  the default is a player too. Two roles × two players gave four behaviours with less code.
  *(Metz — Nothing is Something)*
- **Null objects instead of scattered `None` checks.** Return a `MissingX` that answers the
  same messages; the one remaining conditional lives in a wrapper around the untrusted API.
  *(Metz)* Caveat *(reviewer)*: in pipelines a `MissingPlace` can flow silently into matched
  output. In typed code prefer `X | None` narrowed by the checker, explicit NA, or failing
  loudly; keep null objects for cases where "nothing" is genuinely safe (an empty list).
- **Inject dependencies; wire in `main`.** Depend on a narrow `Protocol` or `Callable` (never
  specifying `__init__`). Arjan wires manually in `main` ~99% of the time; containers only for
  config-driven wiring; FastAPI `Depends` on endpoints; plain calls in small scripts. No
  half-initialised objects: use a `create()` factory instead of "remember to `connect()`".
  *(Arjan — DI; 7 Smells)*
- **Wrap ugly external APIs in an adapter** that changes shape, not work: context manager for
  setup/teardown, properties, `__len__`/`__getitem__`, named exceptions, `__repr__`. He reports
  cutting client codebases from 50k to 10k lines this way. *(Hettinger — Beyond PEP 8)*
  Re-raise with `raise SupplierUnavailable(...) from exc` so the root cause survives;
  `exc.add_note()` (3.11) adds context without a new type. *(reviewer)*

### Naming, comments, layout
- **Name in the language of the domain.** No Lego names, filler words (Manager, Data, Info),
  `Exception` suffixes, stringly-typed primitives, reflexive get/set. Extract Method to name
  intent: `trader.can_view(portfolio)`. *(Henney)*
- **Names carry more weight in Python** (no type next to the variable). Parameter names are
  public API. Use keyword arguments where literals or same-typed positionals are unclear
  (`kw_only=True` or `*` to enforce); namedtuples/dataclasses over positional tuples.
  *(Hettinger; Arjan bonus smell)*
- **Comments say what code can't.** Interface comments first, then fields, then tricky spots
  and lessons from bugs. Delete comments that restate code; the author of unclear code
  explains it with the same muddled thinking, so try a better name or extraction first.
  A short "why" on a Chesterton's fence pays off. *(Ousterhout; Henney; Diederich)*
- **Let a formatter own layout** (`ruff format`, default 88), review the "gorilla" (API shape,
  naming, error semantics), and don't reformat others' code *(Hettinger)*. Layout must
  survive renames; avoid name-length alignment *(Henney)*.

### Refactoring and legacy code
- **Two hats.** Adding behaviour *or* restructuring, never both at once. Tiny steps, tests after
  each, undo when red. Found a bug while refactoring? Switch hats, fix, switch back. *(Fowler;
  Beck)*
- **Refactoring workflows:** TDD refactor step, litter-pickup, comprehension (move
  understanding into the code), preparatory, and long-term (agree a target, nudge toward it on
  every touch). Planned "refactoring tasks" are a smell. Justify cleanup by economics: change
  follows 80/20, so harden hotspots and leave untouched working code. *(Fowler; Feathers)*
- **Seam → dumb green → refactor → delete.** Characterization tests around the edges, route one
  path at a time to new code, make it pass with deliberately dumb code ("red is when you
  scramble toward green, not when you ponder the abstraction"), clean up on green, delete the
  old code without needing to understand it. Expect intermediate steps to look *worse*.
  *(Metz)*
- **Parallels keep design change interruptible:** add the new parameter/field/path, migrate
  callers, delete the old, deploying at each step. *(Beck)*
- **Legacy = code you can't confidently understand or change.** Build confidence with tests as
  tripwires, a type checker ("lean on the compiler"), deliberate breakage, and firewalls
  against non-local effects (inject values instead of reaching into globals). Scratch-refactor
  a throwaway copy to learn it. Dependency-breaking may look ugly; do it anyway. *(Feathers &
  Clausen)*

### Testing
- **Good tests are automated, fast, reliable, informative and focused,** and pass alone or in
  any order. Test code is real code: domain assert helpers apply. *(Batchelder)* Default
  runner: pytest (plain `assert`, fixtures, parametrize); see section 8. *(reviewer)*
- **The trigger for a test is a new behaviour, not a new method.** Test through the module's
  stable public contract (its ports); keep internals free to change. The "unit" is the test's
  isolation, not the class. Write no new tests while refactoring; delete probe tests of
  internals afterwards. *(Cooper)*
- **Test pain is design feedback.** Heavy setup, many mocks, several expectations, peeking at
  private state: the test is reporting a design problem. Fix the design, not the test.
  *(Farley; Rainsberger; Feathers; Schlawack)*
- **Integrated tests are a scam when used for basic correctness.** Paths grow as 2^n (500
  conditionals ≈ a 150-digit number) and suite runtime grows superlinearly; wider tests give
  weaker design feedback, so tangled code keeps passing. Use collaboration tests (stub
  queries, expect actions) plus contract tests (one per equivalence class proving each stub is
  real). An escaped bug means a missing contract test. Integrated tests stay in the thin
  "brave" edge layer. *(Rainsberger; Bernhardt)*
- **Mocks are fragile:** coupled to implementation and drifting from the real format. Prefer
  injected doubles, `create_autospec`/`spec_set`, and refreshed recorded fixtures.
  *(Batchelder; Bernhardt)* Coverage is a poor target: an org paid bonuses for 80% and got
  tests that asserted nothing. *(Farley)*
- **Name tests as propositions** ("an empty stack throws when popped"), one case each.
  *(Henney)* Thin, testable entry point: a plain `main` function. *(Batchelder)* Write it as
  `def main(argv: Sequence[str] | None = None) -> int` calling `parser.parse_args(argv)` and
  `raise SystemExit(main())`. Passing all of `sys.argv` makes argparse parse the program name.
  *(reviewer)*

### Idiomatic Python
- **Real bugs, not style:** mutable defaults, bare `except:` (traps Ctrl-C), manual file
  closing, `shell=True`, `type(x) == T`, `sys.path` luck instead of an installed package.
  *(mCoding)* A grouping generator needs a final `yield` after the loop or the last group is
  silently dropped; use a safe initial value, not a `None` sentinel. *(Rhodes)*
- **Iterate, don't index:** `for x in xs`, `enumerate`, `zip`, `reversed`, `for/else` instead of
  a found flag, two-arg `iter` for sentinel APIs. Sort with a key function (one call per item,
  1M for 1M items) not a comparison (~20M calls). *(Hettinger — Idiomatic Python)* Modern
  forms *(reviewer)*: `zip(strict=True)` raises on length mismatch instead of truncating;
  `itertools.pairwise` replaces `prev = item`; `while chunk := f.read(n):` works for text and
  bytes (two-arg `iter(..., '')` loops forever on a binary file, whose sentinel is `b''`).
- **Right structure:** `defaultdict`/`Counter`, `ChainMap` for layered config, `deque` not
  `pop(0)` (O(n)), `''.join` not `+=` (can go quadratic; CPython's in-place trick isn't
  guaranteed), `x, y = y, x + y` to avoid order bugs. *(Hettinger)*
- **Decorators and context managers separate admin from business logic:** `@lru_cache` on pure
  functions, `with` for files/sessions, `suppress(OSError)` over a racy `exists` (today:
  `Path.unlink(missing_ok=True)`). *(Hettinger)* Cache gotchas *(reviewer)*: arguments must
  be hashable; an unbounded `@cache` over millions of keys eats RAM, so set `maxsize`; never
  cache methods (keeps `self` alive, ruff B019).
- **Know what `@dataclass` writes:** an `__eq__` that only compares instances of the exact same
  class (it does **not** validate field types: `Place(lat="x")` is accepted; validate in
  `__post_init__` or with Pydantic at the edge), `__hash__ = None` when mutable,
  `default_factory` for fresh lists, `field(repr=False)` for secrets, `hash/compare=False` to
  key identity on stable IDs, `frozen=True, order=True` for sortable, set-able values,
  `__post_init__` for invariants (with `frozen=True`, set derived fields via
  `object.__setattr__`). Dataclass ~168 bytes vs namedtuple ~72 bytes per instance (3.7-era);
  `slots=True` (3.10+) closes most of the gap, but columnar still beats per-row objects at
  millions of rows. The same discipline applies to AI as a code generator.
  *(Hettinger — Dataclasses)*

### Performance vs "clean"
- **Measured cost of clean-code rules in hot loops:** per-element virtual dispatch ~35
  cycles/shape; switch on an inline struct ~24 (1.5×); lookup table `CTable[type] * w * h`
  ~3.5 (~10×); with a second property the table version is ~15× faster; simple AVX 20–24×.
  Adding requirements widens the gap. In hot loops use contiguous homogeneous data, lookup
  tables and fused passes; measure before arguing. *(Muratori — "Clean" Code)*
- **Python translation** (note: helper synthesis, not Casey's words): per-row objects and
  `df.apply` over millions of rows are interpreter-bound; vectorised column operations, lookup
  arrays (`np.array([...])[codes]`, `Series.map`, a join) and categorical dtypes are often
  10–100× faster and save RAM. Keep polymorphism at the boundaries (one call per supplier,
  file or batch).
- **Compute index structures once, then slice.** N parallel accumulators mean "the complexity
  of the code is outrunning the complexity of the problem". *(Rhodes)*

---

## 3. Checklists

**Code review checklist (use on every AI-written diff)**
- [ ] Does each new function/class hide something real, or is it a shallow pass-through? Any
      class that is `__init__` + one method, or `class X: pass`? *(Ousterhout; Diederich)*
- [ ] Is I/O at the top, with decisions in pure functions? Any fetch-and-compute in one place?
      *(Rhodes; Bernhardt)*
- [ ] Does the diff mix a refactor with a behaviour change? Split it. *(Fowler; Beck)*
- [ ] New abstraction or "generic" helper with only one or two real cases, or growing
      `if provider == …` branches? Prefer duplication or one adapter per variant. *(Metz; Abramov)*
- [ ] External data parsed into internal types at the edge? No supplier JSON or `| None`-heavy
      blobs leaking into core logic? *(Schlawack)*
- [ ] Errors: bare `except`, swallowed exceptions, unknown responses treated as success, a
      custom exception per file that nobody catches? *(Ousterhout; mCoding; Diederich)*
- [ ] Python bug list: mutable defaults, unclosed resources, `shell=True`, `type ==`, print
      instead of logging. Make them lint rules *(mCoding)*: ruff B006, E722/BLE001, S602,
      E721, T201, B019 (cache on method), B905 (zip without strict), DTZ (naive datetimes),
      PTH (os.path). *(reviewer: codes beyond the first three)*
- [ ] Type checker clean; no new `Any`, `cast` or `# type: ignore` without a stated reason.
      Money in integer minor units or `Decimal`, never float; datetimes timezone-aware.
      *(Feathers — lean on the compiler; Arjan — money; reviewer)*
- [ ] Names in domain words; no Manager/Helper/Data; verbs precise (`add_` vs `create_`);
      keyword args where positionals could swap. *(Henney; Arjan)*
- [ ] Restating comments, get/set pairs, exposed internals; objects building their own
      collaborators or needing a setup call; `self.a.b.c` chains. *(Henney; Arjan)*
- [ ] Row-wise loops, `.apply` or per-row objects over large data; N parallel lists;
      `+=` string building, `pop(0)`. *(Muratori; Rhodes; Hettinger)*
- [ ] Tests: liars (`is not None`), giants, mockeries (patching functions inside the module
      under test), inspectors, test-per-method. *(Farley; Henney; Cooper)*
- [ ] Hand-rolled `__init__/__eq__/__repr__` (use a dataclass); dead or commented-out code.
      *(Hettinger; Diederich)*

**Before refactoring**
- [ ] Is this code a hotspot (touched often, bug-prone, critical)? If it's stable and nobody
      touches it, leave it. *(Feathers; Fowler)*
- [ ] Is a real requirement driving this, not an imagined future? *(Metz)*
- [ ] Behaviour pinned down: characterization/golden tests on a sample of real inputs, at the
      public contract, not on internals. *(Metz; Cooper)*
- [ ] Target shape agreed (one sentence), and the first small step chosen. *(Fowler)*
- [ ] Tests green before starting; one hat on; commit per step; undo on red. *(Fowler; Beck)*
- [ ] For signature, schema or supplier-API changes: plan parallels (add new, migrate, delete
      old). *(Beck)*
- [ ] Unfamiliar code? Scratch-refactor a throwaway copy first; lean on the type checker;
      break something deliberately to see what fails. *(Feathers & Clausen)*
- [ ] Before deleting: union coverage from every entry point (CLI, cron, API server), grep
      for `getattr`/string lookups (keep dynamic dispatch next to its definitions), `git blame`
      for the fence's reason, a CI job that imports every module. *(Diederich)* Add `vulture`
      and ruff unused-code rules. *(helper note)*
- [ ] Tests coupled to structure? Park them, refactor with other confidence, rewrite them
      against the new structure. *(Feathers)*

**Testing strategy decision guide**
- Pure logic (parsing, matching, scoring, field mapping)? → Plain unit tests on small literal
  inputs, no mocks, many of them. *(Bernhardt; Rhodes)*
- Code that talks to a neighbour you own? → Collaboration test: stub queries (one test per
  class of answer: empty, one, many, error), expect actions (one expectation per test).
  Pair every stub with a contract test on the real supplier. *(Rainsberger)*
- Code that talks to the outside world (HTTP, DB, files)? → Keep the layer thin; a few
  integrated or recorded-response tests there, which double as learning tests of the
  provider's quirks. Refresh fixtures when the provider changes. *(Rainsberger; Batchelder)*
- A DB in developer tests? → Fine if it's fast and not shared between tests (fresh in-memory
  DB per test); avoid only for shared-fixture leaks or slowness. *(Cooper)* Use the same
  engine as production (testcontainers, or a transaction rolled back per test); in-memory
  SQLite differs from Postgres in types, JSON, upserts and locking. *(reviewer)*
- Need to know the whole thing is wired and in the right order? → A handful of end-to-end
  tests on the shell. Not for checking small things. *(Bernhardt; Rainsberger)*
- Legacy code? → Characterization tests at the edges first. Bug? → Failing test first.
  *(Metz; Feathers; Ousterhout)*
- Don't know how to go green? → Shift down: small probe tests of internals, then delete them.
  Implementation obvious? → Write the clean version directly and watch coverage for
  speculative branches. *(Cooper)*
- Test hard to write? → Change the design, don't add mocks or test-only APIs. *(Farley)*
- Never: coverage targets, one test per method, `mock.patch` deep inside your own code as the
  default, mocking the class under test. *(Farley; Cooper; Rhodes)*

---

## 4. Failure modes to hunt for

- Tactical tornado output; over-decomposition into entangled 3-line helpers. *(Ousterhout)*
- Another branch on a big conditional; premature DRY growing special-case parameters.
  *(Metz; Abramov)*
- A fat base class whose combination case breaks it (RandomEchoHouse). *(Metz)*
- Java-style scaffolding: one-method classes, empty bases, deep packages. *(Diederich)*
- Franken-types serving API + DB + domain; classes full of optionals. *(Schlawack)*
- I/O buried deep, forcing `mock.patch`; tests that only test the mocks. *(Rhodes; Cooper)*
- An integrated test added after every escaped bug. *(Rainsberger)*
- A six-month design freeze or planned rewrite instead of continuous movement. *(Beck; Fowler)*
- "Defining away" network failures by not handling them. *(Ousterhout)*
- Clean polymorphic objects in a per-row hot loop. *(Muratori)*

---

## 5. Where the teachers disagree, and how to resolve it

- **TDD as design.** Ousterhout rejects it (one test becomes the unit of design → tactical
  point solutions) but wants high-coverage tests; Farley and Cooper see test-first as the main
  design feedback; Hickey calls tests guard rails, not steering. *Resolve:* design the
  abstraction first, use test-first inside it and for bug fixes, and treat test pain as design
  feedback (all agree on that).
- **Mocks and the "unit".** Batchelder shows `mock.patch` (with warnings); Rhodes (quoted by
  Schlawack) calls monkey-patching "software bankruptcy"; Cooper mocks only expensive or shared boundaries
  and allows a fast, unshared DB; Bernhardt designs the need away; Rainsberger prefers simple
  doubles over hand-built fakes. *Resolve:* hoist I/O so the core needs no doubles; inject
  stubs plus contract tests at real boundaries; never mock your own internals.
- **Port-level vs core-level tests.** Cooper tests through ports; Schlawack wants an IO-free
  core tested directly. *Resolve:* make the domain core the stable contract where you can.
- **Assertions per test.** Farley and Rainsberger say one; read it as one behaviour.
- **Small methods vs deep modules.** Metz pushes small methods and objects; Ousterhout warns
  against "split over N lines". *Resolve:* judge by depth. Each Metz class hides a whole
  variant behind one message; tiny helpers you can't understand alone are not deep.
- **Plain data vs typed records.** Hickey: maps and sequences so generic transforms work;
  Schlawack: raw dicts are easy, not simple, so use precise per-state types. *Resolve
  (reviewer):* bulk data stays columnar (DataFrame/Arrow) and pass-through services may use
  dicts; anything that drives decisions (bookings, offers, match results) becomes a typed
  frozen record or `TypedDict` at the edge.
- **Polymorphism vs tables.** Metz and Hickey prefer polymorphism to switches; Muratori
  measures it 1.5–15× slower. *Resolve (notes' synthesis):* polymorphism at boundaries for
  change-heavy rules, table-driven data-oriented code in per-row hot loops.
- **Generality.** Ousterhout wants somewhat general-purpose interfaces even with one caller;
  Metz and Abramov wait for real cases. *Resolve:* general, simple *interfaces*; no
  speculative shared *implementations*.
- **Logic on data vs verbs.** Arjan moves totals onto dataclasses ("Tell, Don't Ask"); Rhodes
  keeps logic in verbs. *Resolve:* simple derived properties are fine; workflows stay in
  functions. Likewise Diederich's "two-method class is a function" vs Rhodes keeping classes
  for introspection: default to functions, classes for real state or inspectable config.
- **DI vs hoisting.** Arjan and Batchelder inject; Rhodes warns injection spreads into
  "kitchen sink" signatures when I/O sits deep. *Resolve:* hoist first, inject the few
  boundary collaborators at the composition root.
- **Custom exceptions and comments.** Hettinger wraps crummy errors in named ones, Diederich
  says built-ins until a caller must catch something specific: create one when a caller reacts
  differently, subclassing a built-in (`class MissingFare(KeyError)`). Ousterhout prizes
  interface comments, Henney and Diederich call most comments noise: keep interface docs and
  "why" notes, drop restatements.

---

## 6. Applying this to Sam's work

- **Supplier integrations (Viator, RateHawk, Coras):** one deep adapter per provider
  (`search`, `quote`, `book`, `cancel`) hiding auth, pagination, retries and mapping; pure
  `build_request`/`parse_response` tested on recorded JSON; HTTP client injected; output in
  Trameter state types (`Offer`, `ConfirmedBooking`); errors collapsed to retryable /
  unavailable / our bug; providers as role players in a registry dict, not `if provider ==`.
  Prices as integer minor units plus ISO currency (Arjan), never float; datetimes
  timezone-aware (`zoneinfo`), with the supplier's local date kept apart from the UTC instant
  *(reviewer)*.
- **Location and GTFS pipelines:** read → pure chunked transforms → write, with inspectable
  intermediate Parquet. Normalise names, dtypes and nulls once at ingestion; make stages
  idempotent; columns and lookup tables in hot paths, dataclasses only at the boundaries.
  Pure stages make process pools *possible* (Bernhardt), not free *(reviewer)*: send paths or
  chunk ids, not DataFrames; size workers to RAM; don't wrap Polars/DuckDB (already
  multithreaded) in a pool. Decide concurrency in one place on 16 GB (Hickey).
- **Legacy scripts (`locations_transform.py`):** golden outputs on a sample, then nudge toward
  pure core + I/O shell + typed records on every touch. Hotspots:
  `git log --since=1.year --format= --name-only | grep -v '^$' | sort | uniq -c | sort -rn | head -20`.
- **Working with AI:** two designs before code; refactor *or* feature per step; the dumb
  duplicated version before abstracting; section 3 checklist; weekly deletion.

---

## 7. Where to look in `references/`

| Need | File (see INDEX.md) |
|---|---|
| Ousterhout with Orosz (deep modules, design it twice, errors, TDD view), Batchelder testing basics and mocks, Hettinger Beyond PEP 8 (adapters), Metz All the Little Things (Gilded Rose refactor, duplication), Arjan DI, Muratori performance benchmark, mCoding 25 Python habits | `craft-1-*` |
| Ousterhout Google talk (strategic programming, define errors away), Henney seven habits (naming, comments, tests), Bernhardt Boundaries (functional core), Rhodes Clean Architecture (hoisting I/O), Metz Nothing is Something (null objects, composition), Fowler refactoring workflows, Farley TDD anti-patterns | `craft-2-*` |
| Cooper TDD (test behaviour at ports), Hettinger idiomatic Python, Feathers & Clausen legacy code, Schlawack Design Pressure (domain models, illegal states), Beck Tidy First? (coupling, parallels), Diederich Stop Writing Classes, Abramov WET Codebase | `craft-3-*` |
| Rhodes nouns and verbs (micro-patterns), Hickey Simple Made Easy, Rainsberger integrated tests (collaboration + contract), Hettinger dataclasses, Hickey Hammock Driven Development, Arjan 7 code smells, Diederich deleting code | `craft-4-*` |

---

## 8. Not covered by the sources, but know this (late 2026)

*Reviewer addition: none of this comes from the 28 talks. Verify versions before relying on them.*

- **Baseline.** Target Python 3.12+ (3.10 reached end of life in Oct 2026; 3.14 and 3.15 are
  current). 3.14: annotations are evaluated lazily (no `from __future__ import annotations`);
  Linux `multiprocessing` defaults to `forkserver`, so workers don't inherit parent memory
  (keep the `if __name__ == "__main__":` guard, pass paths not DataFrames); the free-threaded
  build is supported but optional. 3.15: `lazy import`, a stdlib sampling profiler
  (`python -m profiling.sampling`), UTF-8 as the default encoding.
- **Types: the "compiler" Feathers says to lean on.** Annotate every public function; run
  pyright (or basedpyright, or `mypy --strict`) in CI, strict on new modules. Astral's `ty`
  and Meta's `pyrefly` are fast but pre-1.0: editor aids for now. Use `X | None`, `list[int]`,
  `Protocol`, `TypedDict` for JSON-shaped dicts, `Literal`/`StrEnum` instead of magic
  strings, `type` aliases and `def f[T]` (3.12), `@override`, `match` + `assert_never`.
  Annotations aren't checked at runtime: validate untrusted input at the edge with Pydantic v2
  (discriminated unions) or msgspec, DataFrames with Pandera or Polars schemas. `Any`, `cast`
  and `# type: ignore` are the type-checker version of a mock: each needs a reason.
- **Concurrency rule.** Many network calls → `asyncio` + `httpx.AsyncClient`,
  `asyncio.TaskGroup` and `asyncio.timeout` (3.11+), a `Semaphore` for rate limits, retries
  with backoff and jitter, idempotency keys on bookings; never block inside `async def`
  (`requests`, `time.sleep`): use `asyncio.to_thread`. A few blocking calls →
  `ThreadPoolExecutor`. CPU-bound pure Python → `ProcessPoolExecutor` over chunks, workers
  sized to RAM (16 GB ÷ peak per worker). Polars, DuckDB and NumPy already multithread; don't
  nest a process pool around them.
- **Project layout.** One `pyproject.toml`, `src/<pkg>/` and `tests/`, managed with uv
  (`uv init --package`, `uv add`, `uv sync`, `uv run pytest`; commit `uv.lock`). Dev tools in
  `[dependency-groups]`, CLIs in `[project.scripts]`. One-off scripts: PEP 723 inline metadata
  and `uv run script.py`. Ruff lints and formats (replaces black, isort, flake8).
- **Profiling (measure before arguing, as Muratori says).** `perf_counter` per stage;
  `py-spy` or `profiling.sampling` to attach to a running job; pyinstrument or cProfile +
  snakeviz for call trees; `line_profiler` for one hot function; memray, `tracemalloc`,
  `df.memory_usage(deep=True)` for memory. Profile a representative sample, then double the
  input to check cost grows linearly.
- **pytest.** Fixtures over `setUp` (`yield` for teardown; `tmp_path`, `monkeypatch`,
  `caplog`; session scope only for read-only data). `@pytest.mark.parametrize(..., ids=...)`
  gives one case per equivalence class (Rainsberger); `pytest.raises(match=...)`,
  `pytest.approx`. `pytest-randomly` enforces "pass in any order" (Batchelder);
  `pytest-xdist` for speed. Golden outputs: syrupy or inline-snapshot. Recorded HTTP: respx,
  pytest-recording or vcrpy. Hypothesis for parsers and normalisers: idempotence
  (`f(f(x)) == f(x)`), round-trips, "no duplicate keys after dedupe", "row counts conserved";
  pin each found bug with `@example`. `mutmut` (mutation testing) finds Farley's Liars better
  than coverage does.
- **Hettinger's idioms, updated (his talk is 2013, Python 2).** Dicts keep insertion order
  (3.7+), so `OrderedDict` is rarely needed; `zip(strict=True)`, `itertools.pairwise`,
  `itertools.batched` (API batches); `while chunk := f.read(n):`; `d1 | d2`;
  `functools.cache`; `pathlib`; `raise X from err` and `err.add_note()`; `except*` with
  `TaskGroup`; `f"{x=}"` for debugging; `datetime.now(UTC)` + `zoneinfo` (`utcnow()` is
  deprecated). pandas 3 (Jan 2026) is copy-on-write: chained assignment
  (`df["a"][m] = v`) no longer writes.
