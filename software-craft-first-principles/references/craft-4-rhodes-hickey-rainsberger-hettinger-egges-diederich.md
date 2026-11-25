# craft-4 — cross-cutting principles

1. **Simple ≠ easy.** Simple means not intertwined, and that is objective. Easy means familiar or at hand, and that is relative. Judge code by the complexity of the running *artifact*, not by how pleasant it was to type. This applies doubly to AI-generated code that "looks familiar". (Hickey, Simple Made Easy)
2. **Verbs and nouns.** Put most logic in stateless, idempotent functions (verbs). Keep stateful data (nouns) *legible* (printing shows all state) and *shallow* (methods do one thing and return, as leaves of the call graph). Combine them; don't mix them. (Rhodes, Skip the Design Patterns)
3. **State leaks.** A method that hides mutable state but returns different results for the same input spreads complexity to every caller. Contain state behind a truly functional interface. Prefer values: frozen dataclasses, tuples. (Hickey, Simple Made Easy; Hettinger, Dataclasses)
4. **Represent information as plain data.** Use dicts, sets, sequences, DataFrames and simple dataclasses, not a bespoke class hierarchy per concept. Generic transforms over plain data are written once and reused everywhere. (Hickey, Simple Made Easy; Arjan, 7 Code Smells #1)
5. **Keep I/O at the top and call graphs shallow** (functional core, imperative shell). Helpers return data to `main`, and `main` does I/O and the next step. Burying I/O deep couples everything above it and forces `mock.patch`. A deep call graph forces you to thread new parameters through layers or use globals. (Rhodes)
6. **Separate domain behaviour from technology integration.** Mixing them is what pushes you into integrated tests. Put one thin "brave" layer at the edge that talks to the outside world, and only it gets integrated or learning tests. (Rainsberger, Integrated Tests Are A Scam)
7. **Collaboration + contract tests instead of integrated tests.** On the client side, stub queries and expect actions (one expectation per test). On the supplier side, prove every stubbed answer and every expected action really happens, with one test per equivalence class. When a bug slips through, look for the missing contract test. (Rainsberger)
8. **Test pain is design feedback.** Lots of mocks, long setup, several expectations per test all mean the interaction is too detailed or an abstraction is missing. Big integrated tests hide this, so tangled code keeps passing. (Rainsberger)
9. **Depend on small interfaces and inject dependencies.** Use a narrow `typing.Protocol` listing only the methods you call. Build concrete clients in `main` (the composition root) and pass them in. Objects shouldn't construct their own collaborators. (Arjan #7; Hickey on small interfaces and sub-components as arguments)
10. **Give functions the data they need, not a handle to go find it** (Law of Demeter). `process_payment(reference, amount)` beats `process_payment(order_id)` plus a repository lookup. It removes coupling and whole interfaces. (Arjan #5)
11. **Do one thing at a time and watch for code complexity outrunning the problem.** Split "find the boundaries" from "act on them". Compute index structures once, then slice or vectorise, instead of N parallel accumulators. Parallel lists in lockstep mean you need a record type. (Rhodes; Arjan #1)
12. **Decouple when/where with queues.** If A calls B directly, A is complected with where B lives and when it runs. Queues or intermediate files between stages fix that. Gather scattered business rules (the "why") into declarative data or rules instead of conditionals strewn everywhere. (Hickey)
13. **Use code generators, but know exactly what they write.** `@dataclass` gives an eq that only compares instances of the same class (note: it does not validate field types at runtime), `__hash__=None` when mutable, a good repr, `default_factory`, and `field(repr/hash/compare=False)`. The win is quality defaults more than saved lines. The same discipline applies to AI as a code generator: read what it produced and check that it's what you wanted. (Hettinger)
14. **Name things precisely.** If a method called `create_*` only appends, the responsibility is muddled. Use keyword arguments where same-typed positional arguments could be swapped silently. Use safe initial values rather than `None` sentinels. (Arjan #2, bonus; Rhodes micro-patterns)
15. **Make objects impossible to hold half-initialised.** Replace "construct, then remember to call connect()" with a factory (`classmethod create(...) -> Self`, async if needed). (Arjan #6)
16. **Most expensive bugs are misconceptions.** No test or type checker catches them. Write the problem statement, facts, constraints, unknowns ("question marks on the page"), prior art, and at least two candidate solutions with trade-offs, then sleep on it. A small solution is a strong sign it's right. (Hickey, Hammock Driven Development)
17. **Feed the background mind and manage load-in cost.** Think hard away from the screen, keep 2–3 projects in rotation (one per day), switch when stuck, and capture ideas when they arrive. Change your mind without apology when facts change. (Hickey, Hammock)
18. **Write it twice.** First solutions are rarely best. A quick second approach pays back within a year. (Rhodes Q&A)
19. **Code is a liability. Delete continuously.** Gather coverage across every entry point, delete stale scripts (they keep dead code trees looking alive), keep dynamic `getattr` lookups near their definitions, avoid metaprogramming cleverness, check Chesterton's fence via `git blame`, and keep a team-level (not individual) delete culture. (Diederich, Deleting Code Is Hard)
20. **Tests are guard rails, not steering.** Every field bug passed the tests and the type checker. Reliability tools are secondary to simplicity and to being able to reason about the program. (Hickey, Simple Made Easy; echoed by Rainsberger's design-feedback argument)

---

## Brandon Rhodes — Skip the Design Patterns: Architecting with Nouns and Verbs (https://www.youtube.com/watch?v=v-N6r8lcsNc)
**Modern Python is built from verbs (stateless, idempotent functions) and nouns (stateful, mutable, legible, shallow data), combined rather than mixed. Most code belongs in verbs.**

- **Mental model: three stages of a programmer.**
  1. *Procedural*: use nouns and verbs others designed (list, dict, datetime, zip, enumerate). A 50-line top-level pandas script that merges two CSVs can make you the office hero. A whole career can live here.
  2. *Abstraction*: invent new verbs (your own functions).
  3. *Encapsulation*: invent new nouns (attach behaviour to data). This is the hardest and riskiest. Classic OOP teaching starts beginners here, which is backwards.
- **Definitions.**
  - Verb = stateless (stores nothing between calls) + idempotent (same input gives the same answer every time). A class with `__init__(pattern)` + `__call__(msg)` (like a logging Filter) is still a *verb*. It is a function configured at runtime.
  - Noun = stateful, mutable data. Good Python nouns are **legible** (print/inspect shows all of its *state*, though it may hide its *implementation*, like dict hides its hash tables) and **shallow** (each method does one operation and returns. It is a leaf of the call graph and never starts a cascade across other objects).
  - Encapsulation begins not when you type `class` but when a plain data structure (dataclass, Django model) gains its first behaviour method.
- **Why nouns must be simple:** when you read a .py file, the code is on the page but the data lives only in your head. You simulate the list or dataframe as you read. Brooks: show me your data structures and I won't need your flowcharts. Data should be the easy half.
- **Classic OOP, in its academic form, banned standalone verbs.** Every line attached to the nearest noun. The result is "method ping-pong" between classes, and subclass overrides scatter behaviour across many files. Many GoF patterns were remedies for that mistake. When the mistake goes away, the patterns go away too. (Even GoF said favour composition over inheritance.)
- **The pinnacle: an API.** Wrap a whole subsystem of verbs and nouns so it looks like one legible, shallow noun (pandas DataFrame, requests Session plus Response objects). In practice it is a small constellation of cooperating objects.

**Worked example: plot sensor time series with gaps for missing data.**
- First design `split(t, data)` yields (x, y) segments. Called twice for temperature and humidity, it scans the time array twice (duplicate work). To generalise to N data lists you would need nested loops, N accumulators and N appends: *"the complexity of the code is outrunning the complexity of the problem."* Then say "Nope".
- Fix with **do one thing at a time** (from Hettinger): `segments(t)` only finds `(start, end)` index pairs. Main then *slices* each array with `t[i:j]`, `temp[i:j]`, `hum[i:j]`. He had been reimplementing slicing with a thousand `append` calls. Gap detection runs once, and any number of series reuse it.

**Micro-patterns (daily habits that matter more than GoF patterns):**
1. Hoist constants out of loops (build `timedelta(minutes=1)` once at module level). He optimises early when waste distracts him.
2. Remember the previous item: assign `prev = item` at the bottom of the loop.
3. **Safe initial value, not a sentinel.** `prev = t[0]` and not `prev = None`. A None sentinel forces an `is not None` check on every iteration and gives you an `Optional` type, all to special-case the first pass.
4. Grouping generators need **two yields**: one at each break and one after the loop for the final group. Forgetting the last one silently drops data.
5. Avoid duplicate work. Notice it early.
6. Do one thing at a time. Produce an intermediate data structure (indices) that feeds the next step.

**Abstraction habits (when and how to extract a function):**
- **Don't mix levels of abstraction.** Big outward I/O calls (`go.Scatter`) should not sit next to loop-index fiddling.
- **Keep I/O near the top (main) and don't bury it.** If A→B→C→I/O, the I/O is hidden but still *coupled*. You cannot test A without the I/O running, and so people `mock.patch` it. Reaching for mock.patch is the symptom of that coupling. (This is the functional core / imperative shell, a.k.a. clean or hexagonal architecture. He points to the *Cosmic Python* book.)
- **Shallow call graphs.** Let helpers return data to main and have main take the next step. If a new requirement such as a configurable gap interval arrives, main just passes it in. In a deep graph you either thread an arg through layers that don't care about it, or you use a global ("spooky action at a distance").
- Write it twice. First solutions are rarely the best. Trying a second approach usually pays back within a year.

**Classes vs closures (nuancing Jack Diederich's "Stop Writing Classes"):** a two-method class (init + call) is a verb, and could be `functools.partial` or a closure. But class instances with public attributes allow **introspection**. His `logging_tree` package walks loggers, filters and handlers because they are plain attributes. That is a valid reason to keep a class.

**Relevance to Sam:**
- In data scripts, compute index or boundary structures once (masks, group boundaries) and then slice or vectorise. Don't append row by row into several accumulators.
- Keep supplier HTTP calls and file I/O in the top-level orchestrator. Have pure functions that transform payloads into records, which makes them testable with no mocks.
- Make your domain records legible dataclasses with shallow methods, and put the business logic in functions.
- When AI-generated code starts carrying N parallel accumulators or deep helper chains, that is the "complexity outrunning the problem" signal to redesign.

## Rich Hickey — Simple Made Easy (https://www.youtube.com/watch?v=SxdOUGdseq4)
**"Simple" (not intertwined) and "easy" (near at hand or familiar) are different things. Optimise for simple artifacts, not easy authoring, or complexity will slow you to a crawl.**

- **Definitions (from word roots).**
  - *Simple* = one fold or braid: one role, concept, task or dimension, with no interleaving. It is **objective**: you can look for entanglement. It is not about cardinality. Many small things hanging straight is simpler than two things in a knot. An interface with several operations can still be simple.
  - *Easy* = lie near. It has three senses: near at hand (installed, available), near our understanding (familiar), and near our capabilities. Easy is **relative** (easy for whom?). Saying "simple" when you mean "I already know it" kills objective design discussion.
  - *Complect* = braid together (bad). *Compose* = place together (good).
- **Construct vs artifact.** We judge tools by the experience of typing them (fewer characters, familiar syntax). Users and maintainers live with the *running artifact*: does it work, can we rely on it, can we fix it, can we change it. Judge constructs by the complexity their artifacts yield. Employers also favour "easy" because it makes programmers replaceable.
- **Limits of understanding.** We can hold only a handful of things in mind, like a juggler with 3 to 12 balls. Every intertwining forces you to pull another thing into your head, so the burden grows combinatorially. You can learn more, but you can't get much smarter. You have to bring the problem closer by simplifying it.
- **Changing and debugging require reasoning** (informal, not proofs). Every bug in the field passed the type checker and all the tests. Tests are "guard rails": useful safety nets, but they don't steer you anywhere and don't create simplicity. Refactoring tools won't make a knitted castle as changeable as a Lego castle.
- **Speed over the long run.** Focusing on ease lets you sprint at the start, but complexity accumulates until each sprint achieves less and mostly redoes old work. Focusing on simplicity costs some up-front thinking and then keeps velocity up. Calling every 100 metres a new sprint doesn't change that.
- **Incidental complexity** comes from tools and choices, not the problem ("incidental is Latin for your fault"). **Environmental / inherent complexity** (GC, CPU and memory contention, thread pools) is not your fault, but policies like "size my pool to the number of cores" don't compose when everyone does it. Those decisions belong to whoever has global information.

**Complex → simpler toolkit:**
| Complex | Simpler | What gets complected |
|---|---|---|
| State, objects | Values | Objects mix state, identity and value. State mixes value and time. |
| Methods | Functions, namespaces | Function plus state, and the class as a poor namespace |
| Variables | Managed references | Value and time |
| Inheritance | Polymorphism à la carte (protocols, type classes) | Types with each other |
| Switch / pattern matching | Open polymorphism | Who acts and what happens, in one closed place |
| Syntax | Data | Meaning and order |
| Imperative loops, fold | Set functions | What vs how (fold also implies order) |
| Actors | Queues | What vs who |
| ORM | Declarative data manipulation (SQL, LINQ, Datalog) | Logic and representation |
| Conditionals everywhere | Rules systems | Policy and program structure |
| Inconsistency (eventual) | Transactions + values | Things that stand apart |

- **State is poison that leaks.** A method that hides a variable but returns different results for the same args has *not* contained the complexity. It spreads to every caller. The only containment is to wrap it so the outside is truly functional (same input, same output). This is about understanding programs, not concurrency. Languages where immutability is the default end up with orders of magnitude less state, because most of it was never needed.
- **Modularity is not simplicity.** Separate classes or modules can still be completely complected through hidden assumptions ("this presumes that never returns 17"). What matters is what each component is *allowed to know about*. Partitioning and layering are *enabled* by simple components but don't create simplicity. Don't be fooled by code organisation.

**Designing simple things (who / what / when / where / why), with the attitude "I don't know, I don't want to know":**
- **What** = operations. Name *small* sets of function specifications (interfaces or protocols), much smaller than typical Java interfaces. They should take and return only values and other abstractions. Don't complect what with *how*, whether directly (a concrete class) or subtly (semantics that force an implementation, like fold's ordering). Separate what from how so the how can be handed to a DB or logic engine.
- **Who** = entities and data. Build components from sub-components passed in as arguments (dependency injection), not hardwired. Expect *more* sub-components than you have. Finding five components for a job is a win.
- **How** = implementation. Connect implementations via polymorphism, not switch statements, and keep them as islands.
- **When/where**: if A calls B directly, you have complected A with where B is and when B runs. **Put a queue in between.**
- **Why** = policy and rules. They are usually strewn through the app as conditionals. Gather them into a declarative or rule system that someone can read.
- **Information is simple. Don't ruin it.** Represent data as data, using maps, sets and sequences directly. Wrapping every new piece of information in a bespoke class (or an ORM) blocks generic reusable data-manipulation code and ties logic to representation. Objects were meant for encapsulating I/O devices, not information.
- Use data for communication too. Don't make everyone write parsers for ad hoc strings.
- **Simplifying someone else's code** = disentangling: follow the threads and label everything.
- **Simplicity is a choice** that needs vigilance and "entanglement radar": the ability to see interconnections between things that could be independent. Simplifying often yields *more* things, and that's fine.

**Relevance to Sam:**
- Treat records from suppliers (Viator, RateHawk) as plain data: dicts, dataclasses, DataFrames. Write generic transforms over them instead of a class hierarchy per provider.
- Separate *what* (a small `SupplierClient` protocol: search, get_product) from *how* (each provider adapter).
- Pull business rules (matching thresholds, mapping rules) out of nested conditionals into data or config tables.
- Decouple pipeline stages with queues or intermediate files (Parquet) instead of stage A calling stage B.
- Don't size every worker pool to the core count independently. Have one place that decides concurrency on the 16 GB machine.
- With AI-written code, judge output by entanglement, not by how familiar or short it looks.
- (note: Clojure-specific constructs aside, the Python equivalents are frozen dataclasses or tuples for values, `typing.Protocol` / `functools.singledispatch` for à-la-carte polymorphism, and `queue` / multiprocessing queues.)

## J.B. Rainsberger — Integrated Tests Are A Scam / Beware the Integrated Tests Scam (https://www.youtube.com/watch?v=fhFa4tkFUFw)
**Don't rely on integrated tests to check the basic correctness of your code. Use micro tests (collaboration + contract tests) for that, and system tests only for emergent, whole-system behaviour. Push integrated tests down to one thin layer at the edge.**

- **Definition.** An integrated test runs several "interesting" parts together. Values (numbers, strings, timestamps, in DDD terms) are not interesting. *Services*, especially ones that wrap outside technology (DB, web service, files), are.
- **Root cause.** Domain behaviour and technology integration live in the same function, class or module. That pushes you toward integrated tests. Separate domain logic from integration code and most of the system becomes micro-testable.
- **The scam (a vicious loop).** All tests pass but there's a bug, usually in the *interaction* between modules rather than in a calculation. The reflex is "fill the holes with integrated tests". But bigger-scope tests give weak **design feedback**: tangled code with wrong-way dependencies still passes. So the code gets more tangled, and tangled code is *harder to isolate* ("hardwired dependencies"). That means fewer micro tests, more holes, and more bugs. The remedy feeds the disease. It grows exponentially: it seems fine for a long time, then suddenly buries you.
- **Softened stance.** Integrated tests aren't evil. The trap is *over-relying* on them under deadline pressure, especially if you also want TDD's design feedback. Those two goals conflict.

**The replacement: collaboration tests + contract tests, one pair of layers at a time.**
- Put an interface between client and supplier. In Python that can be a Protocol, an abstract base class, or simply a function parameter. A lambda is a one-method interface.
- **Collaboration tests (client side, "do I talk to my neighbours correctly?")** use test doubles and run fully in memory, so they are fast and focused.
  - **Stub queries.** "Assume the repository returns an empty list, one customer, a handful" gives one test per class of answer. That beats tricking a real DB or API into producing each case.
  - **Expect actions.** For side effects (send message, write row, fire event), assert only that the right command went out at the right time. Don't inspect downstream effects.
  - **Gotcha:** treating every interaction as an action to verify is what makes mock tests brittle and mirror the implementation. Stubbed queries need no verification.
  - **Rule:** one assertion or expectation per test as a starting point. Needing several expectations hints that the interaction with the next layer is too detailed, i.e. a missing abstraction.
  - Painful test setup (many mocks, long arrange) is design feedback that the interaction is too complicated.
- **Contract tests (supplier side, "do I actually behave as my clients assume?")** check the *rules of behaviour*, not only the signature: what's returned when, null vs exception, which exception, what the value contains (pre/postconditions, invariants). Static types can encode some of it, e.g. Optional/Maybe.
  - Fixes **test-double drift.** Every stub ("foo may return None") needs a matching contract test showing *when* the real implementation returns None. Every expected action needs a test that really executes it.
  - Cover each **equivalence class** of responses (5 classes means at least 5 tests). One representative value per class is enough.
  - Design feedback: if the supplier can never actually produce an outcome the interface allows, remove it from the contract, and the clients get simpler.
  - Matching stubs to contract tests is a manual review step. Nothing automates it.
  - New diagnostic: when a bug slips through, look for the **missing contract test** instead of adding an integrated test. ("Two unit tests, no contract test.")
- **Prefer test doubles (mocks or stubs, or functions returning hard-coded values) over hand-built in-memory fakes.** Fakes grow until they need their own tests, and then they're production code. Doubles state the contract in the fewest words.
- **Recursion of the tree.** The system is framework → your code → libraries. Apply collaboration + contract pairs at each layer and the same trick works all the way down.
- **Where integrated tests remain:** only in the last "brave" layer that talks to the "horrible outside world". Those are really *learning / characterisation tests of the third-party thing*, not tests of your app. For plain values in memory, just write simple assert-equals tests.
- **Keep that layer thin as the system grows.** When several features talk to the same external service, remove duplication. Either (a) the shared part is the I/O, so extract it behind an interface, or (b) the differing parts are the I/O and they converge on one interface. Either way the integrated-test boundary moves closer to the outside world, and more code becomes micro-testable.
- **System tests** are still right for checking emergent behaviour and whether you chose the right parts. Just don't use them to check something small.

**Relevance to Sam:**
- For each supplier (Viator, RateHawk, Coras), build one thin client layer that does HTTP plus auth and returns parsed domain values. Integrated or recorded tests (VCR-style fixtures, sandbox calls) live only there, and they double as learning tests of the provider's quirks.
- The booking flow and matching logic are tested with stubs: "supplier returns no availability / one rate / rate with price change / raises timeout".
- Contract tests on the adapter prove when it really returns empty, raises `RateLimited`, and so on. Each stub in the booking tests must map to one of those.
- In pytest, a stub can be a lambda or a small fake object passed as a parameter, with no `mock.patch` needed if dependencies are injected.
- Use AI to draft the equivalence-class lists for each provider response, then verify each one has both a stub and a contract test.

## Raymond Hettinger — Dataclasses: The code generator to end all code generators (https://www.youtube.com/watch?v=T-TwcmT6Rcw)
**`@dataclass` is a code generator for data-holder classes. It saves a few lines but, more importantly, builds in best practices most people forget (type-checked `__eq__`, safe hashing, good repr). To use it well you must know exactly what code it writes for you.**

- **How to evaluate any code generator.** Learn how to use it, then what code it writes, then whether that is what you actually wanted, then whether the time spent learning and teaching it beats the time saved. You read and debug code far more than you write it. An abstraction hides things, which hurts debugging, but a well-tested generator also removes a whole class of hand-written bugs. Teams that only ever use dataclasses may never learn what `__init__`, `__repr__` and `__hash__` do. That's an education problem, cheap to fix with a lunch talk.
- **Two world views, both valid.** (1) A *data holder*, like a struct or a mutable namedtuple with defaults. (2) A *class writer* that removes boilerplate so you can focus on methods and business logic.
- **Lineage:** dicts / tuples / hand-written classes, then namedtuple, `types.SimpleNamespace` (very light), ORMs (Django, SQLAlchemy: field specs generate structure), traits / traitlets (validation via class variables), PEP 526 variable annotations (`x: int`), and the `attrs` package, the direct inspiration.

**Basic case: `@dataclass class Color: hue: int; saturation: float; lightness: float = 0.5`.**
- You get: `__init__` with annotations, `__repr__` using `__qualname__` (better than most hand-written ones), field access by name, `dataclasses.replace(c, hue=...)` (copy with changes), `asdict()` (a plain dict), and `astuple()`.
- **Generated `__eq__` checks `other.__class__ is self.__class__`**, so it refuses to compare apples to oranges (unlike namedtuples or IntEnums, where two unrelated `1` values compare equal).
- **Sets `__hash__ = None` because you defined eq.** Otherwise it would hash by identity while comparing by value, a nasty surprise. Default dataclasses are mutable and unhashable, so they can't be dict keys or set members.
- It also leaves introspection data: `__dataclass_params__`, `__dataclass_fields__` / `fields()`, including metadata. One quirk: a default value becomes a class variable as well as an instance attribute.
- Net: about 4 lines written for about 10 generated. The *quality* win matters more than the line count.

**Dataclass vs namedtuple (3.7-era numbers on his machine):**
| | dataclass | namedtuple |
|---|---|---|
| Mutability | mutable by default | immutable |
| Hashable | no (by default) | yes |
| Iterable/unpackable | no (use `astuple`) | yes |
| Ordering | none by default | sorts like tuples |
| Storage | instance `__dict__`, ~168 bytes | tuple, ~72 bytes |
| Attribute access | ~33 ns | ~61 ns (he planned a C speed-up) |
| Helpers | module functions `replace`, `asdict` (not visible in `dir`) | `_replace`, `_asdict` methods (discoverable). The underscore only avoids clashing with field names; they are not private. |

- **`@dataclass(order=True, frozen=True)`** adds type-checked `<, <=, >, >=` (better than `functools.total_ordering`, which lacks the same-class check), immutability via `__setattr__`/`__delattr__` that raise for fields but call `super()` otherwise, and `__hash__` over all fields. Now instances can be sorted and deduplicated in a set. For purely frozen records, namedtuple gives all this in C for free and is often enough. Dataclasses win when you need a *mix*: partially mutable, selectively ordered.
- **Rule:** if you don't ask, you get what you usually need. The danger is assuming a default you didn't get. Tests will catch it.

**Advanced field control (Employee example):**
- `field(default_factory=list)` gives a fresh list per instance, avoiding the shared mutable default bug. Factory fields are excluded from the hash automatically.
- `field(repr=False)` hides sensitive (salary) or noisy (long access log) data from the repr, which keeps debugging output readable and avoids leaking secrets into logs.
- `field(hash=False)` excludes fields that change over time (age, salary) from the hash. `compare=False` excludes non-orderable or irrelevant data from ordering and eq.
- `field(metadata={...})` gives you a data dictionary, readable with `fields()` (e.g. currency units).
- `unsafe_hash=True` forces `__hash__` on a mutable class. The name is a warning: read the docs before using it.
- Ordinary methods are just written in the class body.

**Gotchas and limits (as of 3.7):**
- The generated `__init__` doesn't call `super().__init__`. Use `__post_init__` for extra init or validation.
- Hard to extend an immutable parent's `__new__`. Trying to rebuild namedtuple from dataclasses gains nothing.
- (note: `__slots__` with defaults and keyword-only fields were limitations at the time. Python 3.10 added `@dataclass(slots=True, kw_only=True)` and `field(kw_only=True)`, so both are now solved. Validators are still not built in; use `__post_init__`, attrs or pydantic.)

**Relevance to Sam:**
- Use frozen dataclasses (with `slots=True` on 3.10+) for normalised location or supplier records. They give value semantics, hashability for dedupe and dict keys, and a readable repr when debugging matches.
- Use `repr=False` for API keys or tokens in client config objects.
- Use `compare=False`/`hash=False` to define identity on stable keys only (e.g. `geoname_id`, `provider_product_code`).
- For millions of rows, don't create a dataclass per row in hot loops. Keep the data columnar (pandas, Polars, Arrow) and use dataclasses at the boundaries, for config, API payloads and small result sets. The bytes per instance in the table above add up fast.
- Use `__post_init__` for cheap invariant checks on data coming from providers.
- When AI writes classes with hand-rolled `__init__`/`__eq__`/`__repr__`, convert them to dataclasses for the quality defaults.

## Rich Hickey — Hammock Driven Development (https://www.youtube.com/watch?v=f84n5oFoZBc)
**The cheapest place to fix bugs is design. Most big software failures are misconceptions, not implementation errors, and solving problems well is a practicable skill that relies on deliberately loading your "background mind" with hard, written-down thinking, then stepping away.**

- **First principle.** Tests, type systems and QA catch *implementation* defects. They can't tell you whether you had a good idea or are solving the right problem. So spend real effort on "step one", the thinking before building, even though 90s-style analysis and design (comprehensive plans, waterfall, diagrams) was rightly discredited.
- **A minimal definition of analysis and design:** (1) identify the problem, (2) assess whether the proposed solution actually solves it. Nothing more.
- **Solve problems, not feature lists.** A feature is just an attribute ("the chrome knob, not the purpose of the car"). Customer feature lists, despite their best efforts, often don't solve their real problem and can add new ones. Dig under them and reconcile. Also, **avoiding** a problem with a clever workaround is not the same as **solving** it.
- **Problem solving is a practisable skill** (reference: Pólya, *How to Solve It*). You get good at what you practise. Practising general problem solving has more leverage than mastering any particular methodology. Unlike maths, software has no proof that your solution is good, so judgment matters more.

**The process (an experience report, not a methodology):**
1. **State the problem explicitly, out loud or in writing.** "The problem is X, therefore Y." Lots of systems get built where nobody ever wrote down what problem they solve. Saying it out loud is a mnemonic, the seed of solving it.
2. **Understand the problem.** Write down:
   - the *facts*: requirements, context, constraints (hardware, runtime, power, user count);
   - the *known unknowns*: where does the input data come from, what happens when the main source is down;
   - your awareness that unknown unknowns exist.
   - Watch for missing reasoning: "we have problem P, so we need a NoSQL DB" skips *which characteristics* of P lead to that solution space. That gap is where the interesting work is.
3. **Look at other solutions to similar problems.** You are almost never the first. Studying them puts you one incremental step above prior work instead of starting from scratch. Read narrowly (same problem) and broadly (related problems, even ACM papers, even if you understand only a fraction).
4. **Be critical, especially of your own solution.** Hunt for technical errors, logic errors, and errors of taste or abstraction, and fix them up front. Tear apart prior art in your head. You often find the next idea by "crucifying" the last one.
5. **Real trade-offs need at least two candidate solutions** with pros and cons for each. "Trade-offs" usually just means the parts of my design that suck. Write the comparison down.
6. **There should be question marks on the page.** If your design notes have no open questions, you skipped the "what don't I know" step. His design sheets look like a "document of despair" full of can't-do-this and unknowns. That's positive: it names the challenges.
7. **Get lots of input.** Synthesis needs raw material. A narrow slice of "what I deliver next week" gives too few inputs.

**The mind model:**
- **Waking mind** is analytical, critical and tactical. It's great at local decisions and finding the local maximum, and bad at jumping off the current track to a higher hill. Its other job is to sanity-check ideas the background mind produces.
- **Background mind** (sleep being the main instance) is good at synthesis, strategy, analogy and finding hidden relations, which is where abstractions come from. Abstraction is *strategic* (it must be right in many contexts you can't foresee), so it needs this mode.
- **You must feed the background mind.** It only works on things you thought hard about while awake. Ten minutes of thinking and then a movie won't get solved overnight. Hard daytime thinking makes the problem an "agenda item".
- **Working-memory limit (7±2).** Problems have more parts than that. Write everything down (problem, facts, unknowns, prior art, sketch of solution) and survey it repeatedly, swapping different subsets into your head, like a juggler whose assistant swaps ball colours.
- **"Mind's eye time":** step away from the computer, close your eyes, no input, and recall and turn over the pieces. Recall without input is what loads the agenda. Computers are the worst source of distraction. Hammocks work because people assume you're asleep.
- **Sleep on it, at least one night, for any important decision.** Big abstractions or multi-constraint problems can take weeks or months.
- **Focus costs something.** You'll drop emails and calls. Tell people you care about that it's not personal.

**Managing it under deadlines:**
- Keep **2–3 projects** in rotation. Loading a problem into your head costs an hour to a day, so work one per day and stay on it for days to amortise that. If you're stuck (no progress on question marks, no new possibilities), switch projects or get more input, rather than staying stuck.
- Answers may arrive for a *different* project. Capture them immediately.
- **Then code.** Ideas must be validated by more analysis and by running them. A **small answer is one of the best signs it's good.** Try to avoid lots of typing. You'll iterate (not waterfall), but don't lean on trial and error as the design method ("test-driven dentistry" can't stand in for thinking, as he jokes).
- **You will be wrong, and facts will change** (missed facts, changed requirements). When they do, redo the analysis and change your mind without digging in or apologising. Don't fear being wrong.

**Relevance to Sam:**
- Before a new pipeline or supplier integration, write a one-page design note: problem statement; facts and constraints (16 GB RAM, row counts, provider rate limits, SLA); question marks (fallback when a supplier is down, ID stability, dedupe keys); prior art (how others match GeoNames/OSM, how other OTAs model bookings); and at least two candidate approaches with trade-offs.
- Leaning on AI to write code makes this step *more* important. AI speeds up typing, not conception, and misconception bugs survive every test.
- Use AI as the "input gathering" and "critic" partner (find prior art, attack my design), but keep the hard thinking and the sleeping-on-it.
- His multi-project rotation fits Sam's many parallel projects. Batch by day, capture overnight ideas in a notes inbox, and switch when stuck.

## Arjan Egges (ArjanCodes) — Purge These 7 Code Smells From Your Python Code (https://www.youtube.com/watch?v=Kl3_Gmn4Ujg)
**A code smell is not a bug. It's a hint that the code will be hard to maintain or extend. Most smells here come down to wrong data structures, misplaced responsibilities and hardwired dependencies, fixed by refactoring toward small dataclasses, precise names, passing data in, and injecting dependencies via Protocols.**

Example: a point-of-sale system with `Order` (dataclass with customer fields plus parallel lists), `StripePaymentProcessor`, and `POSSystem`. Prices are stored as integers in the smallest currency unit (cents), a good practice for money.

1. **Parallel lists / ill-fitting data structures.** `items`, `quantities`, `prices` were separate lists that must stay the same length, or totals break. Fix: a `LineItem` dataclass (item, quantity, price) and `items: list[LineItem] = field(default_factory=list)`. *Choosing the right data structure is the core design act.* Make the method take a `LineItem` so `Order` doesn't need to know how one is built.
2. **Misleading names.** `create_line_item` didn't create anything, it appended. Rename to `add_line_item`. Be precise with verbs like create, get, delete, find. Precision also clarifies who owns or manages the lifecycle of the object.
3. **Too many instance variables means too many responsibilities.** `Order` also held customer id, name, address, postal code, city, email. Extract a `Customer` dataclass and give `Order` a `customer: Customer` field. Now customers are reusable elsewhere and `Order` focuses on items, status and id.
4. **"Verb–subject": a function that takes one object and only pokes at its data.** `POSSystem.compute_order_total_price(order)` pulls data out of the order and computes. Move the behaviour to the data: `LineItem.total_price` property (`quantity * price`) and `Order.total_price = sum(li.total_price for li in self.items)` (a generator in `sum` instead of an accumulator loop). (note: he calls this "ask, don't tell". The standard name is **"Tell, Don't Ask"**: tell the object to compute instead of asking for its internals. Compare with Brandon Rhodes, above, who prefers logic in standalone verbs and nouns kept shallow. A simple derived property on a dataclass is shallow and fine under both views. Don't push cross-object workflows into the data classes.)
5. **Backpedalling / Law of Demeter violations.** A method that isn't given the data it needs has to go fetch it: the payment processor took an `order_id`, looked the order up via an `OrderRepository` protocol (implemented by POSSystem), then asked it to compute the total. Symptoms are `self.a.b.c` chains and knowing how other components are wired. Fix: **pass exactly the data needed**: `process_payment(reference: str, price: int)`. The processor no longer depends on POS, the repository protocol disappears, and coupling drops sharply.
6. **Hardwired sequences with a required order.** You had to call `setup_payment_processor(url)` / `connect_to_service` after constructing, or it raised. Fix: a factory that returns a ready object, `StripePaymentProcessor.create(url)`, which constructs, connects and returns. A factory can be `async` when connecting needs awaiting, which `__init__` can't be.
   - (note: he used `@staticmethod` plus `from __future__ import annotations` for the self-referencing return type. In modern Python prefer `@classmethod def create(cls, url) -> Self` (`typing.Self`, 3.11+), which also works for subclasses. Python 3.14's deferred annotations remove the need for the future import.)
7. **Objects that create their own collaborators.** `POSSystem.__init__` instantiated `StripePaymentProcessor` directly, so it was hard-wired to Stripe and hard to test. Fix: **dependency injection** plus a `typing.Protocol` `PaymentProcessor` with only what's used: `def process_payment(self, reference: str, price: int) -> None: ...`. `main` builds the concrete processor and passes it in. Object creation lives in one explicit place (the composition root), and tests can pass a fake.
- **Bonus: positional arguments of the same type are ambiguous.** `LineItem("cable", 500, 6)`: is that 500 cables at 6 or 6 at 500? Swapping postal code and city silently mis-ships. Use keyword arguments at call sites. (A formatter like Black then puts one per line.) Enforce it with `kw_only=True` on dataclasses (3.10+) or `*` in signatures.

**Rules of thumb:**
- Several collections indexed in lockstep means you need a record type.
- If you can't name a method precisely, its responsibility is unclear.
- A class with a long list of fields from different concepts should be split by concept.
- Give a function the data it needs, not a handle to go find it.
- Depend on a narrow Protocol describing only the methods you call.
- Construct concrete objects in `main` and inject them.
- If an object needs a setup step to be usable, give it a factory so it can never exist half-initialised.

**Relevance to Sam:**
- Data scripts often grow parallel lists or dicts (names[], lats[], lons[]). Move to a DataFrame or a record dataclass early.
- Supplier integrations: inject a `SupplierClient` Protocol into the booking service. Build the real client with an async `create()` factory that handles auth and token fetch, and tests pass a stub (this lines up with Rainsberger's collaboration tests).
- Booking or payment functions should take `(reference, amount_minor_units, currency)`, not a whole repository.
- Store money as integer minor units.
- When reviewing AI-generated code, scan specifically for these 7 smells. They are typical LLM output patterns: god-dataclasses, inline construction of clients, and positional-argument soup.

## Jack Diederich — Deleting Code Is Hard and You Should Do It (https://www.youtube.com/watch?v=XD24Il-l7YQ)
**Code is a liability, not an asset. Customers pay for what the product does, not for lines. Less code means fewer engineers needed, easier maintenance, and easier proof that the rest is used. Deleting is hard in Python, so build tooling, conventions and culture that make it routine.** (Short lightning-talk version of his PyOhio talk. Same speaker as "Stop Writing Classes".)

- **Who sees code.** Only engineering sees the back end. Colleagues see the product. Customers just want it fast and working. Nobody outside engineering is paying for clever back-end code.
- **Bad reasons not to delete:**
  - *Not invented here / ownership.* If an open-source project already does it, switch. Someone else maintains it, and you can upstream your customisations.
  - *"I might need it later."* You won't, and version control archives it anyway. Commented-out code can always go.
  - *"It might be used somewhere."* This one is real in a dynamic language. Use tools, but know their limits.
- **Finding what's actually used (all imperfect):**
  - **coverage.py** shows only lines that actually ran, so you must record *every entry point*: web servers, cron jobs, crawlers, ML/NLP workers. Then collate. Their 800-module codebase had only 261, 425 and 426 modules imported on three machine types, and the two similar-looking sets were only about 60% identical. Hint: if something looks web-only and doesn't show in the web server's coverage profile, it's probably dead.
  - **grep** finds `obj.hello` easily, but `getattr(obj, "hello")` or string-built names hide calls. War story: a colleague was sure a line was unused, and it was reached via an indirect lookup. Deploying the deletion would have broken production.
  - Python has no compiler to prove something is unreferenced. (note: modern tools like `vulture`, `ruff` (unused imports/args), and IDE "find usages" help, but have the same dynamic-dispatch blind spots.)
- **Mitigations that keep code deletable:**
  - **Keep dynamic lookups close to the definition:** in the same class, or at least the module, or at worst the package. That keeps the search space small.
  - **Avoid being clever.** Metaclasses rarely help anyone, class decorators are slightly better, and `__getattr__` can hide things.
  - **Delete continuously.** Less code means a smaller search and more certainty. Dead code forms **cycles** (A calls B calls A) that make each other look used.
  - **Delete unused entry-point scripts.** A stale script in `scripts/` makes a whole tree of dead code look alive, and people leave "someone else's script" alone.
  - **Import-test every module nightly** (find every .py and `python -c "import module"`). It catches circular imports. It took ~30 min on their codebase, too slow for smoke tests but fine daily, so a breakage narrows to "something today". `git bisect` is painful when each step is that slow, which is why daily granularity matters.
- **Chesterton's fence.** Don't remove something until you know why it's there. Version control archives both *content* and *purpose*: use `git blame` and walk back to the commit that introduced the line and its message. If the reason is bad, delete. If it guards a real edge case, keep it (and document it).
  - Cheap trick: snapshot the repo (`cp -a`) quarterly into a folder you can recursively grep for half-remembered names.
- **Culture.** "Trash day" every Friday: if nothing urgent, delete something. **No individual deletion metrics.** People game them, or kindly avoid deleting others' code to let them get the credit. Keep only team-level stats per quarter. If you can't figure out a piece, ask its author. Absent a good reason, announce "deleting this unless you stop me in 30 minutes".
- **Result:** they added features while keeping total lines flat (Python replaced much more Java and XML), then cut Python lines another ~10% in a quarter while still adding features.
- Q&A: comments are mostly lies, but for genuinely non-obvious code (a Chesterton's fence) a short "why" comment pays off.

**Relevance to Sam:**
- A solo builder using AI generates code fast, so the deletion habit matters even more. AI adds code readily and rarely removes it.
- Schedule a weekly cleanup: delete one-off scripts, dead branches of matching logic, and commented-out experiments in the GeoNames, transit and supplier repos.
- Run `coverage run` across each real entry point (pipeline CLI, API server, cron jobs) and union the reports before deleting.
- Avoid `getattr`-driven provider dispatch scattered across modules. Keep a single registry dict next to the adapters so usages are greppable.
- Write commit messages that state *why*, so future-you (or an AI agent) can check the fence before tearing it down.
- Put a nightly "import every module" check plus `ruff`/`vulture` in CI.

